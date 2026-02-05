# QA & Testing Plugin for Claude Code

**Complete quality assurance toolkit for modern software testing.**

From test planning to automation, API testing to performance - everything you need to ensure software quality and ship with confidence.

## Quick Install

```bash
# Install via Claude Code CLI
/plugin install latestaiagents/agent-skills/plugins/qa-testing

# Or using npx
npx skills add latestaiagents/agent-skills/plugins/qa-testing
```

> **Note**: For complete plugin documentation, see the [official Claude Code plugins guide](https://code.claude.com/docs/en/plugins).

## What's Included

### 3 Slash Commands

| Command | Description |
|---------|-------------|
| `/write-test-cases` | Generate comprehensive test cases from requirements |
| `/test-plan` | Create test strategy and planning documents |
| `/bug-report` | Structure clear, actionable bug reports |

### 6 Professional Skills

| Skill | Category | Description |
|-------|----------|-------------|
| **Test Planning** | Strategy | Create test strategies, coverage matrices, schedules |
| **Playwright Patterns** | Automation | E2E testing with Page Object Model, fixtures |
| **API Test Patterns** | API Testing | REST/GraphQL testing, contract testing |
| **Performance Testing** | Performance | k6, Artillery load and stress testing |
| **Test Case Design** | Test Writing | Techniques: BVA, equivalence, decision tables |
| **Bug Reporting** | Test Writing | Clear, actionable defect documentation |

## MCP Tool Integrations

Optional connections to testing tools:

| Tool | What It Enables |
|------|-----------------|
| **TestRail** | Test case management, reporting |
| **Xray** | Jira test management integration |
| **Jira** | Bug tracking, test linking |
| **GitHub** | CI/CD test integration |
| **BrowserStack** | Cross-browser testing |
| **Datadog** | Test monitoring, synthetics |

## Usage Examples

### Generate Test Cases

```
You: /write-test-cases

Feature: Password reset flow
Format: BDD/Gherkin

Claude: Here are comprehensive test scenarios...
[Generates Gherkin scenarios for happy path,
invalid email, expired tokens, rate limiting,
account lockout, and edge cases]
```

### Create Test Plan

```
You: /test-plan

Project: Mobile app v2.0 release
Timeline: 3 weeks
Risks: New payment provider, iOS 18 compatibility

Claude: Here's your test plan...
[Generates complete plan with scope, approach,
schedule, risk matrix, and exit criteria]
```

### Write Bug Report

```
You: /bug-report

Checkout crashes when applying promo code.
Shows spinner forever, then page goes blank.

Claude: Let me structure that bug report...
[Creates detailed report with reproduction steps,
environment checklist, severity assessment,
and evidence collection guidance]
```

### E2E Test Help

```
You: Help me write a Playwright test for user login

Claude: I'll help with a Page Object Model approach...
[Provides LoginPage class, test structure,
locator best practices, and CI integration]
```

## Skills Auto-Activation

Skills activate based on context:

- Mention "test plan" or "test strategy" → Test Planning
- Ask about "Playwright" or "E2E test" → Playwright Patterns
- Discuss "API test" or "endpoint testing" → API Test Patterns
- Say "load test" or "k6" → Performance Testing
- Mention "test cases" or "BDD" → Test Case Design
- Ask about "bug report" or "defect" → Bug Reporting

## Test Coverage

```
Testing Pyramid Covered:

        /\
       /E2E\          ← Playwright, Cypress
      /─────\
     /Integr-\        ← API Testing, Contract
    /──ation──\
   /───────────\
  /    Unit     \     ← (Development scope)
 /───────────────\
```

## Requirements

- Claude Code CLI (latest version)
- Node.js 18+ (for npx and test tools)
- Optional: Playwright, k6, Artillery for automation
- Optional: TestRail/Xray account for test management

## Security & Trust

This plugin includes MCP server configurations. Before enabling:

- **Review credentials**: Only connect tools you trust
- **Test environments**: Avoid running load tests on production
- **Data privacy**: Use synthetic test data, not real user data
- **Audit MCP servers**: Review `.mcp.json` before copying

All MCP integrations are marked `optional: true`.

## Contributing

Want to add a testing pattern or tool integration?

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

See [CLAUDE.md](../../CLAUDE.md) for development guidelines.

## License

MIT License - See [LICENSE](../../LICENSE) for details.

---

**Built by [latestaiagents](https://github.com/latestaiagents)** | Part of the [Agent Skills](https://github.com/latestaiagents/agent-skills) collection
