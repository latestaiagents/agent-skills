---
name: managed-agents-api
description: |
  Use Anthropic's Managed Agents API (/v1/agents, /v1/sessions) — server-side agent runtime that handles the tool loop, compaction, memory, and scaling for you. Covers when to pick Managed over SDK, request shape, and cost model.
  Use this skill when building production agents at scale, deciding between SDK vs Managed, or migrating from self-hosted agent loops.
  Activate when: Managed Agents API, /v1/agents, /v1/sessions, server-side agent, Anthropic agent runtime.
---

# Managed Agents API

**The Managed Agents API lets Anthropic run the agent loop server-side. You POST a user message; they run the tool loop, manage context, and stream results. You lose control but gain simplicity and durability.**

## When to Use

- Production agents where you don't want to operate infrastructure
- Long-running agents that span hours/days — server-side compaction handles it
- Multi-client agents (web + mobile + CLI) sharing sessions
- Scaling without worrying about loop-runner uptime

## When NOT to Use

- Agents with heavy client-side context (local files, IDE state) — SDK fits better
- You need custom tool-execution environments not exposed by Managed
- Debug-heavy development — self-hosted gives you more visibility

## Core Objects

1. **Agent** (`/v1/agents`) — a template: name, system prompt, tools, model
2. **Session** (`/v1/sessions`) — a live conversation bound to an agent
3. **Message** — added to a session; triggers agent run

## Create an Agent

```ts
const agent = await client.beta.agents.create({
  name: "support-bot",
  description: "Customer support for ACME",
  model: "claude-sonnet-4-6",
  system_prompt: "You are ACME's support agent. Be concise and accurate.",
  tools: [
    { type: "search_tickets", name: "search_tickets", /* schema */ },
    { type: "create_escalation", name: "create_escalation" },
  ],
  mcp_servers: [
    { type: "url", url: "https://mcp.acme.com/sse", name: "acme" },
  ],
});
// Returns: { id: "agt_...", name: "support-bot", ... }
```

Agents are versionable — update without breaking live sessions.

## Start a Session

```ts
const session = await client.beta.sessions.create({
  agent_id: agent.id,
  metadata: { user_id: "u-42", ticket_id: "T-1001" },
});
// Returns: { id: "ses_...", agent_id: "agt_...", status: "idle", ... }
```

Metadata is yours to use for filtering, auditing, linking to your domain.

## Send a Message

```ts
const stream = client.beta.sessions.messages.stream({
  session_id: session.id,
  content: "My order #12345 is stuck — can you help?",
});

for await (const event of stream) {
  if (event.type === "text") console.log(event.delta);
  if (event.type === "tool_use") console.log("TOOL:", event.name);
  if (event.type === "done") break;
}
```

Managed runs the tool loop server-side. You see streamed text, tool calls, and final response. You don't execute tools — Managed does (either built-in tools or via MCP).

## Resuming Sessions

Sessions are durable. A day later:

```ts
const next = await client.beta.sessions.messages.create({
  session_id: session.id,
  content: "Any update on my issue?",
});
```

Server-side compaction keeps context in budget. You don't manage history — it's stored and pruned automatically.

## Listing & Filtering Sessions

```ts
const sessions = await client.beta.sessions.list({
  agent_id: agent.id,
  metadata: { user_id: "u-42" },
  limit: 50,
});
```

Use metadata filters to scope sessions by user, tenant, or workflow instance.

## Tool Execution

Tools are declared on the Agent. Managed executes them by:

- **Built-in tools** (code_execution, web_search, computer, memory) — Anthropic runs
- **MCP servers** — Managed connects to your server and calls tools there
- **Custom tools (URL endpoints)** — Managed POSTs to your URL with tool input

```ts
tools: [
  {
    type: "url",
    name: "ship_order",
    description: "Ship an order",
    input_schema: { /* ... */ },
    url: "https://api.acme.com/tools/ship",
    auth: { type: "bearer", token: "${ACME_TOKEN}" },
  },
]
```

Your endpoint must return the tool result JSON. Managed handles retries and the loop.

## Cost Model

- Token cost: same as direct API
- Infrastructure cost: included (loop runner, storage, compaction)
- MCP/URL tool calls: you pay your own server costs
- Sessions storage: retained per your retention setting

Typically 5-15% more expensive than self-hosted agents in exchange for zero ops.

## Observability

```ts
const events = await client.beta.sessions.events.list({ session_id, limit: 100 });
// Returns loop events: tool_use, tool_result, thinking, compaction, etc.
```

Every step is logged. You can audit, replay, and debug without running infrastructure.

## Migration from SDK

Incremental:

1. Create a matching Agent with your current tools declared
2. For new users, route to Managed
3. For existing SDK sessions, finish them out
4. Drop SDK once all traffic is on Managed

Or parallel for critical flows — run both and A/B test.

## Limitations

- Agent tools must be declarable (MCP or URL) — you can't run arbitrary Node.js tools in Managed
- Client-side context (open editor, local files) can't be in Managed — use SDK
- Fine-grained loop control unavailable — Managed decides when to compact, when to retry

## Best Practices

1. Use Managed for customer-facing agents where you care about uptime more than control
2. Use SDK for dev tools, IDE integrations, local automation
3. Put tools behind MCP or URL endpoints — keeps them reusable across agents
4. Use session metadata for tenant isolation and auditability
5. Set retention policies explicitly (default may be longer than you want)
6. Monitor session event streams for anomalies — long loops, repeated tool failures
