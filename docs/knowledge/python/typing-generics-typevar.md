---
tags:

- python
- typing
- generics
- tooling

---

# Typing Generics & TypeVar

Python's type hints are purely a static-analysis tool — they do nothing at runtime — but
they're what lets mypy/pyright catch real bugs before you run the code. Generics are the
part of the type system that lets a single class or function be typed *parametrically*:
"this function takes a `list[T]` and returns a `T`" instead of committing to one concrete
type. This article covers `TypeVar`, generic classes, `ParamSpec`, `overload`, and how it
all plugs into a type checker.

## `TypeVar`: The Basic Building Block

A `TypeVar` is a placeholder for "some type, to be determined by the caller":

```python
from typing import TypeVar

T = TypeVar("T")


def first(items: list[T]) -> T:
    return items[0]


first([1, 2, 3])        # inferred as int
first(["a", "b"])       # inferred as str
```

The type checker binds `T` to whatever concrete type is passed in at each call site, and
then checks the *return* type against that same binding. Without the `TypeVar`, `def
first(items: list) -> object` would type-check but lose the connection between input and
output — callers would get back `object`, not the specific element type.

### Bound TypeVars

A **bound** restricts `T` to a type (or its subtypes) — useful when the generic code needs
to call methods that only exist on a family of types:

```python
from typing import TypeVar

class Animal:
    def speak(self) -> str: ...

class Dog(Animal):
    def speak(self) -> str:
        return "Woof"

TAnimal = TypeVar("TAnimal", bound=Animal)

def make_speak(animal: TAnimal) -> TAnimal:
    print(animal.speak())  # allowed: bound guarantees .speak() exists
    return animal
```

`make_speak(Dog())` type-checks and the checker still knows the return type is `Dog`
specifically (not just `Animal`) — the bound narrows what's *allowed*, it doesn't widen what
comes back.

### Constrained TypeVars

A **constraint** restricts `T` to an explicit, closed set of types (not their subtypes,
unlike a bound):

```python
from typing import TypeVar

# T must be EXACTLY str or bytes, not a subtype of either
AnyStr = TypeVar("AnyStr", str, bytes)

def concat(a: AnyStr, b: AnyStr) -> AnyStr:
    return a + b


concat("x", "y")     # ok, T = str
concat(b"x", b"y")   # ok, T = bytes
concat("x", b"y")    # type error: mixed str/bytes
```

!!! note
    Use a bound when you need "any subtype of X, and give me back that exact subtype."
    Use a constraint when the valid types genuinely don't share a useful common
    supertype/behavior and you want to forbid mixing them, as with `str`/`bytes` above.

### Variance: Covariant and Contravariant

Variance controls whether `Container[Dog]` can be used where `Container[Animal]` is
expected. It only matters for *generic classes*, and only when the direction of data flow
(in vs. out) is fixed:

```python
from typing import TypeVar, Generic

T_co = TypeVar("T_co", covariant=True)      # read-only: only produces T
T_contra = TypeVar("T_contra", contravariant=True)  # write-only: only consumes T


class ReadOnlyBox(Generic[T_co]):
    def __init__(self, item: T_co) -> None:
        self._item = item

    def get(self) -> T_co:
        return self._item


class Sink(Generic[T_contra]):
    def push(self, item: T_contra) -> None:
        ...
```

- **Covariant** (`T_co`): if `Dog` is a subtype of `Animal`, `ReadOnlyBox[Dog]` is treated as
  a subtype of `ReadOnlyBox[Animal]`. Safe because the box only ever *hands out* a `T` —
  handing out a `Dog` where an `Animal` was expected is always fine. This is why `Sequence`,
  `Iterable`, and other read-only containers in `typing` are covariant.
- **Contravariant** (`T_contra`): the relationship flips — `Sink[Animal]` is treated as a
  subtype of `Sink[Dog]`, because something that can consume *any* `Animal` can safely
  consume a `Dog` too. This shows up for callback/consumer-shaped generics.
- **Invariant** (the default, no flag): `Box[Dog]` and `Box[Animal]` have no subtype
  relationship at all — required the moment a generic both reads *and* writes `T` (like a
  mutable `list`), because allowing either direction would let you put an `Animal` into
  what's supposed to be a `list[Dog]`.

!!! tip
    You rarely need to set variance by hand for your own classes — it mainly matters when
    writing library-style generic containers. In modern code (3.12+, see below), the `type`
    statement and `class Stack[T]` syntax infer variance automatically in most cases, and
    `typing.Generic` subclasses default to invariant, which is the safe choice when unsure.

## Generic Classes

### Classic Syntax (`Generic[T]`)

```python
from typing import Generic, TypeVar

T = TypeVar("T")


class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()


int_stack: Stack[int] = Stack()
int_stack.push(1)
int_stack.push("x")  # type error: expected int, got str
```

### Modern Syntax (PEP 695, Python 3.12+)

Python 3.12 introduced native syntax for generics that removes the need to import and
declare `TypeVar` separately — the type parameter is scoped directly to the class or
function:

```python
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()


def first[T](items: list[T]) -> T:
    return items[0]

# bounds and constraints inline, no separate TypeVar() call needed
class Repository[T: HasId]: ...
```

This is purely syntax sugar over the same `TypeVar`/`Generic` machinery — a `Stack[T]`
written this way is equivalent to the classic form above, just without the boilerplate.

!!! warning
    PEP 695 syntax requires Python **3.12+** at runtime (it's new grammar, not just a typing
    feature) — a library targeting older Python still needs the classic `TypeVar`/`Generic`
    form, or a `from __future__ import annotations`-adjacent shim isn't available here since
    this is actual syntax, not just annotations.

## `ParamSpec` and `Concatenate`: Typing Higher-Order Functions

