---
name: agent-testing-harness
description: |
  Use this skill when testing AI agent systems. Activate when the user needs to test agent behavior,
  write tests for multi-agent systems, implement agent evaluation frameworks, create test harnesses
  for autonomous agents, or validate agent outputs systematically.
---

# Agent Testing Harness

Design and implement comprehensive testing for AI agent systems.

## When to Use

- Building reliable agent systems
- Validating agent behavior before deployment
- Creating regression tests for agents
- Implementing continuous testing for agent updates
- Evaluating agent performance metrics

## Testing Pyramid for Agents

```
        ┌─────────────────────┐
        │   End-to-End        │  Full workflow tests
        │   Agent Tests       │  (expensive, few)
        ├─────────────────────┤
        │   Integration       │  Multi-agent interaction
        │   Tests             │  (medium cost, some)
        ├─────────────────────┤
        │   Component         │  Individual agent behavior
        │   Tests             │  (cheap, many)
        ├─────────────────────┤
        │   Unit Tests        │  Tools, utilities, helpers
        │                     │  (cheapest, most)
        └─────────────────────┘
```

## Unit Testing Agent Components

### Testing Tools

```typescript
import { describe, it, expect, vi } from 'vitest';

describe('SearchTool', () => {
  const searchTool = new SearchTool();

  it('returns results for valid query', async () => {
    const result = await searchTool.execute({ query: 'test query' });

    expect(result.success).toBe(true);
    expect(result.data.results).toBeInstanceOf(Array);
    expect(result.data.results.length).toBeGreaterThan(0);
  });

  it('handles empty query gracefully', async () => {
    const result = await searchTool.execute({ query: '' });

    expect(result.success).toBe(false);
    expect(result.error.code).toBe('INVALID_INPUT');
  });

  it('respects rate limits', async () => {
    // Make many requests quickly
    const promises = Array(10).fill(null).map(() =>
      searchTool.execute({ query: 'test' })
    );

    const results = await Promise.all(promises);
    const rateLimited = results.filter(r => r.error?.code === 'RATE_LIMITED');

    expect(rateLimited.length).toBeGreaterThan(0);
  });
});
```

### Testing Prompts

```typescript
describe('SystemPrompt', () => {
  it('includes all required sections', () => {
    const prompt = generateSystemPrompt(config);

    expect(prompt).toContain('## Your Role');
    expect(prompt).toContain('## Available Tools');
    expect(prompt).toContain('## Constraints');
  });

  it('correctly formats tool descriptions', () => {
    const prompt = generateSystemPrompt({
      tools: [
        { name: 'search', description: 'Search the web' },
        { name: 'calculate', description: 'Do math' }
      ]
    });

    expect(prompt).toContain('- search: Search the web');
    expect(prompt).toContain('- calculate: Do math');
  });
});
```

## Component Testing Agents

### Mock LLM Responses

```typescript
class MockLLM {
  private responses: Map<string, string> = new Map();

  setResponse(inputPattern: RegExp | string, response: string): void {
    const key = inputPattern instanceof RegExp ? inputPattern.source : inputPattern;
    this.responses.set(key, response);
  }

  async complete(input: string): Promise<string> {
    for (const [pattern, response] of this.responses) {
      const regex = new RegExp(pattern, 'i');
      if (regex.test(input)) {
        return response;
      }
    }
    return 'Default mock response';
  }
}

describe('ResearchAgent', () => {
  let agent: ResearchAgent;
  let mockLLM: MockLLM;

  beforeEach(() => {
    mockLLM = new MockLLM();
    agent = new ResearchAgent({ llm: mockLLM });
  });

  it('formulates search queries from task', async () => {
    mockLLM.setResponse(
      /formulate.*search/i,
      JSON.stringify({ queries: ['query 1', 'query 2'] })
    );

    const result = await agent.planResearch('Find information about X');

    expect(result.queries).toHaveLength(2);
  });

  it('synthesizes findings into report', async () => {
    mockLLM.setResponse(
      /synthesize/i,
      'Based on the research, here are the key findings...'
    );

    const result = await agent.synthesize([
      { source: 'source1', content: 'finding 1' },
      { source: 'source2', content: 'finding 2' }
    ]);

    expect(result).toContain('key findings');
  });
});
```

