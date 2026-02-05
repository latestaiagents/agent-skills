---
name: incident-commander
description: |
  Guide incident response as an Incident Commander with structured communication and coordination.
  Use this skill when there's an active incident, outage, service degradation, or production issue.
  Activate when: incident, outage, service down, production issue, SEV1, SEV2, pages, alerts firing,
  something broke, users complaining, error spike, latency spike.
---

# Incident Commander Guide

**Lead incident response with structured communication, clear ownership, and systematic resolution.**

## Incident Commander Role

The IC (Incident Commander) is responsible for:
- **Coordination**: Ensuring right people are engaged
- **Communication**: Keeping stakeholders informed
- **Decision-making**: Making calls when consensus isn't possible
- **Documentation**: Ensuring timeline is captured

**The IC does NOT need to be the person fixing the problem.**

## Incident Severity Levels

| Level | Description | Response Time | Examples |
|-------|-------------|---------------|----------|
| **SEV1** | Critical - Complete outage | Immediate | Total service down, data loss, security breach |
| **SEV2** | Major - Significant impact | 15 min | Core feature broken, major degradation |
| **SEV3** | Minor - Limited impact | 1 hour | Non-critical feature down, workaround exists |
| **SEV4** | Low - Minimal impact | Best effort | Cosmetic, single user affected |

## Incident Response Workflow

### Phase 1: Detection & Triage (0-5 minutes)

```
1. Acknowledge the alert
2. Quick assessment:
   - What's broken?
   - Who's affected?
   - What's the blast radius?
3. Assign severity level
4. Declare incident if SEV1/SEV2
```

### Phase 2: Mobilization (5-15 minutes)

```
1. Create incident channel: #inc-YYYYMMDD-[brief-description]
2. Post initial summary (template below)
3. Page relevant teams if needed
4. Assign roles:
   - IC (Incident Commander) - you or delegate
   - Tech Lead - driving investigation
   - Comms Lead - external communication
```

**Initial Incident Post Template:**

```
🚨 INCIDENT DECLARED

**Severity:** SEV-[X]
**Status:** Investigating
**Impact:** [Who/what is affected]
**Started:** [Time] UTC

**Current Understanding:**
[Brief description of symptoms]

**Roles:**
- IC: @[name]
- Tech Lead: @[name]
- Comms: @[name]

**Next Update:** [Time] (every 15-30 min for SEV1/2)
```

### Phase 3: Investigation (Ongoing)

```
1. Gather data:
   - Recent deployments?
   - Configuration changes?
   - External dependency issues?
   - Error patterns in logs?
   - Metrics anomalies?

2. Form hypothesis and test

3. Identify mitigation options:
   - Can we rollback?
   - Can we scale?
   - Can we failover?
   - Do we need a hotfix?
```

### Phase 4: Mitigation

```
1. Choose mitigation approach
2. Communicate plan before executing
3. Execute with verification at each step
4. Monitor for improvement
5. Confirm resolution
```

### Phase 5: Resolution

```
1. Verify service is healthy
2. Update status page
3. Send resolution communication
4. Create postmortem ticket
5. Schedule postmortem meeting (within 48h for SEV1/2)
```

## Communication Templates

### Status Update (Every 15-30 min)

```
📊 INCIDENT UPDATE - [Time] UTC

**Status:** [Investigating/Identified/Mitigating/Resolved]
**Impact:** [Current impact]

**Update:**
[What we've learned, what we're doing]

**Next Steps:**
[What's happening next]

**Next Update:** [Time] UTC
```

### Resolution Communication

```
✅ INCIDENT RESOLVED - [Time] UTC

**Duration:** [X hours Y minutes]
**Root Cause:** [Brief description]
**Resolution:** [What fixed it]

**Impact Summary:**
- Users affected: [number]
- Duration: [time]
- SLA impact: [yes/no]

**Next Steps:**
- Postmortem scheduled: [date/time]
- Postmortem doc: [link]

Thank you to everyone who helped respond.
```

## Best Practices

### DO
- Keep incident channel focused on resolution
- Use threads for side discussions
- Update status page early and often
- Make decisions when stuck (bias for action)
- Rotate IC if incident is long (>4 hours)
- Take breaks - fatigue causes mistakes

### DON'T
- Blame individuals
- Make changes without communicating
- Forget to update stakeholders
- Skip the postmortem
- Let scope creep into fixing unrelated issues

## Role Cheat Sheet

| Role | Responsibility | Who |
|------|----------------|-----|
| **IC** | Coordination, decisions, communication | Declared or on-call |
| **Tech Lead** | Investigation, fix implementation | SME for affected service |
| **Comms Lead** | Status page, customer comms | Support/Comms team |
| **Scribe** | Document timeline | Anyone available |
| **Subject Matter Experts** | Deep knowledge | Paged as needed |

## Escalation Triggers

Escalate to leadership when:
- SEV1 lasting >30 minutes
- Data breach suspected
- Customer SLA breach imminent
- Media attention expected
- Regulatory notification required
