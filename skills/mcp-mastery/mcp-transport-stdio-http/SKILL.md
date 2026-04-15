---
name: mcp-transport-stdio-http
description: |
  Choose between MCP transports — stdio for local processes, Streamable HTTP for remote servers, SSE for legacy — and implement each correctly with reconnection, backpressure, and session handling.
  Use this skill when deciding transport for a new MCP server, migrating SSE to Streamable HTTP, or debugging transport-level issues (connection drops, buffering, session loss).
  Activate when: MCP transport, stdio vs HTTP, Streamable HTTP, MCP SSE, MCP reconnection, MCP session.
---

# MCP Transports — stdio vs Streamable HTTP

**Pick the right transport and the rest of your server design follows.**

## When to Use

- Starting a new MCP server and choosing a transport
- Migrating from deprecated SSE transport to Streamable HTTP
- Debugging "connection dropped" or "session expired" errors
- Scaling a remote MCP server horizontally

## Transport Matrix

| Transport | Use when | Pros | Cons |
|---|---|---|---|
| **stdio** | Local tools, CLI integrations, single-user | Zero network, trusted process, simplest | Only local; one client per server process |
| **Streamable HTTP** | Remote multi-user servers | Bidirectional, resumable, load-balanceable | More complex; requires session layer |
| **SSE** (legacy) | Existing deployments | — | **Deprecated**; migrate to Streamable HTTP |

## stdio Transport

Server reads from stdin, writes JSON-RPC to stdout, logs to stderr. Client spawns the process.

```ts
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
const transport = new StdioServerTransport();
await server.connect(transport);
```

Critical rule: **never write non-JSON to stdout**. `console.log` breaks the protocol. Use `console.error` for logs.

```ts
// BAD
console.log("Server started"); // corrupts protocol stream

// GOOD
console.error("Server started");
```

## Streamable HTTP Transport (current spec)

Single HTTP endpoint handles all methods. Server pushes messages via SSE on long-lived GET, receives messages via POST, supports resumable sessions via `Mcp-Session-Id` header.

### Server (TypeScript)

```ts
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const app = express();
app.use(express.json());

const transports = new Map<string, StreamableHTTPServerTransport>();

app.all("/mcp", async (req, res) => {
  const sessionId = req.headers["mcp-session-id"] as string | undefined;
  let transport = sessionId ? transports.get(sessionId) : undefined;

  if (!transport) {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => crypto.randomUUID(),
      onsessioninitialized: (id) => transports.set(id, transport!),
    });
    await server.connect(transport);
  }

  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

### Client

```ts
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";
const transport = new StreamableHTTPClientTransport(new URL("https://mcp.example.com/mcp"));
await client.connect(transport);
```

## Session Lifecycle

1. Client POSTs `initialize` without session header
2. Server generates session ID, returns it in `Mcp-Session-Id` response header
3. Client includes `Mcp-Session-Id` on every subsequent request
4. Server reconnects the request to the same session state
5. Either side can terminate with `DELETE /mcp` + session header

## Resumability

Long-lived SSE stream may drop (proxy timeouts, NAT rebinding). Client reconnects with `Last-Event-ID` and the server replays missed messages from a per-session buffer:

```ts
new StreamableHTTPServerTransport({
  sessionIdGenerator: () => crypto.randomUUID(),
  enableJsonResponse: false,
  eventStore: new RedisEventStore({ ttlSeconds: 3600 }), // custom
});
```

If you skip the event store, reconnects lose in-flight tool results. Fine for dev, bad for production.

## Horizontal Scaling

Multi-instance deployments must share session state. Options:

- **Sticky sessions** at the load balancer (simplest)
- **Shared Redis** for `transports` map and event store
- **Session affinity via header routing** (e.g., hash `Mcp-Session-Id` to a pod)

Never round-robin with in-memory session state — clients will randomly land on the wrong pod and see "session not found".

## Backpressure

Streamable HTTP lets the server push many messages to the client. If the client is slow:

- Buffer in memory with a high-water mark (e.g., 1000 messages)
- On overflow, send a `progress/cancelled` notification and drop the request
- Emit metrics on buffer depth to alert before overflow

## Migrating SSE → Streamable HTTP

Old SSE used two endpoints (`/sse` for events, `/messages` for POST). The new transport unifies them at one endpoint:

```
POST /mcp  → request
GET  /mcp  → open SSE stream for server→client
DELETE /mcp → end session
```

Migration: run both in parallel for a release, redirect SSE clients to the new endpoint, then remove old routes. The MCP SDK's `StreamableHTTPClientTransport` auto-negotiates.

## Debugging

1. **stdio: "not valid JSON"** — you're writing to stdout somewhere other than the SDK. Audit all `console.log`, `print`, debug libs
2. **HTTP 400 "missing session id"** — client is initialized but stripped the header somewhere (proxy, fetch wrapper)
3. **Reconnects lose state** — you have no event store configured
4. **CPU spikes**: SSE stream writes aren't being flushed; wrap in `res.flush()` or use `X-Accel-Buffering: no`

## Best Practices

1. Default to stdio for personal/CLI tools — far simpler
2. Default to Streamable HTTP for remote servers — SSE is deprecated
3. Always configure an event store for production HTTP servers
4. Log session lifecycle events (init, destroy, reconnect) with session IDs
5. Set sensible session TTLs (1-4 hours) and clean up idle sessions
6. Put an idle-connection proxy tunable (`proxy_read_timeout` ≥ 60s) for SSE
