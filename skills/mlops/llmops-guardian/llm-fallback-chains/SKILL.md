---
name: llm-fallback-chains
description: |
  Use this skill when implementing fallback strategies for LLM applications. Activate when the user needs
  graceful degradation for AI services, multi-provider failover, handling LLM outages, or building
  resilient AI systems.
---

# LLM Fallback Chains

Build resilient AI systems that gracefully handle failures across providers and models.

## When to Use

- Primary LLM provider experiences outages
- Need to maintain service during API issues
- Building high-availability AI systems
- Implementing cost-quality tradeoffs
- Managing multi-provider AI infrastructure

## Fallback Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Request Handler                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Fallback Chain                           │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐     │
│  │Primary  │──►│Fallback │──►│Fallback │──►│ Cached  │     │
│  │ Model   │   │ Model 1 │   │ Model 2 │   │Response │     │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## Fallback Chain Implementation

```typescript
interface FallbackProvider {
  name: string;
  model: string;
  client: LLMClient;
  priority: number;
  healthCheck: () => Promise<boolean>;
  isAvailable: boolean;
  lastFailure?: Date;
  failureCount: number;
}

interface FallbackConfig {
  maxRetries: number;
  retryDelayMs: number;
  circuitBreakerThreshold: number;
  circuitBreakerResetMs: number;
}

class FallbackChain {
  private providers: FallbackProvider[] = [];
  private config: FallbackConfig;

  constructor(config: FallbackConfig) {
    this.config = config;
  }

  addProvider(provider: Omit<FallbackProvider, 'isAvailable' | 'failureCount'>): void {
    this.providers.push({
      ...provider,
      isAvailable: true,
      failureCount: 0
    });
    this.providers.sort((a, b) => a.priority - b.priority);
  }

  async complete(params: CompletionParams): Promise<CompletionResponse> {
    const availableProviders = this.providers.filter(p =>
      p.isAvailable || this.shouldRetryProvider(p)
    );

    if (availableProviders.length === 0) {
      throw new Error('All providers unavailable');
    }

    let lastError: Error | null = null;

    for (const provider of availableProviders) {
      try {
        console.log(`Trying provider: ${provider.name}`);
        const response = await this.executeWithTimeout(provider, params);

        // Success - reset failure count
        provider.failureCount = 0;
        provider.isAvailable = true;

        return response;
      } catch (error) {
        lastError = error as Error;
        console.error(`Provider ${provider.name} failed:`, error);

        this.recordFailure(provider);

        if (!this.isRetryableError(error)) {
          throw error; // Don't try other providers for non-retryable errors
        }
      }
    }

    throw lastError || new Error('All providers failed');
  }

  private shouldRetryProvider(provider: FallbackProvider): boolean {
    if (!provider.lastFailure) return true;

    const timeSinceFailure = Date.now() - provider.lastFailure.getTime();
    return timeSinceFailure > this.config.circuitBreakerResetMs;
  }

  private recordFailure(provider: FallbackProvider): void {
    provider.failureCount++;
    provider.lastFailure = new Date();

    if (provider.failureCount >= this.config.circuitBreakerThreshold) {
      provider.isAvailable = false;
      console.warn(`Circuit breaker opened for ${provider.name}`);
    }
  }

  private isRetryableError(error: any): boolean {
    // Rate limits, timeouts, and server errors are retryable
    if (error.status === 429) return true;
    if (error.status >= 500) return true;
    if (error.code === 'ETIMEDOUT' || error.code === 'ECONNRESET') return true;
    return false;
  }

  private async executeWithTimeout(
    provider: FallbackProvider,
    params: CompletionParams
  ): Promise<CompletionResponse> {
    const timeoutMs = 30000;

    return Promise.race([
      provider.client.complete({ ...params, model: provider.model }),
      new Promise<never>((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), timeoutMs)
      )
    ]);
  }
}
```

## Multi-Provider Setup

