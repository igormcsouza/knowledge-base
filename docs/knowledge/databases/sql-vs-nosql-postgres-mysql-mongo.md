---
tags:

- databases
- postgresql
- mysql
- mongodb
- nosql
- fundamentals

---

# SQL vs. NoSQL: PostgreSQL, MySQL, and MongoDB

"SQL vs. NoSQL" is really two separate questions people conflate:
relational vs. document data modeling, and PostgreSQL vs. MySQL. This
article treats them separately — Postgres and MySQL are both relational
databases with more in common with each other than either has with MongoDB,
but they're not interchangeable either.

## Schema-on-Write vs. Schema-on-Read

PostgreSQL and MySQL are **schema-on-write**: the table's shape (columns,
types, constraints) is declared up front, and every row must conform to it
at insert time. The database rejects a write that doesn't fit the schema.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO users (email) VALUES ('not-an-email-but-passes-type-check');
-- fine, type-wise — SQL doesn't validate "is this a real email" for you,
-- but it will reject a NULL email, or a duplicate one, at write time.
```

MongoDB is **schema-on-read**: documents in a collection can have different
shapes, and it's the *application* — not the database — that decides what
shape it expects when it reads a document back.

```javascript
db.users.insertOne({ email: "a@example.com" });
db.users.insertOne({ email: "b@example.com", plan: "pro", betaFlags: ["x"] });
// Both are valid documents in the same collection — the second one just
// has fields the first doesn't.
```

!!! note "Schema-on-read doesn't mean schema-less"
    MongoDB documents still have an implicit schema — it just lives in
    application code and query patterns instead of being enforced by the
    database. `$jsonSchema` validation rules can bring back write-time
    enforcement when you want it; it's optional rather than the default.
    "Flexible" is a trade-off, not a free lunch — an inconsistent shape
    across documents is exactly what makes ad-hoc queries against old data
    error-prone later.

The practical trade-off: schema-on-write catches shape mistakes immediately,
at the cost of needing a migration for every shape change (see
[Zero-Downtime Migrations](zero-downtime-migrations.md)). Schema-on-read
makes adding a new field a non-event, at the cost of every reader needing to
handle "this field might not be there" defensively, forever.

## ACID and Transactions

PostgreSQL and MySQL (InnoDB) are fully ACID — see [ACID](acid.md) for the
four guarantees in depth — across arbitrarily many statements and tables in
a single transaction, and always have been.

MongoDB's story here has a real history that's worth knowing because a lot
of "Mongo isn't ACID" folklore is now out of date:

- Single-document writes have always been atomic in MongoDB, even for
  documents with nested arrays/subdocuments — one document is one atomic
  unit.
- **Multi-document ACID transactions** were added in **MongoDB 4.0** (2018)
  for replica sets, and extended to sharded clusters in **4.2** (2019).
  Before that, there was no way to atomically update two different documents
  (let alone two different collections) together — a classic workaround was
  to model the two things as one document specifically to get atomicity.

```javascript
const session = client.startSession();
try {
  session.startTransaction();
  await accounts.updateOne({ _id: "A" }, { $inc: { balance: -100 } }, { session });
  await accounts.updateOne({ _id: "B" }, { $inc: { balance: 100 } }, { session });
  await session.commitTransaction();
} catch (e) {
  await session.abortTransaction();
  throw e;
} finally {
  session.endSession();
}
```

!!! warning "The historical caveat still matters"
    A lot of production MongoDB schemas — and a lot of blog posts, Stack
    Overflow answers, and older codebases — were designed *around* the
    pre-4.0 lack of multi-document transactions, favoring heavy embedding
    specifically to keep related writes inside one atomic document. If
    you're reading or maintaining an older Mongo schema, that design
    decision may be a historical workaround rather than a deliberate
    "embedding is right here" choice — worth re-evaluating now that
    multi-document transactions exist, rather than assuming it was chosen
    for the reasons the current codebase would choose it today.

Multi-document transactions in MongoDB also carry a real performance cost
(they hold resources across all involved shards/documents for the
transaction's duration) that the docs are upfront about — reach for them
when you need cross-document atomicity, not as a reflexive default the way
`BEGIN`/`COMMIT` is in Postgres/MySQL.

## Modeling Relationships

**Postgres/MySQL** model relationships with foreign keys and `JOIN`s — data
is normalized (each fact stored once), and related data is stitched back
together at query time.

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id)
);

SELECT orders.id, users.email
FROM orders
JOIN users ON users.id = orders.user_id
WHERE orders.id = 9981;
```

