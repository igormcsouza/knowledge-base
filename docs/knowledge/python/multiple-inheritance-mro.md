---
tags:

- python
- oop
- inheritance
- mro

---

# Multiple Inheritance and MRO

Python lets a class inherit from several parents. That raises the question: when a method
exists in more than one ancestor, which wins, and what does `super()` mean? The answer is
the **Method Resolution Order (MRO)** — a single linear ordering of a class's ancestors.

## The Diamond Problem

```python
class A:
    def hello(self): print("A")

class B(A):
    def hello(self): print("B"); super().hello()

class C(A):
    def hello(self): print("C"); super().hello()

class D(B, C):
    def hello(self): print("D"); super().hello()

D().hello()   # D, B, C, A
```

`A` is reached only once, and `C` runs between `B` and `A`, even though `B` inherits
directly from `A`. Inspect the order with:

```python
D.__mro__   # (D, B, C, A, object)
D.mro()
```

## C3 Linearization

Python computes the MRO with the **C3 algorithm**, which guarantees:

1. A class comes before its parents.
1. The order of bases in the `class` statement is preserved (`D(B, C)` → B before C).
1. The ordering is monotonic — a subclass never reorders what its parents established.

Formally, `L[D] = D + merge(L[B], L[C], [B, C])`. If no consistent order exists Python
raises `TypeError: Cannot create a consistent method resolution order`
(e.g. `class X(A, B)` where `B` subclasses `A`: base order contradicts the hierarchy).

## What `super()` Really Does

`super()` does **not** mean "my parent". It means "the next class after me in the MRO of
`type(self)`". In the example above, `B`'s `super().hello()` calls `C.hello`, not
`A.hello`, because `self` is a `D`.

Consequences:

- Every class in a cooperative hierarchy must call `super()` so the chain isn't cut off.
- Signatures must be compatible. The idiom is to accept and forward `**kwargs`:

```python
class Base:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)

class Named(Base):
    def __init__(self, name, **kwargs):
        self.name = name
        super().__init__(**kwargs)
```

- Calling `Parent.__init__(self)` explicitly bypasses the MRO and can run a shared base
  twice or skip siblings — avoid it in multi-inheritance code.

## Mixins: The Practical Use

Most healthy multiple inheritance is the **mixin**: a small class with one focused
behavior, no state of its own, not meant to be instantiated alone.

```python
class JSONMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class User(JSONMixin, Base): ...
```

Put mixins **before** the main base so their overrides win. Name them `*Mixin` so intent
is clear.

## Guidelines

- Prefer composition; use inheritance for "is-a" and mixins for orthogonal capabilities.
- Keep hierarchies shallow, and avoid diamonds unless they're cooperative by design.
- When debugging surprise behavior, print `Cls.__mro__` first.
- `isinstance`/`issubclass` follow the same ancestry; ABCs and `Protocol` are alternatives
  when you only need an interface (see
  [Dependency Injection with Protocols](../architecture/patterns/dependency-injection-protocols.md)).

## Key Takeaways

- MRO is a C3-computed linear order; read it from `__mro__`.
- `super()` = next in the MRO of the instance's type, not the literal parent.
- Cooperative multiple inheritance needs every class to call `super()` and forward
  `**kwargs`.

## Related Articles

- [Metaprogramming & Dunder Methods](metaprogramming-dunder-methods.md) — `__init_subclass__`
  and metaclasses, which also participate in class construction.
