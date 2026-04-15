---
name: mcp-tool-design
description: |
  Design MCP tool schemas, names, and descriptions that AI agents actually pick correctly and use without hand-holding. Covers the anti-patterns that make agents loop, pick wrong tools, or hallucinate arguments.
  Use this skill when designing or reviewing MCP tools, debugging "the agent isn't using my tool", or pruning a bloated tool surface.
  Activate when: MCP tool design, tool description, agent picks wrong tool, too many tools, tool schema, tool naming.
---

# MCP Tool Design

**A well-designed tool is invoked correctly by the agent on the first try. A bad one causes loops, wrong-tool selection, or hallucinated arguments.**

## When to Use

- Adding a new tool to an MCP server
- Debugging "the model never picks this tool" or "the model picks the wrong tool"
- Reviewing a PR that adds tools
- Cleaning up a server with 30+ tools

## The Three Levers

Agents pick tools based on **name**, **description**, and **parameter schema** — in that order of signal strength. Every design choice should strengthen at least one.

## Naming Rules

| Good | Bad | Why |
|---|---|---|
| `search_issues` | `issueSearcher` | snake_case, verb-led |
| `get_user_by_email` | `user_lookup` | specific over vague |
| `create_pr_comment` | `comment` | namespaced by object |
| `list_repos` | `repos` | action is explicit |

**Rule**: `<verb>_<object>[_<qualifier>]`. If two tools could answer the same query, one has the wrong name.

## Description Rules

Descriptions are what the model reads most carefully. Budget ~1-3 sentences:

```
<Verb-led action>. <When to use it / when NOT to use it>. <Any gotchas>.
```

### Example — weak vs strong

**Weak:**
> Creates an issue.

**Strong:**
> Create a GitHub issue in the specified repo. Use this for new bug reports or feature requests. Do NOT use to comment on an existing issue — use `create_issue_comment` for that. Title is required; body supports markdown.

The "when NOT to use" line is the highest-leverage sentence you can write — it routes disambiguation without the agent needing to enumerate all tools.

## Parameter Schema Rules

1. **Describe every field** — `.describe()` in Zod, `Field(description=...)` in Pydantic. Undescribed fields get hallucinated values
2. **Use enums for closed sets** — `priority: z.enum(["low", "med", "high"])` beats `priority: z.string()`
3. **Default the optional** — if 80% of calls use `limit=50`, set the default; don't make the agent guess
4. **Prefer primitives at the top level** — nested objects increase hallucination rates
5. **Avoid freeform "options" bags** — split into discrete flags

### Example — good schema

```ts
server.tool(
  "search_issues",
  "Search GitHub issues across a repo. Returns up to 50 issues matching the query. " +
  "Use for finding issues by keyword or label. For a specific issue by number, use `get_issue` instead.",
  {
    repo: z.string().describe("owner/name format, e.g. 'anthropic/claude-code'"),
    query: z.string().describe("Full-text search query, GitHub search syntax supported"),
    state: z.enum(["open", "closed", "all"]).default("open"),
    labels: z.array(z.string()).optional().describe("Filter by label names (AND semantics)"),
    limit: z.number().int().min(1).max(100).default(25),
  },
  async (args) => { /* ... */ },
);
```

## Return Shape Rules

The model sees the tool result as text. Structure matters:

1. **Lead with a summary line** — "Found 3 open issues matching 'auth bug':"
2. **Include IDs** so the model can chain calls
3. **Truncate aggressively** — don't return 200 rows when 10 will do; include `"... 190 more, use offset=10"` hint
4. **Never return raw HTML** — strip or convert to markdown

```ts
return {
  content: [{
    type: "text",
    text: [
      `Found ${results.length} issues matching "${query}":`,
      ...results.slice(0, 10).map(r => `- #${r.number} ${r.title} (${r.state})`),
      results.length > 10 ? `... ${results.length - 10} more. Narrow query or paginate with offset.` : "",
    ].filter(Boolean).join("\n"),
  }],
};
```

## Surface Size

Tool selection accuracy degrades past ~15 tools per connected server set. If you have 40 CRUD operations, collapse them:

- **Before:** `create_issue, update_issue, delete_issue, close_issue, reopen_issue, assign_issue, ...` (12 tools)
- **After:** `issue_action(action: "create"|"update"|"close"|"reopen", ...)` (1 tool with discriminated schema)

Only collapse if the sub-actions share 80%+ of their schema. Otherwise the union becomes a mess.

## Anti-Patterns

1. **The God Tool** — `execute(query: string)` that dispatches everything. Model can't pick wisely; hallucinates queries
2. **Duplicate tools** — `search`, `find`, `lookup`, `query` all doing similar things
3. **Hidden required args** — `options: object` where some keys are required; agent skips them
4. **Silent truncation** — returning 10 of 500 results without saying so; agent assumes it got everything
5. **Tool-name collisions across servers** — if user has 3 servers each with a `search` tool, the model gets confused. Prefix: `linear_search`, `github_search`

## Validation Checklist

Before shipping a tool:

- [ ] Name is verb-led and unambiguous
- [ ] Description has ≥1 "when NOT to use" sentence if there's a nearby tool
- [ ] Every parameter has `.describe()`
- [ ] Closed-set fields use enums
- [ ] Defaults cover the common case
- [ ] Return includes IDs for chaining
- [ ] Return summarizes + truncates large lists
- [ ] Tested with MCP Inspector that the model can invoke it zero-shot

## Best Practices

1. Read your descriptions aloud — if you stumble, the model will too
2. Pilot with one real conversation before adding 10 tools at once
3. When a user reports "the agent won't use my tool", check description first, schema second, name third
4. Log unresolved tool calls (agent tried to call a non-existent tool) — it reveals the tool it WANTED
