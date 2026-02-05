---
name: deployment-strategies
description: |
  Implement safe deployment strategies including rolling, blue-green, canary, and feature flags.
  Use this skill when planning deployments, reducing deployment risk, or implementing progressive delivery.
  Activate when: deployment strategy, rolling update, blue-green, canary deployment, feature flags,
  progressive delivery, zero downtime deployment, rollback, deployment risk.
---

# Deployment Strategies

**Deploy with confidence using progressive delivery and safe rollback.**

## Strategy Comparison

| Strategy | Risk | Complexity | Rollback Speed | Best For |
|----------|------|------------|----------------|----------|
| **Rolling** | Medium | Low | Medium | Standard updates |
| **Blue-Green** | Low | Medium | Instant | Critical services |
| **Canary** | Low | High | Fast | High-traffic services |
| **Feature Flags** | Lowest | Medium | Instant | A/B testing, gradual rollout |

## Rolling Deployment

### How It Works

```
Initial:  [v1] [v1] [v1] [v1] [v1]

Step 1:   [v2] [v1] [v1] [v1] [v1]  # Replace 1 pod

Step 2:   [v2] [v2] [v1] [v1] [v1]  # Replace 2nd pod

Step 3:   [v2] [v2] [v2] [v1] [v1]  # Continue...

Final:    [v2] [v2] [v2] [v2] [v2]  # All replaced
```

### Kubernetes Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired during update
      maxUnavailable: 0  # Never reduce below desired count
  template:
    spec:
      containers:
      - name: api
        image: api-service:v2
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
```

### Best Practices

```
✓ Always set readinessProbe
✓ Use maxUnavailable: 0 for zero downtime
✓ Monitor error rates during rollout
✓ Have rollback command ready
```

## Blue-Green Deployment

### How It Works

```
Before:
┌─────────────────────────────────────────┐
│  Load Balancer                          │
│       │                                  │
│       ▼                                  │
│  [Blue Environment - v1] ← Active       │
│  [Green Environment - idle]             │
└─────────────────────────────────────────┘

Deploy to Green:
┌─────────────────────────────────────────┐
│  Load Balancer                          │
│       │                                  │
│       ▼                                  │
│  [Blue Environment - v1] ← Active       │
│  [Green Environment - v2] Testing       │
└─────────────────────────────────────────┘

Switch Traffic:
┌─────────────────────────────────────────┐
│  Load Balancer                          │
│       │                                  │
│       ▼                                  │
│  [Blue Environment - v1] Standby        │
│  [Green Environment - v2] ← Active      │
└─────────────────────────────────────────┘
```

### Kubernetes Implementation

```yaml
# Blue deployment (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-blue
  labels:
    version: blue
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: api
        version: blue
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-green
  labels:
    version: green
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: api
        version: green
---
# Service points to active version
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
    version: blue  # Switch to 'green' to cutover
  ports:
  - port: 80
    targetPort: 8080
```

### Switch Traffic

```bash
# Cutover to green
kubectl patch service api -p '{"spec":{"selector":{"version":"green"}}}'

# Rollback to blue
kubectl patch service api -p '{"spec":{"selector":{"version":"blue"}}}'
```

## Canary Deployment

### How It Works

```
Phase 1: 1% canary
┌──────────────────────────────────────────────────────┐
│  [Production v1] ████████████████████████████████ 99% │
│  [Canary v2]     █                                 1% │
└──────────────────────────────────────────────────────┘

Phase 2: 10% canary
┌──────────────────────────────────────────────────────┐
│  [Production v1] ████████████████████████████ 90%    │
│  [Canary v2]     ████                         10%    │
└──────────────────────────────────────────────────────┘

Phase 3: 50% canary
┌──────────────────────────────────────────────────────┐
│  [Production v1] ██████████████             50%      │
│  [Canary v2]     ██████████████             50%      │
└──────────────────────────────────────────────────────┘

