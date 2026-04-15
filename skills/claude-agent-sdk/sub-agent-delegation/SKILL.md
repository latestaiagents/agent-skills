---
name: sub-agent-delegation
description: |
  Delegate work to sub-agents via the Task/Agent tool — parallel research, isolated context windows, specialized expertise. Covers when sub-agents help vs hurt, prompt shape, and result handling.
  Use this skill when building agents that need to research in parallel, process independent work items, or isolate context-heavy sub-tasks.
  Activate when: sub-agents, Task tool, Agent tool, parallel agents, agent delegation, spawn agent, multi-agent.
---

# Sub-Agent Delegation

**Sub-agents are isolated agent invocations spawned from a parent. They run in a fresh context, do focused work, and return a single result. Use them to parallelize, specialize, and protect context.**

## When to Use

- Parallel research (5 topics × 1 agent each, run concurrently)
- Context-heavy sub-tasks that would bloat the parent's window
- Specialist work (a "code reviewer" sub-agent, a "security auditor" sub-agent)
- Independent work items where errors shouldn't compound

## When NOT to Use

- Trivial single-step tasks — sub-agent overhead isn't worth it
- Tasks that need to see the parent's full context — you'll have to pass it, defeating the isolation
- Highly interactive work — sub-agents don't ask clarifying questions back

## The Model

Parent agent sees a `Task` (or `Agent`) tool. When invoked, it spawns a sub-agent with:

- A fresh conversation (no parent history)
- Its own tool set (may be a subset of parent's)
- A single prompt describing the work
- A requirement to return ONE final response

The sub-agent's intermediate tool calls are NOT visible to the parent — only the final result.

## SDK Usage

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const msg of query({
  prompt: "Research and compare the top 3 vector databases. Summarize pros/cons.",
  options: {
    model: "claude-sonnet-4-6",
    allowedTools: ["Task", "WebFetch", "Read"],
  },
})) { /* ... */ }
```

The parent agent decides to invoke `Task` with a prompt like "Research Pinecone's pros and cons. Use WebFetch. Report in under 200 words." Three parallel Task calls, three independent contexts, three summaries returned.

## Parallel Spawning

The parent can issue multiple tool calls in one step:

```
Parent: Let me research all three in parallel.
[tool_use: Task]   Research Pinecone...
[tool_use: Task]   Research Weaviate...
[tool_use: Task]   Research Qdrant...
```

Three sub-agents run concurrently. Parent waits for all three results, then synthesizes.

## Prompt Shape for Sub-Agents

Sub-agents start with nothing. Prompt them like a smart colleague who just walked in:

1. State the goal
2. Give enough context to make judgment calls
3. Specify output format and length
4. List what's in scope vs out of scope

**Bad prompt:** "Find info about Pinecone"
**Good prompt:** "Research Pinecone as a vector DB for a 10M-vector production workload. Find pricing, latency at 10M scale, clustering support, filtering support. Report as a 200-word summary with bullet points for each metric. Do not cover marketing fluff."

## Handling Sub-Agent Results

```ts
hooks: {
  PostToolUse: async ({ toolName, output }) => {
    if (toolName === "Task") {
      await db.logDelegation({ prompt: output.prompt, result: output.result });
    }
  },
}
```

Log every delegation. Sub-agents can fail, loop, or hallucinate like any agent — audit them.

## Context Economics

Sub-agents cost extra — they run a full model inference. But they save parent tokens:

- Without sub-agent: parent processes full raw research → 50K tokens in context
- With sub-agent: parent sees 500-word summary → 500 tokens in context

Net: you pay more compute, save context window for reasoning. Worth it when parent is doing long reasoning on top of research.

## Specialization Pattern

Create multiple agent "personas" — each a sub-agent type with specialized prompts and tools:

```ts
// Parent system prompt:
"You orchestrate work across:
- 'researcher' sub-agents (WebFetch, Read) for gathering info
- 'reviewer' sub-agents (Read, Grep) for code review
- 'writer' sub-agents (Read, Write) for drafting

Delegate to the right one based on the task."
```

The parent plans; specialists execute.

## Sub-Agent Error Handling

A sub-agent can return errors:

- "I couldn't find relevant info" — retry with refined prompt or move on
- "The tool I need isn't available" — parent decides alternative
- Silent bad output — parent should verify critical results before using

Always have the parent sanity-check sub-agent output before passing to user.

## Limits

- No nested delegation by default — sub-agents don't spawn sub-sub-agents (prevents runaway chains)
- Sub-agent budget tokens come from parent's budget
- Too many parallel sub-agents can rate-limit you
- Sub-agents can't communicate with each other — only through the parent

## Anti-Patterns

1. **Delegating trivial work** — sub-agent spin-up costs more than the task
2. **Needing the parent's context** — passing full history to sub-agent defeats purpose
3. **Sub-agent that returns "the full details"** — parent gets a firehose; should return a summary
4. **No log of delegations** — can't debug when something went wrong
5. **Unbounded parallel spawn** — 50 sub-agents = 50× rate limit consumption

## Best Practices

1. Use sub-agents for research, context isolation, specialist work
2. Prompt them self-contained — they have no prior context
3. Require summarized output (word limit)
4. Parallelize independent work; sequentialize when dependencies exist
5. Log every delegation with prompt and result
6. Cap parallel spawn count (e.g., max 5 concurrent)
7. Have parent verify critical sub-agent output before acting on it
