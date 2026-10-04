---
tags:

- achievements
- python
- fastapi
- aws
- s3
- observability
- grafana
- performance

---

# Draftr: Cut a Slow Endpoint's p95 from ~2.3 s to ~0.24 s by Moving S3 Deletes to a Background Task

Used the telemetry FastAPI/OpenTelemetry gives almost for free, wired it into Grafana, and
spotted that one endpoint (`PUT /api/drawings/{id}/commit`) had a p95 around 2.3 s while
every other route was far faster. The cause was synchronous S3 `DeleteObject` calls on the
request path; moving them to a FastAPI `BackgroundTask` dropped that endpoint's p95 to
roughly 0.24 s.

- **Project:** [draftr](https://github.com/igormcsouza/draftr) — private Excalidraw
  workspace (Next.js + FastAPI on AWS serverless)
- **Date:** 2026-10-04
- **Role:** sole developer — instrumentation, diagnosis, fix

## Context

Draftr's backend is a FastAPI app on AWS Lambda (container image, behind the Lambda Web
Adapter running `uvicorn`), storing drawing scenes and thumbnails in S3 and metadata in
a database. Traces, metrics, and logs go over OTLP to Grafana Cloud.

`/commit` is the save endpoint: the client uploads the new scene (and thumbnail) to S3
with presigned URLs, then calls `/commit` to atomically point the drawing at the new
version. Once committed, the *previous* scene/thumbnail objects are no longer needed and
get deleted.

## Problem

After wiring the telemetry into Grafana I looked at p95 latency per route. Everything was
comfortably fast except `/commit`:

| Route | p95 (around deploy time) |
| --- | --- |
| `PUT /api/drawings/{id}/commit` | **~2.3 s** |
| `GET /api/drawings/{id}` | ~0.09 s |
| `GET /api/library` | ~0.04 s |
| `POST /api/drawings/{id}/upload-urls` | ~0.69 s |

(Pulled from Grafana Cloud Prometheus, `http_server_request_duration_seconds_bucket`.
Low traffic, so these are small samples — directionally right, not benchmark-grade.)

One outlier route is exactly what per-route percentiles are for: an average over all
endpoints would have hidden it.

## What I Did

Read the handler. After the database commit succeeded, it deleted the superseded objects
from S3 inline, before returning:

```python
# before: the client waits for S3 deletes it doesn't care about
blobs.delete(*(k for k in (old_scene, old_thumb) if k not in (body.scene_key, body.thumb_key)))
return {"version": version}
```

The client doesn't need the delete to finish to know the commit succeeded — the data is
already safe. So the delete is cleanup, not part of the answer. FastAPI's
`BackgroundTasks` runs a callable after the response is sent:

```python
@router.put("/drawings/{id}/commit")
def commit(id: str, body: s.Commit, bg: BackgroundTasks, ...):
    ...
    # after the response
    bg.add_task(blobs.delete, *(k for k in (old_scene, old_thumb) if k not in (body.scene_key, body.thumb_key)))
    return {"version": version}
```

Applied the same change to `set_thumb`, which had the identical pattern. Commit:
[`38704ed`](https://github.com/igormcsouza/draftr/commit/38704ed) ("Run commit/thumb S3
deletes after the response").

## Result

Same endpoint, p95 in Grafana (20-minute rate window, 2026-10-04):

- Before the change: **~2.3 s** (2.28–2.35 s)
- After the change: **~0.24 s**

Roughly a 10× improvement on the one slow endpoint, with a three-line diff per handler.

## Takeaways

- **Per-route percentiles find what averages hide.** One slow route among fast ones only
  shows up when you break latency down by route and look at p95, not the mean.
- **Ask whether the client needs the work to finish.** Cleanup, notifications, and cache
  invalidation usually don't belong on the response path.
- **Trade-off I accepted:** background deletes are best-effort. On Lambda the execution
  environment can be frozen after the response is returned, so a delete may be delayed or
  skipped, leaving an orphaned object. That's acceptable here because orphans are only
  wasted storage (S3 versioning plus a 7-day retention rule keeps recovery possible), but
  it would not be acceptable for work that must happen. For that, use a queue.
- **Interview version:** "I added observability, found one endpoint 5-10× slower than the
  rest from its p95, traced it to synchronous S3 deletes on the request path, and moved
  them off it — p95 went from ~2.3 s to ~0.24 s."

## Related Articles

- [Troubleshooting](../troubleshooting/index.md)
