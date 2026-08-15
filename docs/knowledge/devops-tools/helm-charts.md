---
tags:

- devops
- kubernetes
- helm
- containers

---

# Helm for Kubernetes

Helm is the package manager for Kubernetes. This article assumes the
Deployment/Service/ConfigMap-level basics from
[Kubernetes Fundamentals](kubernetes.md) and focuses on what Helm adds on
top of raw manifests: templating, packaging, versioned releases, and
dependency management between charts.

## What Helm Solves

A single environment's Kubernetes setup is rarely one manifest — it's a
Deployment, a Service, a ConfigMap, a Secret, maybe an Ingress and an HPA,
all related and usually applied together. Hand-editing that set of YAML
files per environment (dev, staging, prod) quickly runs into the same
problems that motivate config templating anywhere else:

- **Duplication** — the dev and prod manifests are 95% identical; the only
  real differences are replica count, image tag, and resource limits, but
  copy-pasted YAML means every shared field has to be edited N times.
- **No versioning of "the whole set together"** — `kubectl apply -f` doesn't
  track what was previously deployed as a unit, so there's no built-in
  "roll back this entire release to what was running before."
- **No parameterization** — raw manifests are static text; there's no clean
  way to say "same template, different values per environment" without a
  separate templating layer bolted on (`sed`, Kustomize overlays, etc.).

Helm packages a set of Kubernetes resources as one versioned,
parameterizable unit called a **chart**, templated from a single set of
YAML files plus a values file that varies per install.

## Chart Structure

```text
my-app/
├── Chart.yaml          # chart metadata: name, version, dependencies
├── values.yaml         # default values referenced by the templates
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl    # reusable template snippets (named templates)
└── charts/              # vendored subcharts (dependencies), if any
```

- **`Chart.yaml`** — the chart's identity: name, chart version, the version
  of the application it deploys, and any chart dependencies.
- **`values.yaml`** — the default configuration values every template in
  `templates/` reads from. This is the file environment-specific overrides
  replace or extend.
- **`templates/`** — Kubernetes manifests written as Go templates, with
  `{{ }}` placeholders substituted at render time.
- **`_helpers.tpl`** — not a manifest itself (the leading underscore tells
  Helm to skip rendering it as a resource); it defines **named templates**
  (reusable snippets like a standard set of labels) that other templates
  call with `{{ include "my-app.labels" . }}`, keeping common boilerplate in
  one place instead of repeated across `deployment.yaml`, `service.yaml`,
  etc.

## Templating Basics

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-api
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          {{- if .Values.resources }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- end }}
```

```yaml
# values.yaml
replicaCount: 2
image:
  repository: myregistry/api
  tag: "1.4.0"
service:
  port: 8000
resources: {}
```

- `.Values.x` reads from `values.yaml` (or whatever overrides it at install
  time); `.Release.Name` is a built-in object giving the release's name;
  `.Chart` and `.Files` are similarly built in.
- `{{- if .Values.resources }} ... {{- end }}` is a conditional — the
  `resources:` block is only rendered at all if a value was actually
  supplied, avoiding an empty `resources: {}` cluttering every rendered
  manifest.
- `{{- range .Values.envVars }} ... {{- end }}` loops over a list or map in
  `values.yaml` — the standard way to template a variable-length list of
  environment variables, ports, or volume mounts without hardcoding a fixed
  number of them in the template.
- **Whitespace control**: `{{-` and `-}}` trim the newline/whitespace
  immediately before or after the tag. Templates without them tend to render
  ragged blank lines and stray indentation wherever a directive was — Go
  templates don't strip anything on their own, so `{{- if ... }}` (trimming
  before) and `{{ ... -}}` (trimming after) are the normal way to write
  conditionals and loops, not an occasional nicety.

```bash
helm template ./my-app -f values-prod.yaml   # render without installing — inspect output
helm lint ./my-app                            # catch template/schema mistakes early
```

!!! tip "Render before you apply"
    `helm template` is the single most useful debugging command when a
    chart isn't producing the manifest you expect — it renders the final
    YAML to stdout with no cluster interaction at all, which is much faster
    to iterate on than `helm install --dry-run` against a real cluster.

## Overriding Values Per Environment

`values.yaml` holds the defaults; overrides layer on top at install time,
last one wins:

```bash
# a whole file of prod-specific overrides
helm install my-app ./my-app -f values-prod.yaml

# one-off overrides on the command line, dot notation for nested keys
helm upgrade my-app ./my-app --set replicaCount=5 --set image.tag=1.5.0

