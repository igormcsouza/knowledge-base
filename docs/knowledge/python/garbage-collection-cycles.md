---
tags:

- python
- garbage-collection
- memory
- cpython

---

# Cyclic Garbage Collector in Python

CPython manages memory with two cooperating mechanisms: **reference counting**, which
handles almost everything immediately, and a **cyclic garbage collector**, which exists
for one case reference counting cannot solve — reference cycles.

## Reference Counting

Every object carries a count of references to it. When it drops to zero the object is
freed on the spot (and `__del__` runs). This is deterministic: leaving a function scope
frees its locals right away.

```python
import sys
x = []
sys.getrefcount(x)   # 2: `x` plus the temporary argument
```

## Where It Fails: Cycles

```python
class Node:
    def __init__(self):
        self.other = None

a, b = Node(), Node()
a.other = b
b.other = a
del a, b   # counts are still 1 each -> never reach zero
```

Each object keeps the other alive although nothing outside can reach them. Self-references
(`lst.append(lst)`) and closures/callbacks that point back at their owner create the same
problem.

## How the Cycle Collector Works

The `gc` module's collector only tracks **container objects** (lists, dicts, instances,
etc. — things that can hold references). Atomic types like `int` and `str` can't form
cycles and aren't tracked. It finds garbage roughly like this:

1. For each tracked object, copy its refcount.
2. Subtract references that come *from other tracked objects* (internal references).
3. Anything left with a positive count is referenced from outside the set → reachable.
4. Everything reachable from those is kept; the rest is unreachable cyclic garbage and is
   freed.

### Generations

Most objects die young, so collecting everything each time would waste work. Objects are
grouped into three **generations**:

- Gen 0 — new objects; collected most often.
- Gen 1 / Gen 2 — survivors get promoted; collected progressively less often.

A collection of a generation is triggered when the count of allocations minus
deallocations exceeds that generation's threshold. (Newer CPython versions have adjusted
the exact scheme, so treat the numbers as tunable, not fixed.)

```python
import gc

gc.get_threshold()      # e.g. (700, 10, 10)
gc.collect()            # force a full collection, returns number unreachable found
gc.disable()            # only do this if you know you create no cycles
gc.get_stats()
```

## Finalizers and `weakref`

Since Python 3.4 (PEP 442), objects with `__del__` in cycles *can* be collected, but
finalizer timing is still unpredictable — don't rely on `__del__` for releasing resources;
use [context managers](context-managers.md).

Break cycles by design with **weak references**, which don't increase the refcount:

```python
import weakref

class Child:
    def __init__(self, parent):
        self.parent = weakref.ref(parent)   # call self.parent() to dereference
```

`weakref.WeakValueDictionary` and `WeakSet` are handy for caches and registries that
shouldn't keep objects alive.

## Practical Notes

- Memory "leaks" in Python are usually lingering references (globals, caches, closures,
  stored exceptions/tracebacks), not GC failures. `gc.get_referrers`, `tracemalloc`, and
  `objgraph` help track them down.
- Large latency-sensitive services sometimes `gc.freeze()` after startup or tune
  thresholds to avoid long pauses, but measure first.
- Freed memory isn't always returned to the OS; the allocator (pymalloc) keeps arenas.
- Other implementations differ: PyPy uses tracing GC and no refcounts, so don't depend on
  immediate destruction.

## Key Takeaways

- Refcounting frees most objects instantly; the cyclic GC is a backstop for cycles.
- Only container objects are tracked; collection is generational.
- Avoid cycles with `weakref`; don't use `__del__` for resource cleanup.

## Related Articles

- [GIL, Threading, Multiprocessing and the Memory Model](gil-threading-multiprocessing.md)
- [Closures and Lexical Scope](closures-lexical-scope.md) — closures are a common source
  of hidden references.
