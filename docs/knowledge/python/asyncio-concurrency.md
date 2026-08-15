---
tags:

- python
- asyncio
- concurrency
- async

---

# Designing Concurrent Tasks with asyncio

`asyncio` gives Python single-threaded concurrency built around an **event loop**:
instead of OS threads taking turns via preemption, coroutines cooperatively yield control
at `await` points, and the loop decides what runs next. This article covers the mental
model, task scheduling, structured concurrency with `TaskGroup`, cancellation, the
synchronization primitives, and the pitfalls that actually bite in production code.

## The Event Loop Model

There's one event loop per thread (in the common case, one loop total), and it's the thing
actually running your coroutines. A coroutine defined with `async def` doesn't execute when
called — calling it just creates a coroutine *object*; nothing runs until something
`await`s it or schedules it on the loop:

```python
import asyncio


async def greet(name: str) -> str:
    await asyncio.sleep(1)
    return f"Hello, {name}"


async def main() -> None:
    result = await greet("Ada")  # runs greet, suspending main() until it's done
    print(result)


asyncio.run(main())  # creates the loop, runs main() to completion, closes the loop
```

`await` is the cooperative yield point: it hands control back to the event loop, which can
then run other pending work, and resumes the coroutine when whatever it awaited (an I/O
result, a timer, another task) completes. Nothing preempts a coroutine mid-execution — it
runs uninterrupted until it hits an `await`. This is why a synchronous, CPU-bound loop with
no `await` inside an `async def` blocks *everything else on that loop*, not just itself.

## Creating and Scheduling Tasks

`await`ing a coroutine directly runs it to completion before moving on — no concurrency.
`asyncio.create_task` is what actually schedules a coroutine to run *concurrently*,
returning immediately with a `Task` handle:

```python
async def fetch(url: str) -> str:
    await asyncio.sleep(1)  # stand-in for a real async HTTP call
    return f"data from {url}"


async def sequential() -> None:
    a = await fetch("a")  # ~1s
    b = await fetch("b")  # then another ~1s — total ~2s


async def concurrent() -> None:
    task_a = asyncio.create_task(fetch("a"))  # starts running immediately
    task_b = asyncio.create_task(fetch("b"))  # also starts immediately
    a = await task_a  # both were already in flight — total ~1s
    b = await task_b
```

`create_task` schedules the coroutine on the loop right away (it starts making progress the
next time the loop gets control, even before you `await` the task) — that's the difference
between "concurrent" and "sequential" above.

## `asyncio.gather` vs. `asyncio.TaskGroup`

Both run multiple coroutines/tasks concurrently and wait for all of them; they differ
sharply in error handling.

### `gather`

```python
results = await asyncio.gather(fetch("a"), fetch("b"), fetch("c"))
# results == ["data from a", "data from b", "data from c"]
```

By default, if one of the awaited coroutines raises, `gather` propagates that exception —
but the *other* still-running tasks are **not** automatically cancelled; they keep running
in the background, and you generally lose track of them and their exceptions (they're only
retrievable if you kept references and don't await them again). `gather(...,
return_exceptions=True)` instead collects exceptions *as results* in the returned list,
which avoids this leak but pushes error handling onto you at the call site.

### `TaskGroup` (Python 3.11+)

```python
async def run_all() -> list[str]:
    results = []
    async with asyncio.TaskGroup() as tg:
        task_a = tg.create_task(fetch("a"))
        task_b = tg.create_task(fetch("b"))
        task_c = tg.create_task(fetch("c"))
    # __aexit__ waits for all tasks — every task is done by this point
    return [task_a.result(), task_b.result(), task_c.result()]
