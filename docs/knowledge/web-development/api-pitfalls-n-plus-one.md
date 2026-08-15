---
tags:

- web-development
- api-design
- performance
- rest

---

# API Pitfalls: Over-, Under-Fetching & N+1

Three problems keep showing up in REST API design, and they're related: the
**N+1 query problem** (an endpoint quietly fires far more database queries
than it looks like it should), **over-fetching** (the response carries more
than the client needed), and **under-fetching** (the client needs several
round-trips to assemble one screen). All three are consequences of the same
root cause — a REST endpoint returns a fixed shape, decided by the server,
that doesn't necessarily match what any given client actually wants.

## The N+1 Query Problem

This is the most common of the three, and the easiest to introduce by
accident. An endpoint fetches a list of N parent records, then — usually via
an ORM's lazy-loaded relationship accessed inside a loop — fires one
additional query *per parent* to fetch related data. N parents means N+1
total queries instead of the 2 it should take.

```python
# looks innocent, is an N+1 bug
@router.get("/authors")
async def list_authors(session: AsyncSession = Depends(get_session)):
    authors = await session.scalars(select(Author))  # 1 query
    return [
        {"name": a.name, "book_count": len(a.books)}  # N queries — one per author,
        for a in authors                              # each access lazy-loads a.books
    ]
```

For 50 authors that's 51 queries to render one list. It works fine in local
testing with 3 seed rows and falls over in staging/production once the table
has real data — the classic profile of an N+1 bug.

### The Fix: Eager Loading

Tell the ORM up front to fetch the related rows in the *same* query (a JOIN)
or in one *batched* follow-up query, instead of one query per row accessed
later:

```python
from sqlalchemy.orm import selectinload, joinedload

# joinedload: one query, single JOIN — good for to-one or small to-many
stmt = select(Author).options(joinedload(Author.profile))

# selectinload: two queries total (authors, then all their books in one
# WHERE author_id IN (...) query) — usually the better choice for
# to-many relationships, since a JOIN would duplicate the author row
# once per book
stmt = select(Author).options(selectinload(Author.books))

authors = await session.scalars(stmt)
```

`selectinload` is generally the right default for one-to-many relationships:
it costs exactly 2 queries regardless of N, versus `joinedload`'s single
query that returns a duplicated, wider row set for every related child.

### The Fix: Batching / DataLoader-Style

The same principle applies outside the ORM, e.g. calling another internal
service per item instead of a DB relationship. Collect the IDs, make one
batched call:

```python
# N+1 over HTTP — one call per author to a separate service
book_counts = [await books_service.count_for_author(a.id) for a in authors]

# batched — one call for all author IDs
author_ids = [a.id for a in authors]
book_counts = await books_service.count_for_authors(author_ids)  # returns a dict
```

This is exactly the shape of the fix [GraphQL uses for its own N+1
problem](rest-vs-graphql.md#graphqls-own-n1-problem) via DataLoader — collect
the keys requested during one unit of work, issue a single batched fetch,
then fan the results back out to each caller.

!!! tip
    N+1 bugs are easy to miss in code review because the code *looks*
    correct — `a.books` reads like a plain attribute access. Catch them with
    query-count assertions in tests (most ORM test setups can assert "this
    call site issued exactly 2 queries") or by watching the SQL log for a
    suspiciously repeating query shape under a loop.

## Over-Fetching

A fixed-shape REST response returns everything the resource has, even when
the client rendering a list view only needs `id`, `name`, and a thumbnail
URL — not the full object graph (bio, settings, timestamps, nested
relations). This wastes bandwidth and server work on every request, and gets
worse as the resource grows fields over the API's lifetime.

**GraphQL's answer** is structural: the client's query *is* the response
shape, so there's nothing to over-fetch by design — see
[REST vs. GraphQL](rest-vs-graphql.md).

**REST's partial fix** is a **sparse fieldset** query parameter, letting the
client opt into a subset of fields on an otherwise fixed endpoint:

```text
GET /users/42?fields=id,name,avatar_url
```

```python
@router.get("/users/{user_id}")
async def get_user(user_id: int, fields: str | None = None):
    user = await get_user_by_id(user_id)
    data = user.model_dump()
    if fields:
        wanted = set(fields.split(","))
        data = {k: v for k, v in data.items() if k in wanted}
    return data
```

This is a partial fix, not a full one — it doesn't extend to filtering
*nested* relationships the way a GraphQL query naturally does, and every
endpoint needs to opt in individually.

## Under-Fetching

The opposite problem: assembling one screen needs data from several
resources, and no single REST endpoint returns that combination, so the
client makes several sequential (or parallel, but still separate) requests.
A profile page needing a user, their recent posts, and each post's comment
count is the standard example — see the [REST vs. GraphQL side-by-side
example](rest-vs-graphql.md#side-by-side-the-same-use-case) for the full
request breakdown.

**REST's partial fixes:**

- **Resource expansion** — an `?expand=` param that inlines a related
  resource into the response instead of making the client fetch it
  separately:

  ```text
  GET /users/42?expand=posts
  ```

  ```json
  {
    "id": 42,
    "name": "Ada",
    "posts": [
      {"id": 17, "title": "Hello"},
      {"id": 18, "title": "World"}
    ]
  }
  ```

- **BFF (Backend-for-Frontend) / aggregation endpoints** — a purpose-built
  endpoint that exists specifically to assemble one screen's data in one
  call, even if it doesn't map cleanly onto a single REST resource:

  ```text
  GET /profile-page/42
  ```

  returning exactly the user + posts + comment-count shape that screen
  needs, computed server-side (ideally *with* eager loading, not by
  internally re-triggering N+1).

Both are workarounds for the same underlying limitation GraphQL solves
structurally — they trade a cleaner resource model for fewer round-trips,
which is a reasonable trade when a specific screen's fetch pattern is known
and stable.

## Summary

- N+1: an endpoint that loops over N parents and fires one query/call per
  parent for related data — fix with eager loading (`selectinload`/
  `joinedload`) or batching (DataLoader-style).
- Over-fetching: fixed REST responses carry unused fields — sparse
  fieldsets are REST's partial fix; GraphQL solves it structurally.
- Under-fetching: one screen needs several REST round-trips — resource
  expansion (`?expand=`) and BFF/aggregation endpoints are REST's partial
  fixes; GraphQL solves it structurally.
- All three trace back to the same root cause: a server-fixed response
  shape that doesn't necessarily match what any given client needs.

## Related Articles

- [REST vs. GraphQL](rest-vs-graphql.md) — the structural alternative that
  solves over-/under-fetching by letting the client specify the shape.
- [API Lifecycle & Design](api-lifecycle-design.md) — designing an API
  surface so these pitfalls are easier to avoid as it grows.
- [API Optimization & Resilience](api-optimization-resilience.md) — the
  performance/reliability concerns that come after the N+1/fetching shape
  itself is fixed.
