# Developer Toolkit Plugin for Claude Code

**Complete developer productivity toolkit for modern software engineering.**

Master Git workflows, debug like a pro, write better code with AI assistance, and follow best practices - all in one plugin.

## Quick Install

```bash
# Install via Claude Code CLI
/plugin install latestaiagents/agent-skills/plugins/developer-toolkit

# Or using npx
npx skills add latestaiagents/agent-skills/plugins/developer-toolkit
```

> **Note**: For complete plugin documentation, see the [official Claude Code plugins guide](https://code.claude.com/docs/en/plugins).

## What's Included

### 2 Slash Commands

| Command | Description |
|---------|-------------|
| `/git-undo` | Safely undo any git mistake with guided recovery |
| `/debug-this` | Systematic debugging assistance for any error |

### 19 Professional Skills

#### Git Mastery (6 skills)

| Skill | Description |
|-------|-------------|
| **Git Undo Wizard** | Recover from any git mistake safely |
| **Merge Conflict Surgeon** | Resolve conflicts with context analysis |
| **Rebase Safely** | Interactive rebase without losing work |
| **Branch Strategy Advisor** | GitFlow, trunk-based, feature flags |
| **Commit Message Crafter** | Write clear, conventional commits |
| **Git History Detective** | Find when/why code changed |

#### Debug Detective (6 skills)

| Skill | Description |
|-------|-------------|
| **Stack Trace Decoder** | Understand and fix stack traces |
| **Log Forensics** | Extract insights from logs |
| **Error Pattern Analyzer** | Identify recurring error patterns |
| **Performance Profiler** | Find and fix performance bottlenecks |
| **Dependency Conflict Resolver** | Fix version conflicts |
| **Reproduction Builder** | Create minimal repro cases |

#### Code Intelligence (7 skills)

| Skill | Description |
|-------|-------------|
| **AI Code Reviewer** | Get thorough code reviews |
| **Refactor with AI** | Safe, incremental refactoring |
| **Debug with AI** | AI-assisted debugging |
| **Test Generation Patterns** | Generate comprehensive tests |
| **Code Explanation Generator** | Understand complex code |
| **Doc Sync Automation** | Keep docs in sync with code |
| **Codebase Context Builder** | Build context for AI assistance |

## MCP Tool Integrations

Optional connections to developer tools:

| Tool | What It Enables |
|------|-----------------|
| **GitHub** | Repository management, PRs, issues |
| **GitLab** | Repository management, CI/CD |
| **Jira** | Issue tracking, sprint management |
| **Linear** | Modern issue tracking |
| **Sentry** | Error tracking, debugging |

## Usage Examples

### Undo Git Mistake

```
You: /git-undo

I accidentally committed to main instead of my feature branch

Claude: No problem! Have you pushed yet?
[Provides safe steps to move commit to correct branch,
with different approaches based on push status]
```

### Debug an Error

```
You: /debug-this

TypeError: Cannot read property 'map' of undefined
at UserList.render (UserList.js:15)

Claude: This error means you're calling .map() on undefined...
[Analyzes stack trace, identifies likely causes,
provides fix with null check and explanation]
```

## Skills Auto-Activation

Skills activate based on context:

- Mention "git undo" or "revert commit" → Git Undo Wizard
- Ask about "merge conflict" → Merge Conflict Surgeon
- Discuss "stack trace" or "error" → Stack Trace Decoder
- Say "code review" or "review this" → AI Code Reviewer
- Mention "refactor" → Refactor with AI

## Requirements

- Claude Code CLI (latest version)
- Node.js 18+ (for npx)
- Git (for git skills)
- Optional: GitHub/GitLab account for integrations

## Security & Trust

This plugin includes MCP server configurations. Before enabling:

- **Review credentials**: Only add tokens for services you trust
- **Principle of least privilege**: Use scoped tokens
- **Audit MCP servers**: Review `.mcp.json` before copying

All MCP integrations are marked `optional: true`.

## Contributing

Want to add a developer skill?

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

See [CLAUDE.md](../../CLAUDE.md) for development guidelines.

## License

MIT License - See [LICENSE](../../LICENSE) for details.

---

**Built by [latestaiagents](https://github.com/latestaiagents)** | Part of the [Agent Skills](https://github.com/latestaiagents/agent-skills) collection
