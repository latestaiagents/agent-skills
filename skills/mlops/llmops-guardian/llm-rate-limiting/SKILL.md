---
name: llm-rate-limiting
description: |
  Use this skill when implementing rate limiting for LLM APIs. Activate when the user needs to prevent
  API quota exhaustion, implement backoff strategies, handle rate limit errors, or manage concurrent
  LLM requests.
---

# LLM Rate Limiting

Implement robust rate limiting to prevent quota exhaustion and handle API limits gracefully.

## When to Use

- Hitting API rate limits
- Managing concurrent requests
- Preventing quota exhaustion
- Implementing fair usage policies
- Handling burst traffic

## API Rate Limits (2026)

### Anthropic Claude

| Tier | Requests/min | Tokens/min | Tokens/day |
|------|-------------|------------|------------|
| Free | 5 | 20K | 300K |
| Tier 1 | 50 | 40K | 1M |
| Tier 2 | 1000 | 80K | 2.5M |
| Tier 3 | 2000 | 160K | 5M |
| Tier 4 | 4000 | 400K | 10M |

### OpenAI

| Tier | RPM | TPM |
|------|-----|-----|
| Free | 3 | 40K |
| Tier 1 | 500 | 200K |
| Tier 2 | 5000 | 450K |
| Tier 3 | 5000 | 800K |
| Tier 4 | 10000 | 2M |

## Rate Limiter Implementation

### Token Bucket Algorithm

```typescript
class TokenBucket {
  private tokens: number;
  private lastRefill: number;

  constructor(
    private capacity: number,      // Max tokens
    private refillRate: number,    // Tokens per ms
  ) {
    this.tokens = capacity;
    this.lastRefill = Date.now();
  }

  private refill(): void {
    const now = Date.now();
    const elapsed = now - this.lastRefill;
    const newTokens = elapsed * this.refillRate;

    this.tokens = Math.min(this.capacity, this.tokens + newTokens);
    this.lastRefill = now;
  }

  async acquire(tokens: number = 1): Promise<boolean> {
    this.refill();

    if (this.tokens >= tokens) {
      this.tokens -= tokens;
      return true;
    }

    return false;
  }

  async waitForTokens(tokens: number = 1): Promise<void> {
    while (!(await this.acquire(tokens))) {
      const waitTime = (tokens - this.tokens) / this.refillRate;
      await sleep(Math.min(waitTime, 1000)); // Max 1 second wait
    }
  }
}

// Usage
const limiter = new TokenBucket(
  1000,  // 1000 tokens capacity
  1000 / 60000  // 1000 tokens per minute = ~16.67 per second
);

async function makeRequest() {
  await limiter.waitForTokens(1);
  return callAPI();
}
```

### Sliding Window Rate Limiter

```typescript
class SlidingWindowLimiter {
  private timestamps: number[] = [];

  constructor(
    private maxRequests: number,
    private windowMs: number
  ) {}

  canProceed(): boolean {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Remove old timestamps
    this.timestamps = this.timestamps.filter(t => t > windowStart);

    return this.timestamps.length < this.maxRequests;
  }

  recordRequest(): void {
    this.timestamps.push(Date.now());
  }

  getWaitTime(): number {
    if (this.canProceed()) return 0;

    const oldestInWindow = this.timestamps[0];
    return oldestInWindow + this.windowMs - Date.now();
  }
}

// 50 requests per minute
const limiter = new SlidingWindowLimiter(50, 60000);

async function rateLimitedRequest() {
  while (!limiter.canProceed()) {
    const waitTime = limiter.getWaitTime();
    console.log(`Rate limited, waiting ${waitTime}ms`);
    await sleep(waitTime);
  }

  limiter.recordRequest();
  return makeAPICall();
}
```

### Concurrent Request Limiter

```typescript
class ConcurrencyLimiter {
  private running = 0;
  private queue: (() => void)[] = [];

  constructor(private maxConcurrent: number) {}

  async acquire(): Promise<void> {
    if (this.running < this.maxConcurrent) {
      this.running++;
      return;
    }

    // Wait in queue
    return new Promise(resolve => {
      this.queue.push(resolve);
    });
  }

  release(): void {
    this.running--;

    if (this.queue.length > 0) {
      const next = this.queue.shift()!;
      this.running++;
      next();
    }
  }

  async run<T>(fn: () => Promise<T>): Promise<T> {
    await this.acquire();
    try {
      return await fn();
    } finally {
      this.release();
    }
  }
}

// Max 10 concurrent requests
const concurrencyLimiter = new ConcurrencyLimiter(10);

async function processMany(items: string[]): Promise<string[]> {
  return Promise.all(
    items.map(item =>
      concurrencyLimiter.run(() => processItem(item))
    )
  );
}
```

## Exponential Backoff

