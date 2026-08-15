---
tags:

- databases
- sqlalchemy
- alembic
- migrations
- python

---

# SQLAlchemy 2.0 and Alembic Migrations

SQLAlchemy 2.0 changed how models and queries are written well enough that
code from the 1.x era looks meaningfully different, and Alembic is the
migration tool built to work against it. This article covers the 2.0 style
and the practical realities of managing schema changes with Alembic —
autogeneration's real limitations chief among them.

## SQLAlchemy 2.0 Declarative Style

The legacy style declared columns with `Column` and typed them via SQL type
objects passed as arguments:

```python
# Legacy (1.x) style
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String, nullable=False)
```

2.0's style uses `Mapped[]` type annotations and `mapped_column()`, so the
Python type annotation *is* the model's type information — readable by both
SQLAlchemy and any type checker (mypy, pyright) at once, instead of the type
living only in a runtime `Column(...)` call SQLAlchemy has to inspect.

```python
from typing import Optional
from sqlalchemy import String, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)
    bio: Mapped[Optional[str]] = mapped_column(default=None)  # nullable, inferred

    orders: Mapped[list["Order"]] = relationship(back_populates="user")

class Order(Base):
    __tablename__ = "orders"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    total_cents: Mapped[int]

    user: Mapped["User"] = relationship(back_populates="orders")
```

`Mapped[Optional[str]]` (or `Mapped[str | None]`) makes the column nullable
without an explicit `nullable=True` — the type annotation itself is read as
the source of truth for nullability, one less thing that can drift out of
sync with reality.

### `select()` Replaces `Query`

The legacy `session.query(User)` API is still supported but no longer the
recommended path. 2.0-style code builds a `select()` construct — the same
one used for Core (non-ORM) queries — and executes it through the session:

```python
from sqlalchemy import select

# Legacy
users = session.query(User).filter(User.email == "a@example.com").all()

# 2.0 style
stmt = select(User).where(User.email == "a@example.com")
users = session.execute(stmt).scalars().all()

# A join, 2.0 style
stmt = (
    select(Order)
    .join(User)
    .where(User.email == "a@example.com")
)
orders = session.execute(stmt).scalars().all()
```

Unifying Core and ORM queries under one `select()` construct means the same
mental model (and the same query-building helpers) applies whether you're
querying mapped ORM objects or raw table rows — the two APIs had diverged
more than that in 1.x.

## Alembic Autogeneration and Its Real Limitations

Alembic compares the current database schema against the SQLAlchemy models
and generates a migration script for the difference:

```bash
alembic revision --autogenerate -m "add bio column to users"
```

```python
"""add bio column to users

Revision ID: a1b2c3d4e5f6
"""
from alembic import op
import sqlalchemy as sa

def upgrade() -> None:
    op.add_column("users", sa.Column("bio", sa.String(), nullable=True))

def downgrade() -> None:
    op.drop_column("users", "bio")
```

This works well for the common case (a new column, a new table, a new
index) but autogeneration is explicitly *not* a full schema diff — several
categories of change either go undetected or get detected as the wrong
operation:

- **Column type changes** are not detected by default (`compare_type` must
  be enabled in `env.py`, and even then Alembic's type comparison is
  conservative — some dialect-specific type changes still need a manual
  review).
- **Renames look like drop + add.** Alembic has no way to know that
  `user_name` becoming `username` was a rename rather than "delete this
  column, create an unrelated one" — it generates `drop_column` +
  `add_column`, which silently **discards existing data** in that column if
  applied as generated. A rename must be hand-edited into `op.alter_column`.

  ```python
  # What autogenerate produces for a rename — data-destroying if run as-is
  def upgrade() -> None:
      op.drop_column("users", "user_name")
      op.add_column("users", sa.Column("username", sa.String(), nullable=True))

  # What you actually want
  def upgrade() -> None:
      op.alter_column("users", "user_name", new_column_name="username")
  ```

- **Check constraints** are not autodetected in most database dialects
  (constraint introspection support varies by backend) — added, changed, or
  removed `CHECK` constraints typically need to be written by hand.
- Table/column **comments**, some kinds of **server-side defaults**, and
  most **data changes** (as opposed to schema changes) are outside what
  autogeneration looks at at all.

!!! warning "Always read the generated migration before running it"
    Autogeneration is a draft, not a verdict. Read every generated script
    for exactly the traps above — a rename that dropped data is the classic
    incident here, and it's entirely avoidable by reading the diff before
    running `alembic upgrade head` against anything that matters.

## Writing Explicit `upgrade()` / `downgrade()`

For anything autogeneration doesn't handle well — renames, data
backfills, constraint changes — write the operations directly using
Alembic's `op` API:

```python
def upgrade() -> None:
    op.alter_column("users", "user_name", new_column_name="username")
    op.create_check_constraint(
        "ck_orders_total_nonnegative", "orders", "total_cents >= 0"
    )

def downgrade() -> None:
    op.drop_constraint("ck_orders_total_nonnegative", "orders", type_="check")
    op.alter_column("users", "username", new_column_name="user_name")
```

`downgrade()` should always be the literal inverse of `upgrade()` — it's
what makes `alembic downgrade -1` a reliable escape hatch during a bad
deploy, and it's cheap to write correctly at the same time as `upgrade()`
while the change is fresh in mind, versus reconstructing it later under
pressure.

