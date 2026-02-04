---
name: secure-code-review
description: |
  Systematic security code review methodology. Use this skill when reviewing pull
  requests for security issues, auditing critical code paths, or performing security
  assessments. Activate when: security review, code audit, secure code, review PR
  for security, find vulnerabilities, security assessment.
---

# Secure Code Review

**A systematic approach to finding security vulnerabilities in code.**

## When to Use

- Reviewing pull requests
- Auditing security-critical code
- Before production deployments
- Compliance requirements
- After security incidents

## Review Methodology

### 1. Understand Context

```markdown
## Pre-Review Questions

- [ ] What does this code do?
- [ ] What data does it handle? (PII, financial, auth)
- [ ] Who can access this functionality?
- [ ] What are the trust boundaries?
- [ ] What could go wrong?
```

### 2. Security Review Checklist

```markdown
## Input Handling
- [ ] All user input validated
- [ ] Input length limits enforced
- [ ] Type checking performed
- [ ] Whitelisting over blacklisting

## Authentication
- [ ] Authentication required where needed
- [ ] Passwords hashed properly (bcrypt/argon2)
- [ ] Session management secure
- [ ] MFA considered for sensitive actions

## Authorization
- [ ] Authorization checks on all endpoints
- [ ] Resource ownership verified
- [ ] No privilege escalation paths
- [ ] Default deny policy

## Data Protection
- [ ] Sensitive data encrypted at rest
- [ ] TLS for data in transit
- [ ] No sensitive data in logs
- [ ] Proper data masking

## Injection Prevention
- [ ] Parameterized queries used
- [ ] No eval() with user data
- [ ] Command injection prevented
- [ ] XSS prevention (encoding/CSP)

## Error Handling
- [ ] No stack traces to users
- [ ] No sensitive data in errors
- [ ] Proper logging of security events

## Dependencies
- [ ] No known vulnerabilities
- [ ] Packages from trusted sources
- [ ] Lock files up to date
```

### 3. High-Risk Areas

Focus extra attention on:

```javascript
// File uploads
app.post('/upload', (req, res) => {
  // Check: file type validation, size limits, storage location
});

// Authentication
app.post('/login', (req, res) => {
  // Check: rate limiting, timing attacks, error messages
});

// Authorization
app.get('/admin/*', (req, res) => {
  // Check: role verification, access control
});

// Data queries
db.query(sql, params);
// Check: parameterized queries, access control

// External API calls
fetch(url);
// Check: SSRF prevention, URL validation

// Serialization
JSON.parse(input);
pickle.loads(input);
// Check: deserialization safety

// Crypto operations
crypto.createCipher();
// Check: algorithm strength, key management
```

## Vulnerability Patterns by Language

### JavaScript/TypeScript

```javascript
// VULNERABLE: Prototype pollution
Object.assign(target, userInput);
target[userKey] = userValue;

// SAFE: Use Map or validate keys
const safeObj = Object.create(null);
if (!['__proto__', 'constructor'].includes(key)) {
  safeObj[key] = value;
}

// VULNERABLE: ReDoS
const regex = /^(a+)+$/;  // Catastrophic backtracking

// SAFE: Use bounded quantifiers
const regex = /^a{1,100}$/;

// VULNERABLE: Path traversal
const file = path.join(uploadDir, userFilename);

// SAFE: Validate and normalize
const safeName = path.basename(userFilename);
const file = path.join(uploadDir, safeName);
```

### Python

```python
# VULNERABLE: Format string injection
query = "SELECT * FROM users WHERE id = %s" % user_id
eval(f"config['{user_input}']")

# SAFE: Parameterized queries
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# VULNERABLE: Arbitrary file read
with open(user_path) as f:
    return f.read()

# SAFE: Validate path
if not user_path.startswith(ALLOWED_DIR):
    raise ValueError("Invalid path")
```

### Java

```java
// VULNERABLE: XML External Entities
DocumentBuilder db = DocumentBuilderFactory.newInstance().newDocumentBuilder();
Document doc = db.parse(userInput);

// SAFE: Disable XXE
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);

// VULNERABLE: Unsafe reflection
Class.forName(userInput).newInstance();

// SAFE: Whitelist allowed classes
if (ALLOWED_CLASSES.contains(className)) {
    Class.forName(className).newInstance();
}
```

## Review Comments Template

```markdown
### 🔴 Critical Security Issue

**Location:** `src/auth/login.js:45`
**Issue:** SQL injection vulnerability
**Impact:** Database compromise, data exfiltration
**Fix:**
```javascript
// Before (vulnerable)
const query = `SELECT * FROM users WHERE email = '${email}'`;

// After (safe)
const query = 'SELECT * FROM users WHERE email = $1';
const result = await db.query(query, [email]);
```

---

### 🟡 Security Concern

**Location:** `src/api/users.js:23`
**Issue:** Missing rate limiting on authentication endpoint
**Risk:** Brute force attacks
**Recommendation:** Add rate limiting (see example in `/middleware/rateLimit.js`)

---

### 🟢 Security Suggestion

**Location:** `src/utils/crypto.js:12`
**Suggestion:** Consider using Argon2 instead of bcrypt for password hashing
**Reason:** Better resistance to GPU attacks
```

## Automated Security Scanning

```bash
# Static Analysis (SAST)
# JavaScript
npx eslint --plugin security .
npx njsscan .

# Python
pip install bandit
bandit -r .

# Java
# Use SpotBugs with FindSecBugs plugin

# Multi-language
# Semgrep
semgrep --config auto .

# CodeQL (GitHub)
# Configure in .github/workflows/codeql.yml
```

## Security Review Report Template

```markdown
# Security Review Report

**Project:** [Name]
**Reviewer:** [Name]
**Date:** [Date]
**Scope:** [Files/Features reviewed]

## Executive Summary
[1-2 paragraph summary of findings]

## Risk Rating
- Critical: X
- High: X
- Medium: X
- Low: X

## Findings

### Critical Findings
1. [Finding title]
   - Location: [file:line]
   - Description: [Details]
   - Impact: [What could happen]
   - Remediation: [How to fix]

### High Findings
[...]

## Recommendations
1. [Priority recommendation]
2. [...]

## Appendix
- Tools used
- Time spent
- Out of scope items
```

## Best Practices

1. **Review in Layers**: Input → Processing → Output
2. **Think Like an Attacker**: How would you exploit this?
3. **Follow Data Flow**: Track untrusted data through the code
4. **Check Trust Boundaries**: Where does trust change?
5. **Use Checklists**: Don't rely on memory
6. **Automate What You Can**: SAST tools catch low-hanging fruit
7. **Document Findings**: Clear, actionable reports
8. **Verify Fixes**: Re-review remediated issues
