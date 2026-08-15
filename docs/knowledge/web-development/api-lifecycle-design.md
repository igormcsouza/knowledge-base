---
tags:

- web-development
- api-design
- rest
- versioning

---

# API Lifecycle & Design

An API outlives the code that first implements it — other teams, other
companies, or your own future self write clients against it, and those
clients keep running long after you've moved on to the next feature. Most of
the pain in API maintenance comes from decisions made (or skipped) before
the first endpoint ships: how the contract is defined, how it changes over
time, and how those changes get communicated.

## Design-First vs. Code-First

**Code-first**: write the implementation, let the framework generate the
API contract from it. FastAPI does this well — the OpenAPI spec is derived
automatically from your route signatures and Pydantic models:

```python
@router.post("/orders", response_model=OrderRead, status_code=201)
async def create_order(payload: OrderCreate) -> OrderRead:
    ...
```

FastAPI generates `/openapi.json` and the interactive docs from this
directly — fast to iterate, and the docs can never drift out of sync with
the code, because they *are* the code.

**Design-first**: write the OpenAPI spec *before* any implementation exists,
treat it as the contract, and generate server stubs/client SDKs from it (or
implement by hand against it). This front-loads design review — API
consumers, or a separate frontend team, can react to the shape of the
contract before a single line of business logic is written, and multiple
teams can build against the same spec in parallel once it's agreed.

```yaml
# orders.openapi.yaml — written and reviewed before any code
paths:
  /orders:
    post:
      requestBody:
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/OrderCreate"
      responses:
        "201":
          description: Order created
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/OrderRead"
```

!!! note
    Code-first is the pragmatic default for a single-team internal service —
    it minimizes ceremony and the framework keeps the spec honest. Design-first
    earns its overhead once multiple independent consumers (external partners,
    separate frontend/mobile teams, other companies) need to agree on and
    build against a contract before an implementation is ready.

## Versioning Strategies

Three common approaches, each with a real trade-off:

**URL path versioning** (`/v1/orders`, `/v2/orders`) — the most explicit and
the easiest for clients and infrastructure (caches, gateways, logs) to
reason about, since the version is visible in every request without
inspecting headers. Downside: it implies whole-resource versioning even when
only one field changed, and running `/v1` and `/v2` side by side long-term
means maintaining two full code paths.

**Header-based versioning** (`Accept: application/vnd.myapi.v2+json`, or a
custom `API-Version` header) — keeps URLs stable (nicer for caching and
bookmarking a single canonical resource URL), and allows finer-grained
negotiation. Downside: invisible in server logs and browser address bars,
easy for clients to forget to set, and harder to test casually with a
browser or curl without remembering the header.

**No versioning + additive-only changes** — never introduce a `/v2` at all;
instead, commit to only ever making backward-compatible changes (see below)
to a single stable contract. Simplest to operate and easiest on clients, but
it's a discipline commitment, not a technique — it only works if the team
is genuinely willing to say no to (or find another way to ship) any change
that would otherwise be breaking.

!!! tip "Pro Tip"
    Pick one strategy per API and apply it consistently — mixing versioning
    styles across endpoints of the same API is far more confusing to
    consumers than any single choice's individual downsides.

## Backward-Compatible vs. Breaking Changes

The core distinction that every other decision here depends on:

| Change | Compatible? | Why |
|---|---|---|
| Add a new optional field to a response | Safe | Existing clients ignore fields they don't know about |
| Add a new optional request field with a default | Safe | Existing clients that omit it get prior behavior |
| Add a new endpoint | Safe | Doesn't affect existing consumers at all |
| Remove a field | **Breaking** | Any client reading that field now fails or silently loses data |
| Rename a field | **Breaking** | Functionally a remove + add — the old name simply disappears |
| Change a field's type (`string` → `int`) | **Breaking** | Client-side parsing/deserialization can throw or misbehave |
| Make an optional request field required | **Breaking** | Existing clients that omit it now get rejected |
| Change validation to be stricter | **Breaking** | Previously-valid requests can start failing |
| Change error response shape | **Breaking** | Clients that parse error bodies (increasingly common) break |

