---
name: agent-cost-budgeting
description: |
  Use this skill when managing AI agent costs. Activate when the user needs to control token usage,
  implement cost limits for agents, optimize LLM spending, track agent costs, or prevent runaway
  API bills in agent systems.
---

# Agent Cost Budgeting

Control and optimize token usage across AI agent systems.

## When to Use

- Preventing runaway API costs
- Implementing per-task cost limits
- Optimizing multi-agent token usage
- Tracking and reporting AI spending
- Building cost-aware agent behaviors

## Cost Model

```typescript
interface CostModel {
  provider: string;
  model: string;
  inputCostPer1K: number;  // $ per 1000 input tokens
  outputCostPer1K: number; // $ per 1000 output tokens
  cacheCostPer1K?: number; // $ per 1000 cached tokens (if supported)
}

const COST_MODELS: Record<string, CostModel> = {
  'claude-3-opus': {
    provider: 'anthropic',
    model: 'claude-3-opus-20240229',
    inputCostPer1K: 0.015,
    outputCostPer1K: 0.075
  },
  'claude-3-sonnet': {
    provider: 'anthropic',
    model: 'claude-3-sonnet-20240229',
    inputCostPer1K: 0.003,
    outputCostPer1K: 0.015
  },
  'claude-3-haiku': {
    provider: 'anthropic',
    model: 'claude-3-haiku-20240307',
    inputCostPer1K: 0.00025,
    outputCostPer1K: 0.00125
  },
  'gpt-4-turbo': {
    provider: 'openai',
    model: 'gpt-4-turbo',
    inputCostPer1K: 0.01,
    outputCostPer1K: 0.03
  },
  'gpt-4o': {
    provider: 'openai',
    model: 'gpt-4o',
    inputCostPer1K: 0.005,
    outputCostPer1K: 0.015
  },
  'gpt-4o-mini': {
    provider: 'openai',
    model: 'gpt-4o-mini',
    inputCostPer1K: 0.00015,
    outputCostPer1K: 0.0006
  }
};

function calculateCost(
  model: string,
  inputTokens: number,
  outputTokens: number
): number {
  const costModel = COST_MODELS[model];
  if (!costModel) throw new Error(`Unknown model: ${model}`);

  return (
    (inputTokens / 1000) * costModel.inputCostPer1K +
    (outputTokens / 1000) * costModel.outputCostPer1K
  );
}
```

## Budget Management

```typescript
interface Budget {
  id: string;
  name: string;
  limitUSD: number;
  spentUSD: number;
  period: 'task' | 'hourly' | 'daily' | 'monthly';
  resetAt?: Date;
  alertThresholds: number[]; // e.g., [0.5, 0.8, 0.95]
  hardLimit: boolean; // Stop vs warn at limit
}

class BudgetManager {
  private budgets = new Map<string, Budget>();

  async checkBudget(budgetId: string, estimatedCost: number): Promise<BudgetCheck> {
    const budget = this.budgets.get(budgetId);
    if (!budget) return { allowed: true };

    const remaining = budget.limitUSD - budget.spentUSD;
    const newTotal = budget.spentUSD + estimatedCost;
    const utilizationAfter = newTotal / budget.limitUSD;

    // Check thresholds
    const crossedThresholds = budget.alertThresholds.filter(
      t => budget.spentUSD / budget.limitUSD < t && utilizationAfter >= t
    );

    if (crossedThresholds.length > 0) {
      await this.sendAlerts(budget, crossedThresholds);
    }

    // Check limit
    if (estimatedCost > remaining) {
      if (budget.hardLimit) {
        return {
          allowed: false,
          reason: `Budget exceeded: ${remaining.toFixed(4)} USD remaining`,
          remaining
        };
      } else {
        return {
          allowed: true,
          warning: `Budget will be exceeded`,
          remaining
        };
      }
    }

    return { allowed: true, remaining };
  }

  async recordSpend(budgetId: string, cost: number): Promise<void> {
    const budget = this.budgets.get(budgetId);
    if (!budget) return;

    budget.spentUSD += cost;

    // Check if period should reset
    if (budget.resetAt && new Date() >= budget.resetAt) {
      budget.spentUSD = cost; // Start fresh with current spend
      budget.resetAt = this.calculateNextReset(budget.period);
    }
  }
}
```

## Token Estimation

```typescript
// Rough estimation before API call
function estimateTokens(text: string): number {
  // Rough heuristic: ~4 characters per token for English
  return Math.ceil(text.length / 4);
}

// More accurate estimation using tiktoken (for OpenAI)
import { encoding_for_model } from 'tiktoken';

function countTokensAccurate(text: string, model: string): number {
  const enc = encoding_for_model(model);
  const tokens = enc.encode(text);
  enc.free();
  return tokens.length;
}

// Estimate cost before execution
function estimateCallCost(
  model: string,
  systemPrompt: string,
  userMessage: string,
  expectedOutputTokens: number
): number {
  const inputTokens = estimateTokens(systemPrompt + userMessage);
  return calculateCost(model, inputTokens, expectedOutputTokens);
}
```

## Cost-Aware Agent

