---
tags:

- python
- gil
- threading
- multiprocessing
- concurrency
- memory-model

---

# GIL, Threading, Multiprocessing and the Memory Model

This is the deeper companion to the short section in
[Python Tips & Tricks](python-tips.md#concurrency-and-parallelism). It covers *why* the
GIL exists, what it does and doesn't protect, and how memory is shared (or not) under
threads versus processes.

## Why the GIL Exists

CPython's memory management uses non-atomic reference counting. If two threads changed a
refcount simultaneously the count could be corrupted, leading to leaks or use-after-free.
The **Global Interpreter Lock** is one mutex that only lets one thread execute Python
bytecode at a time, making refcounts and interpreter internals safe without fine-grained
locks — and keeping single-threaded code and C extensions simple and fast.

Trade-off: no CPU parallelism across threads in one process.

## When the GIL Is Released

- Blocking I/O: sockets, file reads, `time.sleep`.
- Many C extensions during heavy work (NumPy, hashing, compression, `re` in some cases).
- The interpreter forces a switch periodically (`sys.getswitchinterval()`, 5 ms by
  default) so a CPU-bound thread can't starve others forever.

So threads help for **I/O-bound** work and for C code that releases the lock, but not for
pure-Python CPU-bound loops.

## The GIL Does Not Make Your Code Thread-Safe

A single bytecode is atomic, but a statement usually isn't:

```python
counter = 0

def work():
    global counter
    for _ in range(100_000):
        counter += 1   # load, add, store -> a switch can happen between them
```

Run in several threads, the final `counter` can fall short. Use `threading.Lock`, or
better, `queue.Queue` and immutable data. Things like `list.append` or `dict[key] = v` are
atomic in CPython, but relying on that is fragile — compound "check then act" sequences
still race.

## Threads vs. Processes

| | `threading` | `multiprocessing` |
| --- | --- | --- |
| Memory | Shared address space | Separate memory per process |
| GIL | One shared | One per process |
| CPU-bound speedup | No (pure Python) | Yes |
| Startup / overhead | Cheap | Heavy (fork/spawn) |
| Data exchange | Direct object access | Pickled through pipes/queues or shared memory |
| Failure isolation | A crash can take down all | Process dies alone |

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=16) as pool:      # I/O-bound
    pages = list(pool.map(fetch, urls))

with ProcessPoolExecutor() as pool:                   # CPU-bound
    results = list(pool.map(crunch, chunks))
```

`concurrent.futures` gives both a single interface; prefer it over raw `Thread`/`Process`
unless you need fine control.

## Multiprocessing Memory Model

- Each child has its own copy of memory. Arguments and return values are **pickled**, so
  they must be picklable and large payloads are expensive.
- Start method matters: `fork` (Linux default in older versions) copies the parent
  copy-on-write — fast, but unsafe with threads or held locks; `spawn` (macOS/Windows
  default) starts a fresh interpreter and re-imports your module, so guard entry points
  with `if __name__ == "__main__":`.
- To share data without copying, use `multiprocessing.shared_memory`, `Value`/`Array`, or
  a manager; otherwise pass messages.
- Note that reference counting writes to objects, so "copy-on-write" pages get copied
  anyway as children touch shared objects.

## Choosing a Model

- Waiting on network/disk, moderate concurrency → threads.
- Massive number of concurrent connections → [asyncio](asyncio-concurrency.md).
- Heavy pure-Python computation → processes (or push work into NumPy/C/Rust).
- Mixed → asyncio or threads for I/O, `run_in_executor` with a process pool for CPU.

## The GIL's Future

Python 3.13 introduced an experimental **free-threaded** build (PEP 703) that can run
without the GIL, and later releases continue to develop it. It requires extension
compatibility and has some single-thread overhead, so the default builds still have the
GIL. Even without it, shared mutable state still needs locks.

## Key Takeaways

- The GIL protects the interpreter's internals, not your data.
- Threads for I/O, processes for CPU; asyncio for many connections.
- Processes don't share memory: pay for pickling or use explicit shared memory.
- Always protect compound operations on shared state with locks or queues.

## Related Articles

- [Python Tips & Tricks](python-tips.md)
- [Designing Concurrent Tasks with asyncio](asyncio-concurrency.md)
- [Cyclic Garbage Collector](garbage-collection-cycles.md)
