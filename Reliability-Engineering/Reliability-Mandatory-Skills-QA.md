# Reliability Mandatory Skills - Deep Tricky Interview Questions & Answers

## CloudWatch, OpenTelemetry, Prometheus/Grafana

---

### Q1: Your CloudWatch bill jumped from $5K to $45K in one month. No new services were deployed. How do you diagnose and fix this without losing observability?

**Answer:**

**CloudWatch Cost Breakdown (where the money goes):**

```
CloudWatch Pricing Traps:
├── Custom Metrics: $0.30/metric/month (first 10K)
│   └── High-cardinality labels explode metric count
├── Log Ingestion: $0.50/GB
│   └── Debug logging accidentally left on
├── Log Storage: $0.03/GB/month
│   └── No retention policy = infinite storage
├── Metric Queries (Dashboards): $0.01/1000 metrics requested
│   └── Auto-refresh dashboards polling every 10s
├── Embedded Metrics Format: Same as custom metrics
│   └── EMF with high-cardinality dimensions
└── Contributor Insights: $0.02/rule/evaluation
```

**Diagnosis:**

```bash
# Step 1: Use Cost Explorer to identify the cost category
aws ce get-cost-and-usage \
  --time-period Start=2024-05-01,End=2024-06-01 \
  --granularity DAILY \
  --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["AmazonCloudWatch"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# Common culprits in output:
# CW:MetricMonitorUsage     → Custom metrics explosion
# CW:DataProcessing-Bytes   → Log ingestion spike
# CW:DataStorage-Bytes      → Log retention too long
# CW:GMD-Metrics            → GetMetricData API calls (dashboards)
```

**Root Cause 1: Custom Metric Explosion (Most Common)**

```python
# BAD: Creating a metric per request_id (infinite cardinality!)
cloudwatch.put_metric_data(
    Namespace='App',
    MetricData=[{
        'MetricName': 'RequestLatency',
        'Value': 145,
        'Dimensions': [
            {'Name': 'RequestId', 'Value': 'uuid-xxx'},  # UNIQUE PER REQUEST!
            {'Name': 'Endpoint', 'Value': '/api/v1/users'},
            {'Name': 'StatusCode', 'Value': '200'},
            {'Name': 'CustomerId', 'Value': 'cust-123'}  # High cardinality!
        ]
    }]
)
# Each unique dimension combination = separate metric
# 1M requests/day × 4 dimensions = millions of metrics = $$$

# GOOD: Use bounded dimensions only
cloudwatch.put_metric_data(
    Namespace='App',
    MetricData=[{
        'MetricName': 'RequestLatency',
        'Value': 145,
        'Unit': 'Milliseconds',
        'Dimensions': [
            {'Name': 'Endpoint', 'Value': '/api/v1/users'},  # Bounded (~50 values)
            {'Name': 'StatusCode', 'Value': '2xx'},           # Bounded (5 values)
        ]
    }]
)
# 50 endpoints × 5 status codes = 250 metrics = ~$75/month
```

**Root Cause 2: Log Ingestion Spike**

```bash
# Find which log groups are ingesting the most
aws cloudwatch get-metric-data \
  --metric-data-queries '[{
    "Id": "incoming",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/Logs",
        "MetricName": "IncomingBytes",
        "Dimensions": [{"Name":"LogGroupName","Value":"/ecs/payment-api"}]
      },
      "Period": 86400,
      "Stat": "Sum"
    }
  }]' \
  --start-time 2024-06-01T00:00:00Z \
  --end-time 2024-06-15T00:00:00Z
```

```python
# Fix: Set log levels properly and add retention
import boto3
logs = boto3.client('logs')

# Set retention on ALL log groups (default is NEVER expire!)
log_groups = logs.describe_log_groups()['logGroups']
for lg in log_groups:
    if 'retentionInDays' not in lg:
        logs.put_retention_policy(
            logGroupName=lg['logGroupName'],
            retentionInDays=30  # Or 7 for dev, 90 for prod
        )
        print(f"Set 30-day retention on {lg['logGroupName']}")
```

**Root Cause 3: Dashboard Auto-Refresh Storm**

```
Problem: 20 dashboards × 50 metrics each × refreshing every 10 seconds
= 20 × 50 × 8640 refreshes/day = 8.6M GetMetricData calls/day
= $86/day just for dashboards!

Fix: Set dashboard refresh to 5 minutes (not 10 seconds)
Fix: Use CloudWatch Metrics Insights (aggregated queries) instead of individual metrics
```

**Cost Optimization Strategy:**

```hcl
# 1. Metric filters instead of custom metrics (FREE!)
resource "aws_cloudwatch_log_metric_filter" "errors" {
  name           = "error-count"
  pattern        = "[timestamp, level=ERROR, ...]"
  log_group_name = "/ecs/payment-api"
  
  metric_transformation {
    name      = "ErrorCount"
    namespace = "App/PaymentAPI"
    value     = "1"
    # This is FREE — extracted from logs you're already paying for
  }
}

# 2. Use EMF with bounded dimensions
# 3. Set log retention on ALL groups
# 4. Use Contributor Insights sparingly
# 5. CloudWatch cross-account observability (centralize, don't duplicate)
```

---

### Q2: You're implementing OpenTelemetry (ADOT) across 200 microservices on EKS. After rollout, services report 15% latency increase and 30% more memory usage. How do you reduce the observability overhead to < 2%?

**Answer:**

**Why OpenTelemetry Adds Overhead:**

```
Overhead Sources:
├── Trace Context Propagation: ~0.1ms per hop (negligible)
├── Span Creation/Export: ~0.5ms per span (adds up with nested spans)
├── Log Correlation: ~0.1ms per log line
├── Metric Collection: ~1% CPU for scraping
├── OTLP Export (gRPC): Network I/O + serialization
├── Batching Buffer: Memory for buffered telemetry
└── SDK Auto-Instrumentation: Hooks into EVERY library call
    └── THIS is usually the 15% latency culprit
```

**Solution: Sampling + Tail-Based Collection + Tuning**

**Fix 1: Head-Based Sampling (Reduce volume by 90%)**

```yaml
# ADOT Collector Configuration
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Probabilistic sampling: Only trace 10% of requests
  probabilistic_sampler:
    sampling_percentage: 10  # 90% reduction in trace volume
  
  # BUT: Always sample errors and slow requests (tail-based)
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    policies:
      # Always keep errors
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]
      # Always keep slow requests (> 2 seconds)
      - name: latency-policy
        type: latency
        latency:
          threshold_ms: 2000
      # Sample 10% of everything else
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
  
  # Batch to reduce network calls (critical for performance!)
  batch:
    timeout: 5s
    send_batch_size: 1024
    send_batch_max_size: 2048
  
  # Memory limiter (prevent OOM)
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

exporters:
  otlp/xray:
    endpoint: "https://xray.us-east-1.amazonaws.com"
    compression: zstd
  
  prometheusremotewrite:
    endpoint: "https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-xxx/api/v1/remote_write"
    auth:
      authenticator: sigv4auth
    # Reduce cardinality before export
    resource_to_telemetry_conversion:
      enabled: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, tail_sampling, batch]
      exporters: [otlp/xray]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheusremotewrite]
```

