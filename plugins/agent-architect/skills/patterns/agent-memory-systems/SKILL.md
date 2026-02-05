---
name: agent-memory-systems
description: |
  Use this skill when implementing memory for AI agents. Activate when the user needs agents to remember
  past interactions, implement context persistence, build knowledge bases for agents, design agent state
  management, or create shared memory between multiple agents.
---

# Agent Memory Systems

Design and implement memory systems that give agents persistent knowledge and context.

## When to Use

- Agents need to remember across sessions
- Multiple agents share knowledge
- Long-running tasks require state persistence
- Building agents that learn from experience

## Memory Types

```
┌─────────────────────────────────────────────────────────────┐
│                    AGENT MEMORY TAXONOMY                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Working Memory (Active Context)                             │
│  └─ Current conversation, immediate task state               │
│                                                              │
│  Short-Term Memory (Session)                                 │
│  └─ Recent interactions, temporary facts                     │
│                                                              │
│  Long-Term Memory (Persistent)                               │
│  ├─ Episodic: Past events, experiences                       │
│  ├─ Semantic: Facts, knowledge, learned info                 │
│  └─ Procedural: How to do things, skills                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Working Memory

The agent's current context window.

```typescript
interface WorkingMemory {
  // Current task context
  task: {
    description: string;
    requirements: string[];
    progress: string[];
  };

  // Active entities being discussed
  entities: Map<string, Entity>;

  // Recent messages
  conversationWindow: Message[];

  // Scratchpad for reasoning
  scratchpad: string;
}

class WorkingMemoryManager {
  private memory: WorkingMemory;
  private MAX_MESSAGES = 20;

  addMessage(message: Message) {
    this.memory.conversationWindow.push(message);

    // Evict old messages
    if (this.memory.conversationWindow.length > this.MAX_MESSAGES) {
      const evicted = this.memory.conversationWindow.shift();
      // Optionally summarize and store
      this.summarizeToShortTerm(evicted);
    }
  }

  updateEntity(id: string, entity: Entity) {
    this.memory.entities.set(id, entity);
  }

  getContext(): string {
    return `
Task: ${this.memory.task.description}
Progress: ${this.memory.task.progress.join(', ')}
Key Entities: ${[...this.memory.entities.values()].map(e => e.summary).join('\n')}
Recent Discussion: ${this.memory.conversationWindow.slice(-5).map(m => m.content).join('\n')}
    `;
  }
}
```

## Short-Term Memory

Session-scoped, expires after inactivity.

```typescript
interface ShortTermMemory {
  sessionId: string;
  startedAt: Date;
  lastAccessedAt: Date;
  expiresAt: Date;

  // Conversation summary
  summary: string;

  // Key facts learned this session
  facts: Fact[];

  // Decisions made
  decisions: Decision[];

  // Files accessed
  touchedFiles: string[];
}

class ShortTermStore {
  private store = new Map<string, ShortTermMemory>();
  private TTL_MS = 30 * 60 * 1000; // 30 minutes

  async get(sessionId: string): Promise<ShortTermMemory | null> {
    const memory = this.store.get(sessionId);

    if (!memory) return null;
    if (Date.now() > memory.expiresAt.getTime()) {
      // Expired - archive to long-term
      await this.archiveToLongTerm(memory);
      this.store.delete(sessionId);
      return null;
    }

    // Touch
    memory.lastAccessedAt = new Date();
    memory.expiresAt = new Date(Date.now() + this.TTL_MS);

    return memory;
  }

  async addFact(sessionId: string, fact: Fact) {
    const memory = await this.getOrCreate(sessionId);
    memory.facts.push(fact);

    // Check if fact should be promoted to long-term
    if (this.shouldPromote(fact)) {
      await this.promoteToLongTerm(fact);
    }
  }
}
```

## Long-Term Memory

### Episodic Memory (Events/Experiences)

```typescript
interface Episode {
  id: string;
  timestamp: Date;
  type: 'task_completed' | 'error_resolved' | 'user_feedback' | 'learning';