### Testing Agent Decision Making

```typescript
describe('Agent Decision Making', () => {
  it('selects appropriate tool for task', async () => {
    const agent = new Agent({
      tools: [searchTool, calculatorTool, fileReaderTool]
    });

    // Math task should use calculator
    const mathDecision = await agent.decideTool('Calculate 15% of 200');
    expect(mathDecision.tool).toBe('calculator');

    // Search task should use search
    const searchDecision = await agent.decideTool('Find the latest news about AI');
    expect(searchDecision.tool).toBe('search');
  });

  it('handles ambiguous tasks appropriately', async () => {
    const agent = new Agent({ tools: [searchTool, fileReaderTool] });

    const decision = await agent.decideTool('Read about quantum computing');

    // Should clarify or make reasonable choice
    expect(['search', 'clarify']).toContain(decision.action);
  });
});
```

## Integration Testing Multi-Agent Systems

```typescript
describe('Multi-Agent Workflow', () => {
  let supervisor: SupervisorAgent;
  let researcher: ResearchAgent;
  let writer: WriterAgent;
  let reviewer: ReviewerAgent;

  beforeEach(() => {
    researcher = new ResearchAgent();
    writer = new WriterAgent();
    reviewer = new ReviewerAgent();
    supervisor = new SupervisorAgent({
      workers: [researcher, writer, reviewer]
    });
  });

  it('coordinates agents to complete task', async () => {
    const result = await supervisor.execute(
      'Write a blog post about renewable energy'
    );

    expect(result.success).toBe(true);
    expect(result.steps).toContainEqual(
      expect.objectContaining({ agent: 'researcher', status: 'completed' })
    );
    expect(result.steps).toContainEqual(
      expect.objectContaining({ agent: 'writer', status: 'completed' })
    );
  });

  it('handles agent failure gracefully', async () => {
    // Make researcher fail
    vi.spyOn(researcher, 'execute').mockRejectedValue(new Error('API Error'));

    const result = await supervisor.execute('Research topic X');

    expect(result.success).toBe(false);
    expect(result.error).toContain('researcher failed');
    expect(result.recoveryAttempts).toBeGreaterThan(0);
  });

  it('respects budget constraints', async () => {
    const result = await supervisor.execute(
      'Complex research task',
      { budgetUSD: 0.01 } // Very low budget
    );

    expect(result.totalCost).toBeLessThanOrEqual(0.01);
  });
});
```

## End-to-End Agent Tests

```typescript
describe('E2E: Content Creation Pipeline', () => {
  // Use real LLM but with test account
  const agent = new ContentCreationAgent({
    llm: new OpenAI({ apiKey: process.env.TEST_API_KEY })
  });

  it('creates blog post from topic', async () => {
    const result = await agent.createContent({
      type: 'blog_post',
      topic: 'Benefits of unit testing',
      targetLength: 500
    });

    // Structure validation
    expect(result.title).toBeDefined();
    expect(result.content.length).toBeGreaterThan(400);
    expect(result.content.length).toBeLessThan(600);

    // Content validation
    expect(result.content.toLowerCase()).toContain('test');
    expect(result.sections.length).toBeGreaterThanOrEqual(3);
  }, 60000); // Long timeout for real API calls

  it('handles user feedback loop', async () => {
    const draft = await agent.createContent({
      type: 'blog_post',
      topic: 'AI in healthcare'
    });

    const revised = await agent.reviseContent(draft, {
      feedback: 'Make it more technical and add statistics'
    });

    expect(revised.content).not.toEqual(draft.content);
    // Check for more technical language
    expect(revised.content).toMatch(/\d+%|\d+ percent/);
  }, 120000);
});
```

## Evaluation Metrics

