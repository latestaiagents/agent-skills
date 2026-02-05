---
name: agent-supervisor-pattern
description: |
  Use this skill when designing supervisor-based multi-agent systems. Activate when the user needs to
  orchestrate multiple AI agents, coordinate agent workflows, implement a central controller for agents,
  design hub-and-spoke agent architecture, or build hierarchical agent systems.
---

# Agent Supervisor Pattern

Design centralized orchestration systems where a supervisor agent coordinates specialized worker agents.

## When to Use

- Complex tasks requiring multiple specialized agents
- Need clear visibility into agent decision-making
- Workflows requiring sequential or conditional agent execution
- Systems where human oversight is important
- Tasks with clear subtask decomposition

## Architecture Overview

```
                    ┌─────────────┐
                    │  SUPERVISOR │
                    │   AGENT     │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │ Research │    │  Writer  │    │ Reviewer │
    │  Agent   │    │  Agent   │    │  Agent   │
    └──────────┘    └──────────┘    └──────────┘
```

## Core Components

### 1. Supervisor Agent

The brain that orchestrates the workflow.

```typescript
interface SupervisorConfig {
  name: string;
  description: string;
  workers: WorkerAgent[];
  maxIterations: number;
  routingStrategy: 'sequential' | 'parallel' | 'conditional';
}

const supervisorPrompt = `
You are a supervisor agent coordinating specialized workers.

