# Agent Architect Plugin for Claude Code

**The complete toolkit for building production multi-agent systems.**

Design, build, and deploy sophisticated multi-agent architectures with LangGraph workflows, supervisor patterns, human-in-the-loop controls, and durable state management.

## Quick Install

```bash
# Install via Claude Code CLI
/plugin install latestaiagents/agent-skills/plugins/agent-architect

# Or using npx
npx skills add latestaiagents/agent-skills/plugins/agent-architect
```

> **Note**: For complete plugin documentation, see the [official Claude Code plugins guide](https://code.claude.com/docs/en/plugins).

## What's Included

### 2 Slash Commands

| Command | Description |
|---------|-------------|
| `/design-agent` | Design multi-agent architecture for your use case |
| `/debug-agent` | Diagnose and fix multi-agent system issues |

### 13 Professional Skills

#### LangGraph (3 skills)

| Skill | Description |
|-------|-------------|
| **LangGraph Workflows** | Build stateful, cyclical agent workflows |
| **Durable State Patterns** | Persist agent state across sessions |
| **Agent Checkpointing** | Save and resume agent execution |

#### Patterns (9 skills)

| Skill | Description |
|-------|-------------|
| **Agent Supervisor Pattern** | Orchestrate worker agents with oversight |
| **Agent Handoff Protocols** | Transfer context between agents |
| **Human-in-Loop Agents** | Add human approval checkpoints |
| **Agent Mesh Architecture** | Peer-to-peer agent collaboration |
| **A2A Protocols** | Agent-to-agent communication standards |
| **Agent Memory Systems** | Short/long-term memory for agents |
| **Agent Tool Routing** | Dynamic tool selection and execution |
| **Agent Error Recovery** | Graceful failure handling |
| **Agent Cost Budgeting** | Token limits and cost controls |

#### Testing (1 skill)

| Skill | Description |
|-------|-------------|
| **Agent Testing Harness** | Test multi-agent systems reliably |

## MCP Tool Integrations

Optional connections for agent development:

| Tool | What It Enables |
|------|-----------------|
| **LangSmith** | Agent tracing, debugging, evaluation |
| **Redis** | State and memory persistence |
| **PostgreSQL** | Durable checkpointing |
| **OpenAI** | LLM calls for agents |
| **Slack** | Human-in-the-loop notifications |

## Usage Examples

### Design a Multi-Agent System

```
You: /design-agent

I need to build a research assistant that can search the web,
analyze documents, and write summaries.

Claude: I'll design a multi-agent architecture for this...
[Provides supervisor pattern with SearchAgent, AnalyzerAgent,
WriterAgent, complete LangGraph implementation, and state schema]
```

### Debug Agent Issues

```
You: /debug-agent

My supervisor agent keeps selecting the wrong worker.
The search agent gets called when it should be the analyzer.

Claude: Let's diagnose the routing issue...
[Analyzes router prompt, identifies ambiguity, provides
improved routing logic with clearer task descriptions]
```

## Skills Auto-Activation

Skills activate based on context:

- Mention "LangGraph" or "workflow" → LangGraph Workflows
- Ask about "supervisor" or "orchestration" → Supervisor Pattern
- Discuss "human approval" or "human in loop" → Human-in-Loop Agents
- Say "agent memory" or "conversation history" → Memory Systems
- Mention "agent testing" or "test agents" → Testing Harness

## Architecture Patterns

```
Single Agent:
User → Agent → Response

Supervisor (Star):
        ┌─────────────┐
        │  Supervisor │
        └──────┬──────┘
     ┌────┬────┼────┬────┐
     ▼    ▼    ▼    ▼    ▼
    A1   A2   A3   A4   A5

Hierarchical (Tree):
              ┌───────┐
              │ Chief │
              └───┬───┘
        ┌─────────┴─────────┐
        ▼                   ▼
   ┌─────────┐         ┌─────────┐
   │Manager A│         │Manager B│
   └────┬────┘         └────┬────┘
     ┌──┴──┐             ┌──┴──┐
     ▼     ▼             ▼     ▼
    W1    W2            W3    W4

Mesh (P2P):
    A1 ←──→ A2
    ↑  ╲  ╱  ↑
    │   ╲╱   │
    │   ╱╲   │
    ↓  ╱  ╲  ↓
    A4 ←──→ A3
```

## Requirements

- Claude Code CLI (latest version)
- Node.js 18+ (for npx)
- Python 3.9+ (for LangGraph implementations)
- Optional: LangSmith account for tracing

## Security & Trust

This plugin includes MCP server configurations. Before enabling:

- **Review credentials**: Only add API keys for services you trust
- **Principle of least privilege**: Use scoped API keys
- **Audit MCP servers**: Review `.mcp.json` before copying

All MCP integrations are marked `optional: true`.

## Contributing

Want to add an agent pattern?

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

See [CLAUDE.md](../../CLAUDE.md) for development guidelines.

## License

MIT License - See [LICENSE](../../LICENSE) for details.

---

**Built by [latestaiagents](https://github.com/latestaiagents)** | Part of the [Agent Skills](https://github.com/latestaiagents/agent-skills) collection
