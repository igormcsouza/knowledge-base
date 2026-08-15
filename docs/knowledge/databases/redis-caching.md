---
tags:

- databases
- redis
- caching
- performance

---

# Redis as a Caching Layer

Redis is an in-memory key/value store, and caching is its single most common
job: put a copy of expensive-to-compute or expensive-to-fetch data somewhere
that answers in microseconds instead of hitting a database, a slow join, or a
third-party API on every request. Getting the cache *design* right — keys,
TTLs, eviction, invalidation — matters more than the fact that it's Redis;
get it wrong and the cache becomes a second source of truth that silently
disagrees with the first one.

## Key Design and Namespacing

Redis is a single flat keyspace (per logical database), so a naming
convention is the only thing standing between an organized cache and a pile
of collisions. The standard pattern is colon-delimited namespacing:

```text
<domain>:<entity>:<id>[:<field>]

user:profile:42
user:session:8f3a...c2
order:items:9981
leaderboard:weekly:2026-08-11
```

This buys you two things: predictable `SCAN`-based debugging (`redis-cli
--scan --pattern 'user:session:*'`) and safe bulk invalidation by prefix
without needing a separate index of "which keys belong to this user."

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def user_profile_key(user_id: int) -> str:
    return f"user:profile:{user_id}"

def cache_user_profile(user_id: int, profile: dict, ttl_seconds: int = 3600) -> None:
    r.hset(user_profile_key(user_id), mapping=profile)
    r.expire(user_profile_key(user_id), ttl_seconds)
```

!!! warning "SCAN in production, not KEYS"
    `KEYS pattern` walks the entire keyspace in one blocking call — on a
    cache with millions of keys it can stall every other client for seconds.
    `SCAN` does the same walk incrementally with a cursor, so use it (or a
    client-side wrapper like `redis-py`'s `r.scan_iter()`) for anything that
    touches production.

## TTLs and Eviction Policies

Two separate mechanisms decide when a cached value goes away, and conflating
them is a common source of confusion.

**TTL (time-to-live)** is a per-key expiration you set explicitly —
`EXPIRE key seconds` or `SET key value EX seconds`. It's how you say "this
value is stale after N seconds," independent of memory pressure.

**Eviction policy** is what Redis does when it's hit `maxmemory` and needs to
free space *now*, configured server-wide via `maxmemory-policy`:

| Policy | Behavior |
|---|---|
| `noeviction` | Refuse new writes with an error once full. Default — usually wrong for a cache. |
| `allkeys-lru` | Evict the least-recently-used key, from any key, TTL or not. The standard choice for a pure cache. |
| `allkeys-lfu` | Evict the least-*frequently*-used key. Better than LRU when "popular" and "recent" diverge (e.g. a daily digest read once a day by everyone but not recently). |
| `volatile-lru` | LRU eviction, but only among keys that have a TTL set — keys without a TTL are never evicted, so they'd rather error than lose "permanent" data. |
| `volatile-ttl` | Evict the key with the shortest remaining TTL first — closest to "about to expire anyway." |
| `volatile-random` / `allkeys-random` | Evict a random key from the eligible set — cheap, unpredictable, rarely the best choice. |

```conf
# redis.conf — a dedicated cache instance
maxmemory 2gb
maxmemory-policy allkeys-lru
```

!!! tip "Pro Tip"
    If the same Redis instance mixes cached data with data you actually need
    to keep (session tokens, rate-limit counters, a job queue), use
    `volatile-lru` and set TTLs only on the truly-cache keys — that keeps
    eviction pressure from ever touching the keys you can't afford to lose.
    Better still: run a separate Redis instance (or logical DB) for cache vs.
    everything else, so a `FLUSHDB` or a maxmemory event on the cache can
    never touch real state.

## Cache-Aside, Write-Through, and Write-Behind

Three standard patterns for keeping a cache and the underlying database in
sync, each with a different trade-off between read speed, write speed, and
staleness risk.

**Cache-aside (lazy loading)** — the application checks the cache first; on a
miss, it reads from the database and populates the cache for next time. This
is the most common pattern because the cache stays optional: if Redis is
down or flushed, reads still work, just slower.

```python
import json

