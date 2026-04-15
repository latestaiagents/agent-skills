---
name: mcp-auth-oauth
description: |
  Implement OAuth 2.1 + PKCE authentication for remote MCP servers, including dynamic client registration, token refresh, and scope design. Covers the 2025 MCP auth spec that Claude Desktop, Claude Code, and ChatGPT use.
  Use this skill when building a remote MCP server that needs per-user auth, debugging OAuth flows for MCP, or migrating a bearer-token MCP server to OAuth.
  Activate when: MCP OAuth, remote MCP auth, MCP authorization, PKCE, dynamic client registration, MCP 401.
---

# MCP Auth — OAuth 2.1

**Remote MCP servers use OAuth 2.1 + PKCE with Dynamic Client Registration. Any spec-compliant MCP client — Claude Desktop, Claude Code, Cursor, ChatGPT Apps — can authenticate users without hardcoded client IDs.**

## When to Use

- Building a remote MCP server that serves multiple users
- Migrating a static-bearer-token server to per-user auth
- Debugging "401 Unauthorized" or "failed to authenticate" in an MCP client
- Adding scope-based permission to an MCP server

## The Flow (RFC 8414 + RFC 7591 + PKCE)

1. Client connects to MCP server → server returns `401` with `WWW-Authenticate` pointing to the authorization server metadata URL
2. Client fetches `/.well-known/oauth-authorization-server` → gets `authorization_endpoint`, `token_endpoint`, `registration_endpoint`
3. Client POSTs to `registration_endpoint` (RFC 7591) → receives a `client_id` dynamically
4. Client opens browser to `authorization_endpoint` with PKCE challenge → user approves
5. Client exchanges code at `token_endpoint` → receives access + refresh tokens
6. Client retries MCP request with `Authorization: Bearer <token>`

No pre-shared client ID. No manual app registration. The user just clicks "Allow" in a browser.

## Server-Side Requirements

### 1. Return discovery metadata

```ts
// GET /.well-known/oauth-authorization-server
app.get("/.well-known/oauth-authorization-server", (req, res) => {
  res.json({
    issuer: "https://mcp.example.com",
    authorization_endpoint: "https://mcp.example.com/oauth/authorize",
    token_endpoint: "https://mcp.example.com/oauth/token",
    registration_endpoint: "https://mcp.example.com/oauth/register",
    scopes_supported: ["read:issues", "write:issues"],
    response_types_supported: ["code"],
    grant_types_supported: ["authorization_code", "refresh_token"],
    code_challenge_methods_supported: ["S256"],
    token_endpoint_auth_methods_supported: ["none"], // public clients
  });
});
```

### 2. 401 with WWW-Authenticate

```ts
app.use("/mcp", (req, res, next) => {
  const token = req.headers.authorization?.replace("Bearer ", "");
  if (!token || !verifyJwt(token)) {
    res.setHeader(
      "WWW-Authenticate",
      `Bearer resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource"`,
    );
    return res.status(401).json({ error: "unauthorized" });
  }
  req.user = decodeJwt(token);
  next();
});
```

### 3. Dynamic Client Registration (RFC 7591)

```ts
app.post("/oauth/register", async (req, res) => {
  const { redirect_uris, client_name } = req.body;
  const clientId = `mcp_${crypto.randomUUID()}`;
  await db.clients.insert({ clientId, redirectUris: redirect_uris, name: client_name });
  res.json({
    client_id: clientId,
    redirect_uris,
    token_endpoint_auth_method: "none",
    grant_types: ["authorization_code", "refresh_token"],
  });
});
```

### 4. Authorization endpoint (PKCE S256)

```ts
app.get("/oauth/authorize", async (req, res) => {
  const { client_id, redirect_uri, code_challenge, code_challenge_method, state, scope } = req.query;
  if (code_challenge_method !== "S256") return res.status(400).send("S256 required");
  // render consent UI; on approval:
  const code = crypto.randomUUID();
  await db.codes.insert({ code, clientId: client_id, userId: req.session.userId, codeChallenge: code_challenge, scope });
  res.redirect(`${redirect_uri}?code=${code}&state=${state}`);
});
```

### 5. Token endpoint (PKCE verify)

```ts
app.post("/oauth/token", async (req, res) => {
  const { grant_type, code, code_verifier, refresh_token } = req.body;
  if (grant_type === "authorization_code") {
    const row = await db.codes.findAndDelete({ code });
    const hash = crypto.createHash("sha256").update(code_verifier).digest("base64url");
    if (hash !== row.codeChallenge) return res.status(400).json({ error: "invalid_grant" });
    return res.json(issueTokens(row.userId, row.scope));
  }
  if (grant_type === "refresh_token") {
    const userId = verifyRefresh(refresh_token);
    return res.json(issueTokens(userId));
  }
});
```

## Scope Design

Group by resource + action, not by tool:

```
read:issues        → list/get issues
write:issues       → create/update/close issues
read:repos         → list/get repos
admin:webhooks     → create/delete webhooks
```

Don't use `full_access` — clients can't reason about what they're granting.

Map scopes to MCP tools in your handler:

```ts
server.tool("create_issue", "...", schema, async (args, { authInfo }) => {
  if (!authInfo.scopes.includes("write:issues")) {
    throw new Error("Missing scope: write:issues. Reconnect with this scope enabled.");
  }
  // ...
});
```

## Token Lifetimes

- **Access token**: 1 hour (JWT, stateless)
- **Refresh token**: 30 days, rotating (one-time use, issue new one on each refresh)
- **Authorization code**: 60 seconds, one-time

## Client-Side (Reference Only)

Claude Desktop, Claude Code, and the Claude Agent SDK handle all of this automatically when you specify an `url` server without `headers`:

```json
{
  "mcpServers": {
    "linear": { "url": "https://mcp.linear.app/sse" }
  }
}
```

## Debugging

1. **Browser never opens**: check `WWW-Authenticate` header is present on 401
2. **"Client registration failed"**: your `/oauth/register` must accept `application/json` and not require auth
3. **"Invalid grant"**: PKCE verifier mismatch — the code was issued with a different challenge than the token request is using
4. **Scope escalation**: client asks for scopes you don't advertise in discovery → reject with `invalid_scope`
5. **Token expires mid-session**: client should refresh on 401; if your server returns 403 for expired tokens, fix it to return 401

## Best Practices

1. Always require PKCE S256 — reject `plain` and missing challenges
2. Rotate refresh tokens on every use; revoke entire chain on reuse detection
3. Bind tokens to the client_id that issued them — block cross-client token use
4. Expose a `/.well-known/oauth-protected-resource` metadata doc per MCP 2025 spec
5. Log every token issuance with user, client, scope, IP for audit
6. Never log tokens themselves; log JTI (JWT ID) instead
