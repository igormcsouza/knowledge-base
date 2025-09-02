# GitHub Copilot Instructions for Igor's Knowledge Base

## Project Overview

This is a comprehensive knowledge base built with **MkDocs Material** that contains software engineering knowledge, tips, and best practices. The site is automatically deployed to GitHub Pages and features full-text search, tag-based organization, and responsive design.

🔗 **Live Site**: https://igormcsouza.github.io/knowledge-base/

## Technology Stack

- **Static Site Generator**: MkDocs Material
- **Content Format**: Markdown with extensions
- **Dependency Management**: Poetry (Python 3.11+)
- **Deployment**: GitHub Actions → GitHub Pages
- **Code Quality**: Pre-commit hooks

## Project Structure

```
knowledge-base/
├── docs/                          # All content files
│   ├── index.md                   # Homepage
│   ├── getting-started.md         # Contributor guide
│   └── knowledge/                 # Main content directory
│       ├── index.md               # Knowledge section overview
│       ├── python-tips.md         # Python development content
│       ├── python-flask-framework.md
│       ├── git-tricks.md          # Git commands and workflows
│       └── git-configurations.md
├── mkdocs.yml                     # Site configuration and navigation
├── pyproject.toml                 # Poetry dependencies
└── .github/workflows/deploy.yml   # Automated deployment
```

## Content Guidelines

### Creating New Articles

1. **File Location**: Place new `.md` files in appropriate subdirectories under `/docs/knowledge/`
2. **File Naming**: Use lowercase with hyphens (e.g., `python-advanced-features.md`)
3. **Front Matter**: Include tags and metadata at the top of articles:

```markdown
---
tags:
  - Python
  - Advanced
  - Best Practices
---

# Article Title

Your content here...
```

### Content Organization

- **Navigation**: Update `mkdocs.yml` nav section when adding new articles
- **Categories**: Organize content by technology/topic (Python, Git, Web Frameworks, etc.)
- **Tags**: Use consistent tags for searchability and categorization
- **Cross-references**: Link related articles using relative paths

### Markdown Conventions

This project uses **PyMdown Extensions** with these features:

- **Code Blocks**: Use syntax highlighting with language specifiers
- **Admonitions**: Use `!!! note`, `!!! warning`, `!!! tip` for callouts
- **Tabs**: Use `=== "Tab Title"` for multi-option content
- **Collapsible Sections**: Use `??? "Collapsible Title"` for optional details

Example:
```markdown
!!! tip "Pro Tip"
    This is a helpful tip for readers.

```python
def example_function():
    """Always include docstrings."""
    return "Hello, World!"
```

=== "Option 1"
    Content for first option.

=== "Option 2"
    Content for second option.
```

## Development Workflow

### Local Development Setup

1. **Clone Repository**:
   ```bash
   git clone https://github.com/igormcsouza/knowledge-base.git
   cd knowledge-base
   ```

2. **Install Dependencies**:
   ```bash
   pip install poetry
   poetry install
   ```

3. **Start Development Server**:
   ```bash
   poetry run mkdocs serve
   ```
   Site available at http://127.0.0.1:8000

4. **Build Site Locally**:
   ```bash
   poetry run mkdocs build
   ```

### Content Creation Process

1. Create new markdown files in appropriate `/docs/knowledge/` subdirectory
2. Add navigation entry to `mkdocs.yml` if creating new sections
3. Use consistent tags and follow markdown conventions
4. Test locally with `mkdocs serve`
5. Commit changes and create pull request

### Navigation Configuration

When adding new content, update the `nav` section in `mkdocs.yml`:

```yaml
nav:
  - Home: index.md
  - Getting Started: getting-started.md
  - Knowledge Base:
      - Overview: knowledge/index.md
      - Python:
          - Tips & Tricks: knowledge/python-tips.md
          - New Article: knowledge/python-new-topic.md  # Add here
      - Git:
          - Tricks & Commands: knowledge/git-tricks.md
```

## Common Patterns and Examples

### Article Structure Template

```markdown
---
tags:
  - [Technology]
  - [Category]
  - [Difficulty Level]
---

# Article Title

Brief introduction explaining what this article covers.

## Table of Contents

- [Section 1](#section-1)
- [Section 2](#section-2)

## Section 1

Content with practical examples.

```bash
# Command examples
git status
```

!!! tip "Best Practice"
    Include actionable tips and best practices.

## Section 2

More detailed content.

## Summary

- Key takeaway 1
- Key takeaway 2
- Key takeaway 3

## Related Articles

- [Related Topic](relative-link.md)
- [Another Topic](../category/other-article.md)
```

### Tag Categories

Use these consistent tag categories:

- **Technologies**: `Python`, `Git`, `Flask`, `Django`, `JavaScript`, etc.
- **Categories**: `Tips`, `Tutorial`, `Reference`, `Best Practices`, `Commands`
- **Difficulty**: `Beginner`, `Intermediate`, `Advanced`
- **Topics**: `Web Development`, `DevOps`, `Testing`, `Performance`

## Deployment

- **Automatic**: Pushes to `main` branch trigger GitHub Actions deployment
- **Manual**: Never needed - all deployments are automated
- **Preview**: Use `mkdocs serve` for local preview before committing

## Code Quality

- **Pre-commit Hooks**: Automatically run on commits
- **Linting**: Markdown files are checked for consistency
- **Link Validation**: Ensure internal links work correctly

## Assistance Guidelines for Copilot

When helping with this project:

1. **Content Creation**: Suggest appropriate file locations, tags, and navigation entries
2. **Markdown**: Use project-specific extensions and formatting conventions
3. **Structure**: Follow the established organization pattern
4. **Navigation**: Always suggest updating `mkdocs.yml` when adding new content
5. **Tags**: Recommend consistent, searchable tags
6. **Examples**: Include practical code examples with proper syntax highlighting
7. **Cross-references**: Suggest linking to related existing content

## Troubleshooting

### Common Issues

- **Build Failures**: Check `mkdocs.yml` syntax and file paths
- **Navigation Issues**: Ensure all referenced files exist
- **Search Problems**: Verify tags are properly formatted
- **Styling Issues**: Check markdown extension syntax

### Commands for Debugging

```bash
# Validate configuration
poetry run mkdocs config

# Build with verbose output
poetry run mkdocs build --verbose

# Check for broken links
poetry run mkdocs serve --strict
```

This knowledge base serves as a living document of software engineering expertise. When contributing, focus on practical, actionable content that helps developers solve real problems.