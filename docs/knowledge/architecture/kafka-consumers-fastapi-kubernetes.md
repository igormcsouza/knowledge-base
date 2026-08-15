---
tags:

- architecture
- kafka
- fastapi
- kubernetes
- redis
- event-driven

---

# Kafka Consumers Behind a FastAPI API on Kubernetes

Notes from working through how to wire a Kafka consumer into a FastAPI
service that's deployed on Kubernetes — specifically, the questions that
actually matter once you go past "consume some messages": where consumed
state lives, what happens when you scale to multiple pods, and when the
consumer earns its own deployment. These are general principles, not a
specific project's setup.

## Don't Query Kafka Directly From an Endpoint

An HTTP endpoint has no business polling Kafka for an answer. The consumer's
job is to continuously turn a stream of events into queryable **state**; the
endpoint's job is to read that state.

```text
External service → Kafka → Consumer → application state/storage → HTTP endpoint
```

This is the same shape as
[Event-Driven Architecture](event-driven-architecture.md)'s producer/consumer
split, just made concrete for "a web API needs an answer that arrives
asynchronously via events."

## A Consumer Is a Long-Lived Loop, Not a Per-Message Task

A Kafka consumer is one continuously running process (or async task):
subscribe once, then loop — wait for a message, process it, wait for the
next one. You never spin up a new consumer per incoming message; that's the
opposite of how Kafka's client libraries and consumer groups are designed to
work.

## Consumer Groups Set the Scaling Ceiling

Kafka distributes a topic's **partitions** across the consumers sharing a
`group.id`. That's the whole mechanism for horizontally scaling consumption
— more consumers in the group, more partitions processed in parallel.

The consequence worth internalizing: **the number of partitions is the
ceiling on useful parallelism**. Four partitions and ten consumers in the
same group still means only four consumers are ever actively pulling
messages — the rest sit idle. Partition count has to be planned with the
target consumer scale in mind, not added as an afterthought.

## Async Consumption Can Coexist With FastAPI's Event Loop

