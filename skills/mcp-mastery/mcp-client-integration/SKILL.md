---
name: mcp-client-integration
description: |
  Connect MCP servers to AI clients — Claude Desktop, Claude Code, Cursor, the Claude Agent SDK, and custom apps — including config file locations, auth, and debugging connection failures.
  Use this skill when the user wants to wire an MCP server into a client, troubleshoot "server not connecting" errors, or build a custom MCP client with the SDK.
  Activate when: MCP client, mcp.json, claude_desktop_config.json, connect MCP, MCP not working, list MCP tools, programmatic MCP client.
---

# MCP Client Integration

**Wire any MCP server into the client that will consume it — declaratively for IDEs, programmatically for custom apps.**

## When to Use

- Installing an MCP server into Claude Desktop, Claude Code, or Cursor
- Building a custom MCP client (web app, CLI, agent framework)
- Debugging "server failed to start" or "no tools showing up"
- Sharing MCP config across a team

## Config File Locations

| Client | Config path |
|---|---|
| Claude Desktop (macOS) | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Claude Desktop (Windows) | `%APPDATA%\Claude\claude_desktop_config.json` |
| Claude Code | `~/.claude/settings.json` (user) or `.mcp.json` (project) |
| Cursor | `~/.cursor/mcp.json` (user) or `.cursor/mcp.json` (project) |

## Declarative Install — stdio Server

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

## Declarative Install — HTTP/SSE Server

```json
{
  "mcpServers": {
    "linear": {
      "url": "https://mcp.linear.app/sse",
      "headers": {
        "Authorization": "Bearer ${LINEAR_TOKEN}"
      }
    }
  }
}
```

Claude Code supports OAuth: omit `headers` and you'll be prompted to authenticate in-browser on first tool call.

## Project-Scoped Config

Check a `.mcp.json` into the repo so teammates auto-inherit the right servers:

```json
{
  "mcpServers": {
    "postgres-dev": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/dev"]
    }
  }
}
```

Claude Code reads this on startup. Add `.mcp.json` to `.gitignore` only if it contains secrets; otherwise commit it.

## Programmatic Client — TypeScript

```ts
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

const transport = new StdioClientTransport({
  command: "npx",
  args: ["-y", "@modelcontextprotocol/server-github"],
  env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN! },
});

const client = new Client({ name: "my-app", version: "1.0.0" });
await client.connect(transport);

const tools = await client.listTools();
const result = await client.callTool({
  name: "create_issue",
  arguments: { repo: "owner/repo", title: "Bug", body: "..." },
});
```

## Programmatic Client — HTTP Transport

```ts
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

const transport = new StreamableHTTPClientTransport(
  new URL("https://mcp.example.com"),
  { requestInit: { headers: { Authorization: `Bearer ${token}` } } },
);
await client.connect(transport);
```

## Using MCP with the Claude Agent SDK

```ts
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic();

const response = await client.beta.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 4096,
  mcp_servers: [
    { type: "url", url: "https://mcp.linear.app/sse", name: "linear" },
  ],
  messages: [{ role: "user", content: "List my open Linear issues" }],
});
```

The SDK handles tool-use loop internally.

## Debugging Connection Failures

1. **Check stderr** — Claude Desktop logs MCP server stderr to `~/Library/Logs/Claude/mcp*.log`. Tail it while restarting the client
2. **Run the command manually** — if `npx -y @modelcontextprotocol/server-github` fails in your terminal, it'll fail for Claude
3. **Absolute paths** — `command: "node"` depends on PATH; prefer `command: "/usr/local/bin/node"` or `npx`
4. **Env var substitution** — Claude Desktop doesn't expand `${VAR}` in all fields; inline the value or use an env manager
5. **Tool list empty?** — the server connected but registered no tools. Check server logs for schema validation errors

## Best Practices

1. Prefer `npx -y` over local installs — updates flow automatically
2. Pin versions (`@modelcontextprotocol/server-github@1.2.3`) for production CI
3. Use project-scoped `.mcp.json` for team-shared servers, user-scoped for personal ones
4. Never put secrets in committed config — use env var interpolation or OS keychain
5. When a server misbehaves, start with MCP Inspector before blaming the client
