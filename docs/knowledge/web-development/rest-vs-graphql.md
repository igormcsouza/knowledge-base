---
tags:

- web-development
- rest
- graphql
- api-design

---

# REST vs. GraphQL

Both are ways to expose an API over HTTP, but they organize the contract
around different units: REST organizes around **resources and verbs**,
GraphQL organizes around a **single endpoint and a query language**. Neither
is strictly better — they solve different shapes of problem, and picking
wrong shows up later as either chronic over-fetching or a client team that's
constantly blocked on backend changes.

## REST: Resources and Verbs

REST models an API as a set of **resources**, each with a URL, manipulated
through HTTP's existing verbs (`GET`, `POST`, `PUT`/`PATCH`, `DELETE`) and
status codes:

```text
GET    /users/42
GET    /users/42/posts
POST   /users/42/posts
GET    /posts/17/comments
```

Each endpoint returns a fixed shape. This maps naturally onto HTTP's own
machinery: a `GET` on a URL is cacheable by browsers, CDNs, and reverse
proxies purely based on the URL and cache headers (`Cache-Control`, `ETag`),
with no application-specific caching logic required. That "the URL *is* the
cache key" property is REST's biggest structural advantage, and it's easy to
undervalue until you've had to build the equivalent by hand.

## GraphQL: A Single Endpoint and a Query Language

GraphQL exposes one endpoint (conventionally `POST /graphql`) and lets the
client describe exactly what data it wants in a query, structured against a
schema the server defines:

```graphql
query {
  user(id: 42) {
    name
    posts {
      title
      commentCount
    }
  }
}
```

The server returns exactly that shape — no more, no less. There are three
operation kinds:

- **Query** — read data, like the example above.
- **Mutation** — write data (`mutation { createPost(title: "Hi") { id } }`),
  explicitly separated from reads so effects are never hidden inside a query.
- **Subscription** — a long-lived operation the server pushes updates to over
  time (typically over WebSockets), for real-time data. See
  [WebSockets](websockets.md) for the transport it's usually built on.

## The Problem GraphQL Directly Solves

REST's fixed response shapes create two chronic problems, covered in more
depth in [API Pitfalls: Over-, Under-Fetching & N+1](api-pitfalls-n-plus-one.md):

- **Over-fetching** — a `GET /users/42` returns the full user object even
  when the client only needed the display name.
