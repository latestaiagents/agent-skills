---
name: root-cause-analysis
description: |
  Systematic root cause analysis using 5 Whys, fishbone diagrams, and fault tree analysis.
  Use this skill when investigating why an incident happened, performing RCA, or writing postmortems.
  Activate when: root cause, why did this happen, 5 whys, incident analysis, postmortem investigation,
  how did this happen, what caused, failure analysis.
---

# Root Cause Analysis (RCA)

**Find the real cause, not just the symptoms, to prevent recurrence.**

## RCA Principles

1. **Look for systems failures**, not human errors
2. **Ask "why" until you find actionable causes**
3. **Multiple contributing factors** are common
4. **Prevention > blame**

## Method 1: 5 Whys

Keep asking "why" until you reach an actionable root cause.

### Example: API Outage

```
Problem: API returned 500 errors for 45 minutes

Why #1: Why did the API return 500 errors?
→ The database connection pool was exhausted

Why #2: Why was the connection pool exhausted?
→ Connections weren't being released after queries

Why #3: Why weren't connections being released?
→ A code change introduced a bug that skipped connection.close()

Why #4: Why wasn't this caught before production?
→ Our integration tests don't check for connection leaks

Why #5: Why don't integration tests check for connection leaks?
→ We haven't implemented connection pool monitoring in tests

ROOT CAUSE: Missing connection leak detection in test suite
ACTION: Add connection pool assertions to integration tests
```

### 5 Whys Guidelines

| Do | Don't |
|----|-------|
| Use data, not assumptions | Stop at "human error" |
| Consider multiple branches | Accept vague answers |
| Verify each "because" | Skip to conclusions |
| Look for systemic issues | Blame individuals |

## Method 2: Contributing Factors Analysis

Most incidents have multiple contributing factors.

```
┌─────────────────────────────────────────────────────────────┐
│                    INCIDENT: API OUTAGE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Direct Cause:                                               │
│  └─ Database connection pool exhaustion                     │
│                                                              │
│  Contributing Factors:                                       │
│  ├─ [Code] Connection leak bug in PR #1234                  │
│  ├─ [Process] Code review didn't catch the bug              │
│  ├─ [Testing] No connection leak tests                      │
│  ├─ [Monitoring] No alert for connection pool usage         │
│  ├─ [Deploy] Deployed during high-traffic period            │
│  └─ [Recovery] Runbook for this scenario was outdated       │
│                                                              │
│  Environmental Factors:                                      │
│  ├─ Team was understaffed (vacation season)                 │
│  └─ Similar incident 6 months ago, action items incomplete  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Method 3: Fault Tree Analysis

Work backwards from failure to identify all paths.

```
                    [API Outage]
                         │
            ┌────────────┴────────────┐
            │                         │
    [DB Connections              [App Server
     Exhausted]                   Crashed]
            │                         │
    ┌───────┴───────┐                │
    │               │                │
[Connection    [Too Many         [OOM
 Leak]         Requests]          Error]
    │               │                │
    │         ┌─────┴─────┐          │
    │         │           │          │
[Bug in    [Traffic    [Missing   [Memory
 Code]      Spike]     Rate       Leak]
              │        Limit]
              │
        [Marketing
         Campaign]
```

## Method 4: Timeline Reconstruction

Detailed timeline helps identify the chain of events.

```
Timeline: API Outage - 2026-01-15

Time (UTC)  | Event                        | Source
------------|------------------------------|--------
09:00       | Deploy v2.3.4 started        | GitHub
09:15       | Deploy completed             | K8s
09:45       | Marketing email sent (50k)   | Marketing
10:02       | Traffic spike begins         | Datadog
10:15       | Connection pool at 80%       | Metrics
10:23       | First 500 errors             | Logs
10:25       | Alert fired                  | PagerDuty
10:27       | On-call acknowledged         | PagerDuty
10:35       | Root cause identified        | Slack
10:42       | Rollback initiated           | K8s
10:48       | Service recovering           | Datadog
11:00       | All clear declared           | Slack

Key Finding: 38 minutes between deploy and issue detection
             Deploy + traffic spike = perfect storm
```

## Common Root Cause Categories

### Technical
- Code bugs
- Configuration errors
- Infrastructure failures
- Dependency failures
- Capacity limits

### Process
- Inadequate testing
- Missed code review
- Incomplete runbooks
- Poor change management
- Insufficient monitoring

### Organizational
- Understaffing
- Knowledge silos
- Communication gaps
- Incomplete training
- Technical debt

## Action Item Quality

Good action items are **SMART**:

| Criteria | Bad Example | Good Example |
|----------|-------------|--------------|
| **S**pecific | "Improve testing" | "Add connection pool leak test to CI" |
| **M**easurable | "Monitor better" | "Alert when pool > 80% for 5 min" |
| **A**ssignable | "Team should fix" | "@jane owns implementation" |
| **R**ealistic | "Rewrite entire system" | "Add circuit breaker to DB calls" |
| **T**ime-bound | "Soon" | "Complete by 2026-02-01" |

## RCA Template

```markdown
## Root Cause Analysis

### Direct Cause
[What directly caused the incident]

### 5 Whys Analysis
1. Why? → [Answer]
2. Why? → [Answer]
3. Why? → [Answer]
4. Why? → [Answer]
5. Why? → [Root cause]

### Contributing Factors
- **Technical:** [List]
- **Process:** [List]
- **Organizational:** [List]

### Why Wasn't This Caught?
- In development: [Why]
- In code review: [Why]
- In testing: [Why]
- In staging: [Why]
- By monitoring: [Why]

### Action Items
| Priority | Action | Owner | Due | Prevents |
|----------|--------|-------|-----|----------|
| P0 | [Action] | @name | [Date] | Direct cause |
| P1 | [Action] | @name | [Date] | Detection |
| P2 | [Action] | @name | [Date] | Future risk |
```

## Anti-Patterns to Avoid

1. **"Human error"** - The human made an error, but the system allowed it
2. **"Lack of attention"** - Why did the system require such attention?
3. **"Should have known"** - How could they have known?
4. **"Didn't follow procedure"** - Why was the procedure not followed?
5. **Single root cause** - Usually there are multiple contributing factors
