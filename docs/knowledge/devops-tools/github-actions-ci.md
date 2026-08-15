---
tags:

- devops
- ci
- github-actions
- tools

---

# CI with GitHub Actions

Continuous Integration (CI) is the practice of automatically building,
linting, and testing every change before it merges, so breakage is caught by
a machine in minutes instead of by a teammate (or production) days later.
The value isn't the tooling — it's the feedback loop: the faster a broken
change is flagged, the cheaper it is to fix, both because the author still
has full context and because nobody else has built on top of the breakage
yet. GitHub Actions is GitHub's built-in CI/CD system, and this article
covers its mechanics alongside those general CI concepts.

## Workflow File Structure

A workflow is a YAML file under `.github/workflows/`; GitHub picks up every
file in that directory automatically:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch: {}
```

- **`push`** — runs on commits pushed to matching branches. Scoping it to
  `main` (rather than every branch) avoids running the full pipeline on
  every push to a feature branch, where a PR trigger usually already covers
  it.
- **`pull_request`** — runs when a PR is opened or updated against the
  target branch; this is what actually gates merging, especially combined
  with a branch protection rule requiring the check to pass.
- **`schedule`** — cron-based, for periodic runs unrelated to a code change
  (nightly full test suite, dependency freshness checks, scheduled
  cleanups). Cron here is UTC and, like most schedulers, isn't guaranteed to
  fire at the exact minute under load — treat it as "around this time," not
  precise.
- **`workflow_dispatch`** — a manual trigger, runnable from the Actions tab
  (optionally with input parameters) — useful for a deploy workflow that
  shouldn't fire automatically on every push.

## Jobs and Steps

A workflow is made of one or more **jobs**, each made of ordered **steps**:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt
      - run: pytest
```

Jobs run in **parallel by default** unless one declares `needs:` on another,
which forces it to wait:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    runs-on: ubuntu-latest
    steps: [...]

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps: [...]
```

Here `lint` and `test` run concurrently on separate runners; `build` only
starts once both succeed — parallelizing independent work is one of the
easiest wins for CI wall-clock time, since `lint` and `test` share no state
and gain nothing from running sequentially.

## The Runner Model

Each job runs on a **fresh, ephemeral virtual machine** — nothing persists
from a previous run and nothing is shared between jobs in the same workflow
unless explicitly passed along (via `actions/cache`, artifacts, or job
outputs). This is deliberate: it guarantees every run starts from the same
known state instead of accumulating drift from whatever a previous run left
behind, at the cost of needing to reinstall dependencies from scratch every
time — which is exactly what caching (below) exists to soften.

### Matrix Builds

A **matrix** runs the same job multiple times with different variable
values — the standard way to test against multiple language versions,
OSes, or dependency versions in parallel instead of writing N near-identical
jobs by hand:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt
      - run: pytest
```

This expands into three independent jobs (one per Python version) running
in parallel. `strategy.fail-fast: false` keeps the other matrix legs running
even after one fails — useful when you want to see *all* the failing
versions in one run rather than only the first one GitHub happened to fail.

## Caching Dependencies

Since every job starts from a clean VM, reinstalling dependencies from
scratch on every run adds up fast. `actions/cache` persists a directory
between runs, keyed so it's invalidated automatically when the underlying
dependency spec changes:

```yaml
- name: Cache pip dependencies
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      pip-${{ runner.os }}-
```

`key` includes a hash of `requirements.txt`, so the cache is fresh
automatically whenever dependencies change — no manual cache-busting. The
`restore-keys` prefix lets a run fall back to the *closest* prior cache
(e.g. from before a minor dependency bump) rather than a fully cold cache
when there's no exact key match, trading a little staleness for still
avoiding a from-scratch install.

Many setup actions bake this in without a separate `actions/cache` step:

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: "pip"                    # caches ~/.cache/pip keyed on requirements.txt/poetry.lock
```

`actions/setup-node` (`cache: "npm"`/`"yarn"`), `actions/setup-python`
(`cache: "pip"`/`"poetry"`), and similar setup actions across ecosystems all
offer this — prefer it over a hand-rolled `actions/cache` block when the
setup action already supports the package manager in use, since it handles
picking the right lockfile-based key for you.

## Secrets and Environment Variables

- **Repository secrets** (Settings → Secrets and variables → Actions) are
  available to any workflow in the repo, referenced as `${{
  secrets.MY_SECRET }}`. They're masked in logs and never printed even by
  accident.
- **Environment secrets** are scoped to a named **environment** (e.g.
  `production`), and an environment can require **manual approval from a
  designated reviewer** before a job targeting it runs — the standard way to
  gate an actual deploy behind a human, even when the rest of the pipeline
  is fully automated:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production   # pauses for required-reviewer approval
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh
        env:
          API_KEY: ${{ secrets.PROD_API_KEY }}   # only visible to this environment
```

