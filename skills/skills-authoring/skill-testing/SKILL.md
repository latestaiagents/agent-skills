---
name: skill-testing
description: |
  Test skills for correct activation, content quality, and regression — both automated checks (frontmatter validity, lint) and manual verification (query-suite activation testing). Covers CI integration and how to catch skill regressions before users do.
  Use this skill when adding skills to a repo, setting up CI for a skill library, or debugging "the skill exists but doesn't work".
  Activate when: test skills, validate skills, skill CI, skill linting, skill activation test, skill regression.
---

# Skill Testing

**An untested skill is a skill that silently breaks. Build a test harness covering frontmatter, content, and activation, run it in CI, and you'll catch regressions long before users file bugs.**

## When to Use

- Setting up a new skills repository
- Adding skills to an existing repo without test coverage
- Debugging why a skill exists but never activates
- Reviewing PRs that add or modify skills

## Three Layers of Testing

1. **Static validation** — frontmatter parses, required fields present
2. **Content lint** — length, structure, code-block sanity
3. **Activation testing** — does the skill fire on realistic queries?

## Layer 1: Static Validation

```ts
import matter from "gray-matter";
import { readFileSync } from "fs";
import { globby } from "globby";

async function validateAll() {
  const files = await globby("skills/**/SKILL.md");
  const errors: string[] = [];

  for (const file of files) {
    const raw = readFileSync(file, "utf-8");
    let parsed;
    try {
      parsed = matter(raw);
    } catch (e) {
      errors.push(`${file}: invalid YAML`);
      continue;
    }
    const { data } = parsed;
    if (!data.name) errors.push(`${file}: missing name`);
    if (!data.description) errors.push(`${file}: missing description`);
    if (data.description && data.description.length < 50) {
      errors.push(`${file}: description too short (${data.description.length} chars)`);
    }
    // Name must match directory
    const dirName = file.split("/").slice(-2)[0];
    if (data.name && data.name !== dirName) {
      errors.push(`${file}: name '${data.name}' doesn't match directory '${dirName}'`);
    }
  }

  if (errors.length) {
    console.error(errors.join("\n"));
    process.exit(1);
  }
}
validateAll();
```

Run on every PR. Catches 80% of authoring mistakes.

## Layer 2: Content Lint

Check the body for common quality issues:

```ts
function lintBody(body: string, file: string) {
  const issues = [];
  if (body.length > 15_000) issues.push("body too long (>500 lines equiv)");
  if (!body.includes("## When to Use") && !body.includes("## When To Use")) {
    issues.push("missing 'When to Use' section");
  }
  if (!body.includes("## Best Practices")) {
    issues.push("missing 'Best Practices' section");
  }
  if (!/```/.test(body)) issues.push("no code example");
  // detect stale model IDs
  if (/claude-3[.-]/.test(body)) issues.push("stale model ID (claude-3-*)");
  // detect TODO markers
  if (/TODO|FIXME|XXX/.test(body)) issues.push("contains TODO marker");
  return issues.map((i) => `${file}: ${i}`);
}
```

Enforce via CI. Fail on errors; warn on style issues.

## Layer 3: Activation Testing

The highest-value and hardest layer. For each skill, maintain a test file:

```yaml
# skills/claude-4-6-features/memory-tool/tests.yaml
should_activate:
  - "how do I give my agent persistent memory?"
  - "I want Claude to remember user preferences across sessions"
  - "What's the memory tool in the Anthropic API?"
  - "My agent forgets everything after each conversation"
should_not_activate:
  - "How do I reduce my agent's token cost?"        # about cost, not memory
  - "My RAM is full, how do I fix it?"              # system memory, not the tool
  - "Add caching to my prompts"                     # different feature
```

Run these through your skill-activation engine and assert:

```ts
for (const query of tests.should_activate) {
  const activated = engine.selectSkills(query);
  assert(activated.includes(skillName), `${skillName} failed to activate on: ${query}`);
}
for (const query of tests.should_not_activate) {
  const activated = engine.selectSkills(query);
  assert(!activated.includes(skillName), `${skillName} wrongly activated on: ${query}`);
}
```

If you don't have a real activation engine handy, use Claude itself:

```ts
async function measureActivation(skill: Skill, query: string): Promise<boolean> {
  const prompt = `Given this skill:
Name: ${skill.name}
Description: ${skill.description}

Should this skill activate for the query: "${query}"?
Answer only "yes" or "no".`;

  const response = await client.messages.create({
    model: "claude-haiku-4-5",
    max_tokens: 10,
    messages: [{ role: "user", content: prompt }],
  });
  return response.content[0].text.trim().toLowerCase().startsWith("y");
}
```

Cheap, reproducible, catches bad descriptions.

## Layer 3b: Precision/Recall Scoreboard

Aggregate results across all skills:

```ts
interface Result {
  skill: string;
  truePositive: number;  // should activate + did
  falsePositive: number; // shouldn't activate + did
  falseNegative: number; // should activate + didn't
}

function score(results: Result[]) {
  for (const r of results) {
    const precision = r.truePositive / (r.truePositive + r.falsePositive || 1);
    const recall = r.truePositive / (r.truePositive + r.falseNegative || 1);
    console.log(`${r.skill}: P=${precision.toFixed(2)} R=${recall.toFixed(2)}`);
  }
}
```

Target: P ≥ 0.85, R ≥ 0.90. Lower P = over-activation; lower R = under-activation.

## Regression Testing

When you change a skill's description, re-run the activation suite:

```bash
npm run test:skills -- --skill memory-tool
```

Any query that previously activated and now doesn't (or vice versa) is flagged for review.

## Content Examples Testing

If your skill shows runnable code, compile-check it:

```ts
function extractCodeBlocks(body: string, lang: "ts" | "py") {
  const re = new RegExp("```" + lang + "\\n([\\s\\S]*?)```", "g");
  const blocks = [];
  let m;
  while ((m = re.exec(body))) blocks.push(m[1]);
  return blocks;
}
```

Then for each TS block, pipe through `tsc --noEmit` on a tmp file. For Python, `py_compile`. Won't run tests, but catches syntax errors.

## CI Integration

```yaml
# .github/workflows/skills.yml
name: skills
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run validate:skills
      - run: npm run lint:skills
      - run: npm run test:activation -- --changed  # only changed skills
```

Gate merges on all three.

## Smoke Testing Published Skills

After deploy, run a scheduled job that exercises a known query per skill:

```ts
for (const skill of allSkills) {
  const query = skill.tests.should_activate[0];
  const activated = await engine.selectSkills(query);
  if (!activated.includes(skill.name)) await alert(`Skill ${skill.name} not activating in prod`);
}
```

Catches deployment/config drift.

## Anti-Patterns

1. **No CI** — regressions ship to users
2. **Only frontmatter validation** — catches typos, not activation quality
3. **Activation tests without negative cases** — you miss over-activation
4. **Hand-testing** — doesn't scale past 20 skills
5. **Ignoring score drops** — one-off dips become normalized

## Best Practices

1. Three layers: static validation, content lint, activation tests
2. Per-skill `tests.yaml` with both positive and negative queries
3. Target precision ≥ 0.85, recall ≥ 0.90; track in a scoreboard
4. Run on every PR; gate merges
5. Scheduled smoke tests in prod to catch drift
6. When scoring drops, diff the skill description first — it's almost always the cause
