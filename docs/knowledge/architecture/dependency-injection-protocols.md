---
tags:

- architecture
- dependency-injection
- python
- protocols
- design-patterns

---

# Dependency Injection with Protocols (Avoiding Hierarchy Hell)

Dependency Injection (DI) means a class receives its collaborators from the
outside — via its constructor or a parameter — instead of constructing them
itself. It's what makes code testable (swap a real dependency for a fake in
tests) and flexible (swap implementations without touching the consumer). The
part that trips people up isn't the "injection," it's *how you type the
thing being injected* — and that's where inheritance-based interfaces tend
to cause pain.

## The Traditional Approach: Abstract Base Classes

In many languages, and in "classic" Python, DI is paired with an interface
defined as an abstract base class (ABC). To be injectable, a concrete class
must explicitly **inherit** from that ABC:

```python
from abc import ABC, abstractmethod


class Notifier(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...


class EmailNotifier(Notifier):
    def send(self, message: str) -> None:
        print(f"Emailing: {message}")


class OrderService:
    def __init__(self, notifier: Notifier) -> None:
        self._notifier = notifier
```

This works, but the requirement to *inherit* is the problem. To satisfy
`Notifier`, a class has to declare itself as a `Notifier` in its class
hierarchy — even if it's otherwise unrelated to every other `Notifier`.
Scale this to a real codebase with a `Notifier`, a `Logger`, a `Serializer`,
a `Repository`, and every concrete class that needs to satisfy more than one
of them ends up juggling multiple inheritance and MRO (method resolution
order) just to type-check. Third-party or standard-library classes that
already have the right methods but weren't written to inherit from *your*
ABC can't satisfy the interface at all without a wrapper. This is
**hierarchy hell**: class trees that exist purely to satisfy typing, not to
share behavior.

## The Fix: Structural Typing with `Protocol`

Python's `typing.Protocol` ([PEP 544](https://peps.python.org/pep-0544/))
describes a **shape** — the methods/attributes a type must have — without
requiring any inheritance at all. Any class whose methods match the
protocol's signatures satisfies it automatically. This is **structural**
subtyping ("if it has `send(self, message: str) -> None`, it's a
`Notifier`") instead of **nominal** subtyping ("it's a `Notifier` only if it
says so in its class declaration").

```python
from typing import Protocol


class Notifier(Protocol):
    def send(self, message: str) -> None: ...


class EmailNotifier:
    def send(self, message: str) -> None:
        print(f"Emailing: {message}")


class SlackNotifier:
    def send(self, message: str) -> None:
        print(f"Slacking: {message}")


class OrderService:
    def __init__(self, notifier: Notifier) -> None:
        self._notifier = notifier

    def complete_order(self, order_id: str) -> None:
        self._notifier.send(f"Order {order_id} completed")
```

Neither `EmailNotifier` nor `SlackNotifier` inherits from `Notifier` — there
is no `Notifier` in their MRO at all. A type checker (`mypy`, `pyright`)
still verifies that anything passed to `OrderService.__init__` has a
matching `send(self, message: str) -> None` method. If it doesn't, that's a
type error, exactly like with the ABC — but the classes providing the
behavior stay completely decoupled from the contract's declaration.

## Why This Avoids Hierarchy Hell

- **No forced inheritance** — a class satisfies as many protocols as it
  structurally matches, with zero coordination between them. No multiple
  inheritance, no MRO conflicts, no diamond problem.
- **Works retroactively** — any existing class, including third-party or
  standard-library ones you don't control, satisfies a protocol the moment
  its shape matches. You never have to (and can't) go add `Notifier` to
  `smtplib`'s class hierarchy.
- **Contracts describe behavior, not identity** — the protocol says "must be
  able to `send` a message," not "must be a member of this class family."
  That's what DI actually needs: a capability, not a lineage.
- **Trivial to fake in tests** — a test double just needs the right method
  shape, not a base class to extend:

```python
class FakeNotifier:
    def __init__(self) -> None:
        self.sent: list[str] = []

    def send(self, message: str) -> None:
        self.sent.append(message)


def test_complete_order_notifies():
    notifier = FakeNotifier()
    service = OrderService(notifier)
    service.complete_order("123")
    assert notifier.sent == ["Order 123 completed"]
```

## `runtime_checkable` Protocols

By default, protocols are a **static** typing construct only — `isinstance()`
against a plain `Protocol` raises `TypeError`. Adding
`@runtime_checkable` allows `isinstance()` checks:

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Notifier(Protocol):
    def send(self, message: str) -> None: ...


isinstance(EmailNotifier(), Notifier)  # True
```

!!! warning
    Runtime protocol checks only verify that the *names* exist on the object
    — not their signatures, argument types, or return types. Treat
    `isinstance()` against a protocol as a coarse existence check, and rely
    on static type checking (`mypy`/`pyright`) for real signature
    verification.

## When ABCs Still Earn Their Place

Protocols are a pure contract — they carry no shared implementation. If you
actually want **shared code** across implementations (a base class with real
method bodies subclasses inherit and extend), that's what ABCs and mixins are
for — a different job than DI's "what shape does this dependency need."
Rule of thumb: reach for `Protocol` when you're describing a capability a
dependency must have; reach for an ABC/mixin when you're sharing real
behavior between related classes.

## Summary

- DI decouples a class from concretely constructing its dependencies —
  `Protocol` decouples the *dependency's type* from any inheritance
  requirement.
- ABC-based interfaces force every implementer into a shared class lineage,
  which multiplies into hierarchy hell as the number of interfaces grows.
- `Protocol` uses structural typing: any class with the right method shape
  satisfies it, with zero coordination or shared base class.
- Use `@runtime_checkable` sparingly, and only as a coarse existence check —
  the real verification happens at static type-check time.
- Keep ABCs for genuine shared implementation; use `Protocol` for pure
  behavioral contracts, which is what most DI actually needs.

## Related Articles

- [DDD & the Service Layer](ddd-service-layer.md) — services are exactly
  where injected, protocol-typed dependencies tend to live.
- [Factory Pattern in Python](factory-pattern.md) — factories are often what
  actually produces the concrete objects injected behind a `Protocol`.