**Fix 2: SDK Configuration (Reduce Application Overhead)**

```python
# Python OpenTelemetry SDK optimization
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

# Optimized tracer provider
provider = TracerProvider(
    resource=Resource.create({
        "service.name": "payment-api",
        "deployment.environment": "production"
    }),
    # Use parent-based sampler (inherit decision from parent span)
    sampler=trace.sampling.ParentBasedTraceIdRatio(0.1)  # 10% sampling
)

# Batch processor: accumulate spans, export in batches
processor = BatchSpanProcessor(
    OTLPSpanExporter(endpoint="http://adot-collector:4317", insecure=True),
    max_queue_size=2048,         # Buffer size
    max_export_batch_size=512,   # Export 512 at a time
    schedule_delay_millis=5000,  # Export every 5 seconds
    export_timeout_millis=10000  # Timeout for export call
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
```

**Fix 3: Selective Auto-Instrumentation (Don't Instrument Everything)**

```python
# DON'T: Instrument every single library
# opentelemetry-instrument (auto-instruments EVERYTHING)
# This adds hooks to: HTTP, DB, Redis, gRPC, filesystem, DNS...
# Result: 100+ spans per request → 15% overhead

# DO: Only instrument what you need
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Only instrument entry point + outgoing calls + DB
FastAPIInstrumentor.instrument_app(app)
HTTPXClientInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument(engine=db_engine)

# Skip: filesystem, DNS, internal function calls
# These add noise without actionable insights
```

**Fix 4: Collector Architecture (Reduce Per-Pod Overhead)**

```
BAD Architecture (high overhead):
Each Pod → Sidecar Collector → X-Ray
(200 pods × sidecar = 200 collectors, each using 256MB RAM)

GOOD Architecture (low overhead):
Each Pod → (lightweight SDK export) → DaemonSet Collector → X-Ray
(3 nodes × 1 DaemonSet = 3 collectors, shared)

BEST Architecture (minimal overhead):
Each Pod → (UDP/gRPC to local) → DaemonSet Agent → Central Collector → X-Ray
(Agent: 64MB RAM, minimal processing)
```

```yaml
# DaemonSet collector (one per node, shared by all pods)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: adot-collector
spec:
  template:
    spec:
      containers:
        - name: collector
          image: public.ecr.aws/aws-observability/aws-otel-collector:latest
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          ports:
            - containerPort: 4317  # gRPC OTLP
              hostPort: 4317       # Accessible from all pods on node
```

**Performance Target Achieved:**
```
Before: 15% latency increase, 30% more memory
After:
- Sampling (10%): Reduces export volume by 90%
- Selective instrumentation: Reduces span count by 70%
- DaemonSet vs sidecar: Reduces memory by 80%
- Batch export: Reduces network calls by 95%
Result: < 2% latency impact, < 5% memory increase
```

---

### Q3: Your Prometheus instance on Amazon Managed Prometheus (AMP) has 5 million active time series. Query performance is degrading and costs are increasing. How do you reduce cardinality while maintaining useful observability?

**Answer:**

**Understanding the Problem:**

```
Cardinality = unique combinations of metric name + label values
Example:
  http_requests_total{method="GET", path="/api/users/123", status="200", pod="pod-abc123"}
  
  If you have:
  - 5 methods × 10,000 unique paths × 5 status codes × 200 pods
  = 50,000,000 time series! (CATASTROPHIC)
  
  AMP pricing: $0.03/1000 samples ingested/month
  5M series × 15s scrape interval × 30 days = $$$
```

**Diagnosis: Find High-Cardinality Offenders**

```promql
-- PromQL: Find metrics with highest cardinality
topk(20, count by (__name__)({__name__=~".+"}))

-- Find labels causing explosion
count by (path)(http_requests_total)
-- If this returns 10,000+ → "path" label has unbounded cardinality

-- Check total series count
prometheus_tsdb_head_series
```

**Solution 1: Relabeling (Drop/Aggregate Before Storage)**

```yaml
# prometheus.yml: relabel_configs (processing BEFORE ingestion)
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    
    metric_relabel_configs:
      # DROP high-cardinality labels
      - source_labels: [__name__]
        regex: 'http_request_duration_seconds_bucket'
        action: drop
        # Drop histogram buckets (use summary instead)
      
      # REPLACE: Normalize URL paths (remove IDs)
      - source_labels: [path]
        regex: '/api/users/[0-9a-f-]+'
        target_label: path
        replacement: '/api/users/{id}'
      
      - source_labels: [path]
        regex: '/api/orders/[0-9]+'
        target_label: path
        replacement: '/api/orders/{id}'
      
      # DROP: Remove pod-specific labels (aggregate at service level)
      - action: labeldrop
        regex: '(pod|pod_template_hash|controller_revision_hash)'
      
      # KEEP: Only metrics we actually use in dashboards/alerts
      - source_labels: [__name__]
        regex: '(http_requests_total|http_request_duration_seconds_sum|http_request_duration_seconds_count|container_cpu_usage_seconds_total|container_memory_working_set_bytes|kube_pod_status_phase)'
        action: keep
        # DROP everything else (typically 80% of metrics are unused!)
```

**Solution 2: Recording Rules (Pre-Aggregate)**

```yaml
# recording_rules.yml: Pre-compute expensive queries
groups:
  - name: service_metrics_5m
    interval: 30s
    rules:
      # Instead of querying raw per-pod metrics in dashboards:
      # Pre-aggregate to service level
      - record: service:http_requests:rate5m
        expr: sum by (service, method, status_code) (rate(http_requests_total[5m]))
      
      - record: service:http_request_duration:p99_5m
        expr: histogram_quantile(0.99, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))
      
      - record: service:http_request_duration:p50_5m
        expr: histogram_quantile(0.50, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))
      
      # Error rate per service (used in SLO dashboard)
      - record: service:http_error_rate:ratio_5m
        expr: |
          sum by (service) (rate(http_requests_total{status_code=~"5.."}[5m]))
          /
          sum by (service) (rate(http_requests_total[5m]))

  - name: node_metrics_5m
    interval: 60s
    rules:
      # CPU utilization per node (instead of per-container)
      - record: node:cpu_utilization:avg5m
        expr: 1 - avg by (node) (rate(node_cpu_seconds_total{mode="idle"}[5m]))
```

**Solution 3: Application-Level Instrumentation Fix**

```python
# BAD: Unbounded labels in application metrics
from prometheus_client import Histogram

request_duration = Histogram(
    'http_request_duration_seconds',
    'Request duration',
    ['method', 'path', 'status', 'user_id']  # user_id = DISASTER
)

@app.middleware("http")
async def metrics_middleware(request, call_next):
    response = await call_next(request)
    request_duration.labels(
        method=request.method,
        path=request.url.path,          # /api/users/uuid-xxx (unbounded!)
        status=response.status_code,
        user_id=request.user.id         # Millions of unique values!
    ).observe(duration)

# GOOD: Bounded labels with path normalization
request_duration = Histogram(
    'http_request_duration_seconds',
    'Request duration',
    ['method', 'endpoint', 'status_class'],  # All bounded
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

def normalize_path(path: str) -> str:
    """Convert /api/users/abc-123 to /api/users/{id}"""
    import re
    path = re.sub(r'/[0-9a-f-]{36}', '/{id}', path)  # UUIDs
    path = re.sub(r'/\d+', '/{id}', path)             # Numeric IDs
    return path

@app.middleware("http")
async def metrics_middleware(request, call_next):
    response = await call_next(request)
    request_duration.labels(
        method=request.method,
        endpoint=normalize_path(request.url.path),  # Bounded!
        status_class=f"{response.status_code // 100}xx"  # 5 values only
    ).observe(duration)
```

**Solution 4: AMP Workspace Optimization**

```hcl
# Use multiple workspaces for cost isolation
resource "aws_prometheus_workspace" "production" {
  alias = "production-metrics"
  
  # Only ingest metrics we actually use
  # Configure ADOT collector to filter before sending
}

# Remote write with filtering
# In ADOT collector config:
exporters:
  prometheusremotewrite:
    endpoint: "https://aps-workspaces.../api/v1/remote_write"
    # Write-ahead log for reliability
    wal:
      directory: /tmp/wal
      buffer_size: 300
      truncate_frequency: 15m
    # Only send metrics matching these patterns:
    resource_to_telemetry_conversion:
      enabled: true
    external_labels:
      cluster: production
      region: us-east-1
```

**Cardinality Reduction Results:**

```
Before: 5,000,000 active time series
After optimizations:
- Drop unused metrics (keep list): -60% → 2,000,000
- Path normalization: -50% → 1,000,000
- Remove pod-level labels: -40% → 600,000
- Recording rules (pre-aggregate): queries 10x faster

Final: ~600K active time series (88% reduction)
Cost: Reduced from ~$15K/month to ~$2K/month
```

---

### Q4: Your team wants to implement the "Four Golden Signals" (Latency, Traffic, Errors, Saturation) for 50 microservices. How do you design this with OpenTelemetry instrumentation that works across Python, Java, and Node.js services?

**Answer:**

**Architecture: Unified Observability with OTEL SDK**

```
┌─────────────────────────────────────────────────────────────────┐
│  Four Golden Signals Implementation                              │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Application (any language)                                  │ │
│  │                                                             │ │
│  │ OTEL SDK (auto + manual instrumentation)                    │ │
│  │ ├── Latency: request duration histogram                     │ │
│  │ ├── Traffic: request count by endpoint                      │ │
│  │ ├── Errors: error count + error rate                        │ │
│  │ └── Saturation: CPU, memory, queue depth, connections       │ │
│  └─────────────────────────┬──────────────────────────────────┘ │
│                             │ OTLP (gRPC)                        │
│  ┌──────────────────────────▼─────────────────────────────────┐ │
│  │ ADOT Collector (DaemonSet)                                  │ │
│  │ ├── Metrics → AMP (Prometheus)                              │ │
│  │ ├── Traces → X-Ray                                          │ │
│  │ └── Logs → CloudWatch Logs                                  │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                             │                                     │
│  ┌──────────────────────────▼─────────────────────────────────┐ │
│  │ Grafana (AMG) - Unified Dashboards                          │ │
│  │ ├── Service Overview (4 golden signals per service)          │ │
│  │ ├── SLO Dashboard (error budgets)                           │ │
│  │ └── Alerting (Prometheus AlertManager rules)                │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Standard Metric Names (Cross-Language Convention):**

```yaml
# Semantic conventions (OpenTelemetry standard)
golden_signals:
  latency:
    metric: http.server.request.duration
    type: histogram
    unit: seconds
    labels: [http.method, http.route, http.status_code, service.name]
    
  traffic:
    metric: http.server.request.count
    type: counter
    labels: [http.method, http.route, service.name]
    
  errors:
    metric: http.server.request.count (where status_code >= 500)
    type: counter
    derived: error_rate = errors / total
    
  saturation:
    metrics:
      - process.runtime.cpu.utilization (gauge)
      - process.runtime.memory.usage (gauge)
      - http.server.active_requests (gauge)
      - db.client.connections.usage (gauge)
```

**Python Implementation:**

```python
from opentelemetry import metrics, trace
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
import time
import psutil

# Setup meter
exporter = OTLPMetricExporter(endpoint="http://adot-collector:4317", insecure=True)
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=15000)
meter_provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(meter_provider)

