---
name: performance-testing
description: |
  Design and execute performance tests using k6, Artillery, and other tools.
  Use this skill when load testing APIs, stress testing systems, or establishing performance baselines.
  Activate when: performance test, load test, stress test, k6, artillery, benchmark, scalability testing.
---

# Performance Testing

**Ensure your systems handle load with comprehensive performance testing.**

## When to Use

- Establishing performance baselines
- Load testing before releases
- Stress testing to find breaking points
- Capacity planning
- Identifying bottlenecks

## Performance Test Types

| Type | Purpose | Load Pattern |
|------|---------|--------------|
| **Load Test** | Verify expected load | Steady, target users |
| **Stress Test** | Find breaking point | Ramp up until failure |
| **Spike Test** | Handle sudden traffic | Quick spike, then down |
| **Soak Test** | Find memory leaks | Steady load, long duration |
| **Scalability** | Auto-scaling validation | Step increases |

## k6 Load Testing

### Installation

```bash
# macOS
brew install k6

# Docker
docker run -i grafana/k6 run - <script.js
```

### Basic Load Test

```javascript
// tests/load/basic.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 }, // Ramp up to 100 users
    { duration: '5m', target: 100 }, // Stay at 100 users
    { duration: '2m', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests < 500ms
    http_req_failed: ['rate<0.01'],   // Error rate < 1%
  },
};

export default function () {
  const response = http.get('https://api.example.com/users');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1); // Think time between requests
}
```

### Stress Test

```javascript
// tests/load/stress.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 300 },
    { duration: '5m', target: 300 },
    { duration: '2m', target: 400 }, // Breaking point?
    { duration: '5m', target: 400 },
    { duration: '10m', target: 0 },  // Recovery
  ],
};

export default function () {
  const response = http.get('https://api.example.com/search?q=test');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'no errors': (r) => !r.body.includes('error'),
  });
}
```

### Spike Test

```javascript
// tests/load/spike.js
export const options = {
  stages: [
    { duration: '10s', target: 100 },  // Normal load
    { duration: '1m', target: 100 },
    { duration: '10s', target: 1400 }, // Spike to 1400 users
    { duration: '3m', target: 1400 },  // Stay at spike
    { duration: '10s', target: 100 },  // Scale down
    { duration: '3m', target: 100 },   // Recovery
    { duration: '10s', target: 0 },
  ],
};
```

### API Test with Authentication

```javascript
// tests/load/authenticated.js
import http from 'k6/http';
import { check } from 'k6';

export function setup() {
  // Login once, share token across VUs
  const loginRes = http.post('https://api.example.com/auth/login', {
    email: 'loadtest@example.com',
    password: 'testpassword',
  });

  return { token: loginRes.json('token') };
}

export default function (data) {
  const params = {
    headers: {
      Authorization: `Bearer ${data.token}`,
      'Content-Type': 'application/json',
    },
  };

  const response = http.get('https://api.example.com/profile', params);

  check(response, {
    'authenticated': (r) => r.status !== 401,
    'has user data': (r) => r.json('user') !== undefined,
  });
}
```

### Custom Metrics

```javascript
import http from 'k6/http';
import { Counter, Trend, Rate } from 'k6/metrics';

// Custom metrics
const orderSuccessRate = new Rate('order_success');
const orderDuration = new Trend('order_duration');
const ordersCreated = new Counter('orders_created');

export default function () {
  const start = Date.now();

  const response = http.post('https://api.example.com/orders', {
    product_id: 123,
    quantity: 1,
  });

  const duration = Date.now() - start;

  orderDuration.add(duration);
  orderSuccessRate.add(response.status === 201);

  if (response.status === 201) {
    ordersCreated.add(1);
  }
}
```

## Artillery Testing

### Installation

```bash
npm install -g artillery
```

### Basic Test

```yaml
# tests/load/artillery.yml
config:
  target: 'https://api.example.com'
  phases:
    - duration: 60
      arrivalRate: 5
      name: Warm up
    - duration: 120
      arrivalRate: 20
      name: Sustained load
    - duration: 60
      arrivalRate: 50
      name: Peak load

scenarios:
  - name: 'Browse and search'
    flow:
      - get:
          url: '/products'
          capture:
            - json: '$.products[0].id'
              as: 'productId'
      - get:
          url: '/products/{{ productId }}'
      - get:
          url: '/search?q=test'
```

### With Authentication

```yaml
config:
  target: 'https://api.example.com'
  phases:
    - duration: 120
      arrivalRate: 10
  payload:
    path: 'users.csv'
    fields:
      - 'email'
      - 'password'

scenarios:
  - name: 'Authenticated user flow'
    flow:
      - post:
          url: '/auth/login'
          json:
            email: '{{ email }}'
            password: '{{ password }}'
          capture:
            - json: '$.token'
              as: 'token'
      - get:
          url: '/profile'
          headers:
            Authorization: 'Bearer {{ token }}'
```

## Performance Metrics

### Key Metrics to Track

| Metric | Good | Warning | Critical |
|--------|------|---------|----------|
| Response Time (p50) | <200ms | <500ms | >1000ms |
| Response Time (p95) | <500ms | <1000ms | >2000ms |
| Response Time (p99) | <1000ms | <2000ms | >5000ms |
| Error Rate | <0.1% | <1% | >5% |
| Throughput | >target | >80% target | <80% target |

### Interpreting Results

```markdown
## Performance Test Report

### Summary
- **Test Duration:** 10 minutes
- **Virtual Users:** 100 (peak)
- **Total Requests:** 50,000
- **Successful:** 49,850 (99.7%)
- **Failed:** 150 (0.3%)

### Response Times
| Percentile | Time |
|------------|------|
| p50 | 145ms |
| p90 | 320ms |
| p95 | 450ms |
| p99 | 890ms |

### Throughput
- **Avg:** 83 req/s
- **Peak:** 120 req/s

### Errors
| Code | Count | % |
|------|-------|---|
| 200 | 49,850 | 99.7% |
| 500 | 100 | 0.2% |
| 503 | 50 | 0.1% |

### Recommendations
1. Investigate 500 errors - likely database timeout
2. p99 approaching threshold - consider caching
3. Throughput meets requirements
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  schedule:
    - cron: '0 2 * * *'  # Nightly
  workflow_dispatch:

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install k6
        run: |
          curl https://github.com/grafana/k6/releases/download/v0.47.0/k6-v0.47.0-linux-amd64.tar.gz -L | tar xvz
          sudo mv k6-v0.47.0-linux-amd64/k6 /usr/local/bin/

      - name: Run load test
        run: k6 run tests/load/basic.js --out json=results.json

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: results.json

      - name: Check thresholds
        run: |
          if grep -q '"thresholds":{".*":{"ok":false' results.json; then
            echo "Performance thresholds failed!"
            exit 1
          fi
```

## Best Practices

1. **Baseline first** - Establish baseline before optimizing
2. **Realistic data** - Use production-like data volumes
3. **Think time** - Include realistic pauses between requests
4. **Ramp gradually** - Don't spike immediately to target load
5. **Monitor everything** - Track server metrics during tests
6. **Isolate environment** - Don't test on production
7. **Repeat tests** - Run multiple times for consistency
8. **Document results** - Keep historical records