```typescript
class CostAwareAgent {
  private budget: Budget;
  private spent = 0;

  constructor(budgetUSD: number) {
    this.budget = {
      id: 'agent-budget',
      name: 'Agent Task Budget',
      limitUSD: budgetUSD,
      spentUSD: 0,
      period: 'task',
      alertThresholds: [0.5, 0.8],
      hardLimit: true
    };
  }

  async execute(task: string): Promise<Result> {
    // Estimate cost
    const estimate = this.estimateTaskCost(task);

    if (estimate > this.remaining) {
      return this.handleBudgetExceeded(task, estimate);
    }

    // Choose model based on budget
    const model = this.selectModelForBudget(task);

    // Execute with tracking
    const result = await this.llm.complete({
      model,
      messages: [{ role: 'user', content: task }],
      onUsage: (usage) => this.recordUsage(usage, model)
    });

    return result;
  }

  private selectModelForBudget(task: string): string {
    const complexity = this.assessComplexity(task);
    const remaining = this.budget.limitUSD - this.spent;

    // Use cheaper models when budget is tight
    if (remaining < 0.10) {
      return 'gpt-4o-mini'; // Cheapest
    }

    if (remaining < 0.50 || complexity === 'low') {
      return 'claude-3-haiku';
    }

    if (remaining < 2.00 || complexity === 'medium') {
      return 'claude-3-sonnet';
    }

    return 'claude-3-opus'; // Full power when budget allows
  }

  private handleBudgetExceeded(task: string, estimate: number): Result {
    // Options:
    // 1. Simplify the task
    // 2. Use cheaper model
    // 3. Return partial result
    // 4. Request budget increase

    const cheaperModel = this.findCheapestViableModel(task);
    if (cheaperModel) {
      return this.execute(task); // Retry with cheaper model
    }

    return {
      success: false,
      error: 'Budget exceeded',
      partialResult: null,
      budgetInfo: {
        remaining: this.remaining,
        estimated: estimate
      }
    };
  }

  get remaining(): number {
    return this.budget.limitUSD - this.spent;
  }
}
```

## Multi-Agent Cost Distribution

```typescript
interface AgentCostAllocation {
  agentId: string;
  allocatedUSD: number;
  spentUSD: number;
  priority: 'low' | 'medium' | 'high';
}

class MultiAgentBudgetManager {
  private totalBudget: number;
  private allocations = new Map<string, AgentCostAllocation>();

  allocateBudget(agents: { id: string; priority: string }[]): void {
    // Priority weights
    const weights = { high: 3, medium: 2, low: 1 };

    const totalWeight = agents.reduce(
      (sum, a) => sum + weights[a.priority], 0
    );

    for (const agent of agents) {
      const share = (weights[agent.priority] / totalWeight) * this.totalBudget;
      this.allocations.set(agent.id, {
        agentId: agent.id,
        allocatedUSD: share,
        spentUSD: 0,
        priority: agent.priority as any
      });
    }
  }

  // Reallocate from under-spending to over-spending agents
  rebalance(): void {
    const underSpenders = Array.from(this.allocations.values())
      .filter(a => a.spentUSD < a.allocatedUSD * 0.5);

    const overSpenders = Array.from(this.allocations.values())
      .filter(a => a.spentUSD > a.allocatedUSD * 0.8);

    for (const over of overSpenders) {
      const needed = over.spentUSD - over.allocatedUSD * 0.8;

      for (const under of underSpenders) {
        const available = under.allocatedUSD * 0.5 - under.spentUSD;
        const transfer = Math.min(needed, available);

        if (transfer > 0) {
          under.allocatedUSD -= transfer;
          over.allocatedUSD += transfer;
        }
      }
    }
  }
}
```

## Cost Optimization Strategies

### 1. Prompt Caching

```typescript
// Anthropic prompt caching
async function callWithCaching(messages: Message[]): Promise<Response> {
  // Mark system prompt for caching
  const cachedMessages = messages.map((m, i) => {
    if (i === 0 && m.role === 'system') {
      return {
        ...m,
        cache_control: { type: 'ephemeral' }
      };
    }
    return m;
  });

  return anthropic.messages.create({
    model: 'claude-3-sonnet-20240229',
    messages: cachedMessages
  });
}
```

### 2. Model Routing

```typescript
function routeToOptimalModel(task: string, budget: number): string {
  const complexity = assessComplexity(task);

  // Complexity vs cost matrix
  const modelMatrix = {
    simple: ['gpt-4o-mini', 'claude-3-haiku'],
    medium: ['claude-3-sonnet', 'gpt-4o'],
    complex: ['claude-3-opus', 'gpt-4-turbo']
  };

  const candidates = modelMatrix[complexity];

  // Find cheapest that fits budget
  for (const model of candidates) {
    const estimatedCost = estimateCallCost(model, task, 500);
    if (estimatedCost <= budget) {
      return model;
    }
  }

  return candidates[candidates.length - 1]; // Fallback to cheapest
}
```

### 3. Context Compression

```typescript
async function compressContext(
  context: string,
  targetTokens: number
): Promise<string> {
  const currentTokens = estimateTokens(context);

  if (currentTokens <= targetTokens) {
    return context;
  }

  // Use cheap model to summarize
  const summary = await llm.complete({
    model: 'gpt-4o-mini',
    messages: [{
      role: 'user',
      content: `Summarize this context in under ${targetTokens} tokens, preserving key information:\n\n${context}`
    }]
  });

  return summary;
}
```

## Reporting

```typescript
interface CostReport {
  period: { start: Date; end: Date };
  totalSpent: number;
  byModel: Record<string, { calls: number; tokens: number; cost: number }>;
  byAgent: Record<string, { calls: number; cost: number }>;
  byTask: Record<string, { cost: number; success: boolean }>;
  trends: {
    dailyAverage: number;
    projectedMonthly: number;
    topCostDrivers: string[];
  };
}

function generateCostReport(usageData: UsageRecord[]): CostReport {
  // ... aggregation logic
}
```

## Best Practices

1. **Set budgets early** - Don't wait for the bill
2. **Monitor in real-time** - Not after the fact
3. **Use tiered models** - Right-size for the task
4. **Cache aggressively** - Reuse where possible
5. **Compress context** - Less tokens = less cost
6. **Alert early** - 80% threshold, not 100%
7. **Track by task** - Know what's expensive
8. **Review regularly** - Optimize based on data
