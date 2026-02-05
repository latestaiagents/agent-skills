---
name: alerting-strategies
description: |
  Design effective alerting strategies that catch real issues without causing alert fatigue.
  Use this skill when setting up alerts, reducing noise, or improving on-call experience.
  Activate when: alerting, alerts, pagerduty, on-call, alert fatigue, too many alerts,
  missed alerts, monitoring thresholds, alert tuning.
---

# Alerting Strategies

**Get paged for real problems, not noise.**

## Alerting Philosophy

> "Every alert should be actionable, and every action should have a runbook."

### The Goal
- Page for **symptoms** (user impact), not **causes** (internal metrics)
- Every page should require **human judgment**
- False positives erode trust; false negatives cause outages

## Alert Severity Levels

| Level | Response | Time to Ack | Example |
|-------|----------|-------------|---------|
| **P1/Critical** | Page immediately | 5 min | Service down, data loss |
| **P2/High** | Page during hours | 30 min | Degraded performance |
| **P3/Medium** | Ticket | Next day | Non-critical feature broken |
| **P4/Low** | Review weekly | N/A | Cleanup tasks, warnings |

## Alert Types

### 1. Symptom-Based (Recommended)

Alert on what users experience:

```yaml
# Good: Users are experiencing errors
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
    /
    sum(rate(http_requests_total[5m]))
    > 0.01
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error rate above 1%"
    runbook: "https://wiki/runbooks/high-error-rate"
```

### 2. Cause-Based (Use Sparingly)

Alert on infrastructure issues that will cause symptoms:

```yaml
# Acceptable: Will cause problems soon
- alert: DiskSpaceLow
  expr: node_filesystem_avail_bytes / node_filesystem_size_bytes < 0.1
  for: 15m
  labels:
    severity: warning
  annotations:
    summary: "Disk space below 10%"
```

### 3. SLO-Based (Best Practice)

Alert on error budget consumption:

```yaml
# Excellent: Based on SLO burn rate
- alert: SLOBurnRateHigh
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h]))
      /
      sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)  # 14.4x burn rate
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Burning error budget 14x faster than sustainable"
```

## Multi-Window, Multi-Burn-Rate Alerts

Google SRE's recommended approach:

```yaml
# Fast burn (page immediately)
- alert: SLOBurnRateFast
  expr: |
    (
      job:slo_errors_per_request:ratio_rate1h > (14.4 * 0.001)
      and
      job:slo_errors_per_request:ratio_rate5m > (14.4 * 0.001)
    )
  for: 2m
  labels:
    severity: critical

# Slow burn (page during business hours)
- alert: SLOBurnRateSlow
  expr: |
    (
      job:slo_errors_per_request:ratio_rate6h > (1 * 0.001)
      and
      job:slo_errors_per_request:ratio_rate30m > (1 * 0.001)
    )
  for: 15m
  labels:
    severity: warning
```

## Alert Design Patterns

### Pattern 1: Percentage-Based Thresholds

```yaml
# Alert when error rate exceeds normal baseline
- alert: ErrorRateAnomaly
  expr: |
    (
      sum(rate(http_errors_total[5m]))
      /
      sum(rate(http_requests_total[5m]))
    )
    >
    (
      sum(rate(http_errors_total[1d] offset 1d))
      /
      sum(rate(http_requests_total[1d] offset 1d))
    ) * 2
```

### Pattern 2: Absence Detection

```yaml
# Alert when service stops reporting
- alert: ServiceDown
  expr: absent(up{job="api-service"} == 1)
  for: 5m
```

### Pattern 3: Derivative-Based

```yaml
# Alert on rapid change
- alert: LatencySpike
  expr: deriv(http_request_duration_seconds_sum[5m]) > 0.1
  for: 2m
```

## Alert Fatigue Prevention

### Checklist Before Creating Alert

```
□ Is this actionable? What should responder do?
□ Does a runbook exist?
□ Is this a symptom or a cause?
□ What's the false positive rate likely to be?
□ Can this be a ticket instead of a page?
□ Is the threshold based on data, not gut feel?
□ Does it have appropriate for/pending duration?
```

### Noisy Alert Remediation

| Problem | Solution |
|---------|----------|
| Too many pages | Increase threshold or duration |
| Flapping alerts | Add hysteresis (different up/down thresholds) |
| Duplicate alerts | Use alert grouping/inhibition |
| Low-signal alerts | Convert to ticket or remove |
| Night pages for non-urgent | Route to next business day |

### Alert Hygiene Process

```
Weekly:
- Review all alerts that fired
- Tag: actionable / noise / duplicate
- Fix or remove noisy alerts

Monthly:
- Review alert coverage vs incidents
- Identify incidents with no alerts (gaps)
- Identify alerts that never fired (remove?)

Quarterly:
- Full alert audit
- Update thresholds based on SLO performance
- Review on-call burden metrics
```

## Alert Routing

### Example PagerDuty Integration

```yaml
# Route based on service and severity
receivers:
  - name: 'platform-critical'
    pagerduty_configs:
      - service_key: '<platform-team-key>'
        severity: critical

  - name: 'platform-warning'
    pagerduty_configs:
      - service_key: '<platform-team-key>'
        severity: warning

  - name: 'tickets'
    webhook_configs:
      - url: 'https://jira.company.com/webhook'

route:
  group_by: ['alertname', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'platform-warning'
  routes:
    - match:
        severity: critical
      receiver: 'platform-critical'
    - match:
        severity: low
      receiver: 'tickets'
```

## Alert Documentation Template

Every alert should have:

```markdown
# Alert: HighErrorRate

## What This Means
Error rate has exceeded 1% for the past 5 minutes.
Users are experiencing failures.

## Impact
- Users see error pages
- API consumers get 500 responses
- Potential revenue impact

## First Response
1. Check deployment timeline - recent deploy?
2. Check dependency status (database, external APIs)
3. Look at error logs for specific error messages

## Runbook
[Link to detailed runbook]

## Escalation
If unresolved after 15 minutes, page @platform-lead

## Historical Context
- Normal error rate: 0.01-0.05%
- Common causes: bad deploys, DB issues, traffic spikes
```

## Metrics for Alerting Health

Track these to improve your alerting:

| Metric | Target | Why |
|--------|--------|-----|
| **MTTA** (Mean Time to Acknowledge) | <5 min | Are pages noticed? |
| **Pages per week per engineer** | <10 | Alert fatigue risk |
| **% actionable pages** | >80% | Signal vs noise |
| **Incidents with no alerts** | <10% | Coverage gaps |
| **False positive rate** | <20% | Trust in alerts |
