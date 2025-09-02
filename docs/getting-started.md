# Getting Started

Welcome to Igor's Knowledge Base! This guide will help you navigate, use, and contribute to this documentation site.

## Overview

This knowledge base is built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/), providing a modern, searchable, and maintainable documentation experience.

## How to Navigate

### Using the Menu
- Use the **top navigation tabs** to browse different topic areas (Python, Git, etc.)
- Click on **sections in the sidebar** to access specific articles
- Use the **search bar** (top right) to find content quickly

### Using Tags
Each article is tagged with relevant topics. You can:

- Browse all available tags on the search options
- Click on any tag to see related articles
- Use tags in the search to filter content

Example tags you'll find:
- `#python` - Python-related content
- `#git` - Git and version control topics
- `#flask` - Flask framework articles
- `#tips` - Quick tips and tricks

## How to Add New Pages/Notes

To add new content to the knowledge base:

1. **Create a new Markdown file** in the `docs/knowledge/` directory
2. **Add tags at the top** of your file:
   ```markdown
   tags:
     - python
     - tips
     - advanced

   # Your Article Title
   ```
3. **Use descriptive filenames** based on content (e.g., `python-decorators.md`, `git-advanced-workflows.md`)
4. **Update navigation** in `mkdocs.yml` if needed
5. **Follow existing content structure** for consistency

## How to Use Tags for Search

Tags make content discovery easier:

### In Articles
Add tags at the top of each markdown file:
```markdown
tags:
  - python
  - web-development
  - flask

# Article Title
Your content here...
```

### Searching with Tags
- Use the search bar to find tag-specific content
- Click on tags within articles to find related content

## How to Preview the Site Locally

To preview changes before publishing:

### Prerequisites
- Python 3.11+
- Poetry installed

### Steps
1. **Install dependencies**:
   ```bash
   poetry install
   ```

2. **Activate the virtual environment**:
   ```bash
   poetry shell
   ```

3. **Start the development server**:
   ```bash
   mkdocs serve
   ```

4. **Open your browser** to `http://127.0.0.1:8000`

The site will automatically reload when you make changes to markdown files.

## Content Guidelines

When adding or editing content:

### File Naming
- Use descriptive, lowercase names with hyphens
- Include the main topic: `python-data-structures.md`
- Be specific: `git-rebase-interactive.md` vs `git-stuff.md`

### Content Structure
- Start with a clear title
- Add relevant tags
- Use proper markdown formatting
- Include code examples where helpful
- Add links to external resources

### Tags Best Practices
- Use existing tags when possible
- Keep tags lowercase with hyphens for multi-word tags
- Be specific but not overly granular
- Common tag categories:
  - **Technology**: `python`, `git`, `docker`
  - **Type**: `tips`, `tutorial`, `reference`
  - **Level**: `beginner`, `intermediate`, `advanced`

## Deployment

The site is automatically deployed to GitHub Pages when changes are merged to the main branch via GitHub Actions. No manual deployment is needed.

## Questions or Issues?

If you encounter any issues or have suggestions:

1. Check existing documentation first
2. Search through current articles and tags
3. Open an issue on the [GitHub repository](https://github.com/igormcsouza/knowledge-base)

Happy learning! 🚀
