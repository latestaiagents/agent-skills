---
name: agent-handoff-protocols
description: |
  Use this skill when designing task handoffs between agents. Activate when the user needs to pass work
  between agents, transfer context between agents, implement agent-to-agent communication, or design
  protocols for agents to collaborate on sequential tasks.
---

# Agent Handoff Protocols

Design clean, reliable protocols for passing tasks and context between AI agents.

## When to Use

- Agents need to pass work to each other
- Context must be preserved across agent transitions
- Building agent pipelines or workflows
- Implementing specialized agent chains

## Handoff Fundamentals

### What Gets Transferred

```typescript
interface HandoffPackage {
  // The work item
  task: {
    id: string;
    type: string;
    description: string;
    requirements: string[];
  };

  // Context from previous work
  context: {
    originalRequest: string;
    completedSteps: Step[];
    intermediateResults: Record<string, unknown>;
    relevantFiles: string[];
    decisions: Decision[];
  };

  // Metadata
  metadata: {
    sourceAgent: string;
    targetAgent: string;
    timestamp: Date;
    priority: number;
    deadline?: Date;
    attempt: number;
  };

  // Expectations
  expectations: {
    outputFormat: JSONSchema;
    successCriteria: string[];
    timeoutMs: number;
  };
}
```

### Handoff States

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ PENDING │───►│ ACCEPTED │───►│ WORKING  │───►│ COMPLETE │
└─────────┘    └──────────┘    └──────────┘    └──────────┘
     │              │               │               │
     ▼              ▼               ▼               ▼
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ TIMEOUT │    │ REJECTED │    │ FAILED   │    │ RETURNED │
└─────────┘    └──────────┘    └──────────┘    └──────────┘
```

## Protocol Patterns

### Pattern 1: Direct Handoff

Simple point-to-point transfer.

```typescript
class DirectHandoff {
  async transfer(
    from: Agent,
    to: Agent,
    package: HandoffPackage
  ): Promise<HandoffResult> {
    // Validate target can accept
    const canAccept = await to.canAccept(package);
    if (!canAccept.ok) {
      return { status: 'rejected', reason: canAccept.reason };
    }

    // Prepare context summary
    const contextSummary = await from.summarizeContext(package.context);

    // Execute handoff
    const ack = await to.receive({
      ...package,
      context: {
        ...package.context,
        summary: contextSummary
      }
    });

    if (ack.accepted) {
      await from.releaseTask(package.task.id);
      return { status: 'accepted', receivedAt: ack.timestamp };
    }

    return { status: 'rejected', reason: ack.reason };
  }
}
```

### Pattern 2: Queued Handoff

For async/decoupled systems.

```typescript
class QueuedHandoff {
  constructor(private queue: MessageQueue) {}

  async submit(package: HandoffPackage): Promise<string> {
    const ticket = generateTicketId();

    await this.queue.enqueue({
      ticket,
      package,
      status: 'pending',
      submittedAt: new Date()
    });

    return ticket;
  }

  async checkStatus(ticket: string): Promise<HandoffStatus> {
    return this.queue.getStatus(ticket);
  }

  // Target agent polls for work
  async claim(agentId: string): Promise<HandoffPackage | null> {
    const item = await this.queue.dequeue({
      filter: { targetAgent: agentId }
    });

    if (item) {
      await this.queue.updateStatus(item.ticket, 'claimed');
    }

    return item?.package || null;
  }
}
```

### Pattern 3: Brokered Handoff

Intermediary manages transfers.

```typescript
class HandoffBroker {
  private pending = new Map<string, PendingHandoff>();

  async initiateHandoff(
    source: Agent,
    targetCapability: string,
    package: HandoffPackage
  ): Promise<HandoffResult> {
    // Find suitable targets
    const candidates = await this.registry.findByCapability(targetCapability);

    // Negotiate with candidates
    for (const candidate of candidates) {
      const offer = await candidate.negotiate({
        taskType: package.task.type,
        estimatedTokens: this.estimateTokens(package),
        deadline: package.metadata.deadline
      });

      if (offer.accepted) {
        // Execute transfer through broker
        return this.executeTransfer(source, candidate, package, offer);
      }
    }

    // No takers - queue or return to source
    return this.handleNoTakers(source, package);
  }
}
```

## Context Transfer Strategies

### Strategy 1: Full Context

Transfer everything - simple but expensive.

```typescript
function fullContextTransfer(context: Context): TransferredContext {
  return {
    ...context,
    _transferType: 'full',
    _tokenEstimate: estimateTokens(context)
  };
}
```

### Strategy 2: Summarized Context

AI summarizes before transfer.

```typescript
async function summarizedTransfer(
  context: Context,
  targetNeeds: string[]
): Promise<TransferredContext> {
  const summary = await summarizeAgent.run(`
    Summarize this context for an agent that needs to:
    ${targetNeeds.join('\n')}

    Context:
    ${JSON.stringify(context)}

    Keep only information relevant to those needs.
    Be concise but complete.
  `);

  return {
    summary,
    _transferType: 'summarized',
    _originalTokens: estimateTokens(context),
    _summaryTokens: estimateTokens(summary)
  };
}
```

### Strategy 3: Reference-Based

Transfer references, target fetches as needed.

```typescript
interface ReferenceContext {
  taskId: string;
  contextUrl: string; // URL to fetch full context
  keyPoints: string[]; // Essential facts
  fileReferences: {
    path: string;
    relevantSections: string[];
  }[];
}

