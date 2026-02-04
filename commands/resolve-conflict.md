---
description: Systematic merge conflict resolution with context analysis
---

# /resolve-conflict

Systematic merge conflict resolution. I'll analyze each conflict and help you resolve it correctly.

## Step 1: Identify All Conflicts

```bash
# List all conflicted files
git diff --name-only --diff-filter=U
```

## Step 2: Analyze Each Conflict

For each conflicted file, I'll show you:

1. **What changed in YOUR branch** (HEAD)
2. **What changed in THEIR branch** (incoming)
3. **What the common ancestor looked like** (base)
4. **Recommended resolution** with reasoning

### Conflict Format Explained

```
<<<<<<< HEAD
Your changes (current branch)
=======
Their changes (incoming branch)
>>>>>>> branch-name
```

## Step 3: Resolution Strategy

For each conflict, I'll recommend one of:

| Strategy | When to Use |
|----------|-------------|
| **Keep Ours** | Your changes are correct, discard theirs |
| **Keep Theirs** | Their changes are correct, discard yours |
| **Merge Both** | Both changes are needed, combine them |
| **Rewrite** | Neither is right, write new code |

## Step 4: Resolve Systematically

For each file:

```bash
# View the conflict in context
git diff --cc filename

# After resolving:
git add filename
```

## Step 5: Complete the Merge

```bash
# Verify all conflicts resolved
git diff --name-only --diff-filter=U  # Should be empty

# Complete the merge
git commit -m "Merge branch 'feature' - resolved conflicts in [files]"
```

## Conflict Prevention Tips

1. **Pull frequently** - Smaller, more frequent merges = fewer conflicts
2. **Communicate** - Know who's working on what files
3. **Use feature flags** - Merge incomplete features safely
4. **Rebase before merge** - `git pull --rebase origin main`

---

**Show me the conflicted file(s) and I'll analyze each conflict.**