  // What happened
  description: string;
  context: Record<string, unknown>;

  // Outcome
  outcome: 'success' | 'failure' | 'partial';
  lessons: string[];

  // For retrieval
  embedding: number[];
  tags: string[];
}

class EpisodicMemory {
  private vectorStore: VectorStore;

  async remember(episode: Episode): Promise<void> {
    // Generate embedding for semantic search
    episode.embedding = await this.embed(
      `${episode.description} ${episode.lessons.join(' ')}`
    );

    await this.vectorStore.insert(episode);
  }

  async recall(query: string, limit: number = 5): Promise<Episode[]> {
    const queryEmbedding = await this.embed(query);

    return this.vectorStore.similaritySearch(queryEmbedding, {
      limit,
      threshold: 0.7
    });
  }

  async recallByType(type: Episode['type'], limit: number = 10): Promise<Episode[]> {
    return this.vectorStore.filter({ type }, { limit, orderBy: 'timestamp DESC' });
  }
}
```

### Semantic Memory (Facts/Knowledge)

```typescript
interface Fact {
  id: string;
  subject: string;
  predicate: string;
  object: string;
  confidence: number;
  source: string;
  learnedAt: Date;
  lastVerified: Date;
  embedding: number[];
}

class SemanticMemory {
  private facts: VectorStore<Fact>;

  async learn(fact: Omit<Fact, 'id' | 'embedding'>): Promise<void> {
    // Check for conflicts
    const existing = await this.findRelated(fact.subject, fact.predicate);

    if (existing.length > 0) {
      // Handle contradiction or update
      await this.resolveConflict(existing, fact);
      return;
    }

    // Store new fact
    await this.facts.insert({
      ...fact,
      id: generateId(),
      embedding: await this.embed(`${fact.subject} ${fact.predicate} ${fact.object}`)
    });
  }

  async query(question: string): Promise<Fact[]> {
    // Semantic search for relevant facts
    const embedding = await this.embed(question);
    return this.facts.similaritySearch(embedding, { limit: 10 });
  }

  async getFactsAbout(subject: string): Promise<Fact[]> {
    return this.facts.filter({ subject });
  }
}
```

### Procedural Memory (Skills/How-To)

```typescript
interface Procedure {
  id: string;
  name: string;
  description: string;
  trigger: string; // When to use this
  steps: string[];
  examples: Example[];
  successRate: number;
  usageCount: number;
}

class ProceduralMemory {
  private procedures: Map<string, Procedure> = new Map();

  async findProcedure(task: string): Promise<Procedure | null> {
    // Match task to known procedures
    const candidates = await this.matchProcedures(task);

    if (candidates.length === 0) return null;

    // Return best match by success rate and relevance
    return candidates.sort((a, b) =>
      (b.successRate * b.relevanceScore) - (a.successRate * a.relevanceScore)
    )[0];
  }

  async recordOutcome(procedureId: string, success: boolean): Promise<void> {
    const proc = this.procedures.get(procedureId);
    if (!proc) return;

    // Update success rate with exponential moving average
    const alpha = 0.1;
    proc.successRate = alpha * (success ? 1 : 0) + (1 - alpha) * proc.successRate;
    proc.usageCount++;
  }

  async learnProcedure(
    name: string,
    steps: string[],
    fromEpisode: Episode
  ): Promise<void> {
    // Create new procedure from successful episode
    this.procedures.set(generateId(), {
      id: generateId(),
      name,
      description: fromEpisode.description,
      trigger: this.extractTrigger(fromEpisode),
      steps,
      examples: [{ input: fromEpisode.context, output: fromEpisode.outcome }],
      successRate: 1.0,
      usageCount: 1
    });
  }
}
```

## Shared Memory (Multi-Agent)

```typescript
interface SharedMemory {
  // Namespace for isolation
  namespace: string;

  // Read/write with locking
  read(key: string): Promise<unknown>;
  write(key: string, value: unknown): Promise<void>;

