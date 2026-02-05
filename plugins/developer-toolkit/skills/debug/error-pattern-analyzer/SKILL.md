---
name: error-pattern-analyzer
description: |
  Use this skill when analyzing error patterns in applications. Activate when the user has recurring errors,
  wants to find root causes of issues, needs to identify systemic problems, is analyzing error logs,
  or wants to categorize and prioritize bugs.
---

# Error Pattern Analyzer

Identify recurring error patterns and systemic issues in your codebase.

## When to Use

- Same errors appearing repeatedly
- Investigating production incidents
- Prioritizing bug fixes
- Finding root causes
- Analyzing error trends

## Error Classification

### By Frequency

| Category | Frequency | Priority |
|----------|-----------|----------|
| Critical | >100/hour | P0 - Immediate |
| High | 10-100/hour | P1 - Same day |
| Medium | 1-10/hour | P2 - This week |
| Low | <1/hour | P3 - Backlog |

### By Impact

```typescript
interface ErrorImpact {
  affectedUsers: number;
  revenue: 'blocking' | 'degraded' | 'none';
  dataIntegrity: 'at_risk' | 'safe';
  cascading: boolean; // Does it cause other failures?
}
```

## Pattern Detection Workflow

### Step 1: Collect Error Data

```typescript
interface ErrorRecord {
  timestamp: Date;
  errorType: string;
  message: string;
  stackTrace: string;
  context: {
    userId?: string;
    endpoint?: string;
    input?: unknown;
    environment: string;
  };
  metadata: Record<string, unknown>;
}

// Aggregate errors over time window
async function collectErrors(
  timeRange: { start: Date; end: Date }
): Promise<ErrorRecord[]> {
  return errorStore.query({
    timestamp: { $gte: timeRange.start, $lte: timeRange.end }
  });
}
```

### Step 2: Group by Similarity

```typescript
function groupErrors(errors: ErrorRecord[]): ErrorGroup[] {
  const groups = new Map<string, ErrorRecord[]>();

  for (const error of errors) {
    // Create fingerprint from error type + normalized message
    const fingerprint = createFingerprint(error);
    const existing = groups.get(fingerprint) || [];
    existing.push(error);
    groups.set(fingerprint, existing);
  }

  return Array.from(groups.entries()).map(([fp, errs]) => ({
    fingerprint: fp,
    count: errs.length,
    firstSeen: errs[0].timestamp,
    lastSeen: errs[errs.length - 1].timestamp,
    sample: errs[0],
    affectedUsers: new Set(errs.map(e => e.context.userId)).size
  }));
}

function createFingerprint(error: ErrorRecord): string {
  // Normalize message (remove variable parts)
  const normalizedMessage = error.message
    .replace(/\b[0-9a-f]{8,}\b/gi, '<ID>') // UUIDs, hashes
    .replace(/\d{4}-\d{2}-\d{2}/g, '<DATE>') // Dates
    .replace(/\d+/g, '<N>'); // Numbers

  // Use first meaningful stack frame
  const keyFrame = extractKeyFrame(error.stackTrace);

  return `${error.errorType}:${normalizedMessage}:${keyFrame}`;
}
```

### Step 3: Identify Patterns

```typescript
interface ErrorPattern {
  name: string;
  description: string;
  affectedGroups: ErrorGroup[];
  rootCause: string;
  suggestedFix: string;
  confidence: number;
}

const KNOWN_PATTERNS = [
  {
    name: 'Null Pointer Pattern',
    detector: (group: ErrorGroup) =>
      /cannot read property|undefined is not|null pointer/i.test(group.sample.message),
    rootCause: 'Missing null checks on data access',
    suggestedFix: 'Add optional chaining or explicit null checks'
  },
  {
    name: 'Race Condition Pattern',
    detector: (group: ErrorGroup) => {
      // Same error at similar times from different users
      const timestamps = group.errors.map(e => e.timestamp.getTime());
      const clusters = findTimeClusters(timestamps, 1000);
      return clusters.some(c => c.length > 5);
    },
    rootCause: 'Concurrent access without proper synchronization',
    suggestedFix: 'Implement locking or use atomic operations'
  },
  {
    name: 'Resource Exhaustion Pattern',
    detector: (group: ErrorGroup) =>
      /too many|limit exceeded|quota|memory|timeout/i.test(group.sample.message),
    rootCause: 'Resource limits being hit under load',
    suggestedFix: 'Implement pooling, caching, or increase limits'
  },
  {
    name: 'Integration Failure Pattern',
    detector: (group: ErrorGroup) =>
      /ECONNREFUSED|ETIMEDOUT|network|api.*failed/i.test(group.sample.message),
    rootCause: 'External service unavailability',
    suggestedFix: 'Add circuit breakers and fallback mechanisms'
  }
];

function detectPatterns(groups: ErrorGroup[]): ErrorPattern[] {
  const patterns: ErrorPattern[] = [];

  for (const detector of KNOWN_PATTERNS) {
    const matches = groups.filter(g => detector.detector(g));
    if (matches.length > 0) {
      patterns.push({
        name: detector.name,
        description: detector.rootCause,
        affectedGroups: matches,
        rootCause: detector.rootCause,
        suggestedFix: detector.suggestedFix,
        confidence: calculateConfidence(matches)
      });
    }
  }

  return patterns;
}
```

### Step 4: Analyze Trends