This is the practical difference between "CI" and "CD" in the same system:
`lint`/`test` jobs need no approval and no privileged secrets, while a
`deploy` job pointed at `environment: production` both scopes its secrets
tighter and can force a human to click approve before anything touches
production.

!!! warning "Secrets are still visible to whatever a step chooses to do with them"
    Masking hides a secret from being *printed* in logs, but a compromised
    or malicious step (e.g. a third-party action pinned to a mutable tag)
    can still exfiltrate it over the network. Pin third-party actions to a
    commit SHA rather than a tag like `@v4` for anything handling real
    secrets, and keep environment-scoped secrets scoped — don't hand a
    `lint` job access to production credentials it never needs.

## Reusable Workflows vs. Composite Actions

Both let CI logic be shared instead of copy-pasted across repos, but at
different granularity:

- **Composite action** — packages a sequence of *steps* as one reusable
  step, used inside a job:

```yaml
# .github/actions/setup-project/action.yml
name: Setup project
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: "3.12"
        cache: "pip"
    - run: pip install -r requirements.txt
      shell: bash
```

```yaml
# used from a workflow:
steps:
  - uses: ./.github/actions/setup-project
  - run: pytest
```

- **Reusable workflow** (`workflow_call`) — packages an entire *workflow*
  (one or more jobs) that another workflow calls as a job, useful for
  sharing a whole pipeline shape (e.g. "lint, test, build, and push an
  image") across many repositories rather than one setup step:

```yaml
# .github/workflows/reusable-build.yml
on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: pip install -r requirements.txt && pytest
```

```yaml
# called from another repo's workflow:
jobs:
  build:
    uses: my-org/shared-workflows/.github/workflows/reusable-build.yml@main
    with:
      python-version: "3.12"
```

Rule of thumb: reach for a **composite action** to dedupe a handful of
repeated steps within or across a few workflows; reach for a **reusable
workflow** when the thing being shared is an entire multi-job pipeline
that many repositories should run identically (an org-wide "this is how
every service builds and publishes an image" standard).

## A Concrete Example: Lint + Test + Build on PR

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: "pip"
      - run: pip install ruff
      - run: ruff check .

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: "pip"
      - run: pip install -r requirements.txt
      - run: pytest --cov

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t my-app:${{ github.sha }} .
```

`lint` and `test` run in parallel and gate `build` via `needs:` — a broken
lint rule or a failing test on either supported Python version stops the
image from ever being built, and the whole thing runs on every PR against
`main` before a human even reviews it. This matches the shape (though not
the exact content) of this knowledge base's own
`.github/workflows/pre-commit.yml`, which runs a single pre-commit job,
caching `~/.cache/pre-commit` keyed on `.pre-commit-config.yaml`, on every
PR into `main`.

## Summary

- CI exists to catch breakage automatically, before merge, while the fix is
  still cheap — that's the point, independent of which tool runs it.
- `on:` triggers (`push`, `pull_request`, `schedule`, `workflow_dispatch`)
  decide when a workflow runs; scope `push` narrowly since `pull_request`
  usually already covers feature-branch changes.
- Jobs run in parallel by default; `needs:` creates ordering; a matrix
  expands one job definition into many parallel runs across versions/OSes.
- Every job is a fresh, ephemeral VM — `actions/cache` (or a setup action's
  built-in `cache:` option) is what avoids reinstalling dependencies from
  scratch on every single run.
- Repo secrets are available everywhere; environment-scoped secrets can
  require a manual reviewer approval, which is the natural gate between
  "CI" and an actual deploy.
- Composite actions share steps; reusable workflows (`workflow_call`) share
  entire pipelines — pick based on how much is actually being shared.

## Related Articles

- [Docker Fundamentals](docker.md) — the `docker build` step a CI pipeline
  typically ends with before pushing an image.
- [Helm for Kubernetes](helm-charts.md) — often the deploy step a
  `workflow_call`/environment-gated job runs after CI passes.
- [Kubernetes Fundamentals](kubernetes.md) — the target a CI/CD pipeline is
  usually building and deploying toward.
