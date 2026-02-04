# latestaiagents Skills

**39 professional skills for AI coding agents** — Git workflows, debugging, multi-agent architecture, LLMOps, and safety guardrails.

Works with Claude Code, Cursor, Codex, Windsurf, and [35+ other AI agents](https://skills.sh).

```bash
npx skills add latestaiagents/skills
```

---

## What Are Skills?

Skills are **instructions that teach AI agents how to handle specific tasks**. When you install skills, your AI assistant automatically knows:

- How to resolve merge conflicts systematically (not just "fix it somehow")
- How to safely run database migrations (backup first, check existing data)
- How to design multi-agent systems (patterns that actually work)
- When to ask for confirmation before destructive operations

**Skills make AI agents smarter and safer.**

---

## How to Use

### 1. Install Skills

```bash
# Install all skills
npx skills add latestaiagents/skills

# Or install a specific suite
npx skills add latestaiagents/skills/skills/git-mastery
```

### 2. Just Talk to Your AI Agent

After installation, skills activate **automatically** based on what you're doing. No special commands needed.

**Examples:**

| You Say | Skill Activates | What Happens |
|---------|-----------------|--------------|
| "I have a merge conflict" | `merge-conflict-surgeon` | Systematic conflict resolution workflow |
| "Run the database migration" | `migration-safety` | Checks existing data, creates backup, asks confirmation |
| "Delete all the old log files" | `file-operation-safety` | Shows what will be deleted, asks for explicit confirmation |
| "Help me design a multi-agent system" | `agent-supervisor-pattern` | Provides architecture patterns and code templates |
| "Why is this API so expensive?" | `token-cost-analyzer` | Analyzes token usage with 2026 pricing |

### 3. Skills Protect You Automatically

The Safety Guardrails skills are **always active**. Your AI agent will:

- ✅ Ask before running `rm -rf` on any directory
- ✅ Create backups before destructive database operations
- ✅ Warn about `git push --force` on shared branches
- ✅ Show exactly how many records will be affected before DELETE
- ✅ Require explicit confirmation like "yes, delete 1234 rows"

**No more accidental data loss.**

---

## Skill Suites

### 🔧 Git Mastery
> Stop fighting Git, start shipping code

| Skill | What It Does |
|-------|--------------|
| `merge-conflict-surgeon` | Step-by-step conflict resolution with context analysis |
| `commit-message-crafter` | Conventional commits that tell a story |
| `branch-strategy-advisor` | GitFlow vs trunk-based — choose what fits |
| `git-history-detective` | Find exactly when and where bugs were introduced |
| `rebase-safely` | Interactive rebase without losing work |
| `git-undo-wizard` | Recover from reset, rebase, and force push disasters |

---

### 🧠 Code Intelligence
> Ship 10x faster with AI that understands your codebase

| Skill | What It Does |
|-------|--------------|
| `codebase-context-builder` | Create CLAUDE.md and optimal context for AI |
| `ai-code-reviewer` | Systematic review of AI-generated code |
| `refactor-with-ai` | Safe, incremental refactoring workflows |
| `test-generation-patterns` | AI-driven test creation that actually works |
| `debug-with-ai` | Structured debugging: hypothesize → verify → fix |
| `doc-sync-automation` | Keep docs updated when code changes |
| `code-explanation-generator` | Clear explanations for complex code |

---

### 🤖 Agent Architect
> Design multi-agent systems that actually work

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

---

### 🔍 Debug Detective
> Find bugs in minutes, not hours

| Skill | What It Does |
|-------|--------------|
| `error-pattern-analyzer` | Group errors by root cause, find patterns |
| `log-forensics` | Extract insights from log files |
| `stack-trace-decoder` | Parse and explain complex stack traces |
| `reproduction-builder` | Create minimal reproduction cases |
| `performance-profiler` | Identify bottlenecks systematically |
| `dependency-conflict-resolver` | Untangle version conflicts |

---

### 📊 LLMOps Guardian
> Production LLM systems that don't break the bank or the law

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

### 🛡️ AI Safety Guardrails
> Never let AI destroy your data without asking first

| Skill | What It Does |
|-------|--------------|
| `destructive-operation-guard` | Core safety protocols for all destructive operations |
| `migration-safety` | Safe database migrations with backup requirements |
| `database-safety` | Prevent accidental DELETE, DROP, TRUNCATE |
| `file-operation-safety` | Protection against rm -rf and bulk deletions |
| `git-safety` | Guard against force push, reset --hard, history loss |

---

## Installation Options

### Option 1: Skills (Works with 35+ AI Agents)

```bash
# All 39 skills
npx skills add latestaiagents/skills

# Just Git skills
npx skills add latestaiagents/skills/skills/git-mastery

# Just safety guardrails
npx skills add latestaiagents/skills/skills/ai-safety-guardrails

# Single skill
npx skills add latestaiagents/skills/skills/git-mastery/git-undo-wizard
```

### Option 2: Plugin (Claude Code Only — With Slash Commands)

```bash
# In Claude Code, run:
/plugin add latestaiagents/skills
```

### Option 3: Add Our Marketplace (Claude Code — Browse All Plugins)

Add our marketplace to browse and install individual suites:

**Step 1: Add the marketplace**
```bash
/plugin marketplace add latestaiagents/skills
```

Or via Settings → Plugins → "Add marketplace by URL":
```
https://github.com/latestaiagents/skills.git
```

**Step 2: Install plugins from the marketplace**
```bash
# Install specific suite
/plugin install git-mastery@latestaiagents
/plugin install ai-safety-guardrails@latestaiagents

# Or install everything
/plugin install latestaiagents-full@latestaiagents
```

**Available plugins in our marketplace:**

| Plugin | Install Command |
|--------|-----------------|
| Complete Suite (39 skills + 10 commands) | `/plugin install latestaiagents-full@latestaiagents` |
| Git Mastery (6 skills) | `/plugin install git-mastery@latestaiagents` |
| Code Intelligence (7 skills) | `/plugin install code-intelligence@latestaiagents` |
| Agent Architect (8 skills) | `/plugin install agent-architect@latestaiagents` |
| Debug Detective (6 skills) | `/plugin install debug-detective@latestaiagents` |
| LLMOps Guardian (7 skills) | `/plugin install llmops-guardian@latestaiagents` |
| AI Safety Guardrails (5 skills) | `/plugin install ai-safety-guardrails@latestaiagents` |

**Marketplace management commands:**
```bash
/plugin marketplace list              # List all added marketplaces
/plugin marketplace update latestaiagents  # Update to get new plugins
/plugin marketplace remove latestaiagents  # Remove marketplace
```

---

## Plugin: Slash Commands (Claude Code)

After installing the plugin, you get **10 powerful slash commands**:

| Command | What It Does |
|---------|--------------|
| `/git-undo` | Recover from any Git mistake — reset, rebase, force push |
| `/resolve-conflict` | Systematic merge conflict resolution |
| `/safety-check` | Run safety checks before destructive operations |
| `/migration-check` | Analyze database migrations for risks |
| `/design-agent` | Design multi-agent system architectures |
| `/cost-analyze` | Analyze and optimize LLM API costs |
| `/debug-this` | Structured debugging workflow |
| `/review-ai-code` | Review AI-generated code for bugs and security |
| `/build-context` | Create optimal CLAUDE.md for your codebase |
| `/explain-code` | Generate clear explanations for complex code |

### How to Use Slash Commands

1. **Install the plugin:**
   ```
   /plugin add latestaiagents/skills
   ```

2. **Type any command:**
   ```
   /git-undo
   ```

3. **Follow the interactive workflow** — the command guides you step by step.

### Example: /migration-check

```
You: /migration-check

Claude: I'll analyze your pending migrations for risks.

## Step 1: Checking environment...
Environment: development
Database: myapp_dev

## Step 2: Pending migrations found:
1. 2024_01_15_drop_legacy_users (⛔ CRITICAL - drops table with 1,247 rows)
2. 2024_01_16_add_email_index (🟢 LOW - safe index addition)

## Recommendation:
Create backup before running: pg_dump -t legacy_users db > backup.sql

Proceed with migration? Type "yes, I have a backup" to continue.
```

---

## Compatibility

Works with any AI agent that supports [skills.sh](https://skills.sh):

- **Claude Code** (Anthropic)
- **Cursor**
- **Codex CLI** (OpenAI)
- **Windsurf**
- **Cline**
- **Aider**
- **Continue**
- **And 30+ more...**

---

## How Skills Work (Technical)

Each skill is a `SKILL.md` file with:

1. **YAML frontmatter** — Name and description (used for auto-activation)
2. **Markdown body** — Detailed instructions, workflows, code examples

When your AI agent encounters a relevant task, it reads the skill and follows the instructions. The `description` field contains trigger keywords that tell the agent when to activate.

Example structure:
```
skills/
├── git-mastery/
│   ├── merge-conflict-surgeon/
│   │   └── SKILL.md
│   └── git-undo-wizard/
│       └── SKILL.md
└── ai-safety-guardrails/
    └── migration-safety/
        └── SKILL.md
```

---

## Contributing

Found a bug? Have a skill idea? [Open an issue](https://github.com/latestaiagents/skills/issues).

---

## Disclaimer

**USE AT YOUR OWN RISK.**

These skills are provided as educational resources and best-practice guidelines for AI agents. By using these skills, you acknowledge and agree that:

1. **No Warranty**: The skills are provided "as is" without warranty of any kind, express or implied. We make no guarantees about the accuracy, reliability, completeness, or suitability of these skills for any purpose.

2. **No Liability**: The authors and contributors shall not be held liable for any damages, data loss, system failures, or other issues arising from the use of these skills. This includes but is not limited to:
   - Accidental deletion of files or data
   - Database corruption or data loss
   - Failed migrations or deployments
   - Security vulnerabilities
   - Financial losses from API usage
   - Any direct, indirect, incidental, or consequential damages

3. **Your Responsibility**: You are solely responsible for:
   - Testing skills in a safe environment before production use
   - Maintaining backups of your data
   - Reviewing AI-generated actions before execution
   - Verifying that skills are appropriate for your use case
   - Compliance with applicable laws and regulations

4. **Not Professional Advice**: These skills do not constitute professional, legal, financial, or security advice. Consult qualified professionals for critical decisions.

5. **AI Behavior**: AI agents may interpret these skills differently. Always review and confirm actions before allowing execution, especially for destructive operations.

**By installing or using these skills, you accept full responsibility for any outcomes.**

---

## License

MIT License — See [LICENSE](LICENSE) file.

Use freely in personal and commercial projects, subject to the disclaimer above.