meter = metrics.get_meter("payment-api", "1.0.0")

# Custom metrics for golden signals
request_duration = meter.create_histogram(
    name="http.server.request.duration",
    description="Server request duration",
    unit="s"
)

active_requests = meter.create_up_down_counter(
    name="http.server.active_requests",
    description="Number of active requests"
)

# Saturation: Custom callback for system metrics
def cpu_callback(options):
    yield metrics.Observation(psutil.cpu_percent() / 100.0)

def memory_callback(options):
    yield metrics.Observation(psutil.virtual_memory().percent / 100.0)

meter.create_observable_gauge(
    name="process.runtime.cpu.utilization",
    callbacks=[cpu_callback]
)
meter.create_observable_gauge(
    name="process.runtime.memory.utilization",
    callbacks=[memory_callback]
)

# Middleware that captures all 4 golden signals
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

class GoldenSignalsMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Traffic + Saturation: Track active requests
        active_requests.add(1, {"http.method": request.method})
        
        start = time.perf_counter()
        try:
            response = await call_next(request)
            status_code = response.status_code
        except Exception as e:
            status_code = 500
            raise
        finally:
            duration = time.perf_counter() - start
            
            labels = {
                "http.method": request.method,
                "http.route": self._normalize_route(request.url.path),
                "http.status_code": str(status_code),
            }
            
            # Latency
            request_duration.record(duration, labels)
            
            # Active requests (saturation)
            active_requests.add(-1, {"http.method": request.method})
        
        return response
    
    def _normalize_route(self, path: str) -> str:
        import re
        path = re.sub(r'/[0-9a-f-]{36}', '/{id}', path)
        path = re.sub(r'/\d+', '/{id}', path)
        return path

app.add_middleware(GoldenSignalsMiddleware)
```

**Grafana Dashboard (PromQL for 4 Golden Signals):**

```promql
# LATENCY: P50, P95, P99 by service
histogram_quantile(0.99, 
  sum by (le, service_name) (
    rate(http_server_request_duration_seconds_bucket[5m])
  )
)

# TRAFFIC: Requests per second by service
sum by (service_name) (
  rate(http_server_request_duration_seconds_count[5m])
)

# ERRORS: Error rate (5xx) by service
sum by (service_name) (rate(http_server_request_duration_seconds_count{http_status_code=~"5.."}[5m]))
/
sum by (service_name) (rate(http_server_request_duration_seconds_count[5m]))