Phase 4: Full rollout
┌──────────────────────────────────────────────────────┐
│  [Production v2] ████████████████████████████ 100%   │
└──────────────────────────────────────────────────────┘
```

### Kubernetes with Istio

```yaml
# VirtualService for traffic splitting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api
spec:
  hosts:
  - api
  http:
  - route:
    - destination:
        host: api
        subset: stable
      weight: 90
    - destination:
        host: api
        subset: canary
      weight: 10
---
# DestinationRule defines subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: api
spec:
  host: api
  subsets:
  - name: stable
    labels:
      version: v1
  - name: canary
    labels:
      version: v2
```

### Canary Analysis

```yaml
# Argo Rollouts analysis
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.99
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status!~"5.*",app="api",version="{{args.version}}"}[5m]))
          /
          sum(rate(http_requests_total{app="api",version="{{args.version}}"}[5m]))
```

## Feature Flags

### How It Works

```
Code deployed but inactive:
┌─────────────────────────────────────────────────────┐
│  if (featureFlags.isEnabled("new-checkout")) {      │
│    return newCheckoutFlow();  // 0% of users       │
│  } else {                                           │
│    return oldCheckoutFlow();  // 100% of users     │
│  }                                                  │
└─────────────────────────────────────────────────────┘

Gradually enable:
- 1% → Internal users only
- 5% → Beta users
- 25% → Random subset
- 50% → Half of traffic
- 100% → Fully enabled
```

### Implementation Example

```python
from launchdarkly import LDClient

ld_client = LDClient("sdk-key")

def checkout(user, cart):
    # Check feature flag
    if ld_client.variation("new-checkout-flow", user, False):
        return new_checkout(user, cart)
    else:
        return legacy_checkout(user, cart)
```

### Rollout Strategy

```yaml
# Feature flag configuration
flag: new-checkout-flow
environments:
  production:
    rollout:
      - date: "2026-01-15"
        percentage: 1
        targets: ["internal"]

      - date: "2026-01-17"
        percentage: 10
        targets: ["beta-users"]

      - date: "2026-01-20"
        percentage: 50

      - date: "2026-01-25"
        percentage: 100

    kill_switch: true  # Can instantly disable
```

## Choosing a Strategy

```
┌─────────────────────────────────────────────────────┐
│ What type of change?                                │
├────────────────────────────┬────────────────────────┤
│ Infrastructure/config      │ Database migration     │
│         │                  │         │              │
│         ▼                  │         ▼              │
│   Rolling Update           │  Blue-Green +          │
│                            │  Feature Flag          │
├────────────────────────────┼────────────────────────┤
│ User-facing feature        │ High-risk change       │
│         │                  │         │              │
│         ▼                  │         ▼              │
│   Feature Flag +           │  Canary with           │
│   Canary                   │  Auto-rollback         │
└────────────────────────────┴────────────────────────┘
```

## Deployment Checklist

```markdown
### Pre-Deployment
- [ ] Code reviewed and approved
- [ ] Tests passing
- [ ] Runbook updated
- [ ] Rollback plan documented
- [ ] Monitoring dashboards ready
- [ ] On-call notified

### During Deployment
- [ ] Watch error rates
- [ ] Watch latency
- [ ] Check logs for errors
- [ ] Verify feature works

### Post-Deployment
- [ ] All pods healthy
- [ ] Smoke tests passing
- [ ] No alert spikes
- [ ] Update deployment log
```

## Emergency Rollback

```bash
# Kubernetes rollback
kubectl rollout undo deployment/api-service

# To specific revision
kubectl rollout undo deployment/api-service --to-revision=3

# Check history
kubectl rollout history deployment/api-service

# Blue-green instant switch
kubectl patch service api -p '{"spec":{"selector":{"version":"blue"}}}'

# Feature flag kill switch
curl -X PATCH https://api.launchdarkly.com/flags/new-checkout \
  -H "Authorization: api-key" \
  -d '{"on": false}'
```
