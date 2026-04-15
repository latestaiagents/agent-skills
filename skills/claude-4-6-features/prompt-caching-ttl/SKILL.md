---
name: prompt-caching-ttl
description: |
  Use Claude's prompt caching with 5-minute and 1-hour TTLs to slash costs on repeated context — codebases, system prompts, long documents. Covers cache breakpoints, hit-rate optimization, and the common mistakes that silently disable caching.
  Use this skill when building apps with repeated large context, optimizing LLM spend, or debugging "why are my cache reads zero?"
  Activate when: prompt caching, cache_control, cache hit rate, 5 minute cache, 1 hour TTL cache, ephemeral cache, reduce Claude cost.
---

# Prompt Caching — 5min & 1h TTL

**Prompt caching reuses already-processed prefixes. Cache reads cost ~10% of fresh input. For apps with large repeated context, this is the single biggest lever on your bill.**

## When to Use

- Large system prompts reused across many requests
- Long documents/codebases with many follow-up questions
- Multi-turn conversations with growing history
- Tool/function definitions shared across sessions
- Any call where > 1024 tokens would be repeated (2048 for Haiku)

## Two TTLs

| TTL | Use case | Cost of cache write |
|---|---|---|
| **5 min (ephemeral)** | Conversation, active session, interactive tools | ~1.25× input |
| **1 hour** | System prompts, knowledge bases, codebases | ~2× input |

Cache reads are ~0.1× input cost regardless of TTL. The only difference is how long the cache persists and what the write costs.

Rule: use 1h when the cache lives across sessions or independent users; use 5min for in-session reuse.

## Basic Usage

```ts
const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 4096,
  system: [
    { type: "text", text: "You are a helpful assistant." },
    {
      type: "text",
      text: giantCodebase,
      cache_control: { type: "ephemeral", ttl: "1h" },
    },
  ],
  messages: [{ role: "user", content: "Where is auth handled?" }],
});

console.log(response.usage);
// { input_tokens: 120, cache_creation_input_tokens: 450000, cache_read_input_tokens: 0, output_tokens: 200 }
```

Next call within 1h:

```
{ input_tokens: 120, cache_creation_input_tokens: 0, cache_read_input_tokens: 450000, output_tokens: 180 }
```

## Cache Breakpoints

You can place up to **4 cache breakpoints** per request. Everything up to a breakpoint is cached as a prefix. Typical pattern:

```ts
const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 4096,
  system: [
    { type: "text", text: systemPrompt, cache_control: { type: "ephemeral", ttl: "1h" } },
  ],
  tools: [
    // all tool definitions
    { ...lastTool, cache_control: { type: "ephemeral", ttl: "1h" } }, // breakpoint at end of tools
  ],
  messages: [
    { role: "user", content: "Long context document..." },
    {
      role: "assistant",
      content: [{ type: "text", text: "Understood.", cache_control: { type: "ephemeral", ttl: "5m" } }],
    },
    { role: "user", content: "Current question." }, // NOT cached — this changes every call
  ],
});
```

Four breakpoints: system, tools, document, conversation history. The current turn stays uncached.

## Prefix Matching Rules

Cache hits require **byte-exact prefix match** up to the breakpoint. Common breakers:

1. **Timestamps / UUIDs in system prompt** — moves every call; kills the cache
2. **Reordering tools** — same tools in different order = cache miss
3. **Changing any earlier message** — even whitespace; cache invalidates
4. **Different `model`** — caches are per-model
5. **Different `max_tokens`** — doesn't break cache, but other param changes might

Keep everything before the breakpoint stable and deterministic.

## Minimum Cacheable Size

- Most models: 1024 tokens
- Haiku: 2048 tokens

Below the minimum, `cache_control` is ignored silently.

## Inspecting Hit Rate

```ts
const usage = response.usage;
const hitRate = usage.cache_read_input_tokens /
                (usage.cache_read_input_tokens + usage.cache_creation_input_tokens + usage.input_tokens);
console.log(`Cache hit rate: ${(hitRate * 100).toFixed(1)}%`);
```

Target: > 80% hit rate for production workloads on stable prefixes.

## Multi-Turn Pattern

For conversations, put a 5m breakpoint at the last assistant turn. Each new user turn extends the cache:

```ts
function addCacheBreakpoint(messages: any[]) {
  const copy = [...messages];
  const lastAssistant = [...copy].reverse().find((m) => m.role === "assistant");
  if (lastAssistant && Array.isArray(lastAssistant.content)) {
    const lastBlock = lastAssistant.content[lastAssistant.content.length - 1];
    if (lastBlock.type === "text") lastBlock.cache_control = { type: "ephemeral", ttl: "5m" };
  }
  return copy;
}
```

As the conversation grows, each call reuses the accumulated cache.

## Combining with 1M Context

1M context is expensive. Cache it:

```ts
const response = await client.messages.create(
  {
    model: "claude-sonnet-4-6",
    max_tokens: 4096,
    system: [
      { type: "text", text: giantCodebase, cache_control: { type: "ephemeral", ttl: "1h" } },
    ],
    messages: [{ role: "user", content: userQuestion }],
  },
  { headers: { "anthropic-beta": "context-1m-2025-08-07" } },
);
```

First call: pay ~2× input for cache write. Every subsequent question in the hour: ~10% input. Breaks even after ~2 questions.

## Anti-Patterns

1. **Dynamic content before breakpoint** — timestamp, request ID, rand in system prompt
2. **No breakpoint placed** — `cache_control` missing, nothing cached
3. **Cache smaller than minimum** — silently ignored
4. **Ignoring hit rate metrics** — you think caching is working but it's not
5. **Using 5m for cross-session knowledge bases** — re-paying cache write constantly

## Debugging Zero Hit Rate

1. Log `cache_creation_input_tokens` and `cache_read_input_tokens` every call
2. If `creation > 0` on every call: the prefix is changing. Diff consecutive requests byte by byte
3. If both are 0: prefix is below minimum tokens, or `cache_control` isn't being sent
4. If `creation` on first call but `read` is 0 on second within TTL: different model, different region, or parameter change broke the cache

## Best Practices

1. Place breakpoints at stable boundaries: end of system prompt, end of tools, end of large document
2. Use 1h TTL for knowledge bases reused across requests; 5m for conversations
3. Keep everything before breakpoints byte-stable — no timestamps, no UUIDs
4. Monitor hit rate as a first-class metric
5. For multi-turn, move a 5m breakpoint to the last assistant message each call
6. Combine with 1M context for big corpora — caching makes long-context affordable