```typescript
const fallbackChain = new FallbackChain({
  maxRetries: 3,
  retryDelayMs: 1000,
  circuitBreakerThreshold: 5,
  circuitBreakerResetMs: 60000
});

// Primary: Claude Sonnet
fallbackChain.addProvider({
  name: 'anthropic-sonnet',
  model: 'claude-3.5-sonnet-20241022',
  client: new AnthropicClient(process.env.ANTHROPIC_API_KEY!),
  priority: 1,
  healthCheck: () => anthropicHealthCheck()
});

// Fallback 1: GPT-4o
fallbackChain.addProvider({
  name: 'openai-gpt4o',
  model: 'gpt-4o',
  client: new OpenAIClient(process.env.OPENAI_API_KEY!),
  priority: 2,
  healthCheck: () => openaiHealthCheck()
});

// Fallback 2: Claude Haiku (cheaper, faster)
fallbackChain.addProvider({
  name: 'anthropic-haiku',
  model: 'claude-3-haiku-20240307',
  client: new AnthropicClient(process.env.ANTHROPIC_API_KEY!),
  priority: 3,
  healthCheck: () => anthropicHealthCheck()
});

// Fallback 3: GPT-4o-mini (cheapest)
fallbackChain.addProvider({
  name: 'openai-gpt4o-mini',
  model: 'gpt-4o-mini',
  client: new OpenAIClient(process.env.OPENAI_API_KEY!),
  priority: 4,
  healthCheck: () => openaiHealthCheck()
});
```

## Graceful Degradation Strategies

### Strategy 1: Quality Degradation

```typescript
interface DegradationLevel {
  level: number;
  model: string;
  maxTokens: number;
  features: string[];
}

const degradationLevels: DegradationLevel[] = [
  { level: 0, model: 'claude-3-opus', maxTokens: 4000, features: ['full', 'reasoning', 'creativity'] },
  { level: 1, model: 'claude-3.5-sonnet', maxTokens: 2000, features: ['full', 'reasoning'] },
  { level: 2, model: 'claude-3-haiku', maxTokens: 1000, features: ['basic'] },
  { level: 3, model: 'gpt-4o-mini', maxTokens: 500, features: ['minimal'] },
];

class GracefulDegrader {
  private currentLevel = 0;

  async execute(task: string): Promise<{ response: string; degraded: boolean }> {
    for (let level = this.currentLevel; level < degradationLevels.length; level++) {
      try {
        const config = degradationLevels[level];
        const response = await this.callModel(config, task);

        return {
          response,
          degraded: level > 0
        };
      } catch (error) {
        console.log(`Level ${level} failed, degrading...`);
        this.currentLevel = level + 1;
      }
    }

    // Final fallback: cached/static response
    return {
      response: this.getCachedResponse(task),
      degraded: true
    };
  }

  private getCachedResponse(task: string): string {
    return 'We are experiencing high demand. Please try again shortly.';
  }
}
```

### Strategy 2: Feature-Based Fallback

```typescript
interface FeatureConfig {
  name: string;
  requiredCapabilities: string[];
  fallbackBehavior: 'skip' | 'simplify' | 'cache';
}

class FeatureFallback {
  private featureConfigs: FeatureConfig[] = [
    { name: 'streaming', requiredCapabilities: ['sse'], fallbackBehavior: 'skip' },
    { name: 'functionCalling', requiredCapabilities: ['tools'], fallbackBehavior: 'simplify' },
    { name: 'imageAnalysis', requiredCapabilities: ['vision'], fallbackBehavior: 'skip' },
  ];

  async executeWithFeatures(
    task: string,
    requestedFeatures: string[],
    provider: LLMProvider
  ): Promise<ExecutionResult> {
    const supportedFeatures = provider.capabilities;
    const enabledFeatures: string[] = [];
    const skippedFeatures: string[] = [];

    for (const feature of requestedFeatures) {
      const config = this.featureConfigs.find(f => f.name === feature);
      if (!config) continue;

      const supported = config.requiredCapabilities.every(
        cap => supportedFeatures.includes(cap)
      );

      if (supported) {
        enabledFeatures.push(feature);
      } else {
        skippedFeatures.push(feature);
        console.log(`Feature ${feature} unavailable, using fallback: ${config.fallbackBehavior}`);
      }
    }

    return this.execute(task, enabledFeatures, skippedFeatures);
  }
}
```

