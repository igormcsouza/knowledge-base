---
tags:

- architecture
- interview
- backend
- senior-engineer
- system-design

---

# Senior Software Engineer Architecture Interview Cheat Sheet

A practical cheat sheet for senior software engineer / backend interviews,
focused on recurring architecture and distributed-systems problems. It's
built around the technologies I actually work with, and optimized for
interview recall: given a problem, identify the pattern, explain the usual
solution, and discuss the trade-offs.

!!! warning "How to use this"
    Do not memorize a single architecture as *the* answer to a problem. The
    goal is to explain **why** a pattern fits the requirements, what
    assumptions it depends on, and what trade-offs it introduces. Senior-level
    answers show judgment, not just technology recall.

## Interview Response Template

For almost any architecture question, structure the answer as:

1. Clarify functional and non-functional requirements.
1. Estimate scale: RPS, data size, latency, availability, growth.
1. Start with the simplest viable architecture.
1. Identify the bottleneck or failure mode.
1. Pick the technology/pattern that addresses it.
1. Explain data consistency and failure semantics.
1. Explain scaling and backpressure.
1. Add security and observability.
1. State trade-offs and what you would avoid overengineering.

## Problem → Pattern Table

| Problem / Scenario | Technologies that solve it | How we usually solve it | Trade-offs / Things to mention |
|---|---|---|---|
| High traffic / CPU or request load | Load balancer, Docker/Kubernetes, AWS ECS/EKS/Lambda, FastAPI | Keep services stateless and scale horizontally; use load balancing; scale vertically only when appropriate | Horizontal scaling improves capacity/resilience but increases infrastructure and coordination complexity |
| Long-running request / blocking work | Kafka, RabbitMQ, SQS, Celery, Lambda | Accept the request quickly, enqueue work, process asynchronously, persist result/status, expose polling/webhook if needed | Eventual consistency; retries and duplicate processing must be handled |
| Synchronous vs asynchronous service communication | REST/HTTP, Kafka, RabbitMQ | REST when caller needs an immediate response; messaging when decoupling, buffering, retries, fan-out or resilience matter | Messaging adds operational complexity, eventual consistency and harder debugging |
| Message delivery reliability | Kafka, RabbitMQ, SQS | Durable queues/topics, acknowledgements, retries, dead-letter queues, consumer idempotency | At-least-once delivery is common; duplicates are expected and must be safe |
| Duplicate message processing | PostgreSQL, Redis, DynamoDB | Use idempotency keys/event IDs, deduplication store, unique constraints and atomic state transitions | Dedup state has storage/retention cost; exactly-once is difficult end-to-end |
| DB transaction + event publishing consistency | PostgreSQL + Kafka/RabbitMQ | Transactional Outbox: write business change and outbox event in one DB transaction, publish asynchronously, mark event as sent | Extra table/process; consumers still need idempotency |
| Distributed transaction across services | Saga, Kafka/RabbitMQ, compensating actions | Split workflow into local transactions and coordinate through events/orchestration; add compensating transactions for failures | More complex reasoning; temporary inconsistency is expected |
| Database bottleneck | PostgreSQL, Redis, read replicas | Measure first with EXPLAIN/EXPLAIN ANALYZE; indexes, query optimization, pooling, caching, replicas, partitioning when justified | Indexes increase write/storage cost; replicas introduce replication lag; caches create invalidation problems |
| Hot / repeated reads | Redis, PostgreSQL | Cache frequently accessed data with TTL/invalidation; use cache-aside where appropriate | Stale data, cache stampede, memory cost and invalidation complexity |
| Cache stampede | Redis, locks, request coalescing | TTL jitter, distributed lock/single-flight, background refresh | Locking adds latency and failure modes |
| Rate limiting | Redis, API Gateway, load balancer | Token bucket/leaky bucket/fixed or sliding window; enforce per client/API key/IP as appropriate | Distributed counters and strict global limits are harder; Redis becomes part of the control path |
| External API is slow/unreliable | HTTP client, async, retries, circuit breaker, queue | Timeouts, bounded retries with exponential backoff + jitter, circuit breaker, fallback/async processing | Retries can amplify outages; idempotency is critical for non-read operations |
| Third-party API rate limit | Queue, Redis, Celery/Kafka/SQS | Buffer work in a queue and control consumer concurrency/rate; retry after server-provided limits | Higher latency; requires durable backlog and monitoring |
| Service unavailable / cascading failure | Circuit breaker, timeout, bulkhead, queue | Fail fast on unhealthy dependencies, isolate resources, queue when work can be deferred | Can cause partial functionality and more state-machine complexity |
| Zero-downtime deployments | Kubernetes/ECS, load balancer, CI/CD | Rolling/blue-green/canary deployments, readiness checks, backward-compatible DB changes | More infrastructure and rollout complexity |
| Backward-compatible database migration | PostgreSQL, Alembic | Expand-and-contract: add new schema, deploy compatible code, backfill, switch reads/writes, remove old schema later | Multiple application versions must coexist temporarily |
| Event ordering | Kafka partitions, message keys | Partition by entity key so related events go to the same partition; preserve sequence metadata when necessary | Ordering is usually scoped, not global; partitioning can create hot partitions |
| Consumer cannot keep up | Kafka/RabbitMQ/SQS, Kubernetes | Scale consumers, increase partition/worker capacity, optimize handler, apply backpressure | More consumers can increase DB/API contention; partition limits may constrain Kafka throughput |
| Background job retries | Celery, SQS, Kafka, RabbitMQ | Exponential backoff, retry limits, dead-letter queue, observability, idempotent handlers | Some failures should not be retried; retry storms are dangerous |
| Database concurrency / lost updates | PostgreSQL | Transactions, row locks, optimistic concurrency/version column, appropriate isolation level | Stronger isolation/locking can reduce throughput and increase contention |
| Authentication / authorization | OAuth2/OIDC, JWT, API Gateway | Authenticate at edge/service boundary; authorize based on roles/scopes/claims; keep service-to-service identity explicit | JWT revocation/rotation and distributed authorization need careful design |
| Secrets/configuration | AWS Secrets Manager/SSM, environment variables, Kubernetes Secrets | Keep secrets out of source control; inject at runtime; rotate credentials | Secret management has operational overhead; careless logging can leak secrets |
| Observability | Datadog, CloudWatch, OpenTelemetry, structured logging | Instrument logs, metrics and traces; define SLIs/SLOs; correlate requests with trace/request IDs | Instrumentation has cost; excessive logs create noise and expense |
| Debugging distributed failures | OpenTelemetry, Datadog, correlation IDs, centralized logs | Follow one request/event across services using trace IDs, structured logs and metrics | Requires consistent instrumentation and propagation |
| Async Python service throughput | FastAPI, asyncio | Use async I/O for concurrent network-bound work; avoid blocking calls inside event loop; use worker processes for CPU-bound work | Async does not make CPU-bound Python code parallel; incorrect blocking code can stall the entire loop |
| CPU-bound Python workload | multiprocessing, worker processes, job queue | Move CPU-heavy work to separate processes/workers or specialized services | Higher memory/IPC overhead; deployment becomes more complex |
| Python thread vs asyncio | threading, asyncio | Threads can be useful for blocking I/O and legacy libraries; asyncio for high-concurrency cooperative I/O | Threads have overhead and synchronization concerns; asyncio requires async-compatible code |
| Clean architecture / maintainability | Python, FastAPI, dependency injection, repositories | Keep domain/application logic independent of frameworks and infrastructure; invert dependencies | Adds abstractions; overengineering is possible for small/simple services |
| Hexagonal architecture | Ports & adapters, Python | Define domain-facing ports and implement adapters for DB, HTTP, messaging, etc. | More indirection and boilerplate; useful when infrastructure changes or domain logic is complex |
| Testability | Pytest, mocks, integration tests, Testcontainers | Unit-test business logic, integration-test boundaries, use contract/E2E tests selectively; TDD where valuable | Too many mocks can test implementation details; broad suites can be slower |
| API consistency / pagination | FastAPI, PostgreSQL | Validate inputs, consistent errors, cursor/offset pagination, stable ordering | Cursor pagination is more complex but scales better for large/changing datasets |
| File processing at scale | S3, SQS/Kafka, Lambda/workers | Store object in S3, publish event/message, process asynchronously, persist status/result | Requires eventual consistency and lifecycle/cleanup strategy |
| Search / large read models | PostgreSQL, Elasticsearch/OpenSearch, Kafka | Keep source of truth in DB, build/search a read model asynchronously | Operational duplication and eventual consistency |

