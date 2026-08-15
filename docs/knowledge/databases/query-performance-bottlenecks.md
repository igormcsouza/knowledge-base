---
tags:

- databases
- performance
- postgresql
- mysql

---

# Query Performance Bottlenecks

"The database is slow" is almost never true as stated — it's usually one
specific query doing more work than it needs to, an index that doesn't
exist or doesn't get used, or the application making far more round trips
than the task requires. This article covers how to read `EXPLAIN` output,
when indexes actually help, common anti-patterns, and the tooling to find
the slow query in the first place.

## Reading `EXPLAIN` / `EXPLAIN ANALYZE`

`EXPLAIN` shows the query planner's *chosen* execution plan and its cost
estimates without running the query; `EXPLAIN ANALYZE` actually runs it and
shows real timings and row counts alongside the estimates — the second is
what you want when diagnosing an already-slow query.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';
```

```text
Seq Scan on orders  (cost=0.00..18334.00 rows=12 width=97)
                    (actual time=0.021..142.558 rows=11 loops=1)
  Filter: (customer_id = 42 AND status = 'pending')
  Rows Removed by Filter: 999989
Planning Time: 0.112 ms
Execution Time: 142.601 ms
```

What to read out of this:

- **Seq Scan vs. Index Scan vs. Index-Only Scan.** A `Seq Scan` reads every
  row in the table and filters afterward — fine for a small table, a red
  flag on a large one being filtered down to a handful of rows, as above (1
  million rows scanned to find 11). An `Index Scan` uses an index to jump
  straight to matching rows, then fetches the full row from the table. An
  `Index-Only Scan` is the fastest of the three: every column the query
  needs is present *in the index itself*, so it never touches the table at
  all.
- **Cost estimates** (`cost=0.00..18334.00`) are the planner's *predicted*
  relative cost (startup cost..total cost, in arbitrary planner units, not
  milliseconds) — useful for comparing plans, not for reading as real time.
- **`rows=12` (estimated) vs. `rows=11 actual`** — a small gap like this is
  normal. A *large* mismatch (planner expects 12 rows, actually gets
  400,000) is one of the most common root causes of a bad plan: the planner
  chooses a plan that would be efficient for the row count it expects, and
  that choice turns out wrong for the row count it actually gets. This
  usually means table statistics are stale — `ANALYZE tablename` refreshes
  them.

```sql
EXPLAIN ANALYZE
SELECT customer_id FROM orders WHERE customer_id = 42;
```

```text
Index Only Scan using idx_orders_customer_id on orders
  (cost=0.42..8.65 rows=11 width=4)
  (actual time=0.018..0.021 rows=11 loops=1)
  Index Cond: (customer_id = 42)
  Heap Fetches: 0
Planning Time: 0.083 ms
Execution Time: 0.041 ms
```

Same table, indexed column, only selecting the indexed column: 142ms drops
to 0.04ms. That gap is the entire reason indexing matters.

## When an Index Helps (and When It Doesn't)

**Selectivity** is the key factor: an index only helps when it narrows the
result down to a small fraction of the table. An index on a `status` column
with only two possible values (`active`/`inactive`, roughly 50/50 split)
gives the planner little to work with — it may reasonably choose a sequential
scan anyway, since "half the table" isn't meaningfully cheaper to fetch via
an index than via a scan.

```sql
-- Low selectivity — an index here often won't get used
CREATE INDEX idx_orders_status ON orders (status);  -- 'pending'/'shipped' only

-- High selectivity — a great index candidate
CREATE INDEX idx_orders_customer_id ON orders (customer_id);  -- many distinct values
```

**Composite index column order matters.** A multi-column index is only
usable left-to-right — an index on `(customer_id, status)` speeds up
queries filtering on `customer_id` alone or on `customer_id AND status`, but
does *nothing* for a query filtering on `status` alone.

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);

-- Uses the index (customer_id is the leading column)
SELECT * FROM orders WHERE customer_id = 42;
SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';

-- Cannot use this index (status is not the leading column)
SELECT * FROM orders WHERE status = 'pending';
```

