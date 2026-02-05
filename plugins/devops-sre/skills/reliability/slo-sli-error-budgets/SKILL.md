---
name: slo-sli-error-budgets
description: |
  Implement Service Level Objectives (SLOs), Service Level Indicators (SLIs), and error budgets.
  Use this skill when defining reliability targets, measuring service health, or balancing reliability vs velocity.
  Activate when: SLO, SLI, SLA, error budget, reliability targets, service level, uptime target,
  availability target, latency target, nine nines, 99.9%.
---

# SLOs, SLIs, and Error Budgets

**Define and measure reliability in terms that matter to users.**

## Terminology

| Term | Definition | Example |
|------|------------|---------|
| **SLI** | Service Level Indicator - What you measure | 99.2% of requests succeed |
| **SLO** | Service Level Objective - Your target | 99.9% availability |
| **SLA** | Service Level Agreement - Contract with customers | 99.5% with refund clause |
| **Error Budget** | Allowed unreliability (100% - SLO) | 0.1% = 43 min/month downtime |

## Common SLI Types

### Availability

```
availability = successful_requests / total_requests

# Prometheus query
sum(rate(http_requests_total{status!~"5.."}[30d]))
/
sum(rate(http_requests_total[30d]))
```

### Latency

```
latency_sli = requests_under_threshold / total_requests

# Example: 99% of requests under 200ms
sum(rate(http_request_duration_seconds_bucket{le="0.2"}[30d]))
/
sum(rate(http_request_duration_seconds_count[30d]))
```

### Throughput

```
throughput_sli = successful_operations / attempted_operations

# Example: Batch jobs
sum(job_succeeded_total) / sum(job_attempted_total)
```

### Freshness

```
freshness_sli = fresh_data_requests / total_requests

# Example: Data updated within 1 minute
sum(data_age_seconds < 60) / count(data_age_seconds)
```

## Choosing SLO Targets

### The Nines Table

| Availability | Downtime/Year | Downtime/Month | Downtime/Week |
|--------------|---------------|----------------|---------------|
| 99% | 3.65 days | 7.31 hours | 1.68 hours |
| 99.5% | 1.83 days | 3.65 hours | 50.4 min |
| 99.9% | 8.77 hours | 43.8 min | 10.1 min |
| 99.95% | 4.38 hours | 21.9 min | 5.04 min |
| 99.99% | 52.6 min | 4.38 min | 1.01 min |
| 99.999% | 5.26 min | 26.3 sec | 6.05 sec |

### Guidelines for Setting SLOs

```
1. Start with user expectations
   - What do users actually need?
   - What are they getting today?

2. Consider dependencies
   - Your SLO can't exceed your dependencies
   - If database is 99.9%, you can't be 99.99%

3. Start conservative, tighten later
   - Easier to tighten SLO than loosen
   - Build confidence before committing

4. Different SLOs for different tiers
   - Premium customers: 99.99%
   - Free tier: 99.5%
```

## Error Budget

### Calculating Error Budget

```
Monthly Error Budget = (1 - SLO) × Time Period

Example for 99.9% SLO:
- Monthly budget = 0.1% × 43,200 minutes = 43.2 minutes
- Weekly budget = 0.1% × 10,080 minutes = 10.08 minutes
```

### Error Budget Policies

```yaml
# Example Error Budget Policy
error_budget_policy:
  healthy: # >50% budget remaining
    - Continue feature development
    - Normal deployment velocity

  caution: # 25-50% budget remaining
    - Reduce deployment frequency
    - Prioritize reliability work
    - Review recent incidents

  critical: # <25% budget remaining
    - Freeze non-critical deployments
    - All hands on reliability
    - Daily error budget review

  exhausted: # 0% budget remaining
    - Emergency only deployments
    - Postmortem all incidents
    - Leadership escalation
```

### Error Budget Visualization

