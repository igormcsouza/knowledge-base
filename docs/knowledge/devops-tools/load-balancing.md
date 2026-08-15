---
tags:

- devops
- load-balancing
- networking
- scalability

---

# Load Balancing

A load balancer sits in front of multiple backend instances of a service and
distributes incoming traffic across them, for two related but distinct
reasons: **scalability** (no single instance has to handle all the traffic —
add more instances behind the balancer to handle more load) and
**availability** (an instance that goes down is simply routed around, so one
failure doesn't take the whole service down). Almost every production system
sits behind one somewhere, even when that fact is invisible day to day.

## Layer 4 vs. Layer 7

Load balancers are usually categorized by which OSI layer they operate at,
and the distinction has real practical consequences:

- **Layer 4 (transport)** — routes based on IP address and TCP/UDP port
  alone, without looking at the actual application data inside the
  connection. It's fast (minimal per-packet work) and protocol-agnostic (it
  works the same whether the payload is HTTP, a database protocol, or raw
  TCP), but it can't make routing decisions based on anything in the
  request itself — no path, no header, no cookie.
- **Layer 7 (application)** — terminates the connection, actually parses the
  HTTP request, and can route based on path (`/api` vs `/static`), host
  header, cookie value, or any other request content. This is what makes
  path-based routing to different backend services, TLS termination at the
  load balancer (so backend instances never see raw TLS at all), and
  content-based routing possible — at the cost of more per-request work
  than L4.

