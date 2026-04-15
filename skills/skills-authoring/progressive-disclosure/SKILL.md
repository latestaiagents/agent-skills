---
name: progressive-disclosure
description: |
  Structure SKILL.md content so the model reads just enough — concise summary up front, progressively deeper detail, examples on demand. Covers section ordering, length budgets, when to split into multiple skills.
  Use this skill when writing or refactoring a skill body, one skill has grown too long, or a skill is wordy but not useful.
  Activate when: SKILL.md structure, skill content, skill too long, split skill, progressive disclosure, skill body.
---

# Progressive Disclosure in Skill Content

**A skill is consumed by a model with finite attention. Put the highest-signal content first, let depth come later. Length without structure is noise.**

## When to Use

- Writing a new SKILL.md body
- Refactoring a skill that's grown bloated
- Splitting a monster skill into focused ones
- Reviewing skills for publication quality

## The Shape of a Good Skill

Target ≈ 150-300 lines. Order sections from most-essential to least:

```
1. ---frontmatter---
2. One-line bolded summary
3. When to Use (bullet list)
4. Core concept (≤ 5 sentences)
5. Minimal example
6. Deeper patterns / variations
7. Anti-patterns
8. Best Practices (numbered list)
```

The model reads top-down. Every section should earn its place.

## Section-by-Section

### The Opening

```markdown
# Skill Title

**One sentence that captures the whole skill's value proposition.**
```

If the reader stops after this line, they still learned something.

### "When to Use"

A bullet list of 3-6 situations. Concrete, not abstract:

```markdown
## When to Use

- Your agent needs to remember user preferences across sessions
- You're replacing "stuff everything into system prompt" with structured state
- You're building coding agents that learn about a codebase over time
```

Bad:
- "For when you need memory." (abstract, unhelpful)

### Core Concept

5 sentences. Maximum. Explain the mental model:

```markdown
The Memory tool gives Claude a file system. Claude issues read/write ops;
your app executes them on real storage. Per-user roots isolate data.
Unlike RAG, the agent curates what's worth remembering. Unlike context-stuffing,
state persists without growing every message.
```

If you need 3 paragraphs, you probably have two concepts and should split.

### Minimal Example

The fewest lines of code that demonstrate the thing:

```ts
const response = await client.messages.create({
  model: "claude-sonnet-4-6",
  tools: [{ type: "memory_20250818", name: "memory" }],
  messages: [...],
});
```

No setup boilerplate the reader already knows. Start where the skill-specific code starts.

### Deeper Patterns

After the minimal, show variations and production patterns. Use subheadings:

```markdown
### Persistent Containers
### Streaming Results
### Error Handling
```

Each pattern: 1 short paragraph + 1 short code block. Resist the urge to write an essay.

### Anti-Patterns

Numbered list of things people get wrong:

```markdown
## Anti-Patterns

1. Running on a host with real data — one injection away from disaster
2. Using Computer Use when an API exists — 10-100× more expensive
3. No human-in-loop for destructive actions
```

This section is high-value. Readers who've hit these pain points become believers.

### Best Practices

Numbered list of recommendations. Each one actionable:

```markdown
## Best Practices

1. Use sub-agents for research and context isolation
2. Prompt sub-agents self-contained — they have no prior context
3. Require summarized output with word limits
```

Not "be careful", not "consider X" — imperatives.

## Length Budgets

| Skill type | Target length |
|---|---|
| Quick reference (git commands, API shapes) | 80-150 lines |
| Concept + workflow (most skills) | 150-300 lines |
| Deep dive (rare; prefer splitting) | 300-500 lines |

Past 500 lines you almost always have two or three skills masquerading as one.

## When to Split

Signals your skill should become multiple:

- Table of contents is needed to navigate it
- Readers only care about half of it at a time
- Two distinct audiences (developers vs architects)
- Two distinct workflows (create vs debug vs monitor)

Split along:

- **Lifecycle** — design / implement / test / monitor
- **Role** — backend / frontend / ops
- **Level** — beginner / advanced

## Links & Cross-References

Reference sibling skills by name, not duplicate their content:

```markdown
See the `mcp-security-sandboxing` skill for full threat model.
```

If your skill is re-explaining what another skill covers, trust the reader will load both.

## Code Example Discipline

1. **Real, runnable** — not pseudocode
2. **Minimal** — strip imports and setup the reader knows
3. **Single concept per block** — don't cram three features into one snippet
4. **Comments only when non-obvious** — don't narrate what the code does

## What NOT to Include

1. History of the feature ("in 2023...") — out of date immediately
2. Marketing prose ("powerful new capability") — reader doesn't need selling
3. Every edge case — pick the important 3
4. Obvious things the model knows — "import your dependencies"
5. Long tables of API params — link to the official docs for reference material

## The Scannability Test

Skim the skill in 10 seconds. Can you answer:

- What's this skill for?
- When do I use it?
- What's the minimal example?

If yes on all three, you're there. If no, the structure's off — reorder or trim.

## Anti-Patterns

1. **Wall of text** — no headings, no examples, no structure
2. **"Everything I know about X"** — ten concepts shoved into one skill
3. **Weak intro, strong ending** — model may stop reading before the good part
4. **Copy-pasted boilerplate** — same `await fetch(...)` noise in every block
5. **No concrete example** — abstract skills don't help the model or the user

## Best Practices

1. Lead with the headline; put detail behind it
2. Target 150-300 lines; split past 500
3. Real examples, minimal boilerplate
4. Numbered Anti-Patterns and Best Practices lists
5. Link to sibling skills instead of duplicating
6. Scannability-test: can a 10-second skim answer "what, when, example"?
