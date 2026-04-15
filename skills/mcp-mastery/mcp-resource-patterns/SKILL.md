---
name: mcp-resource-patterns
description: |
  Use MCP Resources correctly — the read-only, URI-addressable data primitive — and know when to pick resources, tools, or prompts. Covers templated URIs, subscriptions, and common mistakes.
  Use this skill when designing MCP servers that expose data, deciding between tool vs resource, or implementing resource subscriptions for live data.
  Activate when: MCP resource, resource template, resource vs tool, subscribe resource, MCP URI schema.
---

# MCP Resource Patterns

**Resources are read-only, URI-addressable data. Get them right and your server feels like a filesystem the agent can browse.**

## When to Use

- Deciding whether a read operation should be a tool or a resource
- Exposing hierarchical data (files, tickets, logs) to an agent
- Implementing live-updating data via subscriptions
- Designing URI schemes for a new MCP server

## Tool vs Resource vs Prompt

| Aspect | Tool | Resource | Prompt |
|---|---|---|---|
| Invocation | Agent calls it | Agent reads it | User selects it |
| Side effects | Allowed | No | No |
| Addressing | Name + args | URI | Name |
| Result | Agent-chosen | User/agent-attached | Template expansion |
| Example | `create_issue` | `github://issues/123` | `/summarize-pr` |

**Rule of thumb**: if the agent needs to decide *whether* to read it, it's a tool. If the user or agent needs to *attach* specific known data to context, it's a resource.

## Static Resources

```ts
server.resource(
  "changelog",
  "file://changelog.md",
  async () => ({
    contents: [{
      uri: "file://changelog.md",
      mimeType: "text/markdown",
      text: await fs.readFile("CHANGELOG.md", "utf-8"),
    }],
  }),
);
```

## Templated Resources

Use URI templates (RFC 6570) for addressable collections:

```ts
server.resource(
  "issue",
  "github://repos/{owner}/{repo}/issues/{number}",
  async (uri, { owner, repo, number }) => {
    const issue = await gh.issues.get({ owner, repo, issue_number: +number });
    return {
      contents: [{
        uri: uri.href,
        mimeType: "application/json",
        text: JSON.stringify(issue.data),
      }],
    };
  },
);
```

The client auto-discovers the template and can fetch any matching URI.

## Listing Resources

```ts
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
  resources: [
    { uri: "github://repos/acme/api/issues", name: "Open issues", mimeType: "application/json" },
    { uri: "github://repos/acme/api/prs", name: "Open PRs", mimeType: "application/json" },
  ],
}));
```

Keep the list short — this is what the user sees in the resource picker.

## Subscriptions (Live Data)

For data that changes (logs, metrics, ticket state), support subscriptions so the client re-reads on update:

```ts
server.resource(
  "live-log",
  "logs://app/{stream}",
  { subscribe: true },
  async (uri, { stream }) => ({ contents: [{ uri: uri.href, text: await tail(stream) }] }),
);

// When data changes:
logStream.on("update", (stream) => {
  server.sendResourceUpdated({ uri: `logs://app/${stream}` });
});
```

The client listens for `notifications/resources/updated` and re-fetches.

## URI Scheme Design

A good scheme is:

1. **Hierarchical** — `github://repos/{owner}/{repo}/issues/{number}`, not `github://issue?id=123&repo=...`
2. **Guessable** — if the agent knows `github://repos/acme/api/issues/5`, it can guess `.../issues/6`
3. **Unique per server** — prefix with your server's identity (`github://`, not `resource://`)
4. **MIME-tagged** — always include `mimeType` so the client renders correctly

Avoid opaque IDs in URIs when human-readable keys exist — `linear://issues/ENG-42` beats `linear://issues/a3f8c21d`.

## Binary Resources

```ts
return {
  contents: [{
    uri: uri.href,
    mimeType: "image/png",
    blob: base64Encode(pngBytes),
  }],
};
```

Clients that support vision models will attach PNGs directly. Text-only clients will skip binary resources.

## Pagination

MCP has no native pagination. Two conventions:

1. **Sub-resources**: `github://issues?page=2` as a distinct URI
2. **Cursor in body**: return JSON with `nextPageUri` and let the agent fetch it

Prefer (1) — clients cache per-URI.

## Common Mistakes

1. **Using tools for reads** — `get_issue` should be `github://issues/{number}` so the user can attach it as context
2. **Using resources for actions** — a resource read must not mutate state, ever
3. **No `mimeType`** — breaks rich clients and model understanding
4. **Templates with too many variables** — 5+ vars means you probably need a search tool instead
5. **Unstable URIs** — if `github://issues/123` points to different data tomorrow, caching breaks

## Best Practices

1. Design URIs as if they'll be bookmarked — stable, human-readable, hierarchical
2. Expose bulk endpoints as resource listings, detail endpoints as templated resources
3. Keep resource lists small (< 50 items); let templates handle the long tail
4. Use subscriptions sparingly — each one holds a connection open
5. Document your URI scheme in the server README — users browse before they call
