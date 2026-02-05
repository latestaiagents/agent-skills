---
description: Generate comprehensive on-call handoff reports with active issues, context, and watch items
---

# /on-call-handoff

Generate a comprehensive handoff report for on-call rotation transitions.

## What I Need

Tell me:
- Your name (outgoing on-call)
- Incoming on-call engineer's name
- Any specific issues to highlight

Or just run `/on-call-handoff` and I'll gather from connected tools.

## Generated Handoff Report

```markdown
# On-Call Handoff Report

**From:** [Outgoing Engineer]
**To:** [Incoming Engineer]
**Rotation:** [Team Name]
**Handoff Time:** YYYY-MM-DD HH:MM UTC

---

## 🚨 Active Incidents

| ID | Severity | Status | Summary | Started |
|----|----------|--------|---------|---------|
| INC-123 | SEV2 | Investigating | API latency spike | 2h ago |

**INC-123 Context:**
- Root cause: Suspected database connection pool exhaustion
- Current state: Scaled up read replicas, monitoring
- Next steps: Review connection pooling config tomorrow
- Key contacts: @dba-team if it recurs

---

## ⚠️ Watch Items

### High Priority
1. **Deployment in progress** - v2.3.4 rolling out to prod-west
   - Canary at 10%, promoting to 50% at 14:00 UTC
   - Rollback command ready: `kubectl rollout undo deploy/api`

2. **Elevated error rate** - payments-service at 0.5% (normal: 0.1%)
   - Started after morning deploy, monitoring
   - No user complaints yet

### Medium Priority
3. **Planned maintenance** - RDS maintenance window tonight 02:00-04:00 UTC
   - Expected: 30s failover, auto-reconnect should handle
   - Runbook: `database/rds-maintenance-checklist`

---

## 📊 System Health Summary

| Service | Status | Notes |
|---------|--------|-------|
| api-gateway | 🟢 Healthy | |
| user-service | 🟢 Healthy | |
| payments | 🟡 Degraded | Elevated errors, monitoring |
| search | 🟢 Healthy | |
| notifications | 🟢 Healthy | |

**Key Metrics (Last 24h):**
- Error rate: 0.12% (SLO: <0.5%) ✓
- P99 latency: 245ms (SLO: <500ms) ✓
- Availability: 99.95% (SLO: 99.9%) ✓

---

## 📝 Shift Summary

### Incidents Handled
- INC-121 (SEV3) - Resolved - Cache invalidation issue
- INC-122 (SEV4) - Resolved - Flaky test in CI

### Changes Made
- Scaled worker pool 3→5 instances (can scale back Monday)
- Updated alerting threshold for memory usage (was too noisy)

### Deferred Work
- [ ] Investigate slow query on reports endpoint (non-urgent)
- [ ] Update runbook for certificate rotation

---

## 📞 Key Contacts

| Role | Name | Reach |
|------|------|-------|
| Secondary On-Call | @jane | Slack, PagerDuty |
| Engineering Manager | @bob | Slack (daytime only) |
| DBA On-Call | @dba-team | PagerDuty |
| Security | @security-team | PagerDuty (SEV1 only) |

---

## 🔗 Quick Links

- [Runbooks](https://wiki.company.com/runbooks)
- [Dashboards](https://datadog.com/dashboard/main)
- [Incident Slack](https://slack.com/channels/incidents)
- [PagerDuty Schedule](https://pagerduty.com/schedules)
- [Status Page Admin](https://statuspage.io/admin)

---

*Handoff generated at YYYY-MM-DD HH:MM UTC*
*Good luck! 🍀*
```

## Data Sources

When connected, I'll pull from:
- **PagerDuty**: Active incidents, on-call schedule
- **Datadog**: System health, metrics, active alerts
- **Slack**: Recent incident channels, discussions
- **GitHub**: Recent deployments, open PRs
- **Jira**: Related tickets, known issues

## Quick Commands

```
/on-call-handoff                   # Generate full handoff report
/on-call-handoff brief             # Quick summary only
/on-call-handoff to @username      # Specify incoming engineer
```
