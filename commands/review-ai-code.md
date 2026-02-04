---
description: Review AI-generated code for bugs, security issues, and best practices
---

# /review-ai-code

Systematic review of AI-generated code for bugs, security issues, and best practices.

## What to Review

Paste the AI-generated code, or point me to the file(s) to review.

## Review Checklist

I'll check for these common AI code issues:

### 1. Correctness Issues

| Issue | What I Look For |
|-------|-----------------|
| **Hallucinated APIs** | Methods/functions that don't exist |
| **Wrong signatures** | Incorrect parameter types or order |
| **Logic errors** | Off-by-one, wrong comparisons, missing cases |
| **Incomplete handling** | Missing error cases, edge cases |
| **Outdated patterns** | Deprecated APIs, old syntax |

### 2. Security Issues

| Issue | What I Look For |
|-------|-----------------|
| **Injection vulnerabilities** | SQL injection, XSS, command injection |
| **Hardcoded secrets** | API keys, passwords in code |
| **Insecure defaults** | Missing auth, permissive CORS |
| **Data exposure** | Logging sensitive data, verbose errors |

### 3. Quality Issues

| Issue | What I Look For |
|-------|-----------------|
| **Over-engineering** | Unnecessary abstractions |
| **Under-engineering** | Missing validation, error handling |
| **Inconsistent style** | Doesn't match codebase conventions |
| **Poor naming** | Unclear variable/function names |
| **Missing types** | TypeScript any, missing interfaces |

## Review Process

### Step 1: Quick Scan

```markdown
## First Impressions

- [ ] Code compiles/runs
- [ ] Imports exist
- [ ] Functions are called correctly
- [ ] Types are correct
```

### Step 2: Line-by-Line Review

I'll annotate issues:

```typescript
// ❌ Issue: SQL injection vulnerability
const query = `SELECT * FROM users WHERE id = ${userId}`;

// ✅ Fixed: Use parameterized query
const query = 'SELECT * FROM users WHERE id = $1';
const result = await db.query(query, [userId]);
```

### Step 3: Verification Steps

```markdown
## Verify Before Using

1. **Test the happy path**
   - Does it work with valid inputs?

2. **Test edge cases**
   - Empty inputs, null values, boundaries

3. **Test error cases**
   - Invalid inputs, network failures, timeouts

4. **Check against real APIs**
   - Do the function calls actually exist?
   - Are the parameters correct?
```

### Step 4: Summary Report

```markdown
## Review Summary

**File:** auth.ts
**Lines reviewed:** 150
**Issues found:** 4

### Critical (fix before using)
1. Line 45: SQL injection in user lookup
2. Line 78: Hardcoded API key

### Important (should fix)
3. Line 92: Missing error handling for API call

### Suggestions (nice to have)
4. Line 120: Could use more specific TypeScript types

### Verdict
⚠️ **Do not use as-is** - Fix critical issues first
```

## Common AI Code Mistakes

### 1. Hallucinated Methods

```typescript
// AI wrote this, but the method doesn't exist
const result = await prisma.user.findByEmail(email);

// Actual Prisma API
const result = await prisma.user.findUnique({
  where: { email }
});
```

### 2. Wrong Async Handling

```typescript
// Missing await
const users = db.query('SELECT * FROM users');
console.log(users); // Promise, not data!

// Fixed
const users = await db.query('SELECT * FROM users');
```

### 3. Security Issues

```typescript
// Dangerous: user input in HTML
element.innerHTML = userInput;

// Safe: use textContent or sanitize
element.textContent = userInput;
```

---

**Paste the code and I'll perform a thorough review.**
