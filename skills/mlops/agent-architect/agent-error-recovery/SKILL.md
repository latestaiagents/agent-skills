---
name: agent-error-recovery
description: |
  Use this skill when implementing error handling for AI agents. Activate when the user needs agents to
  handle failures gracefully, implement retry strategies, design fault-tolerant agent systems, or build
  agents that can recover from errors without human intervention.
---

# Agent Error Recovery

Design fault-tolerant agent systems that recover gracefully from failures.

## When to Use

- Building production-grade agent systems
- Agents need to handle API failures
- Implementing autonomous error recovery
- Designing resilient multi-agent workflows
- Setting up monitoring and alerting

## Error Classification

```typescript
enum ErrorCategory {
  // Transient - retry likely to succeed
  RATE_LIMIT = 'rate_limit',
  TIMEOUT = 'timeout',
  NETWORK = 'network',
  SERVICE_UNAVAILABLE = 'service_unavailable',

  // Recoverable - different approach may work
  INVALID_INPUT = 'invalid_input',
  CONTEXT_OVERFLOW = 'context_overflow',
  TOOL_FAILURE = 'tool_failure',

  // Terminal - cannot proceed
  AUTHENTICATION = 'authentication',
  AUTHORIZATION = 'authorization',
  NOT_FOUND = 'not_found',
  VALIDATION = 'validation',

  // Unknown
  UNKNOWN = 'unknown'
}

interface AgentError {
  category: ErrorCategory;
  code: string;
  message: string;
  retryable: boolean;
  context: Record<string, unknown>;
  timestamp: Date;
  stackTrace?: string;
}

function classifyError(error: Error): AgentError {
  // Rate limits
  if (error.message.includes('429') || error.message.includes('rate limit')) {
    return {
      category: ErrorCategory.RATE_LIMIT,
      code: 'RATE_LIMITED',
      message: error.message,
      retryable: true,
      context: { waitTime: extractWaitTime(error) },
      timestamp: new Date()
    };
  }

  // Timeouts
  if (error.message.includes('timeout') || error.message.includes('ETIMEDOUT')) {
    return {
      category: ErrorCategory.TIMEOUT,
      code: 'TIMEOUT',
      message: error.message,
      retryable: true,
      context: {},
      timestamp: new Date()
    };
  }

  // Context overflow
  if (error.message.includes('context length') || error.message.includes('too long')) {
    return {
      category: ErrorCategory.CONTEXT_OVERFLOW,
      code: 'CONTEXT_OVERFLOW',
      message: error.message,
      retryable: true, // Can retry with truncated context
      context: {},
      timestamp: new Date()
    };
  }

  // Default
  return {
    category: ErrorCategory.UNKNOWN,
    code: 'UNKNOWN',
    message: error.message,
    retryable: false,
    context: {},
    timestamp: new Date(),
    stackTrace: error.stack
  };
}
```

## Recovery Strategies

### Strategy 1: Retry with Backoff

```typescript
interface RetryConfig {
  maxAttempts: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
  jitterMs: number;
}

async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  config: RetryConfig
): Promise<T> {
  let lastError: Error;
  let delay = config.initialDelayMs;

  for (let attempt = 1; attempt <= config.maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error as Error;
      const classified = classifyError(lastError);

      // Don't retry non-retryable errors
      if (!classified.retryable) {
        throw lastError;
      }

      // Last attempt - throw
      if (attempt === config.maxAttempts) {
        throw lastError;
      }

      // Calculate delay with jitter
      const jitter = Math.random() * config.jitterMs;
      const waitTime = Math.min(delay + jitter, config.maxDelayMs);

      console.log(`Attempt ${attempt} failed, retrying in ${waitTime}ms...`);
      await sleep(waitTime);

      // Increase delay for next attempt
      delay *= config.backoffMultiplier;
    }
  }

  throw lastError!;
}
```

### Strategy 2: Circuit Breaker

