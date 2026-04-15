---
name: mcp-server-authoring
description: |
  Build production-quality MCP (Model Context Protocol) servers that expose tools, resources, and prompts to AI clients like Claude Desktop, Claude Code, Cursor, and the Claude Agent SDK.
  Use this skill when the user wants to build an MCP server, expose internal tooling to an AI agent, wrap an API for agents, or publish a reusable MCP server to a registry.
  Activate when: MCP server, Model Context Protocol, expose tools to Claude, @modelcontextprotocol/sdk, mcp.json, stdio server, HTTP MCP server.
---

# MCP Server Authoring

**Build MCP servers in TypeScript or Python that any MCP-compatible client can consume.**

## When to Use

- You want to expose an internal API or tool to Claude Desktop / Claude Code / Cursor
- You're building a shareable MCP server (npm, PyPI, or Smithery registry)
- You need to wrap read-only data sources as MCP resources
- You want agents to invoke your service without custom glue code per client

## Core Model

An MCP server exposes three primitives:

| Primitive | Purpose | Side effects |
|---|---|---|
| **Tool** | Action the agent calls | May have side effects (writes, network calls) |
| **Resource** | Data the agent reads | Read-only, addressed by URI |
| **Prompt** | Reusable prompt template | No side effects; user-invocable |

Pick the right primitive — exposing read operations as tools wastes context when they should be resources.

## Quickstart — TypeScript (stdio)

```bash
npm init -y
npm i @modelcontextprotocol/sdk zod
npm i -D typescript tsx @types/node
```

```ts
// src/server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "weather-server",
  version: "1.0.0",
});

server.tool(
  "get_forecast",
  "Get a weather forecast for a location",
  {
    location: z.string().describe("City name or lat,lng"),
    days: z.number().int().min(1).max(14).default(3),
  },
  async ({ location, days }) => {
    const data = await fetchForecast(location, days);
    return {
      content: [{ type: "text", text: JSON.stringify(data) }],
    };
  },
);

server.resource(
  "station",
  "weather://stations/{id}",
  async (uri) => {
    const id = uri.pathname.split("/").pop()!;
    const station = await fetchStation(id);
    return {
      contents: [{ uri: uri.href, mimeType: "application/json", text: JSON.stringify(station) }],
    };
  },
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

Run it and register in the client's `mcp.json`:

```json
{
  "mcpServers": {
    "weather": {
      "command": "npx",
      "args": ["-y", "tsx", "src/server.ts"]
    }
  }
}
```

## Quickstart — Python (stdio)

```bash
uv add "mcp[cli]"
```

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather-server")

@mcp.tool()
async def get_forecast(location: str, days: int = 3) -> dict:
    """Get a weather forecast for a location."""
    return await fetch_forecast(location, days)

@mcp.resource("weather://stations/{id}")
async def station(id: str) -> dict:
    return await fetch_station(id)

if __name__ == "__main__":
    mcp.run()
```

## Tool Contract Rules

1. **Name**: snake_case, verb-led (`search_issues`, not `IssueSearcher`)
2. **Description**: one sentence, starts with a verb, mentions key args
3. **Schema**: use Zod / Pydantic with descriptive `.describe()` on every field
4. **Return shape**: always `{ content: [{ type: "text", text: ... }] }` — wrap JSON as text
5. **Errors**: throw — the SDK converts to `isError: true`. Include remediation in the message
6. **Idempotency**: write tools should accept an `idempotency_key` when possible

## Progressive Disclosure Pattern

Don't dump 40 tools on the client. Expose a small stable surface and let tools return follow-up handles:

```ts
server.tool("list_projects", "List accessible projects", {}, async () => {
  const projects = await api.listProjects();
  return {
    content: [{
      type: "text",
      text: `Found ${projects.length} projects. Use get_project with id to read one:\n` +
            projects.map(p => `- ${p.id}: ${p.name}`).join("\n"),
    }],
  };
});
```

## Testing

Use the MCP Inspector during development:

```bash
npx @modelcontextprotocol/inspector tsx src/server.ts
```

It gives you a UI to call every tool, inspect every resource, and replay prompts. Run it in CI against a mock backend to catch schema regressions.

## Publishing

- **npm**: publish the server as a `bin` entry so users run `npx your-mcp-server`
- **Smithery registry**: add `smithery.yaml` with config schema for one-click install
- **Private**: ship as a Docker image and point `command` at `docker run`

## Best Practices

1. Keep your tool surface ≤ 15 tools — more hurts model selection accuracy
2. Return text, not binary — for large payloads use resources with URIs
3. Stream long-running tools via `server.sendNotification` progress events
4. Version your server's `name` suffix (`github-v2`) when making breaking changes
5. Never read env vars for secrets at tool-call time — load at startup and fail fast
6. Log every tool invocation server-side — agents behave unpredictably, you'll need the audit trail
