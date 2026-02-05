---
description: Start structured incident response with automatic triage, communication, and resolution tracking
---

# /incident-response

Initiate a structured incident response workflow with automated triage, stakeholder communication, and resolution tracking.

## What I Need

Tell me about the incident:
- What's broken? (service name, error messages, symptoms)
- When did it start? (or when was it detected)
- What's the impact? (users affected, revenue impact, SLA breach risk)
- Severity level if known (SEV1-4)

## Workflow

### Step 1: Rapid Triage (2 minutes)

I'll analyze the incident and determine:

```
┌─────────────────────────────────────────────────────────────┐
│                    INCIDENT TRIAGE                          │
├─────────────────────────────────────────────────────────────┤
│  Service:        [affected service]                         │
│  Started:        [timestamp]                                │
│  Severity:       SEV-[1-4]                                 │
│  Impact:         [user/business impact]                     │
│  Blast Radius:   [affected systems/regions]                │
├─────────────────────────────────────────────────────────────┤
│  Initial Hypothesis: [likely cause based on symptoms]       │
│  Recommended Actions: [immediate steps]                     │
└─────────────────────────────────────────────────────────────┘
```

### Step 2: Incident Channel Setup

I'll help you:
- Create incident Slack channel (#inc-YYYYMMDD-service)
- Post initial incident summary
- Page on-call if not already done
- Set up incident document

### Step 3: Investigation

Using connected observability tools, I'll:
- Query recent deployments (GitHub)
- Check error rates and latency (Datadog/Prometheus)
- Review recent alerts (PagerDuty)
- Examine logs for error patterns (CloudWatch)
- Check infrastructure changes (Terraform/K8s)

### Step 4: Mitigation Options

Based on investigation, I'll suggest:

| Option | Risk | Time | Recommendation |
|--------|------|------|----------------|
| Rollback | Low | 5min | If recent deploy |
| Scale up | Low | 2min | If capacity issue |
| Failover | Medium | 10min | If region issue |
| Hotfix | High | 30min+ | If code bug |

### Step 5: Communication Templates

I'll generate status updates for:
- Internal stakeholders (Slack)
- Status page (customers)
- Executive summary (leadership)

### Step 6: Resolution & Handoff

Once resolved:
- Document timeline and actions taken
- Create postmortem ticket
- Schedule postmortem meeting
- Update runbooks if needed

## Severity Definitions

| Level | Response Time | Examples |
|-------|---------------|----------|
| **SEV1** | Immediate | Complete outage, data loss, security breach |
| **SEV2** | 15 minutes | Major feature down, significant degradation |
| **SEV3** | 1 hour | Minor feature down, workaround available |
| **SEV4** | Next business day | Cosmetic issues, minor bugs |

## Quick Commands

```
/incident-response start           # Begin new incident
/incident-response status          # Current incident status
/incident-response update          # Post status update
/incident-response resolve         # Mark resolved, start postmortem
```
