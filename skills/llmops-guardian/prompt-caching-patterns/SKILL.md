---
name: prompt-caching-patterns
description: |
  Use this skill when implementing caching for LLM applications. Activate when the user wants to reduce
  API costs through caching, implement semantic caching, cache LLM responses, optimize repeated prompts,
  or set up efficient caching strategies for AI applications.
---

# Prompt Caching Patterns

Implement effective caching strategies to reduce LLM costs by up to 90%.

## When to Use

- Same or similar prompts are sent repeatedly
- Large system prompts are reused across requests
- Responses can be reused for identical queries
- Need to reduce latency for common requests
- Optimizing costs for high-volume applications

## Caching Strategies

### 1. Provider-Level Caching (Anthropic)

Anthropic offers built-in prompt caching with 90% cost reduction.

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

// Large system context that will be reused
const systemContext = `
[Your long system prompt, documentation, examples, etc.]
This can be many thousands of tokens that you want to cache.
`;

async function queryWithCache(userQuestion: string) {
  const response = await client.messages.create({
    model: 'claude-3-sonnet-20240229',
    max_tokens: 1024,
    system: [
      {
        type: 'text',
        text: systemContext,
        cache_control: { type: 'ephemeral' } // Cache for 5 minutes
      }
    ],
    messages: [
      { role: 'user', content: userQuestion }
    ]
  });

  // Check cache usage
  console.log('Cache read tokens:', response.usage.cache_read_input_tokens);
  console.log('Cache creation tokens:', response.usage.cache_creation_input_tokens);

  return response;
}
```

**Pricing with cache:**
- Cache write: 25% more than base input price
- Cache read: 90% less than base input price
- Break-even: ~2 requests with same cached content

### 2. Response Caching

Cache LLM responses for identical or similar queries.

```typescript
interface CacheEntry {
  response: string;
  createdAt: number;
  ttlMs: number;
  metadata: {
    model: string;
    inputTokens: number;
    outputTokens: number;
  };
}

class ResponseCache {
  private cache = new Map<string, CacheEntry>();

  private hashPrompt(prompt: string): string {
    // Simple hash for exact matching
    return crypto.createHash('sha256').update(prompt).digest('hex');
  }

  get(prompt: string): string | null {
    const key = this.hashPrompt(prompt);
    const entry = this.cache.get(key);

    if (!entry) return null;

    // Check TTL
    if (Date.now() - entry.createdAt > entry.ttlMs) {
      this.cache.delete(key);
      return null;
    }

    return entry.response;
  }

  set(prompt: string, response: string, options: { ttlMs?: number; metadata?: any } = {}): void {
    const key = this.hashPrompt(prompt);
    this.cache.set(key, {
      response,
      createdAt: Date.now(),
      ttlMs: options.ttlMs || 3600000, // 1 hour default
      metadata: options.metadata
    });
  }
}

// Usage
const cache = new ResponseCache();

async function cachedQuery(prompt: string): Promise<string> {
  // Check cache first
  const cached = cache.get(prompt);
  if (cached) {
    console.log('Cache hit!');
    return cached;
  }

  // Make API call
  const response = await llm.complete(prompt);

  // Cache the response
  cache.set(prompt, response, { ttlMs: 3600000 });

  return response;
}
```

### 3. Semantic Caching

Cache based on meaning, not exact match.

```typescript
import { OpenAIEmbeddings } from 'langchain/embeddings/openai';

class SemanticCache {
  private entries: { embedding: number[]; response: string; prompt: string }[] = [];
  private embeddings: OpenAIEmbeddings;
  private similarityThreshold = 0.95;

  constructor() {
    this.embeddings = new OpenAIEmbeddings();
  }

  async get(prompt: string): Promise<string | null> {
    const queryEmbedding = await this.embeddings.embedQuery(prompt);

    // Find most similar cached prompt
    let bestMatch: { similarity: number; response: string } | null = null;

    for (const entry of this.entries) {
      const similarity = this.cosineSimilarity(queryEmbedding, entry.embedding);

      if (similarity > this.similarityThreshold) {
        if (!bestMatch || similarity > bestMatch.similarity) {
          bestMatch = { similarity, response: entry.response };
        }
      }
    }

    return bestMatch?.response || null;
  }

  async set(prompt: string, response: string): Promise<void> {
    const embedding = await this.embeddings.embedQuery(prompt);
    this.entries.push({ embedding, response, prompt });
  }

  private cosineSimilarity(a: number[], b: number[]): number {
    let dotProduct = 0;
    let normA = 0;
    let normB = 0;

    for (let i = 0; i < a.length; i++) {
      dotProduct += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }

    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
  }
}

// Usage
const semanticCache = new SemanticCache();

// These would hit the cache:
// "What is the capital of France?" -> cached
// "What's France's capital city?" -> semantic match!
```

### 4. Template Caching

Cache static parts, vary dynamic parts.

```typescript
interface PromptTemplate {
  staticPart: string;
  dynamicParts: string[];
}

class TemplateCache {
  private templates = new Map<string, {
    staticPartHash: string;
    responses: Map<string, string>; // dynamicHash -> response
  }>();

  generateKey(template: PromptTemplate, values: Record<string, string>): {
    templateKey: string;
    valuesKey: string;
  } {
    const templateKey = this.hash(template.staticPart);
    const valuesKey = this.hash(JSON.stringify(values));
    return { templateKey, valuesKey };
  }