The critical distinction, covered in more depth in
[FastAPI's Event Loop](../web-development/fastapi-event-loop.md), is between:

```python
await consume_forever()          # blocks startup forever — never returns
asyncio.create_task(consume_forever())  # runs alongside everything else
```

`await`-ing an infinite consumer loop directly in startup means startup never
completes, because that call never returns. Instead, launch it as a
background task — its own `await`s on Kafka I/O yield control back to the
loop between messages, so the same event loop keeps serving HTTP requests.
FastAPI's **lifespan** handler is the natural place to start the task on
startup and cancel it cleanly on shutdown.

## Keep the Consumer as Separate Logic, Even Co-Located

Even when the consumer runs inside the same process as the API (a reasonable
starting point — see below), keep it as its own module with its own
responsibility, not tangled into route handlers:

- the app entrypoint wires everything together and starts both the API and
  the consumer task,
- the consumer module owns the consume loop,
- a separate state/store module owns how consumed data is stored and read.

This mirrors the [Service Layer](ddd-service-layer.md) idea — logic separated
by responsibility, not by "how it happens to be deployed right now." The
payoff shows up later: moving the consumer into its own Kubernetes Deployment
becomes a deployment change, not a rewrite.

## Correlation IDs Tie the Pieces Together

When an HTTP request kicks off an external, asynchronous operation, and the
results come back later as a stream of Kafka messages, a **correlation ID**
generated at request time is what lets a later message get matched back to
the right piece of state. Structure state as a map keyed by that ID, not a
list — `correlation_id → [message, message, ...]` gives O(1) lookup for "what
do we know about this operation so far," which is exactly what the read-side
endpoint needs.

## In-Memory State Is Pod-Local

This is the Kubernetes concept that changes everything else in this list.
State held in a Python dict inside one pod's process exists **only in that
pod's memory**. A second pod handling a later request for the same
correlation ID has no way to see it — different process, different machine,
zero shared memory. A single-replica deployment never notices this; the
moment you scale to more than one replica, "works on my machine" silently
becomes "works on whichever pod happened to handle both requests."

## Sticky Sessions Are a Partial Fix, With a Real Cost

Sticky routing keeps requests for the same session/key going to the same
pod, which makes pod-local state work again — as long as that pod stays
alive. If it crashes or gets rescheduled, whatever state it was holding is
simply gone. That tradeoff can be acceptable for short-lived, low-stakes
state; it's not a substitute for shared state when durability matters even a
little.

## Redis for Shared, Short-Lived State

The general fix for "which pod owns this data" is to stop storing it on any
particular pod:

```text
Pod A ─┐
Pod B ─┼→ Redis
Pod C ─┘
```

Any pod can read or write the shared state, so it no longer matters which
pod handled which request. Redis specifically suits **short-lived** state
well because of TTLs — state for a finished or abandoned operation expires
on its own, without a separate cleanup job.

## Kafka, Redis, and a Database Solve Different Problems

Introducing Kafka doesn't automatically mean you need a relational database
too — the three tools cover different concerns and can be adopted
independently:

- **Kafka** — event transport between services.
- **Redis** — shared, short-lived, frequently-read state (correlation-ID
  lookups, in-flight operation tracking).
- **A database** (PostgreSQL, etc.) — durable application data that needs to
  outlive a TTL or survive being queried long after the fact.

If the data in question only matters for the lifetime of one short
operation, Redis is often the better fit than reaching for a full
relational database by default.

## Horizontal Scaling Is What Creates the State-Ownership Problem

Vertical scaling (bigger pod: more CPU/RAM) never raises the "which pod has
my state" question, because there's only ever one pod. Horizontal scaling
(more, smaller pods) is what Kubernetes makes natural and cheap — and it's
exactly what turns pod-local in-memory state from a convenience into a bug.
Any design that starts with in-memory state should have an explicit answer
for what happens the day it needs a second replica.

## Design Consumers for At-Least-Once Delivery

Kafka's normal delivery guarantee is **at-least-once** — a message can be
redelivered, particularly around consumer crashes and offset-commit timing.
Consumers should assume duplicates will happen and be written so that
processing the same message twice doesn't produce two copies of the same
logical result. A sequence number or message ID on each message is what
makes deduplication possible.

This is the same idempotency requirement covered generally in
[Idempotency](../databases/idempotency.md) — Kafka's delivery model is one of
the concrete cases that principle exists for.

## Kafka Offsets Aren't "Delete on Read"

Unlike a traditional queue, consuming a Kafka message doesn't remove it.
Messages stay in the topic per the configured retention policy; what moves
is the **consumer's offset** — its recorded position in the log, tracked per
consumer group. Multiple consumer groups can read the same topic
independently, each at its own offset, because reading never removes
anything. This is also what makes replay possible, which is one of Kafka's
defining differences from RabbitMQ/SQS, covered in
[Event-Driven Architecture](event-driven-architecture.md).

## Independent, Idempotent Messages Scale Well

Messages that can be processed without needing to coordinate with other
messages are the ideal shape for Kafka consumption — no ordering
dependencies between them means partitions and additional consumers can
absorb more load later without a redesign. Independence and idempotency
compound: independent messages parallelize cleanly, idempotent processing
makes redelivery safe, and together they're what makes "just add more
consumers" actually work.

## When to Split the Consumer Into Its Own Deployment

Running the consumer inside the same pod as the API is a reasonable starting
point when API traffic and Kafka consumption need roughly the same scaling
characteristics. Split it into its own Kubernetes Deployment once those
characteristics diverge — for example:

- the API needs many more (or fewer) replicas than the consumer does,
- consumer processing becomes CPU/memory intensive independent of API load,
- consumer lag needs to scale on its own signal, not API request volume,
- the two need different resource limits or independent monitoring/
  autoscaling,
- a consumer crash shouldn't be able to take down API availability.

"Production systems usually separate these" isn't itself a reason — the
reason is a genuine divergence in one of the above. Keeping the consumer as
its own module (see above) from the start is what makes this split cheap
whenever it does become the right call.

## The Shape to Keep in Mind

```text
HTTP request → generate correlation_id → kick off external operation → return/track correlation_id

(independently, continuously)
Kafka → async consumer → receive message → read correlation_id → update state

HTTP GET → read state by correlation_id → return accumulated status/results
```

The state store starts as an in-memory map for a single-replica deployment,
moves behind Redis the moment more than one replica needs to see the same
state, and the consumer moves into its own Deployment once its scaling needs
diverge from the API's. The core principle underneath all of it: keep event
consumption, the HTTP API, and state storage **conceptually separate**, even
while they're deployed together — that's what keeps each of those scaling
moves cheap instead of a rewrite.

## Summary

- Endpoints read state a consumer maintains; they don't query Kafka directly.
- A consumer is one long-lived loop, not one instance per message.
- Consumer group size beyond the partition count buys nothing.
- Background the consumer with `asyncio.create_task` (started/stopped via
  FastAPI's lifespan), never `await` it directly in startup.
- Separate consumer / state / API logic by responsibility from day one, even
  when deployed together.
- Correlation IDs plus a map (not a list) is how async, event-driven results
  get matched back to the request that started them.
- In-memory state is pod-local — the central fact that everything about
  scaling and shared state follows from.
- Redis solves cross-pod shared state for short-lived data via TTLs;
  reach for a database only when the data needs to outlive that.
- Design for at-least-once delivery: idempotent, independently-processable
  messages are what make Kafka consumption scale safely.
- Split the consumer into its own deployment when its scaling/lifecycle
  needs genuinely diverge from the API's — not by default.

## Related Articles

- [Event-Driven Architecture](event-driven-architecture.md) — the general
  Kafka/RabbitMQ/SQS comparison this builds on.
- [FastAPI Event Loop](../web-development/fastapi-event-loop.md) — why
  `asyncio.create_task` instead of `await` matters for a background consumer.
- [DDD & the Service Layer](ddd-service-layer.md) — the same "separate by
  responsibility" instinct applied to a consumer instead of a router.
- [Idempotency](../databases/idempotency.md) — the general principle behind
  designing for Kafka's at-least-once delivery.
