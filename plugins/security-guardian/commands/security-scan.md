---
description: Run comprehensive security scan on codebase for OWASP vulnerabilities
---

# /security-scan

Scan your codebase for common security vulnerabilities.

## What I Need

Tell me:
- What language/framework is your project?
- Any specific concerns (auth, API, data handling)?
- Scope: full scan or specific files?

## Scan Coverage

### OWASP Top 10 Checks

1. **A01 - Injection** - SQL, NoSQL, Command injection
2. **A02 - Broken Auth** - Session management, passwords
3. **A03 - Sensitive Data** - Encryption, data exposure
4. **A04 - XXE** - XML processing vulnerabilities
5. **A05 - Access Control** - Authorization flaws
6. **A06 - Misconfig** - Security settings, defaults
7. **A07 - XSS** - Cross-site scripting
8. **A08 - Deserialization** - Unsafe object handling
9. **A09 - Vulnerable Components** - Dependencies
10. **A10 - Logging** - Insufficient monitoring

## Scan Process

### Step 1: Static Analysis
I'll search for vulnerable patterns in your code:
- String concatenation in queries
- Eval/exec with user input
- Hardcoded secrets
- Missing input validation

### Step 2: Dependency Check
I'll review your dependencies:
- Known CVEs
- Outdated packages
- Unused dependencies

### Step 3: Configuration Review
I'll check security settings:
- CORS configuration
- Security headers
- Cookie settings
- Environment variables

## Output

I'll provide:
- Severity-ranked findings
- Specific file:line locations
- Fix recommendations with code examples
- Links to relevant documentation

## Quick Commands

```bash
# Run with Semgrep
semgrep --config=p/owasp-top-ten .

# Check dependencies (npm)
npm audit

# Check dependencies (Python)
pip-audit

# Scan secrets
gitleaks detect --source .
```
