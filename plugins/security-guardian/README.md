# Security Guardian Plugin for Claude Code

**Complete application security toolkit covering OWASP Top 10 and beyond.**

Protect your applications from common vulnerabilities with expert guidance on secure coding, vulnerability detection, and security best practices.

## Quick Install

```bash
# Install via Claude Code CLI
/plugin install latestaiagents/agent-skills/plugins/security-guardian

# Or using npx
npx skills add latestaiagents/agent-skills/plugins/security-guardian
```

> **Note**: For complete plugin documentation, see the [official Claude Code plugins guide](https://code.claude.com/docs/en/plugins).

## What's Included

### 2 Slash Commands

| Command | Description |
|---------|-------------|
| `/security-scan` | Run comprehensive OWASP vulnerability scan on codebase |
| `/fix-vulnerability` | Step-by-step guidance to fix specific security issues |

### 10 Security Skills

#### OWASP Top 10 (5 skills)

| Skill | OWASP ID | Description |
|-------|----------|-------------|
| **Injection Prevention** | A01 | SQL, NoSQL, Command injection defense |
| **XSS Prevention** | A07 | Cross-site scripting protection |
| **Broken Auth Detector** | A02 | Authentication vulnerability detection |
| **Access Control Audit** | A05 | Authorization and access control |
| **Security Misconfiguration** | A06 | Configuration security checks |

#### Common Security (5 skills)

| Skill | Description |
|-------|-------------|
| **Secrets Detection** | Find hardcoded credentials and API keys |
| **JWT Security** | Secure JSON Web Token implementation |
| **API Security** | REST/GraphQL API protection patterns |
| **Secure Code Review** | Security-focused code review checklist |
| **Dependency Vulnerability** | Third-party component security |

## MCP Tool Integrations

Optional connections to security tools:

| Tool | What It Enables |
|------|-----------------|
| **Snyk** | Dependency vulnerability scanning |
| **GitHub Security** | Dependabot, code scanning alerts |
| **Semgrep** | Static analysis with security rules |
| **Trivy** | Container and IaC security |
| **HashiCorp Vault** | Secrets management |

## Usage Examples

### Security Scan

```
You: /security-scan

Claude: I'll scan your codebase for security vulnerabilities. What language/framework?

You: Node.js Express app

Claude: Running OWASP Top 10 analysis...
[Identifies SQL injection in user controller,
missing CSRF protection, hardcoded API key,
provides severity-ranked findings with fixes]
```

### Fix Specific Vulnerability

```
You: /fix-vulnerability

This code has XSS:
res.send(`<h1>Hello ${req.query.name}</h1>`)

Claude: This is a reflected XSS vulnerability...
[Explains attack vector, provides escaped version,
shows CSP headers to add, suggests testing approach]
```

## Skills Auto-Activation

Skills activate based on context:

- Mention "SQL injection" or "parameterized query" → Injection Prevention
- Ask about "XSS" or "script injection" → XSS Prevention
- Discuss "JWT" or "token security" → JWT Security
- Say "secrets" or "API key exposed" → Secrets Detection
- Mention "code review" for security → Secure Code Review

## OWASP Coverage

```
OWASP Top 10 (2021)
├── A01: Broken Access Control      ✓ access-control-audit
├── A02: Cryptographic Failures     ✓ (covered in multiple skills)
├── A03: Injection                  ✓ injection-prevention
├── A04: Insecure Design            ✓ secure-code-review
├── A05: Security Misconfiguration  ✓ security-misconfiguration
├── A06: Vulnerable Components      ✓ dependency-vulnerability
├── A07: Auth Failures              ✓ broken-auth-detector
├── A08: Data Integrity Failures    ✓ jwt-security
├── A09: Logging Failures           ✓ (in security-misconfiguration)
└── A10: SSRF                       ✓ (in injection-prevention)
```

## Requirements

- Claude Code CLI (latest version)
- Node.js 18+ (for npx)
- Optional: Security scanning tools (Snyk, Semgrep, etc.)

## Security & Trust

This plugin includes MCP server configurations for security tools. Before enabling:

- **Review credentials**: Only add tokens for tools you trust
- **Principle of least privilege**: Use read-only API keys
- **Audit MCP servers**: Review `.mcp.json` before copying

All MCP integrations are marked `optional: true`.

## Contributing

Found a security pattern we should cover?

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

See [CLAUDE.md](../../CLAUDE.md) for development guidelines.

## License

MIT License - See [LICENSE](../../LICENSE) for details.

---

**Built by [latestaiagents](https://github.com/latestaiagents)** | Part of the [Agent Skills](https://github.com/latestaiagents/agent-skills) collection
