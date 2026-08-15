---
tags:

- databases
- locking
- concurrency
- transactions

---

# Locking, Deadlocks, and Concurrency

Two transactions touching the same row at the same time need to be kept from
corrupting each other's work — that's what locking exists to do. This
article covers how relational databases lock rows and tables, the two
philosophies (pessimistic vs. optimistic) for handling concurrent writes,
what a deadlock actually is, and how to avoid causing one in production.

## Pessimistic Locking

Pessimistic locking assumes conflict is likely, so it locks the row *before*
reading it, blocking any other transaction that wants to touch the same row
until the lock is released.

```sql
BEGIN;

-- Locks this row until COMMIT/ROLLBACK; any other transaction trying to
-- SELECT ... FOR UPDATE (or UPDATE/DELETE) the same row blocks and waits.
SELECT * FROM inventory WHERE sku = 'WIDGET-1' FOR UPDATE;

UPDATE inventory SET quantity = quantity - 1 WHERE sku = 'WIDGET-1';

COMMIT;
```

Between the `SELECT ... FOR UPDATE` and the `COMMIT`, no other transaction
can read-and-lock (or update) that row — they queue up behind it. This is
the right tool when conflicts are frequent enough that retrying optimistic
writes would waste more work than the wait costs (a "decrement limited
stock" endpoint under real contention is the textbook case).

**`FOR UPDATE SKIP LOCKED`** is the pattern behind most hand-rolled SQL job
queues: instead of blocking on a locked row, skip it and grab the next
available one. Multiple workers can then pull from the same table
concurrently without piling up behind each other.

```sql
BEGIN;

SELECT id, payload FROM jobs
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;

UPDATE jobs SET status = 'processing' WHERE id = 123;

COMMIT;
```

Without `SKIP LOCKED`, N workers running the plain `FOR UPDATE` version
would serialize on the same "next pending row" — each worker blocking behind
whichever one got there first, even though they'd happily take *different*
rows if they weren't forced to wait for the exact one another worker already
claimed.

## Optimistic Locking

Optimistic locking assumes conflict is rare: read without locking, then make
the write conditional on nothing having changed since the read, typically
via a version column.

```sql
-- Read: note the current version
SELECT balance, version FROM accounts WHERE id = 'A';  -- version = 7

-- Write: only succeeds if version is still 7
UPDATE accounts SET balance = 500, version = version + 1
WHERE id = 'A' AND version = 7;
-- 0 rows affected => someone else updated it first; the app must retry
-- (re-read, re-check the business logic, and re-attempt the write).
```

This is the same compare-and-swap technique covered in
[Idempotency](idempotency.md#common-techniques) for making retries safe —
the same mechanism serves both goals, since a conditional write is exactly
what makes both "retry this safely" and "don't clobber someone else's
concurrent change" work.

Optimistic locking never blocks a reader and scales better under low
contention, but it pushes the retry loop into application code, and under
*high* contention it can actually do more total work than pessimistic
locking would (repeated read-fail-retry cycles vs. one queue-and-wait).

!!! tip "Pro Tip"
    Choose based on measured contention, not instinct: pessimistic locking
    for a hot, contended resource (limited inventory, a singleton counter);
    optimistic locking for the common case of "usually nobody else is
    touching this row right now" (a user editing their own profile).

## Row-Level vs. Table-Level Locks

Both Postgres and MySQL (InnoDB) default to **row-level locking** — an
`UPDATE` or `SELECT ... FOR UPDATE` locks only the specific rows it touches,
letting unrelated rows in the same table be modified concurrently by other
transactions. This is what makes row-level locking scale: two transactions
updating different rows of a million-row table don't contend with each
other at all.

**Table-level locks** lock the entire table, and come up in narrower cases:
DDL (`ALTER TABLE` typically needs one), explicit `LOCK TABLE` statements,
or as an escalation path some databases use under specific conditions. A
table lock blocks *everything* touching that table, so it's a much bigger
hammer — reach for row-level locking (the default for ordinary DML) unless
there's a specific reason a whole table needs to be exclusive.

## What a Deadlock Actually Is

A deadlock is two (or more) transactions each holding a lock the other one
needs, so neither can proceed — a cycle of "waiting for you" with no way
out except one of them being forced to give up.

Concretely, with two transactions and two rows:

```sql
-- Transaction 1                      -- Transaction 2
BEGIN;                                BEGIN;
UPDATE accounts                       UPDATE accounts
  SET balance = balance - 50            SET balance = balance - 50
  WHERE id = 'A';                       WHERE id = 'B';
-- T1 now holds a lock on row A       -- T2 now holds a lock on row B

UPDATE accounts                       UPDATE accounts
  SET balance = balance + 50            SET balance = balance + 50
  WHERE id = 'B';                       WHERE id = 'A';
-- T1 wants row B, held by T2         -- T2 wants row A, held by T1
-- T1 blocks, waiting for T2...       -- ...T2 blocks, waiting for T1.
```

T1 is transferring money A→B; T2 is transferring money B→A, at the same
moment, with the row updates issued in the *opposite order* from each
other. T1 has locked A and wants B; T2 has locked B and wants A. Neither
transaction can ever get what it's waiting for, because the thing each of
them needs is held by the other one, which is itself stuck waiting. Without
intervention this would wait forever.

## Detection and Resolution

Both Postgres and MySQL (InnoDB) run active **deadlock detection**: they
watch for exactly this kind of wait cycle and, once found, pick one
transaction as the **victim** and force it to fail with an error, releasing
its locks so the other transaction can proceed.

```text
ERROR:  deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678; blocked by
        process 5678.
        Process 5678 waits for ShareLock on transaction 1234; blocked by
        process 1234.
```

The victim's transaction is rolled back automatically — it never partially
applies. The application is expected to catch this specific error and
retry the whole transaction from the start (typically after the row it was
waiting on has since become free).

```python
from sqlalchemy.exc import OperationalError

def transfer(session, from_id, to_id, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            session.execute(
                "UPDATE accounts SET balance = balance - :amt WHERE id = :id",
                {"amt": amount, "id": from_id},
            )
            session.execute(
                "UPDATE accounts SET balance = balance + :amt WHERE id = :id",
                {"amt": amount, "id": to_id},
            )
            session.commit()
            return
        except OperationalError as e:
            session.rollback()
            if "deadlock" not in str(e).lower() or attempt == max_retries - 1:
                raise
            # retry — the deadlock victim's rollback already freed the lock
```

!!! note "This isn't a bug — it's the system working as designed"
    A deadlock error means the database correctly detected an impossible
    wait cycle and broke it rather than hanging forever. The bug, if there
    is one, is upstream: the application let two transactions lock the same
    two rows in opposite orders. Treat a deadlock error as a signal to fix
    lock ordering, not just as "add a retry and move on" — though the retry
    is also necessary, since occasional deadlocks under real concurrency are
    normal even with good lock ordering.

## Practical Prevention

**Consistent lock ordering.** The deadlock above only happens because T1 and
T2 locked A and B in opposite order. If every transaction that touches both
rows always locks them in the same order (e.g. always by ascending primary
key), the cycle becomes structurally impossible — one transaction always
gets there first for *both* rows.

```sql
-- Always lock in ascending ID order, regardless of "from"/"to" semantics:
UPDATE accounts SET balance = balance - 50 WHERE id = LEAST('A', 'B');
UPDATE accounts SET balance = balance + 50 WHERE id = GREATEST('A', 'B');
```

**Keep transactions short.** The longer a transaction holds a lock, the
longer the window for another transaction to want the same row. Do
non-database work (calling an external API, rendering a template) *outside*
the transaction, not in the middle of it.

**Avoid lock order depending on user input.** A batch operation that locks
rows in whatever order they arrived in a request payload (user-controlled)
makes lock order effectively random across requests — exactly the setup for
occasional deadlocks under concurrency. Sort by a stable key (primary key)
before locking, every time, regardless of input order.

```python
# Bad: lock order follows whatever order the client sent
for account_id in request.account_ids:
    lock_and_update(account_id)

# Good: deterministic order regardless of input order
for account_id in sorted(request.account_ids):
    lock_and_update(account_id)
```

## Isolation Levels

Isolation level and locking are related but distinct: isolation level
controls what a transaction is allowed to *see* from concurrent
transactions (dirty reads, non-repeatable reads, phantoms), while locking is
one of the mechanisms the database uses to *enforce* that. Higher isolation
levels generally mean more locking (or, in Postgres's MVCC-based approach,
more serialization-failure retries instead of blocking). See
[ACID](acid.md#isolation) for the isolation levels themselves and the
phenomena each one prevents — not repeated here.

## Summary

- Pessimistic locking (`SELECT ... FOR UPDATE`) blocks conflicting readers
  immediately; `FOR UPDATE SKIP LOCKED` is the standard building block for
  SQL-backed job queues with multiple concurrent workers.
- Optimistic locking (version columns / compare-and-swap) never blocks and
  scales better under low contention, pushing retry logic into the
  application — the same mechanism [Idempotency](idempotency.md) uses for
  safe retries.
- Row-level locking is the default and the right level for almost all DML;
  table-level locks are a much bigger hammer reserved for DDL and special
  cases.
- A deadlock is a cycle of transactions each waiting on a lock the other
  holds; Postgres/MySQL detect the cycle and roll back one transaction (the
  victim) to break it.
- Prevention beats retries: consistent lock ordering, short transactions,
  and never letting user-controlled input determine lock order.
- Isolation level and locking are related but distinct — see
  [ACID](acid.md#isolation) for isolation levels themselves.

## Related Articles

- [ACID](acid.md) — isolation levels and the phenomena they prevent.
- [Idempotency](idempotency.md) — compare-and-swap writes as the shared
  mechanism behind optimistic locking and safe retries.
- [Query Performance Bottlenecks](query-performance-bottlenecks.md) — long
  transactions and unindexed lookups both widen the window for lock
  contention and deadlocks.