## Core Technologies to Emphasize

- Python / FastAPI / asyncio
- PostgreSQL / Redis
- Kafka / RabbitMQ / SQS
- AWS Lambda / API Gateway / ECS/EKS / S3 / DynamoDB / RDS
- Docker / Kubernetes
- Pytest / TDD
- OpenTelemetry / Datadog / CloudWatch
- Clean Architecture / Hexagonal Architecture / SOLID

## Key Interview Reminder

Do not memorize a single architecture as the answer. Explain why a pattern
fits the requirements, what assumptions it depends on, and what trade-offs
it introduces. Senior-level answers should show judgment rather than just
technology knowledge.

## Related Articles

- [Event-Driven Architecture: Kafka vs. SQS vs. RabbitMQ](event-driven-architecture.md)
  — deeper dive into the messaging row of the table above.
- [Understanding Consistency in Distributed Systems](consistency-in-distributed-systems.md)
  — the reasoning behind the consistency-related rows (duplicate processing,
  distributed transactions, database bottlenecks).
- [Kafka Consumers Behind a FastAPI API on Kubernetes](kafka-consumers-fastapi-kubernetes.md)
  — a concrete case of the "consumer cannot keep up" and "async Python
  service throughput" rows.
- [Dependency Inversion via Interfaces and Abstract Classes](dependency-inversion-abstract-classes.md)
  — the SOLID principle behind the clean/hexagonal architecture rows.