!!! tip "Pro Tip"
    Put the column used in equality filters (`= value`) before columns used
    in range filters (`> value`, `BETWEEN`) in a composite index — an
    equality-then-range order lets the index narrow down by the equality
    column first, then scan a tight range within it; range-then-equality
    can't narrow the same way.

**Covering indexes** include every column a query needs (in the `SELECT`,
`WHERE`, and `ORDER BY`), so the query never touches the table at all — the
index-only scan case above. Postgres's `INCLUDE` clause adds columns to the
index for this purpose without making them part of the sort order:

```sql
CREATE INDEX idx_orders_covering
  ON orders (customer_id)
  INCLUDE (status, total_cents);
```

An index isn't free, though — every index adds write overhead (each insert/
update maintains every index on the table) and storage. An index that no
query actually uses is pure cost; periodically checking for unused indexes
(`pg_stat_user_indexes` in Postgres) is worth doing on a mature schema.

## The N+1 Problem (Database-Access-Pattern Level)

At the database level, N+1 is simply: one query to fetch a list of N
records, followed by N *more* individual queries — one per record — to
fetch related data, instead of one query (a join, or a single `WHERE id IN
(...)`) that gets it all at once.

```python
# N+1: 1 query for orders, then 1 query per order for its customer
orders = db.execute("SELECT * FROM orders LIMIT 50").fetchall()
for order in orders:
    customer = db.execute(
        "SELECT * FROM customers WHERE id = %s", (order.customer_id,)
    ).fetchone()

# Fixed: 2 queries total, regardless of how many orders there are
orders = db.execute("SELECT * FROM orders LIMIT 50").fetchall()
customer_ids = [o.customer_id for o in orders]
customers = db.execute(
    "SELECT * FROM customers WHERE id = ANY(%s)", (customer_ids,)
).fetchall()
```

This is exactly the pattern an ORM's lazy-loaded relationship attribute
tends to trigger silently — looping over `order.customer.name` for 50 orders
issues 50 separate queries unless the relationship was eagerly loaded up
front. The API-shaped version of this same problem — a GraphQL resolver or a
REST endpoint fanning out into per-item requests — is covered separately in
[N+1 Queries in APIs](../web-development/n-plus-one-queries.md); the root
cause and fix are the same idea, just one layer up the stack.

## Common Anti-Patterns

