---
name: model-routing-strategy
description: |
  Use this skill when implementing model selection for LLM applications. Activate when the user needs to
  choose between different AI models, implement cost-efficient model routing, balance quality vs cost,
  or build intelligent model selection systems.
---

# Model Routing Strategy

Dynamically select the optimal model for each task to balance quality, cost, and latency.

## When to Use

- Building applications using multiple LLM providers
- Optimizing costs while maintaining quality
- Need different model capabilities for different tasks
- Implementing fallback strategies
- A/B testing model performance

## Model Comparison Matrix

### Capability vs Cost

| Capability | Best Models | Cost Tier |
|------------|-------------|-----------|
| Complex reasoning | Claude Opus, o1 | $$$ |
| General tasks | Claude Sonnet, GPT-4o | $$ |
| Simple tasks | Claude Haiku, GPT-4o-mini | $ |
| Code generation | Claude Sonnet, GPT-4o | $$ |
| Creative writing | Claude Opus, GPT-4 | $$$ |
| Extraction/Classification | Claude Haiku, GPT-4o-mini | $ |

### Latency Comparison

| Model | Typical Latency (TTFB) |
|-------|------------------------|
| Claude Haiku | 200-400ms |
| Claude Sonnet | 400-800ms |
| Claude Opus | 800-1500ms |
| GPT-4o-mini | 200-400ms |
| GPT-4o | 400-700ms |
| GPT-4 Turbo | 500-1000ms |

## Routing Strategies

### Strategy 1: Complexity-Based Routing

```typescript
type Complexity = 'simple' | 'medium' | 'complex';

interface Task {
  prompt: string;
  requirements: {
    needsReasoning: boolean;
    needsCreativity: boolean;
    needsAccuracy: boolean;
    maxLatencyMs?: number;
    maxCostUSD?: number;
  };
}

function assessComplexity(task: Task): Complexity {
  const prompt = task.prompt.toLowerCase();

  // Complex indicators
  const complexPatterns = [
    /analyze.*and.*compare/,
    /explain.*step.*by.*step/,
    /write.*comprehensive/,
    /evaluate.*trade.*offs/,
    /design.*architecture/,
    /debug.*complex/,
    /review.*security/
  ];

  // Simple indicators
  const simplePatterns = [
    /summarize.*briefly/,
    /extract.*from/,
    /classify.*as/,
    /format.*as.*json/,
    /translate.*to/,
    /fix.*typo/
  ];

  if (complexPatterns.some(p => p.test(prompt)) ||
      task.requirements.needsReasoning ||
      task.requirements.needsCreativity) {
    return 'complex';
  }

  if (simplePatterns.some(p => p.test(prompt))) {
    return 'simple';
  }

  return 'medium';
}

function selectModel(complexity: Complexity): string {
  const modelMap = {
    simple: 'claude-3-haiku',     // $0.25/$1.25 per 1M
    medium: 'claude-3.5-sonnet',  // $3/$15 per 1M
    complex: 'claude-3-opus'      // $15/$75 per 1M
  };

  return modelMap[complexity];
}
```

### Strategy 2: Cost-Constrained Routing

```typescript
interface ModelOption {
  name: string;
  provider: string;
  inputCostPer1K: number;
  outputCostPer1K: number;
  qualityScore: number; // 0-1
  avgLatencyMs: number;
}

const models: ModelOption[] = [
  { name: 'claude-3-opus', provider: 'anthropic', inputCostPer1K: 0.015, outputCostPer1K: 0.075, qualityScore: 0.98, avgLatencyMs: 1200 },
  { name: 'claude-3.5-sonnet', provider: 'anthropic', inputCostPer1K: 0.003, outputCostPer1K: 0.015, qualityScore: 0.95, avgLatencyMs: 600 },
  { name: 'claude-3-haiku', provider: 'anthropic', inputCostPer1K: 0.00025, outputCostPer1K: 0.00125, qualityScore: 0.85, avgLatencyMs: 300 },
  { name: 'gpt-4o', provider: 'openai', inputCostPer1K: 0.005, outputCostPer1K: 0.015, qualityScore: 0.94, avgLatencyMs: 550 },
  { name: 'gpt-4o-mini', provider: 'openai', inputCostPer1K: 0.00015, outputCostPer1K: 0.0006, qualityScore: 0.82, avgLatencyMs: 300 },
];

function selectWithinBudget(
  estimatedInputTokens: number,
  estimatedOutputTokens: number,
  maxCost: number,
  minQuality: number = 0.8
): ModelOption | null {
  const viable = models
    .filter(m => {
      const cost = (estimatedInputTokens / 1000) * m.inputCostPer1K +
                   (estimatedOutputTokens / 1000) * m.outputCostPer1K;
      return cost <= maxCost && m.qualityScore >= minQuality;
    })
    .sort((a, b) => b.qualityScore - a.qualityScore); // Best quality within budget

  return viable[0] || null;
}
```

### Strategy 3: Latency-Optimized Routing

```typescript
function selectForLatency(
  maxLatencyMs: number,
  minQuality: number = 0.8
): ModelOption | null {
  return models
    .filter(m => m.avgLatencyMs <= maxLatencyMs && m.qualityScore >= minQuality)
    .sort((a, b) => b.qualityScore - a.qualityScore)[0] || null;
}

// For real-time applications
const realtimeModel = selectForLatency(500, 0.8); // Haiku or GPT-4o-mini

// For batch processing
const batchModel = selectForLatency(2000, 0.95); // Sonnet or Opus
```

