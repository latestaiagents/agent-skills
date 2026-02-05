---
name: secrets-detection
description: |
  Find and prevent leaked secrets, API keys, and credentials in code. Use this skill
  when reviewing code for exposed secrets, setting up pre-commit hooks, or auditing
  repositories. Activate when: leaked secret, API key exposed, credentials in code,
  hardcoded password, secret scanning, git secrets, pre-commit hook.
---

# Secrets Detection

**Find and prevent leaked API keys, passwords, and credentials in your codebase.**

## When to Use

- Reviewing code for hardcoded secrets
- Setting up CI/CD security checks
- Auditing git history for leaks
- Configuring pre-commit hooks
- Responding to secret exposure incidents

## Common Secret Patterns

| Secret Type | Pattern Example |
|------------|-----------------|
| AWS Access Key | `AKIA[0-9A-Z]{16}` |
| AWS Secret Key | 40-character base64 |
| GitHub Token | `ghp_[a-zA-Z0-9]{36}` |
| Stripe API Key | `sk_live_[a-zA-Z0-9]{24}` |
| Private Key | `-----BEGIN RSA PRIVATE KEY-----` |
| JWT Secret | High entropy string |

## Detection Tools

### 1. Gitleaks (Recommended)

```bash
# Install
brew install gitleaks

# Scan current directory
gitleaks detect -v

# Scan git history
gitleaks detect --source . -v

# CI/CD integration
gitleaks detect --source . --exit-code 1
```

```yaml
# .gitleaks.toml - Custom rules
[allowlist]
  paths = [
    '''vendor/''',
    '''node_modules/''',
    '''\.test\.'''
  ]

[[rules]]
  description = "Custom API Key"
  id = "custom-api-key"
  regex = '''myapp_[a-zA-Z0-9]{32}'''
  tags = ["key", "custom"]
```

### 2. git-secrets (AWS)

```bash
# Install
brew install git-secrets

# Add AWS patterns
git secrets --register-aws

# Scan repository
git secrets --scan

# Install hooks
git secrets --install
```

### 3. TruffleHog

```bash
# Scan repository
trufflehog git file://. --only-verified

# Scan GitHub org
trufflehog github --org=myorg --only-verified

# CI/CD
trufflehog git file://. --fail --only-verified
```

## Pre-Commit Hooks

### Husky + lint-staged

```bash
npm install -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,ts,jsx,tsx}": [
      "gitleaks detect --no-git -v"
    ]
  }
}
```

```bash
# .husky/pre-commit
#!/bin/sh
npx lint-staged
gitleaks protect --staged -v
```

### Pre-commit Framework

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/awslabs/git-secrets
    rev: master
    hooks:
      - id: git-secrets

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

```bash
# Install
pip install pre-commit
pre-commit install
```

## CI/CD Integration

### GitHub Actions

```yaml
name: Secret Scanning

on: [push, pull_request]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  trufflehog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified
```

### GitLab CI

```yaml
secret_detection:
  stage: test
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --source . --exit-code 1 -v
  allow_failure: false
```

## Safe Secrets Management

### Environment Variables

```javascript
// Load from environment
const config = {
  apiKey: process.env.API_KEY,
  dbPassword: process.env.DB_PASSWORD,
  jwtSecret: process.env.JWT_SECRET
};

// Validate required secrets
const requiredSecrets = ['API_KEY', 'DB_PASSWORD', 'JWT_SECRET'];
for (const secret of requiredSecrets) {
  if (!process.env[secret]) {
    throw new Error(`Missing required secret: ${secret}`);
  }
}
```

### .env Files (Development Only)

```bash
# .env.example (commit this)
API_KEY=your_api_key_here
DB_PASSWORD=your_db_password_here

# .env (NEVER commit)
API_KEY=sk_live_actual_key_12345
DB_PASSWORD=actual_password
```

```gitignore
# .gitignore
.env
.env.local
.env.*.local
*.pem
*.key
```

### Secrets Manager Integration

```javascript
// AWS Secrets Manager
const { SecretsManager } = require('@aws-sdk/client-secrets-manager');

async function getSecret(secretName) {
  const client = new SecretsManager({ region: 'us-east-1' });
  const response = await client.getSecretValue({ SecretId: secretName });
  return JSON.parse(response.SecretString);
}

// HashiCorp Vault
const vault = require('node-vault')({
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN
});

async function getVaultSecret(path) {
  const { data } = await vault.read(path);
  return data.data;
}
```

## Incident Response: Secret Leaked

```bash
# 1. Immediately revoke the secret
# - AWS: IAM console -> Delete access key
# - GitHub: Settings -> Developer settings -> Delete token
# - Stripe: Dashboard -> API keys -> Roll key

# 2. Remove from git history
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/secret/file" \
  --prune-empty --tag-name-filter cat -- --all

# Or use BFG Repo-Cleaner (faster)
bfg --delete-files secret-file.txt
bfg --replace-text secrets.txt

# 3. Force push (coordinate with team!)
git push origin --force --all
git push origin --force --tags

# 4. Audit for unauthorized access
# Check service logs for the compromised credential

# 5. Generate new secret and update references
```

## Code Review Checklist

- [ ] No hardcoded API keys or passwords
- [ ] No private keys in repository
- [ ] No credentials in configuration files
- [ ] Environment variables used for secrets
- [ ] .env files in .gitignore
- [ ] Pre-commit hooks configured
- [ ] CI/CD secret scanning enabled
- [ ] Secrets manager used in production
- [ ] Example config files don't contain real secrets

## Best Practices

1. **Never Commit Secrets**: Use pre-commit hooks
2. **Use Secrets Managers**: Vault, AWS Secrets Manager
3. **Rotate Regularly**: Automate secret rotation
4. **Audit Access**: Log who accesses secrets
5. **Least Privilege**: Minimal permissions for each secret
6. **Scan Git History**: Secrets may be in old commits
7. **Act Fast on Leaks**: Revoke immediately, then clean up
