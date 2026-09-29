---
tags:

- python
- decorators
- functools
- closures

---

# Decorators in Python

A decorator is a callable that takes a function (or class) and returns a replacement,
usually a wrapper that adds behavior. The `@` syntax is sugar:

```python
@decorator
def f(): ...

# is exactly
def f(): ...
f = decorator(f)
```

It happens once, at definition time. Decorators rely on
[closures](closures-lexical-scope.md) to remember the original function.

## A Well-Behaved Decorator

```python
import functools
import time


def timed(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            return func(*args, **kwargs)
        finally:
            print(f"{func.__name__} took {time.perf_counter() - start:.3f}s")

    return wrapper
```

Always use **`functools.wraps`**: it copies `__name__`, `__doc__`, `__module__`, and sets
`__wrapped__`, so debugging, introspection, and `help()` still point at the original
function instead of "wrapper". Forward `*args, **kwargs` so any signature works, and
return the result.

## Decorators With Arguments

`@retry(times=3)` means `retry(times=3)` is called first and must *return* the real
decorator — so you need three nested levels:

```python
def retry(times: int = 3, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise
        return wrapper
    return decorator


@retry(times=5, exceptions=(ConnectionError,))
def fetch(): ...
```

## Stacking and Order

Decorators apply bottom-up (closest to the function first) but the resulting wrappers run
top-down:

```python
@a
@b
def f(): ...   # f = a(b(f)); calling f() enters a's wrapper first
```

Order matters: `@login_required` above `@cache` means auth is checked before the cache;
reverse them and cached results skip the check.

## Class-Based and Class Decorators

A class with `__call__` can be a decorator, which is convenient when it needs state
(call counts, a cache). A decorator can also receive a *class* and return it modified
(e.g. `@dataclass`) — often a lighter alternative to a metaclass; see
[Metaprogramming & Dunder Methods](metaprogramming-dunder-methods.md).

## Decorating Methods and Async Functions

- A plain function wrapper works on methods because `self` arrives in `*args`.
- For `async def`, the wrapper must itself be `async def` and `await` the call —
  otherwise you return an un-awaited coroutine and skip timing/retry logic entirely.

## Built-in Decorators Worth Knowing

- `@property`, `@staticmethod`, `@classmethod`
- `@functools.lru_cache` / `@functools.cache` — memoization (arguments must be hashable;
  on methods it also keeps `self` alive)
- `@functools.singledispatch` — dispatch on argument type
- `@contextlib.contextmanager` — see [context managers](context-managers.md)
- `@dataclasses.dataclass`

## Key Takeaways

- `@dec` is `f = dec(f)`, run at definition time.
- Use `functools.wraps`, forward `*args, **kwargs`, return the result.
- Decorator with arguments = a factory returning the decorator (three levels).
- Stacking applies bottom-up, executes top-down.

## Related Articles

- [Closures and Lexical Scope](closures-lexical-scope.md)
- [Context Managers](context-managers.md)