```typescript
interface AgentEvaluation {
  taskCompletion: number;      // 0-1: Did it complete the task?
  accuracy: number;            // 0-1: Is the output correct?
  efficiency: number;          // 0-1: Token/time efficiency
  safety: number;              // 0-1: No harmful outputs
  reliability: number;         // 0-1: Consistent results
}

class AgentEvaluator {
  async evaluate(
    agent: Agent,
    testCases: TestCase[]
  ): Promise<EvaluationReport> {
    const results: EvaluationResult[] = [];

    for (const testCase of testCases) {
      const startTime = Date.now();
      const result = await agent.execute(testCase.input);
      const duration = Date.now() - startTime;

      const evaluation: AgentEvaluation = {
        taskCompletion: this.assessCompletion(result, testCase.expected),
        accuracy: this.assessAccuracy(result, testCase.expected),
        efficiency: this.assessEfficiency(result, duration),
        safety: this.assessSafety(result),
        reliability: 1 // Will be calculated across runs
      };

      results.push({ testCase, result, evaluation, duration });
    }

    // Run reliability tests (same input multiple times)
    const reliabilityScores = await this.testReliability(agent, testCases);

    return this.compileReport(results, reliabilityScores);
  }

  private assessCompletion(result: Result, expected: Expected): number {
    if (!result.success) return 0;

    // Check required outputs are present
    const requiredKeys = Object.keys(expected.requiredOutputs || {});
    const presentKeys = requiredKeys.filter(k => result.output[k] !== undefined);

    return presentKeys.length / Math.max(requiredKeys.length, 1);
  }

  private assessAccuracy(result: Result, expected: Expected): number {
    if (!expected.groundTruth) return 1; // No ground truth to compare

    // Use LLM to judge similarity
    return this.llmJudge(result.output, expected.groundTruth);
  }
}
```

## Test Data Management

```typescript
interface TestCase {
  id: string;
  name: string;
  category: string;
  input: AgentInput;
  expected: {
    success: boolean;
    requiredOutputs?: Record<string, any>;
    groundTruth?: string;
    maxDurationMs?: number;
    maxCostUSD?: number;
  };
  tags: string[];
}

// Test case factory
function createTestCase(
  name: string,
  input: string,
  expected: Partial<TestCase['expected']>
): TestCase {
  return {
    id: generateId(),
    name,
    category: 'default',
    input: { task: input },
    expected: {
      success: true,
      ...expected
    },
    tags: []
  };
}

// Example test suite
const codeReviewTestCases: TestCase[] = [
  createTestCase(
    'Identifies security vulnerability',
    'Review this code: eval(userInput)',
    {
      requiredOutputs: {
        vulnerabilities: expect.arrayContaining([
          expect.objectContaining({ type: 'code_injection' })
        ])
      }
    }
  ),
  createTestCase(
    'Catches null pointer risk',
    'Review: const name = user.profile.name',
    {
      requiredOutputs: {
        warnings: expect.arrayContaining([
          expect.objectContaining({ type: 'null_safety' })
        ])
      }
    }
  )
];
```

## Continuous Testing Pipeline

```yaml
# .github/workflows/agent-tests.yml
name: Agent Tests

on:
  push:
    paths:
      - 'agents/**'
      - 'prompts/**'
  schedule:
    - cron: '0 0 * * *'  # Daily regression

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- --grep "unit"

  component-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- --grep "component"
        env:
          MOCK_LLM: true

  integration-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, component-tests]
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- --grep "integration"
        env:
          TEST_API_KEY: ${{ secrets.TEST_API_KEY }}

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - run: npm test -- --grep "e2e"
        env:
          TEST_API_KEY: ${{ secrets.TEST_API_KEY }}
```

## Best Practices

1. **Mock at boundaries** - Mock LLM, not agent logic
2. **Test deterministically** - Set seeds, use fixed responses
3. **Measure what matters** - Completion, accuracy, safety
4. **Automate regression** - Catch prompt regressions
5. **Test failure modes** - Agents should fail gracefully
6. **Budget for tests** - Real API tests cost money
7. **Version test cases** - Track alongside prompt changes
8. **Use golden outputs** - Compare against known-good results
