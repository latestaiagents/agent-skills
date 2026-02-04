---
name: git-history-detective
description: |
  Use this skill when investigating Git history. Activate when the user wants to find when a bug was introduced,
  who changed a line of code, track down a regression, use git bisect, search commit history, understand
  why code was changed, or investigate when and how something broke.
---

# Git History Detective

Find when bugs were introduced and understand code evolution.

## When to Use

- Finding when a bug was introduced
- Understanding why code was changed
- Finding who last modified a line
- Tracking down a regression
- Searching for commits by content

## Quick Reference

| Task | Command |
|------|---------|
| Who changed this line? | `git blame <file>` |
| Find commit by message | `git log --grep="keyword"` |
| Find commit by code change | `git log -S "code"` |
| Binary search for bug | `git bisect` |
| See file at old commit | `git show <commit>:<file>` |

## Investigation Techniques

### 1. Git Blame - Find Line Authors

```bash
# Basic blame
git blame <file>

# Blame specific lines
git blame -L 10,20 <file>

# Ignore whitespace changes
git blame -w <file>

# Show original author (ignore moves/copies)
git blame -M -C <file>

# Blame with email
git blame -e <file>
```

**Reading blame output:**
```
abc1234 (John Doe 2024-01-15 10:30:45 +0000 42) function processData() {
│        │         │                       │    │
│        │         │                       │    └─ Line content
│        │         │                       └─ Line number
│        │         └─ Timestamp
│        └─ Author
└─ Commit SHA (first 7 chars)
```

### 2. Git Log - Search History

```bash
# Search commit messages
git log --grep="fix login"

# Search code that was added/removed
git log -S "functionName"  # "pickaxe" search

# Search with regex
git log -G "function.*deprecated"

# Find commits touching a file
git log --follow -- <file>

# Find commits by author
git log --author="John"

# Find commits in date range
git log --since="2024-01-01" --until="2024-02-01"

# Combine searches
git log --author="John" --grep="refactor" --since="2024-01-01"
```

### 3. Git Bisect - Binary Search for Bugs

When you know a bug exists now but didn't before:

```bash
# Start bisect
git bisect start

# Mark current (broken) commit as bad
git bisect bad

# Mark known good commit
git bisect good v1.2.0  # or a commit SHA

# Git checks out middle commit
# Test it, then:
git bisect good  # if bug NOT present
git bisect bad   # if bug IS present

# Repeat until Git finds the culprit
# "abc1234 is the first bad commit"

# End bisect
git bisect reset
```

**Automated bisect with test script:**
```bash
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run npm test  # or any command that exits 0 for good, 1 for bad
```

### 4. Track File Changes Over Time

```bash
# See all commits that touched a file
git log --oneline -- <file>

# See actual changes in each commit
git log -p -- <file>

# Follow file through renames
git log --follow -p -- <file>

# Show file at specific commit
git show <commit>:<file>

# Compare file between commits
git diff <commit1> <commit2> -- <file>
```

### 5. Find Deleted Code

```bash
# Find when code was deleted
git log -S "deletedFunction" --diff-filter=D

# Find deleted file
git log --all --full-history -- "**/deleted-file.js"

# Restore deleted file
git checkout <commit-before-deletion>^ -- <file>
```

## Investigation Workflow

### Scenario: "This feature used to work"

**Step 1: Identify the symptom**
```bash
# Find when the feature last worked
# Check release tags, deployment dates
git tag -l --sort=-version:refname | head -10
```

**Step 2: Narrow the time range**
```bash
# Find commits between working and broken state
git log --oneline v1.2.0..HEAD -- src/feature/
```

**Step 3: Use bisect to pinpoint**
```bash
git bisect start
git bisect bad HEAD
git bisect good v1.2.0
# Test each checkout
```

**Step 4: Analyze the culprit commit**
```bash
git show <culprit-commit>
git log -1 --format="%B" <culprit-commit>  # Full message
```

### Scenario: "Who broke this?"

```bash
# Find who last touched the broken code
git blame -L 45,50 src/broken-file.js

# See the full commit
git show <commit-from-blame>

# See what else changed in that commit
git show --stat <commit>
```

### Scenario: "When was this function added?"

```bash
# Find first commit with the function
git log --reverse -S "functionName" --oneline

# See the full commit
git show <first-commit>
```

### Scenario: "Why was this changed?"

```bash
# Get the commit that last changed this line
git blame -L 42,42 <file>

# See full commit message (hopefully explains why)
git show <commit>

# See related commits around that time
git log --oneline --since="2024-01-01" --until="2024-01-15" -- <file>
```

## Advanced Techniques

### Find Commit That Introduced Regex Match
```bash
git log -G "TODO:.*hack" --oneline
```

### Compare Branches
```bash
# Commits in feature not in main
git log main..feature --oneline

# Commits in either but not both
git log main...feature --oneline
```

### Visualize History
```bash
# ASCII graph
git log --graph --oneline --all

# Show merge history
git log --first-parent --oneline
```

### Search Across All Branches
```bash
git log --all -S "searchTerm"
git branch --contains <commit>
```

## Tools Integration

```bash
# Open blame in browser (GitHub)
gh browse --blame <file>

# Interactive blame in VS Code
# Install GitLens extension

# GUI tools
gitk --all  # Built-in
tig        # Terminal UI
```

## Common Patterns

### Find Security Vulnerabilities Introduced
```bash
git log -S "eval(" --oneline
git log -S "innerHTML" --oneline
git log -G "password.*=.*['\"]" --oneline
```

### Find When Tests Started Failing
```bash
git bisect start
git bisect bad HEAD
git bisect good <last-known-good>
git bisect run npm test
```

### Track Refactoring Impact
```bash
# Find the refactoring commit
git log --grep="refactor" --oneline -- src/

# See what files it touched
git show --stat <refactor-commit>

# Compare before/after
git diff <commit>^..<commit>
```
