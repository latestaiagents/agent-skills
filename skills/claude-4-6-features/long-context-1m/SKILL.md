---
name: long-context-1m
description: |
  Use Claude's 1M-token context window effectively — when to use it, how to structure inputs for recall, how to price it, and how to combine with prompt caching to keep it affordable.
  Use this skill when building apps that feed large codebases, long documents, or entire conversation histories to Claude, or when weighing 1M context vs RAG.
  Activate when: 1M context, long context, big context window, context vs RAG, Claude 1 million tokens, context-beta header.
---

# 1M Context Window

**Claude Opus 4.6 and Sonnet 4.6 support 1M token context with the `context-1m-2025-08-07` beta header. Use it well or burn money for nothing.**

## When to Use

- You have a codebase, book, log bundle, or document set that fits in 1M tokens
- You need cross-document reasoning that chunked RAG can't deliver
- You're deciding between 1M-context vs a RAG pipeline
- You want to cache a giant system prompt / knowledge base across requests

## Enabling 1M Context

```ts
import Anthropic from "@anthropic-ai/sdk";
const client = new Anthropic();

const response = await client.messages.create(
  {
    model: "claude-sonnet-4-6",
    max_tokens: 4096,
    messages: [{ role: "user", content: giantDocument + "\n\nSummarize." }],
  },
  { headers: { "anthropic-beta": "context-1m-2025-08-07" } },
);
```

Without the beta header, requests over 200K tokens will error.

## Pricing Tiers

Long context is priced differently above 200K input tokens. Check your provider's current rates; as a rule of thumb input above 200K costs ~2× the base rate. Output price is unchanged.

**Rule**: if you're only going to use 200K, don't enable 1M. Only pay for long-context pricing when you actually need > 200K.

## 1M Context vs RAG

| When 1M context wins | When RAG wins |
|---|---|
| Cross-document synthesis | Fresh data that updates hourly |
| Full-codebase refactoring | Unbounded corpus (> 1M tokens) |
| Holistic code review | Per-user personal data (privacy isolation) |
| Single-shot analysis | Many cheap lookups on small queries |
| Exploration where you don't know what's relevant | Known query patterns |

Hybrid: RAG retrieves the top 500K tokens; stuff those into 1M context. Best of both.

## Structuring Long Inputs for Recall

Claude's long-context recall is strong but not uniform. Tips:

1. **Put the instruction at the END** — "Given the above, answer X" recalls better than instruction-then-context
2. **Section headers with XML tags** — `<document index="1" title="...">...</document>` — the model indexes on these
3. **Repeat critical instructions** — once at top, once at bottom
4. **Avoid homogeneous blobs** — chunk with delimiters; recall degrades in undifferentiated text

```ts
const prompt = `
You will analyze the codebase below, then answer questions.

<codebase>
<file path="src/auth.ts">...</file>
<file path="src/db.ts">...</file>
...
</codebase>

Given the codebase above, answer: <question>How does auth flow work?</question>
`;
```

## Combine with Prompt Caching

1M context is expensive per call. If you're asking multiple questions against the same corpus, cache it:

```ts
const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 4096,
  system: [
    { type: "text", text: "You are a code reviewer." },
    { type: "text", text: giantCodebase, cache_control: { type: "ephemeral", ttl: "1h" } },
  ],
  messages: [{ role: "user", content: "What's the auth flow?" }],
});
```

First call: full cost. Subsequent calls in the TTL window: ~10% of input cost for the cached portion. See the `prompt-caching-ttl` skill.

## Latency

1M input takes longer to process — TTFT can be 10-30s for a full context. Mitigate:

- **Stream** the response so the user sees output early
- **Extended thinking** on top only if you need reasoning (adds latency)
- **Cache** so subsequent calls skip the bulk of input processing

## Token Accounting

Use the token counter before sending:

```ts
const { input_tokens } = await client.messages.countTokens({
  model: "claude-sonnet-4-6",
  messages: [{ role: "user", content: text }],
});
if (input_tokens > 1_000_000) throw new Error("Over context limit");
```

Budget with a 5% safety margin — actual tokenization varies slightly.

## Anti-Patterns

1. **Always-on 1M**: enabling the beta even for small requests. Wastes money via long-context pricing
2. **Dump-and-pray**: no structure in the input. Model can't find what it needs
3. **Instruction at top only**: with 900K of content below, the instruction gets diluted
4. **No caching for repeat queries**: every question against the same document pays full price

## Best Practices

1. Check the token count before enabling 1M beta
2. Cache the static portion (the document/codebase) with 1h TTL
3. Put actionable instructions at the end, not the start
4. Use XML tags to structure sections
5. Stream responses — users don't want to wait 30s staring at nothing
6. For queries on a large corpus with many questions, prefer cached-1M over RAG