!!! warning
    "Just widening a type" is deceptively risky too — changing an `int` ID
    to a `string` ID is technically additive-feeling but breaks any
    statically-typed client. Treat *any* type change as breaking, not just
    narrowing ones.

## Deprecation Policy

A breaking change is unavoidable eventually — a deprecation policy is what
keeps it from being an unannounced outage for consumers.

- **Announce before removing.** Give consumers a deprecation window (weeks
  to months, depending on how many external consumers exist and how quickly
  they can realistically react) between "this is deprecated" and "this is
  gone."
- **Signal it in the response itself**, not just in changelog docs, so
  automated tooling and less-attentive consumers still get the signal:

  ```http
  HTTP/1.1 200 OK
  Deprecation: true
  Sunset: Sat, 31 Oct 2026 00:00:00 GMT
  Link: <https://api.example.com/docs/migration-v2>; rel="deprecation"
  ```

  The `Sunset` header (RFC 8594) gives an exact date the endpoint will stop
  working — it's machine-readable, so client tooling can programmatically
  flag "you're calling something that's going away."
- **Keep the deprecated version fully functional** until the sunset date —
  a deprecation warning followed by silently degraded behavior defeats the
  purpose of announcing it at all.
- **Communicate through every channel consumers actually watch** —
  changelog, email/notification if you have consumer contacts, and the
  response headers above as the fallback for anyone who missed the rest.

## API Governance Basics

As an API surface grows past a handful of endpoints, **consistency** across
them starts to matter more than any individual endpoint's cleverness — a
consumer building against endpoint #40 should be able to guess its shape
correctly from having used endpoints #1 through #39.

Concretely, agree on and enforce (via review, or ideally a shared linter
against the OpenAPI spec) conventions for:

- **Naming** — plural nouns for collections (`/orders`, not `/order`),
  consistent casing (`snake_case` or `camelCase`, not a mix depending on who
  wrote which endpoint).
- **Pagination** — one pagination style (cursor-based or offset-based) with
  one parameter naming scheme, everywhere. A client that has to remember
  "this endpoint uses `page`/`per_page` but that one uses `cursor`/`limit`"
  is a client that ships bugs.
- **Error shape** — one consistent error body structure across every
  endpoint, so client error-handling code can be written once:

  ```json
  {
    "error": {
      "code": "validation_error",
      "message": "amount must be positive",
      "field": "amount"
    }
  }
  ```

- **Filtering/sorting conventions** — the same query-param pattern
  (`?sort=-created_at`, `?status=active`) across every list endpoint that
  supports it.

The payoff compounds with API surface size: inconsistency across 5 endpoints
is a minor annoyance; across 200 endpoints built by a dozen different
engineers over several years, it's the difference between an API developers
enjoy integrating and one every consumer needs a wrapper library to survive.

## Summary

- Code-first (FastAPI-style, spec generated from code) suits fast-moving
  internal services; design-first (spec written and reviewed before
  implementation) suits APIs with multiple independent consumers.
- Versioning strategies (URL path, header-based, additive-only/no
  versioning) trade explicitness against long-term maintenance burden —
  pick one per API and stay consistent.
- Adding optional fields/endpoints is safe; removing, renaming, retyping a
  field, or tightening validation is breaking — know the difference before
  shipping a change.
- Deprecate loudly and machine-readably (`Deprecation`/`Sunset` headers)
  with a real window before removal, and never let a "deprecated" endpoint
  silently misbehave before its sunset date.
- Naming, pagination, and error-shape consistency matter more as the API
  surface grows — it's what lets consumers generalize correctly instead of
  special-casing every endpoint.

## Related Articles

- [REST vs. GraphQL](rest-vs-graphql.md) — a prior design decision that
  shapes how much of this versioning/governance discussion even applies.
- [API Pitfalls: Over-, Under-Fetching & N+1](api-pitfalls-n-plus-one.md) —
  design mistakes that show up after an API ships, not before.
- [OAuth2 & JWT Authentication](oauth2-jwt-authentication.md) — the access
  control layer that sits alongside these contract-design concerns.
