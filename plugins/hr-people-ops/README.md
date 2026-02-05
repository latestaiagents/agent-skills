# HR & People Ops Plugin for Claude Code

**Complete HR toolkit for modern people operations.**

Streamline recruiting, onboarding, performance management, and policy creation with AI-powered assistance that ensures compliance and promotes inclusive practices.

## Quick Install

```bash
# Install via Claude Code CLI
/plugin install latestaiagents/agent-skills/plugins/hr-people-ops

# Or using npx
npx skills add latestaiagents/agent-skills/plugins/hr-people-ops
```

> **Note**: For complete plugin documentation, see the [official Claude Code plugins guide](https://code.claude.com/docs/en/plugins).

## What's Included

### 2 Slash Commands

| Command | Description |
|---------|-------------|
| `/write-job-description` | Generate inclusive job descriptions for any role |
| `/performance-review` | Create constructive reviews with specific feedback |

### 5 Professional Skills

| Skill | Category | Description |
|-------|----------|-------------|
| **Job Description Writer** | Recruiting | Inclusive, compelling JDs that attract diverse talent |
| **Interview Question Bank** | Recruiting | Structured questions with scoring rubrics |
| **Performance Review Assistant** | Performance | Constructive feedback using SBI model |
| **Compensation Analysis** | Performance | Salary bands, pay equity, total rewards |
| **Onboarding Checklist** | Onboarding | 30-60-90 day plans, new hire setup |
| **Policy Document Writer** | Policies | Compliant HR policies and handbooks |

## MCP Tool Integrations

Optional connections to HR systems:

| Tool | What It Enables |
|------|-----------------|
| **Greenhouse** | ATS integration, candidate management |
| **Lever** | Recruiting workflows |
| **Workday** | HCM, employee data |
| **BambooHR** | Employee records, time-off |
| **Slack** | Employee communications |
| **Google Calendar** | Interview scheduling |

## Usage Examples

### Write a Job Description

```
You: /write-job-description

Role: Senior Product Designer
Team: Consumer Apps
Level: Senior (5+ years)
Remote: Hybrid

Claude: Here's an inclusive job description...
[Generates complete JD with responsibilities, requirements
separated into must-have/nice-to-have, compensation section,
and bias-checked language]
```

### Prepare Interview Questions

```
You: I need to interview a engineering manager candidate

Claude: I'll help with structured interview questions...
[Provides behavioral questions for leadership, technical judgment,
team management, with scoring rubrics and red flags to watch for]
```

### Write a Performance Review

```
You: /performance-review

Employee: Alex Kim, Account Executive
Period: 2025
Accomplishments: 142% quota, largest deal in company history
Development areas: Forecasting accuracy

Claude: Here's the performance review...
[Generates specific, SBI-model feedback with accomplishments,
strengths, development areas, and SMART goals]
```

### Create Company Policy

```
You: We need a remote work policy for our startup

Claude: I'll draft a remote work policy covering...
[Creates policy with eligibility, expectations, equipment,
security requirements, and approval process]
```

## Skills Auto-Activation

Skills activate based on context:

- Mention "job description" or "JD" → Job Description Writer
- Ask about "interview questions" → Interview Question Bank
- Discuss "performance review" or "feedback" → Performance Review Assistant
- Say "compensation" or "salary bands" → Compensation Analysis
- Mention "onboarding" or "new hire" → Onboarding Checklist
- Ask about "policy" or "handbook" → Policy Document Writer

## Key Features

### Inclusive Language
All content is checked for:
- Gendered language
- Age-coded phrases
- Unnecessary requirements
- Cultural bias

### Compliance Awareness
Built-in knowledge of:
- Pay transparency laws
- Anti-discrimination requirements
- Interview legal boundaries
- Policy best practices

### Evidence-Based
Uses established frameworks:
- SBI model for feedback
- STAR method for interviews
- SMART goals for objectives

## Requirements

- Claude Code CLI (latest version)
- Node.js 18+ (for npx)
- Optional: HR system access for integrations

## Security & Trust

This plugin includes MCP server configurations for HR systems. Before enabling:

- **Review credentials**: Only connect systems you trust
- **Principle of least privilege**: Use read-only access where possible
- **Data privacy**: Be mindful of employee PII
- **Audit MCP servers**: Review `.mcp.json` before copying

All MCP integrations are marked `optional: true`.

## Contributing

Want to add an HR skill or template?

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

See [CLAUDE.md](../../CLAUDE.md) for development guidelines.

## License

MIT License - See [LICENSE](../../LICENSE) for details.

---

**Built by [latestaiagents](https://github.com/latestaiagents)** | Part of the [Agent Skills](https://github.com/latestaiagents/agent-skills) collection