Available Workers:
{{#each workers}}
- {{name}}: {{description}}
{{/each}}

Your responsibilities:
1. Analyze the incoming task
2. Decompose into subtasks for appropriate workers
3. Route subtasks to the right worker
4. Synthesize results into final output
5. Handle errors and retry if needed

For each step, output:
{
  "reasoning": "Why this worker for this subtask",
  "worker": "worker_name",
  "task": "Specific instructions for the worker",
  "expectedOutput": "What you expect back"
}

When complete, output:
{
  "status": "complete",
  "result": "Final synthesized result"
}
`;
```

### 2. Worker Agents

Specialized agents for specific tasks.

```typescript
interface WorkerAgent {
  name: string;
  description: string;
  systemPrompt: string;
  tools: Tool[];
  outputSchema?: JSONSchema;
}

const researchWorker: WorkerAgent = {
  name: 'researcher',
  description: 'Searches and analyzes information from various sources',
  systemPrompt: `You are a research specialist.
    Search for accurate, up-to-date information.
    Always cite sources.
    Return structured findings.`,
  tools: [webSearch, documentReader],
  outputSchema: {
    type: 'object',
    properties: {
      findings: { type: 'array' },
      sources: { type: 'array' },
      confidence: { type: 'number' }
    }
  }
};
```

### 3. Router/Dispatcher

Routes tasks to appropriate workers.

```typescript
interface RouterDecision {
  worker: string;
  task: string;
  priority: number;
  dependencies: string[];
}

async function routeTask(
  task: string,
  context: WorkflowContext
): Promise<RouterDecision[]> {
  // Supervisor decides routing
  const decision = await supervisor.decide({
    task,
    availableWorkers: workers.map(w => ({
      name: w.name,
      description: w.description,
      currentLoad: w.pendingTasks
    })),
    completedSteps: context.history
  });

  return decision.subtasks;
}
```

## Implementation Patterns

### Pattern 1: Sequential Pipeline

```
Task → Worker A → Worker B → Worker C → Result
```

```typescript
async function sequentialPipeline(task: string) {
  const supervisor = createSupervisor({
    routingStrategy: 'sequential',
    workers: [researcher, writer, reviewer]
  });

  let context = { task, results: {} };

  // Supervisor determines order
  const plan = await supervisor.plan(task);

  for (const step of plan.steps) {
    const worker = workers.find(w => w.name === step.worker);
    const result = await worker.execute(step.task, context);

    context.results[step.worker] = result;

    // Supervisor validates and decides next step
    const validation = await supervisor.validate(result, step);
    if (!validation.proceed) {
      return supervisor.handleFailure(validation.reason, context);
    }
  }

  return supervisor.synthesize(context.results);
}
```

### Pattern 2: Conditional Branching

```
                    ┌─→ Path A Workers ─┐
Task → Classifier ─┤                    ├→ Result
                    └─→ Path B Workers ─┘
```

```typescript
async function conditionalWorkflow(task: string) {
  // Supervisor classifies the task
  const classification = await supervisor.classify(task);

  const workflow = workflowMap[classification.type];

  return executeWorkflow(workflow, task);
}

const workflowMap = {
  'code_review': [securityReviewer, performanceReviewer, styleReviewer],
  'content_creation': [researcher, writer, editor],
  'data_analysis': [dataFetcher, analyzer, visualizer]
};
```

### Pattern 3: Parallel Fan-Out

```
           ┌─→ Worker A ─┐
Task → ────┼─→ Worker B ─┼─→ Aggregator → Result
           └─→ Worker C ─┘
```

```typescript
async function parallelExecution(task: string) {
  // Supervisor splits task
  const subtasks = await supervisor.decompose(task);

  // Execute in parallel
  const results = await Promise.all(
    subtasks.map(async (subtask) => {
      const worker = selectWorker(subtask);
      return {
        worker: worker.name,
        result: await worker.execute(subtask)
      };
    })
  );

  // Supervisor synthesizes
  return supervisor.aggregate(results);
}
```

## Supervisor Prompting Best Practices

### Clear Role Definition

```markdown
## Your Role
You are the Supervisor Agent for a content creation system.

## Your Capabilities
- Decompose complex writing tasks into subtasks
- Assign subtasks to specialized worker agents
- Validate worker outputs meet quality standards
- Synthesize multiple outputs into coherent final product

## Your Constraints
- Never attempt tasks yourself - always delegate to workers
- Maximum 5 worker invocations per task
- Must validate each output before proceeding
```

### Decision Transparency

```markdown
## Output Format
For every decision, provide:

```json
{
  "thought": "Your reasoning process",
  "decision": "What you decided",
  "rationale": "Why this is the right choice",
  "nextAction": {
    "type": "delegate|validate|synthesize|complete|error",
    "details": {...}
  }
}
```
```

### Error Handling Instructions

```markdown
## When Workers Fail

1. First retry: Same worker, rephrased instructions
2. Second retry: Same worker, simpler subtask
3. Third failure: Try alternative worker if available
4. Final failure: Return partial results with explanation

Always log failures for human review.
```

## State Management

```typescript
interface WorkflowState {
  taskId: string;
  originalTask: string;
  currentPhase: string;
  completedSteps: Step[];
  pendingSteps: Step[];
  workerOutputs: Map<string, WorkerOutput>;
  errors: Error[];
  startTime: Date;
  tokenUsage: TokenUsage;
}

class SupervisorStateManager {
  async checkpoint(state: WorkflowState): Promise<void> {
    // Persist state for recovery
    await this.store.save(state.taskId, state);
  }

  async recover(taskId: string): Promise<WorkflowState | null> {
    return this.store.load(taskId);
  }
}
```

## Monitoring and Observability

```typescript
interface SupervisorMetrics {
  taskId: string;
  totalDuration: number;
  workerInvocations: {
    worker: string;
    duration: number;
    tokens: number;
    success: boolean;
  }[];
  decisionPoints: {
    timestamp: Date;
    decision: string;
    reasoning: string;
  }[];
  finalStatus: 'success' | 'partial' | 'failed';
}

// Log for analysis
function logSupervisorRun(metrics: SupervisorMetrics) {
  console.log(JSON.stringify({
    type: 'supervisor_run',
    ...metrics
  }));
}
```

## When NOT to Use Supervisor Pattern

- Simple single-step tasks
- Real-time/low-latency requirements
- When worker independence is valuable
- Budget-constrained scenarios (supervisor adds token overhead)

## Alternatives

| Pattern | Best For |
|---------|----------|
| Supervisor | Complex orchestration, visibility needed |
| Swarm/Mesh | Peer-to-peer collaboration, resilience |
| Pipeline | Linear workflows, streaming |
| Hierarchical | Large scale, nested supervision |