The foreign key isn't just documentation — `REFERENCES users(id)` is
*enforced*: the database physically will not let you insert an order for a
`user_id` that doesn't exist, or delete a user that still has orders
(without an explicit `ON DELETE` policy). That enforcement is a large part
of what "relational integrity" buys you.

**MongoDB** has two modeling choices instead of one, and picking between
them is the central document-modeling decision:

- **Embedding** — store the related data *inside* the parent document.
- **Referencing** — store an ID and look the related document up separately
  (optionally joined via `$lookup`, covered below).

```javascript
// Embedding — order line items live inside the order document.
{
  _id: "order_9981",
  userId: "user_42",
  items: [
    { sku: "ABC", qty: 2, price: 1999 },
    { sku: "XYZ", qty: 1, price: 4999 }
  ]
}

// Referencing — comments are their own collection, referencing the post.
{ _id: "comment_1", postId: "post_500", body: "Nice article" }
```

**Embedding wins** when the child data is always read together with the
parent, has a bounded size, and doesn't need to be queried independently —
order line items, an address on a user profile, settings on an account. One
read gets everything; no join-equivalent needed.

**Referencing wins** when the child data grows unboundedly, is queried
independently of the parent, or is shared across multiple parents — comments
on a post (could be thousands, and you might page through them without
loading the post), a product referenced by many orders, a user referenced by
many sessions. Embedding an unbounded array inside a document risks hitting
MongoDB's 16 MB per-document size limit and makes every read of the parent
drag along data you often don't need.

!!! tip "Pro Tip"
    A useful rule of thumb: embed when the answer to "does this grow
    without bound, or need to be queried on its own?" is no for *both*.
    The moment either is yes, reference it instead — "a post has a few
    hundred tags" can stay embedded; "a post has comments" almost never
    should, once the product succeeds.

## Postgres vs. MySQL

Both are mature, ACID-compliant, open-source relational databases — the
differences are real but narrower than "SQL vs. NoSQL," and worth knowing
specifically because the two get treated as interchangeable more than they
should be.

**MVCC implementation.** Both use Multi-Version Concurrency Control to let
readers avoid blocking writers, but differently under the hood. Postgres
stores old row versions directly in the table (a new row version on every
update) and relies on `VACUUM` to reclaim dead tuples — skip vacuuming on a
write-heavy table and it bloats. MySQL's InnoDB keeps old versions in a
separate **undo log** and reconstructs older versions on demand for
long-running transactions — no table bloat from updates, but a very
long-running transaction can bloat the undo log instead.

**JSON support.** Postgres's `JSONB` is a first-class, indexable, binary
storage type — you can `GIN`-index into it and query nested fields nearly as
efficiently as a native column.

```sql
CREATE TABLE events (id SERIAL PRIMARY KEY, payload JSONB);
CREATE INDEX idx_events_payload ON events USING GIN (payload);

SELECT * FROM events WHERE payload @> '{"type": "signup"}';
```

MySQL's `JSON` type is also indexable, but only through **generated
(virtual) columns** — you extract a value into a real column and index that,
rather than indexing into the JSON structure directly.

```sql
ALTER TABLE events
  ADD COLUMN event_type VARCHAR(50)
  GENERATED ALWAYS AS (payload->>'$.type') STORED,
  ADD INDEX idx_event_type (event_type);
```

