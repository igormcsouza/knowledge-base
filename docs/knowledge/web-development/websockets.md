---
tags:

- web-development
- websockets
- fastapi
- python
- async

---

# WebSockets

HTTP is request/response: the client asks, the server answers, the
connection's job for that exchange is done. WebSockets replace that with a
single **persistent, full-duplex** connection — either side can send a
message at any time, without waiting for the other to ask first. That
difference is exactly what makes WebSockets the right tool for some problems
and total overkill for others.

## The Handshake: HTTP Upgrade

A WebSocket connection starts as a normal HTTP request that asks to be
upgraded:

```http
GET /ws/chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

The `101 Switching Protocols` response is the pivot point: the same TCP
connection that carried the HTTP handshake now carries the WebSocket
protocol instead. This is why WebSockets work through existing HTTP
infrastructure (ports 80/443, most proxies and load balancers) without
needing a separate connection setup from scratch — but it also means an
intermediary that doesn't understand `Upgrade` can silently break it.

## Full-Duplex vs. Request/Response

| | HTTP | WebSocket |
|---|---|---|
| Who can initiate | Client only | Either side, any time |
| Connection lifetime | One request, then closed (or reused via keep-alive for the *next* request) | Held open for the session |
| Server push | Not native (needs polling, SSE, or long-polling workarounds) | Native |
| Overhead per message | Full HTTP headers each time | Minimal framing after the initial handshake |

The practical consequence: once a WebSocket is open, the server can push a
message the instant something happens, with no client request to trigger it
— that's the entire reason it exists.

## When WebSockets Are the Right Tool

WebSockets earn their complexity when the traffic is genuinely
**bidirectional and low-latency**:

- Chat applications — either party can send a message at any moment.
- Live dashboards with server-pushed updates (stock tickers, live metrics).
- Collaborative editing (multiple clients need to both send and receive
  changes continuously, in real time).
- Multiplayer/real-time gaming state sync.

## When Something Simpler Fits Better

**Server-Sent Events (SSE)** — one-directional, server → client only, over
plain HTTP (no upgrade handshake, no special protocol). The browser's
`EventSource` API handles reconnection automatically. Reach for SSE when the
client never needs to *send* anything after the initial request — a live
notification feed, streaming an LLM response token by token, a progress bar
for a long-running job:

```python
from fastapi.responses import StreamingResponse

@app.get("/events")
async def stream_events():
    async def event_generator():
        while True:
            data = await get_next_update()
            yield f"data: {data}\n\n"
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

SSE is simpler to implement, debug, and scale precisely because it's still
plain HTTP — existing infrastructure (proxies, load balancers, HTTP/2
multiplexing) already understands it, unlike the WebSocket upgrade.