### Strategy 4: Task-Type Routing

```typescript
type TaskType = 'code' | 'creative' | 'analysis' | 'extraction' | 'chat' | 'classification';

const taskModelMap: Record<TaskType, string[]> = {
  code: ['claude-3.5-sonnet', 'gpt-4o', 'claude-3-opus'],
  creative: ['claude-3-opus', 'gpt-4', 'claude-3.5-sonnet'],
  analysis: ['claude-3-opus', 'o1', 'claude-3.5-sonnet'],
  extraction: ['claude-3-haiku', 'gpt-4o-mini', 'claude-3.5-sonnet'],
  chat: ['claude-3.5-sonnet', 'gpt-4o', 'claude-3-haiku'],
  classification: ['claude-3-haiku', 'gpt-4o-mini', 'claude-3.5-sonnet']
};

function selectForTaskType(taskType: TaskType, budget: 'low' | 'medium' | 'high'): string {
  const candidates = taskModelMap[taskType];

  const budgetIndex = { low: 2, medium: 1, high: 0 };
  return candidates[Math.min(budgetIndex[budget], candidates.length - 1)];
}
```

## Fallback Chains

```typescript
interface FallbackChain {
  primary: string;
  fallbacks: string[];
  retryConfig: {
    maxRetries: number;
    backoffMs: number;
  };
}

const chains: Record<string, FallbackChain> = {
  highQuality: {
    primary: 'claude-3-opus',
    fallbacks: ['claude-3.5-sonnet', 'gpt-4o', 'claude-3-haiku'],
    retryConfig: { maxRetries: 3, backoffMs: 1000 }
  },
  costEffective: {
    primary: 'claude-3-haiku',
    fallbacks: ['gpt-4o-mini', 'claude-3.5-sonnet'],
    retryConfig: { maxRetries: 2, backoffMs: 500 }
  }
};

async function executeWithFallback(
  chain: FallbackChain,
  prompt: string
): Promise<string> {
  const allModels = [chain.primary, ...chain.fallbacks];

  for (let i = 0; i < allModels.length; i++) {
    const model = allModels[i];

    try {
      return await callModel(model, prompt);
    } catch (error) {
      console.log(`Model ${model} failed, trying fallback...`);

      if (i < allModels.length - 1) {
        await sleep(chain.retryConfig.backoffMs * (i + 1));
      }
    }
  }

  throw new Error('All models in fallback chain failed');
}
```

## Dynamic Routing with Learning

```typescript
interface ModelPerformance {
  model: string;
  successRate: number;
  avgLatency: number;
  avgQuality: number; // From human feedback or automated eval
  totalCalls: number;
}

class AdaptiveRouter {
  private performance = new Map<string, ModelPerformance>();

  recordOutcome(
    model: string,
    success: boolean,
    latencyMs: number,
    qualityScore?: number
  ): void {
    const perf = this.performance.get(model) || {
      model,
      successRate: 1,
      avgLatency: latencyMs,
      avgQuality: 0.9,
      totalCalls: 0
    };

    // Exponential moving average
    const alpha = 0.1;
    perf.successRate = alpha * (success ? 1 : 0) + (1 - alpha) * perf.successRate;
    perf.avgLatency = alpha * latencyMs + (1 - alpha) * perf.avgLatency;
    if (qualityScore !== undefined) {
      perf.avgQuality = alpha * qualityScore + (1 - alpha) * perf.avgQuality;
    }
    perf.totalCalls++;

    this.performance.set(model, perf);
  }

  selectBest(
    candidates: string[],
    weights: { quality: number; latency: number; reliability: number }
  ): string {
    let bestScore = -Infinity;
    let bestModel = candidates[0];

    for (const model of candidates) {
      const perf = this.performance.get(model);
      if (!perf) continue;

      const score =
        weights.quality * perf.avgQuality +
        weights.reliability * perf.successRate +
        weights.latency * (1 - perf.avgLatency / 5000); // Normalize to 0-1

      if (score > bestScore) {
        bestScore = score;
        bestModel = model;
      }
    }

    return bestModel;
  }
}
```

## A/B Testing Models

```typescript
interface ABTest {
  name: string;
  variants: { model: string; weight: number }[];
  metrics: string[];
}

class ModelABTester {
  private tests = new Map<string, ABTest>();
  private results = new Map<string, { model: string; metric: string; value: number }[]>();

  selectVariant(testName: string, userId: string): string {
    const test = this.tests.get(testName);
    if (!test) throw new Error(`Test ${testName} not found`);

    // Deterministic selection based on userId
    const hash = this.hashUserId(userId);
    let cumWeight = 0;

    for (const variant of test.variants) {
      cumWeight += variant.weight;
      if (hash < cumWeight) {
        return variant.model;
      }
    }

    return test.variants[0].model;
  }

  recordMetric(testName: string, model: string, metric: string, value: number): void {
    const results = this.results.get(testName) || [];
    results.push({ model, metric, value });
    this.results.set(testName, results);
  }

  getResults(testName: string): Record<string, Record<string, { avg: number; count: number }>> {
    const results = this.results.get(testName) || [];
    // Aggregate by model and metric
    // ...
  }
}
```

## Best Practices

1. **Start simple** - Complexity-based routing covers most cases
2. **Monitor quality** - Cheaper isn't better if quality drops
3. **Use fallbacks** - Always have a backup
4. **Test routing logic** - Unit test your router
5. **Log decisions** - Know why each model was chosen
6. **Review regularly** - Model capabilities and pricing change
7. **Consider latency** - User experience matters
