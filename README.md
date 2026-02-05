# Agent Skills by latestaiagents

**67 professional skills + 7 full-featured plugins for AI coding agents** — organized by audience so you can find exactly what you need.

Works with **Claude Code**, **Claude Cowork**, Cursor, Codex, Windsurf, and [35+ other AI agents](https://skills.sh).

```bash
# Cross-platform skills (works with any AI agent)
npx skills add latestaiagents/agent-skills --all

# Claude Code / Cowork plugins (with MCP integrations)
/plugin marketplace add latestaiagents/agent-skills
/plugin install devops-sre@latestaiagents-agent-skills
```

---

## Quick Start

### Option 1: Skills CLI (Cross-Platform)

Works with Claude Code, Cursor, Codex, Windsurf, Cline, Aider, and 35+ other AI agents.

```bash
npx skills add latestaiagents/agent-skills --all
```

### Option 2: Claude Code / Cowork Plugins

Full plugins with **MCP tool integrations** and **slash commands**. Works with Claude Code CLI and Claude Cowork desktop app.

```bash
# Step 1: Add our marketplace
/plugin marketplace add latestaiagents/agent-skills

# Step 2: Install plugins you need
/plugin install devops-sre@latestaiagents-agent-skills
/plugin install qa-testing@latestaiagents-agent-skills
/plugin install hr-people-ops@latestaiagents-agent-skills
```

### Install by Audience

| You Are | Skills CLI | Claude Plugin |
|---------|-----------|---------------|
| **Everyone** | `npx skills add latestaiagents/agent-skills/skills/safety --all` | `/plugin install safety@latestaiagents-agent-skills` |
| **Developer** | `npx skills add latestaiagents/agent-skills/skills/developer --all` | `/plugin install developer-toolkit@latestaiagents-agent-skills` |
| **DevOps/SRE** | `npx skills add latestaiagents/agent-skills/skills/mlops --all` | `/plugin install devops-sre@latestaiagents-agent-skills` |
| **RAG Engineer** | `npx skills add latestaiagents/agent-skills/skills/rag-architect --all` | `/plugin install rag-plugin@latestaiagents-agent-skills` |
| **Security Engineer** | `npx skills add latestaiagents/agent-skills/skills/security --all` | `/plugin install security-guardian@latestaiagents-agent-skills` |
| **QA/Testing** | — | `/plugin install qa-testing@latestaiagents-agent-skills` |
| **HR/People Ops** | — | `/plugin install hr-people-ops@latestaiagents-agent-skills` |

---

## What Are Skills?

Skills are **instructions that teach AI agents how to handle specific tasks**. After installation, your AI assistant automatically knows:

- How to resolve merge conflicts systematically (not just "fix it somehow")
- How to safely run database migrations (backup first, check existing data)
- How to design multi-agent systems (patterns that actually work)
- When to ask for confirmation before destructive operations

**No special commands needed** — skills activate automatically based on what you're doing.

---

## Skill Categories

### Safety (Essential for Everyone)

> Safety skills that prevent accidental data loss. **Recommended for all users.**

```bash
npx skills add latestaiagents/agent-skills/skills/safety --all
```

| Skill | What It Does |
|-------|--------------|
| `destructive-operation-guard` | Core safety protocols for all destructive operations |
| `migration-safety` | Safe database migrations with backup requirements |
| `database-safety` | Prevent accidental DELETE, DROP, TRUNCATE |
| `file-operation-safety` | Protection against rm -rf and bulk deletions |
| `git-safety` | Guard against force push, reset --hard, history loss |

---

### Developer (For Software Developers)

> Git workflows, code intelligence, and debugging tools — 19 skills total.

```bash
npx skills add latestaiagents/agent-skills/skills/developer --all
```

#### Git Mastery (6 skills)

| Skill | What It Does |
|-------|--------------|
| `merge-conflict-surgeon` | Step-by-step conflict resolution with context analysis |
| `commit-message-crafter` | Conventional commits that tell a story |
| `branch-strategy-advisor` | GitFlow vs trunk-based — choose what fits |
| `git-history-detective` | Find exactly when and where bugs were introduced |
| `rebase-safely` | Interactive rebase without losing work |
| `git-undo-wizard` | Recover from reset, rebase, and force push disasters |

#### Code Intelligence (7 skills)

| Skill | What It Does |
|-------|--------------|
| `codebase-context-builder` | Create CLAUDE.md and optimal context for AI |
| `ai-code-reviewer` | Systematic review of AI-generated code |
| `refactor-with-ai` | Safe, incremental refactoring workflows |
| `test-generation-patterns` | AI-driven test creation that actually works |
| `debug-with-ai` | Structured debugging: hypothesize → verify → fix |
| `doc-sync-automation` | Keep docs updated when code changes |
| `code-explanation-generator` | Clear explanations for complex code |

#### Debug Detective (6 skills)

| Skill | What It Does |
|-------|--------------|
| `error-pattern-analyzer` | Group errors by root cause, find patterns |
| `log-forensics` | Extract insights from log files |
| `stack-trace-decoder` | Parse and explain complex stack traces |
| `reproduction-builder` | Create minimal reproduction cases |
| `performance-profiler` | Identify bottlenecks systematically |
| `dependency-conflict-resolver` | Untangle version conflicts |

---

### MLOps (For Architects & Platform Engineers)

> Multi-agent systems, LLMOps, and production AI patterns — 20 skills total.

```bash
npx skills add latestaiagents/agent-skills/skills/mlops --all
```

#### Agent Architect (13 skills)

| Skill | What It Does |
|-------|--------------|
| `agent-supervisor-pattern` | Hub-and-spoke orchestration with reasoning transparency |
| `agent-mesh-architecture` | Peer-to-peer resilient agent networks |
| `agent-handoff-protocols` | Clean task delegation between agents |
| `agent-memory-systems` | Working, short-term, and long-term memory patterns |
| `agent-tool-routing` | Dynamic tool selection and MCP integration |
| `agent-error-recovery` | Circuit breakers, retry policies, graceful degradation |
| `agent-cost-budgeting` | Token allocation across agent swarms |
| `agent-testing-harness` | Unit and integration testing for agents |
| `langgraph-workflows` | LangGraph 1.0 state machines and graph patterns |
| `durable-state-patterns` | Persistent agent state across failures/restarts |
| `human-in-loop-agents` | Approval workflows and agent supervision |
| `agent-checkpointing` | Recovery, replay, and debugging patterns |
| `a2a-protocols` | Agent-to-Agent communication and MCP integration |

#### LLMOps Guardian (7 skills)

| Skill | What It Does |
|-------|--------------|
| `token-cost-analyzer` | Audit usage with 2026 pricing (Claude, GPT-4, etc.) |
| `prompt-caching-patterns` | Reduce costs by 73%+ with semantic caching |
| `model-routing-strategy` | Route to cheaper models when appropriate |
| `llm-rate-limiting` | Token bucket, exponential backoff, circuit breakers |
| `ai-audit-logging` | EU AI Act compliant audit trails |
| `prompt-injection-guard` | Input validation, canary tokens, output filtering |
| `llm-fallback-chains` | Multi-provider failover strategies |

---

### RAG Architect (For AI/ML Engineers)

> Build production-ready RAG systems with 2026 best practices — 7 skills total.

```bash
npx skills add latestaiagents/agent-skills/skills/rag-architect --all
```

| Skill | What It Does |
|-------|--------------|
| `hybrid-retrieval` | Vector + keyword search with RRF fusion and reranking |
| `chunking-strategies` | Optimal chunking: semantic, parent-child, late chunking |
| `graphrag-patterns` | Knowledge graph + RAG for multi-hop reasoning |
| `agentic-rag` | Agent-driven retrieval with planning and reflection |
| `corrective-rag` | Self-healing RAG with document grading and fallbacks |
| `rag-evaluation` | RAGAS metrics, testing, and benchmarking |
| `production-rag-checklist` | End-to-end production deployment guide |

---

### Security (For Security Engineers & All Developers)

> Comprehensive security skills including OWASP Top 10 and common security practices — 16 skills total.

```bash
npx skills add latestaiagents/agent-skills/skills/security --all
```

#### OWASP Guardian (10 skills)

| Skill | What It Does |
|-------|--------------|
| `injection-prevention` | SQL, NoSQL, and command injection prevention |
| `broken-auth-detector` | Authentication and session security |
| `sensitive-data-protection` | Encryption, data masking, secure storage |
| `xxe-prevention` | XML External Entity attack prevention |
| `access-control-audit` | Authorization and IDOR detection |
| `security-misconfiguration` | Server, cloud, and app hardening |
| `xss-prevention` | Cross-site scripting prevention |
| `insecure-deserialization` | Safe object serialization |
| `dependency-vulnerability` | Package and supply chain security |
| `logging-monitoring` | Security event logging and alerting |

#### Common Security (6 skills)

| Skill | What It Does |
|-------|--------------|
| `api-security` | REST and GraphQL API security |
| `secrets-detection` | Find leaked credentials and API keys |
| `secure-code-review` | Security-focused code review methodology |
| `csrf-protection` | Cross-site request forgery prevention |
| `secure-headers` | HTTP security headers (CSP, HSTS, etc.) |
| `jwt-security` | Secure JWT implementation |

---

## Claude Code / Cowork Plugins (With MCP)

These 7 plugins include **MCP tool integrations** for connecting to external services like Datadog, PagerDuty, Jira, Pinecone, and more.

> **Note**: For complete plugin documentation, see the [official Claude Code plugins guide](https://code.claude.com/docs/en/plugins).

### Available Plugins

| Plugin | Skills | Commands | MCP Integrations |
|--------|--------|----------|------------------|
| **devops-sre** | 8 | 6 | Datadog, PagerDuty, AWS, Kubernetes, Prometheus |
| **rag-plugin** | 7 | 3 | Pinecone, Weaviate, Qdrant, Neo4j, Elasticsearch |
| **security-guardian** | 10 | 2 | Snyk, Semgrep, Trivy, Vault, GitHub Security |
| **agent-plugin** | 13 | 2 | LangSmith, Redis, PostgreSQL, OpenAI |
| **developer-toolkit** | 19 | 2 | GitHub, GitLab, Jira, Linear, Sentry |
| **hr-people-ops** | 6 | 2 | Greenhouse, Lever, Workday, BambooHR |
| **qa-testing** | 6 | 3 | TestRail, Xray, BrowserStack, Jira |

### Plugin Installation

```bash
# Step 1: Add our marketplace (one-time)
/plugin marketplace add latestaiagents/agent-skills

# Step 2: Install plugins
/plugin install devops-sre@latestaiagents-agent-skills
/plugin install rag-plugin@latestaiagents-agent-skills
/plugin install security-guardian@latestaiagents-agent-skills
/plugin install agent-plugin@latestaiagents-agent-skills
/plugin install developer-toolkit@latestaiagents-agent-skills
/plugin install hr-people-ops@latestaiagents-agent-skills
/plugin install qa-testing@latestaiagents-agent-skills
```

### Plugin Features

Each plugin includes:
- **Skills**: Auto-activated based on context
- **Slash Commands**: Quick access to common workflows (e.g., `/incident-response`)
- **MCP Configs**: Pre-configured connections to external tools (optional, requires your API keys)

### MCP Setup

After installing a plugin, configure MCP integrations:

1. Open the plugin's `.mcp.json` file
2. Add your API keys for the services you use
3. All MCP integrations are marked `optional: true` — only enable what you need

Example (DevOps SRE plugin):
```json
{
  "mcpServers": {
    "datadog": {
      "command": "npx",
      "args": ["-y", "@datadog/mcp-server"],
      "env": { "DD_API_KEY": "your-key", "DD_APP_KEY": "your-key" }
    }
  }
}
```

---

## Installation Options (Detailed)

### Option 1: Skills CLI (Cross-Platform)

Works with Claude Code, Cursor, Codex, Windsurf, Cline, Aider, and 35+ other AI agents.

```bash
# Everything
npx skills add latestaiagents/agent-skills --all

# By category
npx skills add latestaiagents/agent-skills/skills/safety --all
npx skills add latestaiagents/agent-skills/skills/developer --all
npx skills add latestaiagents/agent-skills/skills/mlops --all
npx skills add latestaiagents/agent-skills/skills/rag-architect --all
npx skills add latestaiagents/agent-skills/skills/security --all

# Individual suite
npx skills add latestaiagents/agent-skills/skills/developer/git-mastery --all

# Single skill
npx skills add latestaiagents/agent-skills/skills/developer/git-mastery/git-undo-wizard
```

### Option 2: Claude Marketplace (Recommended for Claude Code/Cowork)

```bash
# Step 1: Add the marketplace (one-time)
/plugin marketplace add latestaiagents/agent-skills

# Step 2: Browse available plugins
/plugin marketplace list

# Step 3: Install what you need
/plugin install devops-sre@latestaiagents-agent-skills
/plugin install qa-testing@latestaiagents-agent-skills

# Skills-only (no MCP)
/plugin install safety@latestaiagents-agent-skills
/plugin install git-mastery@latestaiagents-agent-skills
/plugin install owasp-guardian@latestaiagents-agent-skills
```

### Option 3: Direct Plugin Install

```bash
# Install full plugin directly
/plugin install latestaiagents/agent-skills/plugins/devops-sre
/plugin install latestaiagents/agent-skills/plugins/qa-testing
```

---

## Slash Commands (Claude Code / Cowork)

After installing plugins, you get **20+ powerful slash commands**:

### DevOps/SRE Commands
| Command | What It Does |
|---------|--------------|
| `/incident-response` | Start incident workflow with severity assessment |
| `/runbook` | Generate or execute operational runbooks |
| `/postmortem` | Create blameless postmortem documents |
| `/slo-check` | Check SLO/SLI compliance and error budgets |
| `/on-call` | On-call handoff and escalation workflows |
| `/k8s-debug` | Debug Kubernetes pods, deployments, services |

### RAG Commands
| Command | What It Does |
|---------|--------------|
| `/build-rag` | Design and implement RAG pipelines |
| `/rag-debug` | Debug RAG retrieval quality issues |
| `/eval-rag` | Evaluate RAG system with RAGAS metrics |

### Security Commands
| Command | What It Does |
|---------|--------------|
| `/security-scan` | Run security analysis on code |
| `/fix-vulnerability` | Get remediation for vulnerabilities |

### Developer Commands
| Command | What It Does |
|---------|--------------|
| `/git-undo` | Recover from any Git mistake |
| `/debug-this` | Structured debugging workflow |

### QA/Testing Commands
| Command | What It Does |
|---------|--------------|
| `/write-test-cases` | Generate comprehensive test cases |
| `/test-plan` | Create test strategy documents |
| `/bug-report` | Structure actionable bug reports |

### HR/People Ops Commands
| Command | What It Does |
|---------|--------------|
| `/write-job-description` | Create inclusive job descriptions |
| `/performance-review` | Write effective performance reviews |

### Agent Architect Commands
| Command | What It Does |
|---------|--------------|
| `/design-agent` | Design multi-agent architectures |
| `/debug-agent` | Debug agent workflows and state |

**Usage:**
```
/incident-response
```

The command guides you step-by-step through the workflow.

---

## How Skills Work

Skills activate **automatically** based on what you're doing:

| You Say | Skill Activates | What Happens |
|---------|-----------------|--------------|
| "I have a merge conflict" | `merge-conflict-surgeon` | Systematic conflict resolution workflow |
| "Run the database migration" | `migration-safety` | Checks existing data, creates backup, asks confirmation |
| "Delete all the old log files" | `file-operation-safety` | Shows what will be deleted, asks for explicit confirmation |
| "Help me design a multi-agent system" | `agent-supervisor-pattern` | Provides architecture patterns and code templates |
| "Why is this API so expensive?" | `token-cost-analyzer` | Analyzes token usage with current pricing |
| "Build a RAG system" | `hybrid-retrieval` | Vector + keyword search with reranking |
| "My RAG gives wrong answers" | `corrective-rag` | Self-healing RAG with document grading |
| "Implement GraphRAG" | `graphrag-patterns` | Knowledge graph integration for multi-hop reasoning |
| "Is this code secure?" | `secure-code-review` | Security-focused code review methodology |
| "Check for SQL injection" | `injection-prevention` | SQL injection detection and prevention |

**No special commands needed** — just describe what you're doing.

---

## Staying Updated

Check for skill updates and keep your installation current:

```bash
# Check for available updates
npx skills check

# Update all installed skills to latest versions
npx skills update

# List your installed skills
npx skills list
```

---

## Directory Structure

```
agent-skills/
├── skills/                      # Cross-platform skills (via skills.sh)
│   ├── safety/                  # Essential safety (5 skills)
│   │   └── ai-safety-guardrails/
│   ├── developer/               # For developers (19 skills)
│   │   ├── git-mastery/
│   │   ├── code-intelligence/
│   │   └── debug-detective/
│   ├── mlops/                   # For architects (20 skills)
│   │   ├── agent-architect/     # Multi-agent + LangGraph (13 skills)
│   │   └── llmops-guardian/     # LLM operations (7 skills)
│   ├── rag-architect/           # RAG systems (7 skills)
│   └── security/                # Security skills (16 skills)
│       ├── owasp-guardian/      # OWASP Top 10 (10 skills)
│       └── common-security/     # Common practices (6 skills)
│
├── plugins/                     # Claude Code / Cowork plugins (with MCP)
│   ├── devops-sre/              # DevOps/SRE (8 skills, 6 commands)
│   │   ├── .claude-plugin/
│   │   ├── .mcp.json            # Datadog, PagerDuty, AWS, K8s
│   │   ├── commands/
│   │   └── skills/
│   ├── rag-architect/           # RAG (7 skills, 3 commands)
│   │   └── .mcp.json            # Pinecone, Weaviate, Neo4j
│   ├── security-guardian/       # Security (10 skills, 2 commands)
│   │   └── .mcp.json            # Snyk, Semgrep, Trivy, Vault
│   ├── agent-architect/         # Multi-agent (13 skills, 2 commands)
│   │   └── .mcp.json            # LangSmith, Redis
│   ├── developer-toolkit/       # Developer (19 skills, 2 commands)
│   │   └── .mcp.json            # GitHub, Jira, Sentry
│   ├── hr-people-ops/           # HR (6 skills, 2 commands)
│   │   └── .mcp.json            # Greenhouse, Lever, Workday
│   └── qa-testing/              # QA (6 skills, 3 commands)
│       └── .mcp.json            # TestRail, BrowserStack
│
├── commands/                    # Base slash commands
├── .claude-plugin/              # Root plugin configuration
│   ├── plugin.json
│   └── marketplace.json         # All plugins listed here
├── CLAUDE.md                    # AI context file
└── README.md
```

---

## Compatibility

Works with any AI agent that supports [skills.sh](https://skills.sh):

- Claude Code (Anthropic)
- Cursor
- Codex CLI (OpenAI)
- Windsurf
- Cline
- Aider
- Continue
- And 30+ more...

---

## Contributing

Found a bug? Have a skill idea? [Open an issue](https://github.com/latestaiagents/agent-skills/issues).

See [CLAUDE.md](CLAUDE.md) for development guidelines.

---

## Security & Trust

### MCP Server Security

Plugins include optional MCP (Model Context Protocol) configurations. Before enabling:

- **Review credentials**: Only connect tools you trust
- **Use environment variables**: Never commit API keys to version control
- **Test environments**: Be careful with production credentials
- **Audit configs**: Review `.mcp.json` files before copying

All MCP integrations are marked `optional: true` — only enable what you need.

### Data Privacy

- Skills run locally — no data sent to external servers
- MCP servers connect only to services you configure
- No telemetry or tracking in this repository

---

## Official Documentation

- **Claude Code Plugins**: [code.claude.com/docs/en/plugins](https://code.claude.com/docs/en/plugins)
- **Skills.sh**: [skills.sh](https://skills.sh)
- **MCP Protocol**: [modelcontextprotocol.io](https://modelcontextprotocol.io)

---

## Disclaimer

**USE AT YOUR OWN RISK.**

These skills are provided as educational resources and best-practice guidelines for AI agents. By using these skills, you acknowledge:

1. **No Warranty**: Skills are provided "as is" without warranty of any kind.
2. **No Liability**: Authors shall not be held liable for any damages or data loss.
3. **Your Responsibility**: Test in safe environments, maintain backups, review AI actions before execution.
4. **AI Behavior**: AI agents may interpret skills differently. Always confirm before destructive operations.
5. **MCP Integrations**: Connecting to external services is at your own risk. Secure your API keys appropriately.

---

## License

MIT License — See [LICENSE](LICENSE) file.

Use freely in personal and commercial projects.