# combine: file first, then --set on top of it
helm upgrade my-app ./my-app -f values-prod.yaml --set image.tag=1.5.0
```

```yaml
# values-prod.yaml — only the keys that differ from values.yaml
replicaCount: 5
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

`values-prod.yaml` doesn't need to repeat every key from `values.yaml` — it
only needs the ones that differ; Helm merges it over the chart's defaults.
This is the same "one template, N parameter sets" idea as the Kubernetes
Deployment/Service split in [Kubernetes Fundamentals](kubernetes.md), just
one layer up: the chart is the template, `values-*.yaml` is the
per-environment parameterization.

## Releases and Rollbacks

Every `helm install`/`helm upgrade` creates a numbered **release
revision**, tracked by Helm independently of `kubectl`:

```bash
helm install my-app ./my-app -f values-prod.yaml   # first install → revision 1
helm upgrade my-app ./my-app --set image.tag=1.5.0  # revision 2
helm history my-app                                  # list all revisions
helm rollback my-app 1                               # revert to revision 1
```

This is the versioning gap raw manifests leave open: `helm rollback` reverts
*the entire set of resources* to exactly what was rendered for a previous
revision, in one command — the chart-level equivalent of a Deployment's
`kubectl rollout undo`, but covering every resource the chart manages, not
just one Deployment's ReplicaSet.

!!! warning "Rollback reverts Helm's records, not necessarily live drift"
    If something was changed directly with `kubectl edit` outside of Helm
    (a debugging patch, a manual scale) and never reflected back into a
    chart value, `helm rollback` won't know about it — Helm reasons about
    the state it rendered, not the cluster's true live state. Treat
    `kubectl edit` on a Helm-managed resource as a temporary, "someone will
    undo this or the next `helm upgrade` overwrites it" action, not a
    persistent change.

## Dependencies Between Charts

A chart can depend on other charts — **subcharts** — declared in
`Chart.yaml`:

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
version: 1.0.0
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
helm dependency update ./my-app   # fetches subcharts into charts/
```

`condition: redis.enabled` ties the subchart to a boolean in
`values.yaml`, so an environment that already has an external Redis can
disable the bundled one (`redis.enabled: false`) instead of forking the
chart. A parent chart's `values.yaml` can also pass values down into a
subchart by nesting them under the subchart's name:

```yaml
# my-app's values.yaml, configuring the redis subchart
redis:
  enabled: true
  auth:
    enabled: false
```

### Umbrella Charts

An **umbrella chart** takes this a step further: a top-level chart with
little or no templates of its own, whose entire purpose is to declare
several application charts as dependencies and install them together as one
release — e.g. one umbrella chart pulling in `api`, `worker`, and `redis`
subcharts so a whole system's worth of services deploys, upgrades, and rolls
back as a single unit. This trades some flexibility (harder to release one
service's chart independently of the others) for the guarantee that
everything in the umbrella is always deployed as a consistent, versioned
set — useful when several services are tightly coupled enough that they
should never be upgraded out of sync with each other.

## Summary

- Helm packages a set of Kubernetes manifests as one versioned,
  parameterizable chart instead of hand-edited YAML per environment.
- `Chart.yaml` is metadata + dependencies, `values.yaml` is defaults,
  `templates/` is the Go-templated manifests, `_helpers.tpl` holds reusable
  named-template snippets.
- `{{ .Values.x }}`, `if`/`range`, and `{{-`/`-}}` whitespace trimming are
  the templating fundamentals — `helm template` renders output locally
  without touching a cluster, the fastest way to debug a template.
- Environment differences live in override files (`-f values-prod.yaml`) or
  one-off `--set` flags layered on top of the chart's defaults.
- `helm install`/`upgrade` create numbered release revisions; `helm
  rollback` reverts the entire release to a prior revision in one command —
  but only for what Helm itself tracks, not manual out-of-band changes.
- Chart dependencies (subcharts) let one chart bundle others; an umbrella
  chart takes this further, deploying a whole set of related services as one
  release.

## Related Articles

- [Kubernetes Fundamentals](kubernetes.md) — the Deployment/Service/Ingress
  objects Helm charts are templating on top of.
- [Docker Fundamentals](docker.md) — the container images referenced by
  `image.repository`/`image.tag` in a chart's values.
- [GitHub Actions & CI](github-actions-ci.md) — where `helm upgrade` (or
  `helm template` for validation) typically runs as a deploy step.
