---
name: cost-quality-tradeoff
description: |
  Measure and optimize the cost/quality curve — which model, prompt, and settings give the best quality per dollar. Covers Pareto analysis, break-even thresholds, and when to spend more vs less.
  Use this skill when optimizing LLM spend, picking a default model for a feature, or deciding whether a premium model is worth it.
  Activate when: cost vs quality, model selection, eval cost, Pareto frontier, cheaper model, premium model tradeoff.
---

# Cost vs Quality Tradeoff

**Quality without cost context is half a decision. You need the Pareto frontier — for each quality bar, what's the cheapest config that hits it?**

## When to Use

- Choosing a default model for a new feature
- Reducing LLM spend on an existing feature
- Justifying (or not) an upgrade to a premium model
- Trading off prompt complexity, model size, and thinking budget

## The Pareto Frontier

Plot each candidate config (model × prompt × settings) on quality (y-axis) vs cost per request (x-axis). The frontier is the set of configs where no other config is both cheaper AND better.

Any config NOT on the frontier is dominated — always strictly worse than another option. Drop it.

```
  quality
   ↑
 1 |    *A (opus + thinking)
   |    *B (opus)
   |*G *D (sonnet + few-shot)
   |*F *C (sonnet)
 0 |*E (haiku)
   +---------------→ cost
```

Pareto: A, B, D, C, E. Dominated: F (worse than E at same cost), G (worse than D at same cost).

## Measurement

For each candidate, measure:

| Metric | Example |
|---|---|
| Input tokens / request | 2,500 |
| Output tokens / request | 400 |
| $ / request | $0.012 |
| Quality score | 0.87 |
| p95 latency | 1.8s |

```ts
const costPerRequest = (usage.input_tokens / 1e6) * inputRate +
                       (usage.output_tokens / 1e6) * outputRate +
                       (usage.cache_creation_input_tokens / 1e6) * cacheWriteRate +
                       (usage.cache_read_input_tokens / 1e6) * cacheReadRate;
```

Always include cache costs — they dominate on cached workloads.

## Common Configs to Compare

For any feature, try at least:

1. Haiku with concise prompt
2. Haiku with longer / few-shot prompt
3. Sonnet with concise prompt
4. Sonnet with few-shot + structured output
5. Sonnet with extended thinking
6. Opus with concise prompt
7. Opus with extended thinking

One of these usually sits on the frontier for your workload. Don't assume — measure.

## Prompt as a Lever

Before jumping to a bigger model, try prompt levers:

- Few-shot examples (2-5) often bump quality 5-15% at small cost
- Structured output (JSON schema) reduces parsing errors
- Chain-of-thought prompting helps reasoning without thinking tokens
- Better system prompt scoping (what's in vs out) improves accuracy

A better prompt on Haiku can beat a mediocre prompt on Sonnet — and cost 10× less.

## Break-Even Analysis

When considering an upgrade, compute when it pays off:

```
Cost increase per request: Δcost = new - old
Quality increase: Δquality = new - old
Value per quality point: V (estimated from business metrics)

Worth it if: Δquality × V > Δcost
```

Example: If every 1% quality gain increases user retention revenue by $0.003/request, and upgrading Haiku→Sonnet costs +$0.002/request for +5% quality:
- Value gain: 5 × $0.003 = $0.015
- Cost: $0.002
- Net: +$0.013 per request → upgrade.

## Tiered Routing

You don't have to pick one. Route by difficulty:

```ts
const difficulty = await classifyDifficulty(query);
const model = difficulty === "simple" ? "claude-haiku-4-5"
            : difficulty === "medium" ? "claude-sonnet-4-6"
            : "claude-opus-4-6";
```

Classification is a cheap Haiku call. Most queries are simple; you save money. Hard queries get the premium treatment.

Measure: does tiered routing actually improve your cost/quality position? Sometimes classification errors wipe out the gains.

## Latency as a Third Axis

Cost-quality isn't enough; latency matters too. Examples where it dominates:

- Chat UX needs first token < 2s → rules out unthinking Opus
- Voice agent needs full response < 500ms → forces Haiku
- Background summarization: latency doesn't matter, optimize cost/quality only

Report 3-tuples: (quality, cost, p95 latency). The frontier in 3D is smaller; choose by which axis has a constraint.

## Cache-Aware Selection

If you can cache 90% of your input:

- Effective input cost drops ~10×
- Long-context + cache often beats short-context + no cache at the same quality
- Larger models become more affordable per request

Decisions made without caching factored in are usually wrong. Re-measure with cache.

## Sample Budget

Don't eval each config on 10,000 items. Start small:

1. Shortlist with 50 items → narrow to 3 configs
2. Confirm with 200 items → pick winner
3. Validate in production with canary on 5% traffic → full rollout

Saves 10-100× on eval cost.

## Anti-Patterns

1. **Choosing the biggest model by default** — often dominated
2. **Ignoring cache costs** — skews the whole picture
3. **One-dimensional optimization** — quality-only misses cost blowouts
4. **Tiered routing without measuring** — classification errors can negate gains
5. **Per-product model choices** — you can't reuse infrastructure investment
6. **Skipping prompt levers** — jumping to bigger model before trying few-shot

## Best Practices

1. Measure cost AND quality AND latency; plot the Pareto frontier
2. Try prompt levers (few-shot, CoT, structure) before upsizing the model
3. Compute break-even: is the quality gain worth the cost delta?
4. Consider tiered routing; measure that it actually helps
5. Always factor in cache reads/writes — they can 10× swing the decision
6. Eval with small samples first (50-200); canary in prod before full rollout
7. Revisit choices quarterly — pricing and model quality move
