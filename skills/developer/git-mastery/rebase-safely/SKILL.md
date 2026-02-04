---
name: rebase-safely
description: |
  Use this skill when rebasing Git branches. Activate when the user mentions rebase, interactive rebase,
  squash commits, reorder commits, edit commit history, clean up commits before merge, rebase onto main,
  or fixing up commit history. Also use when rebase fails or causes conflicts.
---

# Rebase Safely

Master Git rebase without losing work or causing team chaos.

## When to Use

- Updating feature branch with latest main
- Cleaning up commit history before PR
- Squashing multiple commits into one
- Reordering or editing commits
- Fixing commit messages

## The Golden Rule

**NEVER rebase commits that have been pushed to a shared branch.**

Rebasing rewrites history. If others have based work on those commits, you'll cause merge conflicts and confusion.

```
Safe to rebase:
✓ Local commits not yet pushed
✓ Your own feature branch (not yet merged)
✓ Commits only you have seen

NOT safe to rebase:
✗ main/develop branch
✗ Shared feature branches
✗ Already merged commits
```

## Basic Rebase Operations

### Update Feature Branch with Main

```bash
# Fetch latest
git fetch origin

# Rebase onto main
git checkout feature-branch
git rebase origin/main

# If conflicts, resolve then:
git add <resolved-files>
git rebase --continue

# Force push YOUR branch only
git push --force-with-lease
```

### Interactive Rebase - Clean Up History

```bash
# Rebase last 3 commits
git rebase -i HEAD~3

# Rebase from where branch diverged
git rebase -i main
```

**Interactive rebase commands:**
```
pick   = use commit as-is
reword = use commit, but edit message
edit   = use commit, but stop to amend
squash = meld into previous commit (keep message)
fixup  = meld into previous commit (discard message)
drop   = remove commit entirely
```

## Common Scenarios

### 1. Squash Multiple Commits

Before:
```
abc123 fix: typo in login
def456 fix: another typo
ghi789 feat: add login page
```

```bash
git rebase -i HEAD~3
```

Change to:
```
pick ghi789 feat: add login page
squash def456 fix: another typo
squash abc123 fix: typo in login
```

Result: One clean commit with combined message.

### 2. Reorder Commits

```bash
git rebase -i HEAD~3
```

Just change the order of pick lines:
```
pick ghi789 third commit
pick abc123 first commit  # moved down
pick def456 second commit # moved down
```

### 3. Edit a Past Commit

```bash
git rebase -i HEAD~3
```

Change `pick` to `edit` for the commit you want to change:
```
pick abc123 first commit
edit def456 second commit  # stop here
pick ghi789 third commit
```

Make your changes, then:
```bash
git add <files>
git commit --amend
git rebase --continue
```

### 4. Fix Commit Message

```bash
git rebase -i HEAD~3
```

Change `pick` to `reword`:
```
pick abc123 first commit
reword def456 bad message  # will prompt for new message
pick ghi789 third commit
```

### 5. Split a Commit

```bash
git rebase -i HEAD~2
```

Mark commit as `edit`:
```
edit abc123 big commit to split
pick def456 next commit
```

Then:
```bash
# Undo the commit but keep changes
git reset HEAD^

# Create multiple commits
git add file1.js
git commit -m "feat: first part"
git add file2.js
git commit -m "feat: second part"

git rebase --continue
```

## Handling Conflicts

### During Rebase
```bash
# See conflicted files
git status

# Resolve conflicts in each file
# Remove conflict markers (<<<<, ====, >>>>)

# Stage resolved files
git add <file>

# Continue rebase
git rebase --continue

# If it's too messy, abort
git rebase --abort
```

### Conflict Tips

1. **Understand both sides:**
   ```bash
   git show HEAD:<file>     # Your version
   git show REBASE_HEAD:<file>  # Incoming version
   ```

2. **Use merge tool:**
   ```bash
   git mergetool
   ```

3. **Skip problematic commit (careful!):**
   ```bash
   git rebase --skip
   ```

## Safe Rebase Workflow

### Before Starting
```bash
# Create backup branch
git branch backup-feature-branch

# Make sure working directory is clean
git status
```

### During Rebase
```bash
# Check progress
git status

# See what's being applied
git log --oneline REBASE_HEAD~1..REBASE_HEAD
```

### After Rebase
```bash
# Compare with backup to ensure nothing lost
git diff backup-feature-branch feature-branch

# Delete backup when satisfied
git branch -d backup-feature-branch
```

## Rebase vs Merge

| Aspect | Rebase | Merge |
|--------|--------|-------|
| History | Linear, clean | Preserves all commits |
| Conflicts | Resolved per-commit | Resolved once |
| Safe for shared branches | No | Yes |
| Traceability | Less (rewrites) | More (merge commits) |

**Use rebase for:** Local cleanup, updating feature branches
**Use merge for:** Integrating shared branches, preserving history

## Autosquash - Quick Fixups

```bash
# Create a fixup commit for previous commit
git commit --fixup=<commit-to-fix>

# Autosquash during rebase
git rebase -i --autosquash main
```

The fixup commits automatically get positioned correctly.

## Recovering from Bad Rebase

### Using Reflog
```bash
# Find pre-rebase state
git reflog

# Look for entry like "rebase (start)"
# Find the commit BEFORE that

# Reset to pre-rebase state
git reset --hard HEAD@{5}  # or specific SHA
```

### Using ORIG_HEAD
```bash
# Immediately after rebase, ORIG_HEAD points to pre-rebase
git reset --hard ORIG_HEAD
```

## Common Mistakes to Avoid

- **Rebasing shared branches** - This breaks others' history
- **Force pushing without lease** - Use `--force-with-lease`
- **Not creating backup** - Always have an escape route
- **Rebasing huge commit ranges** - Do smaller rebases
- **Ignoring conflicts** - Each conflict needs careful resolution
- **Rebasing merge commits** - Gets complicated, prefer `--rebase-merges`

## Team Guidelines

```markdown
## Rebase Policy

1. Rebase your feature branch onto main before creating PR
2. Never rebase after PR is opened (unless you're the only reviewer)
3. Squash fixup commits before merge
4. Use --force-with-lease, never --force
5. If unsure, ask before rebasing
```