def get_product(product_id: int) -> dict:
    key = f"product:{product_id}"
    cached = r.get(key)
    if cached is not None:
        return json.loads(cached)

    product = db.query_product(product_id)  # cache miss — hit the DB
    r.set(key, json.dumps(product), ex=600)  # populate for next time
    return product
```

**Write-through** — every write goes to the cache *and* the database
synchronously, as one logical operation, before the write is considered
done. Reads are always fresh, but every write pays the latency of both
stores, and a cache that's down blocks writes too.

```python
def update_product_price(product_id: int, price_cents: int) -> None:
    db.update_product_price(product_id, price_cents)   # source of truth first
    r.set(f"product:{product_id}", json.dumps(db.get_product(product_id)), ex=600)
```

**Write-behind (write-back)** — writes go to the cache immediately and are
flushed to the database asynchronously, in the background, often batched.
This makes writes very fast, at the cost of a window where the database is
behind the cache — a crash before the flush loses that data unless the
buffer is itself durable (e.g. backed by a queue).

!!! note "Opinion"
    Write-behind is rarely worth the complexity outside of high-throughput
    counters or metrics pipelines that can tolerate losing a few seconds of
    data. For anything transactional, cache-aside plus a short TTL is almost
    always the better trade-off — it's simpler and it fails safe (stale, not
    wrong).

## Cache Stampede (Thundering Herd)

A cache stampede happens when a single hot key expires and a burst of
concurrent requests all miss the cache at the same instant — all of them then
hit the database simultaneously to recompute the same value. On a
high-traffic key this can turn one slow query into hundreds of identical
concurrent slow queries and take the database down, ironically *because* of
the cache, not despite it.

```python
# The naive cache-aside code above has this bug: if `product:42` expires
# right as 500 requests/sec are hitting the endpoint, all 500 see a miss
# and all 500 query the database in the same instant.
```

Three standard mitigations, usually combined:

**Locking (a.k.a. "cache stampede lock")** — the first request to miss
acquires a short-lived lock and recomputes; everyone else either waits
briefly and retries the cache, or gets served a slightly stale value if one
exists.

```python
import time

def get_product_locked(product_id: int) -> dict:
    key = f"product:{product_id}"
    cached = r.get(key)
    if cached is not None:
        return json.loads(cached)

    lock_key = f"lock:{key}"
    got_lock = r.set(lock_key, "1", nx=True, ex=10)  # SET NX EX — atomic
    if got_lock:
        try:
            product = db.query_product(product_id)
            r.set(key, json.dumps(product), ex=600)
            return product
        finally:
            r.delete(lock_key)
    else:
        time.sleep(0.05)
        cached = r.get(key)
        if cached is not None:
            return json.loads(cached)
        return db.query_product(product_id)  # fall back rather than loop forever
```

**Jitter** — instead of a fixed TTL, randomize it slightly (`600 ±
random(0, 60)` seconds) so that keys populated around the same time don't all
expire in the same instant, spreading the recompute load out.

```python
import random

ttl = 600 + random.randint(-60, 60)
r.set(key, json.dumps(product), ex=ttl)
```

**Probabilistic early expiration** — each read has a small, TTL-proportional
chance of recomputing the value *before* it actually expires, so recomputes
happen spread out ahead of the deadline instead of all bunching up exactly at
it (this is the idea behind the XFetch algorithm). A simplified version:

```python
import math
import random
import time

def get_with_early_recompute(key: str, ttl: int, beta: float = 1.0):
    cached = r.hgetall(key)  # store value + the time it was computed
    if cached:
        value, computed_at = cached["value"], float(cached["computed_at"])
        delta = time.time() - computed_at
        # Probability of early recompute grows as `delta` approaches `ttl`.
        if delta < ttl - beta * math.log(random.random()) * -1:
            return json.loads(value)

    value = db.query_product(key)  # recompute — either a true miss or "early"
    r.hset(key, mapping={"value": json.dumps(value), "computed_at": time.time()})
    r.expire(key, ttl)
    return value
