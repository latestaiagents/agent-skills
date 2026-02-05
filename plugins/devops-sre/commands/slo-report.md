---
description: Generate SLO/SLA status reports with error budget analysis and recommendations
---

# /slo-report

Generate SLO status reports with error budget consumption, trends, and recommendations.

## What I Need

Tell me:
- Time period (last week/month/quarter)
- Specific service or all services
- Report format (summary/detailed)

## Sample Report

```markdown
# SLO Status Report

**Period:** January 2026
**Team:** Platform Engineering
**Generated:** 2026-02-01

---

## Executive Summary

| Status | Services |
|--------|----------|
| 🟢 Healthy | 8 |
| 🟡 At Risk | 2 |
| 🔴 Breached | 0 |

**Overall Error Budget:** 67% remaining (was 78% last month)

---

## Service-by-Service Status

### api-gateway
| SLO | Target | Actual | Status | Budget |
|-----|--------|--------|--------|--------|
| Availability | 99.9% | 99.95% | 🟢 | 85% |
| Latency P99 | <200ms | 156ms | 🟢 | 100% |
| Error Rate | <0.1% | 0.04% | 🟢 | 92% |

**Trend:** ↗️ Improving (+5% budget vs last month)

---

### payments-service
| SLO | Target | Actual | Status | Budget |
|-----|--------|--------|--------|--------|
| Availability | 99.99% | 99.97% | 🟡 | 23% |
| Latency P99 | <500ms | 467ms | 🟢 | 78% |
| Error Rate | <0.05% | 0.03% | 🟢 | 85% |

**Trend:** ↘️ Declining (-15% budget vs last month)

**⚠️ At Risk:** Availability budget consumed faster than expected
- 3 incidents this month (INC-121, INC-125, INC-128)
- Projected budget exhaustion: Feb 15

**Recommendations:**
1. Prioritize reliability work over new features
2. Review INC-128 postmortem action items
3. Consider adding redundancy to payment processor

---

### search-service
| SLO | Target | Actual | Status | Budget |
|-----|--------|--------|--------|--------|
| Availability | 99.5% | 99.82% | 🟢 | 100% |
| Latency P99 | <1000ms | 678ms | 🟢 | 100% |
| Relevance | >85% | 87% | 🟢 | 100% |

**Trend:** → Stable

---

## Error Budget Consumption

```
January Error Budget Burn Rate

api-gateway    ████░░░░░░░░░░░░░░░░ 15% consumed
payments       ████████████████░░░░ 77% consumed  ⚠️
user-service   ██████░░░░░░░░░░░░░░ 28% consumed
search         ░░░░░░░░░░░░░░░░░░░░  0% consumed
notifications  ████████░░░░░░░░░░░░ 35% consumed
```

## Incidents Impact

| Incident | Duration | Services | Budget Impact |
|----------|----------|----------|---------------|
| INC-121 | 23 min | payments | -12% |
| INC-125 | 8 min | payments, api | -5% |
| INC-128 | 45 min | payments | -18% |
| INC-130 | 5 min | notifications | -3% |

## Recommendations

### Immediate (This Week)
1. 🔴 **payments-service**: Implement circuit breaker for payment processor
2. 🟡 **api-gateway**: Review rate limiting configuration

### This Month
3. Add synthetic monitoring for critical paths
4. Implement SLO alerting at 50% budget consumption
5. Schedule quarterly SLO review meeting

### This Quarter
6. Evaluate SLO targets - some may be too aggressive
7. Implement error budget-based deployment gates

---

*Report generated from Datadog SLO data*
*Next review: 2026-02-01*
```

## Quick Commands

```
/slo-report summary               # Quick summary all services
/slo-report [service]             # Detailed report for service
/slo-report monthly               # Full monthly report
/slo-report budget payments       # Error budget deep-dive
```
