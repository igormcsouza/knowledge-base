---
tags:

- devops
- kubernetes
- containers
- orchestration

---

# Kubernetes Fundamentals

Kubernetes (K8s) orchestrates containers across a cluster of machines: it
decides where containers run, restarts them when they crash, scales them up
or down, and rolls out new versions — so "run this container, keep N copies
of it healthy" becomes a declarative instruction instead of something a
human babysits.

## Core Objects

- **Pod** — the smallest deployable unit. Usually one container (occasionally
  a small group that must share network/storage), with its own IP address.
  Pods are disposable — Kubernetes routinely kills and recreates them, so
  nothing important should live only inside one.
- **Deployment** — declares "keep N replicas of this pod spec running."
  Handles rolling updates (replace old pods with new ones gradually) and
  rollbacks (revert to a previous version) without downtime.
- **Service** — a stable network identity (a fixed name/IP) in front of a
  set of pods, since individual pod IPs change constantly as pods are
  recreated. Types include `ClusterIP` (internal-only), `NodePort`, and
  `LoadBalancer` (externally reachable, typically provisioning a cloud load
  balancer).
- **ConfigMap / Secret** — inject configuration and sensitive values into
  pods as environment variables or mounted files, decoupled from the
  container image itself.
- **Namespace** — a logical partition within a cluster, used to separate
  environments or teams sharing the same physical cluster.

## A Minimal Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myregistry/api:1.4.0
          ports:
            - containerPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP
```

`kubectl apply -f deployment.yaml` reconciles the cluster's actual state
toward this declared spec — Kubernetes continuously works to keep 3 healthy
`api` pods running, restarting or rescheduling any that fail.

## Scaling

- **Manual**: `kubectl scale deployment api --replicas=5`.
- **Horizontal Pod Autoscaler (HPA)**: automatically adjusts replica count
  based on observed CPU/memory usage or custom metrics, within a configured
  min/max range — the mechanism that actually makes "scale under load"
  hands-off.

This is **horizontal** scaling (more pods) — the kind Kubernetes makes
cheap and routine, as opposed to **vertical** scaling (a bigger pod with
more CPU/RAM). Horizontal scaling is also what introduces the "which pod has
my data" problem for anything holding state in memory — see
[Kafka Consumers Behind a FastAPI API on Kubernetes](../architecture/kafka-consumers-fastapi-kubernetes.md)
for that specific gotcha in depth.

## Self-Healing

- **Liveness probes** — periodic health checks; a pod that fails enough of
  them gets restarted.
- **Readiness probes** — a pod that fails these is removed from a Service's
  routing until it passes again, without being restarted — useful for
  "starting up, not ready for traffic yet."
- **Restart policy** — a crashed container in a pod is restarted
  automatically per the pod's restart policy (`Always` by default for pods
  managed by a Deployment).

Together these are what let a Deployment maintain "N healthy replicas" as an
ongoing guarantee rather than a one-time launch.

## Rolling Updates and Rollbacks

Updating a Deployment's image (`kubectl set image deployment/api
api=myregistry/api:1.5.0`) replaces pods gradually — new pods come up and
pass their readiness probe before old ones are terminated, so there's no gap
in capacity. If the new version is broken, `kubectl rollout undo
deployment/api` reverts to the previous ReplicaSet.

## `kubectl` Basics

```bash
kubectl get pods                    # list pods
kubectl describe pod <name>         # detailed status, recent events
kubectl logs -f <pod>               # follow a pod's logs
kubectl exec -it <pod> -- bash      # shell into a running pod
kubectl apply -f manifest.yaml      # create/update resources from a manifest
kubectl rollout status deployment/api  # watch a rollout progress
```

## Ingress

A `Service` of type `LoadBalancer` provisions one external load balancer per
service — expensive and wasteful when a cluster hosts many services. An
**Ingress** resource instead defines HTTP routing rules (by hostname/path)
in front of a single shared load balancer, fanning requests out to the right
internal Service — the standard way to expose multiple services under one
entry point.

## Summary

- Pods are ephemeral; Deployments keep a declared number of them healthy;
  Services give them a stable network identity.
- Horizontal scaling (more pods) is what Kubernetes is built for, and it's
  exactly what breaks assumptions like "state lives safely in this process's
  memory."
- Liveness/readiness probes plus a Deployment's restart policy are what
  make self-healing and zero-downtime rolling updates possible.
- Ingress consolidates routing to many services behind one external entry
  point instead of one load balancer per service.

## Related Articles

- [Docker Fundamentals](docker.md) — the container images Kubernetes
  actually runs.
- [Kafka Consumers Behind a FastAPI API on Kubernetes](../architecture/kafka-consumers-fastapi-kubernetes.md)
  — a concrete case study in what horizontal scaling breaks (pod-local
  state) and how to design around it.