```

`TaskGroup` implements **structured concurrency**: every task created inside the `async
with` block is guaranteed to be finished (either completed or cancelled) by the time the
block exits — there's no way to accidentally leak a running task past the `with`. If any
task raises, `TaskGroup` automatically cancels every other task in the group and re-raises
once they've all wound down, wrapped in an `ExceptionGroup` (or a subclass, if all the
underlying exceptions share a type):

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(fetch("a"))
        tg.create_task(bad_fetch())  # raises ValueError
except* ValueError as eg:
    for exc in eg.exceptions:
        print("failed:", exc)
```

`except*` (PEP 654, also 3.11+) is the matching syntax for unpacking an `ExceptionGroup` —
a single `try`/`except` can't catch a group's individual exceptions directly.

!!! tip
    Prefer `TaskGroup` over `gather` for new code targeting 3.11+: it cancels siblings on
    failure instead of leaving them to run unsupervised, and it makes "all these tasks
    finish before we leave this scope" a structural guarantee instead of something you have
    to remember to enforce by hand. Reach for `gather` mainly when you're stuck on 3.10 or
    earlier, or specifically want `return_exceptions=True`'s "collect all results and
    errors together" behavior instead of fail-fast cancellation.

## Timeouts and Cancellation

```python
async def slow() -> str:
    await asyncio.sleep(10)
    return "done"


async def with_timeout() -> None:
    try:
        result = await asyncio.wait_for(slow(), timeout=2)
    except TimeoutError:
        print("gave up after 2s")
```

`asyncio.wait_for` races the coroutine against a timer; on timeout it cancels the inner task
and raises `TimeoutError`. Cancellation itself works by throwing `asyncio.CancelledError`
into the coroutine at its *next* `await` point — it isn't instantaneous, and it isn't
delivered if the coroutine is in the middle of a long synchronous stretch with no `await`.

Cleanup on cancellation belongs in `finally`, same as any other exception:

```python
async def worker(queue: asyncio.Queue) -> None:
    conn = await open_connection()
    try:
        while True:
            item = await queue.get()
            await process(conn, item)
    except asyncio.CancelledError:
        print("worker cancelled, cleaning up")
        raise  # re-raise — swallowing CancelledError breaks cancellation for callers
    finally:
        await conn.close()
```

!!! warning
    Never swallow `CancelledError` without re-raising it. `asyncio` (and structured
    concurrency features like `TaskGroup`) rely on cancellation actually propagating —
    catching it, logging it, and moving on silently turns a task that was supposed to stop
    into one that keeps running, and can make `TaskGroup`/`wait_for` hang waiting for a task
    that will never actually finish cancelling.

## Synchronization Primitives

Even though only one coroutine runs at a time, you still need coordination whenever a
sequence of `await`s isn't meant to interleave with another coroutine touching the same
state — the GIL doesn't help here, since the danger is losing control mid-*operation* at an
`await`, not simultaneous execution.

```python
lock = asyncio.Lock()

async def transfer(frm: Account, to: Account, amount: int) -> None:
    async with lock:
        frm.balance -= amount
        await db.save(frm)      # another coroutine could run here without the lock
        to.balance += amount
        await db.save(to)
```

**`asyncio.Semaphore`** caps concurrency — useful for capping in-flight requests to a rate-
limited API instead of firing hundreds of tasks at once:

```python
sem = asyncio.Semaphore(5)  # at most 5 concurrent

async def fetch_limited(url: str) -> str:
    async with sem:
        return await fetch(url)


async def main() -> None:
    async with asyncio.TaskGroup() as tg:
        for url in urls:  # even with 100 urls, at most 5 run at once
            tg.create_task(fetch_limited(url))
```

**`asyncio.Queue`** is the standard producer/consumer pattern — bounded queues also give you
natural backpressure, since `put()` blocks once the queue is full:

```python
async def producer(queue: asyncio.Queue) -> None:
    for item in range(10):
        await queue.put(item)
    await queue.put(None)  # sentinel: tells consumers to stop


async def consumer(queue: asyncio.Queue) -> None:
    while (item := await queue.get()) is not None:
        await process(item)
        queue.task_done()


async def main() -> None:
    queue: asyncio.Queue[int | None] = asyncio.Queue(maxsize=100)
    async with asyncio.TaskGroup() as tg:
        tg.create_task(producer(queue))
        tg.create_task(consumer(queue))
```

