# Getting Started

Welcome to Igor's Knowledge Base! This guide covers how to navigate and preview the site.
Looking to add content instead? Head straight to [Contributing](contributing.md).

## Overview

This knowledge base is built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/), providing a modern, searchable, and maintainable documentation experience.

## How to Navigate

### Using the Menu

- Use the **top navigation tabs** to browse top-level sections (Knowledge Base, Roadmap,
  Troubleshooting, etc.)
- Within Knowledge Base, use the **sidebar** to move between categories (Machine Learning
  & AI, Python, Web Development, DevOps & Tools) and their articles
- Use the **search bar** (top right) to find content quickly

### Using Tags

Each article is tagged with relevant topics. You can:

- Browse all available tags on the search options
- Click on any tag to see related articles
- Use tags in the search to filter content

Example tags you'll find:

- `#machine-learning` - ML/AI content
- `#python` - Python-related content
- `#git` - Git and version control topics
- `#troubleshooting` - Logged fixes for specific problems

## How to Preview the Site Locally

To preview changes before publishing:

### Prerequisites

- Python 3.11+
- [uv](https://docs.astral.sh/uv/) installed

### Steps

1. **Install dependencies**:

   ```bash
   uv sync
   ```

1. **Start the development server**:

   ```bash
   uv run mkdocs serve
   ```

1. **Open your browser** to `http://127.0.0.1:8000`

The site will automatically reload when you make changes to markdown files.

## Deployment

The site is automatically deployed to GitHub Pages when changes are merged to the main branch via GitHub Actions. No manual deployment is needed.

## Questions or Issues

If you encounter any issues or have suggestions:

1. Check existing documentation first

1. Search through current articles and tags

1. Open an issue on the [GitHub repository](https://github.com/igormcsouza/knowledge-base)

Happy learning! 🚀
