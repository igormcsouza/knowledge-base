---
tags:

- architecture
- ddd
- fastapi
- backend
- design-patterns

---

# DDD & the Service Layer (FastAPI Best Practices Style)

Domain-Driven Design (DDD) is a full methodology for modeling complex business
domains (ubiquitous language, bounded contexts, aggregates, entities, value
objects, repositories). Most backend projects don't need the *full* toolbox —
what they borrow, and what actually pays off day to day, is a lighter set of
ideas: organize code by **domain/feature**, not by technical layer, and pull
business logic out of your HTTP layer into a **service layer**. This is the
approach popularized by the
[fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices)
repository, and it's a pragmatic subset of DDD, not textbook DDD.

## Structure by Domain, Not by File Type

A common but scaling-poorly layout groups files by *type*:

```text
app/
  routers/
    users.py
    posts.py
  models/
    users.py
    posts.py
  schemas/
    users.py
    posts.py
```

The problem: working on one feature means jumping between three or four
distant folders, and the "users" and "posts" concerns are interleaved
everywhere. The fastapi-best-practices layout instead groups by **domain**:

```text
src/
  auth/
    router.py       # HTTP endpoints — routing only
    schemas.py       # Pydantic request/response models
    models.py         # DB models
    service.py         # business logic lives here
    dependencies.py     # FastAPI Depends() for this domain
    exceptions.py         # domain-specific exceptions
    constants.py            # domain-specific constants/enums
  posts/
    router.py
    schemas.py
    models.py
    service.py
    dependencies.py
    exceptions.py
  config.py
  database.py
  main.py
```

Everything related to `auth` lives in `auth/`. Adding a feature to posts means
working almost entirely inside `posts/`. This is the practical payoff of
"bounded contexts" without needing to formally model them.

## The Service Layer

The core rule: **routers stay thin**. A router's job is HTTP concerns only —
parsing the request, calling a service function, mapping the result to a
response and status code. All business logic — validation beyond what
Pydantic gives you for free, orchestration, calls to other services, side
effects — lives in `service.py`.

```python
# posts/router.py — thin, HTTP concerns only
from fastapi import APIRouter, Depends

from posts import service
from posts.schemas import PostCreate, PostRead

router = APIRouter(prefix="/posts")


@router.post("/", response_model=PostRead, status_code=201)
async def create_post(payload: PostCreate, user=Depends(get_current_user)):
    return await service.create_post(author_id=user.id, data=payload)
```

```python
# posts/service.py — business logic lives here
from posts.exceptions import DuplicateTitleError
from posts.models import Post
from posts.schemas import PostCreate


async def create_post(author_id: int, data: PostCreate) -> Post:
    if await _title_taken(author_id, data.title):
        raise DuplicateTitleError(data.title)

    post = Post(author_id=author_id, **data.model_dump())
    await post.save()
    await _notify_followers(author_id, post)
    return post
```

Notice the router doesn't know *how* a post gets created — it just calls
`service.create_post(...)`. That function is what actually contains the
domain rules (no duplicate titles, notify followers, etc.).

## Why It's Worth Doing

- **Testability** — `service.create_post` can be unit-tested directly with
  plain function calls and mocked dependencies, without spinning up a test
  client or going through HTTP/`Depends()` resolution.
- **Reusability** — the same service function can be called from a router, a
  background task, a CLI command, or a scheduled job. Logic that lives inside
  a router can only ever be triggered by an HTTP request.
- **Readability** — a router file becomes a table of contents for the
  domain's endpoints; the actual behavior is one clear place to read
  (`service.py`) instead of scattered across route handlers.
- **Change isolation** — swapping the ORM, adding caching, or changing how
  notifications are sent touches `service.py`, not every router that happens
  to trigger that behavior.

!!! note "This isn't full DDD"
    There's no repository interfaces, no aggregate roots enforcing
    invariants, no explicit bounded-context boundaries with anti-corruption
    layers. It's DDD's organizing *spirit* — domain-first structure,
    business logic separated from delivery mechanism — applied pragmatically.
    That's a reasonable, deliberate trade-off for most web APIs; reach for
    full DDD patterns only when the domain complexity actually demands them
    (e.g. multiple teams, many invariants, several data sources per
    aggregate).

## Summary

- Organize code by domain/feature folder, not by technical file type.
- Keep routers thin: HTTP parsing in, response mapping out.
- Put business logic in `service.py` — it's the reusable, testable core.
- This is deliberately a lighter, practical slice of DDD, not the full
  methodology.

## Related Articles

- [Event-Driven Architecture](event-driven-architecture.md) — another way
  services stay decoupled, this time across process boundaries.
- [FastAPI Event Loop](../web-development/fastapi-event-loop.md) — how the
  router/service code you write actually gets executed.
- [Dependency Injection with Protocols](dependency-injection-protocols.md) —
  typing a service's dependencies without forcing them into a shared class
  hierarchy.
- [Factory Pattern in Python](factory-pattern.md) — a natural place for the
  construction logic a service's dependencies need.
