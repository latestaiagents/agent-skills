---
name: agent-tool-routing
description: |
  Use this skill when implementing tool selection for AI agents. Activate when the user needs agents to
  choose the right tools, implement dynamic tool routing, integrate MCP servers, design tool selection
  logic, or build agents that can use external services effectively.
---

# Agent Tool Routing

Design intelligent systems for agents to select and use the right tools at the right time.

## When to Use

- Agents need to choose between multiple tools
- Implementing MCP (Model Context Protocol) integrations
- Building agents with external API access
- Designing tool fallback strategies
- Optimizing tool usage for cost/performance

## Tool Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         AGENT                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     TOOL ROUTER                              │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐ │
│  │ Classifier  │  │ Capabilities │  │ Cost/Latency       │ │
│  │             │  │ Matcher      │  │ Optimizer          │ │
│  └─────────────┘  └──────────────┘  └────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   ┌─────────┐          ┌─────────┐          ┌─────────┐
   │ Tool A  │          │ Tool B  │          │ Tool C  │
   │ (Local) │          │  (API)  │          │  (MCP)  │
   └─────────┘          └─────────┘          └─────────┘
```

## Tool Definition

```typescript
interface Tool {
  name: string;
  description: string;
  category: string;

  // Schema
  inputSchema: JSONSchema;
  outputSchema: JSONSchema;

  // Capabilities
  capabilities: string[];
  limitations: string[];

  // Execution
  execute: (input: unknown) => Promise<ToolResult>;

  // Metadata
  metadata: {
    costPerCall?: number;
    avgLatencyMs?: number;
    rateLimit?: RateLimit;
    requiresAuth?: boolean;
    supportsBatching?: boolean;
  };
}

interface ToolResult {
  success: boolean;
  data?: unknown;
  error?: {
    code: string;
    message: string;
    retryable: boolean;
  };
  metadata: {
    durationMs: number;
    tokensUsed?: number;
  };
}
```

## Tool Registry

```typescript
class ToolRegistry {
  private tools = new Map<string, Tool>();
  private capabilityIndex = new Map<string, Set<string>>();

  register(tool: Tool): void {
    this.tools.set(tool.name, tool);

    // Index by capability
    for (const cap of tool.capabilities) {
      if (!this.capabilityIndex.has(cap)) {
        this.capabilityIndex.set(cap, new Set());
      }
      this.capabilityIndex.get(cap)!.add(tool.name);
    }
  }

  findByCapability(capability: string): Tool[] {
    const toolNames = this.capabilityIndex.get(capability) || new Set();
    return Array.from(toolNames).map(name => this.tools.get(name)!);
  }

  getAll(): Tool[] {
    return Array.from(this.tools.values());
  }

  // Generate tool descriptions for LLM
  getToolDescriptions(): string {
    return this.getAll()
      .map(t => `- ${t.name}: ${t.description}`)
      .join('\n');
  }
}
```

## Router Strategies

### Strategy 1: LLM-Based Selection

Let the model choose based on descriptions.

```typescript
class LLMToolRouter {
  async route(
    task: string,
    availableTools: Tool[]
  ): Promise<RoutingDecision> {
    const response = await this.llm.complete({
      system: `You are a tool routing assistant.
Given a task and available tools, select the best tool to use.

Available tools:
${availableTools.map(t => `
- ${t.name}
  Description: ${t.description}
  Capabilities: ${t.capabilities.join(', ')}
  Cost: ${t.metadata.costPerCall || 'free'}
  Latency: ${t.metadata.avgLatencyMs || 'unknown'}ms
`).join('\n')}

Respond with JSON: { "tool": "tool_name", "reasoning": "why", "input": {...} }`,
      user: `Task: ${task}`
    });

    return JSON.parse(response);
  }
}
```

### Strategy 2: Rule-Based Selection

Deterministic routing based on patterns.

```typescript
class RuleBasedRouter {
  private rules: RoutingRule[] = [];

  addRule(rule: RoutingRule): void {
    this.rules.push(rule);
    this.rules.sort((a, b) => b.priority - a.priority);
  }

  route(task: string, context: Context): RoutingDecision {
    for (const rule of this.rules) {
      if (rule.matches(task, context)) {
        return {
          tool: rule.targetTool,
          reasoning: rule.description
        };
      }
    }

    return { tool: 'default', reasoning: 'No specific rule matched' };
  }
}

// Example rules
const rules: RoutingRule[] = [
  {
    priority: 100,
    description: 'Use web search for current information',
    matches: (task) => /current|latest|today|news|2024|2025|2026/.test(task),
    targetTool: 'web_search'
  },
  {
    priority: 90,
    description: 'Use code execution for calculations',
    matches: (task) => /calculate|compute|sum|average|math/.test(task),
    targetTool: 'code_interpreter'
  },
  {
    priority: 80,
    description: 'Use file reader for document analysis',
    matches: (task, ctx) => ctx.hasAttachments && /read|analyze|extract/.test(task),
    targetTool: 'file_reader'
  }
];
```

### Strategy 3: Cost-Optimized Selection

Choose based on cost/performance trade-offs.

```typescript
class CostOptimizedRouter {
  async route(
    task: string,
    capableTools: Tool[],
    budget: Budget
  ): Promise<RoutingDecision> {
    // Score each tool
    const scored = capableTools.map(tool => ({
      tool,
      score: this.calculateScore(tool, budget)
    }));

    // Sort by score (higher is better)
    scored.sort((a, b) => b.score - a.score);

    return {
      tool: scored[0].tool.name,
      reasoning: `Best cost/performance ratio within budget`
    };
  }

