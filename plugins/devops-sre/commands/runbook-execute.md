---
description: Execute runbook procedures step-by-step with safety checks and verification
---

# /runbook-execute

Execute runbook procedures with guided steps, safety checks, and verification at each stage.

## What I Need

Tell me:
- Which runbook to execute (or describe the problem)
- Target environment (prod/staging/dev)
- Any context about current state

## How It Works

### Step 1: Runbook Selection

I'll help identify the right runbook:

```
Available Runbooks:
├── Service Restart
│   ├── api-service-restart
│   ├── worker-restart
│   └── cache-flush-restart
├── Scaling
│   ├── horizontal-scale-up
│   ├── horizontal-scale-down
│   └── vertical-resize
├── Database
│   ├── connection-pool-reset
│   ├── slow-query-kill
│   └── replica-failover
├── Deployment
│   ├── rollback-deployment
│   ├── canary-promote
│   └── feature-flag-toggle
└── Recovery
    ├── circuit-breaker-reset
    ├── queue-drain
    └── cache-invalidation
```

### Step 2: Pre-Execution Checks

Before any action:

```
┌─────────────────────────────────────────────────────────────┐
│                    PRE-EXECUTION CHECKLIST                  │
├─────────────────────────────────────────────────────────────┤
│  ✓ Runbook: [name]                                         │
│  ✓ Environment: [prod/staging]                             │
│  ✓ Current on-call: [name]                                 │
│  ✓ Active incidents: [none / INC-XXX]                      │
│  ✓ Recent deployments: [list last 24h]                     │
│  ✓ Change freeze: [not active / ACTIVE - requires approval]│
├─────────────────────────────────────────────────────────────┤
│  ⚠️  Impact Assessment:                                     │
│     - Expected downtime: [X seconds/minutes]               │
│     - Affected services: [list]                            │
│     - User impact: [description]                           │
└─────────────────────────────────────────────────────────────┘

Proceed with execution? [y/N]
```

### Step 3: Guided Execution

Each step runs with verification:

```
Step 1 of 5: Drain connections
├── Command: kubectl drain node-1 --grace-period=30
├── Expected: Node marked as unschedulable
├── Timeout: 60 seconds
│
├── Executing... ████████████████████ 100%
├── Result: ✓ Success
├── Verification: Node status is "SchedulingDisabled"
│
└── Continue to Step 2? [y/N/abort]
```

### Step 4: Rollback Points

At critical steps, I'll create rollback points:

```
💾 Rollback Point Created: step-3-pre-restart
   To rollback: /runbook-execute rollback step-3-pre-restart
```

### Step 5: Post-Execution Verification

After completion:

```
┌─────────────────────────────────────────────────────────────┐
│                  EXECUTION COMPLETE                         │
├─────────────────────────────────────────────────────────────┤
│  Status: ✓ SUCCESS                                         │
│  Duration: 3m 42s                                          │
│  Steps completed: 5/5                                      │
├─────────────────────────────────────────────────────────────┤
│  Health Checks:                                             │
│  ✓ Service responding (latency: 45ms)                      │
│  ✓ Error rate normal (0.01%)                               │
│  ✓ All pods healthy (5/5)                                  │
│  ✓ Database connections stable                             │
├─────────────────────────────────────────────────────────────┤
│  Next: Monitor for 15 minutes before closing               │
└─────────────────────────────────────────────────────────────┘
```

## Safety Features

| Feature | Description |
|---------|-------------|
| **Dry-run mode** | Preview commands without executing |
| **Step-by-step** | Confirm each step before proceeding |
| **Verification** | Automated checks after each step |
| **Rollback points** | Revert to known-good state |
| **Audit logging** | Full record of all actions |
| **Timeout protection** | Auto-abort if step takes too long |

## Quick Commands

```
/runbook-execute list              # List available runbooks
/runbook-execute [name]            # Execute specific runbook
/runbook-execute [name] --dry-run  # Preview without executing
/runbook-execute rollback [point]  # Rollback to checkpoint
/runbook-execute status            # Check execution status
```