```typescript
enum CircuitState {
  CLOSED = 'closed',     // Normal operation
  OPEN = 'open',         // Failing, reject requests
  HALF_OPEN = 'half_open' // Testing if recovered
}

class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failures = 0;
  private lastFailure?: Date;
  private successCount = 0;

  constructor(
    private config: {
      failureThreshold: number;
      resetTimeoutMs: number;
      successThreshold: number;
    }
  ) {}

  async execute<T>(operation: () => Promise<T>): Promise<T> {
    // Check if circuit should transition
    this.checkState();

    if (this.state === CircuitState.OPEN) {
      throw new Error('Circuit breaker is OPEN');
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private checkState(): void {
    if (this.state === CircuitState.OPEN) {
      const elapsed = Date.now() - this.lastFailure!.getTime();
      if (elapsed >= this.config.resetTimeoutMs) {
        this.state = CircuitState.HALF_OPEN;
        this.successCount = 0;
      }
    }
  }

  private onSuccess(): void {
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.config.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.failures = 0;
      }
    } else {
      this.failures = 0;
    }
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailure = new Date();

    if (this.failures >= this.config.failureThreshold) {
      this.state = CircuitState.OPEN;
    }
  }
}
```

### Strategy 3: Fallback Chain

```typescript
interface FallbackOption<T> {
  name: string;
  execute: () => Promise<T>;
  isApplicable: (error: AgentError) => boolean;
}

async function executeWithFallbacks<T>(
  primary: () => Promise<T>,
  fallbacks: FallbackOption<T>[]
): Promise<T> {
  try {
    return await primary();
  } catch (error) {
    const classified = classifyError(error as Error);

    for (const fallback of fallbacks) {
      if (fallback.isApplicable(classified)) {
        console.log(`Primary failed, trying fallback: ${fallback.name}`);
        try {
          return await fallback.execute();
        } catch (fallbackError) {
          console.log(`Fallback ${fallback.name} also failed`);
          continue;
        }
      }
    }

    // All fallbacks failed
    throw error;
  }
}

// Example usage
const result = await executeWithFallbacks(
  () => callPrimaryAPI(),
  [
    {
      name: 'backup_api',
      execute: () => callBackupAPI(),
      isApplicable: (e) => e.category === ErrorCategory.SERVICE_UNAVAILABLE
    },
    {
      name: 'cached_response',
      execute: () => getCachedResponse(),
      isApplicable: (e) => e.category === ErrorCategory.TIMEOUT
    },
    {
      name: 'simplified_request',
      execute: () => callWithReducedParams(),
      isApplicable: (e) => e.category === ErrorCategory.CONTEXT_OVERFLOW
    }
  ]
);
```

### Strategy 4: Self-Healing Agent

```typescript
class SelfHealingAgent {
  async execute(task: Task): Promise<Result> {
    let attempt = 0;
    const maxAttempts = 3;

    while (attempt < maxAttempts) {
      attempt++;

      try {
        return await this.runTask(task);
      } catch (error) {
        const classified = classifyError(error as Error);

        // Can we heal?
        const healingAction = this.determineHealingAction(classified);

        if (!healingAction) {
          throw error;
        }

        console.log(`Attempting self-healing: ${healingAction.description}`);

        // Execute healing
        await healingAction.execute();

        // Modify task if needed
        task = healingAction.modifyTask?.(task) || task;
      }
    }

    throw new Error('Max healing attempts exceeded');
  }

  private determineHealingAction(error: AgentError): HealingAction | null {
    switch (error.category) {
      case ErrorCategory.CONTEXT_OVERFLOW:
        return {
          description: 'Truncating context to fit limits',
          execute: async () => {},
          modifyTask: (task) => ({
            ...task,
            context: this.truncateContext(task.context)
          })
        };

      case ErrorCategory.TOOL_FAILURE:
        return {
          description: 'Switching to alternative tool',
          execute: async () => {
            this.toolRouter.excludeTool(error.context.toolName as string);
          }
        };

      case ErrorCategory.RATE_LIMIT:
        return {
          description: `Waiting ${error.context.waitTime}ms for rate limit`,
          execute: async () => {
            await sleep(error.context.waitTime as number);
          }
        };

      default:
        return null;
    }
  }
}
```

