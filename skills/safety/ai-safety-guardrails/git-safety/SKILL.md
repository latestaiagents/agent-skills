---
name: git-safety
description: |
  CRITICAL SAFETY SKILL - Use this skill before any Git operation that could lose commits, rewrite history,
  or affect shared branches. Activate when the user wants to force push, reset hard, rebase, clean, or
  perform any destructive Git operation. This skill prevents loss of work through verification and confirmation.
---

# Git Safety

**CRITICAL: Some Git operations can permanently lose work. This skill prevents commit loss.**

## Danger Levels

| Level | Operations | Requires |
|-------|------------|----------|
| ⛔ CRITICAL | `push --force` to shared branch, `reset --hard` with unpushed work | Full verification + explicit confirmation |
| 🔴 HIGH | `rebase` on shared branches, `clean -fd`, `checkout .` | Stash check + confirmation |
| 🟠 MEDIUM | `reset --soft/mixed`, `stash drop`, `branch -D` | Verify branch state + confirm |
| 🟢 LOW | `commit`, `push`, `pull`, `fetch`, `branch` | Standard execution |

## Pre-Operation Protocol

### Before Force Push

```markdown
⛔ **CRITICAL: FORCE PUSH REQUESTED**

**Command:** git push --force origin main

**⚠️ THIS REWRITES SHARED HISTORY**

**Current state:**
- Local branch: main
- Local HEAD: abc1234 "Add new feature"
- Remote HEAD: def5678 "Fix bug in login"

**Commits that will be OVERWRITTEN on remote:**
1. def5678 - "Fix bug in login" (by teammate@company.com, 2 hours ago)
2. ghi9012 - "Update documentation" (by teammate@company.com, 3 hours ago)

**These commits will be LOST if not saved elsewhere:**
- 2 commits by other team members
- Anyone who pulled these will have conflicts

**Safety checks:**
- [ ] Is this your personal branch? NO - this is main ⚠️
- [ ] Have you notified the team? Unknown
- [ ] Are the overwritten commits backed up? Unknown

**Safer alternatives:**
1. `git push --force-with-lease` (fails if remote changed)
2. Create a new branch for your changes
3. Merge instead of rebase
4. Coordinate with team first

**I strongly recommend against force pushing to main.**

If you must proceed, type: "yes, force push to main and overwrite 2 commits"
```

### Before Reset --hard

```markdown
⛔ **CRITICAL: HARD RESET REQUESTED**

**Command:** git reset --hard HEAD~3

**⚠️ THIS DISCARDS CHANGES PERMANENTLY**

**What will be LOST:**

**Commits to be removed (3):**
1. abc1234 - "Work in progress on feature X"
2. def5678 - "Add unit tests"
3. ghi9012 - "Fix styling issues"

**Uncommitted changes that will be LOST:**
- Modified: src/App.tsx (45 lines changed)
- Modified: src/utils/helper.ts (12 lines changed)
- Untracked: src/components/NewFeature.tsx (new file, 156 lines)

**Recovery options after reset:**
- Commits: Can recover via `git reflog` (within 90 days)
- Staged changes: Can recover via `git fsck --lost-found`
- Untracked files: PERMANENTLY LOST ⚠️

**Before reset, I will:**
1. Stash uncommitted changes: `git stash push -m "backup before reset"`
2. Create backup branch: `git branch backup-before-reset`
3. Note reflog position: HEAD is currently at abc1234

Proceed with backup and reset? Type "yes, reset after backup" to confirm.
```

### Before Clean

```markdown
🔴 **DANGEROUS: GIT CLEAN REQUESTED**

**Command:** git clean -fd

**This will DELETE untracked files and directories:**

| Type | Path | Size |
|------|------|------|
| File | .env.local | 1.2 KB ⚠️ (may contain secrets) |
| File | config.local.json | 0.5 KB |
| Dir | node_modules/ | 234 MB (can be recreated) |
| Dir | uploads/ | 45 MB ⚠️ (user uploaded files) |
| File | notes.txt | 2 KB (your notes) |

**Files that will be PRESERVED (tracked):**
- All committed files
- All files in .gitignore that are tracked

**⚠️ WARNING: These untracked files cannot be recovered after deletion:**
- .env.local (may contain API keys!)
- uploads/ (user data!)
- notes.txt (your work!)

**Safer alternatives:**
```bash
# See what would be deleted (dry run)
git clean -fdn

# Interactive mode (choose what to delete)
git clean -fdi

# Exclude certain patterns
git clean -fd -e "*.local" -e "uploads/"
```

Type "yes, delete untracked files" to proceed, or use a safer alternative above.
```

### Before Rebase on Shared Branch

```markdown
🔴 **DANGEROUS: REBASE ON SHARED BRANCH**

**Command:** git rebase main feature-branch

**⚠️ REBASE REWRITES HISTORY**

**Branch state:**
- feature-branch has been pushed: Yes
- feature-branch has other contributors: Checking...
  - You: 5 commits
  - teammate@company.com: 2 commits ⚠️

**If you rebase this branch:**
- All 7 commits will get new SHA hashes
- Your teammate will have to force-reset their local branch
- Any PRs referencing old commits will break
- CI/CD caches may be invalidated

**Safer alternatives:**
1. **Merge instead:** `git merge main` (preserves history)
2. **Rebase only your commits:**
   ```bash
   git rebase --onto main HEAD~5  # Only your 5 commits
   ```
3. **Create new branch:**
   ```bash
   git checkout -b feature-branch-v2
   git rebase main
   ```

If you must rebase, notify teammates first and type: "yes, rebase shared branch"
```