# SATURATION: CPU utilization by service
avg by (service_name) (process_runtime_cpu_utilization)
```

**Alert Rules Based on Golden Signals:**

```yaml
# prometheus-alerts.yml
groups:
  - name: golden_signals
    rules:
      # High error rate (> 1% for 5 minutes)
      - alert: HighErrorRate
        expr: |
          sum by (service_name) (rate(http_server_request_duration_seconds_count{http_status_code=~"5.."}[5m]))
          /
          sum by (service_name) (rate(http_server_request_duration_seconds_count[5m]))
          > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service_name }}: Error rate {{ $value | humanizePercentage }}"
      
      # High latency (P99 > 2 seconds)
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, sum by (service_name, le) (rate(http_server_request_duration_seconds_bucket[5m]))) > 2
        for: 5m
        labels:
          severity: warning
      
      # Traffic anomaly (drop > 50% vs same time yesterday)
      - alert: TrafficDrop
        expr: |
          sum by (service_name) (rate(http_server_request_duration_seconds_count[5m]))
          < 0.5 * sum by (service_name) (rate(http_server_request_duration_seconds_count[5m] offset 1d))
        for: 10m
        labels:
          severity: warning
      
      # Saturation: CPU > 80%
      - alert: HighCPUSaturation
        expr: avg by (service_name) (process_runtime_cpu_utilization) > 0.8
        for: 10m
        labels:
          severity: warning
```

---

### Q5: Your Grafana dashboards in Amazon Managed Grafana show "No Data" for some panels intermittently. The data exists in AMP (Prometheus). Users lose trust in the monitoring system. What's happening and how do you fix it?

**Answer:**

**Root Causes (in order of likelihood):**

**Cause 1: Query Timeout (Most Common)**
```
AMP query timeout: 120 seconds (default)
Complex PromQL on 5M time series → exceeds timeout

Fix: Optimize queries
```

```promql
-- BAD: Full scan of all series (slow)
sum(rate(http_requests_total[5m]))

-- GOOD: Add label selector to reduce scan scope
sum(rate(http_requests_total{service="payment-api"}[5m]))

-- BAD: Aggregating high-cardinality metric
avg(container_memory_working_set_bytes)

-- GOOD: Use recording rule (pre-computed)
avg(service:memory_usage:avg5m)
```

**Cause 2: Staleness (Series Disappearing)**
```
Prometheus staleness: If a series isn't scraped for 5 minutes,
it becomes "stale" and disappears from queries.

Happens when:
- Pod restarts (new pod, new series, old series goes stale)
- Scrape fails intermittently (network issue, timeout)
- Collector restart (gap in data)
```

```promql
-- Fix: Use rate() with longer window to bridge gaps
-- BAD: 1m window (if scrape interval is 30s, one miss = no data)
rate(http_requests_total[1m])

-- GOOD: 5m window (tolerates multiple missed scrapes)
rate(http_requests_total[5m])

-- Also: increase_over_time() for counters
-- Also: last_over_time() for gauges with gaps
last_over_time(cpu_utilization[10m])
```

**Cause 3: AMG → AMP Authentication Token Expiry**

```
Amazon Managed Grafana uses SigV4 to authenticate to AMP.
If the IAM role's session expires or rotates incorrectly:
- Some queries succeed (cached auth)
- Some fail (expired token)
- Result: intermittent "No Data"
```

```hcl
# Fix: Ensure Grafana workspace role has proper permissions
resource "aws_grafana_workspace" "main" {
  name                  = "platform-monitoring"
  account_access_type   = "CURRENT_ACCOUNT"
  authentication_providers = ["AWS_SSO"]
  permission_type       = "SERVICE_MANAGED"
  role_arn              = aws_iam_role.grafana.arn
  
  data_sources = ["PROMETHEUS", "CLOUDWATCH", "XRAY"]
}

resource "aws_iam_role_policy" "grafana_amp" {
  role = aws_iam_role.grafana.id
  
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = [
        "aps:QueryMetrics",
        "aps:GetMetricMetadata",
        "aps:GetSeries",
        "aps:GetLabels"
      ]
      Resource = aws_prometheus_workspace.main.arn
    }]
  })
}
```

**Cause 4: Grafana Variable Resolution Failure**

```
Dashboard variables (like $service, $namespace) that use
label_values() queries. If the variable query fails:
- All panels using that variable show "No Data"
- It's not the panel query that's wrong — it's the VARIABLE
```

```yaml
# Fix: Add fallback values and error handling
# In Grafana dashboard JSON:
{
  "templating": {
    "list": [{
      "name": "service",
      "type": "query",
      "query": "label_values(http_requests_total, service_name)",
      "refresh": 2,  # Refresh on time range change (not on every load)
      "includeAll": true,
      "allValue": ".*",  # Regex match all if variable is empty
      "current": {
        "selected": true,
        "text": "All",
        "value": "$__all"
      }
    }]
  }
}
```

**Cause 5: Series Churn (Kubernetes Pod Rotation)**

```
Problem: Kubernetes creates new pods frequently (deploys, scaling, restarts)
Each pod has unique labels → new time series
Old pod's series becomes stale

Query: sum by (pod) (cpu_usage)
Result: Shows data for current pods only. Historical data is "gone"
```

```promql
-- Fix: Aggregate at service level (not pod level)
-- BAD: Per-pod (churns with deploys)
sum by (pod) (container_cpu_usage_seconds_total{namespace="production"})

-- GOOD: Per-service (stable across deploys)
sum by (service) (
  rate(container_cpu_usage_seconds_total{namespace="production"}[5m])
)
-- "service" label comes from pod labels, aggregated across all pods
```

**Comprehensive Fix: Monitoring the Monitoring**

```yaml
# Alert on monitoring system health itself
groups:
  - name: monitoring_health
    rules:
      - alert: PrometheusTargetDown
        expr: up == 0
        for: 5m
        annotations:
          summary: "Scrape target {{ $labels.job }} is down"
      
      - alert: PrometheusHighScrapeInterval
        expr: scrape_duration_seconds > 30
        for: 5m
        annotations:
          summary: "Slow scrape: {{ $labels.job }} taking {{ $value }}s"
      
      - alert: AMPIngestionErrors
        expr: rate(prometheus_remote_storage_failed_samples_total[5m]) > 0
        for: 5m
        annotations:
          summary: "AMP ingestion failures detected"
      
      - alert: GrafanaDatasourceError
        expr: grafana_datasource_request_duration_seconds{status="error"} > 0
        for: 2m
```

---

### Q6: You need to correlate logs, metrics, and traces for a single user request that spans 8 microservices. A user reports "slow checkout" but you don't know which service is the bottleneck. How do you implement end-to-end request tracing with correlation?

**Answer:**

**Correlation Architecture:**

```
User Request (trace_id: abc-123)
    │
    ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ API Gateway │→ │ Order Svc   │→ │ Payment Svc │→ ...
│             │  │             │  │             │
│ trace: abc  │  │ trace: abc  │  │ trace: abc  │
│ span: 001   │  │ span: 002   │  │ span: 003   │
│ log: abc    │  │ log: abc    │  │ log: abc    │
│ metric: abc │  │ metric: abc │  │ metric: abc │
└─────────────┘  └─────────────┘  └─────────────┘

ALL telemetry (logs, metrics, traces) share the SAME trace_id
→ Query ANY signal → correlate to ALL other signals for that request
```

**Implementation: Trace Context Propagation**

```python
# Service A: FastAPI with OTEL instrumentation
from opentelemetry import trace, context
from opentelemetry.propagate import inject, extract
from opentelemetry.trace import SpanKind, StatusCode
import structlog
import httpx

tracer = trace.get_tracer("order-service")
logger = structlog.get_logger()

