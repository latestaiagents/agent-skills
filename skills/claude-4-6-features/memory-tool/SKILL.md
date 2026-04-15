---
name: memory-tool
description: |
  Use Claude's Memory tool to give agents persistent cross-session memory stored in a client-side file directory. Covers setup, directory layout, reading/writing patterns, and when memory beats context-stuffing.
  Use this skill when building agents that need to remember across sessions (user preferences, project state, past decisions), or when moving from "stuff everything into context" to persistent memory.
  Activate when: Claude memory tool, agent memory, persistent memory, memory directory, cross-session state, long-term memory agent.
---

# Memory Tool

**The Memory tool lets Claude read and write files in a client-managed directory, giving agents persistent state across sessions without RAG infrastructure.**

## When to Use

- Agents that need to remember user preferences, project context, or past decisions
- Coding agents that accumulate knowledge about a codebase over sessions
- Support/assistant agents that build up a history per user
- Replacing "dump everything into system prompt" with structured persistent state

## Core Concept

The Memory tool is a server-side tool (provided by Anthropic) that Claude can call to read, write, and list files in a directory **you manage**. The API gives the tool calls; your app executes them against real storage (local fs, S3, DB-backed fs).

```ts
const response = await client.beta.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 4096,
  tools: [{ type: "memory_20250818", name: "memory" }],
  messages: [...],
});
```

## Implementing the Memory Backend

When Claude calls the memory tool, your app handles the file operations. Minimal implementation:

```ts
import fs from "fs/promises";
import path from "path";

const MEMORY_ROOT = "/var/app/memory";

async function handleMemoryCall(input: any, userId: string) {
  const userRoot = path.join(MEMORY_ROOT, userId);
  const safe = (p: string) => {
    const resolved = path.resolve(userRoot, p.replace(/^\//, ""));
    if (!resolved.startsWith(userRoot + path.sep)) throw new Error("Path escapes root");
    return resolved;
  };

  switch (input.command) {
    case "view": {
      const target = safe(input.path);
      const stat = await fs.stat(target).catch(() => null);
      if (!stat) return { error: "not found" };
      if (stat.isDirectory()) {
        const entries = await fs.readdir(target);
        return { content: entries.join("\n") };
      }
      return { content: await fs.readFile(target, "utf-8") };
    }
    case "create": {
      await fs.mkdir(path.dirname(safe(input.path)), { recursive: true });
      await fs.writeFile(safe(input.path), input.file_text);
      return { content: "created" };
    }
    case "str_replace": {
      const f = safe(input.path);
      const text = await fs.readFile(f, "utf-8");
      if (!text.includes(input.old_str)) return { error: "old_str not found" };
      await fs.writeFile(f, text.replace(input.old_str, input.new_str));
      return { content: "replaced" };
    }
    case "insert": {
      const f = safe(input.path);
      const lines = (await fs.readFile(f, "utf-8")).split("\n");
      lines.splice(input.insert_line, 0, input.insert_text);
      await fs.writeFile(f, lines.join("\n"));
      return { content: "inserted" };
    }
    case "delete": {
      await fs.rm(safe(input.path), { recursive: true, force: true });
      return { content: "deleted" };
    }
    case "rename": {
      await fs.rename(safe(input.old_path), safe(input.new_path));
      return { content: "renamed" };
    }
  }
}
```

Per-user roots are critical — never share memory across users.

## Tool-Use Loop

```ts
async function runWithMemory(userId: string, userMessage: string) {
  const messages = [{ role: "user", content: userMessage }];

  while (true) {
    const response = await client.beta.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 4096,
      tools: [{ type: "memory_20250818", name: "memory" }],
      messages,
    });

    messages.push({ role: "assistant", content: response.content });
    if (response.stop_reason !== "tool_use") return response;

    const toolResults = [];
    for (const block of response.content) {
      if (block.type === "tool_use" && block.name === "memory") {
        const result = await handleMemoryCall(block.input, userId);
        toolResults.push({
          type: "tool_result",
          tool_use_id: block.id,
          content: result.content ?? result.error,
          is_error: !!result.error,
        });
      }
    }
    messages.push({ role: "user", content: toolResults });
  }
}
```

## Suggested Directory Layout

Claude will structure the memory itself, but seeding a clean layout helps. Example for a coding agent:

```
/memory/
  user_profile.md          # who the user is, preferences
  current_project.md       # what we're working on
  decisions/
    2026-04-15-auth.md     # architectural decisions
  conventions.md           # code style rules learned
  blockers.md              # open issues
```

Prompt the agent with this layout in the system prompt so it uses consistent paths.

## System Prompt Pattern

```
You have persistent memory at /memory. Before answering, consult:
- /memory/user_profile.md
- /memory/current_project.md

After significant work, update memory:
- Save user preferences to /memory/user_profile.md
- Save decisions to /memory/decisions/YYYY-MM-DD-topic.md
- Keep /memory/current_project.md up to date with what we're doing

Use `view` to read, `create`/`str_replace` to write. Keep memory concise.
```

## Memory vs Context vs RAG

| Approach | Best for | Downside |
|---|---|---|
| **Memory tool** | Structured state the agent curates itself | Agent has to remember to read/write |
| **Large context** | One-shot reasoning on full corpus | Expensive per call |
| **RAG** | Huge static corpus | Brittle retrieval; no synthesis across docs |

Memory shines when **the agent** decides what's worth remembering, not a retrieval system.

## Versioning & Safety

- Store memory in a durable, versioned backend (git-backed fs works well)
- Snapshot before destructive operations (delete, rename)
- Audit-log every write
- Enforce size limits — agents can fill memory; cap per-user MBs
- Never execute anything from memory files (they can be indirectly user-controlled)

## Anti-Patterns

1. **Global memory** across users — huge privacy leak
2. **No size limit** — agent writes 10GB over time
3. **No path validation** — agent uses `../` or absolute paths
4. **Dumping conversation logs** into memory verbatim — grows unbounded
5. **Forgetting to prompt the agent** to consult memory — it won't know to look

## Best Practices

1. Per-user memory root, enforced with path sandboxing
2. Seed a clean directory layout in the system prompt
3. Cap memory size per user (e.g., 10 MB)
4. Snapshot/backup before destructive ops
5. Audit every operation
6. Periodically prompt the agent to compress/clean its own memory