### Strategy 3: Cached Response Fallback

```typescript
class CacheFallback {
  private cache: ResponseCache;

  async executeWithCache(
    task: string,
    chain: FallbackChain
  ): Promise<{ response: string; fromCache: boolean }> {
    try {
      // Try live response first
      const response = await chain.complete({ messages: [{ role: 'user', content: task }] });

      // Cache successful responses
      await this.cache.set(task, response.content);

      return { response: response.content, fromCache: false };
    } catch (error) {
      // Try cache on failure
      const cached = await this.cache.get(task);

      if (cached) {
        console.log('Using cached response due to API failure');
        return { response: cached, fromCache: true };
      }

      throw error;
    }
  }
}
```

## Health Monitoring

```typescript
class ProviderHealthMonitor {
  private healthStatus = new Map<string, {
    isHealthy: boolean;
    lastCheck: Date;
    consecutiveFailures: number;
    latencyMs: number;
  }>();

  async checkHealth(provider: FallbackProvider): Promise<boolean> {
    const startTime = Date.now();

    try {
      const healthy = await provider.healthCheck();
      const latency = Date.now() - startTime;

      this.healthStatus.set(provider.name, {
        isHealthy: healthy,
        lastCheck: new Date(),
        consecutiveFailures: healthy ? 0 : (this.healthStatus.get(provider.name)?.consecutiveFailures || 0) + 1,
        latencyMs: latency
      });

      return healthy;
    } catch (error) {
      this.healthStatus.set(provider.name, {
        isHealthy: false,
        lastCheck: new Date(),
        consecutiveFailures: (this.healthStatus.get(provider.name)?.consecutiveFailures || 0) + 1,
        latencyMs: -1
      });

      return false;
    }
  }

  // Run periodic health checks
  startMonitoring(providers: FallbackProvider[], intervalMs: number = 30000): void {
    setInterval(async () => {
      for (const provider of providers) {
        await this.checkHealth(provider);
      }
    }, intervalMs);
  }

  getStatus(): Record<string, { healthy: boolean; latency: number }> {
    const status: Record<string, { healthy: boolean; latency: number }> = {};

    for (const [name, data] of this.healthStatus) {
      status[name] = {
        healthy: data.isHealthy,
        latency: data.latencyMs
      };
    }

    return status;
  }
}
```

## Alerting on Fallbacks

```typescript
class FallbackAlerts {
  async onFallback(
    primaryProvider: string,
    fallbackProvider: string,
    error: Error
  ): Promise<void> {
    // Log the event
    console.warn(`Fallback activated: ${primaryProvider} -> ${fallbackProvider}`);
    console.warn(`Reason: ${error.message}`);

    // Send alert if primary fails repeatedly
    const recentFallbacks = await this.getRecentFallbacks(primaryProvider);

    if (recentFallbacks > 5) {
      await this.sendAlert({
        severity: 'high',
        message: `Primary provider ${primaryProvider} has failed ${recentFallbacks} times in the last hour`,
        action: 'Investigate provider status'
      });
    }
  }

  async onAllProvidersFailed(): Promise<void> {
    await this.sendAlert({
      severity: 'critical',
      message: 'All LLM providers are failing - service degraded',
      action: 'Immediate investigation required'
    });
  }
}
```

## Best Practices

1. **Order by priority** - Best quality first, cheapest last
2. **Use circuit breakers** - Don't hammer failing providers
3. **Monitor health proactively** - Don't wait for failures
4. **Cache where possible** - Final fallback for availability
5. **Alert on degradation** - Know when you're running degraded
6. **Test failover regularly** - Chaos engineering for AI
7. **Document SLAs** - Know what you're guaranteeing
