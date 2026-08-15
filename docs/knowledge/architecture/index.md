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
- [Dependency Injection with Protocols](dependency-injection-protocols.md) —
  structural typing with `typing.Protocol` instead of ABC inheritance, and
  why that avoids hierarchy hell.
- [Factory Pattern in Python](factory-pattern.md) — from a plain function to
  registries to `@classmethod` alternative constructors.

## Contributing

Learned a new architecture pattern or design principle worth keeping? Add it
here as its own file. See [Contributing](../../contributing.md) for the
how-to.
