---
description: Analyze and optimize LLM API costs with 2026 pricing
---

# /cost-analyze

Analyze and optimize your LLM API costs with current 2026 pricing.

## What I Need

To analyze your costs, provide:

1. **Which models are you using?** (Claude, GPT-4, etc.)
2. **Approximate monthly usage** (requests or tokens)
3. **Your current monthly bill** (optional, for validation)

Or share your code that calls the LLM API and I'll estimate costs.

## Current Pricing (2026)

### Anthropic Claude

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| Claude Opus 4.5 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |

**Prompt Caching Discounts:**
- Cache write: 25% more than base
- Cache read: 90% discount (10% of base price)
- 5-minute TTL

### OpenAI

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|------------------------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| o1 | $15.00 | $60.00 |
| o1-mini | $3.00 | $12.00 |

## Cost Analysis Workflow

### Step 1: Calculate Current Costs

```typescript
// Example calculation
const usage = {
  model: 'claude-3.5-sonnet',
  requestsPerDay: 1000,
  avgInputTokens: 2000,
  avgOutputTokens: 500
};

const monthlyCost = calculateMonthlyCost(usage);
// Input: 1000 * 2000 * 30 = 60M tokens/month
// Output: 1000 * 500 * 30 = 15M tokens/month
// Cost: (60 * $3) + (15 * $15) = $180 + $225 = $405/month
```

### Step 2: Identify Optimization Opportunities

| Opportunity | Potential Savings |
|-------------|------------------|
| **Prompt Caching** | 50-90% on repeated prompts |
| **Model Routing** | 40-70% using smaller models for simple tasks |
| **Prompt Optimization** | 20-40% by reducing token usage |
| **Batching** | 50% with Batch API (24hr turnaround) |

### Step 3: Implement Optimizations

#### A. Prompt Caching (Anthropic)

```typescript
// Before: $3.00/M input tokens
const response = await anthropic.messages.create({
  model: 'claude-3.5-sonnet',
  system: longSystemPrompt,  // 4000 tokens, repeated every call
  messages: [{ role: 'user', content: userMessage }]
});

// After: $0.30/M for cached tokens (90% savings)
const response = await anthropic.messages.create({
  model: 'claude-3.5-sonnet',
  system: [
    {
      type: 'text',
      text: longSystemPrompt,
      cache_control: { type: 'ephemeral' }  // Cache this
    }
  ],
  messages: [{ role: 'user', content: userMessage }]
});
```

#### B. Model Routing

```typescript
function selectModel(task: string, complexity: number): string {
  if (complexity < 3) return 'claude-3-haiku';      // $0.80/M
  if (complexity < 7) return 'claude-3.5-sonnet';   // $3.00/M
  return 'claude-3-opus';                           // $15.00/M
}
```

#### C. Prompt Optimization

```markdown
Before (2000 tokens):
"You are a helpful AI assistant that helps users with their coding questions.
Please analyze the following code and provide detailed feedback about potential
issues, improvements, and best practices. Be thorough and comprehensive..."

After (500 tokens):
"Review this code. List: 1) bugs 2) improvements 3) security issues"
```

### Step 4: Projected Savings

I'll provide a comparison:

```
## Cost Optimization Report

### Current State
- Monthly requests: 30,000
- Model: claude-3.5-sonnet (all requests)
- Monthly cost: $405

### Optimized State
- Prompt caching: -$180 (repeated system prompts)
- Model routing: -$90 (60% routed to Haiku)
- Prompt optimization: -$40 (25% shorter prompts)

### Projected Monthly Cost: $95
### Monthly Savings: $310 (77%)
```

## Quick Wins

| Action | Effort | Savings |
|--------|--------|---------|
| Enable prompt caching | Low | 50-90% |
| Use Haiku for simple tasks | Low | 40-70% |
| Shorten system prompts | Medium | 20-30% |
| Use Batch API for async | Medium | 50% |
| Implement semantic caching | High | 60-80% |

---

**Share your usage details or API code and I'll analyze your costs.**