**Extensions.** Postgres's extension system is a major differentiator —
`PostGIS` (geospatial types and queries), `pgvector` (vector similarity
search for embeddings, increasingly relevant for AI/ML workloads), and
dozens of others turn Postgres into a specialized database without leaving
SQL or adding an entirely separate system. MySQL has no equivalent extension
mechanism; specialized workloads typically mean a separate purpose-built
database alongside it.

**Replication.** MySQL's replication (binlog-based, statement or row-based)
predates and is arguably simpler to reason about for classic
primary-replica setups. Postgres offers both physical (WAL-shipping,
byte-for-byte) and logical (row-change-based, selective) replication, which
is more flexible — logical replication in particular enables replicating a
subset of tables or into a different schema — but has historically had more
moving parts to operate correctly.

!!! note "Opinion"
    For a new project with no strong existing constraint, Postgres's richer
    type system, extension ecosystem, and stronger JSON story make it the
    default reach today over MySQL. That's a preference, not a rule — MySQL
    remains a completely reasonable, battle-tested choice, and an existing
    team's operational experience with one or the other often outweighs
    these feature differences.

## Complex Relationships

### Many-to-Many

Relational databases model many-to-many through a **join table** (a.k.a.
association/junction table) holding a foreign key to each side:

```sql
CREATE TABLE students (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE courses (id SERIAL PRIMARY KEY, title TEXT);

CREATE TABLE enrollments (
    student_id INTEGER REFERENCES students(id),
    course_id INTEGER REFERENCES courses(id),
    enrolled_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (student_id, course_id)
);

SELECT courses.title
FROM enrollments
JOIN courses ON courses.id = enrollments.course_id
WHERE enrollments.student_id = 42;
```

MongoDB has no equivalent single construct — the common approaches are
referencing an array of IDs on one (or both) sides, or a dedicated
"enrollment" collection mirroring the join-table shape above, depending on
whether the relationship itself carries data (like `enrolled_at`) worth
querying on its own.

### Self-Referential Hierarchies

Modeling a tree (categories, org charts, comment threads) inside a
relational table has three well-known approaches, each trading off query
simplicity against write simplicity:

**Adjacency list** — each row stores its direct parent's ID. Simple to
write, but reading an entire subtree needs either a recursive query or N
round trips.

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT,
    parent_id INTEGER REFERENCES categories(id)
);

-- Recursive CTE to fetch a whole subtree in one query (Postgres/MySQL 8+)
WITH RECURSIVE subtree AS (
    SELECT id, name, parent_id FROM categories WHERE id = 1
    UNION ALL
    SELECT c.id, c.name, c.parent_id
    FROM categories c
    JOIN subtree s ON c.parent_id = s.id
)
SELECT * FROM subtree;
```

**Closure table** — a separate table storing *every* ancestor-descendant
pair, not just direct parent-child, including a row's relationship to
itself.

```sql
CREATE TABLE category_paths (
    ancestor_id INTEGER REFERENCES categories(id),
    descendant_id INTEGER REFERENCES categories(id),
    depth INTEGER,
    PRIMARY KEY (ancestor_id, descendant_id)
);

-- All descendants of category 1, no recursion needed
SELECT c.* FROM categories c
JOIN category_paths cp ON cp.descendant_id = c.id
WHERE cp.ancestor_id = 1;
```

Reads are a single non-recursive join at any depth — the trade-off is write
cost: inserting a node means inserting one row per ancestor (depth of the
tree), and moving a subtree means rewriting many rows.

**Materialized path** — each row stores its full ancestry as a string.

```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT,
    path TEXT  -- e.g. '1/4/17/' for a node under 1 -> 4 -> 17
);

