---
description: Systematic debugging assistance for any error or bug
---

# /debug-this

Get systematic help debugging any issue.

## What I Need

Tell me:
- What's the error message or unexpected behavior?
- What did you expect to happen?
- What have you tried so far?
- Share relevant code/logs if possible

## Debug Process

### Step 1: Understand the Problem
I'll help clarify:
- Error type and meaning
- Likely root causes
- What to investigate first

### Step 2: Gather Information
We'll collect:
- Full stack traces
- Relevant logs
- Input data that triggers the issue
- Environment details

### Step 3: Isolate the Cause
Systematic approach:
- Binary search through code changes
- Minimal reproduction case
- Hypothesis testing

### Step 4: Fix and Verify
I'll provide:
- Explanation of root cause
- Fix with code
- How to verify the fix
- Prevention measures

## Quick Debug Commands

```bash
# Node.js - Enable debug logging
DEBUG=* node app.js

# Python - Run with verbose tracing
python -m trace --trace script.py

# Check recent git changes
git log --oneline -10
git diff HEAD~1

# Search codebase for pattern
grep -r "error_pattern" --include="*.js"
```

## Common Issues

| Error Type | First Check |
|------------|-------------|
| TypeError | Variable types, null checks |
| Import Error | Module paths, dependencies |
| Connection Error | Network, credentials, URLs |
| Permission Error | File/folder permissions |
| Memory Error | Data size, leaks, limits |
