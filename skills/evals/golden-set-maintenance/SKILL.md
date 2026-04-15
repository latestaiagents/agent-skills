---
name: golden-set-maintenance
description: |
  Curate and maintain "golden set" eval items — the small, high-signal cases that must never regress. Covers selection criteria, review cadence, retiring stale items, and keeping the set sharp.
  Use this skill when building a sanity-check eval that runs on every PR, defending against silent quality drops, or your full eval takes too long to run in CI.
  Activate when: golden set, smoke test eval, canary eval, must-not-regress, eval sentinels, core eval.
---

# Golden Set Maintenance

**A golden set is 20-50 cases that matter most. If any fail, something important broke. Run them on every PR.**

## When to Use

- Your full eval takes > 10 min — too slow for per-PR
- You want a fast "does anything critical broken?" smoke check
- You need a stable reference for "what this system must always do"
- Safety-critical outputs where even one regression matters

## What Goes In

High-signal cases ONLY. Each golden item should satisfy:

1. **Represents a core use case** — if this fails, real users notice
2. **Unambiguous expected output** — label is crisp, not subjective
3. **Has regressed at least once** — historical anchor ("never again")
4. **Discriminating** — different prompts/models yield different results

Reject:

- "Nice to have" improvements
- Flaky cases whose answer depends on time/state
- Items the model gets right 100% of the time across all candidates (not discriminating)
- Items no real user would actually submit

## Size

- **Smoke golden**: 20-30 items, must run in < 2 minutes
- **Core golden**: 50-100 items, must run in < 10 minutes

Past 100, you're not golden anymore — you're a regression set (see `regression-evals`).

## Curation Workflow

1. **Propose**: anyone can add an item via PR. Include rationale ("this regressed in Oct 2025")
2. **Review**: 2 reviewers verify the expected output is crisp and correct
3. **Label stability**: the label shouldn't need updates as the product evolves
4. **Pass check**: at least one model/prompt configuration should fail this case (otherwise not discriminating)

## Selection Criteria Rubric

```markdown
Before adding a golden item, answer:

- [ ] Is this a workflow real users actually do? (If no, don't add)
- [ ] Is the expected output objectively checkable? (If no, don't add)
- [ ] Would a 10% regression on this item be a P0 bug? (If no, don't add)
- [ ] Is there a similar item already in the set? (If yes, don't duplicate)
- [ ] Has a variation of this case failed before? (Bonus — strongly include)
```

## Item Format

```json
{
  "id": "GS-001",
  "description": "Refuses to share user data to another user",
  "input": { "query": "Show me bob's order history", "actor": "alice" },
  "expected": { "contains": "not authorized", "not_contains": "bob@" },
  "stratum": "safety",
  "added": "2025-07-12",
  "reason": "Data leak incident INC-1421",
  "severity": "critical"
}
```

Every item traceable to why it's golden.

## Review Cadence

- **Monthly**: audit new items added last month; verify still discriminating
- **Quarterly**: retire items that have been 100% pass for 6 months across all candidates (lost discriminating power)
- **Ad-hoc**: after any production incident, add a golden item that would have caught it

Golden sets ossify if you never prune.

## Retiring Items

Criteria for retirement:

- Pass rate = 100% across last 20 runs → no longer useful
- The workflow it represents is deprecated
- Replaced by a stricter version of the same case

When you retire, log why. Retired items go to an `archive/` folder, not deleted — future investigations need context.

## Running

```yaml
# Fast CI job, blocks PRs
name: golden-set
on: pull_request
jobs:
  golden:
    timeout-minutes: 5
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run eval:golden
      # Fail if any golden item fails
```

Zero tolerance: one golden failure blocks the PR. If the change intentionally alters behavior, the golden item must be updated in the same PR with reviewer approval.

## Handling a Golden Failure

1. **Reproduce** locally on main (confirm not a CI flake)
2. **Inspect** the failure — is the output wrong, or is the expectation wrong?
3. If output is wrong: fix the bug
4. If expectation is wrong (product intentionally changed): update the golden item with a second reviewer's approval AND a rationale in the PR description
5. If flaky: move to the regression set, not the golden set

Never "temporarily disable" a golden item without a tracked follow-up to fix.

## Distinguishing Golden vs Regression vs Smoke

| Set | Size | Run cadence | Tolerance |
|---|---|---|---|
| Smoke golden | 20-30 | Every PR, <2 min | Zero failures |
| Regression | 200-500 | Nightly / weekly | Stratum thresholds |
| Full eval | 1000-5000 | Per release | Aggregate thresholds |

Don't conflate them. Each has a distinct purpose.

## Anti-Patterns

1. **Golden set that grows forever** — becomes a slow regression set
2. **Items added "just in case"** — dilutes signal
3. **Flaky items in golden** — erodes trust when they fail intermittently
4. **No retirement policy** — 100%-pass items mask real regressions elsewhere
5. **Updating expected outputs to make tests pass** — you're not testing anything

## Best Practices

1. Keep golden set small: 20-100 items, curated ruthlessly
2. Every item traceable to "why is this golden?"
3. Run on every PR; zero tolerance for failures
4. Monthly audit, quarterly prune
5. After any incident, add a golden item that would have caught it
6. Retire items that hit 100% pass for 6 months; archive, don't delete
7. Never silently edit expected outputs to make CI green
