---
name: ai-audit-logging
description: |
  Use this skill when implementing audit logging for AI systems. Activate when the user needs to track
  AI decisions for compliance, implement audit trails for LLM usage, meet regulatory requirements
  (EU AI Act, SOC2), or create accountability records for AI-generated content.
---

# AI Audit Logging

Implement compliance-ready audit trails for AI system decisions and outputs.

## When to Use

- Meeting regulatory requirements (EU AI Act, SOC2, HIPAA)
- Tracking AI decisions for accountability
- Debugging AI system behavior
- Investigating incidents involving AI
- Demonstrating AI governance

## Regulatory Context (2026)

### EU AI Act Requirements

- **High-risk AI systems** must maintain logs that enable:
  - Traceability of AI decisions
  - Recording of reference data
  - Logging of events throughout lifecycle
  - Logs must be kept for appropriate period

- **Effective August 2026** with fines up to 7% of global revenue

### SOC2 AI Considerations

- **Processing Integrity**: AI outputs must be accurate and complete
- **Confidentiality**: AI must not expose confidential data
- **Availability**: AI systems must maintain audit logs

## Audit Log Schema

```typescript
interface AIAuditLog {
  // Identification
  id: string;
  timestamp: Date;
  correlationId: string;
  sessionId: string;

  // Actor
  actor: {
    type: 'user' | 'system' | 'automated';
    id: string;
    name?: string;
    ip?: string;
    userAgent?: string;
  };

  // AI Operation
  operation: {
    type: 'inference' | 'generation' | 'classification' | 'analysis';
    model: string;
    modelVersion: string;
    provider: string;
  };

  // Input/Output
  io: {
    inputHash: string;        // Hash of input for privacy
    inputTokens: number;
    outputHash: string;       // Hash of output
    outputTokens: number;
    inputSummary?: string;    // Optional human-readable summary
    outputSummary?: string;
  };

  // Decisions
  decision?: {
    action: string;
    confidence: number;
    alternatives?: { action: string; confidence: number }[];
    reasoning?: string;
  };

  // Safety
  safety: {
    contentFiltered: boolean;
    filterReasons?: string[];
    humanReviewRequired: boolean;
    humanReviewCompleted?: boolean;
    reviewerId?: string;
  };

  // Performance
  performance: {
    latencyMs: number;
    queueTimeMs?: number;
    tokensPerSecond?: number;
  };

  // Cost
  cost: {
    inputCost: number;
    outputCost: number;
    totalCost: number;
    currency: string;
  };

  // Context
  context: {
    application: string;
    environment: 'production' | 'staging' | 'development';
    feature?: string;
    tags: string[];
  };

  // Metadata
  metadata: Record<string, unknown>;
}
```

## Implementation

### Logger Class

```typescript
import { createHash } from 'crypto';

class AIAuditLogger {
  constructor(
    private storage: AuditStorage,
    private config: AuditConfig
  ) {}

  async log(event: Partial<AIAuditLog>): Promise<string> {
    const log: AIAuditLog = {
      id: crypto.randomUUID(),
      timestamp: new Date(),
      correlationId: event.correlationId || crypto.randomUUID(),
      sessionId: event.sessionId || 'unknown',
      actor: event.actor || { type: 'system', id: 'unknown' },
      operation: event.operation!,
      io: event.io!,
      safety: event.safety || { contentFiltered: false, humanReviewRequired: false },
      performance: event.performance || { latencyMs: 0 },
      cost: event.cost || { inputCost: 0, outputCost: 0, totalCost: 0, currency: 'USD' },
      context: event.context || { application: 'unknown', environment: 'production', tags: [] },
      metadata: event.metadata || {}
    };

    // Validate required fields
    this.validate(log);

    // Hash sensitive content
    log.io.inputHash = this.hashContent(log.io.inputHash);
    log.io.outputHash = this.hashContent(log.io.outputHash);

    // Store
    await this.storage.write(log);

    // Alert if needed
    if (log.safety.humanReviewRequired) {
      await this.alertForReview(log);
    }

    return log.id;
  }

  private hashContent(content: string): string {
    return createHash('sha256').update(content).digest('hex');
  }

  private validate(log: AIAuditLog): void {
    if (!log.operation?.model) {
      throw new Error('Audit log must include model information');
    }
    if (log.io.inputTokens === undefined || log.io.outputTokens === undefined) {
      throw new Error('Audit log must include token counts');
    }
  }

  private async alertForReview(log: AIAuditLog): Promise<void> {
    // Send to review queue
    await this.config.reviewQueue?.push({
      logId: log.id,
      reason: log.safety.filterReasons?.join(', '),
      priority: 'high'
    });
  }
}
```

### Middleware for API Calls

```typescript
function withAuditLogging(
  client: LLMClient,
  logger: AIAuditLogger,
  context: Partial<AIAuditLog['context']>
): LLMClient {
  return {
    async complete(params: CompletionParams): Promise<CompletionResponse> {
      const startTime = Date.now();

      try {
        const response = await client.complete(params);

        await logger.log({
          operation: {
            type: 'generation',
            model: params.model,
            modelVersion: response.model,
            provider: client.provider
          },
          io: {
            inputHash: params.messages.map(m => m.content).join(''),
            inputTokens: response.usage.input_tokens,
            outputHash: response.content[0].text,
            outputTokens: response.usage.output_tokens
          },
          performance: {
            latencyMs: Date.now() - startTime
          },
          cost: calculateCost(params.model, response.usage),
          context: {
            ...context,
            application: context.application || 'default',
            environment: context.environment || 'production',
            tags: context.tags || []
          }
        });

        return response;
      } catch (error) {
        await logger.log({
          operation: {
            type: 'generation',
            model: params.model,
            modelVersion: 'unknown',
            provider: client.provider
          },
          io: {
            inputHash: params.messages.map(m => m.content).join(''),
            inputTokens: 0,
            outputHash: '',
            outputTokens: 0
          },
          performance: {
            latencyMs: Date.now() - startTime
          },
          metadata: {
            error: (error as Error).message,
            errorType: (error as Error).name
          },
          context: context as any
        });

        throw error;
      }
    }
  };
}
```

