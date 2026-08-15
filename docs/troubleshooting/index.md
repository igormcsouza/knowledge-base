---
title: Troubleshooting
tags:

- troubleshooting
- overview

---

# Troubleshooting

A running log of real problems hit during development (or day-to-day life) and how they
got fixed. The point isn't documentation for its own sake — it's that unsolved-then-solved
issues have a habit of coming back to haunt you later. Writing the fix down once means
next time it's a search away, not a re-investigation.

This is different from the topic-based [Knowledge Base](../knowledge/index.md) sections:
those are "here's how X works," this is "here's a specific problem I hit and exactly what
fixed it."

## When to Add an Entry

Add an entry any time you spend more than a few minutes chasing down a bug, a confusing
error message, or a broken environment/tool — especially if the fix wasn't obvious from
the error itself. If you had to search, experiment, or dig into logs to find the answer,
it's worth capturing.

## How to Add One

1. Create a new file directly under `docs/troubleshooting/`, named for the problem (e.g.
   `flask-cli-command-not-found.md`, `poetry-lock-conflict.md`).

1. Start from [template.md](template.md) — copy its structure (Problem, Environment, Root
   Cause, Fix, Prevention).

1. Add tags (at minimum the relevant technology, plus `troubleshooting`).

See [Contributing](../contributing.md) for the general file/naming conventions shared
across the whole knowledge base.

## Entries

Empty for now — the first real fix logged here starts the list.
