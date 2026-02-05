---
name: branch-strategy-advisor
description: |
  Use this skill when discussing Git branching strategies. Activate when the user asks about branch naming,
  GitFlow, trunk-based development, feature branches, release branches, how to organize branches,
  branch protection, or setting up a branching workflow for their team.
---

# Branch Strategy Advisor

Choose and implement the right branching strategy for your team.

## When to Use

- Setting up a new project's branching workflow
- Deciding between GitFlow, trunk-based, or GitHub Flow
- Standardizing branch naming conventions
- Configuring branch protection rules
- Scaling branching for larger teams

## Strategy Comparison

| Strategy | Best For | Release Cadence | Team Size |
|----------|----------|-----------------|-----------|
| Trunk-Based | Continuous deployment, experienced teams | Daily/hourly | Any |
| GitHub Flow | SaaS products, simple workflow | On-demand | Small-Medium |
| GitFlow | Scheduled releases, multiple versions | Weekly/Monthly | Medium-Large |

## Strategy 1: Trunk-Based Development

**Best for:** Teams with strong CI/CD, continuous deployment, feature flags.

```
main ─────●─────●─────●─────●─────●─────●───→
           \   /       \   /       \   /
feature-a  ●─●        ●─●        ●─●
(short-lived, merged within 1-2 days)
```

### Rules
- `main` is always deployable
- Feature branches live < 2 days
- Use feature flags for incomplete features
- No long-running branches

### Branch Naming
```
feature/ABC-123-short-description
fix/ABC-456-bug-description
```

### Commands
```bash
# Start feature
git checkout -b feature/ABC-123-user-auth main

# Keep updated (daily)
git fetch origin
git rebase origin/main

# Merge (same day or next)
git checkout main
git merge --no-ff feature/ABC-123-user-auth
```

---

## Strategy 2: GitHub Flow

**Best for:** SaaS products, web apps with continuous deployment.

```
main ───────●───────●───────●───────●───────→
             \     /         \     /
              ●───●           ●───●
         feature-branch    bugfix-branch
              (PR)             (PR)
```

### Rules
- `main` is always deployable
- Create branch from `main` for any change
- Open PR for discussion
- Deploy from `main` after merge

### Branch Naming
```
feature/user-authentication
bugfix/login-redirect-loop
hotfix/security-patch
docs/api-documentation
```

### Commands
```bash
# Start work
git checkout -b feature/user-authentication main

# Push and create PR
git push -u origin feature/user-authentication
gh pr create --title "Add user authentication"

# After approval
gh pr merge --squash
```

---

## Strategy 3: GitFlow

**Best for:** Products with scheduled releases, mobile apps, packages.

```
main     ────●─────────────────●─────────────●───→
              \               /               \
release        \───●───●───●─/                 \───→
                \         /
develop  ───●────●───●───●───●───●───●───●───●───→
              \     /   \     /   \         /
feature        ●───●     ●───●     ●───●───●
```

### Branches
| Branch | Purpose | Merges To |
|--------|---------|-----------|
| `main` | Production releases | - |
| `develop` | Integration branch | `release`, `main` |
| `feature/*` | New features | `develop` |
| `release/*` | Release preparation | `main`, `develop` |
| `hotfix/*` | Production fixes | `main`, `develop` |

### Branch Naming
```
feature/ABC-123-user-dashboard
release/1.2.0
hotfix/1.2.1
```

### Commands
```bash
# Start feature
git checkout -b feature/ABC-123-dashboard develop

# Finish feature
git checkout develop
git merge --no-ff feature/ABC-123-dashboard
git branch -d feature/ABC-123-dashboard

# Start release
git checkout -b release/1.2.0 develop

# Finish release
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0
git checkout develop
git merge --no-ff release/1.2.0
```

---

## Branch Naming Convention

### Format
```
<type>/<ticket>-<short-description>
```

### Types
| Type | Usage |
|------|-------|
| `feature/` | New functionality |
| `fix/` or `bugfix/` | Bug fixes |
| `hotfix/` | Urgent production fixes |
| `release/` | Release preparation |
| `docs/` | Documentation |
| `refactor/` | Code refactoring |
| `test/` | Test additions |
| `chore/` | Maintenance |

### Examples
```
feature/AUTH-123-oauth-google-login
fix/API-456-null-pointer-exception
hotfix/SEC-789-sql-injection
release/2.1.0
docs/README-setup-instructions
```

### Rules
- Use lowercase
- Use hyphens, not underscores
- Keep it short but descriptive
- Include ticket number if available

---

## Branch Protection Rules

### For `main`
```yaml
# GitHub branch protection
- Require pull request reviews (1-2 reviewers)
- Require status checks to pass
- Require branches to be up to date
- Include administrators
- Restrict who can push (CI only)
```

### For `develop` (GitFlow)
```yaml
- Require pull request reviews (1 reviewer)
- Require status checks to pass
- Allow force pushes: NO
```

### Setup Commands (GitHub CLI)
```bash
gh api repos/{owner}/{repo}/branches/main/protection -X PUT \
  -f required_pull_request_reviews='{"required_approving_review_count":1}' \
  -f required_status_checks='{"strict":true,"contexts":["ci/test"]}' \
  -F enforce_admins=true
```

---

## Decision Guide

### Choose Trunk-Based If:
- [ ] You deploy multiple times per day
- [ ] Your team has strong CI/CD practices
- [ ] You can use feature flags effectively
- [ ] Developers are experienced with Git
- [ ] Code review happens quickly (< 4 hours)

### Choose GitHub Flow If:
- [ ] You deploy on-demand (not scheduled)
- [ ] Your team is small to medium
- [ ] You want simplicity over structure
- [ ] PRs are your primary review mechanism
- [ ] You don't need to maintain multiple versions

### Choose GitFlow If:
- [ ] You have scheduled releases (sprints)
- [ ] You maintain multiple production versions
- [ ] You need strict release control
- [ ] Your team is medium to large
- [ ] You have dedicated release managers

---

## Migration Between Strategies

### From GitFlow to Trunk-Based
1. Stop creating feature branches from `develop`
2. Create features from `main` instead
3. Implement feature flags
4. Shorten branch lifespan to < 2 days
5. Eventually deprecate `develop`

### From Nothing to GitHub Flow
1. Protect `main` branch
2. Require PRs for all changes
3. Set up CI to run on PRs
4. Establish naming conventions
5. Document in CONTRIBUTING.md
