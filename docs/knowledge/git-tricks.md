---
tags:

- git
- version-control
- commands
- tips

---

# Git Tricks & Commands

Useful Git commands and tricks for daily development workflow.

## See All Files Changed by a Commit

### View Uncommitted Changes

You can use the `git diff` command with the `HEAD` argument to view all changes in the current branch that haven't been committed yet:

```bash
git diff HEAD
```

This shows the differences between your working tree (files you're currently editing) and the most recent commit on the current branch. It displays all files that have been modified or added but not yet committed.

### View Committed Files

To see all files that you've committed on the current branch:

```bash
git log --name-only --pretty=oneline
```

This command shows a list of all commits on the current branch, along with the names of files that were modified or added in each commit.

### Show Most Recent Commit

Another useful command to see the most recent commit and its files:

```bash
git show --name-only
```

### Include Deleted Files

If you want to include deleted files in the results:

```bash
git log --diff-filter=D --summary
```

This shows the commit that deleted each file, along with the commit message.

## Additional Useful Commands

### View File Changes in Specific Commit

```bash
git show <commit-hash> --name-only
```

### Compare Two Commits

```bash
git diff <commit1> <commit2> --name-only
```

### Show Files Changed Between Branches

```bash
git diff <branch1>..<branch2> --name-only
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `git diff HEAD` | Show uncommitted changes |
| `git log --name-only --pretty=oneline` | List files in all commits |
| `git show --name-only` | Show files in latest commit |
| `git log --diff-filter=D --summary` | Show deleted files |
| `git diff <commit1> <commit2> --name-only` | Compare commits |
