---
title: Achievements
tags:

- achievements
- overview

---

# Achievements

A running log of things I built, fixed, or improved that are worth remembering — with the
numbers and the reasoning attached. The point is simple: when someone asks "what have you
done?" or "tell me about a time you...", the answer is a search away instead of a
half-remembered story.

This is different from [Troubleshooting](../troubleshooting/index.md) (a bug I hit and how
the fix works, written so it's findable by symptom) and from the
[Roadmap](../roadmap/index.md) (progress against the Senior competency matrix). An
achievement entry is written from the *outcome*: what changed, by how much, and what I'd
say about it in an interview.

## When to Add an Entry

Add one whenever something ships that I'd be happy to talk about later — a measurable
improvement, a hard problem solved, a project milestone. Capture it the same day, while
the numbers are still in the dashboards.

## How to Add One

1. Create a file directly under `docs/achievements/`, named for the outcome (e.g.
   `draftr-p95-latency-background-s3-deletes.md`).

1. Use the structure below: Summary, Context, Problem, What I Did, Result, Takeaways.

1. Add tags (at minimum `achievements`, plus the relevant technologies).

1. Link the entry from the [Entries](#entries) list below, newest first.

### Entry Template

````markdown
---
tags:

- achievements
- technology

---

# Outcome-Focused Title

One-line summary with the headline number.

- **Project:** name (link to the repo)
- **Date:** YYYY-MM-DD
- **Role:** what I owned

## Context

What the system is and why this mattered.

## Problem

What was wrong, and how I noticed it.

## What I Did

The investigation and the change.

## Result

Before/after numbers, with the source (dashboard, query, time window).

## Takeaways

What I learned, trade-offs accepted, and how I'd tell this story in an interview.
````

## Entries

Newest first.

- [Draftr: Cut a Slow Endpoint's p95 from ~2.3 s to ~0.24 s by Moving S3 Deletes to a Background Task](draftr-p95-latency-background-s3-deletes.md) — 2026-10-04
