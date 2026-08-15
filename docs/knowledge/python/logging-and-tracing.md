---
tags:

- python
- logging
- tracing
- observability

---

# Logging & Tracing in Python

`print()` debugging doesn't survive contact with a real deployed system — you need logs
that can be filtered by severity and module, correlated across concurrent requests, and
shipped somewhere searchable. This article covers the standard `logging` module properly
configured, propagating request-scoped context with `contextvars`, and how distributed
tracing (spans, trace IDs, OpenTelemetry) connects logs across service boundaries.

## The `logging` Module: Loggers, Handlers, Formatters

Four pieces, each with a distinct job:

- **Logger** — the object you call `.info()`/`.warning()`/etc. on; it decides *whether* a
  message is worth processing, based on its level.
- **Handler** — decides *where* a log record goes (stdout, a file, a network sink). A
  logger can have multiple handlers, e.g. one for the console, one shipping to a log
  aggregator.
- **Formatter** — decides *how* a record is rendered to text (or JSON) before a handler
  writes it out.
- **Filter** — optional, fine-grained inclusion/exclusion logic beyond just level.

```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

handler = logging.StreamHandler()
formatter = logging.Formatter(
    "%(asctime)s %(levelname)s %(name)s %(message)s"
)
handler.setFormatter(formatter)
logger.addHandler(handler)

logger.info("user %s logged in", user_id)
```

!!! tip "Always pass args, don't f-string the message"
    `logger.info("user %s logged in", user_id)`, not
    `logger.info(f"user {user_id} logged in")`. The f-string version formats the string
    *every time*, even if the logger would discard it (e.g. `DEBUG` messages when the level
    is `INFO`). The `%s`-style call only formats if the record actually gets emitted — cheap
    at scale, and it also means log aggregators that parse the raw template
    (`"user %s logged in"`) can group these into one metric instead of treating every
    interpolated value as a distinct message.

### Why `getLogger(__name__)`, Not the Root Logger

```python
# in myapp/orders/service.py
logger = logging.getLogger(__name__)  # name becomes "myapp.orders.service"
```

`__name__` gives each module its own logger, named after its dotted module path — this
creates a **hierarchy** that mirrors your package structure (`myapp` → `myapp.orders` →
`myapp.orders.service`). Two things fall out of this that calling `logging.info(...)`
directly on the root logger loses entirely:

- **Per-module level control.** You can set `myapp.orders` to `DEBUG` while leaving
  everything else at `INFO`, to investigate one subsystem without drowning in noise from
  the rest of the app:

  ```python
  logging.getLogger("myapp.orders").setLevel(logging.DEBUG)
  ```

- **Traceable origin.** The formatter's `%(name)s` tells you exactly which module emitted a
  given line — invaluable once an app has more than a handful of files, and much more
  reliable than grepping message text for context.

Child loggers **propagate** records up to their ancestors' handlers by default (`logger.
propagate = True`), which is why you typically configure handlers once, near the root or
your app's top-level logger, rather than attaching a handler in every module — modules just
call `getLogger(__name__)` and log; the handler(s) attached higher up do the actual output.

!!! warning
    Calling the module-level convenience functions (`logging.info(...)`, `logging.debug
    (...)`) implicitly uses the **root logger**, and — worse — the first such call
    auto-configures the root logger with a default handler via `logging.basicConfig()`.
    In a library, this can silently clobber configuration an application expects to set up
    itself. Libraries should log via `getLogger(__name__)` only, and never call
    `basicConfig` — configuring handlers/levels is the *application's* job, not a
    dependency's.

## Structured / JSON Logging

Plain-text logs are fine to read in a terminal but painful to query at scale. Most log
aggregation systems (Datadog, ELK, CloudWatch Logs Insights) work far better against
structured, one-JSON-object-per-line output. See
[Amazon CloudWatch](../devops-tools/aws/cloudwatch.md) for a concrete example of a log
aggregation backend that structured, JSON-formatted logs plug directly into.

```python
import json
import logging


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        if hasattr(record, "request_id"):
            payload["request_id"] = record.request_id
        if record.exc_info:
            payload["exception"] = self.formatException(record.exc_info)
        return json.dumps(payload)


handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logging.getLogger().addHandler(handler)
```

In practice, reach for a maintained library (`python-json-logger`, `structlog`) rather than
hand-rolling this — they handle edge cases like non-serializable extra fields and exception
formatting more robustly. The principle is the same either way: emit key-value structured
data, not a free-text sentence, so the aggregator can filter/aggregate on fields directly
(`level:ERROR AND request_id:abc123`) instead of regexing message strings.

### `logging.config.dictConfig`

For anything beyond a couple of handlers, configure logging declaratively instead of
imperative `addHandler` calls scattered around startup code:

```python
import logging.config

LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "json": {"()": "myapp.logging.JsonFormatter"},
    },
    "handlers": {
        "console": {
            "class": "logging.StreamHandler",
            "formatter": "json",
            "level": "INFO",
        },
    },
    "loggers": {
        "myapp": {"handlers": ["console"], "level": "INFO", "propagate": False},
        "myapp.orders": {"level": "DEBUG"},  # more verbose for this subsystem only
    },
}

