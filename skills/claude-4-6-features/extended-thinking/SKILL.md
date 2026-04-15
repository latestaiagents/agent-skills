---
name: extended-thinking
description: |
  Use Claude's extended thinking (reasoning) mode effectively — budget tokens, interleaved thinking with tool use, when it helps, when it wastes tokens, and how to inspect the thinking trace.
  Use this skill when building reasoning-heavy features (math, code generation, multi-step planning), debugging why a model is shallow on hard problems, or deciding whether to enable thinking.
  Activate when: extended thinking, thinking tokens, budget_tokens, reasoning mode, interleaved thinking, thinking blocks.
---

# Extended Thinking

**Extended thinking gives the model a scratchpad before the final answer. Pay for reasoning tokens, get deeper answers. Use it surgically, not everywhere.**

## When to Use

- Complex reasoning: math, proofs, multi-step logic
- Code generation where the model needs to plan before writing
- Agent planning: deciding which of many tools to call and in what order
- Debugging subtle issues: model can "think through" root causes
- Writing where structure and coherence matter more than speed

## When NOT to Use

- Simple classification, extraction, or formatting — pure overhead
- Latency-sensitive paths — thinking adds 2-30 seconds
- Small prompts where the model gets it right zero-shot anyway
- High-volume batch tasks on a tight cost budget

## Enabling Thinking

```ts
const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 16_000,
  thinking: {
    type: "enabled",
    budget_tokens: 10_000,
  },
  messages: [{ role: "user", content: "Prove that every prime > 3 is of the form 6k±1." }],
});
```

`budget_tokens` is the max thinking tokens. Model may use fewer. `max_tokens` must be > `budget_tokens` (thinking counts toward the total).

## Budget Sizing

| Task | Typical budget |
|---|---|
| Short multi-step reasoning | 2,000-5,000 |
| Code generation with planning | 5,000-10,000 |
| Complex math/proofs | 10,000-32,000 |
| Deep agent planning | 10,000-20,000 |
| Research synthesis | 16,000-32,000 |

Start at 5,000 and measure. Bigger budget ≠ better answers past a point.

## Inspecting the Thinking Trace

```ts
for (const block of response.content) {
  if (block.type === "thinking") {
    console.log("REASONING:", block.thinking);
  } else if (block.type === "text") {
    console.log("ANSWER:", block.text);
  }
}
```

The thinking block reveals the model's reasoning. Useful for:

- Debugging why the model chose a wrong answer
- Surfacing rationale to power users (e.g., "show reasoning" toggle)
- Auditing agent decisions

**Do not** feed thinking blocks back to the user as-is in production — they're not polished prose. And do not modify them before passing back in multi-turn (signature validation will fail).

## Interleaved Thinking with Tools

With the `interleaved-thinking-2025-05-14` beta, the model thinks between tool calls — reasoning about each tool result before picking the next:

```ts
const response = await client.messages.create(
  {
    model: "claude-sonnet-4-6",
    max_tokens: 16_000,
    thinking: { type: "enabled", budget_tokens: 10_000 },
    tools: [searchTool, fetchTool, summarizeTool],
    messages: [{ role: "user", content: "Research X and write a brief." }],
  },
  { headers: { "anthropic-beta": "interleaved-thinking-2025-05-14" } },
);
```

Without interleaved thinking, the model only thinks once at the start. With it, the model can reassess after every tool result — critical for agents that operate under uncertainty.

## Multi-Turn Conversations

When continuing a conversation that included thinking, pass the assistant's full message back unchanged (including thinking blocks):

```ts
messages.push({ role: "assistant", content: response.content });
messages.push({ role: "user", content: "Great. Now prove the converse." });

const next = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 16_000,
  thinking: { type: "enabled", budget_tokens: 10_000 },
  messages,
});
```

Thinking blocks carry signatures that the API validates. Reordering or editing them breaks the request.

## Cost

Thinking tokens are billed as output tokens. A call with 10K thinking + 2K answer costs 12K output tokens.

Rough rule: thinking doubles-to-triples the cost of a reasoning-heavy call. Confirm it's worth it by A/B testing against no-thinking.

## Streaming

```ts
const stream = client.messages.stream({
  model: "claude-sonnet-4-6",
  max_tokens: 16_000,
  thinking: { type: "enabled", budget_tokens: 10_000 },
  messages: [...],
});

for await (const event of stream) {
  if (event.type === "content_block_delta" && event.delta.type === "thinking_delta") {
    // show a "thinking..." spinner or subtle text
  } else if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    process.stdout.write(event.delta.text);
  }
}
```

In UX, show a distinct "thinking" indicator, then switch to streaming the answer.

## Anti-Patterns

1. **Thinking on every request** — massive cost increase with no quality gain for easy tasks
2. **Tiny budget on hard tasks** — 1000 tokens isn't enough for real reasoning; model truncates
3. **Modifying thinking blocks** between turns — breaks signature; API rejects
4. **Showing raw thinking to end users** in production — messy, not polished

## Best Practices

1. Gate thinking on task complexity — classify the query first, enable thinking only for hard ones
2. Start budget at 5K; measure quality and cost; adjust
3. Enable interleaved thinking for any agent with > 3 tools
4. Pass thinking blocks back unmodified in multi-turn
5. Stream the response so users see progress during long thinks
6. Log thinking tokens used per request — detect when the model is consistently maxing out budget
