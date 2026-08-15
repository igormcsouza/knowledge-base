---
tags:

- python
- metaprogramming
- dunder-methods
- advanced

---

# Metaprogramming & Dunder Methods

"Dunder" (double-underscore) methods like `__init__` or `__len__` are how Python objects
plug into the language's own machinery — `len(obj)`, `obj + other`, `for x in obj`, `with
obj:` all just call a dunder method under the hood. Metaprogramming takes this a step
further: writing code that shapes *how classes themselves get built*, via descriptors,
class-creation hooks, and metaclasses. This article covers both, from the data model
protocol up to when a metaclass is actually the right tool.

## The Data Model Protocol

Every dunder method is Python asking "does this object know how to do X?" instead of
special-casing types. Implement the method, and your object works with the built-in that
calls it.

```python
class Money:
    def __init__(self, cents: int) -> None:
        self.cents = cents

    def __repr__(self) -> str:
        return f"Money(cents={self.cents!r})"

    def __str__(self) -> str:
        return f"${self.cents / 100:.2f}"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return self.cents == other.cents

    def __hash__(self) -> int:
        return hash(self.cents)

    def __add__(self, other: "Money") -> "Money":
        if not isinstance(other, Money):
            return NotImplemented
        return Money(self.cents + other.cents)
```

A few details that matter more than they look:

- `__repr__` should be unambiguous and, ideally, look like the code that recreates the
  object (`Money(cents=500)`) — it's what you see in a debugger, a REPL, and a log
  traceback. `__str__` is the human-friendly version; if you don't define `__str__`, Python
  falls back to `__repr__`.
- Return `NotImplemented` (not `False` or raising) from comparison/arithmetic dunders when
  the other operand's type isn't supported. This lets Python try the *other* object's
  reflected method (`other.__radd__(self)`) before giving up with a real `TypeError`.
- **Defining `__eq__` without `__hash__` makes your objects unhashable** — Python sets
  `__hash__` to `None` automatically the moment you override `__eq__`, because it can no
  longer assume equal objects hash the same. If you want equality *and* to use instances as
  dict keys or in sets, you must define both, and they must agree: equal objects need equal
  hashes.

!!! warning
    A mutable object generally shouldn't define `__hash__` at all (or should raise). If two
    "equal" objects can later become unequal because one mutated, and one of them is already
    sitting in a `set`, that set is now silently broken — the object is in the wrong hash
    bucket and lookups for it will fail.

### Container Protocols

Implementing `__len__`, `__getitem__`, `__setitem__`, and `__iter__` makes a custom type
behave like a built-in container:

```python
class Deck:
    def __init__(self) -> None:
        self._cards = [f"{r}{s}" for s in "SHDC" for r in "23456789TJQKA"]

    def __len__(self) -> int:
        return len(self._cards)

    def __getitem__(self, index: int) -> str:
        return self._cards[index]

    def __setitem__(self, index: int, value: str) -> None:
        self._cards[index] = value


deck = Deck()
len(deck)          # 52
deck[0]             # "2S"
deck[-1]            # "AC"
for card in deck:   # __getitem__ alone is enough for iteration...
    ...
```

!!! note
    `__getitem__` starting at index `0` and raising `IndexError` when exhausted is enough
    for `for`/`in` to work, even without `__iter__` — Python falls back to calling
    `__getitem__(0)`, `__getitem__(1)`, ... until it hits `IndexError`. It's a neat trick,
    but writing an explicit `__iter__`/`__next__` pair (below) is clearer and is required if
    your container isn't naturally indexable by integers (a graph, a linked list, a set).

```python
class Countdown:
    def __init__(self, start: int) -> None:
        self.start = start

    def __iter__(self) -> "Countdown":
        self._current = self.start
        return self

    def __next__(self) -> int:
        if self._current <= 0:
            raise StopIteration
        self._current -= 1
        return self._current + 1


for n in Countdown(3):
    print(n)  # 3, 2, 1