  // Atomic operations
  compareAndSwap(key: string, expected: unknown, newValue: unknown): Promise<boolean>;

  // Subscriptions
  subscribe(pattern: string, callback: (key: string, value: unknown) => void): void;
}

class RedisSharedMemory implements SharedMemory {
  constructor(
    private redis: Redis,
    public namespace: string
  ) {}

  private key(k: string): string {
    return `${this.namespace}:${k}`;
  }

  async read(key: string): Promise<unknown> {
    const value = await this.redis.get(this.key(key));
    return value ? JSON.parse(value) : null;
  }

  async write(key: string, value: unknown): Promise<void> {
    await this.redis.set(this.key(key), JSON.stringify(value));
    await this.redis.publish(`${this.namespace}:updates`, JSON.stringify({ key, value }));
  }

  subscribe(pattern: string, callback: (key: string, value: unknown) => void): void {
    const subscriber = this.redis.duplicate();
    subscriber.psubscribe(`${this.namespace}:${pattern}`);
    subscriber.on('pmessage', (_, channel, message) => {
      const { key, value } = JSON.parse(message);
      callback(key, value);
    });
  }
}
```

## Memory Retrieval Strategies

### Strategy 1: Recency-Weighted

```typescript
function recencyWeightedRetrieval(
  memories: Memory[],
  query: string,
  recencyWeight: number = 0.3
): Memory[] {
  const now = Date.now();

  return memories
    .map(m => ({
      memory: m,
      score: (1 - recencyWeight) * m.relevanceScore +
             recencyWeight * Math.exp(-(now - m.timestamp.getTime()) / TIME_DECAY)
    }))
    .sort((a, b) => b.score - a.score)
    .map(x => x.memory);
}
```

### Strategy 2: Importance-Based

```typescript
function importanceBasedRetrieval(
  memories: Memory[],
  query: string
): Memory[] {
  return memories
    .map(m => ({
      memory: m,
      score: m.relevanceScore * m.importance * (m.accessCount / 10)
    }))
    .sort((a, b) => b.score - a.score)
    .map(x => x.memory);
}
```

### Strategy 3: Contextual (RAG)

```typescript
async function contextualRetrieval(
  query: string,
  currentContext: Context
): Promise<Memory[]> {
  // Expand query with context
  const expandedQuery = `
    ${query}
    Current task: ${currentContext.task}
    Related entities: ${currentContext.entities.join(', ')}
  `;

  // Vector search
  const candidates = await vectorStore.search(expandedQuery, { limit: 20 });

  // Rerank with cross-encoder
  return reranker.rerank(query, candidates, { limit: 5 });
}
```

## Memory Maintenance

```typescript
class MemoryMaintenance {
  // Consolidate short-term to long-term
  async consolidate(): Promise<void> {
    const sessions = await shortTermStore.getExpired();

    for (const session of sessions) {
      // Extract key learnings
      const learnings = await this.extractLearnings(session);

      // Store in appropriate long-term stores
      for (const learning of learnings) {
        if (learning.type === 'fact') {
          await semanticMemory.learn(learning);
        } else if (learning.type === 'procedure') {
          await proceduralMemory.learnProcedure(learning);
        } else {
          await episodicMemory.remember(learning);
        }
      }
    }
  }

  // Forget outdated/irrelevant memories
  async forget(): Promise<void> {
    // Remove low-value memories
    await episodicMemory.prune({
      olderThan: days(90),
      accessCountBelow: 2,
      importanceBelow: 0.2
    });

    // Update fact confidence based on verification
    await semanticMemory.decayUnverified({
      olderThan: days(30),
      decayRate: 0.1
    });
  }
}
```

## Best Practices

1. **Separate concerns** - Different memory types for different purposes
2. **Index wisely** - Use embeddings for semantic search
3. **Expire aggressively** - Don't let memory grow unbounded
4. **Version memories** - Track when facts were learned/updated
5. **Handle conflicts** - New information may contradict old
6. **Secure sensitive data** - Some memories shouldn't be shared
