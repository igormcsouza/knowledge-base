---
tags:

- devops
- docker
- containers
- tooling

---

# Docker Fundamentals

Docker packages an application together with everything it needs to run —
code, runtime, libraries, system tools — into a single, portable
**container**. The same container runs the same way on a laptop, in CI, and
in production, because it carries its own environment instead of depending
on whatever happens to be installed on the host.

## Images vs. Containers

An **image** is a read-only template: a filesystem snapshot plus metadata
(entrypoint, exposed ports, env defaults). A **container** is a running
instance of an image, with its own writable layer on top. One image can back
many running containers, each isolated from the others.

## The Dockerfile

Images are built from a `Dockerfile` — a sequence of instructions, each
producing a cached **layer**:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Layer order matters for build speed: instructions that change rarely
(installing dependencies) should come before instructions that change often
(copying application code). With the order above, editing application code
only invalidates the `COPY . .` layer and everything after it — the
dependency-install layer stays cached.

## Multi-Stage Builds

A multi-stage build uses one stage to build/compile, and a separate, smaller
stage to actually ship — keeping build tools and intermediate artifacts out
of the final image:

```dockerfile
FROM node:20 AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

The final image only contains `nginx` and the built static files — no
Node.js, no `node_modules`, no source files.

## Core Commands

```bash
docker build -t myapp:latest .        # build an image from a Dockerfile
docker run -p 8000:8000 myapp:latest  # run a container, map host:container port
docker ps                              # list running containers
docker logs -f <container>             # follow a container's logs
docker exec -it <container> bash       # shell into a running container
docker stop <container>                # stop it
docker rm <container>                  # remove a stopped container
```

## Volumes and Networking

- **Volumes** (`docker volume create`, or `-v myvolume:/data`) persist data
  outside the container's writable layer, surviving container removal —
  needed for anything that shouldn't disappear when the container restarts.
- **Bind mounts** (`-v $(pwd):/app`) map a host directory directly into the
  container — the standard way to get live code reload during local
  development.
- **Port mapping** (`-p host_port:container_port`) is what makes a
  container's internal port reachable from outside; without it, the port is
  only reachable from other containers on the same Docker network.

## docker-compose for Local Multi-Container Setups

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: devpassword
```

`docker compose up` builds/starts every service defined, wires them onto a
shared network (so `api` can reach `db` by hostname), and `docker compose
down` tears the whole thing back down — the standard way to spin up a local
stack that mirrors production's shape without installing Postgres,
Redis, etc. directly on the host.

## Best Practices

- Prefer small base images (`-slim`, `-alpine`) — smaller attack surface,
  faster pulls/pushes.
- Add a `.dockerignore` (same idea as `.gitignore`) so `COPY . .` doesn't
  drag in `.git/`, `node_modules/`, virtualenvs, or secrets.
- Run as a non-root user in the final image where possible.
- Tag images meaningfully (a commit SHA or semantic version, not just
  `latest`) so a running container's exact contents are traceable.
- Push built images to a registry (Docker Hub, or a cloud provider's — e.g.
  [ECR](aws/lambda.md) on AWS) so they can be pulled by CI, Kubernetes, or
  any other environment that needs them.

!!! tip "Pro Tip"
    `docker build` layer caching is invalidated by the *first* changed
    instruction in the file — everything after it rebuilds even if
    unchanged. Structuring a Dockerfile from "changes rarely" to "changes
    often" is the single biggest lever for fast rebuilds.

## Summary

- An image is a template; a container is a running instance of one.
- Order Dockerfile instructions from least- to most-frequently-changing to
  maximize layer cache hits.
- Multi-stage builds keep build-only tooling out of the shipped image.
- Volumes/bind mounts persist or share data beyond a container's own
  writable layer; port mapping is what exposes a container to the host.
- `docker compose` is the standard tool for a local multi-service stack.

## Related Articles

- [Kubernetes Fundamentals](kubernetes.md) — the orchestrator that runs many
  containers like these across a cluster, handling scaling and healing.
