---
description: Structured debugging workflow - hypothesize, verify, fix
---

# /debug-this

Structured debugging workflow. Describe the bug and I'll help you find and fix it.

## Describe the Bug

Tell me:
1. **What should happen?** (expected behavior)
2. **What actually happens?** (actual behavior)
3. **When did it start?** (recent changes?)
4. **Error messages?** (paste any errors)

## Debugging Workflow

### Step 1: Reproduce the Bug

First, I need to reliably trigger the bug:

```markdown
## Reproduction Steps
1. [Step to reproduce]
2. [Step to reproduce]
3. [Bug occurs]

**Reproducible:** Always / Sometimes / Rarely
**Environment:** Dev / Staging / Prod
**Affected users:** All / Some / One
```

### Step 2: Form Hypotheses

Based on symptoms, I'll generate hypotheses ranked by likelihood:

```markdown
## Hypotheses

1. **[Most likely]** Description
   - Evidence for: ...
   - Evidence against: ...
   - How to verify: ...

2. **[Second likely]** Description
   - Evidence for: ...
   - Evidence against: ...
   - How to verify: ...

3. **[Third likely]** Description
   - ...
```

### Step 3: Gather Evidence

For each hypothesis, I'll suggest specific checks:

```typescript
// Add diagnostic logging
console.log('[DEBUG] Variable state:', { var1, var2, var3 });

// Check database state
SELECT * FROM orders WHERE id = 123;

// Verify API responses
curl -v https://api.example.com/endpoint

// Check logs
grep "ERROR\|Exception" /var/log/app.log | tail -50
```

### Step 4: Isolate the Cause

Narrow down using:

| Technique | When to Use |
|-----------|-------------|
| **Binary search** | Bug in large codebase, use git bisect |
| **Simplify inputs** | Bug depends on specific data |
| **Remove components** | Bug might be in dependencies |
| **Fresh environment** | Bug might be env-specific |

### Step 5: Fix and Verify

```markdown
## Fix

**Root cause:** [What was actually wrong]

**Fix:** [Code change]

**Verification:**
- [ ] Bug no longer reproduces
- [ ] Related features still work
- [ ] Tests pass
- [ ] No new errors in logs
```

### Step 6: Prevent Recurrence

```markdown
## Prevention

1. **Add test:** [Test that would catch this bug]
2. **Add validation:** [Input validation to prevent bad state]
3. **Add monitoring:** [Alert if this happens again]
4. **Update docs:** [If this was confusing]
```

## Common Bug Patterns

| Symptom | Likely Cause |
|---------|--------------|
| Works locally, fails in prod | Environment variables, permissions, network |
| Intermittent failures | Race condition, timing, external service |
| Works for some users | Data-dependent bug, permissions, cache |
| Started after deploy | Check recent commits with `git log` |
| Null/undefined errors | Missing data, async timing, optional fields |
| Performance degradation | N+1 queries, memory leak, missing index |

## Quick Diagnostic Commands

```bash
# Recent git changes
git log --oneline -20
git diff HEAD~5 --stat

# Check logs
tail -100 /var/log/app.log | grep -i error

# Check resources
top -l 1 | head -10  # CPU/Memory
df -h                 # Disk space
netstat -an | grep LISTEN  # Open ports

# Database
EXPLAIN ANALYZE SELECT ...;  # Query performance
```

---

**Describe the bug and I'll start the systematic debugging process.**