@app.post("/v1/orders")
async def create_order(order: OrderRequest, request: Request):
    # Extract trace context from incoming request headers
    ctx = extract(request.headers)
    
    with tracer.start_as_current_span(
        "create_order",
        context=ctx,
        kind=SpanKind.SERVER,
        attributes={
            "order.customer_id": order.customer_id,
            "order.amount": float(order.total_amount),
            "order.items_count": len(order.items)
        }
    ) as span:
        # Get current trace and span IDs for log correlation
        span_context = span.get_span_context()
        trace_id = format(span_context.trace_id, '032x')
        span_id = format(span_context.span_id, '016x')
        
        # Structured log with trace correlation
        logger.info(
            "Processing order",
            trace_id=trace_id,
            span_id=span_id,
            customer_id=order.customer_id,
            amount=float(order.total_amount)
        )
        
        # Call downstream service (propagate trace context)
        async with httpx.AsyncClient() as client:
            headers = {}
            inject(headers)  # Injects traceparent header
            
            # Payment service call (child span)
            with tracer.start_as_current_span("call_payment_service", kind=SpanKind.CLIENT):
                response = await client.post(
                    "http://payment-service/v1/charge",
                    json={"amount": order.total_amount, "customer_id": order.customer_id},
                    headers=headers  # Contains: traceparent: 00-{trace_id}-{span_id}-01
                )
                
                if response.status_code != 200:
                    span.set_status(StatusCode.ERROR, "Payment failed")
                    span.set_attribute("error.type", "payment_failure")
                    logger.error(
                        "Payment failed",
                        trace_id=trace_id,
                        status_code=response.status_code
                    )
                    raise PaymentError(response.text)
        
        # Record duration as metric WITH trace exemplar
        request_duration.record(
            duration,
            attributes={
                "http.route": "/v1/orders",
                "http.status_code": "200"
            }
        )
        
        return {"order_id": new_order.id, "trace_id": trace_id}
```

**Log Format (Correlated):**

```json
{
  "timestamp": "2024-06-15T10:30:00.123Z",
  "level": "INFO",
  "service": "order-service",
  "message": "Processing order",
  "trace_id": "abc123def456789",
  "span_id": "0011223344556677",
  "customer_id": "cust-456",
  "amount": 99.99,
  "environment": "production"
}
```

**Querying Correlated Data:**

```sql
-- CloudWatch Logs Insights: Find all logs for a trace
fields @timestamp, service, message, @logStream
| filter trace_id = "abc123def456789"
| sort @timestamp asc

-- Result shows the request flowing through all 8 services:
-- 10:30:00.123  api-gateway     "Request received"
-- 10:30:00.145  order-service   "Processing order"
-- 10:30:00.200  inventory-svc   "Checking stock"
-- 10:30:00.350  payment-svc     "Charging customer"
-- 10:30:02.500  payment-svc     "Payment timeout!"  ← BOTTLENECK FOUND!
-- 10:30:05.100  order-service   "Payment failed, retrying"
```

**X-Ray Trace Map (Visual):**

```
API Gateway (2ms) → Order Service (5005ms) → Payment Service (4800ms) ← SLOW!
                                            → Inventory Service (45ms)
                                            → Notification Service (120ms)
                  
Payment Service breakdown:
├── Stripe API call: 4500ms (external dependency timeout!)
├── DB write: 200ms
└── Event publish: 100ms

Root Cause: Stripe API was degraded, causing 4.5 second response
```

**Metric Exemplars (Jump from Metric → Trace):**

```python
# Record metric with exemplar (link to specific trace)
from opentelemetry.sdk.metrics import Histogram

# When recording a metric, attach the trace_id as exemplar
# Grafana can then show: "Click to see trace for this P99 data point"
request_duration.record(
    duration,
    attributes={"endpoint": "/v1/orders", "status": "200"},
    # Exemplar: links this metric data point to a specific trace
    exemplar={"trace_id": trace_id}
)
```

**Grafana Correlation:**
```
Dashboard Panel (P99 latency spike) 
  → Click data point 
  → "View Trace" (exemplar link)
  → X-Ray trace view (shows full request path)
  → Click slow span
  → "View Logs" (filtered by trace_id + time range)
  → See exact error message

Total investigation time: 30 seconds (vs 30 minutes without correlation)
```

---

### Q7: Your Prometheus alerting fires 50+ alerts during a single incident (cascading alerts). On-call engineer is overwhelmed. How do you implement alert grouping, inhibition, and routing to reduce noise to 1-3 actionable alerts per incident?

**Answer:**

**Problem: Alert Storm During Cascading Failure**

```
Database goes down → 50 alerts fire simultaneously:
├── DBConnectionTimeout (10 services × 5 alerts each)
├── HighErrorRate (all services returning 500)
├── HighLatency (requests queuing)
├── QueueDepthHigh (messages backing up)
├── HealthCheckFailing (services marked unhealthy)
└── PodRestarting (crash loops)

On-call sees 50 PagerDuty pages in 2 minutes. Useless.
Should see: 1 alert → "Database cluster-primary is unreachable"
```

**Solution: AlertManager Configuration**

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/xxx'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# ROUTING: Direct alerts to the right channel
route:
  receiver: 'default-slack'
  group_by: ['alertname', 'service', 'namespace']
  group_wait: 30s        # Wait 30s to batch related alerts
  group_interval: 5m     # Time between batched notifications
  repeat_interval: 4h    # Don't re-fire same alert for 4 hours
  
  routes:
    # Critical: Page immediately
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      group_wait: 10s       # Page faster for critical
      group_interval: 1m
      continue: false       # Stop routing after match
    
    # Warning: Slack during business hours, queue overnight
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_wait: 60s
      group_interval: 15m
      
      routes:
        # Specific team routing
        - match:
            team: data-platform
          receiver: 'slack-data-team'
        - match:
            team: payments
          receiver: 'slack-payments-team'
    
    # Info: Just log
    - match:
        severity: info
      receiver: 'null'  # Silence

# INHIBITION: Suppress downstream alerts when root cause is firing
inhibit_rules:
  # If database is down, suppress all "connection timeout" alerts
  - source_match:
      alertname: 'DatabaseDown'
    target_match:
      alertname: 'DBConnectionTimeout'
    equal: ['database_cluster']  # Must be the same cluster
  
  # If a node is down, suppress all pod alerts on that node
  - source_match:
      alertname: 'NodeNotReady'
    target_match_re:
      alertname: '(PodCrashLooping|PodNotReady|HighCPU|HighMemory)'
    equal: ['node']
  
  # If service is down, suppress SLO burn rate alerts for that service
  - source_match:
      alertname: 'ServiceDown'
    target_match:
      alertname: 'SLOBurnRateHigh'
    equal: ['service_name']
  
  # If cluster-wide issue, suppress individual service alerts
  - source_match:
      scope: cluster
      severity: critical
    target_match:
      scope: service
    equal: ['cluster']

# RECEIVERS
receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '$PD_ROUTING_KEY'
        severity: critical
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        details:
          service: '{{ .GroupLabels.service }}'
          runbook: '{{ .CommonAnnotations.runbook_url }}'
        # Deduplication key: Prevents duplicate pages for same incident
        group: '{{ .GroupLabels.alertname }}-{{ .GroupLabels.service }}'
  
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        title: '⚠️ {{ .GroupLabels.alertname }} ({{ .Alerts | len }} alerts)'
        text: |
          *Service:* {{ .GroupLabels.service }}
          *Summary:* {{ .CommonAnnotations.summary }}
          *Affected:* {{ .Alerts | len }} instances
          *Runbook:* {{ .CommonAnnotations.runbook_url }}
        send_resolved: true
  
  - name: 'null'
    # Intentionally empty — discards alerts
```

