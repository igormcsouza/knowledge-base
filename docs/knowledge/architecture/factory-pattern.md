---
tags:

- architecture
- python
- design-patterns
- factory

---

# Factory Pattern in Python

A **factory** is anything whose job is to create objects on behalf of other
code, so the caller doesn't need to know *which concrete class* to
instantiate or *how* to build it. It's a creational pattern: the value it
adds is decoupling "I need a thing that can do X" from "here's exactly how
that thing gets constructed."

## Why Bother

Without a factory, creation logic and branching on "which implementation do I
need" spreads across every call site that needs an object:

```python
# scattered everywhere a notifier is needed
if config.channel == "email":
    notifier = EmailNotifier(smtp_host=config.smtp_host)
elif config.channel == "slack":
    notifier = SlackNotifier(webhook_url=config.webhook_url)
else:
    raise ValueError(f"unknown channel: {config.channel}")
```

Centralizing that logic behind one function means callers just ask for what
they need, and adding a new implementation later touches one place instead
of every call site (the Open/Closed Principle in practice).

## Simple Factory Function

The most common, most Pythonic version is just a function:

```python
def create_notifier(kind: str) -> Notifier:
    if kind == "email":
        return EmailNotifier()
    if kind == "slack":
        return SlackNotifier()
    raise ValueError(f"unknown notifier kind: {kind}")
```

Callers depend only on `create_notifier` and the `Notifier`
[protocol](dependency-injection-protocols.md) — never on `EmailNotifier` or
`SlackNotifier` directly.

## Registry-Based Factory

Once the `if/elif` chain grows past a handful of branches, a dict-based
registry reads better and scales without touching the factory function
itself when a new type is added:

```python
_NOTIFIER_REGISTRY: dict[str, type[Notifier]] = {
    "email": EmailNotifier,
    "slack": SlackNotifier,
}


def create_notifier(kind: str) -> Notifier:
    try:
        notifier_cls = _NOTIFIER_REGISTRY[kind]
    except KeyError:
        raise ValueError(f"unknown notifier kind: {kind}") from None
    return notifier_cls()
```

Taking it a step further, implementations can register *themselves* with a
decorator — genuinely useful for plugin-style extension points, where new
types shouldn't require editing the registry file at all:

```python
_REGISTRY: dict[str, type[Notifier]] = {}


def register(kind: str):
    def decorator(cls: type[Notifier]) -> type[Notifier]:
        _REGISTRY[kind] = cls
        return cls

    return decorator


@register("email")
class EmailNotifier:
    def send(self, message: str) -> None:
        print(f"Emailing: {message}")


def create_notifier(kind: str) -> Notifier:
    return _REGISTRY[kind]()
```

## The Classic GoF Version: Factory Method

The textbook Factory Method pattern pushes creation behind a class hierarchy
instead of a function — a base factory declares a `create()` method, and
subclasses override it to decide which concrete class to instantiate:

```python
from abc import ABC, abstractmethod


class NotifierFactory(ABC):
    @abstractmethod
    def create(self) -> Notifier: ...


class EmailNotifierFactory(NotifierFactory):
    def create(self) -> Notifier:
        return EmailNotifier(smtp_host="smtp.example.com")


class SlackNotifierFactory(NotifierFactory):
    def create(self) -> Notifier:
        return SlackNotifier(webhook_url="https://hooks.example.com/xyz")
```

This earns its complexity when construction genuinely varies by more than a
single parameter (different dependencies, different setup steps per type),
or when the *factory itself* needs to be swapped as a dependency. For a
simple "pick one of a few known types," it's usually more machinery than the
plain function or registry above.

!!! note "Personal take"
    In most Python code, a plain function or dict registry beats a full
    Factory Method class hierarchy. Python has first-class functions and
    classes-as-objects, so the ceremony a class-based factory pattern needs
    in a language without those (Java, C#) mostly isn't necessary here. Reach
    for the class-based version only when the factory itself needs to carry
    state or be swapped polymorphically.

## Alternative Constructors: `@classmethod` Factories

A very common, lightweight factory pattern in Python doesn't need a separate
factory object at all — a `@classmethod` on the class itself, building the
object from a different kind of input than `__init__` expects:

```python
class User:
    def __init__(self, name: str, email: str) -> None:
        self.name = name
        self.email = email

    @classmethod
    def from_dict(cls, data: dict) -> "User":
        return cls(name=data["name"], email=data["email"])
```

The standard library does this constantly —
`datetime.fromtimestamp(...)`, `dict.fromkeys(...)`, `Path.cwd()` — each is a
factory that builds the object a different way than the plain constructor.

## When to Reach for a Factory

- Object creation involves real branching logic (which concrete type,
  based on config/runtime data).
- You want callers decoupled from concrete implementation classes — pairs
  naturally with [Protocol-based dependency injection](dependency-injection-protocols.md),
  where the factory is what actually produces the protocol-typed object
  being injected.
- You want a plugin-style extension point where new implementations
  register themselves instead of requiring edits to existing code.
- You need an alternative way to build an object from a different input
  shape than `__init__` takes — that's what `@classmethod` factories are for,
  and it's usually all you need.

## Summary

- A factory centralizes "which concrete class, and how to build it" behind
  one place, decoupling callers from concrete implementations.
- Start with a plain function; move to a dict registry once branching grows;
  reach for a full Factory Method class hierarchy only when the factory
  itself needs to be swappable or carry state.
- `@classmethod` alternative constructors are a lightweight, very Pythonic
  factory pattern worth reaching for before building a separate factory
  class.

## Related Articles

- [Dependency Injection with Protocols](dependency-injection-protocols.md)
  — factories are typically what produces the protocol-typed objects that
  get injected.
- [DDD & the Service Layer](ddd-service-layer.md) — a natural place for
  factory functions that build domain objects for a service to use.