## Common Pitfalls

**Blocking the event loop with sync/CPU-bound calls.** The event loop is single-threaded —
a blocking call (`requests.get`, `time.sleep`, a synchronous DB driver, a tight CPU loop)
freezes every other coroutine on that loop until it returns, not just the one that made the
call. Offload it explicitly:

```python
# BAD: freezes the whole loop for the duration of the call
async def bad() -> None:
    time.sleep(2)

# GOOD: runs the blocking call in a threadpool, loop stays responsive
async def good() -> None:
    await asyncio.to_thread(time.sleep, 2)
```

See [FastAPI's Event Loop](../web-development/fastapi-event-loop.md) for exactly how this
plays out in a real web framework — it's the same underlying loop, and FastAPI's automatic
threadpool dispatch for `def` routes is doing precisely this `to_thread`-style offload for
you.

**Forgetting to `await` a coroutine.** Calling `fetch("a")` without `await` or
`create_task` just creates a coroutine object — nothing runs, and Python emits a
`RuntimeWarning: coroutine 'fetch' was never awaited` (easy to miss in noisy logs):

```python
async def main() -> None:
    fetch("a")           # BUG: coroutine created, never scheduled — does nothing
    await fetch("a")      # correct: actually runs it
```

**Fire-and-forget tasks getting garbage collected.** `asyncio.create_task` doesn't keep a
strong reference to the task anywhere except (weakly) inside the event loop's internals — if
you don't hold a reference yourself, the task can be garbage-collected mid-execution,
sometimes silently:

```python
# BUG: nothing holds a reference to this task after create_task returns
asyncio.create_task(log_event(event))

# FIX: keep a reference until the task is done
background_tasks: set[asyncio.Task] = set()

def spawn(coro) -> None:
    task = asyncio.create_task(coro)
    background_tasks.add(task)
    task.add_done_callback(background_tasks.discard)

spawn(log_event(event))
```

This exact pattern (a module-level set plus a done-callback to clean it up) is the standard
fix, and it's called out directly in the `asyncio.create_task` documentation for good
reason — it's a real, easy-to-hit bug, not a theoretical one.

## Summary

- The event loop runs one coroutine at a time; `await` is the cooperative yield point where
  control (and CPU time) can pass to something else.
- `create_task` schedules a coroutine to run concurrently immediately; plain `await` runs it
  to completion first, with no concurrency.
- Prefer `TaskGroup` over `gather` on 3.11+: it gives structured concurrency — sibling tasks
  are cancelled on failure and guaranteed finished by the time the block exits.
- Cancellation delivers `CancelledError` at the next `await`; always re-raise it after
  cleanup in `finally`, never swallow it.
- `Lock`, `Semaphore`, and `Queue` cover the standard coordination needs: mutual exclusion
  across `await` points, capping concurrency, and producer/consumer with backpressure.
- The three pitfalls that actually show up in real code: blocking the loop with sync calls,
  forgetting to `await` a coroutine, and losing fire-and-forget tasks to garbage collection.

## Related Articles

- [FastAPI's Event Loop](../web-development/fastapi-event-loop.md) — where FastAPI actually
  runs sync vs. async route code on this same event loop model; this article is the general
  asyncio patterns, that one is the concrete web-framework application.
- [Python Tips & Tricks](python-tips.md) — concurrency vs. parallelism, the GIL, threading
  vs. multiprocessing — the broader context asyncio's concurrency (not parallelism) model
  fits into.
- [Kafka Consumers Behind a FastAPI API on Kubernetes](../architecture/kafka-consumers-fastapi-kubernetes.md)
  — a concrete case for `asyncio.create_task` running a background consumer loop.
