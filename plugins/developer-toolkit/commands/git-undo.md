---
description: Safely undo any git mistake with guided recovery
---

# /git-undo

Recover from any git mistake safely.

## What I Need

Tell me:
- What did you do that you want to undo?
- Have you pushed to remote yet?
- Is this a shared branch?

## Common Undos

### Undo Last Commit (Not Pushed)

```bash
# Keep changes staged
git reset --soft HEAD~1

# Keep changes unstaged
git reset HEAD~1

# Discard changes completely
git reset --hard HEAD~1
```

### Undo Pushed Commit

```bash
# Create a new commit that undoes the changes (safe for shared branches)
git revert HEAD
git push
```

### Undo Staged Files

```bash
# Unstage specific file
git reset HEAD <file>

# Unstage all files
git reset HEAD
```

### Undo Working Directory Changes

```bash
# Discard changes in specific file
git checkout -- <file>

# Discard all changes (DANGEROUS)
git checkout -- .
```

### Undo Merge

```bash
# If not committed yet
git merge --abort

# If committed but not pushed
git reset --hard HEAD~1

# If pushed (create revert)
git revert -m 1 HEAD
```

### Recover Deleted Branch

```bash
# Find the commit
git reflog

# Recreate branch
git checkout -b <branch-name> <commit-sha>
```

## Safety Rules

1. **Not pushed?** → `reset` is safe
2. **Pushed to shared branch?** → Use `revert`
3. **Unsure?** → Ask before running destructive commands
4. **Always check** → `git status` and `git log` first

## I'll Help With

- Identifying the safest undo method
- Preserving your work
- Avoiding force pushes on shared branches
- Recovering "lost" commits via reflog