**Alert Design Rules (Before AlertManager):**

```yaml
# Rule 1: Alert on SYMPTOMS, not CAUSES
# BAD: Alert on CPU > 80% (cause)
# GOOD: Alert on error rate > 1% (symptom user experiences)

# Rule 2: Multi-window, multi-burn-rate SLO alerts
groups:
  - name: slo_alerts
    rules:
      # Fast burn: 14.4x budget consumption → alert in 2 min
      # (Exhausts 30-day budget in 2 hours)
      - alert: SLOBurnRateHigh_Fast
        expr: |
          (
            sum by (service_name) (rate(http_requests_total{status_code=~"5.."}[5m]))
            / sum by (service_name) (rate(http_requests_total[5m]))
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
          scope: service
        annotations:
          summary: "{{ $labels.service_name }} burning error budget 14.4x faster than allowed"
          runbook_url: "https://wiki.company.com/runbooks/slo-burn"
      
      # Slow burn: 3x budget consumption → alert in 1 hour
      # (Exhausts 30-day budget in 10 days)
      - alert: SLOBurnRateHigh_Slow
        expr: |
          (
            sum by (service_name) (rate(http_requests_total{status_code=~"5.."}[1h]))
            / sum by (service_name) (rate(http_requests_total[1h]))
          ) > (3 * 0.001)
        for: 1h
        labels:
          severity: warning
          scope: service

# Rule 3: Include runbook URL in EVERY alert
# Rule 4: Every alert MUST be actionable
# Rule 5: Review alert quality monthly (false positive rate < 5%)
```

**Result:**

```
Before: 50 alerts per incident, 30% false positive rate
After:
- Inhibition: Suppresses 40 downstream alerts → 10 remain
- Grouping: Groups 10 alerts into 2 groups → 2 notifications
- Routing: Critical → PagerDuty, Warning → Slack
- SLO-based: Alerts on IMPACT not CAUSE

On-call receives: 1-2 pages per real incident
Each page includes: Summary, Runbook, Affected Services
```

---

### Q8: You're migrating from CloudWatch to a Prometheus+Grafana stack (AMP+AMG). During the migration, you need both systems running. How do you ensure no observability gaps and eventually decommission CloudWatch metrics?

**Answer:**

**Migration Strategy: Parallel Run → Validation → Cutover**

```
Phase 1 (Weeks 1-4): Dual-Write
┌──────────┐    ┌─────────────┐    ┌────────────────┐
│ Services │───▶│ ADOT        │───▶│ AMP (new)      │
│          │    │ Collector   │    └────────────────┘
│          │    │ (dual-write)│───▶│ CloudWatch(old)│
└──────────┘    └─────────────┘    └────────────────┘

Phase 2 (Weeks 5-8): Validate Parity
- Same data in both systems
- Grafana dashboards replicate CloudWatch dashboards
- Alerts firing correctly in both

Phase 3 (Weeks 9-12): Cutover
- Switch alerting to Prometheus
- Switch dashboards to Grafana
- Keep CloudWatch read-only (historical data)
- Stop writing to CloudWatch

Phase 4 (Week 13+): Decommission
- Remove CloudWatch custom metric publishing
- Keep CloudWatch for AWS-native metrics only (EC2, RDS, etc.)
```

**Dual-Write Collector Configuration:**

```yaml
# ADOT Collector: Write to BOTH AMP and CloudWatch simultaneously
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }
  
  # Also scrape existing Prometheus endpoints
  prometheus:
    config:
      scrape_configs:
        - job_name: 'kubernetes-pods'
          kubernetes_sd_configs:
            - role: pod

processors:
  batch:
    timeout: 10s
    send_batch_size: 1000

exporters:
  # NEW: Amazon Managed Prometheus
  prometheusremotewrite:
    endpoint: "https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-xxx/api/v1/remote_write"
    auth:
      authenticator: sigv4auth
  
  # OLD: CloudWatch (keep during migration)
  awsemf:
    namespace: 'Application'
    region: 'us-east-1'
    log_group_name: '/metrics/application'
    dimension_rollup_option: "NoDimensionRollup"

extensions:
  sigv4auth:
    region: us-east-1
    service: aps

service:
  extensions: [sigv4auth]
  pipelines:
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch]
      exporters: [prometheusremotewrite, awsemf]  # DUAL WRITE
```

**Validation: Compare Both Systems**

```python
# Script to validate metric parity between CloudWatch and AMP
import boto3
import requests
from datetime import datetime, timedelta

def validate_parity(metric_name, service_name, tolerance_pct=5):
    """Compare same metric in CloudWatch vs AMP."""
    
    # Get from CloudWatch
    cw = boto3.client('cloudwatch')
    cw_response = cw.get_metric_statistics(
        Namespace='Application',
        MetricName=metric_name,
        Dimensions=[{'Name': 'Service', 'Value': service_name}],
        StartTime=datetime.utcnow() - timedelta(hours=1),
        EndTime=datetime.utcnow(),
        Period=300,
        Statistics=['Average', 'Sum']
    )
    cw_avg = cw_response['Datapoints'][0]['Average'] if cw_response['Datapoints'] else None
    
    # Get from AMP (PromQL)
    amp_response = requests.post(
        f"{AMP_ENDPOINT}/api/v1/query",
        data={'query': f'avg(http_request_duration_seconds{{service="{service_name}"}})'},
        auth=SigV4Auth()
    )
    amp_avg = float(amp_response.json()['data']['result'][0]['value'][1]) if amp_response.json()['data']['result'] else None
    
    # Compare
    if cw_avg and amp_avg:
        diff_pct = abs(cw_avg - amp_avg) / cw_avg * 100
        status = "✅ PASS" if diff_pct < tolerance_pct else "❌ FAIL"
        print(f"{status} {metric_name}/{service_name}: CW={cw_avg:.3f}, AMP={amp_avg:.3f}, diff={diff_pct:.1f}%")
        return diff_pct < tolerance_pct
    
    print(f"⚠️  MISSING DATA: CW={cw_avg}, AMP={amp_avg}")
    return False

# Run parity checks
metrics_to_validate = [
    ("RequestCount", "payment-api"),
    ("ErrorRate", "payment-api"),
    ("P99Latency", "payment-api"),
    ("CPUUtilization", "order-service"),
]

results = [validate_parity(m, s) for m, s in metrics_to_validate]
print(f"\nParity: {sum(results)}/{len(results)} metrics match")
```

**Grafana Dashboard Migration Tool:**