```

!!! tip "Pro Tip"
    For a handful of *known* hot keys (a homepage banner, a global config
    blob), the simplest fix beats all three: never let them expire. Refresh
    them proactively on a schedule (a background job that re-populates the
    key every N seconds) instead of relying on TTL expiry plus reactive
    recomputation.

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation
> and naming things." — Phil Karlton

The reason it's hard isn't the mechanics of deleting a key — it's knowing
*exactly when* the cached value has become wrong, especially once more than
one code path can change the underlying data. Three practical strategies:

- **TTL-only ("just let it expire")** — the simplest approach: don't
  explicitly invalidate anything, just keep TTLs short enough that staleness
  is bounded and acceptable. Works well when a few seconds/minutes of
  staleness is fine (product listings, analytics dashboards).
- **Explicit invalidation on write** — when the source-of-truth write
  happens, delete (or update) the corresponding cache key in the same code
  path.

  ```python
  def update_product_price(product_id: int, price_cents: int) -> None:
      db.update_product_price(product_id, price_cents)
      r.delete(f"product:{product_id}")  # invalidate — next read repopulates
  ```

- **Namespaced/versioned keys** — instead of invalidating, change the key
  itself, so old cached values simply become unreachable and expire off on
  their own TTL:

  ```python
  def cache_key_for_catalog(version: int) -> str:
      return f"catalog:v{version}:all"

  # bump `version` in a small "current version" pointer key on any catalog
  # change; readers look up the pointer, then read the versioned key — old
  # versions just age out via TTL instead of needing a synchronous delete.
  ```

!!! warning "The trap: invalidation across services"
    Explicit invalidation only works if *every* write path that changes the
    data also knows to invalidate the cache. This breaks down fast in a
    system with multiple services writing to the same table, a bulk
    admin/backfill job, or a direct database migration — none of which go
    through the application code that owns the `r.delete(...)` call. Short
    TTLs as a safety net (even alongside explicit invalidation) are what
    keep a missed invalidation path from becoming permanent staleness.

## Redis Is More Than a Cache

Caching is the default reach for Redis, but its data structures make it a
reasonable fit for other short-lived, high-throughput state too — worth
knowing so "just use Redis" doesn't collapse into "just use string keys":

```python
# Hashes — a struct-like object without a round-trip-per-field cost.
r.hset("user:profile:42", mapping={"name": "Igor", "plan": "pro"})

# Sorted sets — O(log N) ranked inserts, perfect for leaderboards.
r.zadd("leaderboard:weekly", {"user:42": 1500, "user:7": 2100})
r.zrevrange("leaderboard:weekly", 0, 9, withscores=True)  # top 10

# Lists — a simple FIFO/LIFO queue via LPUSH/BRPOP.
r.lpush("jobs:email", json.dumps({"to": "a@example.com"}))
job = r.brpop("jobs:email", timeout=5)
```

These aren't caching at all — they're using Redis as the primary store for
data that's inherently short-lived or benefits from atomic, low-latency
structure operations. See
[Kafka Consumers Behind a FastAPI API on Kubernetes](../architecture/kafka-consumers-fastapi-kubernetes.md)
for a concrete example of Redis holding shared short-lived state across pods,
distinct from caching a database read.

## Summary

- Namespace keys (`domain:entity:id`) so bulk operations and debugging stay
  sane as the keyspace grows.
- TTL and eviction policy are different mechanisms — TTL says "this is
  stale," `maxmemory-policy` says "what to drop under memory pressure."
  `allkeys-lru` is the sane default for a pure cache.
- Cache-aside is the default pattern (fails safe); write-through keeps reads
  fresh at write-latency cost; write-behind is fast but risks losing data on
  a crash.
- A cache stampede is concurrent requests all missing on the same hot key at
  once — mitigate with a recompute lock, TTL jitter, or probabilistic early
  expiration.
- Cache invalidation is hard because *knowing when data changed* is hard,
  not because `DEL` is hard — explicit invalidation plus a TTL safety net
  covers the paths that miss the explicit call.
- Redis's richer structures (hashes, sorted sets, lists) make it useful for
  more than caching — leaderboards, queues, shared short-lived state.

## Related Articles

- [Idempotency](idempotency.md) — cache-aside repopulation and
  compare-and-swap-style safe writes share the same "make retries safe"
  instinct.
- [Query Performance Bottlenecks](query-performance-bottlenecks.md) — caching
  is one lever for slow reads; indexing and query shape are the others.
- [Kafka Consumers Behind a FastAPI API on Kubernetes](../architecture/kafka-consumers-fastapi-kubernetes.md)
  — Redis used for shared, short-lived state across horizontally scaled pods.