## Migration Branching and Merging

Alembic migrations form a linked list — each revision records its
`down_revision` (the parent it was generated against). When two people
branch off the same revision and each generate a migration independently,
Alembic ends up with two **heads** instead of one:

```text
alembic heads
a1b2c3d4e5f6 (head)
f9e8d7c6b5a4 (head)
```

`alembic upgrade head` fails (ambiguous — which head?) until the branches
are merged with a merge revision, which has both heads as parents and no
operations of its own:

```bash
alembic merge -m "merge heads" a1b2c3d4e5f6 f9e8d7c6b5a4
```

```python
"""merge heads"""
down_revision = ("a1b2c3d4e5f6", "f9e8d7c6b5a4")

def upgrade() -> None:
    pass

def downgrade() -> None:
    pass
```

!!! tip "Pro Tip"
    This is a normal, expected consequence of concurrent development, not a
    sign something went wrong — the fix is `alembic merge`, not manually
    editing `down_revision` pointers by hand. Catch it early by running
    `alembic heads` (or letting CI run it) before merging a branch that adds
    a migration, so the merge-heads step happens deliberately rather than as
    a surprise in someone else's next migration attempt.

## Data Migrations vs. Schema Migrations

A **schema migration** changes structure (`CREATE TABLE`, `ADD COLUMN`,
`CREATE INDEX`) — typically fast, and the kind Alembic's `op` API is built
around. A **data migration** changes the *values* in existing rows —
backfilling a new column, reshaping data after a schema change, fixing bad
historical data.

```python
def upgrade() -> None:
    op.add_column("users", sa.Column("plan", sa.String(), nullable=True))

    # Data migration inline in the same revision — risky on a large table
    connection = op.get_bind()
    connection.execute(sa.text("UPDATE users SET plan = 'free' WHERE plan IS NULL"))

    op.alter_column("users", "plan", nullable=False)
```

Mixing a bulk data change into a schema migration like this is risky for a
few concrete reasons:

- A single `UPDATE` touching every row of a large table takes a lock (or at
  minimum, sustained write load) for the duration — on a multi-million-row
  table that can be minutes of a migration blocking other writes, run
  synchronously as part of a deploy that's supposed to be quick.
- If the data migration fails partway (a constraint violation on row
  500,000 of a million), the whole migration transaction rolls back, but
  now you're debugging a data problem under the pressure of a stuck deploy.
- Alembic migrations typically run inside a single transaction per
  migration (configurable, but that's the default expectation) — a data
  migration doesn't get the batching/backoff/progress-checkpointing that a
  dedicated backfill script would.

!!! note "Prefer a separate backfill step"
    For anything beyond a trivial, fast `UPDATE`, keep the schema migration
    (add the nullable column) and the data migration (backfill it in
    batches, outside of Alembic's single-transaction model) as separate
    steps — see
    [Zero-Downtime Migrations](zero-downtime-migrations.md#backfilling-large-tables-in-batches)
    for the batching approach itself.

## Testing Migrations Before Production

A migration is code, and like any code it can be wrong — the safest habit
is running it against a realistic copy of the schema before it touches
production:

```bash
# Apply every migration from scratch against a throwaway database —
# catches migrations that only work when run against dev's already-hand-
# patched schema, and won't work from a clean history.
alembic upgrade head

# Then verify downgrade actually works, not just upgrade
alembic downgrade -1
alembic upgrade head
```

A CI step that spins up a fresh database, runs `alembic upgrade head` from
an empty schema, and (ideally) round-trips one step of `downgrade` +
`upgrade` catches a large fraction of migration bugs — missing imports, a
typo in a column name, an autogenerated rename that would have silently
dropped data — before they reach a database anyone depends on.

## Summary

- SQLAlchemy 2.0's `Mapped[]` / `mapped_column()` style makes the type
  annotation the source of truth for column type and nullability, and
  unifies ORM and Core queries under one `select()` construct.
- Alembic autogeneration handles new tables/columns/indexes well, but
  misses type changes (without `compare_type`), turns renames into a
  data-destroying drop+add, and generally skips check constraints — always
  read the generated script.
- Write `upgrade()`/`downgrade()` explicitly for anything autogeneration
  gets wrong, and keep `downgrade()` a true inverse of `upgrade()`.
- Multiple concurrent migrations create multiple Alembic heads — a normal
  outcome, resolved with `alembic merge`, not manual `down_revision` edits.
- Keep bulk data migrations separate from schema migrations — a large
  in-migration `UPDATE` risks long locks and doesn't get batching/backoff.
- Run every migration against a fresh schema in CI before it reaches
  production.

## Related Articles

- [Zero-Downtime Migrations](zero-downtime-migrations.md) — the expand/
  contract pattern for schema changes that can't be a single atomic step
  without breaking currently-running application code.
- [Locking, Deadlocks, and Concurrency](locking-deadlocks-concurrency.md) —
  why a long-running data migration inside a schema migration risks lock
  contention with normal application traffic.
- [ACID](acid.md) — migrations typically run inside a transaction, which is
  exactly what makes a failed migration roll back cleanly instead of
  leaving the schema half-changed.