**`SELECT *`** pulls every column, including ones the query doesn't need —
this defeats covering/index-only scans (the index can't cover columns it
doesn't know are needed) and wastes network bandwidth on wide tables with
large text/blob columns nobody asked for.

**Missing indexes on foreign keys.** Postgres does *not* automatically index
foreign key columns (MySQL/InnoDB does, for the FK itself). An unindexed FK
column means every join or lookup on it is a sequential scan, and — because
of how row-level locking interacts with FK constraint checks — it can also
make deletes on the referenced table slower and more lock-contentious than
necessary.

```sql
-- Postgres: this FK gets no automatic index — add one explicitly
ALTER TABLE orders ADD COLUMN customer_id INTEGER REFERENCES customers(id);
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

**Functions applied to indexed columns.** Wrapping an indexed column in a
function in the `WHERE` clause usually prevents the index from being used at
all, because the index stores the raw column value, not the function's
output.

```sql
-- Can't use a plain index on created_at — the function runs on every row
SELECT * FROM orders WHERE DATE(created_at) = '2026-08-15';

-- Rewritten as a range — uses a plain index on created_at
SELECT * FROM orders
WHERE created_at >= '2026-08-15' AND created_at < '2026-08-16';

-- Or: index the expression itself (Postgres)
CREATE INDEX idx_orders_created_date ON orders (DATE(created_at));
```

**Unbounded `OFFSET` pagination.** `LIMIT 20 OFFSET 100000` still has to
count through (and discard) 100,000 rows before returning the next 20 — cost
grows linearly with how deep into the result set you page, which is exactly
where pagination gets used most (infinite scroll on a large, popular
dataset).

```sql
-- Gets slower and slower as offset grows
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;

-- Keyset (cursor) pagination — constant cost regardless of "how deep"
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;
```

Keyset pagination trades away "jump to page N" for "here's the next page
after this cursor," which is the right trade for most feeds and infinite
scroll UIs, and the wrong one for a UI that genuinely needs numbered page
links.

## Connection Pool Exhaustion Isn't a Query Problem

A symptom that looks exactly like "the database is slow" but has nothing to
do with query plans: every connection in the application's pool is checked
out (in use, or leaked and never returned), so new requests queue up
waiting for a connection instead of waiting on a query. From the outside —
request latency spiking — it's indistinguishable from a slow query until
you look at what's actually being waited on.

```text
TimeoutError: QueuePool limit of size 10 overflow 5 reached, connection
timed out, timeout 30
```

Common causes: pool size too small for actual concurrency, a code path that
opens a session/connection and never closes it (missing `try/finally` or
context manager), or — the ironic case — genuinely slow queries holding
connections checked out for longer than usual, which then *causes*
exhaustion as a secondary effect. The fix for a too-small pool is sizing it
correctly for real concurrency; the fix for a leak is finding and closing
the leaked connection, not adding more pool capacity to paper over it.

## Tooling

**`pg_stat_statements`** (Postgres) — an extension that tracks every
distinct query shape the server has executed, aggregated with total/mean
execution time and call count. The single best starting point for "what's
actually slow" on a running Postgres instance, since it doesn't require
guessing which query to `EXPLAIN` — it ranks them for you.

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Slow query log** (MySQL) — logs any query exceeding a configured
threshold, for offline analysis (`mysqldumpslow`, `pt-query-digest`).

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5;  -- log anything over 500ms
```

Both tools answer the same question — "which query is actually costing the
most time in aggregate" — which is a different (and usually more useful)
question than "is this one query I'm staring at slow," since the biggest
win is often a moderately-slow query called 10,000 times an hour, not the
rare 3-second outlier.

## Summary

- `EXPLAIN ANALYZE` shows real timings against the plan; watch for Seq Scan
  on large tables, and a large estimated-vs-actual row mismatch (often a
  stale-statistics problem, fixed with `ANALYZE`).
- Indexes help in proportion to selectivity; composite index column order
  is left-to-right only; covering indexes avoid touching the table at all.
- N+1 at the database level is one query for a list plus N queries for
  related rows — fix with a join or a single `WHERE id IN (...)`; see
  [N+1 Queries in APIs](../web-development/n-plus-one-queries.md) for the
  same problem one layer up.
- Common anti-patterns: `SELECT *`, unindexed foreign keys, functions
  wrapping indexed columns, and `OFFSET`-based pagination that gets slower
  the deeper a user pages.
- Connection pool exhaustion presents as "the database is slow" but is
  actually an application-side resource problem, not a query problem.
- `pg_stat_statements` (Postgres) and the slow query log (MySQL) find the
  actually-expensive query in aggregate, rather than guessing.

## Related Articles

- [ACID](acid.md) — isolation levels affect how much locking a query does
  under concurrency, which shows up as latency too.
- [Locking, Deadlocks, and Concurrency](locking-deadlocks-concurrency.md) —
  long-running queries widen the window for lock contention.
- [Redis as a Caching Layer](redis-caching.md) — caching a hot read is
  often a cheaper fix than optimizing the query further.
- [SQL vs. NoSQL](sql-vs-nosql-postgres-mysql-mongo.md) — an unindexed
  foreign key is a relational-specific trap; MongoDB's `$lookup` has its
  own, different performance characteristics.