A regular `TypeVar` captures a single type. `ParamSpec` captures an entire **parameter
list**, which is what you need to correctly type a decorator that wraps a function without
losing its signature:

```python
from typing import ParamSpec, TypeVar, Callable
import functools
import time

P = ParamSpec("P")
R = TypeVar("R")


def timed(func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.3f}s")
        return result
    return wrapper


@timed
def add(a: int, b: int) -> int:
    return a + b


add(1, 2)      # type-checks: (int, int) -> int, exactly like the original
add("1", "2")  # type error: caught, because P preserved add's real signature
```

Without `ParamSpec`, a typed decorator either collapses to `Callable[..., R]` (losing all
argument-checking on the wrapped function) or has to be hand-typed per call signature.
`P.args`/`P.kwargs` inside the wrapper is special syntax recognized only by type checkers —
it lets `wrapper` accept "whatever `func` accepts" and have that verified at every call site.

`Concatenate` extends this to a decorator that **adds** a parameter (like injecting a
`self` or a context object) while still forwarding the rest of the original signature:

```python
from typing import Concatenate, ParamSpec, TypeVar, Callable

P = ParamSpec("P")
R = TypeVar("R")


def with_logging(func: Callable[Concatenate[str, P], R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return func("prefix", *args, **kwargs)
    return wrapper
```

## `Protocol` for Structural Typing

`Protocol` describes a required *shape* (methods/attributes) rather than a required base
class — a class satisfies it just by having matching methods, no inheritance needed. This
matters a lot for generics: a bound like `TypeVar("T", bound=SomeProtocol)` lets you accept
"anything shaped like X" as a type parameter, without forcing every caller's class into an
inheritance hierarchy.

This is covered in full depth, including why it beats ABC-based interfaces for dependency
injection, in
[Dependency Injection with Protocols](../architecture/patterns/dependency-injection-protocols.md)
— that article is the one to read for `Protocol` itself; the relevant piece for generics is
just that a `Protocol` can be used as a `TypeVar` bound exactly like a concrete class can.

## `overload`: Multiple Signatures for One Function

`@overload` lets a type checker see that a function's return type depends on which argument
*types* were passed, when a single signature can't express that:

```python
from typing import overload


@overload
def get_config(key: str) -> str: ...
@overload
def get_config(key: str, default: int) -> int: ...
def get_config(key: str, default: object = None) -> object:
    value = _config.get(key)
    return value if value is not None else default
```

The `@overload`-decorated stubs are never actually called — they exist purely for the type
checker. The final, undecorated definition is the real implementation, and it must accept
every argument combination the overloads promise. Callers see precise return types
(`get_config("x")` → `str`, `get_config("x", 0)` → `int`) instead of a vague union.

!!! tip
    Reach for `overload` only when the return type genuinely depends on which *argument
    types* were passed — not just as a way to document optional-argument variants,
    which a single signature with defaults already covers fine.

## `TypeAlias` and the `type` Statement

Naming a complex type improves both readability and error messages:

```python
# classic: explicit TypeAlias annotation (3.10+), or just plain assignment before that
from typing import TypeAlias

JsonValue: TypeAlias = "dict[str, JsonValue] | list[JsonValue] | str | int | float | bool | None"

# modern: the `type` statement (3.12+) — a real, distinct alias, lazily evaluated
type JsonValue = dict[str, JsonValue] | list[JsonValue] | str | int | float | bool | None
```

The `type` statement is lazily evaluated (its right-hand side isn't resolved until needed),
which is what lets `JsonValue` reference itself recursively without a forward-reference
string — something the classic `TypeAlias` form needs the quotes for, as above.

## How This Plugs Into Static Checking

None of this article's syntax does anything at runtime by itself — `mypy` or `pyright` read
the annotations and type parameters and report a diagnostic before the code ever runs. A
minimal setup:

```bash
pip install mypy
mypy src/
```

```ini
# mypy.ini or pyproject.toml [tool.mypy]
[mypy]
python_version = 3.12
disallow_untyped_defs = true
warn_return_any = true
```

!!! note
    `pyright` (used by VS Code's Pylance) and `mypy` both implement the same typing PEPs but
    disagree on edge cases occasionally — variance inference and `ParamSpec` corner cases are
    common places they diverge. If CI runs one and your editor runs the other, don't be
    surprised when they disagree; treat CI's checker as the source of truth.

## Summary

- `TypeVar` parametrizes a function/class over "some type"; `bound=` restricts to a
  type-and-subtypes, an explicit constraint list restricts to exact alternatives only.
- Variance (`covariant`/`contravariant`) only matters for generic *classes*, and follows
  data flow: read-only is covariant, write-only is contravariant, read+write is invariant.
- `class Stack[T]` (3.12+) is sugar over the classic `Generic[T]`/`TypeVar` pattern — same
  semantics, less boilerplate, but requires 3.12+ at runtime since it's real syntax.
- `ParamSpec`/`Concatenate` type decorators and other higher-order functions by capturing an
  entire parameter list, not just one type.
- `Protocol` gives structural typing for generics and DI alike — see the dedicated article
  for the full picture.
- `overload` documents type-dependent return types; the real implementation is the final
  undecorated definition.
- All of this is purely static — `mypy`/`pyright` are what actually enforce it; nothing here
  changes runtime behavior.

## Related Articles

- [Dependency Injection with Protocols](../architecture/patterns/dependency-injection-protocols.md)
  — the full treatment of `Protocol` and structural typing referenced above.
- [Metaprogramming & Dunder Methods](metaprogramming-dunder-methods.md) — the dunder methods
  (`__getitem__`, `__iter__`) that generic container classes typically implement alongside
  their type parameters.
- [Python Tips & Tricks](python-tips.md) — general Python idioms and gotchas.
