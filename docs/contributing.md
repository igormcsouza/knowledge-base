---
title: Contributing
tags:

- contributing
- meta

---

# Contributing: How to Add Knowledge

This is the one page to check whenever you've learned something and want to save it. The
goal is friction-free capture: if adding a note takes more than "write a markdown file in
the right folder," something's wrong — fix the process, not the habit.

## 1. Decide Where It Goes

```text
Is it a specific bug/error you hit and fixed?
  → docs/troubleshooting/ (see its own template)

Is it a progress update on your Senior-engineer roadmap?
  → docs/roadmap/index.md (edit the matrix + add a Day X entry)

Otherwise, it's a knowledge-base topic. Does it fit an existing category?
  → docs/knowledge/machine-learning/
  → docs/knowledge/python/
  → docs/knowledge/web-development/
  → docs/knowledge/devops-tools/
  → docs/knowledge/architecture/
  → docs/knowledge/databases/
  → docs/knowledge/algorithms/

Doesn't fit any of those?
  → Create a new category: docs/knowledge/<new-category>/index.md
     (short landing page, same shape as the existing category index.md files)
     then add your article alongside it.
```

There's no fixed ceiling on categories — when a topic doesn't belong anywhere, that's the
signal to create a new one, not to force-fit it or leave it flat at the top level.

A category can also grow a **subfolder** once it accumulates several articles that share a
more specific theme — e.g. `docs/knowledge/architecture/patterns/` for code-level design
patterns, distinct from the broader architectural styles in `architecture/` itself. Give the
subfolder its own `index.md` (same shape as a category's), and link it from the parent
category's `index.md`. Navigation is still automatic — mkdocs nests subfolders under their
parent with no config changes needed. Don't reach for this by default; it's worth it only
once a category has enough related articles that a sub-grouping actually clarifies things.

## 2. Navigation Is Automatic

`mkdocs.yml` has no `nav:` section — the site menu is generated straight from the folder
structure. **You never need to edit `mkdocs.yml` to add a page.** Drop the file in the
right folder and it appears in the nav on the next build. This is deliberate: it's what
keeps "add a topic" a one-file action.

The only reason to touch `mkdocs.yml` is a genuine site-config change (theme, plugins,
extensions) — not routine content additions.

## 3. One File Per Topic

- Each file covers one topic. If an article is trying to cover two unrelated things,
  split it.
- Filename: lowercase, hyphen-separated, descriptive — `neural-networks-math-fundamentals.md`,
  not `notes.md` or `ml2.md`.
- Every category folder has its own `index.md` that lists the articles in it — add a link
  there when you add a new article (see any existing category's `index.md` for the
  pattern).

## 4. Front Matter

Every article starts with YAML front matter and at least a few tags:

```markdown
---
tags:

- technology-or-topic
- category
- difficulty-level

---

# Article Title
```

The blank lines around the tag list aren't optional style — `markdownlint` (run in CI via
pre-commit) enforces MD032 ("lists should be surrounded by blank lines"), and it applies
inside front matter too. Skipping them fails the pre-commit hook.

Reuse existing tags where they fit (browse current articles for examples) rather than
inventing near-duplicates (`ml` vs `machine-learning` — pick one and stay consistent;
this base already uses `machine-learning`).

## 5. Article Structure

A reasonable default shape — not every article needs every section, but this is the
pattern to reach for:

````markdown
---
tags:

- technology
- category
- difficulty

---

# Article Title

One or two sentences on what this covers and why it's useful.

## Section 1

Explanation with practical examples. Prefer self-contained explanations over "watch this
video" — link to videos/articles as *supplementary* resources, not as the entire content.

```python
# code examples with language tags for syntax highlighting
```

!!! tip "Pro Tip"
    Use admonitions (`!!! note`, `!!! tip`, `!!! warning`) for callouts.

## Summary

- Key takeaway 1
- Key takeaway 2

## Related Articles

- [Related Topic](../other-category/other-article.md)
````

Available markdown extensions (already configured, just use them):

- Admonitions: `!!! note`, `!!! tip`, `!!! warning`
- Collapsible sections: `??? "Title"`
- Tabs: `=== "Tab Title"`
- Math: inline `$...$`, block `$$...$$` (MathJax, enabled for the math-heavy ML articles)
- Fenced code blocks with syntax highlighting and copy button (automatic)

## 6. Cross-Link

If a new article relates to an existing one, add a short **Related Articles** section
linking both directions. This is what makes the tag/search system genuinely useful instead
of just a pile of independent files.

## 7. Preview Locally Before Committing

```bash
poetry install
poetry run mkdocs serve
```

Then open `http://127.0.0.1:8000` — the site reloads automatically as you edit. Worth
doing for anything with math, tables, or admonitions, since those are the easiest things
to get subtly wrong in raw markdown.

## 8. Special Sections

Two sections have their own dedicated format, different from a standard topic article:

- **[Troubleshooting](troubleshooting/index.md)** — one file per fixed problem, using
  [the entry template in the troubleshooting index](troubleshooting/index.md#how-to-add-one)
  (Problem / Environment / Root Cause / Fix / Prevention).
- **[Roadmap](roadmap/index.md)** — a single page you edit in place (competency matrix +
  Day X log), not a new file per update.

## Quality Bar

- Prefer self-contained explanations over link dumps.
- Include real, working code examples where the topic calls for it.
- If something's unfinished or a personal opinion rather than settled fact, say so (an
  admonition like `!!! note` works well for this) instead of presenting it as definitive.
