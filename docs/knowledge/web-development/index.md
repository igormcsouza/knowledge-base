---
title: Web Development
tags:

- web-development
- overview

---

# Web Development

Frameworks, patterns, and practical notes for building and shipping web applications.

## Articles

- [Flask Framework](flask-framework.md) — basic setup, project structure, CLI commands,
  and packaging a Flask app as a desktop application.
- [FastAPI Event Loop](fastapi-event-loop.md) — where sync and async routes
  actually run, and how to deal with background tasks.
- [TypeScript Fundamentals](typescript-fundamentals.md) — typing properly:
  interfaces, generics, narrowing, and utility types.
- [REST vs. GraphQL](rest-vs-graphql.md) — resource/verb design vs. a
  single-endpoint query language, over-/under-fetching, and GraphQL's own
  N+1 problem.
- [API Lifecycle & Design](api-lifecycle-design.md) — design-first vs.
  code-first, versioning strategies, deprecation policy, and governance as
  an API surface grows.
- [WebSockets](websockets.md) — the HTTP upgrade handshake, full-duplex
  connections, SSE/polling alternatives, and the pub/sub fan-out problem
  when scaling across server instances.
- [OAuth2 & JWT Authentication](oauth2-jwt-authentication.md) — grant types,
  OIDC, JWT structure and signature verification, and access/refresh token
  handling.
- [Pydantic Advanced: Validators, Aliases & Settings](pydantic-advanced-validators-aliases.md)
  — custom validators, field aliasing, computed fields, and settings
  management with `pydantic-settings`.
- [API Pitfalls: Over-, Under-Fetching & N+1](api-pitfalls-n-plus-one.md) —
  the N+1 query problem and fixes for over-/under-fetching in REST APIs.
- [API Optimization & Resilience](api-optimization-resilience.md) — cold
  starts, exponential backoff with jitter, circuit breakers, and
  idempotency keys at the API-client level.

## Contributing

Building something with a new framework or pattern worth documenting? Add it here as its
own file. See [Contributing](../../contributing.md) for the how-to.
