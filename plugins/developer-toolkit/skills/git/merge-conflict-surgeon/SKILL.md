---
name: merge-conflict-surgeon
description: |
  Use this skill when resolving Git merge conflicts. Activate when the user mentions merge conflicts,
  conflicting changes, failed merges, "both modified", HEAD markers, conflict markers (<<<<<<<, =======, >>>>>>>),
  or asks how to resolve conflicts between branches. Also use when git merge or git rebase fails due to conflicts.
---

# Merge Conflict Surgeon

Resolve Git merge conflicts systematically and safely.

## When to Use

- `git merge` or `git rebase` fails with conflicts
- You see conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) in files
- `git status` shows "both modified" files
- Merging feature branches into main/develop
- Rebasing a branch onto updated main

## Workflow

### Step 1: Assess the Damage

```bash
# See all conflicted files
git status

# See the full diff of conflicts
git diff --name-only --diff-filter=U
```

List all conflicted files and categorize them:
- **Simple conflicts**: Same file, different lines (auto-resolvable logic)
- **Overlap conflicts**: Same lines modified differently (needs human decision)
- **Delete/modify conflicts**: One side deleted, other modified

### Step 2: Understand Both Sides

For each conflicted file, understand:

```bash
# See what YOUR branch changed
git diff HEAD~1 -- <file>

# See what the OTHER branch changed
git diff MERGE_HEAD~1 -- <file>

# See common ancestor
git show :1:<file>  # Base version
git show :2:<file>  # Ours (current branch)
git show :3:<file>  # Theirs (merging branch)
```

### Step 3: Resolve Strategically

**For simple conflicts (different functionality):**
- Keep both changes, ensure they don't break each other
- Run the code mentally or actually to verify

**For overlapping conflicts (same code, different changes):**
1. Understand the INTENT of each change
2. Determine which achieves the goal better
3. Sometimes combine logic from both
4. Never blindly pick "ours" or "theirs"

**For delete/modify conflicts:**
- If deleted intentionally (refactoring), ensure the modification isn't needed
- If deleted accidentally, restore and apply modification

### Step 4: Validate Resolution

```bash
# Mark as resolved
git add <resolved-file>

# Verify no remaining conflicts
git diff --check

# Run tests before completing
npm test  # or your test command

# Complete the merge
git commit  # or git rebase --continue
```

## Resolution Patterns

### Pattern 1: Sequential Changes
```
<<<<<<< HEAD
function processUser(user) {
  validateUser(user);
=======
function processUser(user) {
  logUserAccess(user);
>>>>>>> feature-branch
```
**Resolution:** Keep both - they're independent additions
```javascript
function processUser(user) {
  validateUser(user);
  logUserAccess(user);
```

### Pattern 2: Competing Implementations
```
<<<<<<< HEAD
const result = items.filter(x => x.active).map(x => x.name);
=======
const result = items.reduce((acc, x) => x.active ? [...acc, x.name] : acc, []);
>>>>>>> feature-branch
```
**Resolution:** Choose based on readability/performance needs. The `filter().map()` is clearer.

### Pattern 3: Configuration Changes
```
<<<<<<< HEAD
const API_URL = 'https://api.prod.example.com';
=======
const API_URL = 'https://api.staging.example.com';
>>>>>>> feature-branch
```
**Resolution:** This is environment-specific. Use environment variables instead:
```javascript
const API_URL = process.env.API_URL || 'https://api.prod.example.com';
```

## Tools

```bash
# Use a merge tool (if configured)
git mergetool

# Abort if things go wrong
git merge --abort
git rebase --abort

# Accept all of one side (use carefully)
git checkout --ours <file>    # Keep your version
git checkout --theirs <file>  # Keep their version
```

## Common Mistakes to Avoid

- Never blindly accept "ours" or "theirs" without understanding the changes
- Don't leave conflict markers in code (<<<<<<, =======, >>>>>>>)
- Don't skip running tests after resolution
- Don't force push to shared branches after rebase conflicts
- Don't ignore whitespace or formatting conflicts - they can hide real issues

## Emergency Recovery

```bash
# If you messed up the resolution
git checkout -m <file>  # Restore conflict markers

# If you committed a bad resolution
git reset --soft HEAD~1  # Undo commit, keep changes
# Then resolve again

# Nuclear option: start over
git merge --abort  # or git rebase --abort
```
