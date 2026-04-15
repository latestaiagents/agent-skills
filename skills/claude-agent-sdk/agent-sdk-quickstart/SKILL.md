---
name: agent-sdk-quickstart
description: |
  Build your first Claude Agent SDK agent — TypeScript or Python. Covers installation, minimal agent loop, tool definitions, and the difference between "roll-your-own" and Managed Agents.
  Use this skill when starting a new AI agent project, migrating from raw messages.create loops to the Agent SDK, or evaluating SDK vs direct API.
  Activate when: Claude Agent SDK, @anthropic-ai/claude-agent-sdk, build agent, agent SDK quickstart, agent tutorial.
---

# Claude Agent SDK — Quickstart

**The Claude Agent SDK is Anthropic's framework for building agents. It handles the tool-use loop, context compaction, sub-agent spawning, and MCP integration so you don't reinvent them.**

## When to Use

- Starting a new agent project
- Migrating from hand-rolled `messages.create` loops
- Need sub-agent orchestration, compaction, or skills out-of-box
- Building CLI tools, automation, or backend agents

## SDK vs Direct API vs Managed Agents

| Approach | Best for | Tradeoff |
|---|---|---|
| **Direct `messages.create`** | Simple one-shot calls | You manage everything |
| **Agent SDK** (client-side) | Custom agents with full control | You run the loop, manage state |
| **Managed Agents API** (`/v1/agents`) | Production agents at scale | Anthropic runs the loop server-side |

Use the SDK when you need more control than Managed Agents but don't want to rewrite the tool loop.

## Install

TypeScript:
```bash
npm i @anthropic-ai/claude-agent-sdk
```

Python:
```bash
pip install claude-agent-sdk
```

## Minimal Agent — TypeScript

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "What files are in the current directory?",
  options: {
    model: "claude-sonnet-4-6",
    permissionMode: "default",
  },
})) {
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "text") process.stdout.write(block.text);
    }
  }
}
```

Under the hood: the SDK connects to Claude, provides default tools (file read, bash, web fetch), runs the tool-use loop, and streams results back.

## Minimal Agent — Python

```python
import asyncio
from claude_agent_sdk import query

async def main():
    async for msg in query(
        prompt="What files are here?",
        options={"model": "claude-sonnet-4-6"},
    ):
        if msg.type == "assistant":
            for block in msg.message.content:
                if block.type == "text":
                    print(block.text, end="")

asyncio.run(main())
```

## Adding Custom Tools

```ts
import { query, tool } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const getWeather = tool({
  name: "get_weather",
  description: "Get current weather for a city.",
  inputSchema: z.object({ city: z.string() }),
  handler: async ({ city }) => ({
    content: [{ type: "text", text: `Weather in ${city}: sunny, 72°F` }],
  }),
});

for await (const message of query({
  prompt: "What's the weather in SF?",
  options: {
    model: "claude-sonnet-4-6",
    tools: [getWeather],
  },
})) { /* ... */ }
```

Tools are plain objects with a name, description, schema, and async handler. No magic.

## Permission Modes

```ts
options: {
  permissionMode: "default",        // prompt user for risky actions
  // or "acceptEdits"                // auto-approve edits, prompt for bash
  // or "bypassPermissions"          // YOLO (dev only)
  // or "plan"                       // read-only, no writes
}
```

Use `plan` for dry-runs, `default` for interactive tools, `acceptEdits` only when you trust the agent.

## Hooks

Intercept tool calls before/after execution:

```ts
options: {
  hooks: {
    PreToolUse: async ({ toolName, input }) => {
      console.log(`About to run ${toolName}`);
      if (toolName === "bash" && input.command.includes("rm -rf")) {
        return { decision: "block", reason: "Blocked destructive command" };
      }
    },
    PostToolUse: async ({ toolName, output }) => {
      await audit.log({ toolName, output });
    },
  },
}
```

Hooks run on your machine and can block, modify, or observe every tool call.

## Connecting to MCP Servers

```ts
options: {
  mcpServers: {
    github: { command: "npx", args: ["-y", "@modelcontextprotocol/server-github"] },
  },
}
```

MCP tools appear alongside built-in tools. See the `mcp-client-integration` skill.

## System Prompts

```ts
options: {
  systemPrompt: "You are a code reviewer. Be concise and actionable.",
  // or append to the default:
  appendSystemPrompt: "Always use TypeScript in examples.",
}
```

## Multi-Turn

Use `resume` to continue a previous session:

```ts
const sessionId = "session-123";

for await (const msg of query({
  prompt: "Follow up: what about tests?",
  options: { model: "claude-sonnet-4-6", resume: sessionId },
})) { /* ... */ }
```

See the `session-lifecycle` skill for more.

## Anti-Patterns

1. **Rebuilding the tool loop manually** when the SDK handles it
2. **`bypassPermissions` in production** — first hallucinated rm is your last
3. **Not using hooks** for audit/safety — missed opportunity
4. **Mixing SDK calls with direct `messages.create`** in the same agent — confuses session state

## Best Practices

1. Start with the SDK; drop to direct API only if SDK blocks something specific
2. Use `plan` or `default` permission mode in prod; never `bypassPermissions`
3. Add PreToolUse hooks for audit and safety checks
4. Use `resume` for multi-turn rather than rebuilding history yourself
5. Connect MCP servers for ecosystem tools; don't reimplement GitHub/Slack tools by hand
6. Keep system prompts short — big prompts cost per call (cache them)
