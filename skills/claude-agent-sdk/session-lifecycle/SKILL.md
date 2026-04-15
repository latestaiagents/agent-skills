---
name: session-lifecycle
description: |
  Manage Claude Agent SDK / Managed Agents sessions — creation, resumption, compaction, forking, and termination. Covers when to start fresh, when to resume, and how to handle context window pressure in long sessions.
  Use this skill when building multi-turn agents, debugging "my agent forgot the earlier context", or designing session retention policies.
  Activate when: agent session, session resume, conversation compaction, session fork, context window overflow, session lifecycle.
---

# Session Lifecycle

**A session is a durable conversation thread between user and agent. Manage it well and you avoid blown context windows, lost state, and confused users.**

## When to Use

- Designing multi-turn agent UX
- Debugging context-loss or context-overflow
- Choosing when to fork vs resume vs start fresh
- Setting retention policies for session storage

## Session States

```
[idle] → [running] → [idle]        # user sends → agent responds → waits
[idle] → [compacting] → [idle]     # background context reduction
[idle] → [archived]                # user/TTL-triggered
```

Sessions are long-lived. A single session can span hours to weeks with compaction keeping context in budget.

## Creating a Session

```ts
// SDK
const sessionId = crypto.randomUUID();
for await (const msg of query({ prompt: firstMessage, options: { sessionId } })) { /* ... */ }

// Managed
const session = await client.beta.sessions.create({ agent_id, metadata: { user_id } });
```

Always attach your own identifiers in metadata — user ID, tenant, feature flag — so you can filter and audit later.

## Resuming

```ts
// SDK
for await (const msg of query({ prompt: followUp, options: { resume: sessionId } })) { /* ... */ }

// Managed
await client.beta.sessions.messages.create({ session_id, content: followUp });
```

Resume when:

- Same user, same topic — preserve context
- Debugging an earlier decision — replay state
- Continuing a long-running workflow

## When to Start Fresh

Don't resume when:

- Topic changed completely (user pivoted to a new project)
- Session has stale or incorrect state you can't prompt-correct
- User logged in as a different identity
- Context is so full that compaction would lose critical recent info

Starting fresh is free. Resuming a bad session is expensive.

## Compaction

When context approaches the model's limit, older messages are compressed into a summary. Two flavors:

1. **Auto-compaction** (Managed default, SDK opt-in): system summarizes older turns
2. **Manual compaction**: your app triggers it, optionally with custom instructions

Manual example (SDK):

```ts
await compactSession({
  sessionId,
  instruction: "Preserve every architectural decision and all file paths mentioned.",
});
```

Compaction is lossy. If something matters, pin it to memory (see the `memory-tool` skill) or re-inject it in the next user message.

## Forking

Sometimes you want to explore a branch without polluting the main session:

```ts
const fork = await client.beta.sessions.fork({
  source_session_id: session.id,
  metadata: { purpose: "hypothesis-check" },
});
```

Both sessions share history up to the fork point, then diverge. Useful for:

- "What-if" exploration (try an alternative plan)
- A/B testing prompts on identical context
- Creating a clean branch for a new sub-task

## Archiving & Retention

```ts
await client.beta.sessions.archive({ session_id });
// Archived sessions are retained but not resumable without un-archive
```

Set retention policies:

- **User-facing chats**: retain 30-90 days, archive thereafter
- **Support tickets**: retain for full compliance window (often 7 years)
- **Debug/internal sessions**: retain 7 days, hard-delete after

Storage adds up at scale. Default retention may be longer than you need.

## Handling Overflow

If compaction still can't fit context (rare but possible on huge sessions):

1. Compact aggressively with narrow instruction: "Keep only the last 3 user decisions"
2. Fork and start a summary-carrying child session
3. Move static context to a cached prompt (see `prompt-caching-ttl`)
4. Offload to memory tool so the agent re-fetches only what's needed

## Session Metadata Patterns

```ts
metadata: {
  user_id: "u-42",
  tenant_id: "t-3",
  thread_type: "support",
  topic: "billing",
  started_at: "2026-04-15T10:30:00Z",
  feature_flag: "beta-v2",
}
```

This becomes your query surface. Use it for:

- Dashboards: sessions by tenant, topic, feature
- Audits: every session a user had
- Migration: identify sessions on old config to re-run after changes

## Multi-Device Sessions

User starts on mobile, continues on web. Two options:

1. **Single session ID per logical conversation** — both devices POST to same session (Managed handles concurrency)
2. **Device-scoped sessions** — separate sessions with shared metadata; sync only to user-visible state

Prefer (1) unless you have strong reason — Managed handles concurrent writes with last-writer-wins; simple enough.

## Anti-Patterns

1. **Infinite resume** — never starting fresh; session becomes bloated and incoherent
2. **No compaction strategy** — blown context window errors in prod
3. **No metadata** — can't find sessions after the fact
4. **Archiving without retention policy** — storage bill grows unbounded
5. **Cross-user session sharing** — privacy leak

## Best Practices

1. Start new sessions when the topic changes; resume for true continuations
2. Attach rich metadata at session creation — your future self needs it
3. Use manual compaction with custom instructions for session-critical info
4. Fork for "what-if" exploration; never branch by cloning messages manually
5. Define retention policy per use case; enforce via scheduled archiver
6. Move stable context to cache or memory; leave sessions for the conversation itself
