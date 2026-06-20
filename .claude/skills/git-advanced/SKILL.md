---
name: git-advanced
description: Git past the basics — worktrees, bisect, interactive rebase, hooks, recovery, and handy aliases
---

# Advanced Git

## Worktrees

```bash
# Create a worktree for a feature branch (avoids stashing)
git worktree add ../feature-auth feature/auth

# Create a worktree with a new branch
git worktree add ../hotfix-123 -b hotfix/123 origin/main

# List all worktrees
git worktree list

# Remove a worktree after merging
git worktree remove ../feature-auth
```

A worktree gives you a second branch checked out in its own directory — no stashing, no throwaway WIP commits — while still sharing the one underlying `.git` repository.

## Bisect

```bash
# Start bisect, mark current as bad and known good commit
git bisect start
git bisect bad HEAD
git bisect good v1.5.0

# Git checks out a midpoint commit. Test it, then mark:
git bisect good   # if this commit works
git bisect bad    # if this commit is broken

# Automate with a test script
git bisect start HEAD v1.5.0
git bisect run npm test

# When done, reset
git bisect reset
```

Bisect binary-searches your history to pin down the commit that introduced a bug — and handing it a test command with `run` is by far the fastest way to do it.

## Interactive rebase

```bash
# Rebase last 5 commits interactively
git rebase -i HEAD~5

# Common operations in the editor:
# pick   - keep commit as-is
# reword - change commit message
# edit   - stop to amend the commit
# squash - merge into previous commit, keep both messages
# fixup  - merge into previous commit, discard this message
# drop   - remove the commit entirely

# Rebase feature branch onto latest main
git fetch origin
git rebase origin/main

# Continue after resolving conflicts
git rebase --continue

# Abort if things go wrong
git rebase --abort
```

## Hooks

```bash
#!/bin/sh
# .git/hooks/pre-commit

# Run linter on staged files only
STAGED_FILES=$(git diff --cached --name-only --diff-filter=d | grep -E '\.(ts|tsx|js|jsx)$')
if [ -n "$STAGED_FILES" ]; then
  npx eslint $STAGED_FILES --fix
  git add $STAGED_FILES
fi
```

```bash
#!/bin/sh
# .git/hooks/commit-msg

# Enforce conventional commit format
COMMIT_MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,72}$"

if ! echo "$COMMIT_MSG" | head -1 | grep -qE "$PATTERN"; then
  echo "Error: Commit message must follow Conventional Commits format"
  echo "Example: feat(auth): add OAuth2 login flow"
  exit 1
fi
```

```bash
#!/bin/sh
# .git/hooks/pre-push

# Run tests before pushing
npm test
if [ $? -ne 0 ]; then
  echo "Tests failed. Push aborted."
  exit 1
fi
```

## Getting work back

```bash
# Undo last commit but keep changes staged
git reset --soft HEAD~1

# Recover a deleted branch using reflog
git reflog
git checkout -b recovered-branch HEAD@{3}

# Recover a file from a specific commit
git checkout abc1234 -- path/to/file.ts

# Find lost commits (dangling after reset or rebase)
git fsck --lost-found
git show <dangling-commit-sha>

# Undo a rebase
git reflog
git reset --hard HEAD@{5}  # point before rebase started
```

## Aliases worth having

```bash
# ~/.gitconfig
[alias]
  lg = log --graph --oneline --decorate --all
  st = status -sb
  co = checkout
  unstage = reset HEAD --
  last = log -1 HEAD --stat
  branches = branch -a --sort=-committerdate
  stash-all = stash push --include-untracked
  conflicts = diff --name-only --diff-filter=U
```

## What to avoid

- Force-pushing a shared branch without `--force-with-lease`
- Rebasing commits you've already pushed and others have pulled
- Committing large binaries without Git LFS
- Running `git add .` without first reading `git diff --staged`
- Skipping `.gitignore` for build artifacts, dependencies, and secrets
- Letting feature branches go stale instead of merging often

## Before you ship

- [ ] Parallel branch work happens in worktrees, not stashes
- [ ] `git bisect run` automates the hunt with a test command
- [ ] Interactive rebase tidies commits before they hit main
- [ ] A pre-commit hook lints the staged files
- [ ] A commit-msg hook enforces the message format
- [ ] Force-pushes use `--force-with-lease`, never bare `--force`
- [ ] You check the reflog before anything destructive
- [ ] `.gitignore` covers build output, dependencies, and env files
