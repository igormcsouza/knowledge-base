---
tags:
  - git
  - configuration
  - setup
  - workflow
---

# Git Configurations

Essential Git configurations for an improved development workflow.

## Basic User Configuration

Set up your identity for commits:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

## Useful Global Configurations

### Default Editor

Set your preferred editor for commit messages:

```bash
# For VS Code
git config --global core.editor "code --wait"

# For Vim
git config --global core.editor "vim"

# For Nano
git config --global core.editor "nano"
```

### Line Ending Configuration

Handle line endings automatically:

```bash
# For Windows
git config --global core.autocrlf true

# For macOS/Linux
git config --global core.autocrlf input
```

### Default Branch Name

Set the default branch name for new repositories:

```bash
git config --global init.defaultBranch main
```

## Helpful Aliases

Create shortcuts for common Git commands:

```bash
# Short status
git config --global alias.st status

# Short log with graph
git config --global alias.lg "log --oneline --graph --decorate"

# Unstage files
git config --global alias.unstage "reset HEAD --"

# Show last commit
git config --global alias.last "log -1 HEAD"

# Better diff
git config --global alias.visual "!gitk"
```

## Advanced Configurations

### Push Configuration

Set default push behavior:

```bash
git config --global push.default simple
```

### Pull Configuration

Set default pull behavior to rebase:

```bash
git config --global pull.rebase true
```

### Credential Helper

Store credentials securely:

```bash
# For Windows
git config --global credential.helper manager-core

# For macOS
git config --global credential.helper osxkeychain

# For Linux
git config --global credential.helper store
```

## View Current Configuration

Check your current Git configuration:

```bash
# View all configurations
git config --list

# View specific configuration
git config user.name
git config user.email
```

## Configuration File Locations

- **Global**: `~/.gitconfig` or `~/.config/git/config`
- **System**: `/etc/gitconfig`
- **Repository**: `.git/config` (in the repository directory)

## Quick Setup Script

Here's a quick setup script for new machines:

```bash
#!/bin/bash
# Basic Git setup
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global core.editor "code --wait"
git config --global init.defaultBranch main
git config --global push.default simple
git config --global pull.rebase true

# Useful aliases
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --decorate"
git config --global alias.unstage "reset HEAD --"
git config --global alias.last "log -1 HEAD"

echo "Git configuration complete!"
```