# DevOps/SRE Plugin for Claude Code

**The most comprehensive DevOps and Site Reliability Engineering toolkit for AI coding agents.**

Complete incident response, observability, reliability engineering, on-call management, and deployment automation - all integrated with your existing tools.

## Quick Install

```bash
# Install from GitHub
npx skills add latestaiagents/agent-skills/plugins/devops-sre

# Or clone and install locally
git clone https://github.com/latestaiagents/agent-skills.git
npx skills add ./agent-skills/plugins/devops-sre
```

## What's Included

### 6 Slash Commands

| Command | Description |
|---------|-------------|
| `/incident-response` | Start structured incident response with severity assessment |
| `/postmortem` | Generate blameless postmortems with 5 Whys analysis |
| `/runbook-execute` | Execute runbooks step-by-step with safety checks |
| `/on-call-handoff` | Create comprehensive on-call handoff reports |
| `/deploy-checklist` | Pre/post deployment verification checklist |
| `/slo-report` | Generate SLO status and error budget reports |

### 8 Professional Skills

#### Incident Response
- **Incident Commander** - Lead incidents with clear severity levels, communication templates, and resolution workflows
- **Root Cause Analysis** - Systematic debugging with 5 Whys, fault trees, and contributing factors

#### Observability
- **Metrics, Logs, Traces** - RED/USE methods, structured logging, OpenTelemetry integration
- **Alerting Strategies** - Symptom-based alerts, SLO burn rates, alert fatigue prevention

#### Reliability
- **SLO/SLI/Error Budgets** - Define and measure reliability with actionable error budget policies

#### On-Call
- **On-Call Best Practices** - Sustainable rotations, fair scheduling, handoff protocols

#### Automation
- **Kubernetes Troubleshooting** - Debug pods, services, deployments with systematic approaches
- **Deployment Strategies** - Rolling, blue-green, canary, and feature flag patterns

## MCP Tool Integrations

This plugin includes optional MCP (Model Context Protocol) connections for:

| Tool | What It Enables |
|------|-----------------|
| **Datadog** | Query metrics, check monitors, view dashboards |
| **PagerDuty** | Manage incidents, acknowledge alerts, view schedules |
| **AWS CloudWatch** | Query logs, metrics, and alarms |
| **Kubernetes** | Execute kubectl commands, manage clusters |
| **GitHub** | Create issues, manage PRs, view workflows |
| **Slack** | Post incident updates, notify channels |
| **Prometheus** | Query metrics directly |
| **Terraform** | Plan and apply infrastructure changes |

### Setting Up MCP Tools

1. Copy the MCP configuration:
```bash
cp .mcp.json ~/.claude/.mcp.json
```

2. Add your credentials to each tool you want to use:
```json
{
  "datadog": {
    "command": "npx",
    "args": ["-y", "@anthropic/mcp-datadog"],
    "env": {
      "DD_API_KEY": "your-api-key",
      "DD_APP_KEY": "your-app-key"
    }
  }
}
```

3. Restart Claude Code to load the tools.

## Usage Examples

### During an Incident

```
You: /incident-response

Claude: I'll help you manage this incident. What's happening?

You: API is returning 500 errors, error rate at 15%

Claude: Based on the symptoms, I'm assessing this as SEV2...
[Provides structured response with communication template,
investigation steps, and resolution guidance]
```

### Creating a Postmortem

```
You: /postmortem INC-456

Claude: I'll help create a blameless postmortem for INC-456.
[Generates timeline, 5 Whys analysis, contributing factors,
and action items with owners]
```

### Deploying Safely

```
You: /deploy-checklist api-service v2.3.4

Claude: Running pre-deployment checklist for api-service v2.3.4...
[Executes verification steps, checks dependencies,
monitors rollout, validates success]
```

### On-Call Handoff

```
You: /on-call-handoff

Claude: Generating handoff report...
[Creates comprehensive summary of active issues,
recent incidents, watch items, and runbook updates]
```

## Skills Auto-Activation

Skills activate automatically based on context. For example:

- Mention "incident" or "outage" → Incident Commander activates
- Ask about "error budget" or "SLO" → SLO skill activates
- Say "pod crashing" or "CrashLoopBackOff" → K8s troubleshooting activates
- Discuss "deployment" or "rollout" → Deployment strategies activate

## Best Practices Included

This plugin embeds industry best practices:

- **Google SRE Book** patterns for SLOs and error budgets
- **Incident Command System** for structured response
- **Blameless Culture** principles for postmortems
- **Progressive Delivery** for safe deployments
- **Alert Fatigue Prevention** techniques

## Requirements

- Claude Code CLI (latest version)
- Node.js 18+ (for npx)
- Optional: Access to monitoring/alerting tools for MCP integration

## Contributing

Found an issue or want to add a skill?

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

See [CLAUDE.md](../../CLAUDE.md) for skill development guidelines.

## License

MIT License - See [LICENSE](../../LICENSE) for details.

---

**Built by [latestaiagents](https://github.com/latestaiagents)** | Part of the [Agent Skills](https://github.com/latestaiagents/agent-skills) collection
