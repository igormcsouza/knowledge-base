---
tags:

- databases
- idempotency
- transactions
- fundamentals

---

# Idempotency

An operation is **idempotent** if applying it once has the same effect as
applying it many times. In database terms: if a write gets retried — because
of a timeout, a dropped response, a message redelivered by a queue — an
idempotent write leaves the data in the same state as a single successful
write would have. A non-idempotent one keeps changing the data every time it
runs.

## A Concrete Example

```sql
-- NOT idempotent: retrying this N times debits the account N times
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';

-- Idempotent: retrying this any number of times leaves balance at exactly 500
UPDATE accounts SET balance = 500 WHERE id = 'A';
```

The first statement is a *relative* change — it depends on the current state,
so applying it twice changes the outcome. The second is an *absolute* change
— it sets a known target state, so applying it any number of times converges
to the same result. Most idempotency problems come down to this distinction:
relative writes (`+=`, `-=`, "append", "increment") are dangerous to retry;
absolute writes ("set to X") are safe.

## Why It Matters

The database side of a network call can succeed while the *response* never
makes it back to the caller (timeout, dropped connection, crash right after
commit). The caller has no way to tell "it failed" apart from "it succeeded
but I didn't hear back," so the only safe default is to retry. That's only
actually safe if the retried operation is idempotent — otherwise "just retry
it" silently corrupts data (double-charging a card, double-counting an
inventory decrement, sending a notification twice).

This shows up constantly in systems built around
[event-driven architecture](../architecture/event-driven-architecture.md):
most message brokers (Kafka, RabbitMQ, SQS Standard queues) only guarantee
**at-least-once delivery** — a consumer might see the same event more than
once, by design, in exchange for never silently dropping one. Idempotent
consumers are what make that trade-off safe.

## Common Techniques

**Idempotency keys** — the caller generates a unique key per *logical*
operation (not per HTTP request/retry) and sends it along; the server records
which keys it has already processed and short-circuits duplicates instead of
re-executing the write.

```python
def charge_card(idempotency_key: str, amount: int) -> Charge:
    existing = db.get_charge_by_key(idempotency_key)
    if existing is not None:
        return existing  # already processed — return the prior result, don't charge again

    charge = payment_provider.charge(amount)
    db.save_charge(idempotency_key, charge)
    return charge
```

**Upserts** instead of blind inserts — `INSERT ... ON CONFLICT DO UPDATE`
(PostgreSQL/SQLite) or `MERGE` (SQL Server/Oracle) make "create or update"
safe to repeat, since a retried insert doesn't produce a duplicate row.

```sql
INSERT INTO subscriptions (user_id, plan)
VALUES ('u1', 'pro')
ON CONFLICT (user_id) DO UPDATE SET plan = EXCLUDED.plan;
```

**Conditional / compare-and-swap updates** — instead of a relative update,
make the write depend on the state you expect to be changing *from*, using
optimistic-concurrency-style versioning:

```sql
UPDATE accounts SET balance = 500, version = version + 1
WHERE id = 'A' AND version = 7;
```

If this runs twice, the second attempt matches zero rows (the version already
moved on) instead of applying the change again — the retry becomes a safe
no-op.

**REST verb semantics** — `PUT` and `DELETE` are idempotent *by design* in
HTTP's contract (repeating them should produce the same server state), while
`POST` is not (each call is conventionally treated as creating something
new). This is exactly why "resource creation" endpoints are the ones that
usually need an explicit idempotency key — the HTTP method alone doesn't give
you the guarantee.

## Idempotency vs. ACID

These solve different problems and both matter at once:

- **ACID** guarantees a *single* transaction is executed correctly — all its
  statements commit together, isolated from other concurrent transactions,
  durably. See [ACID](acid.md).
- **Idempotency** guarantees that *repeating* an operation — a whole
  transaction, an API call, a whole retried request — from the outside
  doesn't change the outcome further.

A transaction can be perfectly ACID-compliant and still be non-idempotent:
`UPDATE balance = balance - 100` inside a fully ACID transaction is still
unsafe to retry, because atomicity/isolation/durability say nothing about
what happens when the *same* transaction runs a second time.

## Summary

- Idempotent = same result whether applied once or N times; the key test is
  "is this an absolute state change or a relative one?"
- Retries are unavoidable in distributed systems (timeouts, at-least-once
  delivery) — idempotency is what makes them safe instead of destructive.
- Idempotency keys, upserts, and conditional/compare-and-swap updates are the
  standard tools for turning a relative write into a safe-to-retry one.
- It's a distinct guarantee from ACID — a transaction can be fully
  ACID-compliant and still corrupt data if retried blindly.

## Related Articles

- [ACID](acid.md) — the guarantees a single transaction provides, distinct
  from what happens if it's retried.
- [Event-Driven Architecture](../architecture/event-driven-architecture.md)
  — at-least-once delivery is exactly where idempotent consumers matter most.
- [Kafka Consumers Behind a FastAPI API on Kubernetes](../architecture/kafka-consumers-fastapi-kubernetes.md)
  — a concrete case for designing consumers around at-least-once delivery.
- [AWS Lambda](../devops-tools/aws/lambda.md) and
  [Amazon SQS](../devops-tools/aws/sqs.md) — a concrete AWS-specific case
  where at-least-once delivery makes idempotent processing mandatory.