```typescript
interface ErrorTrend {
  pattern: string;
  direction: 'increasing' | 'decreasing' | 'stable';
  changePercent: number;
  prediction: {
    nextHour: number;
    nextDay: number;
  };
}

function analyzeTrends(
  current: ErrorGroup[],
  previous: ErrorGroup[]
): ErrorTrend[] {
  const trends: ErrorTrend[] = [];

  for (const group of current) {
    const prevGroup = previous.find(p => p.fingerprint === group.fingerprint);
    const prevCount = prevGroup?.count || 0;

    const changePercent = prevCount > 0
      ? ((group.count - prevCount) / prevCount) * 100
      : 100;

    trends.push({
      pattern: group.fingerprint,
      direction: changePercent > 10 ? 'increasing'
        : changePercent < -10 ? 'decreasing' : 'stable',
      changePercent,
      prediction: predictFuture(group)
    });
  }

  return trends.sort((a, b) =>
    Math.abs(b.changePercent) - Math.abs(a.changePercent)
  );
}
```

## Common Error Patterns

### 1. The Cascade Pattern

```
Error A → Triggers Error B → Triggers Error C
```

**Detection:**
```typescript
function detectCascade(errors: ErrorRecord[]): CascadeChain[] {
  const chains: CascadeChain[] = [];

  // Sort by timestamp
  const sorted = errors.sort((a, b) =>
    a.timestamp.getTime() - b.timestamp.getTime()
  );

  // Look for rapid succession of different errors
  for (let i = 0; i < sorted.length - 2; i++) {
    const window = sorted.slice(i, i + 5);
    const uniqueTypes = new Set(window.map(e => e.errorType));

    if (uniqueTypes.size >= 3 &&
        window[4].timestamp.getTime() - window[0].timestamp.getTime() < 5000) {
      chains.push({
        errors: window,
        rootError: window[0],
        cascadeDepth: uniqueTypes.size
      });
    }
  }

  return chains;
}
```

### 2. The Thundering Herd Pattern

Many errors at exactly the same time after a trigger event.

**Detection:**
```typescript
function detectThunderingHerd(errors: ErrorRecord[]): HerdEvent[] {
  const bySecond = new Map<number, ErrorRecord[]>();

  for (const error of errors) {
    const second = Math.floor(error.timestamp.getTime() / 1000);
    const existing = bySecond.get(second) || [];
    existing.push(error);
    bySecond.set(second, existing);
  }

  return Array.from(bySecond.entries())
    .filter(([_, errs]) => errs.length > 50) // Threshold
    .map(([second, errs]) => ({
      timestamp: new Date(second * 1000),
      count: errs.length,
      triggerEvent: identifyTrigger(errs)
    }));
}
```

### 3. The Slow Leak Pattern

Errors gradually increasing over time (memory leak, connection leak).

**Detection:**
```typescript
function detectSlowLeak(
  groups: ErrorGroup[],
  hours: number = 24
): LeakPattern[] {
  return groups.filter(group => {
    const hourlyRates = calculateHourlyRates(group.errors, hours);

    // Check for consistent upward trend
    const trend = calculateTrendLine(hourlyRates);
    return trend.slope > 0.1 && trend.r2 > 0.7; // Increasing with good fit
  }).map(group => ({
    pattern: group.fingerprint,
    currentRate: hourlyRates[hourlyRates.length - 1],
    projectedRate24h: extrapolate(hourlyRates, 24),
    confidence: trend.r2
  }));
}
```

## Report Generation

```typescript
interface ErrorReport {
  summary: {
    totalErrors: number;
    uniquePatterns: number;
    affectedUsers: number;
    topPattern: string;
  };
  criticalIssues: ErrorPattern[];
  trends: ErrorTrend[];
  recommendations: Recommendation[];
}

function generateReport(
  errors: ErrorRecord[],
  timeRange: { start: Date; end: Date }
): ErrorReport {
  const groups = groupErrors(errors);
  const patterns = detectPatterns(groups);

  return {
    summary: {
      totalErrors: errors.length,
      uniquePatterns: groups.length,
      affectedUsers: new Set(errors.map(e => e.context.userId)).size,
      topPattern: groups.sort((a, b) => b.count - a.count)[0]?.fingerprint || 'none'
    },
    criticalIssues: patterns
      .filter(p => p.affectedGroups.some(g => g.count > 100))
      .sort((a, b) => b.confidence - a.confidence),
    trends: analyzeTrends(groups, previousGroups),
    recommendations: generateRecommendations(patterns)
  };
}
```

## AI-Assisted Analysis

```markdown
Analyze these error patterns and provide:

1. **Root Cause Hypothesis**
   - Most likely cause
   - Alternative explanations
   - What data would confirm/deny

2. **Impact Assessment**
   - Affected functionality
   - User impact
   - Business impact

3. **Fix Priority**
   - Immediate actions needed
   - Short-term fixes
   - Long-term improvements

4. **Prevention Strategy**
   - How to prevent recurrence
   - Monitoring to add
   - Tests to write

Error Data:
```[paste error groups]```
```

## Best Practices

1. **Normalize before grouping** - Remove variable data
2. **Track over time** - Point-in-time is misleading
3. **Correlate with changes** - Errors often follow deploys
4. **Alert on trends** - Not just absolute numbers
5. **Automate analysis** - Run regularly, not just during incidents
