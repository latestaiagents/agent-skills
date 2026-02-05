---
name: commit-message-crafter
description: |
  Use this skill when writing Git commit messages. Activate when the user asks about commit message format,
  conventional commits, how to write good commit messages, commit message best practices, or when they're
  about to commit changes and need help crafting the message.
---

# Commit Message Crafter

Write clear, consistent, and meaningful commit messages.

## When to Use

- Writing a commit message for staged changes
- Following conventional commits format
- Standardizing commit messages across a team
- Writing messages for squash commits or PRs

## The Conventional Commits Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

| Type | When to Use |
|------|-------------|
| `feat` | New feature for the user |
| `fix` | Bug fix for the user |
| `docs` | Documentation only changes |
| `style` | Formatting, missing semicolons, etc. (no code change) |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `chore` | Maintenance tasks, dependencies, build changes |
| `ci` | CI/CD configuration changes |
| `revert` | Reverting a previous commit |

### Scope (Optional)

The part of the codebase affected:
- `auth`, `api`, `ui`, `db`, `config`, `deps`, etc.

### Subject Rules

1. Use imperative mood: "Add feature" not "Added feature"
2. Don't capitalize first letter
3. No period at the end
4. Max 50 characters (72 absolute max)
5. Complete the sentence: "This commit will..."

## Examples

### Simple Feature
```
feat(auth): add password reset functionality
```

### Bug Fix with Body
```
fix(api): handle null response from payment gateway

The payment gateway occasionally returns null instead of an error
object when the service is degraded. This caused uncaught exceptions
in the checkout flow.

Fixes #1234
```

### Breaking Change
```
feat(api)!: change user endpoint response format

BREAKING CHANGE: The /api/users endpoint now returns paginated
results. Clients must update to handle the new response structure:
{ data: [], meta: { page, total } }
```

### Refactoring
```
refactor(utils): extract date formatting into separate module

Moved date formatting logic from multiple components into a
centralized dateUtils module for better maintainability.
```

### Multiple Changes (Squash Commit)
```
feat(dashboard): redesign analytics widgets

- Add new chart component for user growth
- Implement date range picker
- Add export to CSV functionality
- Update color scheme to match brand guidelines

Co-authored-by: Jane Doe <jane@example.com>
```

## Workflow

### Step 1: Analyze Changes

```bash
# See what's staged
git diff --staged

# See files changed
git diff --staged --name-only
```

### Step 2: Determine Type

Ask: What is the PRIMARY purpose of these changes?
- Adding something new? → `feat`
- Fixing broken behavior? → `fix`
- Improving without changing behavior? → `refactor`

### Step 3: Identify Scope

Look at the files changed. What's the common area?
- All in `/src/auth/`? → scope is `auth`
- Mixed areas? → omit scope or use broader term

### Step 4: Write Subject

Complete this sentence with your changes:
"If applied, this commit will _______________"

Good: "add user avatar upload"
Bad: "added user avatar upload functionality to the system"

### Step 5: Add Body (If Needed)

Include body when:
- The "why" isn't obvious from the subject
- There are multiple related changes
- Breaking changes need explanation
- Linking to issues/tickets

### Step 6: Add Footer (If Needed)

```
Fixes #123
Closes #456
BREAKING CHANGE: description
Co-authored-by: Name <email>
```

## Quick Templates

### Feature
```
feat(<scope>): <what you added>

<why you added it, if not obvious>

Relates to #<issue>
```

### Bug Fix
```
fix(<scope>): <what you fixed>

<what was broken and why>

Fixes #<issue>
```

### Dependency Update
```
chore(deps): update <package> to <version>

<any breaking changes or notes>
```

## Common Mistakes to Avoid

- **Vague messages**: "fix bug", "update code", "changes"
- **Too long**: Keep subject under 50 chars
- **Wrong tense**: "Added" instead of "Add"
- **Multiple unrelated changes**: Split into separate commits
- **No context**: Just "fix" without explaining what or why

## Team Conventions

Consider documenting your team's conventions:

```markdown
## Commit Message Convention

- Use conventional commits format
- Scope is required for features and fixes
- Reference issue numbers in footer
- Breaking changes must include BREAKING CHANGE footer
```

## Tools

```bash
# Set up commit template
git config --global commit.template ~/.gitmessage

# Use commitizen for interactive commits
npx cz

# Validate commits with commitlint
npm install -D @commitlint/{cli,config-conventional}
```
