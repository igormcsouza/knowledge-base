---
tags:

- web-development
- resilience
- performance
- api-design

---

# API Optimization & Resilience

Making an API fast and correct on the happy path is the easy 80%. The
remaining 20% — handling cold starts, failures, and retries without making
things worse — is what separates an API that degrades gracefully under real
conditions from one that cascades into an outage the first time something
upstream hiccups.

## Cold Starts

A **cold start** is the latency penalty paid when a serverless function or
container has to be freshly provisioned before it can handle a request —
initializing the runtime, running module-level import code — versus a
**warm** invocation that reuses an already-running instance. [AWS
Lambda](../devops-tools/aws/lambda.md) is the canonical example: the first
invocation (or the first after the function has been idle) pays this cost,
and it's directly visible as elevated p99 latency on otherwise-fast
endpoints.

**What causes it:**

- Provisioning a fresh execution environment (allocating resources, booting
  the runtime).
- Running all module-level initialization code — imports, client
  construction, config parsing — before the handler itself starts.
- Larger deployment packages take measurably longer to load into a fresh
  environment than small ones.

**Mitigations:**

- **Provisioned concurrency** — pay to keep a set number of execution
  environments pre-initialized and warm at all times, trading cost directly
  for eliminating cold starts on latency-sensitive paths. See [AWS
  Lambda](../devops-tools/aws/lambda.md#configuration-memory-timeout-concurrency)
  for the mechanism.
- **Keep deployment packages small** — every dependency bundled into the
  package is something that has to be loaded before the handler can run;
  trimming unused dependencies (and picking lighter alternatives where
  reasonable) shrinks that cost directly.
- **Lazy-import heavy dependencies** — move expensive imports (large ML
  libraries, heavy SDKs) inside the function that actually uses them instead
  of at module level, so a cold start only pays that import cost on the
  specific invocations that need it, not on every single one:

  ```python
  # paid on every cold start, even for invocations that never use it
  import heavy_ml_library

  def handler(event, context):
      if event.get("needs_ml"):
          return heavy_ml_library.predict(event["input"])
      return {"ok": True}
  ```

  ```python
  # paid only when actually needed
  def handler(event, context):
      if event.get("needs_ml"):
          import heavy_ml_library  # imported lazily, inside the branch that needs it
          return heavy_ml_library.predict(event["input"])
      return {"ok": True}
  ```

!!! note
    Lazy imports help cold-start latency but don't help *warm* invocations
    much either way (Python caches imported modules, so a second call pays
    almost nothing to "re-import"). The win is specifically for cold starts
    on code paths that don't always need the heavy dependency.

## Retry Strategies

Network calls fail — timeouts, transient 5xxs, connection resets — and a
client that doesn't retry at all is fragile, but a client that retries
naively can turn a brief blip into a self-inflicted outage.

**Exponential backoff with jitter.** Instead of retrying at a fixed
interval, each retry waits exponentially longer than the last, with random
jitter added:

```python
import random
import asyncio


async def call_with_retry(fn, max_retries: int = 5, base_delay: float = 0.5):
    for attempt in range(max_retries):
        try:
            return await fn()
        except TransientError:
            if attempt == max_retries - 1:
                raise
            backoff = base_delay * (2 ** attempt)
            jitter = random.uniform(0, backoff)  # spreads retries out in time
            await asyncio.sleep(jitter)
```

**Why jitter specifically matters**: without it, every client that failed
at the same moment (e.g. because the server briefly went down) retries at
exactly the same fixed intervals — `1s, 2s, 4s, 8s...` for every client in
lockstep — recreating the exact same synchronized burst of traffic on every
retry attempt, a **retry storm** that can keep a recovering service from
ever actually recovering. Randomizing the delay spreads those retries out
across time instead of concentrating them.

**Capping retries / circuit breaking.** Exponential backoff alone still
retries forever by default — bounding it matters just as much:

- **Max retry count** — give up after N attempts and surface the failure,
  rather than retrying indefinitely against a service that's genuinely down.
- **Circuit breaker** — track failure rate to a dependency, and once it
  crosses a threshold, stop calling it entirely for a cooldown period
  (failing fast instead) before allowing a small number of trial requests
  through to check if it's recovered. This protects a struggling downstream
  service from being kept down by a continuous stream of retries from every
  caller, and protects the *caller* from spending its own resources
  (threads, connections, latency budget) waiting on calls that are likely to
  fail anyway.

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, cooldown: float = 30.0):
        self.failure_threshold = failure_threshold
        self.cooldown = cooldown
        self.failures = 0
        self.opened_at: float | None = None

    def record_failure(self) -> None:
        self.failures += 1
        if self.failures >= self.failure_threshold:
            self.opened_at = time.monotonic()

    def record_success(self) -> None:
        self.failures = 0
        self.opened_at = None

    def allow_request(self) -> bool:
        if self.opened_at is None:
            return True
        if time.monotonic() - self.opened_at > self.cooldown:
            return True  # cooldown elapsed — allow a trial request through
        return False
```

!!! warning
    Retries and idempotency are two halves of the same coin: retrying a
    non-idempotent write (e.g. "create an order") without an idempotency
    mechanism turns a network blip into a duplicate side effect. Retry
    logic on its own only makes sense once the operation being retried is
    actually safe to repeat.

## Idempotency for Safe Retries

[Idempotency](../databases/idempotency.md) is covered in full depth
elsewhere — the core guarantee is that applying an operation once has the
same effect as applying it many times, which is exactly what makes "just
retry it" safe instead of dangerous.

At the API-client level specifically, the standard pattern (popularized by
Stripe's API) is a **client-generated idempotency key** sent on `POST`
requests — the one HTTP method with no idempotency guarantee by default:

```http
POST /orders HTTP/1.1
Idempotency-Key: 8f14e45f-ceea-467e-bd3d-cd3b0e6c9e5b
Content-Type: application/json

{"product_id": "sku_123", "quantity": 2}
```

The client generates one key per *logical* operation (not a new key per
retry attempt) and sends the same key on every retry of that same logical
request. The server records which keys it has already processed; on a
retried request with a key it's seen before, it returns the original result
instead of creating a second order. This is what makes automatic
retry-with-backoff safe to apply even to `POST` — the network layer can
retry freely, because the server-side idempotency check is what actually
prevents the duplicate effect, not any guarantee from the HTTP method
itself.

## Summary

- Cold starts come from provisioning a fresh execution environment and
  running module-level init code; provisioned concurrency, small deployment
  packages, and lazy-importing heavy dependencies are the standard
  mitigations.
- Exponential backoff spaces out retries over time; jitter is what actually
  prevents synchronized retry storms across many clients failing together.
- Cap retries with a max count and/or a circuit breaker so retries don't
  compound an outage against an already-struggling dependency.
- Retrying is only safe when the operation being retried is idempotent —
  client-generated `Idempotency-Key` headers on `POST` are the standard way
  to make retries safe at the API-client level.

## Related Articles

- [AWS Lambda](../devops-tools/aws/lambda.md) — the concrete environment
  where cold starts are most visible and most commonly optimized for.
- [Idempotency](../databases/idempotency.md) — the full explanation of why
  and how idempotent writes make retries safe.
- [API Pitfalls: Over-, Under-Fetching & N+1](api-pitfalls-n-plus-one.md) —
  performance problems on the response-shape side, complementary to the
  reliability concerns here.
- [WebSockets](websockets.md) — client-side reconnection with backoff is the
  same jitter/backoff pattern applied to a persistent connection instead of
  a single request.
