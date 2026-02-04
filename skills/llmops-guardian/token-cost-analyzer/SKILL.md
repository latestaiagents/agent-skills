---
name: token-cost-analyzer
description: |
  Use this skill when analyzing and optimizing LLM API costs. Activate when the user wants to reduce
  AI API spending, understand token usage, audit LLM costs, optimize prompts for cost efficiency,
  or track and report on AI expenditure.
---

# Token Cost Analyzer

Audit, analyze, and optimize your LLM API spending.

## When to Use

- Monthly AI bill higher than expected
- Need to understand where tokens are spent
- Optimizing prompts for cost efficiency
- Setting up cost monitoring
- Budgeting for AI features

## Token Pricing Reference (2026)

### Anthropic Claude

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Claude 3 Opus | $15.00 | $75.00 |
| Claude 3.5 Sonnet | $3.00 | $15.00 |
| Claude 3 Haiku | $0.25 | $1.25 |

### OpenAI

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| GPT-4 Turbo | $10.00 | $30.00 |
| GPT-4o | $5.00 | $15.00 |
| GPT-4o mini | $0.15 | $0.60 |
| o1 | $15.00 | $60.00 |

### Google

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Gemini 1.5 Pro | $3.50 | $10.50 |
| Gemini 1.5 Flash | $0.075 | $0.30 |

## Cost Calculation

```typescript
interface TokenUsage {
  inputTokens: number;
  outputTokens: number;
  cachedTokens?: number;
}

interface ModelPricing {
  inputPer1M: number;
  outputPer1M: number;
  cachedPer1M?: number;
}

function calculateCost(usage: TokenUsage, pricing: ModelPricing): number {
  const inputCost = (usage.inputTokens / 1_000_000) * pricing.inputPer1M;
  const outputCost = (usage.outputTokens / 1_000_000) * pricing.outputPer1M;
  const cachedCost = usage.cachedTokens
    ? (usage.cachedTokens / 1_000_000) * (pricing.cachedPer1M || pricing.inputPer1M * 0.1)
    : 0;

  return inputCost + outputCost + cachedCost;
}

// Example
const usage = { inputTokens: 50000, outputTokens: 10000 };
const claude35Sonnet = { inputPer1M: 3.00, outputPer1M: 15.00 };
const cost = calculateCost(usage, claude35Sonnet);
// $0.15 + $0.15 = $0.30
```

## Cost Tracking System

```typescript
interface UsageRecord {
  timestamp: Date;
  model: string;
  operation: string;
  userId?: string;
  inputTokens: number;
  outputTokens: number;
  cost: number;
  metadata: Record<string, unknown>;
}

class CostTracker {
  private records: UsageRecord[] = [];

  record(usage: Omit<UsageRecord, 'timestamp' | 'cost'>): void {
    const pricing = this.getPricing(usage.model);
    const cost = calculateCost(
      { inputTokens: usage.inputTokens, outputTokens: usage.outputTokens },
      pricing
    );

    this.records.push({
      ...usage,
      timestamp: new Date(),
      cost
    });
  }

  // Aggregation methods
  getTotalCost(since: Date): number {
    return this.records
      .filter(r => r.timestamp >= since)
      .reduce((sum, r) => sum + r.cost, 0);
  }

  getCostByOperation(): Map<string, number> {
    const byOp = new Map<string, number>();
    for (const r of this.records) {
      byOp.set(r.operation, (byOp.get(r.operation) || 0) + r.cost);
    }
    return byOp;
  }

  getCostByModel(): Map<string, number> {
    const byModel = new Map<string, number>();
    for (const r of this.records) {
      byModel.set(r.model, (byModel.get(r.model) || 0) + r.cost);
    }
    return byModel;
  }

  getTopExpensiveOperations(limit: number = 10): UsageRecord[] {
    return [...this.records]
      .sort((a, b) => b.cost - a.cost)
      .slice(0, limit);
  }
}
```

## Cost Analysis Report

```typescript
interface CostReport {
  period: { start: Date; end: Date };
  summary: {
    totalCost: number;
    totalInputTokens: number;
    totalOutputTokens: number;
    uniqueOperations: number;
    avgCostPerRequest: number;
  };
  breakdown: {
    byModel: { model: string; cost: number; percentage: number }[];
    byOperation: { operation: string; cost: number; requests: number }[];
    byDay: { date: string; cost: number }[];
  };
  insights: {
    mostExpensiveOperation: string;
    fastestGrowingCost: string;
    optimizationOpportunities: string[];
  };
}

function generateCostReport(
  records: UsageRecord[],
  period: { start: Date; end: Date }
): CostReport {
  const filtered = records.filter(
    r => r.timestamp >= period.start && r.timestamp <= period.end
  );

  const totalCost = filtered.reduce((sum, r) => sum + r.cost, 0);

  // Group by model
  const byModel = new Map<string, number>();
  for (const r of filtered) {
    byModel.set(r.model, (byModel.get(r.model) || 0) + r.cost);
  }

  // Group by operation
  const byOperation = new Map<string, { cost: number; count: number }>();
  for (const r of filtered) {
    const existing = byOperation.get(r.operation) || { cost: 0, count: 0 };
    byOperation.set(r.operation, {
      cost: existing.cost + r.cost,
      count: existing.count + 1
    });
  }

  // Find optimization opportunities
  const opportunities: string[] = [];

  // Check for expensive model usage on simple tasks
  const haiku = byModel.get('claude-3-haiku') || 0;
  const opus = byModel.get('claude-3-opus') || 0;
  if (opus > haiku * 10) {
    opportunities.push('Consider using Haiku for simpler tasks - Opus usage is 10x higher');
  }

  // Check for high output token ratio
  const totalInput = filtered.reduce((s, r) => s + r.inputTokens, 0);
  const totalOutput = filtered.reduce((s, r) => s + r.outputTokens, 0);
  if (totalOutput > totalInput * 2) {
    opportunities.push('High output ratio - consider asking for more concise responses');
  }

  return {
    period,
    summary: {
      totalCost,
      totalInputTokens: totalInput,
      totalOutputTokens: totalOutput,
      uniqueOperations: byOperation.size,
      avgCostPerRequest: totalCost / filtered.length
    },
    breakdown: {
      byModel: Array.from(byModel.entries()).map(([model, cost]) => ({
        model,
        cost,
        percentage: (cost / totalCost) * 100
      })),
      byOperation: Array.from(byOperation.entries()).map(([op, data]) => ({
        operation: op,
        cost: data.cost,
        requests: data.count
      })),
      byDay: groupByDay(filtered)
    },
    insights: {
      mostExpensiveOperation: [...byOperation.entries()]
        .sort((a, b) => b[1].cost - a[1].cost)[0]?.[0] || 'none',
      fastestGrowingCost: calculateGrowth(filtered),
      optimizationOpportunities: opportunities
    }
  };
}
```