```typescript
interface BackoffConfig {
  initialDelayMs: number;
  maxDelayMs: number;
  multiplier: number;
  jitterFactor: number; // 0-1
}

class ExponentialBackoff {
  private attempt = 0;

  constructor(private config: BackoffConfig) {}

  getDelay(): number {
    const baseDelay = this.config.initialDelayMs *
      Math.pow(this.config.multiplier, this.attempt);

    const cappedDelay = Math.min(baseDelay, this.config.maxDelayMs);

    // Add jitter
    const jitter = cappedDelay * this.config.jitterFactor * Math.random();

    return cappedDelay + jitter;
  }

  increment(): void {
    this.attempt++;
  }

  reset(): void {
    this.attempt = 0;
  }
}

async function withBackoff<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 5
): Promise<T> {
  const backoff = new ExponentialBackoff({
    initialDelayMs: 1000,
    maxDelayMs: 60000,
    multiplier: 2,
    jitterFactor: 0.1
  });

  let lastError: Error;

  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      const result = await fn();
      backoff.reset();
      return result;
    } catch (error) {
      lastError = error as Error;

      if (!isRetryable(error)) {
        throw error;
      }

      const delay = backoff.getDelay();
      console.log(`Attempt ${attempt + 1} failed, retrying in ${delay}ms`);
      await sleep(delay);
      backoff.increment();
    }
  }

  throw lastError!;
}

function isRetryable(error: any): boolean {
  // Rate limit errors
  if (error.status === 429) return true;

  // Server errors
  if (error.status >= 500) return true;

  // Network errors
  if (error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') return true;

  return false;
}
```

## Handling Rate Limit Responses

### Anthropic Rate Limit Headers

```typescript
async function handleAnthropicResponse(response: Response): Promise<void> {
  // Check rate limit headers
  const requestsRemaining = response.headers.get('anthropic-ratelimit-requests-remaining');
  const tokensRemaining = response.headers.get('anthropic-ratelimit-tokens-remaining');
  const requestsReset = response.headers.get('anthropic-ratelimit-requests-reset');
  const tokensReset = response.headers.get('anthropic-ratelimit-tokens-reset');

  console.log(`Requests remaining: ${requestsRemaining}`);
  console.log(`Tokens remaining: ${tokensRemaining}`);

  // Proactively slow down if approaching limits
  if (parseInt(requestsRemaining || '999') < 10) {
    const resetTime = new Date(requestsReset!).getTime();
    const waitMs = Math.max(0, resetTime - Date.now());
    console.log(`Approaching rate limit, waiting ${waitMs}ms`);
    await sleep(waitMs);
  }
}
```

### OpenAI Rate Limit Headers

```typescript
interface OpenAIRateLimitInfo {
  limitRequests: number;
  limitTokens: number;
  remainingRequests: number;
  remainingTokens: number;
  resetRequests: string;
  resetTokens: string;
}

function parseOpenAIHeaders(headers: Headers): OpenAIRateLimitInfo {
  return {
    limitRequests: parseInt(headers.get('x-ratelimit-limit-requests') || '0'),
    limitTokens: parseInt(headers.get('x-ratelimit-limit-tokens') || '0'),
    remainingRequests: parseInt(headers.get('x-ratelimit-remaining-requests') || '0'),
    remainingTokens: parseInt(headers.get('x-ratelimit-remaining-tokens') || '0'),
    resetRequests: headers.get('x-ratelimit-reset-requests') || '',
    resetTokens: headers.get('x-ratelimit-reset-tokens') || ''
  };
}
```

## Request Queue

```typescript
interface QueuedRequest {
  id: string;
  execute: () => Promise<any>;
  resolve: (value: any) => void;
  reject: (error: Error) => void;
  priority: number;
  addedAt: number;
}

class RequestQueue {
  private queue: QueuedRequest[] = [];
  private processing = false;

  constructor(
    private rateLimiter: SlidingWindowLimiter,
    private concurrencyLimiter: ConcurrencyLimiter
  ) {}

  async enqueue<T>(
    execute: () => Promise<T>,
    priority: number = 0
  ): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push({
        id: crypto.randomUUID(),
        execute,
        resolve,
        reject,
        priority,
        addedAt: Date.now()
      });

      // Sort by priority (higher first), then by age
      this.queue.sort((a, b) =>
        b.priority - a.priority || a.addedAt - b.addedAt
      );

      this.process();
    });
  }

  private async process(): Promise<void> {
    if (this.processing) return;
    this.processing = true;

    while (this.queue.length > 0) {
      // Wait for rate limit
      while (!this.rateLimiter.canProceed()) {
        await sleep(100);
      }

      const request = this.queue.shift()!;
      this.rateLimiter.recordRequest();

      // Run with concurrency limit
      this.concurrencyLimiter.run(async () => {
        try {
          const result = await request.execute();
          request.resolve(result);
        } catch (error) {
          request.reject(error as Error);
        }
      });
    }

    this.processing = false;
  }
}
```

## Monitoring

```typescript
interface RateLimitMetrics {
  totalRequests: number;
  rateLimitedRequests: number;
  avgWaitTimeMs: number;
  peakConcurrency: number;
  errorRate: number;
}

class RateLimitMonitor {
  private metrics: RateLimitMetrics = {
    totalRequests: 0,
    rateLimitedRequests: 0,
    avgWaitTimeMs: 0,
    peakConcurrency: 0,
    errorRate: 0
  };

  recordRequest(wasLimited: boolean, waitTimeMs: number): void {
    this.metrics.totalRequests++;
    if (wasLimited) {
      this.metrics.rateLimitedRequests++;
    }

    // Update average wait time
    this.metrics.avgWaitTimeMs =
      (this.metrics.avgWaitTimeMs * (this.metrics.totalRequests - 1) + waitTimeMs) /
      this.metrics.totalRequests;
  }

  getMetrics(): RateLimitMetrics {
    return { ...this.metrics };
  }
}
```

## Best Practices

1. **Implement client-side limiting** - Don't rely only on API errors
2. **Use exponential backoff** - With jitter to avoid thundering herd
3. **Monitor remaining quota** - Proactively slow down
4. **Queue requests** - Don't fire and forget
5. **Set appropriate timeouts** - Don't wait forever
6. **Log rate limit events** - For capacity planning
7. **Have fallback strategies** - What happens when limit is hit?