  get(template: PromptTemplate, values: Record<string, string>): string | null {
    const { templateKey, valuesKey } = this.generateKey(template, values);
    return this.templates.get(templateKey)?.responses.get(valuesKey) || null;
  }

  set(template: PromptTemplate, values: Record<string, string>, response: string): void {
    const { templateKey, valuesKey } = this.generateKey(template, values);

    if (!this.templates.has(templateKey)) {
      this.templates.set(templateKey, {
        staticPartHash: templateKey,
        responses: new Map()
      });
    }

    this.templates.get(templateKey)!.responses.set(valuesKey, response);
  }
}

// Usage
const template: PromptTemplate = {
  staticPart: `You are a helpful assistant that translates text.
    Translate the following to the target language.
    Be accurate and natural.`,
  dynamicParts: ['text', 'targetLanguage']
};

// Cache hit for same text + language combo
const cached = templateCache.get(template, {
  text: 'Hello world',
  targetLanguage: 'Spanish'
});
```

## Redis-Based Distributed Cache

```typescript
import Redis from 'ioredis';

class DistributedPromptCache {
  private redis: Redis;
  private prefix = 'llm:cache:';

  constructor(redisUrl: string) {
    this.redis = new Redis(redisUrl);
  }

  private key(prompt: string): string {
    const hash = crypto.createHash('sha256').update(prompt).digest('hex');
    return `${this.prefix}${hash}`;
  }

  async get(prompt: string): Promise<string | null> {
    const cached = await this.redis.get(this.key(prompt));
    if (cached) {
      await this.redis.hincrby(`${this.prefix}stats`, 'hits', 1);
    } else {
      await this.redis.hincrby(`${this.prefix}stats`, 'misses', 1);
    }
    return cached;
  }

  async set(prompt: string, response: string, ttlSeconds: number = 3600): Promise<void> {
    await this.redis.setex(this.key(prompt), ttlSeconds, response);
  }

  async getStats(): Promise<{ hits: number; misses: number; hitRate: number }> {
    const stats = await this.redis.hgetall(`${this.prefix}stats`);
    const hits = parseInt(stats.hits || '0');
    const misses = parseInt(stats.misses || '0');
    const total = hits + misses;

    return {
      hits,
      misses,
      hitRate: total > 0 ? hits / total : 0
    };
  }
}
```

## Cache Invalidation

```typescript
interface CachePolicy {
  ttlMs: number;
  invalidateOn: string[]; // Events that invalidate cache
  tags: string[]; // For tag-based invalidation
}

class SmartCache {
  private cache = new Map<string, { value: string; policy: CachePolicy; createdAt: number }>();
  private tagIndex = new Map<string, Set<string>>(); // tag -> keys

  set(key: string, value: string, policy: CachePolicy): void {
    this.cache.set(key, { value, policy, createdAt: Date.now() });

    // Index by tags
    for (const tag of policy.tags) {
      if (!this.tagIndex.has(tag)) {
        this.tagIndex.set(tag, new Set());
      }
      this.tagIndex.get(tag)!.add(key);
    }
  }

  invalidateByTag(tag: string): number {
    const keys = this.tagIndex.get(tag) || new Set();
    let count = 0;

    for (const key of keys) {
      if (this.cache.delete(key)) count++;
    }

    this.tagIndex.delete(tag);
    return count;
  }

  invalidateByEvent(event: string): number {
    let count = 0;

    for (const [key, entry] of this.cache) {
      if (entry.policy.invalidateOn.includes(event)) {
        this.cache.delete(key);
        count++;
      }
    }

    return count;
  }
}

// Usage
cache.set('user:123:summary', response, {
  ttlMs: 3600000,
  invalidateOn: ['user:123:updated', 'user:123:deleted'],
  tags: ['user:123', 'summaries']
});

// When user updates their profile
cache.invalidateByEvent('user:123:updated');

// Or invalidate all summaries
cache.invalidateByTag('summaries');
```

## Cost Savings Calculator

```typescript
function calculateCacheSavings(
  stats: { hits: number; misses: number },
  avgInputTokens: number,
  avgOutputTokens: number,
  pricing: { inputPer1M: number; outputPer1M: number }
): {
  withoutCache: number;
  withCache: number;
  savings: number;
  savingsPercent: number;
} {
  const totalRequests = stats.hits + stats.misses;

  // Without cache: all requests hit API
  const withoutCache = totalRequests * (
    (avgInputTokens / 1_000_000) * pricing.inputPer1M +
    (avgOutputTokens / 1_000_000) * pricing.outputPer1M
  );

  // With cache: only misses hit API
  const withCache = stats.misses * (
    (avgInputTokens / 1_000_000) * pricing.inputPer1M +
    (avgOutputTokens / 1_000_000) * pricing.outputPer1M
  );

  return {
    withoutCache,
    withCache,
    savings: withoutCache - withCache,
    savingsPercent: ((withoutCache - withCache) / withoutCache) * 100
  };
}
```

## Best Practices

1. **Cache at the right level** - Response, prompt part, or embedding
2. **Set appropriate TTLs** - Balance freshness vs. savings
3. **Monitor hit rates** - Low hit rate means cache isn't helping
4. **Invalidate intelligently** - Don't serve stale data
5. **Use semantic caching carefully** - Embedding costs add up
6. **Warm the cache** - Pre-populate for known queries
7. **Consider cache size** - Memory isn't free either