## Storage Backends

### Database Storage

```typescript
import { PrismaClient } from '@prisma/client';

class DatabaseAuditStorage implements AuditStorage {
  constructor(private prisma: PrismaClient) {}

  async write(log: AIAuditLog): Promise<void> {
    await this.prisma.aiAuditLog.create({
      data: {
        id: log.id,
        timestamp: log.timestamp,
        correlationId: log.correlationId,
        sessionId: log.sessionId,
        actorType: log.actor.type,
        actorId: log.actor.id,
        model: log.operation.model,
        operationType: log.operation.type,
        inputTokens: log.io.inputTokens,
        outputTokens: log.io.outputTokens,
        inputHash: log.io.inputHash,
        outputHash: log.io.outputHash,
        latencyMs: log.performance.latencyMs,
        totalCost: log.cost.totalCost,
        contentFiltered: log.safety.contentFiltered,
        humanReviewRequired: log.safety.humanReviewRequired,
        application: log.context.application,
        environment: log.context.environment,
        metadata: log.metadata as any
      }
    });
  }

  async query(filters: AuditQueryFilters): Promise<AIAuditLog[]> {
    return this.prisma.aiAuditLog.findMany({
      where: {
        timestamp: {
          gte: filters.startDate,
          lte: filters.endDate
        },
        actorId: filters.actorId,
        model: filters.model,
        application: filters.application
      },
      orderBy: { timestamp: 'desc' },
      take: filters.limit || 100
    });
  }
}
```

### Immutable Log Storage

```typescript
// For compliance, logs should be immutable and tamper-evident
class ImmutableAuditStorage implements AuditStorage {
  private previousHash = '0';

  async write(log: AIAuditLog): Promise<void> {
    // Chain logs with hashes for tamper evidence
    const logWithChain = {
      ...log,
      previousLogHash: this.previousHash,
      logHash: this.hashLog(log, this.previousHash)
    };

    await this.storage.append(logWithChain);
    this.previousHash = logWithChain.logHash;
  }

  private hashLog(log: AIAuditLog, previousHash: string): string {
    const content = JSON.stringify({ ...log, previousHash });
    return createHash('sha256').update(content).digest('hex');
  }

  async verifyIntegrity(logs: AIAuditLog[]): Promise<boolean> {
    let expectedHash = '0';

    for (const log of logs) {
      const computedHash = this.hashLog(log, expectedHash);
      if (computedHash !== (log as any).logHash) {
        return false; // Tampering detected
      }
      expectedHash = computedHash;
    }

    return true;
  }
}
```

## Compliance Reports

```typescript
interface ComplianceReport {
  period: { start: Date; end: Date };
  summary: {
    totalAIOperations: number;
    uniqueUsers: number;
    totalCost: number;
    contentFilteredCount: number;
    humanReviewCount: number;
  };
  modelUsage: { model: string; count: number; cost: number }[];
  riskEvents: {
    id: string;
    timestamp: Date;
    type: string;
    severity: 'low' | 'medium' | 'high';
    resolution?: string;
  }[];
  dataRetention: {
    logsRetained: number;
    oldestLog: Date;
    retentionPolicy: string;
  };
}

async function generateComplianceReport(
  storage: AuditStorage,
  period: { start: Date; end: Date }
): Promise<ComplianceReport> {
  const logs = await storage.query({ startDate: period.start, endDate: period.end });

  // Aggregate metrics
  const uniqueUsers = new Set(logs.map(l => l.actor.id)).size;
  const totalCost = logs.reduce((sum, l) => sum + l.cost.totalCost, 0);
  const contentFiltered = logs.filter(l => l.safety.contentFiltered).length;
  const humanReview = logs.filter(l => l.safety.humanReviewRequired).length;

  // Model usage breakdown
  const modelUsage = new Map<string, { count: number; cost: number }>();
  for (const log of logs) {
    const existing = modelUsage.get(log.operation.model) || { count: 0, cost: 0 };
    modelUsage.set(log.operation.model, {
      count: existing.count + 1,
      cost: existing.cost + log.cost.totalCost
    });
  }

  return {
    period,
    summary: {
      totalAIOperations: logs.length,
      uniqueUsers,
      totalCost,
      contentFilteredCount: contentFiltered,
      humanReviewCount: humanReview
    },
    modelUsage: Array.from(modelUsage.entries()).map(([model, data]) => ({
      model,
      ...data
    })),
    riskEvents: logs
      .filter(l => l.safety.contentFiltered || l.safety.humanReviewRequired)
      .map(l => ({
        id: l.id,
        timestamp: l.timestamp,
        type: l.safety.filterReasons?.[0] || 'unknown',
        severity: l.safety.humanReviewRequired ? 'high' : 'medium' as const
      })),
    dataRetention: {
      logsRetained: logs.length,
      oldestLog: logs[logs.length - 1]?.timestamp || new Date(),
      retentionPolicy: '90 days'
    }
  };
}
```

## Best Practices

1. **Log everything** - Better to have too much than too little
2. **Hash sensitive data** - Don't store PII in plain text
3. **Use immutable storage** - Prevent tampering
4. **Set retention policies** - Balance compliance and cost
5. **Enable search** - Logs are useless if unsearchable
6. **Automate reporting** - Compliance shouldn't be manual
7. **Test your logging** - Ensure it works before you need it
