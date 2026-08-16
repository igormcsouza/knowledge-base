---
title: Patterns
tags:

- architecture
- design-patterns
- overview

---

# Patterns

Code-level design patterns: reusable solutions for structuring and
constructing objects within a service, as distinct from the broader
architectural styles in the parent [Architecture](../index.md) section
(how a whole system is organized, how services communicate). The
architectural *goal* these two patterns exist to satisfy —
[Dependency Inversion](../dependency-inversion-abstract-classes.md) — lives
one level up, in Architecture itself, rather than here.

## Articles

- [Dependency Injection with Protocols](dependency-injection-protocols.md) —
  structural typing with `typing.Protocol` instead of ABC inheritance, and
  why that avoids hierarchy hell.
- [Factory Pattern in Python](factory-pattern.md) — from a plain function to
  registries to `@classmethod` alternative constructors.

## Contributing

Picked up a design pattern worth documenting — GoF-style or a Pythonic take
on one? Add it here as its own file. See
[Contributing](../../../contributing.md) for the how-to.
