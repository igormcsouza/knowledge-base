---
tags:

- architecture
- dependency-inversion
- solid
- python
- abstract-classes
- design-patterns

---

# Dependency Inversion via Interfaces and Abstract Classes

The **Dependency Inversion Principle** (the "D" in SOLID) is a design rule:
*high-level modules should not depend on low-level modules — both should
depend on abstractions.* It's a different thing from **Dependency
Injection** (the mechanical act of passing a collaborator in from outside)
even though the two are almost always used together — DIP is the
*architectural rule about which direction dependencies point*, DI is *how
you wire the resulting objects together at runtime*. This article covers
DIP itself and the classic way to express the "abstraction" it requires:
Python's `abc.ABC`.

## Inverting the Dependency Direction

Without DIP, a "high-level" module (business logic) typically depends
directly on a "low-level" module (an implementation detail like a specific
database driver or HTTP client):

```python
import psycopg2


class OrderService:
    def __init__(self) -> None:
        self._conn = psycopg2.connect("dbname=orders")  # concrete, low-level

    def place_order(self, order_id: str) -> None:
        with self._conn.cursor() as cur:
            cur.execute("INSERT INTO orders (id) VALUES (%s)", (order_id,))
```

`OrderService` — the thing that encodes the actual business rule "placing
an order" — is now coupled to `psycopg2` specifically. Swapping databases,
writing a unit test without a real Postgres instance, or mocking the write
all require reaching into `OrderService`'s internals. The dependency points
the "wrong" way: business logic depends on infrastructure.

DIP inverts this: both the high-level and low-level modules depend on an
**abstraction** that the high-level module defines, and the low-level
module implements:

```python
from abc import ABC, abstractmethod


class OrderRepository(ABC):
    @abstractmethod
    def save(self, order_id: str) -> None: ...


class PostgresOrderRepository(OrderRepository):
    def __init__(self, conn) -> None:
        self._conn = conn

    def save(self, order_id: str) -> None:
        with self._conn.cursor() as cur:
            cur.execute("INSERT INTO orders (id) VALUES (%s)", (order_id,))


class OrderService:
    def __init__(self, repository: OrderRepository) -> None:
        self._repository = repository  # depends on the abstraction, not psycopg2

    def place_order(self, order_id: str) -> None:
        self._repository.save(order_id)
```

`OrderService` now depends only on `OrderRepository` — an abstraction *it*
effectively owns the contract for, conceptually living alongside the
business logic rather than the infrastructure. `PostgresOrderRepository`
depends on that same abstraction too. Both point at the interface; neither
points at the other. That's the "inversion": the low-level detail now
depends on something the high-level policy defines, instead of the reverse.

## Why an Abstract Base Class Here

`ABC` is one valid way to express the abstraction DIP requires — and often
the more natural one specifically when you're modeling "this is a
*Repository*, a first-class concept in the domain, and here are its
required operations":

- `@abstractmethod` gives you an **enforcement mechanism**: instantiating a
  subclass that hasn't implemented every abstract method raises `TypeError`
  at construction time, not a mystifying `AttributeError` deep in a call
  stack the first time the missing method is used.
- The class hierarchy can also carry **real shared behavior** — a
  `BaseRepository` ABC can implement common logic (retry wrapping, logging
  around every write) once, with concrete subclasses inheriting it for
  free, on top of the parts they must implement themselves:

```python
class OrderRepository(ABC):
    @abstractmethod
    def save(self, order_id: str) -> None: ...

    def save_with_logging(self, order_id: str) -> None:
        print(f"saving order {order_id}")
        self.save(order_id)  # shared behavior, built on the abstract contract
```

- It documents intent unambiguously: a class that inherits from
  `OrderRepository` is *declaring itself* to be one, which is useful when
  "is a repository" is a meaningful domain concept you want visible in the
  class hierarchy, not just an implicit shape.

## ABC vs. `Protocol`: Two Ways to Satisfy DIP

This knowledge base already covers the other common way to define the
abstraction: `typing.Protocol`, in
[Dependency Injection with Protocols](dependency-injection-protocols.md).
Both are legitimate ways to satisfy DIP — the choice is about **nominal
vs. structural** typing, not about whether DIP is being followed:

| | `ABC` | `Protocol` |
|---|---|---|
| Subtyping | Nominal — must explicitly inherit | Structural — shape match is enough |
| Shared implementation | Yes — can hold real method bodies | No — pure contract, no shared code |
| Retrofits existing/3rd-party classes | No — they'd need to inherit | Yes — automatically, if the shape matches |
| Runtime enforcement | `TypeError` at instantiation if incomplete | Only with `@runtime_checkable`, and only a name check |
| Best fit | The abstraction is a real domain concept with shared logic | The abstraction is a pure capability/interface |

!!! tip
    Default to `Protocol` when you just need "anything with this method
    shape is acceptable here" (the common case for injected dependencies in
    tests and swappable adapters). Reach for `ABC` specifically when the
    abstraction itself is a meaningful, named domain concept that should
    carry shared behavior or enforce completeness at construction time —
    e.g. a `Repository` base class used across a dozen concrete
    repositories that all need the same retry/logging wrapper.

## DIP Is Not the Same as "Just Add an Interface"

A subtle failure mode: defining an ABC that mirrors one *specific*
low-level implementation's method signatures ("interface that just wraps
`psycopg2`'s cursor API") doesn't actually invert anything — the
abstraction is still shaped by the low-level detail, so swapping
implementations still ripples back up. A DIP-respecting abstraction is
shaped by what the **high-level policy needs**, not by what the low-level
implementation happens to expose. `OrderRepository.save(order_id)` above is
phrased in domain terms (save an order), not `execute_sql(query, params)` —
the latter would just be `psycopg2`'s shape wearing an ABC costume.

## Summary

- DIP: high-level and low-level modules should both depend on an
  abstraction, instead of the high-level module depending directly on the
  low-level one. It's an architectural rule, distinct from Dependency
  Injection (the wiring mechanism).
- `abc.ABC` + `@abstractmethod` is one valid way to define that
  abstraction — nominal, enforces completeness at instantiation, and can
  carry real shared implementation.
- The abstraction has to be shaped around what the high-level policy
  needs, not around what one specific low-level implementation happens to
  expose — otherwise nothing was actually inverted.
- `Protocol` (structural) is the other valid way — see
  [Dependency Injection with Protocols](dependency-injection-protocols.md)
  for when that's the better fit.

## Related Articles

- [Dependency Injection with Protocols](dependency-injection-protocols.md)
  — the structural-typing alternative to ABCs, and why it avoids hierarchy
  hell for pure-capability contracts.
- [DDD & the Service Layer](../ddd-service-layer.md) — the service layer is
  typically the "high-level module" whose dependencies get inverted this
  way.
- [Factory Pattern in Python](factory-pattern.md) — factories are often
  what constructs the concrete low-level implementation handed to a
  DIP-respecting constructor.
