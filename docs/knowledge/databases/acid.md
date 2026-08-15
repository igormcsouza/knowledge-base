---
tags:

- databases
- transactions
- acid
- fundamentals

---

# ACID

ACID is the set of four guarantees a database transaction should provide so
that data stays correct even when things go wrong: crashes, concurrent
requests, or partial failures mid-operation. It stands for **Atomicity**,
**Consistency**, **Isolation**, and **Durability**.

A running example throughout: transferring $100 from Account A to Account B.
That "transfer" is really two writes (debit A, credit B) that need to behave
as one unit.

## Atomicity

A transaction is all-or-nothing. Either every statement in it succeeds and
gets committed, or none of them do and the database is rolled back to how it
was before.

In the transfer example: if the debit from A succeeds but the credit to B
fails (network blip, constraint violation, crash), atomicity guarantees the
debit gets rolled back too. Without it, $100 would simply vanish.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';

COMMIT; -- if anything above failed, this becomes ROLLBACK instead
```

## Consistency

A transaction can only move the database from one **valid** state to another
valid state — it can't violate constraints, foreign keys, triggers, or
application-defined invariants (like "balance can never go negative").

This is the one property that's partly the database's job (enforcing
constraints) and partly the application's job (writing correct transaction
logic). The database can stop a transaction that would violate a `CHECK`
constraint, but it can't stop you from writing business logic that's wrong
in a way SQL can't express.

## Isolation

Concurrent transactions shouldn't see each other's uncommitted, in-progress
changes. Isolation defines how strictly transactions are kept from
interfering with one another, and it's a spectrum, not a single guarantee —
controlled by an **isolation level**.

Three classic phenomena isolation levels protect against, from least to most
severe:

- **Dirty read** — reading a row another transaction has changed but not yet
  committed. If that transaction rolls back, you read data that never
  "really" existed.
- **Non-repeatable read** — reading the same row twice in one transaction and
  getting different values, because another transaction committed a change to
  it in between.
- **Phantom read** — re-running the same query twice in one transaction and
  getting a different *set* of rows, because another transaction inserted or
  deleted rows matching the query in between.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible* |
| Serializable | Prevented | Prevented | Prevented |

\* Some databases (e.g. PostgreSQL's `REPEATABLE READ`) prevent phantoms too,
going beyond the SQL standard's minimum guarantee for that level.

!!! tip "Pro Tip"
    `READ COMMITTED` is the default in PostgreSQL, Oracle, and SQL Server, and
    is a reasonable default for most application code. Reach for
    `SERIALIZABLE` only for the specific transactions where correctness under
    concurrency really matters (e.g. moving money, decrementing limited stock)
    — it comes with a real cost in throughput and retry logic (serialization
    failures need to be caught and retried).

In the transfer example: without proper isolation, a concurrent transaction
reading Account A's balance mid-transfer could see a stale or partially
updated value and make a decision based on it (e.g. approving another
withdrawal that overdraws the account).

## Durability

Once a transaction is committed, it stays committed — even if the database
crashes, loses power, or the process is killed a millisecond later. This is
typically achieved with a **write-ahead log (WAL)**: changes are flushed to
durable storage (and fsync'd) before the commit is acknowledged to the
client, so a crash-and-recover cycle can replay the log and reconstruct the
committed state.

In the transfer example: once the API tells the user "transfer complete," a
server crash one second later must not lose that $100 movement.

## Contrast: BASE

NoSQL / distributed systems often trade strict ACID guarantees (particularly
strong consistency) for availability and partition tolerance, following the
looser **BASE** model instead: **B**asically **A**vailable, **S**oft state,
**E**ventual consistency. Neither is strictly better — it's a trade-off
dictated by the system's requirements (a bank ledger wants ACID; a social
media "like" counter can tolerate eventual consistency).

## Summary

- **Atomicity** — all statements in a transaction succeed, or none do.
- **Consistency** — a transaction only moves data between valid states.
- **Isolation** — concurrent transactions don't corrupt each other's view of
  the data; the isolation level tunes how strictly.
- **Durability** — once committed, a transaction survives a crash.

## Related Articles

- [Event-Driven Architecture](../architecture/event-driven-architecture.md) —
  message brokers make different consistency trade-offs than ACID databases.
