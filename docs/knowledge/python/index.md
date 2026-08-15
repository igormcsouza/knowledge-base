---
title: Python
tags:

- python
- overview

---

# Python

Language-level notes: idioms, gotchas, and concepts worth remembering while writing
Python day to day.

## Articles

- [Python Tips & Tricks](python-tips.md) — name mangling, scopes and closures,
  concurrency vs. parallelism, threading vs. multiprocessing.
- [UV: Managing Python Environments and Projects](uv-python-tooling.md) — a
  fast, all-in-one replacement for pip, pyenv, virtualenv, and pip-tools.
- [Metaprogramming & Dunder Methods](metaprogramming-dunder-methods.md) — the data
  model protocol, `__call__`, descriptors, `__init_subclass__`, and when a metaclass
  actually earns its place.
- [Typing Generics & TypeVar](typing-generics-typevar.md) — `TypeVar`, generic classes
  (classic and PEP 695 syntax), `ParamSpec`, `overload`, and static checking with
  mypy/pyright.
- [Designing Concurrent Tasks with asyncio](asyncio-concurrency.md) — the event loop
  model, `TaskGroup` vs. `gather`, cancellation, and the synchronization primitives.
- [Logging & Tracing in Python](logging-and-tracing.md) — the `logging` module done
  right, `contextvars` for async-safe request context, and distributed tracing basics.

## Contributing

Found a Python quirk or pattern worth keeping? Add it here as its own file. See
[Contributing](../../contributing.md) for the how-to.