The practical rule of thumb: reach for L4 when the balancer just needs to
spread raw connections across identical backends as cheaply as possible
(a database connection pool, a generic TCP service); reach for L7 the moment
routing needs to know anything about the request itself — which is most
HTTP-based web traffic, hence why an HTTP(S) load balancer or an [Ingress
controller](kubernetes.md#ingress) is almost always L7.

## Algorithms

How a balancer picks *which* backend gets the next request/connection:

- **Round robin** — cycles through backends in order, one after another.
  Simple and works well when requests are roughly uniform in cost and
  backends are roughly equal in capacity.
- **Least connections** — sends the next request to whichever backend
  currently has the fewest active connections. Better than round robin when
  request cost varies a lot (a slow backend accumulating connections stops
  getting piled on further, since round robin has no idea it's already
  overloaded).
- **Weighted (round robin or least connections)** — backends get a
  proportional share of traffic based on an assigned weight, useful when
  backend instances have different capacity (e.g. a canary instance on
  smaller hardware getting a deliberately small slice of traffic) or during
  a gradual rollout.
- **IP hash / consistent hashing** — routes based on a hash of the client's
  IP (or another key), so the same client consistently lands on the same
  backend. This is the mechanism behind **session affinity** ("sticky
  sessions") — necessary when a backend holds request-relevant state locally
  (an in-memory session, a WebSocket connection) rather than in shared
  external storage.

!!! note "Sticky sessions are a workaround, not a design goal"
    Session affinity solves "this backend has state the request needs," but
    it also undermines the point of load balancing in the first place — a
    backend holding a disproportionate number of long sessions can't be
    evened out by the balancer without breaking those sessions, and taking
    that backend down for a deploy forces every one of its sticky clients to
    re-establish state elsewhere anyway. The more robust fix is usually
    externalizing session state (a shared cache/store) so *any* backend can
    serve *any* request — the same "don't rely on process-local state"
    principle that applies to
    [Kubernetes pods](kubernetes.md#self-healing) and
    [Lambda execution environments](aws/lambda.md#execution-environment-is-not-persistent-state).
    Sticky sessions are a reasonable pragmatic choice, just not a free one.

## Health Checks

A load balancer only helps availability if it actually stops sending traffic
to a backend that can't serve it — an unhealthy instance left in rotation is
worse than not load balancing at all, since some fraction of requests keep
failing until someone notices.

- **Active health checks** — the balancer itself periodically probes each
  backend (an HTTP `GET /health`, or a plain TCP connect) on its own
  schedule, independent of real traffic. A backend that fails enough
  consecutive checks is pulled out of rotation; one that starts passing
  again is added back. This is the same idea as a Kubernetes
  [readiness probe](kubernetes.md#self-healing), applied one layer further
  out.
- **Passive health checks** — the balancer watches real traffic outcomes
  (connection failures, timeouts, a run of 5xx responses) and ejects a
  backend based on those, without a separate dedicated probe. This reacts to
  failures active checks might not model well (e.g. an endpoint that's
  healthy for `/health` but erroring on a specific real route), at the cost
  of some real requests actually failing before the backend gets ejected.

Production setups commonly run both: active checks catch a backend that's
fully down before it's even sent real traffic, passive checks catch
degradation active checks didn't think to probe for. Whichever mechanism
catches it, pulling the backend out **automatically** is the point — a
health check that only feeds a dashboard for a human to act on is much
slower than a balancer that reacts on its own, and "automatically" is what
turns one instance failing into a non-event instead of an incident.

## Where This Shows Up in Practice

[Kubernetes Fundamentals](kubernetes.md) already covers the two places load
balancing shows up directly in a cluster:

- A [`Service` of type `LoadBalancer`](kubernetes.md#core-objects)
  provisions an external (typically L4, cloud-provider-managed) load
  balancer in front of a set of pods.
- [Ingress](kubernetes.md#ingress) puts one shared L7 load balancer in front
  of many services, routing by hostname/path instead of provisioning one
  balancer per service.

Both ultimately rely on the same readiness-probe mechanism described above
to know which pods are currently safe to send traffic to — a pod that's
failing its readiness probe is removed from a Service's routing the same
way an unhealthy backend is pulled from any other load balancer's rotation.

A dedicated load balancer isn't the only way to spread traffic across
multiple backends, though. **DNS-based load balancing** — returning multiple
A/AAAA records, or geo/latency-based routing at the DNS layer (Route 53
routing policies, for example) — spreads traffic across regions or data
centers without a single chokepoint node in the path at all. It's coarser
(DNS has no idea whether the record it just handed out is currently healthy
beyond periodic health-check-driven record removal, and clients/resolvers
cache answers for the record's TTL, so failover isn't instant) but it scales
to a scope a single load balancer instance can't reach, which is why it's
typically used *above* a tier of dedicated load balancers — DNS routing
clients to the nearest region, and a real L4/L7 balancer distributing load
within that region. This layering is conceptually similar to the "event
announced, multiple consumers react independently" decoupling in
[Event-Driven Architecture](../architecture/event-driven-architecture.md) —
both patterns exist to avoid a single point routing every single request
through one synchronous decision point.

## Summary

- A load balancer distributes traffic across backend instances for both
  scalability (more capacity) and availability (route around failures).
- Layer 4 balances on IP/port only — fast and protocol-agnostic; Layer 7
  parses the request itself, enabling path/header/cookie-based routing and
  TLS termination, at higher per-request cost.
- Round robin, least connections, weighted, and IP hash/consistent hashing
  are the standard algorithms — IP hash is what backs session affinity.
- Sticky sessions solve backend-local state but undermine even distribution;
  externalizing session state is usually the more robust fix.
- Active health checks probe backends on a schedule; passive health checks
  react to real traffic failures — production systems commonly use both,
  and the essential property is that ejection happens automatically.
- Kubernetes `Service`/`Ingress` are the in-cluster instance of this same
  pattern; DNS-based load balancing solves it at a coarser, cross-region
  scope above a tier of dedicated load balancers.

## Related Articles

- [Kubernetes Fundamentals](kubernetes.md) — `LoadBalancer` Services and
  Ingress, the in-cluster mechanics this article builds on rather than
  repeats.
- [AWS Lambda](aws/lambda.md) — execution-environment statelessness, the
  same principle behind why sticky sessions are a workaround rather than a
  design goal.
- [Event-Driven Architecture](../architecture/event-driven-architecture.md)
  — a different flavor of decoupling clients from a specific backend
  instance, contrasted here with DNS-based load balancing.
