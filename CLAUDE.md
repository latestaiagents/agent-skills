# Agent Skills - Development Guide

This is the official skills repository for latestaiagents. It contains 55 professional skills and 10 slash commands for AI coding agents.

## Project Structure

```
agent-skills/
├── skills/                      # All skills organized by audience
│   ├── safety/                  # Essential safety skills (5 skills)
│   │   └── ai-safety-guardrails/
│   │       ├── destructive-operation-guard/
│   │       │   └── SKILL.md
│   │       ├── migration-safety/
│   │       ├── database-safety/
│   │       ├── file-operation-safety/
│   │       └── git-safety/
│   ├── developer/               # Developer-focused skills (19 skills)
│   │   ├── git-mastery/         # Git workflows (6 skills)
│   │   ├── code-intelligence/   # AI coding (7 skills)
│   │   └── debug-detective/     # Debugging (6 skills)
│   ├── mlops/                   # For architects/MLOps engineers (15 skills)
│   │   ├── agent-architect/     # Multi-agent systems (8 skills)
│   │   └── llmops-guardian/     # LLM operations (7 skills)
│   └── security/                # Security skills (16 skills)
│       ├── owasp-guardian/      # OWASP Top 10 (10 skills)
│       └── common-security/     # Common security (6 skills)
├── commands/                    # Slash commands (Claude Code only)
│   ├── git-undo.md
│   ├── resolve-conflict.md
│   └── ... (10 total)
├── .claude-plugin/              # Plugin configuration
│   ├── plugin.json              # Main plugin metadata
│   └── marketplace.json         # Marketplace entries
├── CLAUDE.md                    # This file
├── README.md                    # User documentation
└── LICENSE                      # MIT License
```

## Creating a New Skill

### Step 1: Choose the Right Category

| Category | Path | Audience | Examples |
|----------|------|----------|----------|
| `safety` | `skills/safety/` | Everyone | Safety, guardrails, protection |
| `developer` | `skills/developer/` | Software developers | Git, debugging, testing |
| `mlops` | `skills/mlops/` | Architects, DevOps | Multi-agent, LLMOps |
| `security` | `skills/security/` | Security engineers | OWASP, XSS, JWT, API security |

### Step 2: Create the Directory Structure

```bash
# For a new skill in an existing suite
mkdir -p skills/developer/git-mastery/new-skill-name

# For a new suite
mkdir -p skills/developer/new-suite-name/first-skill-name
```

### Step 3: Create SKILL.md

Every skill MUST have a `SKILL.md` file with this exact format:

```markdown
---
name: skill-name-in-kebab-case
description: |
  Brief description of what this skill does. Include activation keywords.
  Use this skill when [specific trigger conditions].
  Activate when user mentions [keywords that should trigger this skill].
---

# Skill Title

**Brief one-line summary of what this skill does.**

## When to Use

- Bullet point 1
- Bullet point 2
- Bullet point 3

## Workflow

### Step 1: First Step
Description and code examples...

### Step 2: Second Step
Description and code examples...

## Best Practices

1. Best practice 1
2. Best practice 2
3. Best practice 3
```

### Step 4: Skill Naming Conventions

- **Skill name**: `kebab-case` (e.g., `merge-conflict-surgeon`)
- **Directory name**: Same as skill name
- **File name**: Always `SKILL.md` (uppercase)

### Step 5: Description Field Best Practices

The `description` field in the YAML frontmatter is critical for auto-activation. Include:

1. **What the skill does** (first sentence)
2. **When to activate** (trigger conditions)
3. **Keywords** that should trigger this skill

Example:
```yaml
description: |
  Systematic merge conflict resolution with context analysis.
  Use this skill when user has merge conflicts, git conflicts, or needs to resolve conflicting changes.
  Activate when: merge conflict, conflict resolution, git merge failed, resolve conflicts.
```

## Creating a New Slash Command

### Step 1: Create the File

```bash
touch commands/new-command.md
```

### Step 2: Use This Format

```markdown
---
description: Brief description of what this command does
---

# /command-name

One-line summary of the command.

## What I Need

Tell me:
- Required input 1
- Required input 2

## Workflow

### Step 1: First Step
...

### Step 2: Second Step
...

## Quick Reference

| Option | Description |
|--------|-------------|
| Option A | What it does |
| Option B | What it does |
```

### Step 3: Naming Conventions

- **File name**: `kebab-case.md` (e.g., `git-undo.md`)
- **Command name**: Same as file name without `.md` (e.g., `/git-undo`)

## Updating marketplace.json

When adding a new suite, add an entry to `.claude-plugin/marketplace.json`:

```json
{
  "name": "suite-name",
  "source": "./skills/category/suite-name",
  "description": "Brief description of the suite.",
  "version": "1.0.0",
  "author": {
    "name": "latestaiagents"
  },
  "homepage": "https://github.com/latestaiagents/agent-skills",
  "repository": "https://github.com/latestaiagents/agent-skills",
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "category": "Category Name"
}
```

## Content Guidelines

### DO

- Use clear, actionable language
- Include real code examples (TypeScript, bash, SQL)
- Provide step-by-step workflows
- Add safety warnings for destructive operations
- Include "Best Practices" section
- Use tables for quick reference

### DON'T

- Use jargon without explanation
- Skip code examples
- Assume user knowledge
- Forget safety considerations
- Write walls of text without structure

## Safety Skills Requirements

Skills in `skills/safety/ai-safety-guardrails/` MUST:

1. Use "Danger Levels" instead of "When to Use"
2. Include explicit confirmation protocols
3. Provide recovery procedures
4. Show exact impact before operations
5. Never proceed without user confirmation

## Testing Locally

```bash
# Install all skills locally
npx skills add ./ --all

# Install specific category
npx skills add ./skills/developer --all

# Install specific suite
npx skills add ./skills/developer/git-mastery --all
```

## File Checklist

Before committing a new skill:

- [ ] `SKILL.md` exists with correct format
- [ ] YAML frontmatter has `name` and `description`
- [ ] Description includes activation keywords
- [ ] Content has clear sections and examples
- [ ] Code examples are correct and tested
- [ ] Safety considerations addressed (if applicable)
- [ ] `marketplace.json` updated (if new suite)
- [ ] README.md updated (if new suite or category)

## Repository URLs

- **GitHub**: https://github.com/latestaiagents/agent-skills
- **Issues**: https://github.com/latestaiagents/agent-skills/issues
- **Homepage**: https://github.com/latestaiagents/agent-skills

## Version

Current version: 1.0.0
