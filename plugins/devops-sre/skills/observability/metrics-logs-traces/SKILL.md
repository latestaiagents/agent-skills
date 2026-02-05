---
name: metrics-logs-traces
description: |
  Implement comprehensive observability with metrics, logs, and distributed traces.
  Use this skill when setting up monitoring, debugging production issues, or implementing observability.
  Activate when: metrics, logs, traces, observability, monitoring, Datadog, Prometheus, Grafana,
  OpenTelemetry, distributed tracing, logging, APM, what's happening in production.
---

# Observability: Metrics, Logs, and Traces

**The three pillars of observability for understanding system behavior.**

## The Three Pillars

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PILLARS                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  METRICS              LOGS                TRACES             │
│  ────────            ─────               ───────             │
│  What's happening    Why it's happening  How requests flow   │
│                                                              │
│  • Request rate      • Error messages    • Request path      │
│  • Error rate        • Stack traces      • Service deps      │
│  • Latency           • Debug info        • Timing breakdown  │
│  • Saturation        • Audit events      • Correlation       │
│                                                              │
│  Aggregated          Individual          Request-scoped      │
│  Time-series         Events              Distributed         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Metrics: The RED Method

For services, track **R**ate, **E**rrors, and **D**uration:

```python
# Example: Prometheus metrics in Python
from prometheus_client import Counter, Histogram, Gauge

# Rate: Request throughput
request_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# Errors: Error rate
error_total = Counter(
    'http_errors_total',
    'Total HTTP errors',
    ['method', 'endpoint', 'error_type']
)

# Duration: Latency distribution
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

# Usage in handler
@request_duration.labels(method='GET', endpoint='/api/users').time()
def get_users():
    request_total.labels(method='GET', endpoint='/api/users', status='200').inc()
    # ... handler logic
```

## Metrics: The USE Method

For resources, track **U**tilization, **S**aturation, and **E**rrors:

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| **CPU** | % busy | Run queue length | - |
| **Memory** | % used | Swap usage, OOM | OOM kills |
| **Disk** | % capacity | I/O wait | Read/write errors |
| **Network** | Bandwidth % | Queue depth | Packet drops |
| **Connections** | Pool usage | Queue length | Timeouts |

## Structured Logging

### Standard Log Format

```json
{
  "timestamp": "2026-01-15T10:23:45.123Z",
  "level": "ERROR",
  "service": "api-gateway",
  "trace_id": "abc123def456",
  "span_id": "789xyz",
  "user_id": "user-456",
  "request_id": "req-789",
  "message": "Database connection failed",
  "error": {
    "type": "ConnectionError",
    "message": "Connection pool exhausted",
    "stack": "..."
  },
  "context": {
    "endpoint": "/api/users",
    "method": "GET",
    "duration_ms": 5023
  }
}
```

### Log Levels

| Level | Use For | Example |
|-------|---------|---------|
| **ERROR** | Failures requiring attention | DB connection failed |
| **WARN** | Potential issues | Retry attempt 2/3 |
| **INFO** | Business events | User signed up |
| **DEBUG** | Diagnostic detail | Query params: {...} |

### Python Structured Logging

```python
import structlog

logger = structlog.get_logger()

def process_order(order_id: str, user_id: str):
    log = logger.bind(
        order_id=order_id,
        user_id=user_id,
        trace_id=get_current_trace_id()
    )

    log.info("processing_order_started")

    try:
        result = process(order_id)
        log.info("processing_order_completed", result=result)
    except PaymentError as e:
        log.error("payment_failed",
            error_type=type(e).__name__,
            error_message=str(e)
        )
        raise
```

## Distributed Tracing

### OpenTelemetry Setup

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Setup
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

# Usage
def process_request(request):
    with tracer.start_as_current_span("process_request") as span:
        span.set_attribute("user.id", request.user_id)
        span.set_attribute("request.path", request.path)

        # Child span for DB call
        with tracer.start_as_current_span("db_query") as db_span:
            db_span.set_attribute("db.statement", "SELECT * FROM users")
            result = db.query(...)

        # Child span for external API
        with tracer.start_as_current_span("external_api") as api_span:
            api_span.set_attribute("http.url", "https://api.external.com")
            response = http_client.get(...)

        return result
```

### Trace Visualization

```
Trace: abc123def456 (Total: 245ms)
│
├─[api-gateway] handle_request ──────────────────────── 245ms
│  │
│  ├─[api-gateway] auth_middleware ─────────── 12ms
│  │  └─[auth-service] validate_token ─────── 10ms
│  │
│  ├─[api-gateway] rate_limit_check ─────────── 2ms
│  │
│  └─[api-gateway] route_request ─────────────── 230ms
│     │
│     └─[user-service] get_user ──────────────── 225ms
│        │
│        ├─[user-service] cache_lookup ────────── 3ms (MISS)
│        │
│        └─[user-service] db_query ──────────── 220ms ⚠️ SLOW
│           └─[postgres] SELECT * FROM users ─── 218ms
```

## Correlation: Tying It Together

### Request Flow with Correlation

```
1. Request arrives → Generate trace_id
2. Pass trace_id in headers to downstream services
3. Include trace_id in all logs
4. Include trace_id in metrics labels (careful with cardinality)
5. When debugging: Search logs by trace_id, view trace in APM
```

### Example: Finding a Slow Request

```
1. Dashboard shows P99 latency spike
2. Click through to see slow requests
3. Pick one trace_id: abc123def456
4. View trace → See db_query took 5s
5. Search logs: trace_id:abc123def456
6. Find log: "Slow query: SELECT * FROM users WHERE..."
7. Root cause: Missing index on users table
```

## Alerting Best Practices

### Alert on Symptoms, Not Causes

```yaml
# Good: Alert on user impact
- alert: HighErrorRate
  expr: sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m])) > 0.01
  for: 5m

# Bad: Alert on implementation detail
- alert: DatabaseConnectionHigh
  expr: db_connections > 80
  # User may not be impacted yet
```

### Alert Fatigue Prevention

| Severity | Response | Example |
|----------|----------|---------|
| **Page** | Wake up | >1% error rate for 5min |
| **Ticket** | Next day | Disk 80% full |
| **Log** | Review weekly | Deprecated API usage |

## Quick Reference

### What to Use When

| Scenario | Primary Tool |
|----------|--------------|
| "Is the service healthy?" | Metrics (dashboards) |
| "Why did this request fail?" | Logs + Traces |
| "Where is latency coming from?" | Traces |
| "What changed before the incident?" | Metrics + Deploy markers |
| "How many users affected?" | Metrics + Logs aggregation |
