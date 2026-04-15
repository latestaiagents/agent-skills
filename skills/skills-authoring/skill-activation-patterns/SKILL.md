---
name: skill-activation-patterns
description: |
  Design skills that fire at the right moment — neither over-eager (noise) nor under-eager (silent). Covers activation specificity, trigger phrases, disambiguation between overlapping skills, and debugging activation.
  Use this skill when multiple skills could fire on the same query, a skill never fires, or a skill fires too often.
  Activate when: skill won't activate, skill over-activates, overlapping skills, skill triggers, skill selection, skill disambiguation.
---

# Skill Activation Patterns

**A skill that fires at the wrong time is as useless as one that never fires. Design activation like you'd design a cron expression: specific, testable, boundaried.**

## When to Use

- Debugging skills that don't fire
- Two skills claiming the same trigger
- Skills firing on unrelated queries
- Designing a new suite with multiple related skills

## The Activation Signal Stack

The model matches your skill against user queries using:

1. **Name** — weak signal
2. **Description first sentence** — strong signal, this is the "what"
3. **"Use this skill when" line** — strongest signal, the "when"
4. **"Activate when" keyword list** — pattern match against query phrases

Invest most in (3) and (4).

## Four Activation Patterns

### 1. Error-Driven

Skill fires when user pastes or describes an error:

```yaml
description: |
  Decode Java stack traces and suggest root causes.
  Use this skill when user shares a stack trace, exception, or runtime error in Java.
  Activate when: Exception, NullPointerException, stack trace, Caused by, Java error, runtime crash.
```

Include the literal tokens that appear in errors. Users paste them verbatim.

### 2. Intent-Driven

Skill fires when user expresses a goal:

```yaml
description: |
  Refactor code to extract methods, rename, and improve structure without changing behavior.
  Use this skill when user asks to refactor, clean up, or restructure existing code.
  Activate when: refactor, clean up, extract method, rename, restructure, DRY up, improve this code.
```

Match imperative verbs and goal phrases.

### 3. Tool/Technology-Driven

Skill fires when the user mentions a specific tool:

```yaml
description: |
  Configure Next.js App Router — server components, server actions, middleware, caching.
  Use this skill when the user is working with Next.js App Router, specifically app/ directory routes.
  Activate when: Next.js, App Router, server component, server action, app/ directory, use server.
```

Name the technology and its key concepts; matches work well.

### 4. Workflow-Driven

Skill fires during a known phase of work:

```yaml
description: |
  Pre-release checklist for a production deploy — changelog, version bump, migration safety, smoke tests.
  Use this skill when user is about to deploy to production, cut a release, or tag a version.
  Activate when: deploy to prod, release, version bump, production release, pre-release check.
```

Workflow skills need to be clearly scoped to that phase or they fire constantly.

## Disambiguating Overlapping Skills

If two skills could both fire on "help me write a test", they compete. Differentiate in the description itself.

### Before (ambiguous)

- Skill A: "Generate tests for JavaScript code."
- Skill B: "Generate tests for React components."

On "write a test for MyComponent", both fire. Model picks randomly.

### After (disambiguated)

- Skill A: "Generate tests for JavaScript code. Use when code is non-React (Node, utilities, backend). Activate: test, unit test. NOT for React components — use react-test-generator instead."
- Skill B: "Generate tests for React components with Testing Library. Use for JSX/TSX component tests. Activate: React test, component test, Testing Library. NOT for backend — use js-test-generator."

The **"NOT for..."** line is a powerful routing hint.

## Over-Activation

Symptom: skill fires on nearly every query. Cause: description too broad.

### Too broad

```yaml
description: |
  Help with coding tasks.
  Activate when: code, programming, development.
```

### Fix — narrow the WHEN

```yaml
description: |
  Help refactoring existing code for readability — NOT for new code generation, bug fixing, or debugging.
  Use this skill when user specifically asks to refactor, clean up, or restructure.
  Activate when: refactor, clean up, extract method, DRY up, improve readability.
```

"NOT for X" sentences are explicit guards. Use them.

## Under-Activation

Symptom: skill has clear use case but never fires. Cause: description misses the user's vocabulary.

### Fix — expand keywords

Audit real queries users send. Add their exact phrasings:

```yaml
Activate when: 
  - merge conflict, resolve conflicts  # jargon
  - CONFLICT markers, <<<<<<< HEAD     # literal tokens
  - git says there's a conflict        # natural phrasing
  - merge failed, can't merge          # error paraphrases
  - rebase stopped, rebase conflict    # related workflows
```

Mix jargon, error strings, and natural phrasings.

## Negative Triggers

Some patterns explicitly should NOT activate the skill. Call them out:

```yaml
description: |
  Write SQL migrations safely.
  Use for: ALTER TABLE, new columns with defaults, adding indexes on prod tables.
  DO NOT use for: read-only queries (use sql-query-helper), ORM model changes, new tables from scratch.
  Activate when: ALTER TABLE, add column, migration, safe migration, Postgres lock, online DDL.
```

Explicit negatives reduce competition with nearby skills.

## Testing Activation

See the `skill-testing` companion skill. Briefly:

1. Compile a list of 10+ realistic user queries that SHOULD activate
2. Compile 10+ queries that SHOULD NOT
3. Run against your skill loader; measure precision & recall
4. Iterate description until both are high

## Common Failure Modes

1. **Generic verbs** — `help`, `handle`, `manage`. Match too broadly
2. **Missing error strings** — users paste exceptions; you don't list them
3. **Skill name matches a common word** — `tests`, `bugs`, `config`
4. **Duplicated descriptions** across related skills — model sees ties, picks randomly
5. **No WHEN clause** — description is all WHAT, zero specificity on triggers

## Best Practices

1. Write the description with real user queries in mind — paste examples in a notebook next to it
2. Include both jargon AND natural phrasing in keywords
3. For each related skill, include an explicit "NOT for X" guard
4. Test with a query suite — measure precision and recall
5. Iterate: bad activation is the #1 reason a skill feels "broken"
6. When in doubt, narrow — an under-firing skill is fixable; an over-firing skill erodes trust
