# GitHub Copilot Instructions for Igor's Knowledge Base

## Project Overview

This is a comprehensive knowledge base built with **MkDocs Material** that contains software engineering knowledge, tips, and best practices. The site is automatically deployed to GitHub Pages and features full-text search, tag-based organization, and responsive design.

🔗 **Live Site**: <https://igormcsouza.github.io/knowledge-base/>

## Technology Stack

- **Static Site Generator**: MkDocs Material
- **Content Format**: Markdown with extensions
- **Dependency Management**: Poetry (Python 3.11+)
- **Deployment**: GitHub Actions → GitHub Pages
- **Code Quality**: Pre-commit hooks

## Content Guidelines

### Creating New Articles

1. **File Location**: Place new `.md` files in appropriate subdirectories under the docs content directory

1. **File Naming**: Use lowercase with hyphens (e.g., `topic-subtopic.md`)

1. **Front Matter**: Include tags and metadata at the top of articles:

```markdown
---
tags:
  - [Technology]
  - [Category]
  - [Difficulty Level]
---

# Article Title

Your content here...
```

### Content Organization

- **Navigation**: Update the site configuration nav section when adding new articles
- **Categories**: Organize content by technology/topic
- **Tags**: Use consistent tags for searchability and categorization
- **Cross-references**: Link related articles using relative paths

### Markdown Conventions

This project uses **PyMdown Extensions** with these features:

- **Code Blocks**: Use syntax highlighting with language specifiers
- **Admonitions**: Use `!!! note`, `!!! warning`, `!!! tip` for callouts
- **Tabs**: Use `=== "Tab Title"` for multi-option content
- **Collapsible Sections**: Use `??? "Collapsible Title"` for optional details

Example patterns:

```markdown
!!! tip "Pro Tip"
    This is a helpful tip for readers.
```

```language
# Code example with appropriate syntax highlighting
function_call()
```

```markdown
=== "Option 1"
    Content for first option.

=== "Option 2"
    Content for second option.
```

## Development Workflow

### Local Development Setup

1. **Clone Repository**: Use standard git clone workflow

1. **Install Dependencies**: Use Poetry for dependency management

1. **Start Development Server**: Use MkDocs serve command for local development

1. **Build Site Locally**: Use MkDocs build command for testing

### Content Creation Process

1. Create new markdown files in appropriate content subdirectories

1. Add navigation entry to site configuration if creating new sections

1. Use consistent tags and follow markdown conventions

1. Test locally with development server

1. Commit changes and create pull request

### Navigation Configuration

When adding new content, update the nav section in the site configuration file:

```yaml
nav:

  - Home: index.md
  - Getting Started: getting-started.md
  - Knowledge Base:
      - Overview: knowledge/index.md
      - [Category]:
          - [Topic]: knowledge/topic-name.md
          - [New Article]: knowledge/new-topic.md  # Add here

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

    language
    # Command or code examples with appropriate syntax highlighting
    command_example

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

### Tag Categories

Use these consistent tag categories:

- **Technologies**: Language and framework names
- **Categories**: `Tips`, `Tutorial`, `Reference`, `Best Practices`, `Commands`
- **Difficulty**: `Beginner`, `Intermediate`, `Advanced`
- **Topics**: `Web Development`, `DevOps`, `Testing`, `Performance`

## Deployment

- **Automatic**: Pushes to `main` branch trigger GitHub Actions deployment
- **Manual**: Never needed - all deployments are automated
- **Preview**: Use local development server for preview before committing

## Code Quality

- **Pre-commit Hooks**: Automatically run on commits
- **Linting**: Markdown files are checked for consistency
- **Link Validation**: Ensure internal links work correctly

## Assistance Guidelines for Copilot

When helping with this project:

1. **Content Creation**: Suggest appropriate file locations, tags, and navigation entries

1. **Markdown**: Use project-specific extensions and formatting conventions

1. **Structure**: Follow the established organization pattern

1. **Navigation**: Always suggest updating site configuration when adding new content

1. **Tags**: Recommend consistent, searchable tags

1. **Examples**: Include practical code examples with proper syntax highlighting

1. **Cross-references**: Suggest linking to related existing content

## Troubleshooting

### Common Issues

- **Build Failures**: Check site configuration syntax and file paths
- **Navigation Issues**: Ensure all referenced files exist
- **Search Problems**: Verify tags are properly formatted
- **Styling Issues**: Check markdown extension syntax

### Commands for Debugging

Use the appropriate Poetry and MkDocs commands for:

- Configuration validation
- Verbose build output
- Link checking and validation

This knowledge base serves as a living document of software engineering expertise. When contributing, focus on practical, actionable content that helps developers solve real problems.

## Maintenance Guidelines

**Important**: Keep examples in this file generic and avoid specific code snippets, file paths, or implementation details that may become outdated. Use placeholders like `[Technology]`, `[Category]`, or `language` instead of actual values. This prevents the instructions from becoming obsolete and requiring frequent updates.