```python
# Convert CloudWatch dashboard JSON to Grafana dashboard JSON
def convert_cw_to_grafana(cw_dashboard):
    """Convert CloudWatch dashboard to Grafana format."""
    grafana_panels = []
    
    for widget in cw_dashboard['widgets']:
        if widget['type'] == 'metric':
            panel = {
                'type': 'timeseries',
                'title': widget['properties'].get('title', ''),
                'datasource': {'type': 'prometheus', 'uid': 'amp-datasource'},
                'targets': []
            }
            
            for metric in widget['properties'].get('metrics', []):
                # Convert CloudWatch metric to PromQL
                promql = convert_metric_to_promql(metric)
                panel['targets'].append({
                    'expr': promql,
                    'legendFormat': '{{service_name}}'
                })
            
            grafana_panels.append(panel)
    
    return {
        'dashboard': {
            'title': f"[Migrated] {cw_dashboard.get('dashboardName', '')}",
            'panels': grafana_panels
        }
    }

def convert_metric_to_promql(cw_metric):
    """Convert CloudWatch metric query to PromQL."""
    # CW: ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/my-alb"]
    # PromQL: sum(rate(http_requests_total{load_balancer="app/my-alb"}[5m]))
    
    namespace = cw_metric[0]
    metric_name = cw_metric[1]
    
    mapping = {
        'RequestCount': 'sum(rate(http_requests_total{__FILTERS__}[5m]))',
        'TargetResponseTime': 'histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{__FILTERS__}[5m])))',
        'HTTPCode_Target_5XX_Count': 'sum(rate(http_requests_total{status_code=~"5.."}[5m]))',
    }
    
    return mapping.get(metric_name, f'# TODO: Convert {metric_name}')
```

**Cutover Checklist:**

```yaml
cutover_checklist:
  pre_cutover:
    - [ ] All dashboards recreated in Grafana
    - [ ] All alerts migrated to Prometheus AlertManager
    - [ ] Parity validation passing > 95% for 2 weeks
    - [ ] On-call team trained on Grafana/PromQL
    - [ ] Runbooks updated with new query syntax
    - [ ] Escalation paths configured in AlertManager
    
  cutover_day:
    - [ ] Disable CloudWatch alarms (don't delete yet)
    - [ ] Enable Prometheus alerting rules
    - [ ] Update PagerDuty/Slack integrations
    - [ ] Monitor for 24 hours
    
  post_cutover:
    - [ ] Remove awsemf exporter from ADOT (stop dual-write)
    - [ ] Keep CloudWatch for AWS-native metrics (RDS, ALB, etc.)
    - [ ] Configure Grafana CloudWatch data source for AWS metrics
    - [ ] Delete custom CloudWatch namespaces after 90 days
    
  keep_in_cloudwatch:
    - AWS/EC2 (instance metrics)
    - AWS/RDS (database metrics)
    - AWS/ApplicationELB (load balancer)
    - AWS/ECS (container insights)
    - AWS/Lambda (invocation metrics)
    # These are FREE and auto-generated by AWS
```

---

### Q9: Your production system needs 99.99% availability SLO. CloudWatch health checks detect failures in 30 seconds, but by then thousands of users are affected. How do you achieve sub-10-second detection?

**Answer:**

**Detection Speed Comparison:**

```
Method                          Detection Time    Coverage
─────────────────────────────────────────────────────────
Route 53 Health Check           30 seconds        Endpoint only
CloudWatch Alarm (1-min period) 60-120 seconds    Metric-based
CloudWatch Alarm (10s period)   10-30 seconds     High-resolution
Synthetic Canaries              60 seconds        User journey
Custom in-app health reporting  1-5 seconds       Real-time
Client-side error detection     < 1 second        Edge detection
```

**Architecture: Multi-Layer Fast Detection**

```
┌──────────────────────────────────────────────────────────────┐
│  Layer 1: Client-Side Detection (< 1 second)                  │
│  Browser/Mobile → Error Rate Spike → CloudFront Function      │
│  → Immediate traffic shift (no backend round-trip needed)     │
└──────────────────────────────────────────────────────────────┘
         │
┌────────▼─────────────────────────────────────────────────────┐
│  Layer 2: Edge Detection (< 5 seconds)                        │
│  ALB Target Health Check: interval=5s, threshold=2             │
│  → Unhealthy target removed from rotation in 10 seconds       │
└──────────────────────────────────────────────────────────────┘
         │
┌────────▼─────────────────────────────────────────────────────┐
│  Layer 3: Application-Level Detection (< 10 seconds)          │
│  High-Resolution CloudWatch Metrics (1-second granularity)    │
│  Custom metric: Error count per second                        │
│  Alarm: evaluationPeriods=5, period=1 (5 seconds of errors)   │
└──────────────────────────────────────────────────────────────┘
         │
┌────────▼─────────────────────────────────────────────────────┐
│  Layer 4: Synthetic Monitoring (< 60 seconds)                 │
│  CloudWatch Synthetics: Canary every 1 minute                 │
│  Validates full user journeys (login, checkout, etc.)         │
└──────────────────────────────────────────────────────────────┘
```

**Layer 1: Client-Side Error Detection**

```javascript
// CloudFront Function: Detect error rate at edge
function handler(event) {
    var request = event.request;
    
    // CloudFront Functions can read custom headers from origin response
    // Use CloudFront real-time logs → Kinesis → Lambda → fast detection
    
    return request;
}

// Real-time log analysis (< 5 second detection)
// CloudFront → Kinesis Data Stream → Lambda → Alarm
```

**Layer 2: ALB Fast Health Checks**

```hcl
resource "aws_lb_target_group" "fast_detection" {
  name     = "api-fast-health"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = var.vpc_id
  
  health_check {
    enabled             = true
    path                = "/health/live"
    port                = "traffic-port"
    protocol            = "HTTP"
    
    # Fast detection settings:
    interval            = 5    # Check every 5 seconds (minimum)
    timeout             = 2    # Fail after 2 seconds
    healthy_threshold   = 2    # 2 passes to mark healthy (10s)
    unhealthy_threshold = 2    # 2 fails to mark unhealthy (10s!)
    matcher             = "200"
  }
  
  # Slow start to prevent overwhelming new targets
  slow_start = 30
}
```

**Layer 3: High-Resolution CloudWatch Alarms**

```hcl
# 1-second resolution custom metric
resource "aws_cloudwatch_metric_alarm" "fast_error_detection" {
  alarm_name          = "api-error-rate-fast"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5       # 5 data points
  period              = 1       # 1-second period (high resolution!)
  # Detection time: 5 seconds
  
  metric_name         = "ErrorCount"
  namespace           = "Application/HighRes"
  statistic           = "Sum"
  threshold           = 10      # 10 errors in 1 second = problem
  
  alarm_actions = [
    aws_sns_topic.fast_response.arn,
    # Trigger auto-remediation Lambda
    aws_lambda_function.auto_failover.arn
  ]
}
```

```python
# Application: Publish high-resolution metrics (1-second)
import boto3
import time
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

class HighResMetricPublisher:
    """Publish metrics every second for fast detection."""
    
    def __init__(self):
        self.error_count = 0
        self.request_count = 0
        self._last_publish = time.time()
    
    def record_request(self, success: bool):
        self.request_count += 1
        if not success:
            self.error_count += 1
        
        # Publish every second
        if time.time() - self._last_publish >= 1.0:
            self._publish()
    
    def _publish(self):
        cloudwatch.put_metric_data(
            Namespace='Application/HighRes',
            MetricData=[
                {
                    'MetricName': 'ErrorCount',
                    'Value': self.error_count,
                    'Timestamp': datetime.utcnow(),
                    'StorageResolution': 1,  # HIGH RESOLUTION (1 second)
                    'Unit': 'Count'
                },
                {
                    'MetricName': 'RequestCount',
                    'Value': self.request_count,
                    'Timestamp': datetime.utcnow(),
                    'StorageResolution': 1,
                    'Unit': 'Count'
                }
            ]
        )
        self.error_count = 0
        self.request_count = 0
        self._last_publish = time.time()
```

