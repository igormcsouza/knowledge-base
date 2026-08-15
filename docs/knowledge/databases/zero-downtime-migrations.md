---
tags:

- databases
- migrations
- zero-downtime
- postgresql

---

# Zero-Downtime Migrations

A rolling deploy means old and new application code run against the same
database at the same time, for however long the rollout takes. Any schema
change that the *old* code can't tolerate breaks production the moment the
first new pod deploys and the migration runs — regardless of how trivial
the change looks in a diff. This article covers the expand/contract pattern
for making schema changes safely under that constraint.

## The Core Problem

During a Kubernetes rolling update (see
[Kubernetes: Rolling Updates and Rollbacks](../devops-tools/kubernetes.md#rolling-updates-and-rollbacks)),
new pods come up and start serving traffic *before* old pods are terminated
— by design, so there's no capacity gap. That means for the whole duration
of the rollout, both the old and new versions of the application are
running concurrently, against the same database.

If a migration and a code deploy assume "the schema change and the code
change happen atomically, together," that assumption is false under a
rolling deploy: the migration typically runs once (often before the new
pods start), but old pods keep running the old code against the *new*
schema for as long as they're still up, and new pods might start before
every old pod is gone. Either direction of mismatch — old code against new
schema, or new code against old schema, depending on ordering — has to be
survivable.

## Why "Just Rename the Column" Isn't Safe

A single migration that renames a column looks like a trivial, one-line
change:

```sql
ALTER TABLE users RENAME COLUMN email TO email_address;
```

But the moment this runs, **every old pod still running the previous
version breaks immediately** — its code says `SELECT email FROM users`, and
that column no longer exists. This isn't a slow degradation or an edge
case; it's every request touching that query failing, for as long as any
old pod is still serving traffic, which during a rolling update is exactly
the transition window that's supposed to be safe.

```text
psycopg2.errors.UndefinedColumn: column "email" does not exist
LINE 1: SELECT email FROM users WHERE id = $1
               ^
```

The one-line-diff feeling is the trap: nothing about the SQL statement's
*size* signals how disruptive it is to a system with two versions of the
application running against it simultaneously.

## The Expand/Contract Pattern

Also called **parallel change**, this pattern splits one "unsafe" schema
change into several individually-safe steps, deployed across multiple
releases, so that at every point in time the schema is compatible with
*both* the old and new application code that might be running against it.

The shape is always: **expand** (add, additively, nothing removed) → **dual
support** (both old and new code work) → **contract** (remove the old thing,
once nothing depends on it anymore).

### Concrete Example: Renaming `email` to `email_address`

**Step 1 — Expand.** Add the new column, nullable (or with a default), next
to the old one. This is purely additive — no existing code, old or new,
is affected by a new column simply existing.

```sql
ALTER TABLE users ADD COLUMN email_address TEXT;
```

Deploy this migration alone. Old code (reads/writes `email`) keeps working
exactly as before — the new column is just unused so far.

**Step 2 — Dual-write.** Deploy application code that writes *both*
columns on every insert/update, while still reading from the old column.

```python
def update_email(user_id: int, new_email: str) -> None:
    db.execute(
        "UPDATE users SET email = %s, email_address = %s WHERE id = %s",
        (new_email, new_email, user_id),
    )
```

At this point old pods (still on the pre-dual-write code) keep working
against `email` untouched; new pods keep both columns in sync. Either
version running is safe.

**Step 3 — Backfill.** Populate `email_address` for every existing row that
predates the dual-write code — see batching below, since this touches every
row in the table.

**Step 4 — Cut over reads.** Deploy application code that reads from
`email_address` instead of `email`, while still dual-writing both (in case
a rollback to the previous release is needed — a rollback at this point
should land on code that still works, not on code expecting a column that's
about to be dropped).

```python
def get_email(user_id: int) -> str:
    row = db.execute(
        "SELECT email_address FROM users WHERE id = %s", (user_id,)
    ).fetchone()
    return row.email_address
```

**Step 5 — Stop dual-writing.** Once every running pod is confirmed to be
on the read-from-new code (no rollback risk to a version that still needs
`email`), deploy code that writes only `email_address`.

**Step 6 — Contract.** Only now, once nothing reads or writes `email`
anymore, drop it.

```sql
ALTER TABLE users DROP COLUMN email;
```

Six steps for what looked like a one-line rename — but at every single
point in that sequence, both the old and new application code that could
plausibly be running have a schema they can work with. That's the entire
goal; nothing else about the sequence is arbitrary.

!!! tip "Pro Tip"
    Not every schema change needs the full six-step version. Purely
    additive changes (a new nullable column nobody reads yet, a new table,
    a new index built `CONCURRENTLY`) are safe in one step — expand/contract
    exists specifically for changes that would otherwise remove or reshape
    something old code still depends on.

## Backfilling Large Tables in Batches

A single `UPDATE` touching every row of a large table holds locks (or at
minimum sustained write load and WAL/binlog volume) for as long as it runs
— on a multi-million-row table that can be minutes of degraded write
throughput and growing replication lag, all from one migration step that's
supposed to be invisible.

```sql
-- Risky on a large table: one long-running statement, one long lock window
UPDATE users SET email_address = email WHERE email_address IS NULL;
```

Batching breaks the same work into many small, short transactions, each
committing (and releasing its locks) before the next batch starts:

```python
BATCH_SIZE = 1000

def backfill_email_address(engine):
    while True:
        with engine.begin() as conn:
            result = conn.execute(
                """
                UPDATE users
                SET email_address = email
                WHERE id IN (
                    SELECT id FROM users
                    WHERE email_address IS NULL
                    LIMIT :batch_size
                    FOR UPDATE SKIP LOCKED
                )
                """,
                {"batch_size": BATCH_SIZE},
            )
        if result.rowcount == 0:
            break
        time.sleep(0.1)  # brief pause — let replicas catch up, ease load
```

`FOR UPDATE SKIP LOCKED` (see
[Locking, Deadlocks, and Concurrency](locking-deadlocks-concurrency.md))
keeps this from fighting concurrent application writes for the same rows,
and the small sleep between batches gives replication and connection pools
room to breathe instead of running the backfill flat-out. Each batch
committing independently also means a crashed/killed backfill can simply
resume where it left off, instead of losing all progress like one giant
transaction would on rollback.

!!! warning "Watch replication lag during a backfill"
    On a primary-replica setup, every row changed on the primary has to
    replay on every replica. A backfill that runs faster than replicas can
    apply changes widens replication lag — and if anything reads from a
    replica (a common scaling pattern), that lag becomes stale reads in
    production for the duration. Batch size and inter-batch pauses are the
    two knobs for keeping a backfill from outrunning replication.

## Summary

- A rolling deploy means old and new code run against the database
  simultaneously — a migration has to be safe for both, not just the
  version being deployed.
- A single "rename the column" migration breaks every currently-running old
  pod instantly, despite looking like a trivial change.
- Expand/contract (parallel change): add new, dual-write, backfill, cut
  over reads, stop dual-writing, then drop the old — each step
  independently safe for whichever app version is running.
- Backfill large tables in small batches with brief pauses between them,
  not one long-running statement — protects both lock contention and
  replication lag.
- Purely additive changes (new nullable column, new index built
  concurrently) don't need the full pattern — it's specifically for changes
  that remove or reshape something old code still relies on.

## Related Articles

- [Kubernetes: Rolling Updates and Rollbacks](../devops-tools/kubernetes.md#rolling-updates-and-rollbacks)
  — the deploy mechanism that creates the old-code/new-code overlap window
  this whole pattern exists to survive.
- [Alembic and SQLAlchemy 2.0 Migrations](alembic-sqlalchemy2-migrations.md)
  — the tooling for writing and sequencing the individual migration steps.
- [Locking, Deadlocks, and Concurrency](locking-deadlocks-concurrency.md) —
  `FOR UPDATE SKIP LOCKED`, used above to batch a backfill without fighting
  concurrent application writes.
- [Query Performance Bottlenecks](query-performance-bottlenecks.md) — an
  unbatched backfill is the same "one query doing too much work" problem
  covered there, just via `UPDATE` instead of `SELECT`.
