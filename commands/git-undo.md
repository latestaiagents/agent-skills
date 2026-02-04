---
description: Recover from any Git mistake - reset, rebase, force push, deleted branches
---

# /git-undo

Recover from any Git mistake. Tell me what happened and I'll help you fix it.

## What I Need From You

Describe what went wrong:
- "I accidentally committed to main"
- "I did a hard reset and lost my changes"
- "I force pushed and overwrote my colleague's work"
- "I deleted a branch with unmerged commits"
- "I rebased and now everything is messed up"

## Recovery Workflow

### Step 1: Assess the Situation

First, I'll check your current Git state:

```bash
# Current branch and status
git status

# Recent history
git log --oneline -10

# Reflog (your safety net)
git reflog -20
```

### Step 2: Identify Recovery Path

Based on your situation, I'll recommend one of these recovery methods:

| Situation | Recovery Method |
|-----------|-----------------|
| Committed to wrong branch | `git cherry-pick` or `git reset --soft` |
| Lost commits after reset | `git reflog` + `git reset` |
| Force push overwrote remote | `git reflog` + coordinate with team |
| Deleted branch | `git reflog` + `git checkout -b` |
| Bad rebase | `git reflog` + `git reset --hard ORIG_HEAD` |
| Accidentally staged files | `git restore --staged` |
| Want to undo last commit | `git reset --soft HEAD~1` |

### Step 3: Execute Recovery

I'll provide the exact commands with safety checks:

1. **Create backup branch first** (always)
   ```bash
   git branch backup-$(date +%Y%m%d-%H%M%S)
   ```

2. **Execute recovery commands**

3. **Verify the fix worked**
   ```bash
   git log --oneline -5
   git status
   ```

## Common Scenarios

### "I committed to main instead of my feature branch"

```bash
# Save the commit hash
COMMIT=$(git rev-parse HEAD)

# Reset main to before your commit
git reset --hard HEAD~1

# Switch to feature branch and apply commit
git checkout feature-branch
git cherry-pick $COMMIT
```

### "I lost commits after git reset --hard"

```bash
# Find lost commits in reflog
git reflog

# Look for your commit message, then:
git reset --hard HEAD@{n}  # n = number from reflog
```

### "I need to undo a pushed commit"

```bash
# Create a revert commit (safe for shared branches)
git revert HEAD
git push
```

## Tell Me What Happened

Describe your situation and I'll provide the exact recovery steps.