```
Error Budget: January 2026

SLO: 99.9% availability
Budget: 43.2 minutes

Consumption:
Week 1: ████░░░░░░░░░░░░░░░░  8 min (INC-121)
Week 2: ██░░░░░░░░░░░░░░░░░░  3 min
Week 3: ████████░░░░░░░░░░░░ 15 min (INC-125, INC-126)
Week 4: ████░░░░░░░░░░░░░░░░  7 min (INC-128)
        ────────────────────
Total:  ████████████████░░░░ 33 min consumed

Remaining: 10.2 min (24% of budget)
Status: ⚠️ CAUTION
```

## SLO Implementation

### Step 1: Define SLIs

```yaml
# slo-config.yaml
slis:
  - name: availability
    description: Proportion of successful HTTP requests
    query: |
      sum(rate(http_requests_total{status!~"5.."}[{{window}}]))
      /
      sum(rate(http_requests_total[{{window}}]))

  - name: latency_p99
    description: 99th percentile request latency under 200ms
    query: |
      histogram_quantile(0.99,
        sum(rate(http_request_duration_seconds_bucket[{{window}}])) by (le)
      ) < 0.2
```

### Step 2: Set SLO Targets

```yaml
slos:
  - name: api-availability
    sli: availability
    target: 0.999  # 99.9%
    window: 30d

  - name: api-latency
    sli: latency_p99
    target: 0.99   # 99% of requests under 200ms
    window: 30d
```

### Step 3: Configure Alerts

```yaml
# Alert at different burn rates
alerts:
  - name: SLOBurnRateCritical
    slo: api-availability
    burn_rate: 14.4  # Exhausts monthly budget in 2 days
    window: 1h
    severity: critical

  - name: SLOBurnRateWarning
    slo: api-availability
    burn_rate: 6     # Exhausts monthly budget in 5 days
    window: 6h
    severity: warning
```

## SLO Review Process

### Weekly Review

```markdown
## SLO Weekly Review - Week 4, January 2026

### Summary
| SLO | Target | Actual | Status |
|-----|--------|--------|--------|
| Availability | 99.9% | 99.85% | 🟡 |
| Latency P99 | <200ms | 187ms | 🟢 |
| Error Rate | <0.1% | 0.08% | 🟢 |

### Error Budget
- Consumed this week: 7 minutes
- Remaining this month: 10.2 minutes (24%)
- Projected end-of-month: 5 minutes (12%)

### Incidents
- INC-128: 7 min downtime (database failover)

### Actions
- [ ] Review INC-128 postmortem action items
- [ ] Consider pausing non-critical deploys
```

### Quarterly Review

```markdown
## SLO Quarterly Review - Q1 2026

### SLO Performance
| SLO | Target | Q1 Actual | Trend |
|-----|--------|-----------|-------|
| Availability | 99.9% | 99.92% | ↗️ |
| Latency | <200ms | 178ms | ↗️ |

### Error Budget Utilization
- January: 76% consumed
- February: 45% consumed
- March: 23% consumed
- Average: 48% consumed ✓

### Recommendations
1. Consider tightening availability SLO to 99.95%
2. Add latency SLO for P50 (currently unmeasured)
3. Review alerting thresholds based on budget consumption
```

## Best Practices

### DO
- Base SLOs on user experience, not internal metrics
- Start with fewer SLOs and add as needed
- Review and adjust SLOs quarterly
- Use error budgets to balance reliability and velocity
- Document SLO decisions and rationale

### DON'T
- Set SLOs higher than dependencies allow
- Create SLOs for every metric
- Ignore error budget policies
- Set SLOs without stakeholder buy-in
- Treat SLOs as unchangeable

## SLO Maturity Model

| Level | Characteristics |
|-------|----------------|
| **L1: Ad-hoc** | No formal SLOs, react to incidents |
| **L2: Defined** | SLOs documented, basic monitoring |
| **L3: Measured** | SLIs tracked, dashboards exist |
| **L4: Managed** | Error budgets enforced, policies in place |
| **L5: Optimized** | SLOs drive prioritization, continuous improvement |
