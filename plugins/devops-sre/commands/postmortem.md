---
description: Generate blameless postmortem documents with timeline, root cause analysis, and action items
---

# /postmortem

Generate a comprehensive, blameless postmortem document following industry best practices.

## What I Need

Provide incident details:
- Incident ID or description
- Timeline of events (I can pull from Slack/PagerDuty if connected)
- What was the root cause?
- What fixed it?

Or just say: `/postmortem for yesterday's API outage` and I'll gather details.

## Generated Document Structure

```markdown
# Postmortem: [Incident Title]

**Date:** YYYY-MM-DD
**Severity:** SEV-X
**Duration:** X hours Y minutes
**Author:** [Your name]
**Status:** Draft / Final

---

## Executive Summary
[2-3 sentence summary for leadership]

## Impact
- **Users Affected:** X,XXX
- **Revenue Impact:** $X,XXX (estimated)
- **SLA Breach:** Yes/No
- **Customer Notifications:** X sent

## Timeline (All times in UTC)

| Time | Event |
|------|-------|
| HH:MM | First alert fired |
| HH:MM | On-call acknowledged |
| HH:MM | Incident channel created |
| HH:MM | Root cause identified |
| HH:MM | Mitigation deployed |
| HH:MM | Service restored |
| HH:MM | Incident resolved |

## Root Cause Analysis

### What Happened
[Detailed technical explanation]

### Why It Happened (5 Whys)
1. Why did X fail? → Because Y
2. Why did Y happen? → Because Z
3. Why did Z occur? → Because...
4. Why... → Because...
5. Why... → **Root Cause**

### Contributing Factors
- [ ] Recent deployment
- [ ] Configuration change
- [ ] Capacity limits
- [ ] External dependency
- [ ] Missing monitoring
- [ ] Unclear runbook

## What Went Well
- Rapid detection (X minutes)
- Clear communication
- Effective collaboration
- [Other positives]

## What Could Be Improved
- Detection took X minutes (target: Y)
- Runbook was outdated
- Escalation path unclear
- [Other improvements]

## Action Items

| Priority | Action | Owner | Due Date | Ticket |
|----------|--------|-------|----------|--------|
| P0 | [Prevent recurrence] | @name | YYYY-MM-DD | JIRA-XXX |
| P1 | [Improve detection] | @name | YYYY-MM-DD | JIRA-XXX |
| P1 | [Update runbook] | @name | YYYY-MM-DD | JIRA-XXX |
| P2 | [Add monitoring] | @name | YYYY-MM-DD | JIRA-XXX |

## Lessons Learned
[Key takeaways for the team and organization]

---
*This postmortem follows blameless principles. We focus on systems
and processes, not individuals.*
```

## Postmortem Principles

### Blameless Culture
- Focus on **what** failed, not **who** failed
- Assume everyone acted with best intentions
- Systems allowed the failure to happen
- Goal: prevent recurrence, not assign blame

### Good Action Items
- **Specific**: Clear definition of done
- **Measurable**: Can verify completion
- **Assigned**: Single owner (not "team")
- **Time-bound**: Realistic due date
- **Tracked**: Linked to ticket system

## Quick Commands

```
/postmortem generate INC-123       # Generate from incident ID
/postmortem template               # Get blank template
/postmortem review                 # Review existing postmortem
```
