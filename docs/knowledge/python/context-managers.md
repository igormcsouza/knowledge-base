---
tags:

- python
- context-managers
- resources
- contextlib

---

# Context Managers in Python

A context manager guarantees that setup and teardown code run as a pair, no matter how
the block in between exits — normally, via `return`, or via an exception. It is Python's
answer to "always release this resource", and the `with` statement is its syntax.

## What `with` Actually Does

```python
with open("data.txt") as f:
    data = f.read()
```

is roughly equivalent to:

```python
mgr = open("data.txt")
f = type(mgr).__enter__(mgr)
try:
    data = f.read()
except BaseException as exc:
    if not type(mgr).__exit__(mgr, type(exc), exc, exc.__traceback__):
        raise
else:
    type(mgr).__exit__(mgr, None, None, None)
```

Two methods make up the protocol:

- `__enter__(self)` runs on entry; its return value is bound to the name after `as`.
- `__exit__(self, exc_type, exc, tb)` runs on exit, always. If the block raised, the
  exception details are passed in; if it did not, all three are `None`. **Returning a
  truthy value from `__exit__` suppresses the exception**; returning `None`/`False` lets
  it propagate.

## Class-Based Context Managers

```python
import time


class Timer:
    def __enter__(self) -> "Timer":
        self.start = time.perf_counter()
        return self

    def __exit__(self, exc_type, exc, tb) -> None:
        self.elapsed = time.perf_counter() - self.start
        # returning None -> exceptions still propagate


with Timer() as t:
    sum(range(10_000_000))
print(t.elapsed)
```

## Generator-Based: `contextlib.contextmanager`

For most cases a generator is shorter. Everything before `yield` is `__enter__`, the
yielded value is the `as` target, and everything after is `__exit__`:

```python
from contextlib import contextmanager


@contextmanager
def transaction(conn):
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()
```

The gotcha: **an exception in the `with` body is re-raised at the `yield`**. If you want
cleanup to happen on failure you need `try`/`finally` (or `except`) around the `yield` —
without it, code after `yield` is skipped when the body raises.

## Useful `contextlib` Tools

- `suppress(*exceptions)` — swallow specific exceptions:
  `with suppress(FileNotFoundError): os.remove(path)`.
- `closing(obj)` — call `obj.close()` on exit for objects that have `close()` but are
  not context managers themselves.
- `redirect_stdout(target)` — temporarily redirect `print` output.
- `ExitStack` — manage a *dynamic* number of context managers, or register arbitrary
  cleanup callbacks:

```python
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(p)) for p in paths]
    # all files are closed on exit, in reverse order, even if one open() failed
```

- `nullcontext()` — a no-op stand-in, handy for "optionally lock/trace":
  `with (lock if needs_lock else nullcontext()):`.

## Multiple Managers and Async

Multiple managers can share one `with`; they are entered left to right and exited in
reverse:

```python
with open("in.txt") as src, open("out.txt", "w") as dst:
    dst.write(src.read())
```

For async resources (DB connections, HTTP sessions) the protocol is `__aenter__` /
`__aexit__`, used with `async with`; `contextlib.asynccontextmanager` is the generator
equivalent. See [asyncio concurrency](asyncio-concurrency.md).

## Key Takeaways

- `with` = guaranteed `__exit__`, including on exceptions and early `return`.
- `__exit__` returning truthy **swallows** the exception — do it deliberately.
- Prefer `@contextmanager` for simple cases; always wrap `yield` in `try`/`finally` when
  cleanup must run on error.
- Reach for `ExitStack` when the number of resources is only known at runtime.

## Related Articles

- [Decorators](decorators.md) — `@contextmanager` is itself a decorator.
- [Metaprogramming & Dunder Methods](metaprogramming-dunder-methods.md) — the data model
  protocol `__enter__`/`__exit__` belong to.