logging.config.dictConfig(LOGGING_CONFIG)
```

This is also what makes it easy to load different logging configs per environment (verbose
and pretty-printed locally, structured JSON at `INFO` in production) from a config file
instead of branching imperative setup code.

## Context Propagation with `contextvars`

Request-scoped data — a request ID, the authenticated user, a trace ID — needs to reach
every log line emitted while handling that request, without threading a parameter through
every single function call. The old answer was `threading.local()`; it doesn't work for
async code, and `contextvars` is the fix.

**Why thread-locals fail across `async`/`await`.** A single OS thread can be running many
concurrently-interleaved coroutines (that's the entire point of asyncio) — a
`threading.local()` value is shared by *all* of them, since they're all on the same thread.
Set it for request A, then `await` something that lets the loop switch to request B's
coroutine, and B now sees A's thread-local value (or overwrites it out from under A). It's a
correctness bug that only shows up under concurrent load, which makes it especially nasty.

`contextvars.ContextVar` solves this because `asyncio` copies the current `Context` into
each `Task` when it's created — each task effectively gets its own isolated snapshot of
context variables, correctly scoped to the coroutine and everything it awaits, immune to
interleaving with other tasks on the same thread:

```python
import contextvars
import logging

request_id_var: contextvars.ContextVar[str] = contextvars.ContextVar(
    "request_id", default="-"
)


class RequestIdFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id_var.get()
        return True


logger = logging.getLogger(__name__)
logger.addFilter(RequestIdFilter())

formatter = logging.Formatter("%(asctime)s [%(request_id)s] %(message)s")
```

```python
import uuid


async def handle_request(request) -> None:
    token = request_id_var.set(str(uuid.uuid4()))
    try:
        logger.info("handling request")  # includes this request's ID automatically
        await process(request)           # anything awaited here still sees the same value
    finally:
        request_id_var.reset(token)      # restore prior value, avoid leaking across tasks
```

Every log line emitted anywhere in the call stack during `handle_request` — without passing
`request_id` as an explicit parameter anywhere — picks it up via the filter. This is exactly
the mechanism most web frameworks' request-ID middleware is built on under the hood.

!!! note
    Always pair `ContextVar.set()` with `.reset(token)` in a `finally`. Without it, a value
    set in one task can leak into whatever reuses that context afterward (thread pools reuse
    workers; some server frameworks reuse contexts between requests on the same connection).

## Distributed Tracing Basics

Logging tells you *what happened in one process*; tracing tells you *how a single request
moved across multiple services*. The core vocabulary:

- **Trace ID** — one ID generated at the very start of a request (e.g. at the edge/gateway),
  and propagated to every downstream service that handles it. Everything with the same trace
  ID belongs to the same end-to-end request.
- **Span** — one unit of work within that trace (an HTTP call, a DB query, a function).
  Spans have a start/end time and their own **span ID**, and reference a **parent span ID**,
  building a tree that shows exactly how time was spent and which call triggered which.
- **Context propagation** — the trace ID (and current span ID, as the new parent) travels
  with the request across process boundaries, typically as HTTP headers
  (`traceparent`, per the [W3C Trace Context](https://www.w3.org/TR/trace-context/)
  standard) or message-broker metadata for async messaging.

### OpenTelemetry

OpenTelemetry (OTel) is the standard instrumentation layer for this — vendor-neutral APIs
for creating spans and exporting them to whatever backend you use (Jaeger, Tempo, Datadog,
etc.):

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)


async def process_order(order_id: str) -> None:
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        await charge_payment(order_id)   # can create its own child span
        await update_inventory(order_id)
```

Most frameworks and libraries have OTel auto-instrumentation packages (`opentelemetry-
instrumentation-fastapi`, `-requests`, `-sqlalchemy`) that create spans automatically for
inbound requests, outbound HTTP calls, and DB queries — you mainly reach for manual spans
(as above) around your own significant business logic.

### Correlating Logs with Traces

The payoff of doing both is being able to jump from "this trace was slow" to "here are the
exact log lines from every service involved" — which requires the trace ID to show up in
both systems. The standard pattern is to pull the active span's trace ID into the same
`contextvars`-based logging context described above:

```python
from opentelemetry import trace


class TraceIdFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        span = trace.get_current_span()
        ctx = span.get_span_context()
        record.trace_id = format(ctx.trace_id, "032x") if ctx.is_valid else "-"
        return True
```

Now every log line emitted while a span is active carries that span's trace ID, and a log
aggregator and a tracing backend that both index on `trace_id` let you pivot directly
between "the slow span" and "the logs printed during it," even when those logs came from a
different service than the one that started the trace.

## Summary

- Use `logging.getLogger(__name__)` per module, not the root logger — it gives you a
  hierarchy for per-module level control and traceable origins, and libraries especially
  should never touch the root logger or call `basicConfig`.
- Prefer `%s`-style lazy formatting over f-strings in log calls, and structured/JSON output
  over free text once logs are shipped to an aggregator.
- `contextvars.ContextVar` (not `threading.local`) is what correctly propagates
  request-scoped data like a request ID across `async`/`await`, because asyncio copies
  context per-task instead of sharing it thread-wide.
- Distributed tracing adds cross-service visibility that single-process logs can't: a trace
  ID ties a whole request together, spans show where time went, and OpenTelemetry is the
  standard instrumentation layer for producing both.
- Propagate the trace ID into your logging context so logs and traces can be correlated —
  that's what actually makes both systems useful together instead of two disconnected views.

## Related Articles

- [Designing Concurrent Tasks with asyncio](asyncio-concurrency.md) — the `contextvars`
  behavior here relies directly on how asyncio schedules and copies context per task.
- [FastAPI's Event Loop](../web-development/fastapi-event-loop.md) — a concrete framework
  where request-scoped context (and OTel auto-instrumentation) attaches per request.
- [Amazon CloudWatch](../devops-tools/aws/cloudwatch.md) — a concrete log aggregation and
  alarming backend that structured JSON logs like these are built to feed.
- [Python Tips & Tricks](python-tips.md) — general Python idioms and gotchas.
