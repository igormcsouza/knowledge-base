---
tags:

- architecture
- event-driven
- messaging
- kafka
- rabbitmq
- sqs

---

# Event-Driven Architecture: Kafka vs. SQS vs. RabbitMQ

Event-Driven Architecture (EDA) decouples services by having **producers**
emit events (or messages) without knowing who — if anyone — is listening, and
**consumers** react to those events asynchronously. Instead of Service A
calling Service B directly (a tight, synchronous coupling), Service A
publishes "OrderPlaced" and moves on; Service B (and C, and D) pick it up
whenever they're ready.

## Core Concepts

- **Event vs. command** — a command ("ChargeCard") tells one specific
  receiver to do something and expects it to happen; an event ("OrderPlaced")
  announces something that already happened, and zero or more listeners may
  react. EDA is centered on events.
- **Message broker** — the middleman that receives messages from producers
  and delivers them to consumers, decoupling the two in time and space.
- **Pub/sub vs. point-to-point** — in pub/sub, a message can be delivered to
  *every* subscriber; in point-to-point (queues), a message is delivered to
  exactly *one* consumer among a competing group.
- **Delivery guarantees** — at-most-once (may drop messages, never
  duplicates), at-least-once (never drops, may duplicate — consumers must be
  idempotent), exactly-once (hardest to achieve, usually built on top of
  at-least-once + deduplication).
- **Ordering** — whether messages are guaranteed to be processed in the order
  they were produced (usually only guaranteed within some partition/shard,
  not globally).

## Kafka

Kafka is a **distributed commit log**, not a traditional message broker.
Producers append events to **topics**, which are split into **partitions**
for parallelism; each partition is an ordered, immutable, append-only log.

- **Retention & replay** — messages aren't deleted on consumption. They stay
  for a configured retention period (or forever, with compaction), so new
  consumers can replay history from the beginning, and multiple independent
  consumer groups can each read the same topic at their own pace.
- **Consumer groups** — consumers in the same group split partitions between
  them (each partition read by one consumer in the group); different groups
  each get their own full copy of the stream.
- **Ordering** — guaranteed *within* a partition (messages with the same key
  land on the same partition), not across the whole topic.
- **Throughput** — built for very high throughput, sequential disk writes and
  zero-copy reads make it fast at scale.
- **Use cases** — event sourcing, stream processing, activity/analytics
  pipelines, log aggregation, anywhere replay or multiple independent readers
  of the same stream matters.

## RabbitMQ

RabbitMQ is a traditional **message broker** implementing AMQP: producers
publish to **exchanges**, which route messages to **queues** based on
**bindings** and routing keys/patterns. Consumers pull from queues.

- **Smart broker, simple consumer** — routing logic (direct, topic, fanout,
  headers exchanges) lives in the broker's configuration, so complex
  fan-out/routing rules don't need to be built in application code.
- **No replay by default** — once a message is consumed and acknowledged,
  it's gone from the queue (unless you explicitly mirror it elsewhere).
- **Ordering** — generally FIFO per queue, though redelivery/retries can
  disturb strict ordering.
- **Throughput** — lower than Kafka at extreme scale, but very solid for
  typical workloads, and simpler to reason about for task-queue-style usage.
- **Use cases** — background job/task queues, RPC-style request/reply,
  complex routing between services, anywhere you want flexible routing more
  than raw throughput or replay.

## SQS

Amazon SQS is a fully managed, simple **queue service** — no servers to run,
scales automatically.

- **Standard vs. FIFO queues** — Standard queues offer at-least-once delivery
  with best-effort ordering and near-unlimited throughput; FIFO queues
  guarantee exactly-once processing and strict ordering per message group, at
  lower throughput.
- **Visibility timeout** — instead of an explicit ack/nack, a consumer that
  reads a message makes it invisible to others for a configured window; if it
  isn't deleted (acknowledged) before the timeout expires, it becomes visible
  again for another consumer to pick up. This is how SQS handles crashed/slow
  consumers.
- **No native pub/sub** — SQS is point-to-point. Fan-out to multiple
  consumers is done by pairing it with **SNS** (SNS topic → multiple SQS
  queues subscribed).
- **Use cases** — decoupling AWS services/Lambdas, worker queues, anywhere
  you want zero operational overhead and are already on AWS.

## Comparison

| | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| Model | Distributed log | Message broker (AMQP) | Managed queue |
| Retention/replay | Yes, configurable | No (consumed = gone) | No |
| Ordering | Per-partition | Per-queue (mostly) | FIFO queues only |
| Routing complexity | Simple (topic-based) | Rich (exchanges/bindings) | Simple, needs SNS for fan-out |
| Throughput | Very high | Moderate-high | High (Standard), lower (FIFO) |
| Ops overhead | You run/tune the cluster (or managed service) | You run/tune the broker | Fully managed |
| Multiple independent readers | Yes (consumer groups) | Needs extra queues/bindings | Needs SNS fan-out |

## When to Pick What

- **Kafka** — event streaming, analytics pipelines, event sourcing, or when
  several independent consumers each need their own full view of the event
  history.
- **RabbitMQ** — classic task/work queues, RPC patterns, or when routing
  logic is genuinely complex and you want the broker to handle it.
- **SQS** — already on AWS, want minimal ops, and the use case is
  straightforward point-to-point decoupling (e.g., feeding work to a Lambda
  or worker fleet).

!!! note "Personal take"
    This is a starting mental model based on what I've read/learned, not a
    substitute for benchmarking your actual workload — throughput,
    durability, and latency needs vary enough between projects that "which
    one is fastest" isn't a fixed answer.

## Related Articles

- [DDD & the Service Layer](ddd-service-layer.md) — services that publish/
  consume events still benefit from keeping business logic out of the
  transport layer.
- [ACID](../databases/acid.md) — message brokers typically trade strong
  consistency for availability/throughput, the opposite end of the spectrum
  from ACID databases.
- [Kafka Consumers Behind a FastAPI API on Kubernetes](kafka-consumers-fastapi-kubernetes.md)
  — applying Kafka's consumer-group model to a real FastAPI/Kubernetes
  deployment.
- [Amazon SQS](../devops-tools/aws/sqs.md) — the practical mechanics of
  actually using the SQS side of this comparison.
