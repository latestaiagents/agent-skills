---
name: skill-frontmatter
description: |
  Write the YAML frontmatter for a SKILL.md file so it activates reliably — name, description, and activation keywords that the model matches against. Covers length, tone, and the most common frontmatter mistakes.
  Use this skill when authoring a new skill, fixing a skill that isn't auto-activating, or reviewing skills for publication.
  Activate when: SKILL.md frontmatter, skill description, skill activation, skill YAML, write a skill, author a skill.
---

# Skill Frontmatter

**The frontmatter is what the model reads to decide whether to load your skill. Get it wrong and the skill never fires.**

## When to Use

- Authoring a new SKILL.md
- Debugging "my skill isn't activating"
- Reviewing a PR that adds a skill

## Required Fields

```yaml
---
name: kebab-case-name
description: |
  Sentence 1: what the skill does.
  Sentence 2: when to use it (specific trigger conditions).
  Sentence 3: activation keywords (phrases users say).
---
```

That's it. `name` and `description`. Nothing else is required by most clients.

## The `name` Field

- **Kebab-case** — `merge-conflict-surgeon`, not `MergeConflictSurgeon`
- **Matches directory name** — directory holding the `SKILL.md` must have the same name
- **Verb-led or role-led**:
  - Action: `resolve-merge-conflict`
  - Role: `owasp-auditor`
  - Topic: `mcp-server-authoring`
- **Unique** across your skill set — two skills with the same name collide

## The `description` Field

This is THE activation signal. Model reads it and matches against the user's message.

### Three-sentence structure

1. **What** — one sentence on the skill's purpose (action-first)
2. **When** — specific trigger conditions
3. **Keywords** — phrases users actually say

```yaml
description: |
  Systematic merge conflict resolution with context analysis and safe resolution.
  Use this skill when the user has merge conflicts, needs help resolving conflicting changes, or is stuck on a rebase.
  Activate when: merge conflict, conflict resolution, git merge failed, rebase conflict, CONFLICT markers.
```

### Length

- Minimum: 2 sentences. Shorter = under-specified; misses activations
- Sweet spot: 3-5 sentences, ~60-150 words
- Maximum: don't exceed ~200 words; longer dilutes the signal

### Voice

- Second-person or imperative
- Specific, not abstract: "when user is debugging a null pointer in Java" beats "helps with bugs"
- Name the tools/frameworks — `tsc`, `Next.js`, `LangGraph` etc. Users and models match on these

## Writing Activation Keywords

Keywords are the easiest win. Think about what the user might literally type:

**Bad:**
> Activate when: development, coding, software.

Too vague. Matches everything → matches nothing well.

**Good:**
> Activate when: git merge failed, CONFLICT markers, resolve conflicts, rebase conflict, auto-merge failed.

Specific phrases, variants, error strings users actually see.

**Include:**

- Domain jargon (`prompt caching`, `OAuth 2.1`, `RAG`)
- Error messages (`null pointer`, `CORS error`)
- Tool names (`@modelcontextprotocol/sdk`, `LangChain`)
- Task verbs (`deploy`, `debug`, `refactor`)

## Common Mistakes

1. **Vague description** — "helps with code" doesn't match anything specifically
2. **Only describing WHAT, not WHEN** — missing trigger conditions
3. **Keywords no user would type** — "Activate when: utilities, helpers" 
4. **Name ≠ directory** — `name: foo` but directory is `bar` → skill may not load
5. **Multiline string without `|`** — YAML parses as a single collapsed line, loses formatting
6. **Typos in YAML** — wrong indentation, missing `---`, unquoted colons
7. **Duplicate descriptions** — 5 sibling skills with identical descriptions collide

## Validating YAML

Every CI run should parse your frontmatter:

```bash
# Quick check
find skills -name SKILL.md -exec sh -c \
  'head -n 20 "$1" | grep -A 20 "^---$" > /dev/null || echo "BAD: $1"' _ {} \;
```

Or a proper script:

```js
import matter from "gray-matter";
import { readFileSync } from "fs";
const { data } = matter(readFileSync(path, "utf-8"));
if (!data.name || !data.description) throw new Error(`Missing fields: ${path}`);
if (data.description.length < 50) console.warn(`Short description: ${path}`);
```

## Worked Example — Refining a Frontmatter

### Draft 1 (bad)

```yaml
name: test-generator
description: Generates tests.
```

Too short. No activation signal. Model never picks it.

### Draft 2 (better)

```yaml
name: test-generator
description: Generate unit tests for JavaScript and TypeScript code.
```

Better, but still missing WHEN and keywords.

### Draft 3 (good)

```yaml
name: test-generator
description: |
  Generate unit and integration tests for JavaScript/TypeScript code using Jest or Vitest.
  Use this skill when the user asks for tests, has added untested code, or needs to improve coverage.
  Activate when: write tests, add tests, test coverage, generate test cases, Jest, Vitest, missing tests.
---
```

Now the model has enough signal to activate confidently.

## Best Practices

1. Write the description from the USER'S angle — what they'd say, not what you'd call your skill
2. Include domain keywords users actually type (errors, tool names, jargon)
3. Keep total length under 200 words; concentrate signal
4. Validate YAML in CI — typos waste hours of debugging
5. Re-read after writing: if you can't tell when this skill should fire, neither can the model
6. Audit for duplicate descriptions across related skills — they should differentiate