**Layer 4: Auto-Remediation (< 30 seconds total)**

```python
# Lambda: Triggered by fast alarm → auto-failover
def auto_remediate(event, context):
    """Automatic failover when fast detection triggers."""
    
    alarm = event['detail']
    
    # Step 1: Verify it's real (not a blip)
    # Check if error rate is sustained
    if not verify_sustained_failure():
        return {"action": "suppressed", "reason": "transient"}
    
    # Step 2: Auto-remediate based on failure type
    if alarm['alarmName'] == 'api-error-rate-fast':
        # Option A: Scale up (if it's a capacity issue)
        if is_capacity_issue():
            scale_up_service()
        
        # Option B: Rollback (if it's a deployment issue)
        elif is_recent_deployment():
            rollback_deployment()
        
        # Option C: Failover to secondary region
        elif is_regional_issue():
            trigger_dns_failover()
    
    # Step 3: Notify (AFTER action, not before)
    notify_oncall(alarm, action_taken)
```

**99.99% SLO Math:**

```
99.99% = 52.6 minutes of allowed downtime per YEAR
= 4.38 minutes per MONTH
= 8.6 seconds per DAY

With 30-second detection: you've already used 30s of your daily budget!
With 5-second detection: you have time to auto-remediate before user impact

Detection (5s) + Failover (10s) + DNS propagation (0s with Global Accelerator)
= 15 seconds total recovery time
= Well within 99.99% budget
```

---

### Q10: Your team is debating: "Should we use CloudWatch native or build a custom observability platform with Prometheus+Grafana?" What are the decision criteria and when would you choose each?

**Answer:**

**Decision Matrix:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  Factor              │ CloudWatch          │ Prometheus+Grafana      │
├──────────────────────┼─────────────────────┼─────────────────────────┤
│ Setup effort         │ Zero (auto-enabled) │ High (infra to manage)  │
│ AWS integration      │ Native (perfect)    │ Exporters needed        │
│ Custom metrics       │ $0.30/metric/month  │ Free (self-hosted)      │
│ High cardinality     │ Expensive           │ Handles well            │
│ PromQL support       │ No (CW query lang)  │ Yes (industry standard) │
│ Alerting flexibility │ Basic (CW Alarms)   │ Advanced (AlertManager) │
│ Dashboard quality    │ Basic               │ Excellent (Grafana)     │
│ Multi-cloud          │ AWS only            │ Cloud-agnostic          │
│ Retention            │ 15 months           │ Configurable (years)    │
│ Team knowledge       │ Low barrier         │ Requires PromQL skills  │
│ Managed service      │ Fully managed       │ AMP+AMG = managed       │
│ Cost at scale        │ Very expensive      │ Moderate                │
│ Kubernetes native    │ Container Insights  │ Native fit              │
│ OpenTelemetry        │ Supported (ADOT)    │ Native                  │
│ Cross-account        │ Cross-account obs.  │ Federated/Thanos        │
└──────────────────────┴─────────────────────┴─────────────────────────┘
```

**When to Choose CloudWatch:**

```yaml
choose_cloudwatch_when:
  - Small team (< 5 engineers)
  - Pure AWS workloads (no multi-cloud)
  - Low custom metric volume (< 1000 metrics)
  - Don't need high-cardinality metrics
  - Want zero operational overhead
  - AWS-native services (Lambda, RDS, ALB) are primary
  - Budget is not a concern at current scale
  - Team doesn't know PromQL
  
  ideal_scenario:
    "Startup with 10 Lambda functions and 3 RDS databases.
     CloudWatch is free for AWS metrics, alarms are simple,
     and there's nothing to operate."
```

**When to Choose Prometheus+Grafana (AMP+AMG):**

```yaml
choose_prometheus_when:
  - Large-scale (> 50 microservices)
  - Kubernetes workloads (EKS/ECS)
  - High-cardinality metrics needed
  - Team knows PromQL (or willing to learn)
  - Need advanced alerting (inhibition, grouping, routing)
  - Multi-cloud or hybrid (same tool everywhere)
  - CloudWatch costs > $10K/month
  - Need Grafana-quality dashboards
  - OpenTelemetry standardization is a goal
  
  ideal_scenario:
    "Platform team managing 200 microservices on EKS.
     Need service mesh metrics, custom SLO dashboards,
     advanced alerting with correlation, and PromQL
     is the industry standard their engineers know."
```

**Best Practice: Hybrid Approach**

```yaml
hybrid_architecture:
  cloudwatch_for:
    - AWS service metrics (free, auto-generated)
      # EC2, RDS, ALB, Lambda, ECS (Container Insights)
    - VPC Flow Logs analysis
    - CloudTrail integration
    - AWS-native alarms (billing, service health)
    - Logs (CloudWatch Logs → queries via Insights)
  
  prometheus_for:
    - Application metrics (custom business metrics)
    - Kubernetes infrastructure metrics
    - SLO/SLI tracking and error budgets
    - High-cardinality metrics (per-endpoint, per-customer)
    - Advanced alerting with AlertManager
    - Dashboards (Grafana is superior to CloudWatch dashboards)
  
  unified_via:
    - Grafana as single pane of glass
    - CloudWatch data source in Grafana (query both)
    - ADOT collector (routes to both destinations)
    - X-Ray for distributed tracing (works with both)
    
  architecture:
    "Application → OTEL SDK → ADOT Collector
     ├── Metrics → AMP (Prometheus) → Grafana
     ├── Traces → X-Ray → Grafana (X-Ray data source)
     └── Logs → CloudWatch Logs → Grafana (CW data source)
     
     AWS Services → CloudWatch (native) → Grafana (CW data source)"
```

**Cost Comparison at Scale:**

```
Scenario: 200 microservices, 100K custom metrics, 50TB logs/month

CloudWatch only:
├── Custom metrics: 100K × $0.30 = $30,000/month
├── Log ingestion: 50TB × $0.50/GB = $25,000/month
├── Log storage: 150TB × $0.03/GB = $4,500/month
├── Dashboards API: ~$2,000/month
└── Total: ~$61,500/month

AMP + AMG + CloudWatch (hybrid):
├── AMP (custom metrics): 100K series × $0.03/1K samples = $3,000/month
├── AMG: $9/user/month × 50 users = $450/month
├── CloudWatch (AWS metrics only): $0 (free tier)
├── Log ingestion (CW): 50TB × $0.50/GB = $25,000/month
│   └── Alternative: OpenSearch for logs = $15K/month
├── ADOT collectors: Runs on existing EKS (no extra cost)
└── Total: ~$28,450/month (54% savings)

Or with log optimization (sampling, filtering):
└── Total: ~$18,000/month (71% savings)
```
