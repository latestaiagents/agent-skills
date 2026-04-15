---
name: regression-evals
description: |
  Set up continuous regression evals so model/prompt/tool changes don't silently break existing behavior. Covers gating thresholds, CI integration, statistical significance, and response to regressions.
  Use this skill when deploying prompts to production, gating model upgrades, or noticing "it worked yesterday" in AI features.
  Activate when: regression eval, eval CI, prompt regression, model upgrade gate, eval threshold, eval alert.
---

# Regression Evals

**Every prompt change, model upgrade, or tool tweak is a potential regression. Regression evals catch breakage before users do — if you gate deploys on them.**

## When to Use

- Any AI feature in production
- Before upgrading model versions
- Before merging prompt changes
- Weekly as a drift/decay check

## The Core Loop

```
1. Maintain a versioned eval dataset (see eval-dataset-design)
2. On every proposed change, run evals on baseline AND candidate
3. Compare per-stratum metrics with significance tests
4. Gate merge on: no stratum regresses beyond threshold
```

## Minimal Harness

```ts
interface EvalCase { id: string; input: any; expected: any; stratum: string }

async function runEvals(model: string, prompt: string, cases: EvalCase[]) {
  const results = [];
  for (const c of cases) {
    const output = await runModel(model, prompt, c.input);
    const score = await scoreOutput(c.expected, output); // 0-1
    results.push({ id: c.id, stratum: c.stratum, score });
  }
  return results;
}

const baseline = await runEvals("claude-sonnet-4-6", oldPrompt, cases);
const candidate = await runEvals("claude-sonnet-4-6", newPrompt, cases);

const regressed = compareByStratum(baseline, candidate, { alpha: 0.05 });
if (regressed.length) throw new Error(`Regressions in: ${regressed.join(", ")}`);
```

## Significance Testing

A 2% drop on 100 items is noise, not signal. Use bootstrap confidence intervals:

```py
import numpy as np
def bootstrap_ci(scores, n=1000, alpha=0.05):
    means = [np.mean(np.random.choice(scores, size=len(scores), replace=True)) for _ in range(n)]
    return np.percentile(means, [100*alpha/2, 100*(1-alpha/2)])

baseline_ci = bootstrap_ci(baseline_scores)
candidate_ci = bootstrap_ci(candidate_scores)

# If CIs don't overlap AND candidate lower, it's a real regression.
```

For pass/fail metrics, use McNemar's test on the paired outcomes. For scalars, paired bootstrap.

## Thresholds

Hard rules:

| Metric | Regression threshold | Action |
|---|---|---|
| Any stratum | ≥ 3% drop with p < 0.05 | Block merge |
| Aggregate | ≥ 5% drop with p < 0.05 | Block merge |
| Single catastrophic item | New case drops from pass→fail | Investigate, likely block |
| Variance | CI widens significantly | Investigate (noisier outputs) |

Soft rules:

- 1-3% drop: warning, review required
- Improvement on one stratum, regression on another: flag for human judgment

## CI Integration

```yaml
# .github/workflows/eval.yml
name: regression-evals
on: pull_request
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run evals:baseline  # fetch cached baseline scores
      - run: npm run evals:candidate
      - run: npm run evals:compare
      - uses: actions/upload-artifact@v4
        with:
          name: eval-report
          path: eval-report.html
```

Publish the report as a PR comment. Reviewers should see which strata changed and by how much.

## Gating Model Upgrades

When Anthropic ships a new model, don't just swap in production. Gate:

1. Run full eval on new model vs current model
2. Bucket regressions: acceptable, acceptable-with-prompt-tweak, hard-regression
3. For hard regressions, either stay on current model or fix before swapping
4. Canary new model to 5-10% of traffic; monitor live metrics for a week before full rollout

## Baseline Management

Your "baseline" needs to be a real, stored artifact — not "whatever main is today".

Store baselines in an artifact store keyed on (prompt version, model, dataset version):

```
evals/baselines/
  prompt-v42_claude-sonnet-4-6_dataset-v3.json
```

When you merge a change, UPDATE the baseline. New baseline is the reference for the next candidate.

## Handling Flaky Evals

Some items are nondeterministic even at temperature 0 (tools, time-dependent data). Options:

1. Mark items as flaky; exclude from regression calc; track separately
2. Run flaky items N times; use majority vote
3. Fix the underlying issue — usually a bad eval item, not a bad model

Don't let flakes erode your alert fidelity.

## Alert Fatigue

If every PR triggers an alert, reviewers stop reading. Tune:

- Raise the p-value threshold for borderline cases
- Classify small drifts as "review required" (not blocking)
- Group related strata to reduce notification count

Fewer, higher-quality alerts get acted on.

## Handling a Real Regression

When CI blocks the PR:

1. Read the report — which stratum dropped?
2. Look at specific failed items — reproduce locally
3. Is it the prompt, the tool, the model? Bisect if needed
4. Fix and re-run. If you can't fix, decide: accept the regression (with sign-off) or revert

Never:

- Lower the threshold to make it pass
- Remove the failing stratum
- Skip eval on "urgent" PRs — those are exactly when you need it most

## Anti-Patterns

1. **Running evals once, shipping forever** — drift happens
2. **No per-stratum breakdown** — aggregate hides localized regressions
3. **No significance testing** — you chase noise
4. **Manual compare** — human eyeballs miss small systematic drops
5. **Baseline updated silently** — you lose the ability to detect drift
6. **Lowering thresholds to make CI green** — defeats the purpose

## Best Practices

1. Run regression evals on every PR, gate merges
2. Report per-stratum metrics with confidence intervals
3. Store versioned baselines in an artifact store
4. Use paired bootstrap or McNemar's for significance
5. Canary model upgrades; don't swap in production cold
6. Publish eval reports as PR comments for visibility
7. Maintain flaky-item exclusion lists explicitly, don't hide them
