---
title: Architecture
tags:

- architecture
- overview

---

# Architecture

Software architecture patterns and system design notes: how to structure code
within a service, and how services communicate with each other.

## Articles

- [DDD & the Service Layer](ddd-service-layer.md) — structuring FastAPI
  projects by domain, and keeping business logic in a service layer instead
  of the router.
- [Event-Driven Architecture: Kafka vs. SQS vs. RabbitMQ](event-driven-architecture.md)
  — event-driven basics and how the three major messaging systems compare.
- [Kafka Consumers Behind a FastAPI API on Kubernetes](kafka-consumers-fastapi-kubernetes.md)
  — consumer groups, background tasks vs. blocking startup, pod-local state,
  and when to split the consumer into its own deployment.

## Patterns

Code-level design patterns — how you structure and construct objects within
a service — live in their own [Patterns](patterns/index.md) subsection,
separate from the broader architectural styles above:

- [Dependency Injection with Protocols](patterns/dependency-injection-protocols.md)
  — structural typing with `typing.Protocol` instead of ABC inheritance, and
  why that avoids hierarchy hell.
- [Factory Pattern in Python](patterns/factory-pattern.md) — from a plain
  function to registries to `@classmethod` alternative constructors.

## Contributing

Learned a new architecture pattern or design principle worth keeping? Add it
here as its own file. See [Contributing](../../contributing.md) for the
how-to.
