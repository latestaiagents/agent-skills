---
name: code-execution
description: |
  Use Claude's Code Execution tool to run Python in a sandboxed environment as part of a response — for calculation, data analysis, chart generation, and verification. Covers enabling, file upload, persistence across turns, and limitations.
  Use this skill when building features that need Claude to actually run code (not just write it), such as data analysis, math verification, or chart creation.
  Activate when: Claude code execution, Python sandbox, run code tool, data analysis agent, code interpreter, code_execution_20250522.
---

# Code Execution Tool

**The Code Execution tool runs Python in an Anthropic-hosted sandbox as part of a model response. Use it when you need the model to actually compute, not just describe.**

## When to Use

- Data analysis and transformation (CSV, JSON, parquet)
- Chart and plot generation (matplotlib, plotly)
- Math verification — let the model check its own work
- Running unit tests the model just wrote
- Any task where "the answer is whatever the code prints"

## Enabling

```ts
const response = await client.beta.messages.create(
  {
    model: "claude-sonnet-4-6",
    max_tokens: 4096,
    tools: [{ type: "code_execution_20250522", name: "code_execution" }],
    messages: [{ role: "user", content: "Analyze the attached CSV and plot monthly revenue." }],
  },
  { headers: { "anthropic-beta": "code-execution-2025-05-22" } },
);
```

No tool-use loop to manage — execution happens server-side and results flow back in the response.

## What's in the Sandbox

- Python 3.12+
- Preinstalled: pandas, numpy, matplotlib, scipy, scikit-learn, pillow, requests, beautifulsoup4, and more
- Ephemeral filesystem (cleared between calls unless using container persistence — see below)
- No network access by default (some betas allow it)
- CPU + RAM limits (conservative — don't train models here)

## File Upload

Upload files with the Files API, then reference them in the message:

```ts
const file = await client.beta.files.upload({ file: fs.createReadStream("sales.csv") });

const response = await client.beta.messages.create(
  {
    model: "claude-sonnet-4-6",
    max_tokens: 4096,
    tools: [{ type: "code_execution_20250522", name: "code_execution" }],
    messages: [
      {
        role: "user",
        content: [
          { type: "container_upload", file_id: file.id },
          { type: "text", text: "Compute YoY growth from sales.csv." },
        ],
      },
    ],
  },
  { headers: { "anthropic-beta": "code-execution-2025-05-22,files-api-2025-04-14" } },
);
```

The file appears at `/mnt/user-data/` inside the sandbox.

## Reading Execution Results

```ts
for (const block of response.content) {
  if (block.type === "code_execution_tool_result") {
    const result = block.content;
    console.log("stdout:", result.stdout);
    console.log("stderr:", result.stderr);
    if (result.return_code !== 0) console.log("FAILED");
    for (const file of result.files ?? []) {
      // file is a generated image/data file with file_id; download via Files API
    }
  }
}
```

## Container Persistence

Re-use the sandbox across calls by passing the container ID:

```ts
// First call creates a container
const first = await client.beta.messages.create({ /* ... */ });
const containerId = first.container?.id;

// Second call re-uses it — pip installs, written files, variables all persist
const second = await client.beta.messages.create({
  /* ... */
  container: containerId,
  messages: [
    ...firstMessages,
    { role: "user", content: "Now compute the 7-day rolling avg on that same DataFrame." },
  ],
});
```

Containers auto-expire after inactivity. Use persistence for multi-turn data analysis; skip it for one-shot.

## Chart Output

matplotlib figures are returned as image files the Files API can serve:

```py
import matplotlib.pyplot as plt
plt.plot([1,2,3], [4,5,6])
plt.savefig("/tmp/out.png")
```

The tool result includes a `file_id`. Download and render:

```ts
const bytes = await client.beta.files.content(fileId);
await fs.writeFile("out.png", bytes);
```

## Pairing with Extended Thinking

For analysis tasks, enable thinking so the model plans before coding:

```ts
const response = await client.beta.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 16_000,
  thinking: { type: "enabled", budget_tokens: 8000 },
  tools: [{ type: "code_execution_20250522", name: "code_execution" }],
  messages: [...],
});
```

Enable interleaved thinking (see the `extended-thinking` skill) so it can reflect on code output before the next step.

## Limitations

- No persistent installed packages across new containers — use `pip install` at start
- No unrestricted network by default — fetch external data beforehand
- File system wiped on container expiry
- Execution time-limited per call (typically 60-120s)
- Not suitable for long-running training or web scraping

## Security

The sandbox is isolated — user data doesn't leak between containers. Your risk is that the **model** might:

- Write your uploaded data to logs you later read back
- Print secrets included in messages to stdout

Scrub outputs before persisting (see `mcp-security-sandboxing` for redaction patterns).

## Anti-Patterns

1. **Using it for code writing** when no execution is needed — waste of server resources
2. **Uploading huge files** (> 100 MB) — slow and often unnecessary; sample first
3. **Not persisting the container** for multi-turn analysis — pays setup cost each time
4. **Ignoring stderr** — silent failures yield wrong answers

## Best Practices

1. Enable for analysis, math, tests, plots. Don't enable for pure text tasks
2. Use container persistence for multi-turn analysis
3. Pair with extended thinking for complex analysis
4. Always check `return_code` — model can confidently report results from failed code
5. Retrieve generated plots via Files API; don't re-ask the model to describe them
6. Limit file upload size; pre-sample large datasets client-side
