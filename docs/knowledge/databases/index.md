---
title: Databases
tags:

- databases
- overview

---

# Databases

Database fundamentals: transactions, consistency guarantees, and the
concepts that hold up regardless of which specific database you're using.

## Articles

- [ACID](acid.md) — Atomicity, Consistency, Isolation, and Durability, plus
  isolation levels and the phenomena they prevent.
- [Idempotency](idempotency.md) — why retries need it, and the standard
  techniques (idempotency keys, upserts, compare-and-swap) for getting it.
- [Redis as a Caching Layer](redis-caching.md) — key design, TTLs and
  eviction policies, cache-aside/write-through/write-behind, cache
  stampedes, and invalidation.
- [SQL vs. NoSQL: PostgreSQL, MySQL, and MongoDB](sql-vs-nosql-postgres-mysql-mongo.md)
  — schema-on-write vs. schema-on-read, transaction support, relationship
  modeling, and a focused Postgres-vs-MySQL comparison.
- [Locking, Deadlocks, and Concurrency](locking-deadlocks-concurrency.md) —
  pessimistic vs. optimistic locking, row vs. table locks, and how
  databases detect and break deadlocks.
- [Query Performance Bottlenecks](query-performance-bottlenecks.md) —
  reading `EXPLAIN` output, when indexes help, and common slow-query
  anti-patterns.
- [SQLAlchemy 2.0 and Alembic Migrations](alembic-sqlalchemy2-migrations.md)
  — the 2.0 declarative style, autogeneration's real limitations, and
  managing migrations safely.
- [Zero-Downtime Migrations](zero-downtime-migrations.md) — the
  expand/contract pattern for schema changes that survive a rolling
  deploy.

## Contributing

Learned something about databases worth keeping — a concept, a gotcha, a
tuning tip? Add it here as its own file. See
[Contributing](../../contributing.md) for the how-to.
