---
name: llm-as-judge
description: |
  Use an LLM as an evaluator for open-ended outputs — rubrics, pairwise comparison, calibration with human labels, bias mitigation. Covers when LLM-judge works, when it fails, and how to trust its scores.
  Use this skill when evaluating generative outputs at scale, building eval pipelines, or replacing expensive human review for non-critical judgments.
  Activate when: LLM as judge, LLM evaluator, automated evaluation, pairwise comparison, rubric evaluation, eval model.
---

# LLM-as-Judge

**Use a strong LLM to evaluate another LLM's output. Done right, it's fast, cheap, and correlates with human judgment. Done wrong, it's biased, inconsistent, and misleading.**

## When to Use

- Scaling eval beyond what humans can review
- Measuring open-ended outputs (summaries, code quality, helpfulness) where rule-based metrics fail
- Pairwise model comparison (A vs B on the same input)
- CI checks on agent outputs

## When NOT to Use

- High-stakes decisions (medical, legal) — need humans
- When the judge is the same model as the generator — biased toward its own style
- Very short outputs where a rule can decide — `exact_match` is cheaper
- Tasks the judge can't do itself — if it can't write good code, it can't judge code well

## Three Common Patterns

### 1. Rubric Scoring

Judge rates one output against explicit criteria on a 1-5 scale.

```ts
const prompt = `You are evaluating a response. Rate it 1-5 on each criterion.

<user_query>${query}</user_query>
<response>${response}</response>

Criteria:
- accuracy: factually correct?
- helpfulness: addresses what the user asked?
- conciseness: no unnecessary verbosity?

Return JSON: {"accuracy": N, "helpfulness": N, "conciseness": N, "reasoning": "..."}`;

const judgment = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 500,
  messages: [{ role: "user", content: prompt }],
});
```

Use a **stronger** model as judge than the one you're evaluating. Opus judges Sonnet; Sonnet judges Haiku.

### 2. Pairwise Comparison

Show two outputs, judge picks which is better. Most reliable pattern.

```ts
const prompt = `Compare two responses to the same query. Pick which is better overall.

<query>${query}</query>
<response_A>${responseA}</response_A>
<response_B>${responseB}</response_B>

Return JSON: {"winner": "A" | "B" | "tie", "reasoning": "..."}`;
```

To control for position bias, run each pair TWICE with order swapped. Average the judgments.

### 3. Reference-Based

Compare output to a gold-standard reference:

```ts
const prompt = `Is the generated answer equivalent to the reference answer?

<reference>${reference}</reference>
<generated>${generated}</generated>

"Equivalent" means factually consistent — wording can differ.
Return: {"equivalent": true|false, "reasoning": "..."}`;
```

Cheaper than rubric but requires good references.

## Known Biases

| Bias | Description | Mitigation |
|---|---|---|
| **Position bias** | Judge prefers first or second option | Randomize; run pairs twice with swapped order |
| **Length bias** | Judge prefers longer responses | Include "conciseness" in rubric; normalize by length |
| **Self-preference** | Judge prefers its own model's style | Use a DIFFERENT model family as judge |
| **Verbosity bias** | Judge prefers confident/flowery language | Rubric explicitly penalizes vagueness |
| **Format bias** | Prefers markdown/bullets over prose | Rubric targets content, not format |

Name the biases in your judge prompt — it reduces them: "Do not prefer longer responses; judge only on accuracy."

## Calibrating Against Humans

Don't trust judge scores in isolation. Calibrate:

1. Sample 50-200 outputs. Have humans label them.
2. Run the judge on the same set. Collect its scores.
3. Compute **Cohen's kappa** or **Pearson correlation** between human and judge.
4. If kappa > 0.6, the judge is reliable for this task. If < 0.4, rewrite the prompt.

```py
from sklearn.metrics import cohen_kappa_score
kappa = cohen_kappa_score(human_labels, judge_labels)
```

Re-calibrate quarterly or whenever you change judge prompts or models.

## Prompt Design for Judges

1. **Role first** — "You are a code review expert"
2. **Criteria explicit** — don't leave "quality" undefined
3. **Output format constrained** — JSON or a fixed label set
4. **Reasoning field included** — forces the model to justify, reduces careless judgments
5. **Examples** — 2-3 few-shots of correct judgments on your criteria

## Structured Output

Get judgments as JSON so you can aggregate:

```ts
const judgment = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 500,
  messages: [
    { role: "user", content: judgePrompt },
    { role: "assistant", content: "{" },
  ],
});
const parsed = JSON.parse("{" + judgment.content[0].text);
```

Prefilling `"{"` nudges valid JSON. Validate with a schema before aggregating.

## Aggregating at Scale

Per dataset:

- **Mean rubric score per criterion** — easy but loses variance
- **Win rate in pairwise** — e.g., "model B wins 62% of the time"
- **Score distribution** — detect regressions in the tail (a few catastrophic outputs even if mean is OK)

Report confidence intervals (bootstrap) — a 2% score gap on 100 samples is likely noise.

## Cost Management

Judging is expensive. Reduce cost:

1. **Sample** — judge 200 items, not 10,000
2. **Cheap judge first pass**, expensive judge for disagreements
3. **Cache judgments** — if input didn't change, reuse the last score
4. **Prompt caching** on the judge prompt (see `prompt-caching-ttl`)

## Anti-Patterns

1. **Judge = generator** — model evaluates itself; huge positive bias
2. **No calibration** — you're reporting numbers with no grounding
3. **No position-bias control** in pairwise
4. **Vague rubric** — "rate quality 1-5" — scores will be noisy
5. **Single-run judgments** — use multiple runs for critical evals; measure variance

## Best Practices

1. Use a stronger, different model family as judge
2. Pairwise comparison > rubric when you have two outputs to compare
3. Always control for position bias — swap and average
4. Calibrate against human labels; target Cohen's kappa > 0.6
5. Name biases in the prompt to mitigate them
6. Return structured JSON with a reasoning field
7. Report confidence intervals, not just means
8. Cache and sample to keep costs bounded