## Error Recovery Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                      Error Occurs                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Classify Error                            │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
       ┌─────────┐      ┌─────────┐      ┌─────────┐
       │Transient│      │Recoverable    │Terminal │
       └────┬────┘      └────┬────┘      └────┬────┘
            │                │                │
            ▼                ▼                ▼
       ┌─────────┐      ┌─────────┐      ┌─────────┐
       │ Retry   │      │ Try     │      │ Log &   │
       │ w/Backoff     │ Fallback│      │ Alert   │
       └────┬────┘      └────┬────┘      └────┬────┘
            │                │                │
            ▼                ▼                ▼
       ┌─────────────────────────────────────────┐
       │           Success or Escalate           │
       └─────────────────────────────────────────┘
```

## Monitoring and Alerting

```typescript
interface ErrorMetrics {
  totalErrors: number;
  errorsByCategory: Map<ErrorCategory, number>;
  errorRate: number; // errors per minute
  recoveryRate: number; // successful recoveries
  mttr: number; // mean time to recover (ms)
}

class ErrorMonitor {
  private errors: AgentError[] = [];
  private recoveries: { error: AgentError; recoveredAt: Date }[] = [];

  recordError(error: AgentError): void {
    this.errors.push(error);
    this.checkAlerts();
  }

  recordRecovery(error: AgentError): void {
    this.recoveries.push({ error, recoveredAt: new Date() });
  }

  private checkAlerts(): void {
    const recentErrors = this.getRecentErrors(60000); // Last minute

    // High error rate alert
    if (recentErrors.length > 10) {
      this.sendAlert({
        severity: 'high',
        message: `High error rate: ${recentErrors.length} errors in last minute`,
        errors: recentErrors
      });
    }

    // Repeated same error alert
    const errorCounts = new Map<string, number>();
    for (const e of recentErrors) {
      const key = `${e.category}:${e.code}`;
      errorCounts.set(key, (errorCounts.get(key) || 0) + 1);
    }

    for (const [key, count] of errorCounts) {
      if (count >= 5) {
        this.sendAlert({
          severity: 'medium',
          message: `Repeated error: ${key} occurred ${count} times`,
          errors: recentErrors.filter(e => `${e.category}:${e.code}` === key)
        });
      }
    }
  }

  getMetrics(): ErrorMetrics {
    const window = 5 * 60 * 1000; // 5 minutes
    const recent = this.getRecentErrors(window);

    const byCategory = new Map<ErrorCategory, number>();
    for (const e of recent) {
      byCategory.set(e.category, (byCategory.get(e.category) || 0) + 1);
    }

    const recentRecoveries = this.recoveries.filter(
      r => Date.now() - r.recoveredAt.getTime() < window
    );

    const recoveryTimes = recentRecoveries.map(
      r => r.recoveredAt.getTime() - r.error.timestamp.getTime()
    );

    return {
      totalErrors: recent.length,
      errorsByCategory: byCategory,
      errorRate: recent.length / (window / 60000),
      recoveryRate: recentRecoveries.length / Math.max(recent.length, 1),
      mttr: recoveryTimes.length > 0
        ? recoveryTimes.reduce((a, b) => a + b, 0) / recoveryTimes.length
        : 0
    };
  }
}
```

## Best Practices

1. **Classify all errors** - Know what you're dealing with
2. **Don't retry everything** - Some errors won't recover
3. **Use exponential backoff** - Avoid hammering failing services
4. **Set circuit breakers** - Protect downstream systems
5. **Log everything** - Debugging is hard without logs
6. **Have fallbacks** - Always have a Plan B
7. **Alert on patterns** - Single errors may be noise, patterns matter
8. **Test failure scenarios** - Chaos engineering