```

`__iter__` returns an iterator (an object with `__next__`); `__next__` raises `StopIteration`
to signal exhaustion — that's the entire contract behind `for` loops, comprehensions,
`list()`, `sum()`, and unpacking.

### Context Managers: `__enter__` / `__exit__`

`with` blocks are just sugar for calling `__enter__` on entry and `__exit__` on exit —
`__exit__` runs even if the block raises, which is what makes it the right place for
cleanup:

```python
class Transaction:
    def __init__(self, conn) -> None:
        self.conn = conn

    def __enter__(self) -> "Transaction":
        self.conn.begin()
        return self

    def __exit__(self, exc_type, exc_value, traceback) -> bool:
        if exc_type is None:
            self.conn.commit()
        else:
            self.conn.rollback()
        return False  # False = don't suppress the exception; re-raise it


with Transaction(conn) as tx:
    conn.execute("UPDATE accounts SET balance = 500 WHERE id = 'A'")
    # commits on clean exit, rolls back if anything above raises
```

`__exit__` returning a truthy value **swallows** the exception — usually not what you want.
Return `False` (or `None`, which is falsy) unless you deliberately mean to suppress specific
exception types.

## `__call__`: Callable Objects

Defining `__call__` lets an instance be invoked like a function — useful for objects that
need to carry configuration/state between calls (a decorator with parameters, a caching
wrapper, a strategy object):

```python
class Multiplier:
    def __init__(self, factor: int) -> None:
        self.factor = factor

    def __call__(self, value: int) -> int:
        return value * self.factor


