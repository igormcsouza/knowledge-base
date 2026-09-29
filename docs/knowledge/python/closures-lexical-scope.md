---
tags:

- python
- closures
- scope
- functions

---

# Closures and Lexical Scope in Python

Python resolves names by **lexical (static) scope**: where a function is *written*
determines which variables it can see — not where it is *called* from. A closure is what
you get when an inner function keeps using variables from the enclosing function after
that function has returned.

## The LEGB Rule

When Python looks up a name inside a function it searches, in order:

1. **L**ocal — the current function.
1. **E**nclosing — any enclosing function scopes, innermost first.
1. **G**lobal — the module level.
1. **B**uilt-in — `len`, `print`, etc.

Only functions, classes, and modules create scopes. `if`/`for`/`while`/`with` blocks do
**not**, so a variable assigned in a loop body is visible after the loop.

## Closures

```python
def make_counter():
    count = 0

    def increment():
        nonlocal count
        count += 1
        return count

    return increment


c = make_counter()
c(), c(), c()  # 1, 2, 3
```

`make_counter` has returned, yet `count` survives: `increment` holds a reference to it
through a **cell**. You can inspect this:

```python
c.__closure__[0].cell_contents  # 3
c.__code__.co_freevars          # ('count',)
```

Each call to `make_counter()` creates a new, independent cell — two counters do not share
state.

## `nonlocal` and `global`

Assignment makes a name **local to the whole function** at compile time. So this fails:

```python
def outer():
    x = 0
    def inner():
        x += 1   # UnboundLocalError: x is treated as local
    inner()
```

`nonlocal x` says "this name lives in the nearest enclosing function scope"; `global x`
says "module scope". Reading a free variable needs no declaration — only rebinding does.
Mutating an object (`items.append(...)`) is not rebinding, so it needs no `nonlocal`.

## The Classic Gotcha: Late Binding

Closures capture the **variable**, not its value at definition time:

```python
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]  # [2, 2, 2] -- all see the final i
```

Fixes: bind the value as a default argument, or use `functools.partial`:

```python
funcs = [lambda i=i: i for i in range(3)]   # [0, 1, 2]
```

## Where Closures Are Used

- **Function factories** and configuration ("make me a validator with this threshold").
- **Callbacks** that need a bit of context without a full class.
- **[Decorators](decorators.md)** — the wrapper closes over the decorated function and any
  decorator arguments.
- Lightweight private state; a class is usually clearer once there is more than one
  piece of state or more than one operation.

## Key Takeaways

- Scope is decided by where code is written (LEGB), not by the call site.
- Closures capture variables via cells; rebinding them requires `nonlocal`.
- Late binding bites in loops — capture with a default argument.
- Only functions/classes/modules create scope; blocks don't.

## Related Articles

- [Python Tips & Tricks](python-tips.md) — a short intro to scopes and closures.
- [Decorators](decorators.md)
