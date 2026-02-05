---
name: log-forensics
description: |
  Use this skill when investigating issues through logs. Activate when the user needs to analyze log files,
  search for specific events in logs, correlate logs across services, investigate incidents through logs,
  or extract insights from application logs.
---

# Log Forensics

Extract insights and trace issues through application logs.

## When to Use

- Investigating production incidents
- Tracing request flows across services
- Finding the root cause of failures
- Analyzing system behavior over time
- Correlating events across components

## Log Structure

### Standardized Log Format

```typescript
interface StructuredLog {
  timestamp: string;          // ISO 8601
  level: 'debug' | 'info' | 'warn' | 'error' | 'fatal';
  message: string;
  service: string;
  traceId?: string;          // For distributed tracing
  spanId?: string;
  userId?: string;
  requestId?: string;
  context: Record<string, unknown>;
  error?: {
    name: string;
    message: string;
    stack?: string;
  };
}

// Example
{
  "timestamp": "2026-02-04T10:30:45.123Z",
  "level": "error",
  "message": "Payment processing failed",
  "service": "payment-service",
  "traceId": "abc123",
  "requestId": "req-456",
  "userId": "user-789",
  "context": {
    "amount": 99.99,
    "currency": "USD",
    "provider": "stripe"
  },
  "error": {
    "name": "PaymentError",
    "message": "Card declined",
    "stack": "..."
  }
}
```

## Search Techniques

### Basic Log Queries

```bash
# Find errors in time range
grep -E "\"level\":\"error\"" logs.json | \
  jq 'select(.timestamp >= "2026-02-04T10:00:00")'

# Find by trace ID
grep "traceId.*abc123" logs/*.json

# Count by level
jq -r '.level' logs.json | sort | uniq -c

# Find unique error messages
jq -r 'select(.level=="error") | .message' logs.json | sort | uniq -c | sort -rn
```

### Advanced Filtering

```typescript
// Query DSL for log analysis
interface LogQuery {
  timeRange: { start: Date; end: Date };
  filters: Filter[];
  aggregations?: Aggregation[];
  limit?: number;
}

// Example: Find all errors for a user in last hour
const query: LogQuery = {
  timeRange: {
    start: new Date(Date.now() - 3600000),
    end: new Date()
  },
  filters: [
    { field: 'level', op: 'eq', value: 'error' },
    { field: 'userId', op: 'eq', value: 'user-123' }
  ],
  aggregations: [
    { type: 'count', field: 'message' },
    { type: 'terms', field: 'service', size: 10 }
  ]
};
```

## Correlation Techniques

### Request Flow Tracing

```typescript
async function traceRequest(traceId: string): Promise<RequestFlow> {
  // Gather all logs for this trace
  const logs = await searchLogs({
    filters: [{ field: 'traceId', op: 'eq', value: traceId }],
    limit: 1000
  });

  // Sort by timestamp
  logs.sort((a, b) =>
    new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime()
  );

  // Group by service
  const byService = new Map<string, StructuredLog[]>();
  for (const log of logs) {
    const existing = byService.get(log.service) || [];
    existing.push(log);
    byService.set(log.service, existing);
  }

  return {
    traceId,
    duration: calculateDuration(logs),
    services: Array.from(byService.keys()),
    timeline: logs,
    errors: logs.filter(l => l.level === 'error'),
    warnings: logs.filter(l => l.level === 'warn')
  };
}
```

### Timeline Reconstruction

```typescript
function reconstructTimeline(
  logs: StructuredLog[]
): TimelineEvent[] {
  const events: TimelineEvent[] = [];

  for (const log of logs) {
    events.push({
      timestamp: new Date(log.timestamp),
      service: log.service,
      event: categorizeEvent(log),
      summary: log.message,
      details: log.context,
      severity: logLevelToSeverity(log.level)
    });
  }

  return events.sort((a, b) =>
    a.timestamp.getTime() - b.timestamp.getTime()
  );
}

// Output as text timeline
function formatTimeline(events: TimelineEvent[]): string {
  return events.map(e => {
    const time = e.timestamp.toISOString().slice(11, 23);
    const icon = severityIcon(e.severity);
    return `${time} ${icon} [${e.service}] ${e.summary}`;
  }).join('\n');
}

/*
Output:
10:30:45.123 ✓ [api-gateway] Request received POST /orders
10:30:45.156 ✓ [auth-service] Token validated for user-123
10:30:45.203 ✓ [order-service] Creating order
10:30:45.456 ⚠ [inventory-service] Low stock warning
10:30:45.789 ✗ [payment-service] Payment failed: Card declined
10:30:45.801 ✗ [order-service] Order creation failed
10:30:45.823 ✓ [api-gateway] Response sent 402
*/
```

### Cross-Service Correlation