### Before Checkout . (Discard All Changes)

```markdown
🔴 **DANGEROUS: DISCARD ALL CHANGES**

**Command:** git checkout . (or git restore .)

**This will DISCARD all uncommitted changes:**

**Modified files (15):**
| File | Lines Changed |
|------|---------------|
| src/App.tsx | +45, -12 |
| src/components/Header.tsx | +23, -5 |
| src/utils/api.ts | +67, -34 |
| ... and 12 more files |

**Total changes to be discarded:**
- 234 lines added
- 89 lines removed
- 15 files modified

**These changes are NOT committed and will be PERMANENTLY LOST.**

**Safer alternatives:**
```bash
# Stash changes (can be recovered later)
git stash push -m "work in progress"

# Discard specific file only
git checkout -- src/App.tsx

# See changes first
git diff
```

Type "yes, discard all uncommitted changes" to proceed.
```

### Before Branch -D (Force Delete)

```markdown
🟠 **WARNING: FORCE DELETE BRANCH**

**Command:** git branch -D feature/experimental

**Branch state:**
- Local branch: feature/experimental
- Commits ahead of main: 8
- Merged to main: No ⚠️
- Has remote tracking: No
- Last commit: 2 days ago

**Commits that may be LOST (not on any other branch):**
1. abc1234 - "Experimental feature WIP"
2. def5678 - "Add database schema"
3. ... and 6 more commits

**Recovery:**
- Can recover via `git reflog` within 90 days
- Cannot recover if git gc runs

**Safer alternatives:**
```bash
# Merge first, then delete
git checkout main
git merge feature/experimental
git branch -d feature/experimental  # -d fails if not merged

# Archive the branch instead
git tag archive/feature-experimental feature/experimental
git branch -D feature/experimental
```

Type "yes, delete unmerged branch" to proceed.
```

### Before Stash Drop

```markdown
🟠 **WARNING: DROP STASH**

**Command:** git stash drop stash@{0}

**Stash contents:**
- Created: 2024-01-15 14:30:00 (3 days ago)
- Message: "WIP: payment integration"
- Branch: feature/payments
- Files:
  - src/services/payment.ts (modified)
  - src/types/payment.ts (new file)
  - tests/payment.test.ts (modified)

**Changes in this stash:**
- 156 lines added
- 23 lines removed

**Recovery after drop:**
- Very difficult (requires git fsck --lost-found)
- May not be recoverable after git gc

**Safer alternatives:**
```bash
# Apply stash first, then drop
git stash apply stash@{0}
# ... verify changes look good ...
git stash drop stash@{0}

# List all stashes to make sure dropping the right one
git stash list
```

Type "yes, drop stash" to proceed.
```

## Safe Git Workflow

### Before Any Destructive Operation

```bash
# 1. Check current state
git status
git log --oneline -5
git stash list

# 2. Create safety backup
git stash push -m "safety backup $(date +%Y%m%d_%H%M%S)"
git branch backup-$(date +%Y%m%d_%H%M%S)

# 3. Note reflog position
git reflog -1  # Remember this hash!
```

### Recovery Commands

```bash
# Recover from reset --hard
git reflog                      # Find the commit before reset
git reset --hard HEAD@{1}       # Go back to that commit

# Recover deleted branch
git reflog                      # Find the branch's last commit
git checkout -b recovered-branch abc1234

# Recover dropped stash
git fsck --lost-found           # Find dangling commits
git show abc1234                # Check if it's your stash
git stash apply abc1234

# Recover from clean
# Unfortunately, untracked files are gone
# Check IDE local history, Time Machine, cloud backups
```

## Protected Branches Configuration

### Git Hooks for Safety

```bash
# .git/hooks/pre-push
#!/bin/bash

protected_branches=("main" "master" "production" "develop")
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

for branch in "${protected_branches[@]}"; do
  if [ "$current_branch" = "$branch" ]; then
    if [[ "$@" == *"--force"* ]] || [[ "$@" == *"-f"* ]]; then
      echo "⛔ Force push to $branch is BLOCKED"
      echo "Create a PR instead, or use --force-with-lease"
      exit 1
    fi
  fi
done

exit 0
```

### Git Aliases for Safety

```bash
# Add to ~/.gitconfig
[alias]
  # Safe force push
  pushf = push --force-with-lease

  # Reset with backup
  safe-reset = "!f() { git branch backup-$(date +%s) && git reset \"$@\"; }; f"

  # Clean with preview
  safe-clean = clean -fdn

  # Delete branch only if merged
  safe-delete = branch -d

  # Show what would be rebased
  rebase-preview = "!f() { git log --oneline HEAD...$1; }; f"
```

## Best Practices Summary

| DO | DON'T |
|----|-------|
| Use `--force-with-lease` | Use `--force` on shared branches |
| Stash before reset | Reset with uncommitted changes |
| Create backup branches | Delete unmerged branches carelessly |
| Use `-d` for branch delete | Use `-D` without checking merge status |
| Run `clean -n` first (dry run) | Run `clean -fd` blindly |
| Check `git status` before operations | Assume working directory is clean |
| Use `git reflog` for recovery | Panic after mistakes |
| Communicate before rebasing shared branches | Rebase and force push silently |

## Pre-Flight Checklist

Before destructive Git operations:
- [ ] Current branch is correct (`git branch`)
- [ ] No uncommitted changes OR they're stashed (`git status`)
- [ ] Backup branch created if needed
- [ ] Not operating on protected/shared branch
- [ ] Understand what will be affected
- [ ] Know how to recover if something goes wrong
- [ ] Team notified if affecting shared branches
