---
tags:

- python
- uv
- tooling
- package-management

---

# UV: Managing Python Environments and Projects

[`uv`](https://github.com/astral-sh/uv) is a Python package and project
manager from Astral (the team behind `ruff`), written in Rust. It aims to
replace the usual pile of tools — `pip`, `pip-tools`, `virtualenv`, `pyenv`,
and much of what `poetry` does — with a single, much faster binary.

## Why Bother Switching

- **Speed** — dependency resolution and installs are typically 10-100x faster
  than `pip`, thanks to a Rust implementation, a global cache, and parallel
  downloads/builds.
- **One tool** — Python version management, virtual environments, dependency
  resolution, and lockfiles are all one CLI instead of `pyenv` + `venv` +
  `pip` + `pip-tools` stitched together.
- **Drop-in compatible** — `uv pip install` works as a fast drop-in for
  `pip install` where you don't want to adopt the full project workflow yet.

## Managing Python Versions

```bash
uv python install 3.12       # download and install a Python version
uv python list                # list available/installed versions
uv python pin 3.12            # pin this project to a version (writes .python-version)
```

This replaces `pyenv` for most day-to-day needs — no separate tool required.

## Starting a Project

```bash
uv init my-project
cd my-project
```

This scaffolds a `pyproject.toml`, a starter `main.py`, and a `.python-version`
file. `pyproject.toml` becomes the single source of truth for dependencies
and project metadata (same idea as Poetry).

## Virtual Environments

```bash
uv venv                # create a .venv in the current directory
```

`uv run` (below) creates and manages this automatically in most workflows, so
manual `uv venv` + `source .venv/bin/activate` is mostly needed when you want
an environment without the rest of the project workflow.

## Adding & Removing Dependencies

```bash
uv add fastapi                  # add a runtime dependency, updates pyproject.toml + lockfile
uv add --dev pytest ruff        # add dev-only dependencies
uv remove fastapi                # remove a dependency
```

Each of these updates `pyproject.toml` *and* re-resolves `uv.lock`
automatically — no separate "update the lockfile" step to remember.

## Running Code

```bash
uv run python main.py    # runs inside the project's venv, auto-syncing deps first
uv run pytest             # same idea for any installed tool
```

`uv run` checks that the environment matches the lockfile before running, and
syncs it if not — so there's no "forgot to activate the venv" or "forgot to
`pip install` after a `git pull`" class of mistake.

## Lockfile: Reproducible Installs

```bash
uv lock     # (re)generate uv.lock from pyproject.toml, without installing
uv sync     # install exactly what's in uv.lock into the environment
```

`uv.lock` pins exact versions (and hashes) of every dependency, direct and
transitive, so `uv sync` on a teammate's machine or in CI reproduces the
exact same environment. Commit `uv.lock` to version control, same as you
would `poetry.lock` or `package-lock.json`.

## Example Workflow End to End

```bash
uv python install 3.12
uv init my-api && cd my-api
uv add fastapi uvicorn
uv add --dev pytest ruff
uv run uvicorn main:app --reload
```

No manual venv activation, no separate `pip freeze`, no drift between what's
declared and what's installed.

!!! tip "Pro Tip"
    `uv pip install -r requirements.txt` and `uv pip compile` also exist as
    faster drop-ins for existing `pip`/`pip-tools` workflows, useful for
    migrating a legacy project incrementally before fully switching to the
    `uv add`/`pyproject.toml` project workflow.

## Summary

- `uv` replaces `pyenv` + `venv` + `pip` + `pip-tools` (and much of `poetry`)
  with one fast Rust-based tool.
- `uv add`/`uv remove` manage dependencies and the lockfile together.
- `uv run` auto-syncs the environment before running anything — no "activate
  first" step to forget.
- Commit `uv.lock` for reproducible installs across machines and CI.

## Related Articles

- [Python Tips & Tricks](python-tips.md) — other Python fundamentals worth
  knowing alongside the tooling.