## Optimization Strategies

### 1. Model Selection

```typescript
type TaskComplexity = 'simple' | 'medium' | 'complex';

function selectCostEffectiveModel(
  task: string,
  complexity: TaskComplexity
): string {
  const modelMap = {
    simple: 'gpt-4o-mini',    // $0.15/$0.60 per 1M
    medium: 'claude-3-haiku', // $0.25/$1.25 per 1M
    complex: 'claude-3.5-sonnet' // $3/$15 per 1M
  };

  return modelMap[complexity];
}

// Auto-detect complexity
function assessComplexity(task: string): TaskComplexity {
  const complexIndicators = ['analyze', 'compare', 'synthesize', 'evaluate', 'complex'];
  const simpleIndicators = ['format', 'extract', 'classify', 'summarize short'];

  const lower = task.toLowerCase();

  if (complexIndicators.some(i => lower.includes(i))) return 'complex';
  if (simpleIndicators.some(i => lower.includes(i))) return 'simple';
  return 'medium';
}
```

### 2. Prompt Optimization

```typescript
// Before: Verbose prompt (many input tokens)
const verbosePrompt = `
You are an expert assistant. Your task is to help users with their questions.
Please be thorough and comprehensive in your responses. Make sure to consider
all aspects of the question and provide detailed explanations. If you're unsure
about something, please say so. Always be polite and professional.

User question: ${question}
`;

// After: Concise prompt (fewer input tokens)
const concisePrompt = `Answer concisely: ${question}`;

// Savings: ~80% reduction in input tokens
```

### 3. Response Length Control

```typescript
// Instruct the model to be concise
const prompt = `
${task}

Respond in under 100 words. Use bullet points.
`;

// Or use max_tokens parameter
const response = await client.messages.create({
  model: 'claude-3-haiku',
  max_tokens: 500, // Limit output
  messages: [{ role: 'user', content: prompt }]
});
```

### 4. Caching

```typescript
// Use Anthropic prompt caching
const response = await client.messages.create({
  model: 'claude-3-sonnet',
  messages: [{
    role: 'user',
    content: [
      {
        type: 'text',
        text: longSystemContext,
        cache_control: { type: 'ephemeral' } // Cache this
      },
      { type: 'text', text: userQuestion }
    ]
  }]
});

// Cached tokens cost 90% less
```

## Alerting

```typescript
interface CostAlert {
  type: 'threshold' | 'spike' | 'anomaly';
  threshold: number;
  action: 'notify' | 'throttle' | 'block';
}

class CostAlertSystem {
  private alerts: CostAlert[] = [
    { type: 'threshold', threshold: 100, action: 'notify' },   // $100/day
    { type: 'threshold', threshold: 500, action: 'throttle' }, // $500/day
    { type: 'spike', threshold: 5, action: 'notify' }          // 5x normal
  ];

  async checkAlerts(currentCost: number, historicalAvg: number): Promise<void> {
    for (const alert of this.alerts) {
      if (alert.type === 'threshold' && currentCost > alert.threshold) {
        await this.triggerAlert(alert, `Daily cost $${currentCost} exceeds $${alert.threshold}`);
      }

      if (alert.type === 'spike' && currentCost > historicalAvg * alert.threshold) {
        await this.triggerAlert(alert, `Cost spike: $${currentCost} is ${(currentCost/historicalAvg).toFixed(1)}x normal`);
      }
    }
  }
}
```

## Dashboard Metrics

```typescript
interface CostDashboard {
  // Real-time
  currentHourCost: number;
  currentDayCost: number;
  currentMonthCost: number;

  // Trends
  costTrend7d: { date: string; cost: number }[];
  projectedMonthEnd: number;

  // Breakdown
  topOperations: { name: string; cost: number; trend: 'up' | 'down' | 'stable' }[];
  modelMix: { model: string; percentage: number }[];

  // Alerts
  activeAlerts: string[];
  budgetUtilization: number;
}
```

## Best Practices

1. **Track everything** - Log all API calls with metadata
2. **Set budgets early** - Don't wait for the first big bill
3. **Use tiered models** - Not every task needs the best model
4. **Optimize prompts** - Fewer tokens = lower cost
5. **Cache when possible** - Reuse expensive computations
6. **Review regularly** - Weekly cost reviews catch issues early
7. **Alert on anomalies** - Catch runaway costs immediately
