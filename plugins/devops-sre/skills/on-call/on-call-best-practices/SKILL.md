---
name: on-call-best-practices
description: |
  Manage on-call rotations with sustainable practices, fair scheduling, and effective handoffs.
  Use this skill when setting up on-call, improving on-call experience, or managing rotations.
  Activate when: on-call, pagerduty, rotation, schedule, handoff, on-call burden, being paged,
  night pages, weekend on-call, on-call fatigue.
---

# On-Call Best Practices

**Sustainable on-call that protects engineers and keeps systems reliable.**

## On-Call Philosophy

> "On-call should be a learning opportunity, not a punishment."

### Core Principles
1. **Fair distribution** - Burden shared equally
2. **Sustainable pace** - No burnout
3. **Clear expectations** - Everyone knows their role
4. **Continuous improvement** - Learn from every incident

## Rotation Design

### Recommended Structure

```
Primary On-Call (24/7):
- First responder for all pages
- 1 week shifts (max)
- Clear handoff process

Secondary On-Call (24/7):
- Backup if primary unavailable
- Can be shadow for training
- Steps in if primary overloaded

Business Hours Escalation:
- Subject matter experts
- Available for complex issues
- Not paged at night
```

### Rotation Schedule Example

```
Week    Mon    Tue    Wed    Thu    Fri    Sat    Sun
───────────────────────────────────────────────────────
Jan 6   Alice  Alice  Alice  Alice  Alice  Alice  Alice
Jan 13  Bob    Bob    Bob    Bob    Bob    Bob    Bob
Jan 20  Carol  Carol  Carol  Carol  Carol  Carol  Carol
Jan 27  Dave   Dave   Dave   Dave   Dave   Dave   Dave
Feb 3   Alice  ...

Secondary follows same pattern, offset by 1 week
```

### Scheduling Guidelines

| Guideline | Recommendation |
|-----------|----------------|
| Shift length | 1 week max, shorter if high volume |
| Gap between shifts | 2+ weeks minimum |
| Consecutive nights | Comp time if >2 pages |
| Holidays | Volunteer-based, compensated |
| Team size | 4+ people for sustainable rotation |

## Response Expectations

### Response Time SLAs

| Severity | Acknowledge | Respond | Escalate If |
|----------|-------------|---------|-------------|
| SEV1 | 5 min | Immediate | No ack in 5 min |
| SEV2 | 15 min | 30 min | No ack in 15 min |
| SEV3 | 1 hour | 4 hours | Business hours |
| SEV4 | Best effort | Next day | N/A |

### What "On-Call" Means

```
During your shift:
✓ Phone charged and with you
✓ Laptop accessible within 15 min
✓ Reliable internet access
✓ Not impaired (alcohol, etc.)
✓ Able to focus if paged

You are NOT expected to:
✗ Be at your desk 24/7
✗ Respond instantly to Slack
✗ Fix everything yourself
✗ Work normal hours + on-call
```

## Handoff Protocol

### End of Shift Checklist

```markdown
## On-Call Handoff

**Outgoing:** @alice
**Incoming:** @bob
**Date:** 2026-01-13 09:00 UTC

### Active Issues
- [ ] INC-123: Monitoring elevated error rate (context: ...)
- [ ] Deployment in progress: api-service v2.3.4

### Watch Items
- Payment processor maintenance tonight 02:00-04:00 UTC
- New monitoring rolled out, may be noisy

### Recent Incidents
- INC-121: Resolved, postmortem scheduled Friday
- INC-122: Resolved, no action needed

### Runbook Updates
- Updated: database/connection-pool-reset (added step 3)
- Outdated: search/reindex (needs review)

### Notes
- Had 3 pages this week, all during business hours
- Nothing woke me up at night
- Good luck! 🍀
```

### Verbal Handoff (5-10 minutes)

```
1. Walk through active issues
2. Highlight anything unusual
3. Share context not in writing
4. Confirm contact info current
5. Test page to verify setup
```

## Reducing On-Call Burden

### Metrics to Track

| Metric | Healthy | Action Needed |
|--------|---------|---------------|
| Pages/week | <5 | Review alert thresholds |
| Night pages/week | <1 | Investigate or fix root causes |
| MTTA | <5 min | Check notification settings |
| Time to resolve | <30 min avg | Improve runbooks |
| % actionable | >80% | Reduce noisy alerts |

### Improvement Strategies

```
1. Fix the root cause
   - Every incident should have action items
   - Track action item completion

2. Improve detection
   - Catch issues before they page
   - Add canary deployments

3. Automate remediation
   - Auto-restart crashed services
   - Auto-scale on high load
   - Self-healing infrastructure

4. Improve runbooks
   - Clear, tested procedures
   - One-click remediation where possible

5. Reduce noise
   - Tune alert thresholds
   - Add deduplication
   - Use proper severity levels
```

## Compensation & Support

### Fair Compensation

```
Recommended compensation models:

1. Stipend Model
   - Fixed amount per on-call week
   - Example: $500/week on-call

2. Per-Page Model
   - Base stipend + per-page bonus
   - Example: $200/week + $50/page

3. Comp Time Model
   - Time off for night/weekend pages
   - Example: 2 hours off per night page

4. Combined Model
   - Stipend + comp time for disruption
   - Most engineer-friendly
```

### Support Structures

```
✓ Clear escalation paths
✓ Secondary on-call backup
✓ Manager support for difficult situations
✓ Mental health resources
✓ Training and shadowing for new on-callers
✓ Blameless postmortem culture
```

## Training New On-Callers

### Shadow Program

```
Week 1: Observe
- Shadow primary on-call
- Read all runbooks
- Review recent incidents

Week 2: Assisted
- Take some pages with backup
- Primary available immediately
- Debrief after each incident

Week 3: Primary with Safety Net
- Primary on-call
- Experienced shadow
- Extended escalation time

Week 4+: Full Primary
- Normal on-call duties
- Standard escalation paths
```

### Required Knowledge

```
□ Access to all systems
□ Can reach all tools (VPN, etc.)
□ Know escalation paths
□ Reviewed all runbooks
□ Understand SLO/SLA
□ Know who SMEs are
□ Have done a test page
□ Know how to declare incident
```

## Well-Being

### Signs of On-Call Burnout

- Dreading on-call shifts
- Anxiety about phone notifications
- Sleep disruption even when not paged
- Decreased job satisfaction
- Avoidance of learning new systems

### Prevention

```
1. Sustainable rotation size (4+ people)
2. Enforce gap between shifts
3. Comp time for disruption
4. Regular feedback loops
5. Continuously reduce burden
6. Leadership does on-call too
```

### If You're Struggling

```
- Talk to your manager
- Request temporary rotation skip
- Ask for additional support
- Suggest rotation improvements
- It's okay to ask for help
```