  private calculateScore(tool: Tool, budget: Budget): number {
    const cost = tool.metadata.costPerCall || 0;
    const latency = tool.metadata.avgLatencyMs || 1000;

    // Can't use if over budget
    if (cost > budget.remaining) return -Infinity;

    // Score: lower cost and latency = higher score
    const costScore = 1 - (cost / budget.max);
    const latencyScore = 1 - Math.min(latency / 5000, 1);

    return costScore * budget.costWeight + latencyScore * budget.latencyWeight;
  }
}
```

## MCP Integration

### MCP Server Connection

```typescript
interface MCPServer {
  name: string;
  transport: 'stdio' | 'http' | 'websocket';
  config: MCPConfig;
}

class MCPToolProvider {
  private clients = new Map<string, MCPClient>();

  async connect(server: MCPServer): Promise<void> {
    const client = new MCPClient(server.transport, server.config);
    await client.connect();

    // Discover tools
    const tools = await client.listTools();

    // Register each tool
    for (const tool of tools) {
      registry.register({
        name: `${server.name}:${tool.name}`,
        description: tool.description,
        inputSchema: tool.inputSchema,
        execute: (input) => client.callTool(tool.name, input),
        metadata: {
          source: 'mcp',
          server: server.name
        }
      });
    }

    this.clients.set(server.name, client);
  }

  async disconnect(serverName: string): Promise<void> {
    const client = this.clients.get(serverName);
    if (client) {
      await client.close();
      this.clients.delete(serverName);
    }
  }
}
```

### MCP Configuration

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-filesystem"],
      "env": {
        "ALLOWED_PATHS": "/Users/dev/projects"
      }
    },
    "database": {
      "transport": "http",
      "url": "http://localhost:3001/mcp",
      "auth": {
        "type": "bearer",
        "token": "${DB_MCP_TOKEN}"
      }
    }
  }
}
```

## Tool Execution

### With Retry Logic

```typescript
async function executeWithRetry(
  tool: Tool,
  input: unknown,
  options: ExecutionOptions = {}
): Promise<ToolResult> {
  const maxRetries = options.maxRetries || 3;
  const backoff = options.backoffMs || 1000;

  let lastError: Error;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      // Check rate limit
      await rateLimiter.acquire(tool.name);

      // Execute
      const result = await tool.execute(input);

      if (result.success) {
        return result;
      }

      if (!result.error?.retryable) {
        return result;
      }

      lastError = new Error(result.error.message);
    } catch (error) {
      lastError = error as Error;
    }

    // Wait before retry
    if (attempt < maxRetries) {
      await sleep(backoff * Math.pow(2, attempt - 1));
    }
  }

  return {
    success: false,
    error: {
      code: 'MAX_RETRIES_EXCEEDED',
      message: `Failed after ${maxRetries} attempts: ${lastError.message}`,
      retryable: false
    },
    metadata: { durationMs: 0 }
  };
}
```

### With Fallback Chain

```typescript
async function executeWithFallback(
  task: string,
  tools: Tool[]
): Promise<ToolResult> {
  for (const tool of tools) {
    try {
      const result = await tool.execute({ task });

      if (result.success) {
        return result;
      }

      console.log(`Tool ${tool.name} failed, trying next...`);
    } catch (error) {
      console.log(`Tool ${tool.name} threw error, trying next...`);
    }
  }

  return {
    success: false,
    error: {
      code: 'ALL_TOOLS_FAILED',
      message: 'All fallback tools failed',
      retryable: false
    },
    metadata: { durationMs: 0 }
  };
}
```

## Monitoring Tool Usage

```typescript
interface ToolUsageMetrics {
  toolName: string;
  invocations: number;
  successRate: number;
  avgLatencyMs: number;
  totalCost: number;
  errorCounts: Map<string, number>;
}

class ToolMetricsCollector {
  private metrics = new Map<string, ToolUsageMetrics>();

  record(toolName: string, result: ToolResult): void {
    const m = this.getOrCreate(toolName);

    m.invocations++;
    m.avgLatencyMs = (m.avgLatencyMs * (m.invocations - 1) + result.metadata.durationMs) / m.invocations;

    if (result.success) {
      m.successRate = (m.successRate * (m.invocations - 1) + 1) / m.invocations;
    } else {
      m.successRate = (m.successRate * (m.invocations - 1)) / m.invocations;
      const errorCode = result.error?.code || 'UNKNOWN';
      m.errorCounts.set(errorCode, (m.errorCounts.get(errorCode) || 0) + 1);
    }
  }

  getReport(): ToolUsageMetrics[] {
    return Array.from(this.metrics.values());
  }
}
```

## Best Practices

1. **Clear tool descriptions** - LLMs select based on descriptions
2. **Appropriate granularity** - Not too broad, not too specific
3. **Handle failures gracefully** - Always have fallbacks
4. **Monitor usage** - Track costs, latency, errors
5. **Version tool schemas** - APIs change over time
6. **Rate limit appropriately** - Respect external service limits
7. **Cache when possible** - Avoid redundant calls
