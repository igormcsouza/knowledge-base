# Git Tricks

## See all the files that were changed by a commit

You can use the `git diff` command with the `HEAD` argument to view all the changes you've made in the current branch that haven't been committed yet.

The command `git diff HEAD` will show you the differences between your working tree (the files in your local repository that you're currently editing) and the most recent commit on the current branch. This will show you a list of all the files that you have modified or added, but not yet committed.

If you want to see all the files that you've committed on the current branch, you can use the command `git log --name-only --pretty=oneline` to see a list of all the commits on the current branch, along with the names of the files that were modified or added in each commit.

Another alternative command is `git show --name-only` which will show the most recent commit and the files in it.

If you also want to include deleted files in the results you can use `git log --diff-filter=D --summary` which will show you the commit that deleted the file, together with the commit message.
