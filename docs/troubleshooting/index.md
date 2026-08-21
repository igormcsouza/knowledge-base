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

1. Copy the structure below (Problem, Environment, Root Cause, Fix, Prevention).

1. Add tags (at minimum the relevant technology, plus `troubleshooting`).

1. Link the new entry from the [Entries](#entries) list below.

See [Contributing](../contributing.md) for the general file/naming conventions shared
across the whole knowledge base.

### Entry Template

````markdown
---
tags:

- troubleshooting
- technology

---

# Short, Specific Problem Title

One sentence describing what went wrong, in plain language — this is what you'll be
scanning for six months from now.

## Environment

- Tool/library + version:
- OS / runtime:
- Any other relevant context (framework version, deployment target, etc.):

## Problem

What happened. Include the exact error message or symptom, verbatim — this is often what
you'll be searching for later.

```text
Paste the exact error/traceback here.
```

## Root Cause

Why it actually happened, not just what made the symptom go away. If you never fully
figured out the root cause, say so explicitly rather than skipping this section.

## Fix

The concrete steps or code that resolved it.

```bash
# commands, if applicable
```

## Prevention

How to avoid hitting this again — a config change, a habit, a check to add to a checklist,
or just "now I know to look here first."
````

## Entries

- [Browser Keeps Prompting to Log In on Every Article (polyfill.io)](polyfill-io-login-prompt.md)
- [WiFi Bufferbloat Collapses the Connection When Streaming to a LAN Device (Expo, Steam Link)](wifi-bufferbloat-lan-streaming-drops.md)
- [WiFi Card Drops Connection Under Load Due to PCIe Power Management (mt7925e Forced Reassociation)](wifi-card-pcie-power-reassociation-drops.md)
