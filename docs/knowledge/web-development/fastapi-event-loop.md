---
tags:

- web-development
- fastapi
- python
- async
- event-loop

---

# FastAPI's Event Loop: Where Sync and Async Calls Go

FastAPI runs on Starlette, which runs on `asyncio`. Understanding exactly
*where* your code executes — directly on the event loop, or off in a
threadpool — is the difference between an app that scales and one that
mysteriously stalls under load.

## `async def` vs. `def` Routes

FastAPI supports both kinds of path operation functions, and they run in
different places:

```python
@app.get("/async-route")
async def async_route():
    ...  # runs directly on the event loop

@app.get("/sync-route")
def sync_route():
    ...  # runs in a threadpool, off the event loop
```

- **`async def` routes** run directly on the single event loop, alongside
  every other concurrent request. This is efficient *as long as* everything
  inside actually `await`s properly (async DB drivers, `httpx.AsyncClient`,
  etc.) and never blocks.
- **`def` (sync) routes** are automatically dispatched to a **threadpool**
  (via Starlette's `run_in_threadpool`, built on AnyIO) instead of running on
  the loop. This is deliberate: it lets you use blocking libraries (`requests`,
  synchronous SQLAlchemy, `time.sleep`) safely, because blocking a worker
  thread doesn't block the loop that's serving every other request.

## The Footgun: Blocking Code Inside `async def`

The event loop is **single-threaded**. If an `async def` route calls
something blocking — a synchronous HTTP call, a blocking DB driver, a
CPU-heavy loop, `time.sleep()` — it freezes the *entire* loop, meaning every
other concurrent request (even for other users, even unrelated routes) stalls
until that call returns.

```python
import requests
import time


@app.get("/bad")
async def bad_route():
    time.sleep(2)          # blocks the WHOLE event loop for 2s
    requests.get(SOME_URL)  # blocking I/O, same problem
    return {"ok": True}
```

```python
import httpx


@app.get("/good")
async def good_route():
    async with httpx.AsyncClient() as client:
        response = await client.get(SOME_URL)  # yields control back to the loop
    return response.json()
```

!!! warning
    This is one of the most common FastAPI performance bugs: reaching for a
    familiar blocking library (`requests`, `psycopg2`, `time.sleep`) inside
    an `async def` route "because it works in testing" — it works fine with
    one request at a time, then falls over under concurrent load because
    every request is now queued behind that one blocking call.

## Rule of Thumb

- Use **`async def`** when everything inside is genuinely async (async DB
  driver like `asyncpg`/async SQLAlchemy, `httpx.AsyncClient`, other
  `await`-able I/O).
- Use plain **`def`** when you need a blocking library — FastAPI's automatic
  threadpool dispatch handles it safely without extra code from you.
- Never mix: don't call blocking code from inside `async def` without
  explicitly offloading it (e.g. `await run_in_threadpool(blocking_fn)` or
  `asyncio.to_thread(blocking_fn)`).

## Background Tasks

`BackgroundTasks` runs a function *after* the response has been sent to the
client:

```python
from fastapi import BackgroundTasks


def send_notification(email: str, message: str) -> None:
    ...  # e.g. call an email provider


@app.post("/signup")
async def signup(email: str, background_tasks: BackgroundTasks):
    create_user(email)
    background_tasks.add_task(send_notification, email, "Welcome!")
    return {"status": "created"}
```

Important limits:

- It still runs **in the same worker process** as the request that scheduled
  it (in the threadpool if it's a sync function, on the loop if async) — it's
  not a separate service, and a crash of that worker loses pending background
  tasks.
- It's meant for short, best-effort work (sending a notification, writing a
  log, invalidating a cache) — not for CPU-heavy or long-running jobs, which
  would tie up the same worker resources you need for serving requests.

For real background/async job processing — retries, persistence across
restarts, distributed workers, scheduling — use a dedicated task queue like
**Celery**, **arq**, or **RQ** instead, backed by Redis or a broker, running
in its own process(es) entirely separate from the web workers.

## Summary

- `async def` routes run on the event loop; `def` routes run in a threadpool
  automatically.
- Never call blocking code inside `async def` without offloading it — it
  stalls every concurrent request, not just the current one.
- `BackgroundTasks` is for small post-response work in the same process, not
  a substitute for a real task queue.
- For heavy or durable background work, reach for Celery/arq/RQ.

## Related Articles

- [DDD & the Service Layer](../architecture/ddd-service-layer.md) — where
  the actual logic called from a route usually lives.
- [Python Tips & Tricks](../python/python-tips.md) — threading vs.
  multiprocessing vs. the GIL, useful background for why this threadpool
  dispatch works the way it does.
- [Kafka Consumers Behind a FastAPI API on Kubernetes](../architecture/kafka-consumers-fastapi-kubernetes.md)
  — a concrete case for `asyncio.create_task` plus the lifespan handler:
  running a background consumer loop alongside the API.