async function referenceTransfer(context: Context): Promise<ReferenceContext> {
  // Store full context
  const contextId = await contextStore.save(context);

  // Extract key points
  const keyPoints = await extractKeyPoints(context);

  return {
    taskId: context.taskId,
    contextUrl: `context://${contextId}`,
    keyPoints,
    fileReferences: context.relevantFiles.map(f => ({
      path: f.path,
      relevantSections: f.sections
    }))
  };
}
```

## Handoff Instructions Template

Include clear instructions for receiving agent.

```markdown
## Handoff Package

### Task
[Clear description of what needs to be done]

### Context Summary
- Original request: [What the user asked for]
- What's been done: [Completed steps]
- Current state: [Where we are now]

### Your Specific Task
[Exact instructions for this agent]

### Key Information
- [Critical fact 1]
- [Critical fact 2]
- [Decision that was made and why]

### Constraints
- [Time/token budget]
- [Must use existing patterns in X file]
- [Cannot modify Y]

### Expected Output
[Format and content expectations]

### Success Criteria
- [Criterion 1]
- [Criterion 2]

### If You Get Stuck
- [Fallback option 1]
- [Escalation path]
```

## Error Handling

### Rejection Handling

```typescript
async function handleRejection(
  package: HandoffPackage,
  reason: string
): Promise<HandoffResult> {
  // Log rejection
  await log.handoffRejected(package.task.id, reason);

  if (package.metadata.attempt < MAX_ATTEMPTS) {
    // Retry with different target
    return retryHandoff({
      ...package,
      metadata: {
        ...package.metadata,
        attempt: package.metadata.attempt + 1
      }
    }, { excludeAgent: package.metadata.targetAgent });
  }

  // Max retries - escalate
  return escalateToHuman(package, `Handoff failed: ${reason}`);
}
```

### Timeout Handling

```typescript
async function monitorHandoff(
  ticket: string,
  timeoutMs: number
): Promise<void> {
  const deadline = Date.now() + timeoutMs;

  while (Date.now() < deadline) {
    const status = await handoffQueue.getStatus(ticket);

    if (status === 'complete') return;
    if (status === 'failed') throw new HandoffError('Task failed');

    await sleep(POLL_INTERVAL);
  }

  // Timeout
  await handoffQueue.cancel(ticket);
  throw new HandoffTimeoutError(ticket);
}
```

### Rollback Handling

```typescript
async function rollbackHandoff(
  package: HandoffPackage,
  reason: string
): Promise<void> {
  // Return task to source
  await package.metadata.sourceAgent.receive({
    ...package,
    context: {
      ...package.context,
      rollbackReason: reason,
      partialResult: await getPartialResult(package.task.id)
    }
  });

  // Clean up any partial work
  await cleanupPartialWork(package.task.id);

  // Log for analysis
  await log.handoffRolledBack(package, reason);
}
```

## Acknowledgment Patterns

```typescript
interface HandoffAck {
  status: 'accepted' | 'rejected' | 'conditional';
  timestamp: Date;
  agentId: string;

  // If rejected
  reason?: string;
  suggestedAlternative?: string;

  // If conditional
  conditions?: {
    needsMoreContext?: string[];
    needsPermission?: string[];
    estimatedCompletionMs?: number;
  };
}

// Receiving agent response
async function acknowledgeHandoff(
  package: HandoffPackage
): Promise<HandoffAck> {
  // Validate we can handle this
  const validation = await this.validateTask(package.task);

  if (!validation.canHandle) {
    return {
      status: 'rejected',
      timestamp: new Date(),
      agentId: this.id,
      reason: validation.reason,
      suggestedAlternative: validation.alternativeAgent
    };
  }

  if (validation.needsMore) {
    return {
      status: 'conditional',
      timestamp: new Date(),
      agentId: this.id,
      conditions: {
        needsMoreContext: validation.missingContext
      }
    };
  }

  return {
    status: 'accepted',
    timestamp: new Date(),
    agentId: this.id
  };
}
```

## Best Practices

1. **Define clear contracts** - What each agent expects and provides
2. **Include enough context** - Target shouldn't need to ask questions
3. **Set explicit timeouts** - Don't let tasks hang forever
4. **Implement idempotency** - Handoffs may be retried
5. **Log all transitions** - Essential for debugging
6. **Version your protocols** - Agents may be updated independently