```typescript
async function correlateAcrossServices(
  incident: Incident
): Promise<ServiceCorrelation[]> {
  const timeWindow = {
    start: new Date(incident.timestamp.getTime() - 60000), // 1 min before
    end: new Date(incident.timestamp.getTime() + 60000)    // 1 min after
  };

  // Get logs from all services in window
  const allLogs = await searchLogs({
    timeRange: timeWindow,
    filters: [{ field: 'level', op: 'in', value: ['error', 'warn'] }]
  });

  // Group by service
  const byService = groupBy(allLogs, 'service');

  // Find correlated events
  const correlations: ServiceCorrelation[] = [];

  for (const [service, logs] of Object.entries(byService)) {
    const related = logs.filter(log =>
      isTemporallyRelated(log, incident) ||
      isContextuallyRelated(log, incident)
    );

    if (related.length > 0) {
      correlations.push({
        service,
        relatedLogs: related,
        correlation: calculateCorrelationScore(related, incident)
      });
    }
  }

  return correlations.sort((a, b) => b.correlation - a.correlation);
}
```

## Pattern Recognition

### Anomaly Detection

```typescript
function detectAnomalies(
  logs: StructuredLog[],
  baseline: LogBaseline
): Anomaly[] {
  const anomalies: Anomaly[] = [];

  // Volume anomaly
  const currentRate = logs.length / (timeRange.end - timeRange.start);
  if (currentRate > baseline.avgRate * 3) {
    anomalies.push({
      type: 'volume_spike',
      severity: 'high',
      message: `Log volume ${currentRate.toFixed(0)}/min vs baseline ${baseline.avgRate.toFixed(0)}/min`
    });
  }

  // Error rate anomaly
  const errorRate = logs.filter(l => l.level === 'error').length / logs.length;
  if (errorRate > baseline.avgErrorRate * 2) {
    anomalies.push({
      type: 'error_rate_spike',
      severity: 'critical',
      message: `Error rate ${(errorRate * 100).toFixed(1)}% vs baseline ${(baseline.avgErrorRate * 100).toFixed(1)}%`
    });
  }

  // New error types
  const currentErrors = new Set(logs.filter(l => l.level === 'error').map(l => l.message));
  const newErrors = [...currentErrors].filter(e => !baseline.knownErrors.has(e));
  if (newErrors.length > 0) {
    anomalies.push({
      type: 'new_errors',
      severity: 'medium',
      message: `${newErrors.length} new error types detected`,
      details: newErrors
    });
  }

  return anomalies;
}
```

### Sequence Pattern Mining

```typescript
function findCommonSequences(
  traces: RequestFlow[]
): SequencePattern[] {
  // Extract event sequences from each trace
  const sequences = traces.map(t =>
    t.timeline.map(e => `${e.service}:${e.event}`)
  );

  // Find common subsequences
  const patterns = new Map<string, number>();

  for (const seq of sequences) {
    // Sliding window of 3-5 events
    for (let windowSize = 3; windowSize <= 5; windowSize++) {
      for (let i = 0; i <= seq.length - windowSize; i++) {
        const pattern = seq.slice(i, i + windowSize).join(' → ');
        patterns.set(pattern, (patterns.get(pattern) || 0) + 1);
      }
    }
  }

  // Return frequent patterns
  return Array.from(patterns.entries())
    .filter(([_, count]) => count > traces.length * 0.1)
    .sort((a, b) => b[1] - a[1])
    .map(([pattern, count]) => ({
      sequence: pattern,
      occurrences: count,
      percentage: count / traces.length
    }));
}
```

## Investigation Workflow

### Step 1: Scope the Investigation

```markdown
## Investigation Scope

**Incident:** [Brief description]
**Time Window:** [Start] to [End]
**Affected Services:** [List]
**Key Identifiers:**
- Trace ID: [if available]
- User ID: [if available]
- Request ID: [if available]
```

### Step 2: Initial Search

```bash
# Quick overview of the time window
jq -r '.level' logs.json | sort | uniq -c

# Find first errors
jq 'select(.level=="error")' logs.json | head -20

# Find affected users
jq -r 'select(.level=="error") | .userId' logs.json | sort | uniq -c
```

### Step 3: Trace the Flow

```bash
# If trace ID is known
grep "traceId.*$TRACE_ID" *.json | jq -s 'sort_by(.timestamp)'

# Find related requests
jq 'select(.userId=="$USER_ID")' logs.json | jq -s 'group_by(.requestId)'
```

### Step 4: Generate Report

```typescript
interface IncidentReport {
  summary: string;
  timeline: TimelineEvent[];
  rootCause: string;
  impact: {
    users: number;
    requests: number;
    duration: string;
  };
  relatedLogs: StructuredLog[];
  recommendations: string[];
}
```

## AI-Assisted Log Analysis

```markdown
Analyze these logs and provide:

1. **Summary** - What happened in plain English
2. **Root Cause** - Most likely cause of the issue
3. **Timeline** - Key events in chronological order
4. **Impact** - What was affected
5. **Recommendations** - How to fix and prevent

Logs:
```[paste logs]```
```

## Best Practices

1. **Use structured logging** - JSON, not plain text
2. **Include trace IDs** - Essential for distributed systems
3. **Log at appropriate levels** - Don't flood with debug
4. **Include context** - Who, what, when, where
5. **Centralize logs** - One place to search
6. **Set retention policies** - Balance cost and needs
7. **Automate analysis** - Don't wait for incidents