- **Under-fetching** — rendering one screen (a user, their posts, and each
  post's comment count) takes three separate round-trips because no single
  REST endpoint returns exactly that combination.

GraphQL fixes both by construction: the client's query *is* the response
shape, across as many nested relationships as the schema allows, in one
round-trip.

## Side-by-Side: The Same Use Case

Rendering a user's profile page with their five most recent posts and each
post's comment count.

=== "REST"

    ```text
    GET /users/42
    GET /users/42/posts?limit=5
    GET /posts/17/comments/count
    GET /posts/18/comments/count
    GET /posts/19/comments/count
    GET /posts/20/comments/count
    GET /posts/21/comments/count
    ```

    Six round-trips, and the user/post responses likely carry fields the
    profile page never renders (bio, email, post body, timestamps).

=== "GraphQL"

    ```graphql
    query {
      user(id: 42) {
        name
        avatarUrl
        posts(limit: 5) {
          title
          commentCount
        }
      }
    }
    ```

    One request, one round-trip, exactly the fields the page needs.

## GraphQL's Own N+1 Problem

GraphQL doesn't eliminate the N+1 problem — it moves it server-side. Each
field in a query can have its own **resolver**, and a naive resolver for
`posts { commentCount }` fetches comment counts one post at a time:

```python
# naive resolver — one DB query per post, N+1 all over again
async def resolve_comment_count(post, info):
    return await db.count_comments(post_id=post.id)
```

For 5 posts that's 5 separate queries fired inside the same GraphQL request
— invisible in the client's request count, but very visible in the database.

**DataLoader** (originated at Facebook alongside GraphQL itself) is the fix:
it batches all the individual key lookups requested during a single tick of
the event loop into one call, then fans the results back out:

```python
from aiodataloader import DataLoader


class CommentCountLoader(DataLoader):
    async def batch_load_fn(self, post_ids: list[int]) -> list[int]:
        counts = await db.count_comments_batch(post_ids)  # one query for all
        return [counts.get(pid, 0) for pid in post_ids]


async def resolve_comment_count(post, info):
    return await info.context["comment_count_loader"].load(post.id)
```

Every resolver call still *looks* like it's fetching one post's count, but
`DataLoader` coalesces the five individual `.load()` calls into a single
`batch_load_fn` invocation with all five IDs — one query instead of five.

!!! tip
    A DataLoader instance is scoped to a single request (created fresh per
    request in the GraphQL context), never shared or cached across requests
    — otherwise one user's batched data could leak into another's response.

## Schema-First Development

GraphQL's schema is written in **SDL** (Schema Definition Language) and is
the actual contract — strongly typed, and both client and server validate
against it:

```graphql
type Post {
  id: ID!
  title: String!
  commentCount: Int!
}

type User {
  id: ID!
  name: String!
  posts(limit: Int): [Post!]!
}

type Query {
  user(id: ID!): User
}
```

`!` marks non-null. Clients can introspect this schema (most GraphQL tooling
— GraphiQL, code generators — relies on introspection), and a query
referencing a field that doesn't exist fails validation before it ever runs,
rather than surfacing as a runtime `KeyError` or a silently missing field.

## Trade-offs

**Caching is harder.** REST's cacheability comes from the URL being the
cache key; GraphQL's single endpoint and per-query shape breaks that model
outright — two different `POST /graphql` bodies are indistinguishable to a
generic HTTP cache. GraphQL clients (Apollo, Relay) work around this with
normalized client-side caches keyed by object ID instead of URL, but that's
application-level caching logic, not something you get for free from
infrastructure.

**Query complexity needs policing.** Because a client can request arbitrarily
deep nested relationships (`user { posts { comments { author { posts { ... }
} } } }`), an unrestricted GraphQL API is an easy target for accidentally or
deliberately expensive queries. Production servers need **depth limiting**
and/or **query cost analysis** (assigning a "cost" to each field and
rejecting queries over a budget) — REST's fixed endpoints don't have this
problem because the server, not the client, decides how much work each
request does.

**REST's simplicity and tooling maturity.** REST needs nothing beyond HTTP
itself — curl, browser dev tools, any HTTP client understands it immediately.
Caching, rate limiting, and monitoring at the infrastructure layer (CDNs,
API gateways, reverse proxies) all assume URL-based semantics and work with
REST out of the box. GraphQL needs a purpose-built server layer and client
library to get equivalent ergonomics.

!!! note
    Many production systems use both: REST for simple CRUD and anything that
    benefits from HTTP caching, GraphQL for screens that aggregate many
    related resources. Treat it as a tool choice per use case, not a
    religious commitment to one style for an entire API surface.

## Summary

- REST models resources + verbs; its URLs are naturally cacheable by
  standard HTTP infrastructure.
- GraphQL models a single endpoint + client-specified query shape; it
  directly solves over-fetching and under-fetching in one round-trip.
- GraphQL pushes N+1 into resolvers — DataLoader-style batching is the fix,
  scoped per request.
- Schema-first SDL gives GraphQL strong typing and query validation before
  execution.
- Trade-offs run the other way too: GraphQL caching is harder (not
  URL-based), and unrestricted query depth/cost needs active limiting; REST
  keeps HTTP's caching and tooling for free.

## Related Articles

- [API Pitfalls: Over-, Under-Fetching & N+1](api-pitfalls-n-plus-one.md) —
  the REST-side version of the problems GraphQL solves by design.
- [API Lifecycle & Design](api-lifecycle-design.md) — versioning and
  governance concerns that apply to REST APIs specifically.
- [WebSockets](websockets.md) — the transport GraphQL subscriptions are
  typically built on.
- [FastAPI Event Loop](fastapi-event-loop.md) — where resolver code (sync vs.
  async) actually executes if you build a GraphQL server on FastAPI.