**Polling** — the client just asks "anything new?" on an interval. Crude,
but genuinely the right choice when updates are infrequent and a few
seconds of staleness is fine — it needs zero special infrastructure and is
trivial to reason about, debug, and scale (it's just regular HTTP requests).

!!! tip "Pro Tip"
    Default to the simplest option that satisfies the actual requirement:
    polling if staleness of a few seconds is acceptable, SSE if only the
    server needs to push, WebSockets only once the client genuinely needs to
    push too. Each step up adds real operational complexity — don't reach
    for WebSockets just because "real-time" sounds like it wants them.

## A Minimal FastAPI WebSocket Example

FastAPI (via Starlette) supports WebSocket routes directly. Route handlers
run on the event loop the same way `async def` HTTP routes do — see
[FastAPI Event Loop](fastapi-event-loop.md) for what that means for blocking
calls inside the handler.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()
connected_clients: set[WebSocket] = set()


@app.websocket("/ws/chat")
async def chat_endpoint(websocket: WebSocket):
    await websocket.accept()
    connected_clients.add(websocket)
    try:
        while True:
            message = await websocket.receive_text()
            for client in connected_clients:
                await client.send_text(message)  # broadcast to everyone connected
    except WebSocketDisconnect:
        connected_clients.discard(websocket)
```

`receive_text()` suspends the coroutine (yielding control back to the event
loop, same as any other `await`) until the client sends something or
disconnects — it doesn't block other requests while it waits.

## Connection Lifecycle Management

A WebSocket looking "open" at the TCP level doesn't mean the other side is
still there — network failures, sleeping mobile devices, and silently
dropped intermediary connections all leave a socket that appears connected
but is actually dead.

**Heartbeats (ping/pong)** — periodically send a small control frame and
expect one back within a timeout; if it doesn't arrive, treat the
connection as dead and close it server-side, freeing the memory/resources
it was holding:

```python
import asyncio

async def heartbeat(websocket: WebSocket, interval: float = 30.0):
    while True:
        await asyncio.sleep(interval)
        try:
            await websocket.send_text('{"type": "ping"}')
        except Exception:
            break  # connection is gone — let the outer handler clean up
```

**Reconnection with backoff (client-side)** — a client should never
reconnect in a tight retry loop after a drop; use exponential backoff (see
[API Optimization & Resilience](api-optimization-resilience.md) for the
same pattern applied to plain HTTP retries) so a server restart doesn't get
hammered by every disconnected client reconnecting simultaneously the
instant it comes back up.

## The Scaling Problem

This is the part that catches teams off guard the first time they scale a
WebSocket service past one process. An HTTP request is stateless between
requests — any server instance behind a load balancer can handle the next
request from any client. A WebSocket connection is **stateful and pinned**:
once a client connects, that TCP connection — and any in-memory state tied
to it (which room it's in, its `WebSocket` object) — lives on exactly *one*
server process, for as long as the connection stays open.

That's fine for messages flowing between a client and the server it's
connected to. It breaks the moment you need to **broadcast** a message to
every connected client when those clients are spread across multiple server
instances — the process that receives the event to broadcast doesn't have
direct access to the sockets held open by its sibling instances:

```text
Client A ---- connected to ----> Server 1 (holds A's socket in memory)
Client B ---- connected to ----> Server 2 (holds B's socket in memory)

Server 1 gets a "new chat message" event meant for both A and B.
Server 1 can push it straight to A — but has no way to reach B's socket.
```

The fix is a **shared pub/sub layer** every server instance subscribes to
(Redis pub/sub is the common choice, though any message broker works): a
server that needs to broadcast publishes the message to a channel instead of
trying to reach other instances' sockets directly, and every instance —
including itself — receives it and forwards it to whichever locally-held
connections need it.

```python
import redis.asyncio as redis

async def broadcaster(app_connections: set[WebSocket]):
    r = redis.from_url("redis://localhost")
    pubsub = r.pubsub()
    await pubsub.subscribe("chat-broadcast")
    async for message in pubsub.listen():
        if message["type"] != "message":
            continue
        for ws in app_connections:  # only this process's own local connections
            await ws.send_text(message["data"])

# elsewhere, when any server instance needs to broadcast:
await r.publish("chat-broadcast", payload)
```

Every server instance runs the same broadcaster loop; publishing goes
through Redis instead of trying to reach sockets held by other processes,
and each instance only ever writes to the sockets it actually owns.

## Summary

- WebSockets upgrade an HTTP connection (`101 Switching Protocols`) into a
  persistent, full-duplex channel — either side can push at any time.
- Use WebSockets for genuinely bidirectional, low-latency traffic (chat,
  live dashboards, collaborative editing); prefer SSE for server-only push,
  and plain polling when some staleness is acceptable — each is
  meaningfully simpler to run.
- Heartbeats detect dead connections the TCP layer won't tell you about;
  clients should reconnect with backoff, not a tight retry loop.
- A WebSocket connection is pinned to one server process holding it in
  memory — broadcasting across multiple instances needs a shared pub/sub
  layer (e.g. Redis) to fan messages out between processes.

## Related Articles

- [FastAPI Event Loop](fastapi-event-loop.md) — how `async def` WebSocket
  handlers execute on the same event loop as HTTP routes, and why blocking
  calls inside one are just as dangerous.
- [API Optimization & Resilience](api-optimization-resilience.md) —
  exponential backoff with jitter, the same reconnection pattern applied to
  plain HTTP clients.
- [REST vs. GraphQL](rest-vs-graphql.md) — GraphQL subscriptions are
  typically transported over WebSockets.