-- All descendants of category 1
SELECT * FROM categories WHERE path LIKE '1/%';
```

Cheap to read (a prefix match, indexable), reasonably cheap to write, but
the path itself needs rewriting for every descendant when a subtree moves,
and the string format is a self-imposed schema the database can't validate.

| Approach | Read subtree | Write a node | Move a subtree |
|---|---|---|---|
| Adjacency list | Recursive query | O(1) | O(1) |
| Closure table | Single join | O(depth) | O(size of moved subtree) |
| Materialized path | Prefix match | O(1) | O(size of moved subtree) |

MongoDB documents can express the same three patterns (a `parentId` field,
a `path` array/string field, or full embedding of children as nested
subdocuments for shallow, bounded trees) — the trade-offs are structurally
the same, since this is a data-modeling problem, not a SQL-specific one.

### `$lookup` Is Not a Join

MongoDB's aggregation pipeline can approximate a join:

```javascript
db.orders.aggregate([
  { $match: { _id: "order_9981" } },
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  }
]);
```

This produces the same *shape* of result a SQL join would, but it isn't the
same operation underneath, in two important ways:

- **No query planner join optimization.** A relational database's query
  planner chooses a join strategy (nested loop, hash join, merge join) based
  on table sizes, available indexes, and cost estimates — and can reorder
  multi-table joins for efficiency. `$lookup` is executed as specified: for
  each document from the first stage, look up matches in the second
  collection. There's no cost-based reordering across `$lookup` stages the
  way a SQL optimizer reorders joins.
- **No referential integrity enforcement.** A SQL foreign key guarantees
  `orders.user_id` always points at a real `users.id` row — the database
  rejects a write that would break that. `$lookup` has no equivalent
  constraint: `userId` on an order document pointing at a deleted or
  nonexistent user is a data-quality bug the application has to prevent and
  detect on its own, not something MongoDB will ever stop you from writing.

Treat `$lookup` as a query-time convenience for read patterns, not as a
substitute for the integrity guarantees a real foreign key provides — that
gap is exactly why the embedding-vs-referencing decision earlier matters so
much more in MongoDB than "should I normalize this" does in Postgres/MySQL.

## Summary

- Postgres/MySQL are schema-on-write (enforced shape, migration needed to
  change it); MongoDB is schema-on-read (flexible shape, enforcement is
  optional and application-owned).
- Postgres and MySQL have always supported full multi-statement ACID
  transactions; MongoDB gained multi-document transactions in 4.0 (2018) —
  older Mongo schemas often embed specifically to work around not having had
  that.
- Foreign keys + joins in relational databases are enforced by the database;
  MongoDB's embedding vs. referencing is a design decision the application
  makes and must maintain the integrity of.
- Embed when data is always read together and bounded in size; reference
  when it grows unboundedly or needs independent access.
- Postgres and MySQL differ meaningfully under the "just SQL" label: MVCC
  implementation, `JSONB` vs. generated-column JSON indexing, Postgres's
  extension ecosystem (`PostGIS`, `pgvector`), and replication flexibility.
- Many-to-many needs a join table in SQL, an array-of-IDs or a dedicated
  collection in Mongo; hierarchies have three classic implementations
  (adjacency list, closure table, materialized path) with the same
  read/write trade-offs in either kind of database.
- `$lookup` produces join-shaped results without a real join's query-planner
  optimization or referential-integrity enforcement.

## Related Articles

- [ACID](acid.md) — the transaction guarantees relational databases have
  always had, and MongoDB gained for multi-document writes in 4.0.
- [Locking, Deadlocks, and Concurrency](locking-deadlocks-concurrency.md) —
  concurrency control specific to relational databases' row/table locks.
- [Query Performance Bottlenecks](query-performance-bottlenecks.md) — why
  foreign keys without an index on the referencing column are a common
  relational-specific slow-query trap.
- [Zero-Downtime Migrations](zero-downtime-migrations.md) — the cost of
  schema-on-write's enforced shape: every change needs a migration.
