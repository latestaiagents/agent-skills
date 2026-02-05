---
description: Pre and post deployment checklist with automated verification and rollback preparation
---

# /deploy-checklist

Comprehensive deployment checklist with automated verification and rollback preparation.

## What I Need

Tell me:
- What are you deploying? (service name, version)
- Target environment (prod/staging)
- Deployment method (rolling/blue-green/canary)

## Pre-Deployment Checklist

```
┌─────────────────────────────────────────────────────────────┐
│              PRE-DEPLOYMENT CHECKLIST                       │
│              Service: api-service v2.3.4                    │
│              Environment: production                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  📋 REQUIREMENTS                                            │
│  ✓ Code reviewed and approved                               │
│  ✓ Tests passing (unit: 342/342, integration: 89/89)       │
│  ✓ Security scan clean (0 high, 2 medium - acknowledged)   │
│  ✓ Staging deployment successful                            │
│  ✓ QA sign-off received                                     │
│                                                              │
│  🚫 BLOCKERS CHECK                                          │
│  ✓ No active incidents (checked PagerDuty)                 │
│  ✓ No change freeze active                                  │
│  ✓ Not during peak hours (current: 2.3k RPM, peak: 10k)   │
│  ✓ On-call engineer available (@jane)                      │
│                                                              │
│  📊 BASELINE METRICS (capture before deploy)                │
│  • Error rate: 0.08%                                        │
│  • P50 latency: 45ms                                        │
│  • P99 latency: 230ms                                       │
│  • Active connections: 1,247                                │
│  • Memory usage: 67%                                        │
│  • CPU usage: 34%                                           │
│                                                              │
│  🔙 ROLLBACK PREPARATION                                    │
│  ✓ Previous version available: v2.3.3                       │
│  ✓ Rollback command ready:                                  │
│    kubectl rollout undo deployment/api-service              │
│  ✓ Rollback tested in staging: Yes                         │
│  ✓ Database migrations: Backward compatible ✓              │
│                                                              │
│  📢 COMMUNICATION                                           │
│  ✓ #deployments channel notified                           │
│  ✓ Status page: No maintenance needed                       │
│  □ Customer notification: Not required                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘

All checks passed. Ready to deploy? [y/N]
```

## During Deployment Monitoring

```
┌─────────────────────────────────────────────────────────────┐
│              DEPLOYMENT IN PROGRESS                         │
│              api-service v2.3.3 → v2.3.4                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Progress: ████████████░░░░░░░░ 60%                        │
│  Strategy: Rolling (2 pods at a time)                       │
│  Elapsed: 3m 24s                                            │
│                                                              │
│  Pod Status:                                                 │
│  ✓ api-service-7b8c9-abc12  v2.3.4  Running  (2m ago)      │
│  ✓ api-service-7b8c9-def34  v2.3.4  Running  (2m ago)      │
│  ✓ api-service-7b8c9-ghi56  v2.3.4  Running  (1m ago)      │
│  ◐ api-service-7b8c9-jkl78  v2.3.4  Starting...            │
│  • api-service-6a7b8-mno90  v2.3.3  Running  (terminating) │
│                                                              │
│  Live Metrics (vs baseline):                                │
│  • Error rate: 0.09% (baseline: 0.08%) ✓                   │
│  • P99 latency: 245ms (baseline: 230ms) ✓                  │
│  • Traffic: 2,456 RPM ✓                                     │
│                                                              │
│  ⚠️  Auto-rollback triggers:                                │
│  • Error rate > 1%                                          │
│  • P99 latency > 500ms                                      │
│  • Pod crash loops                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Post-Deployment Verification

```
┌─────────────────────────────────────────────────────────────┐
│              POST-DEPLOYMENT VERIFICATION                   │
│              api-service v2.3.4                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ✓ All pods healthy (5/5 running)                          │
│  ✓ Health checks passing                                    │
│  ✓ No error spike detected                                  │
│  ✓ Latency within SLO                                       │
│  ✓ No alerts fired                                          │
│                                                              │
│  Smoke Tests:                                                │
│  ✓ GET /health - 200 OK (23ms)                             │
│  ✓ GET /api/users - 200 OK (67ms)                          │
│  ✓ POST /api/orders - 201 Created (134ms)                  │
│  ✓ Auth flow - Success                                      │
│                                                              │
│  Recommendation: Monitor for 15 minutes                     │
│  Rollback window: 1 hour (until database cleanup job)       │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Deployment successful! 🎉
```

## Quick Commands

```
/deploy-checklist pre             # Pre-deployment checks only
/deploy-checklist monitor         # Start deployment monitoring
/deploy-checklist post            # Post-deployment verification
/deploy-checklist rollback        # Execute rollback
```
