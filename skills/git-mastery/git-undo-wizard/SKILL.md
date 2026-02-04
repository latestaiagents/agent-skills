---
name: git-undo-wizard
description: |
  Use this skill when the user needs to undo, revert, or recover from Git mistakes. Activate when they mention:
  undo commit, revert changes, accidentally committed, wrong branch, recover deleted, reset HEAD,
  undo push, undo merge, restore file, git reflog, "I messed up", lost commits, or any Git recovery scenario.
---

# Git Undo Wizard

Recover from any Git mistake safely.

## When to Use

- Accidentally committed to wrong branch
- Need to undo a commit (before or after push)
- Accidentally deleted a branch or file
- Need to recover lost commits
- Want to undo a merge
- Committed sensitive data by mistake

## Quick Reference

| Situation | Command |
|-----------|---------|
| Undo last commit, keep changes | `git reset --soft HEAD~1` |
| Undo last commit, discard changes | `git reset --hard HEAD~1` |
| Undo changes to a file | `git checkout -- <file>` |
| Undo staged changes | `git reset HEAD <file>` |
| Undo a pushed commit | `git revert <commit>` |
| Recover deleted branch | `git reflog` + `git checkout -b <branch> <sha>` |

## Scenarios & Solutions

### 1. Undo Last Commit (Not Pushed)

**Keep the changes (just uncommit):**
```bash
git reset --soft HEAD~1
# Changes are now staged, ready to recommit
```

**Unstage the changes too:**
```bash
git reset HEAD~1
# Changes are in working directory, not staged
```

**Discard everything (dangerous):**
```bash
git reset --hard HEAD~1
# Changes are gone forever
```

### 2. Undo Last Commit (Already Pushed)

**Safe way (creates a new "undo" commit):**
```bash
git revert HEAD
git push
# This creates a new commit that undoes the previous one
```

**If you MUST rewrite history (only on your own branch):**
```bash
git reset --hard HEAD~1
git push --force-with-lease
# Never do this on shared branches!
```

### 3. Committed to Wrong Branch

**Move commit to correct branch:**
```bash
# Note the commit hash
git log -1  # Copy the hash

# Go to correct branch
git checkout correct-branch

# Cherry-pick the commit
git cherry-pick <commit-hash>

# Go back and remove from wrong branch
git checkout wrong-branch
git reset --hard HEAD~1
```

### 4. Undo Changes to a File

**Discard unstaged changes:**
```bash
git checkout -- <filename>
# Or in newer Git:
git restore <filename>
```

**Unstage a file (keep changes):**
```bash
git reset HEAD <filename>
# Or in newer Git:
git restore --staged <filename>
```

**Restore file to specific commit:**
```bash
git checkout <commit-hash> -- <filename>
```

### 5. Recover Deleted Branch

```bash
# Find the last commit of deleted branch
git reflog

# Look for entries like "checkout: moving from deleted-branch to main"
# Note the SHA before deletion

# Recreate the branch
git checkout -b recovered-branch <sha>
```

### 6. Recover Lost Commits

```bash
# Git reflog shows ALL recent HEAD movements
git reflog

# Find your lost commit
# Example output:
# abc1234 HEAD@{0}: reset: moving to HEAD~3
# def5678 HEAD@{1}: commit: Important work  <-- This one!

# Recover it
git checkout -b recovery-branch def5678
# Or cherry-pick it
git cherry-pick def5678
```

### 7. Undo a Merge

**If not pushed:**
```bash
git reset --hard HEAD~1
# Or to specific commit before merge:
git reset --hard <commit-before-merge>
```

**If already pushed:**
```bash
git revert -m 1 <merge-commit-hash>
git push
```

### 8. Committed Sensitive Data

**If not pushed:**
```bash
# Remove from last commit
git reset --soft HEAD~1
# Remove sensitive file
git rm --cached <sensitive-file>
# Add to .gitignore
echo "sensitive-file" >> .gitignore
# Recommit
git add .
git commit -m "Remove sensitive data"
```

**If already pushed (requires rewriting history):**
```bash
# Use git-filter-repo (install first)
git filter-repo --path <sensitive-file> --invert-paths

# Force push all branches (coordinate with team!)
git push --force --all
```

### 9. Undo Staged Changes

```bash
# Unstage everything
git reset

# Unstage specific file
git reset HEAD <file>

# Keep file staged but undo modifications
git checkout HEAD -- <file>
```

### 10. Completely Start Over

```bash
# Discard ALL local changes and commits
git fetch origin
git reset --hard origin/main

# Or clean untracked files too
git clean -fd
```

## The Reflog Safety Net

Git reflog is your time machine. It tracks every HEAD movement for ~90 days.

```bash
# See recent history
git reflog

# See with timestamps
git reflog --date=relative

# Recover ANY previous state
git reset --hard HEAD@{5}  # Go back 5 operations
```

## Best Practices

1. **Before dangerous operations:**
   ```bash
   git stash  # Save work in progress
   # or
   git branch backup-branch  # Create safety branch
   ```

2. **Use `--force-with-lease` instead of `--force`:**
   ```bash
   git push --force-with-lease  # Fails if remote changed
   ```

3. **Check before reset:**
   ```bash
   git log --oneline -10  # See what you're about to affect
   ```

## Common Mistakes to Avoid

- Using `--force` on shared branches
- Running `reset --hard` without checking `git stash list`
- Forgetting that `reflog` is local only
- Not making a backup branch before risky operations
