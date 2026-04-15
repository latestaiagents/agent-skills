---
name: eval-dataset-design
description: |
  Design eval datasets that actually measure model quality — coverage, difficulty distribution, labeling consistency, and avoiding contamination. Covers sourcing, stratification, label quality, and when to generate vs curate.
  Use this skill when building a new eval set, realizing your current evals don't catch regressions, or labeling is inconsistent.
  Activate when: eval dataset, benchmark, test set, eval coverage, label quality, synthetic eval, dataset design.
---

# Eval Dataset Design

**Your evals are only as good as the dataset they run on. Miss a user scenario and you'll never catch regressions on it.**

## When to Use

- Starting an eval program from zero
- Your evals pass but users still hit issues → coverage gap
- Labels are inconsistent across reviewers → quality problem
- Adding evals for a new feature or domain

## Dataset Properties Worth Optimizing

1. **Coverage** — representative of real user queries
2. **Difficulty distribution** — mix of easy/medium/hard, not all easy
3. **Label consistency** — two humans agree on the label
4. **Stability** — same inputs → same evaluable outputs over time
5. **Uncontaminated** — not in the model's training data

## Sourcing Inputs

Best to worst:

1. **Real user queries** (anonymized) — highest signal
2. **Synthetic queries generated from real templates** — fills gaps
3. **Adversarial queries** hand-crafted for known failure modes
4. **Existing benchmarks** — context, but often contaminated and dated

A good eval set mixes all four. Typical split: 60% real, 20% synthetic, 15% adversarial, 5% benchmark.

## Stratification

Split your dataset by categories that matter:

```yaml
dataset:
  categories:
    simple_qa: 100 samples        # easy, high-frequency
    multi_step_reasoning: 50       # medium
    ambiguous_queries: 30          # hard
    edge_cases: 20                 # adversarial
    rare_domains: 20               # coverage of long tail
```

Report metrics per stratum, not just the aggregate. A model can improve on average while regressing on edge cases — you'll only see it stratified.

## Labeling Quality

Two people label the same 50 items independently. Compute inter-annotator agreement:

```py
from sklearn.metrics import cohen_kappa_score
kappa = cohen_kappa_score(labeler_a, labeler_b)
```

Target:
- κ > 0.8: excellent, labels are reliable
- κ 0.6-0.8: good, some ambiguity
- κ < 0.6: **rewrite your labeling rubric** — humans can't agree, so neither can models

Resolve disagreements with a tiebreaker, then update the rubric based on what caused disagreement.

## Labeling Rubric

Write explicit guidelines with examples:

```markdown
### Label: helpful

**Definition**: Response addresses the user's question directly and accurately.

**Examples**:
- Query: "How do I loop in Python?" / Response: Shows `for` loop → YES
- Query: "How do I loop in Python?" / Response: General loop theory → NO (dodges the specific language)
- Query: "Fix this bug" / Response: Points out the bug + fix → YES
- Query: "Fix this bug" / Response: "I'll need more info" (bug is in the code) → NO
```

If two labelers disagree, add their disputed case as a rubric example.

## Contamination

If your eval is in the model's training data, scores are inflated. Check:

1. **Hash the query** and search public datasets / GitHub / web — common sources
2. **Ask the model** to complete the eval query's preamble — if it auto-completes with the expected answer, it's memorized
3. **Regenerate with paraphrasing** — rewrite queries so training data near-matches become mismatches

For production evals, rotate the dataset yearly and keep a private held-out set.

## Difficulty Calibration

Track difficulty via model pass rate:

- `< 30%` pass: too hard; models improve but you can't measure it
- `30-80%` pass: useful range
- `> 95%` pass: too easy; dataset has plateaued

Prune items that reach 100% for several consecutive model generations — they no longer discriminate.

## Synthetic Generation

When you need more coverage:

```ts
const prompt = `Generate 20 diverse user queries that a customer support bot might receive.
Cover: billing (5), technical issues (5), account access (5), general FAQ (5).
Vary wording: formal, casual, angry, confused.
Return JSON array.`;

const response = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 4000,
  messages: [{ role: "user", content: prompt }],
});
```

Then:

1. Human-review every synthetic query for realism
2. Label them the same way as real queries
3. Track whether synthetic vs real have different score profiles — a red flag if they diverge

## Size

- **Smoke test**: 20-50 items, run in CI
- **Regression set**: 200-500 items, run weekly
- **Full eval**: 1000-5000 items, run per major release
- **Beyond that**: sampling with stratification, not more volume

Quality > quantity. 200 well-labeled items beat 5000 noisy ones.

## Versioning

Treat datasets like code:

```
evals/
  customer_support/
    v1/
      dataset.jsonl
      rubric.md
      CHANGELOG.md
    v2/
      dataset.jsonl
      rubric.md
      CHANGELOG.md
```

Never silently edit. Version bumps communicate "scores before v2 are not comparable to scores after".

## Private Held-Out Set

Keep 100-200 items never published, never used for prompt iteration. Only for:

- Measuring generalization on unseen examples
- Catching overfitting to your public eval

Rotate a fraction yearly.

## Anti-Patterns

1. **Evals that only cover easy cases** — model passes your eval, fails in prod
2. **Single labeler** — no way to know if labels are noisy
3. **No versioning** — silent edits invalidate historical trends
4. **Overfit to eval** — you tuned prompts against the eval set; now it doesn't generalize
5. **All real or all synthetic** — synthetic misses distribution, real misses edge cases

## Best Practices

1. Mix real + synthetic + adversarial; stratify and report per category
2. Label with a written rubric; measure inter-annotator kappa
3. Check for training-data contamination; paraphrase suspicious queries
4. Track item-level difficulty; prune items that hit 100% pass
5. Version datasets like code; publish CHANGELOGs
6. Keep a private held-out set for generalization checks
7. Quality > quantity; 200 good labels beat 5000 rushed ones