double = Multiplier(2)
double(21)  # 42 — `double` is used exactly like a function
```

This is also what makes classes usable as decorators, and it's how libraries like PyTorch
implement `Module.__call__` to wrap `forward()` with hooks.

## `__new__` vs. `__init__`

`__new__` **creates** the instance (it's a `staticmethod`, called before an instance
exists); `__init__` **initializes** an already-created instance. Almost all the time you
only need `__init__` — `__new__` matters when you need to control *whether/what* gets
created, not just how it's set up:

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

Other real uses: subclassing immutable built-ins (`int`, `str`, `tuple`) where the value
must be fixed at creation time, not after; and returning an instance of a *different* class
than the one being constructed.

## Class Creation Hooks

### `__init_subclass__`

Runs automatically whenever a subclass is defined — lets a base class validate or register
subclasses without any metaclass machinery:

```python
class Plugin:
    registry: dict[str, type["Plugin"]] = {}

    def __init_subclass__(cls, *, name: str, **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        Plugin.registry[name] = cls


class JsonPlugin(Plugin, name="json"):
    ...


Plugin.registry  # {"json": <class 'JsonPlugin'>}
```

This covers the majority of cases people reach for metaclasses for: plugin registries,
enforcing that subclasses implement a required attribute, auto-generating `__slots__`, and
similar "do something when a subclass is defined" needs.

### `__set_name__`

Called on a descriptor when the class *owning* it is created, telling the descriptor the
attribute name it was assigned to — this is how you avoid repeating the field name twice:

```python
class Field:
    def __set_name__(self, owner: type, name: str) -> None:
        self.name = f"_{name}"

    def __get__(self, instance, owner):
        return getattr(instance, self.name, None)

    def __set__(self, instance, value) -> None:
        setattr(instance, self.name, value)


class User:
    email = Field()  # __set_name__ tells this Field its name is "email"
```

## Descriptors: How `property` Actually Works

A **descriptor** is any object implementing `__get__` (and optionally `__set__`/
`__delete__`) and assigned as a *class* attribute. When you access `instance.attr`, Python
checks whether the class-level `attr` is a descriptor and, if so, routes through its
`__get__`/`__set__` instead of a plain dict lookup. `property` is a thin built-in wrapper
around exactly this protocol:

```python
class Positive:
    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return getattr(instance, self.name)

    def __set__(self, instance, value):
        if value <= 0:
            raise ValueError(f"{self.name[1:]} must be positive, got {value}")
        setattr(instance, self.name, value)


class Rectangle:
    width = Positive()
    height = Positive()

    def __init__(self, width: float, height: float) -> None:
        self.width = width    # goes through Positive.__set__
        self.height = height
```

`property(fget, fset, fdel)` is literally a descriptor built exactly this way — it's why
`@property` behaves identically to a hand-rolled descriptor, just with less boilerplate for
the single-attribute case. Reach for a custom descriptor (like `Positive` above) when the
*same* validation/behavior needs to apply across several attributes or several classes —
otherwise `@property` is simpler and more idiomatic.

!!! tip
    Descriptors defined on a class are shared by every instance of that class — that's why
    `Positive` stores the value on the *instance* (`setattr(instance, self.name, value)`)
    rather than on `self`. Storing it on `self` inside the descriptor would leak the value
    across every instance of every class that uses that `Positive` field.

## Metaclasses

A metaclass is "the class of a class" — just like a class controls how *instances* are
built, a metaclass controls how *classes* are built. `type` is the default metaclass; every
class you write is, unless told otherwise, an instance of `type`.

```python
class Meta(type):
    def __new__(mcs, name, bases, namespace, **kwargs):
        cls = super().__new__(mcs, name, bases, namespace)
        if "process" not in namespace and bases:  # bases empty for the root class
            raise TypeError(f"{name} must define 'process'")
        return cls


class Handler(metaclass=Meta):
    def process(self) -> None: ...


class Broken(metaclass=Meta):  # raises TypeError at class-definition time
    pass
```

The metaclass's `__new__`/`__init__` run when the *class statement* executes — before any
instance exists — which is what lets a metaclass reject, rewrite, or wrap the class object
itself (its methods, its `__dict__`, its bases) at definition time.

### When You'd Actually Reach for One

Metaclasses are the most powerful and least readable tool in this article — they change
behavior at a point most Python developers don't expect to be customizable, which makes
code harder to onboard onto. In practice, a real need for one is rare:

- **`__init_subclass__` covers "run code when a subclass is defined"** — registries,
  validating that required methods exist, auto-wiring configuration. This is almost always
  what people reach for a metaclass for, and it's far easier to read.
- **A class decorator covers "wrap or modify a finished class"** — adding methods,
  registering the class somewhere, injecting `__slots__`. `@dataclass` itself is a class
  decorator, not a metaclass, and it modifies plain classes just fine.
- **Reach for a metaclass when you need to intercept `type.__new__`/`__call__` itself** —
  controlling what happens *before* the class body even finishes executing (rewriting the
  namespace dict as it's built), or controlling instantiation for *every* instance of every
  subclass uniformly (ORMs like Django's models, or enforcing a shared metaclass across an
  entire framework's class hierarchy, are the canonical real examples).

!!! warning
    Metaclasses don't compose the way you'd hope: a class can only have one metaclass, and if
    two base classes have different, unrelated metaclasses, Python raises a
    `TypeError: metaclass conflict` at class-definition time unless you write a metaclass
    that's a subclass of both. This is the single biggest practical reason to prefer
    `__init_subclass__` or a class decorator whenever it's sufficient — they don't have this
    problem at all, because they don't touch the class-of-a-class relationship.

## Summary

- Dunder methods are how a custom type opts into the language's built-in operations —
  `len()`, `+`, `for`, `with`, equality, hashing — each maps to a specific method Python
  looks up automatically.
- Overriding `__eq__` disables `__hash__` unless you define both explicitly; keep them
  consistent, and avoid `__hash__` on mutable objects entirely.
- `__new__` creates the instance, `__init__` configures it — you almost always only need
  the latter.
- `__init_subclass__` and `__set_name__` handle most "customize class creation" needs
  without the complexity of a metaclass.
- Descriptors (`__get__`/`__set__`) are the mechanism `property` is built on; reach for a
  custom descriptor when the same behavior needs to be reused across multiple attributes or
  classes.
- Metaclasses are real but rarely necessary — try `__init_subclass__` or a class decorator
  first, and remember a class can only have one metaclass.

## Related Articles

- [Python Tips & Tricks](python-tips.md) — covers **name mangling** (the `__name` *variable*
  rewriting to `_ClassName__name`), a related but distinct feature from the dunder *methods*
  covered here — mangling is about attribute name rewriting, this article is about the
  operator/protocol hooks the language calls automatically.
- [Dependency Injection with Protocols](../architecture/patterns/dependency-injection-protocols.md)
  — structural typing via `Protocol` is a different (static-typing-only) way classes can
  satisfy a shape/contract, without touching the runtime dunder machinery here.
- [Typing Generics & TypeVar](typing-generics-typevar.md) — typing generic classes that use
  these same dunder methods (`__getitem__`, `__iter__`) with proper type parameters.
