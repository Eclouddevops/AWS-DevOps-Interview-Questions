# CloudWatch & Observability — Complete Knowledge Guide

> **Purpose**: Understand observability from fundamentals — the three pillars (metrics, logs, traces), how CloudWatch works internally, when to use Prometheus/Grafana, and how to build production-grade monitoring systems.

---

## 1. Observability vs Monitoring (The Key Distinction)

```
MONITORING (Traditional):
"Is the system UP or DOWN?"
├── Predefined checks (CPU > 80%? Disk > 90%?)
├── Known-unknowns (you know what to check)
├── Dashboard-driven (human watches screens)
└── Reactive (alerts when thresholds crossed)

OBSERVABILITY (Modern):
"WHY is the system behaving this way?"
├── Explore any question without predefined dashboards
├── Unknown-unknowns (discover problems you didn't expect)
├── Data-driven (query telemetry to find root cause)
└── Proactive (detect anomalies before users notice)

The Three Pillars:
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  METRICS          LOGS              TRACES                       │
│  (Numbers)        (Events)          (Request paths)              │
│                                                                   │
│  "What is         "Why did it       "Where is the               │
│   happening?"      happen?"          bottleneck?"               │
│                                                                   │
│  CloudWatch       CloudWatch        X-Ray                        │
│  Metrics          Logs              (Distributed                 │
│  Prometheus       OpenSearch         Tracing)                    │
│                                                                   │
│  Time-series      Structured text   Spans across                 │
│  Aggregatable     Searchable        services                     │
│  Low cost         High volume       Shows causality              │
│                                                                   │
│  Example:         Example:          Example:                     │
│  CPU = 85%        "ERROR: DB conn   Request → API (2ms)         │
│  Requests = 500/s  timeout after     → Auth (5ms)               │
│  Errors = 2%      30s at line 42"   → DB (2500ms!) ← SLOW      │
│                                      → Cache (1ms)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Amazon CloudWatch (Complete Understanding)

### CloudWatch Components

```
┌─────────────────────────────────────────────────────────────────┐
│  CloudWatch = 5 Services Under One Umbrella                      │
│                                                                   │
│  1. CloudWatch METRICS                                           │
│     ├── Time-series numerical data                              │
│     ├── AWS service metrics (free, automatic)                   │
│     ├── Custom metrics (you publish, $0.30/metric/month)        │
│     ├── Namespaces: AWS/EC2, AWS/RDS, Custom/MyApp             │
│     └── Resolution: Standard (60s) or High-res (1s)            │
│                                                                   │
│  2. CloudWatch LOGS                                              │
│     ├── Centralized log storage and analysis                    │
│     ├── Log Groups → Log Streams → Log Events                  │
│     ├── Logs Insights (SQL-like query language)                 │
│     ├── Metric Filters (extract metrics from logs — FREE!)     │
│     ├── Subscription Filters (stream to Lambda/Kinesis/ES)     │
│     └── Cost: $0.50/GB ingested + $0.03/GB stored              │
│                                                                   │
│  3. CloudWatch ALARMS                                            │
│     ├── Monitor metrics and trigger actions                     │
│     ├── States: OK → ALARM → INSUFFICIENT_DATA                 │
│     ├── Actions: SNS, Auto Scaling, EC2, Lambda                │
│     ├── Composite Alarms (combine multiple alarms)             │
│     └── Anomaly Detection (ML-based dynamic thresholds)        │
│                                                                   │
│  4. CloudWatch DASHBOARDS                                        │
│     ├── Visual display of metrics                               │
│     ├── Cross-account, cross-region                             │
│     ├── Auto-refresh (configurable interval)                    │
│     └── Cost: $3/dashboard/month                                │
│                                                                   │
│  5. CloudWatch SYNTHETICS                                        │
│     ├── Canary scripts (test endpoints proactively)             │
│     ├── Runs on schedule (every 1-5 minutes)                   │
│     ├── Screenshots, HAR files, response times                 │
│     └── Detect issues BEFORE users report them                 │
└─────────────────────────────────────────────────────────────────┘
```

### Metrics: How They Work Internally

```
METRIC ANATOMY:
├── Namespace: "AWS/EC2" or "MyApp/PaymentService"
├── Metric Name: "CPUUtilization" or "RequestLatency"
├── Dimensions: Key-value pairs that identify the metric
│   └── Example: InstanceId=i-123, AutoScalingGroupName=prod-asg
├── Timestamp: When the data point was recorded
├── Value: The actual number
├── Unit: Seconds, Bytes, Count, Percent, etc.
└── Statistics: Sum, Average, Min, Max, SampleCount, pN (percentiles)

IMPORTANT CONCEPT: Dimensions define WHICH metric you're looking at

Example — These are THREE DIFFERENT metrics:
├── CPUUtilization{InstanceId=i-111}  ← Metric for instance 1
├── CPUUtilization{InstanceId=i-222}  ← Metric for instance 2
├── CPUUtilization{AutoScalingGroupName=prod}  ← Aggregated for ASG
                                              
CloudWatch stores them SEPARATELY (no automatic aggregation!)
If you want "average CPU across all instances" → you must query with
the ASG dimension or use Metric Math.
```

### CloudWatch Logs Architecture

```
Log Groups        Log Streams           Log Events
────────────      ─────────────         ─────────────────────────
/ecs/payment-api  ├── ecs/task-abc123   {"time":"...", "msg":"..."}
                  ├── ecs/task-def456   {"time":"...", "msg":"..."}
                  └── ecs/task-ghi789   {"time":"...", "msg":"..."}

/aws/lambda/func  ├── 2024/06/15/[$LATEST]abc
                  └── 2024/06/15/[$LATEST]def

Rules:
├── Log Group = logical grouping (one per service/function)
├── Log Stream = one source within the group (one per container/instance)
├── Log Event = single log line with timestamp
├── Retention: Set per group (1 day to 10 years, or never expire)
└── Encryption: KMS or AWS-managed (always encrypted at rest)
```

### Logs Insights Query Language

```sql
-- Find errors in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50

-- Request latency percentiles
fields @timestamp, duration_ms
| stats avg(duration_ms) as avg_latency,
        percentile(duration_ms, 95) as p95,
        percentile(duration_ms, 99) as p99
  by bin(5m)

-- Top 10 slowest endpoints
fields @timestamp, endpoint, duration_ms
| filter duration_ms > 1000
| stats count(*) as slow_count, avg(duration_ms) as avg_ms by endpoint
| sort slow_count desc
| limit 10

-- Error rate over time
fields @timestamp
| stats count(*) as total,
        count(*) * (filter(@message like /ERROR/)) as errors
  by bin(5m)
| display total, errors, (errors/total)*100 as error_pct

-- Find a specific request by trace ID
fields @timestamp, @message, service, trace_id
| filter trace_id = "abc-123-def-456"
| sort @timestamp asc

-- Parse JSON logs
fields @timestamp, @message
| parse @message '{"level":"*","service":"*","duration":*}' 
  as level, service, duration
| filter level = "ERROR"
| stats count(*) by service
```

### Alarms: The Three States

```
┌──────────────────────────────────────────────────────────────┐
│  CloudWatch Alarm States:                                     │
│                                                               │
│  ┌────────┐     threshold      ┌─────────┐                 │
│  │   OK   │ ────breached────▶  │  ALARM  │                 │
│  │        │ ◀───recovered────  │         │                 │
│  └────────┘                    └─────────┘                 │
│       │                              │                      │
│       └──── missing data ───▶  ┌─────────────────┐         │
│                                │ INSUFFICIENT    │         │
│       ◀──── data resumes ────  │ DATA            │         │
│                                └─────────────────┘         │
│                                                               │
│  Configuration:                                               │
│  ├── Period: How long each data point covers (60s, 300s)    │
│  ├── Evaluation Periods: How many periods to check          │
│  ├── Datapoints to Alarm: How many must breach (M of N)     │
│  └── Treat Missing Data: breaching | notBreaching | missing │
│                                                               │
│  Example: "3 out of 5 consecutive 1-minute periods > 80%"   │
│  = Must be over 80% CPU for 3 of the last 5 minutes         │
│  = Prevents alerting on brief spikes (reduces noise!)        │
└──────────────────────────────────────────────────────────────┘
```

### Composite Alarms (Reduce Alert Noise)

```
PROBLEM: 10 individual alarms fire for same incident
SOLUTION: Composite alarm that combines multiple signals

Example: "Service is degraded" = High Error Rate AND High Latency
(Not just one of them — both together means real problem)

alarm_rule = "ALARM(high-error-rate) AND ALARM(high-latency)"
# Only fires when BOTH conditions are true simultaneously

Example: "AZ failure" = Multiple services down in same AZ
alarm_rule = <<-RULE
  ALARM(service-a-az1-unhealthy) AND 
  ALARM(service-b-az1-unhealthy) AND 
  ALARM(service-c-az1-unhealthy)
RULE
# Single alert for "AZ-1 is having issues" instead of 3 alerts
```

---

## 3. Prometheus & Grafana on AWS

### When to Use Prometheus vs CloudWatch

```
┌────────────────────────────────────────────────────────────────┐
│  Use CloudWatch when:          │  Use Prometheus when:          │
├────────────────────────────────┼────────────────────────────────┤
│ AWS-native services (EC2, RDS) │ Kubernetes workloads           │
│ Low custom metric volume       │ High-cardinality metrics       │
│ Simple alerting needs          │ Complex alerting (AlertMgr)    │
│ Don't want to manage infra     │ Team knows PromQL              │
│ Small team, simple needs       │ Need Grafana dashboards        │
│ Lambda, serverless workloads   │ Multi-cloud/hybrid             │
│ Budget not sensitive to scale  │ Cost-sensitive at scale         │
└────────────────────────────────┴────────────────────────────────┘

AWS Managed Services:
├── AMP (Amazon Managed Prometheus): Prometheus-compatible storage
│   └── You don't run Prometheus server — AWS runs it
│   └── PromQL queries, remote_write ingestion
│
├── AMG (Amazon Managed Grafana): Managed Grafana instance
│   └── Pre-integrated with AMP, CloudWatch, X-Ray
│   └── SSO integration, workspace management
│
└── ADOT (AWS Distro for OpenTelemetry): Collection agent
    └── Replaces: Prometheus node_exporter + CloudWatch agent
    └── Single agent for metrics + traces + logs
```

### Prometheus Data Model

```
METRIC FORMAT:
metric_name{label1="value1", label2="value2"} value timestamp

Examples:
http_requests_total{method="GET", endpoint="/api/users", status="200"} 1547
http_request_duration_seconds_bucket{le="0.5", method="GET"} 129
node_cpu_seconds_total{cpu="0", mode="idle"} 35789.44

METRIC TYPES:
├── Counter: Only goes UP (requests_total, errors_total)
│   └── Use rate() to get per-second rate
│
├── Gauge: Goes up AND down (temperature, queue_size, memory_usage)
│   └── Use directly (current value matters)
│
├── Histogram: Distribution of values (request_duration)
│   └── Stores in buckets: _bucket{le="0.1"}, _bucket{le="0.5"}, etc.
│   └── Use histogram_quantile() for percentiles
│
└── Summary: Pre-computed percentiles (similar to histogram)
    └── Computed on client side (less flexible but accurate)
    └── Use for: When you need exact percentiles, small cardinality

CARDINALITY (THE #1 PERFORMANCE KILLER):
Cardinality = unique combinations of all labels

http_requests_total{method, endpoint, status, pod}
= 5 methods × 100 endpoints × 5 statuses × 200 pods
= 500,000 time series! (EXPENSIVE!)

Rule: Keep cardinality under 100K active series for good performance
Fix: Remove high-cardinality labels (pod, request_id, user_id)
```

### PromQL Essentials

```promql
-- INSTANT VECTOR (current values):
http_requests_total{job="api-server"}
-- Returns: current counter value for each matching series

-- RANGE VECTOR (values over time):
http_requests_total{job="api-server"}[5m]
-- Returns: all values in the last 5 minutes (used with functions)

-- RATE (per-second increase of counter):
rate(http_requests_total{job="api-server"}[5m])
-- "How many requests per second, averaged over 5 minutes"
-- ALWAYS use rate() with counters (raw counter value is useless!)

-- AGGREGATION:
sum(rate(http_requests_total[5m])) by (service)
-- Total request rate per service

avg(rate(http_requests_total[5m])) by (endpoint)
-- Average request rate per endpoint

-- PERCENTILES (from histogram):
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)
-- P99 latency per service

-- ALERTING EXPRESSION:
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)
> 0.01
-- "Error rate > 1% for any service"

-- PREDICT (will disk fill up?):
predict_linear(node_filesystem_avail_bytes[1h], 4*3600) < 0
-- "Based on last hour's trend, will disk be full in 4 hours?"
```

---

## 4. Distributed Tracing (AWS X-Ray)

### How Tracing Works

```
A single user request flows through multiple services:

User → API Gateway → Order Service → Payment Service → Database
                                    → Inventory Service → Cache

WITHOUT tracing:
├── Each service has its own logs
├── Correlating them manually: painful
├── "Why was this request slow?" → No answer

WITH tracing:
├── Each service creates a SPAN (start time, end time, metadata)
├── All spans share a TRACE ID (correlation key)
├── Parent-child relationships show the call chain
├── Duration of each span shows WHERE time was spent
└── Result: Visual timeline of the entire request

TRACE EXAMPLE:
Trace ID: 1-abc-123
├── Span: API Gateway (2ms)
│   └── Span: Order Service (5005ms)
│       ├── Span: Payment Service (4800ms) ← THIS IS SLOW!
│       │   ├── Span: Stripe API call (4500ms) ← External API!
│       │   └── Span: DB write (200ms)
│       ├── Span: Inventory Service (45ms)
│       └── Span: Cache lookup (1ms)
└── Total request time: 5007ms

Root cause: Stripe API was slow (4.5 seconds)
Time to diagnose: 30 seconds (vs 30 minutes without tracing)
```

### X-Ray Concepts

```
TRACE: End-to-end request journey (collection of segments)
SEGMENT: Work done by a single service (has subsegments)
SUBSEGMENT: Detailed breakdown within a segment

Example Segment (Order Service):
{
  "name": "order-service",
  "trace_id": "1-abc-123",
  "start_time": 1718443800.123,
  "end_time": 1718443805.128,
  "http": {
    "request": {"method": "POST", "url": "/v1/orders"},
    "response": {"status": 200}
  },
  "subsegments": [
    {
      "name": "payment-service",
      "type": "http",
      "start_time": 1718443800.200,
      "end_time": 1718443805.000
    },
    {
      "name": "DynamoDB",
      "type": "aws",
      "start_time": 1718443805.001,
      "end_time": 1718443805.050
    }
  ],
  "annotations": {
    "customer_id": "cust-456",   // Searchable!
    "order_value": 99.99          // Searchable!
  },
  "metadata": {
    "order_details": {...}        // Not searchable (detail only)
  }
}

KEY DISTINCTION:
├── Annotations: Key-value pairs that are INDEXED (searchable)
│   └── Use for: customer_id, order_id, environment
│   └── Limited to 50 per segment
│
└── Metadata: Any data, NOT indexed (only visible when viewing trace)
    └── Use for: Full request body, detailed context
    └── No limit (within segment size limit)
```

### Sampling Strategies

```
WHY sample?
├── Tracing 100% of requests is EXPENSIVE at scale
├── 10,000 requests/second × trace data = massive storage
├── Most requests are "normal" (don't need every one)
└── Errors and slow requests are the ones that matter

SAMPLING STRATEGIES:

1. Fixed Rate (Default):
   └── Trace 5% of all requests (1 in 20)
   └── Simple but may miss important requests

2. Reservoir + Rate:
   └── First N requests per second ALWAYS traced (reservoir)
   └── After reservoir: sample at fixed rate
   └── Example: First 1/sec + 5% of rest
   └── Guarantees minimum coverage even at low traffic

3. Rules-Based (X-Ray Sampling Rules):
   └── Different rates for different request types
   └── Example:
       ├── /health → 0% (never trace health checks)
       ├── /api/payments → 50% (important, trace more)
       └── /api/* → 5% (everything else, standard rate)

4. Tail-Based (OpenTelemetry Collector):
   └── Collect ALL traces temporarily
   └── After trace completes, DECIDE whether to keep it
   └── ALWAYS keep: errors, slow requests, specific users
   └── Drop: normal, fast, successful requests
   └── Best of both worlds but requires collector infrastructure
```

---

## 5. OpenTelemetry (OTEL) — The Universal Standard

### What OpenTelemetry Is

```
BEFORE OpenTelemetry:
├── CloudWatch SDK (only for AWS)
├── Prometheus client library (only for Prometheus)
├── Jaeger SDK (only for Jaeger tracing)
├── Datadog SDK (only for Datadog)
└── Problem: Vendor lock-in! Switching tools = rewrite instrumentation

AFTER OpenTelemetry:
┌──────────────────────────────────────────────────────────────┐
│  Application Code                                             │
│  └── OpenTelemetry SDK (ONE library, ALL signals)            │
│      ├── Metrics                                              │
│      ├── Traces                                               │
│      └── Logs                                                 │
└──────────────────────┬───────────────────────────────────────┘
                       │ OTLP (OpenTelemetry Protocol)
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  OTEL Collector (routes telemetry to ANY backend)            │
│  ├── Export to: Prometheus (metrics)                          │
│  ├── Export to: X-Ray (traces)                               │
│  ├── Export to: CloudWatch (logs)                            │
│  ├── Export to: Jaeger, Zipkin, Datadog, etc.               │
│  └── Switch backends WITHOUT changing application code!      │
└──────────────────────────────────────────────────────────────┘

KEY INSIGHT: Instrument ONCE with OTEL, export ANYWHERE
```

### ADOT (AWS Distro for OpenTelemetry)

```
ADOT = AWS's supported distribution of OpenTelemetry

What it replaces:
├── CloudWatch Agent (for custom metrics)
├── X-Ray SDK (for tracing)
├── Prometheus node_exporter (for infra metrics)
└── All three → ONE ADOT agent

Deployment Patterns:
├── Sidecar: One ADOT container per application pod
│   └── Good for: Kubernetes (EKS), fine-grained control
│
├── DaemonSet: One ADOT per node (shared by all pods)
│   └── Good for: Lower overhead, most production workloads
│
└── Standalone: Central collector cluster
    └── Good for: Advanced routing, tail-based sampling

Configuration:
┌────────────┐     ┌──────────────┐     ┌─────────────┐
│ Receivers  │────▶│ Processors   │────▶│ Exporters   │
│            │     │              │     │             │
│ - OTLP     │     │ - Batch      │     │ - AMP       │
│ - Prometheus│     │ - Filter     │     │ - X-Ray     │
│ - StatsD   │     │ - Sample     │     │ - CW Logs   │
│ - HostMetrics│   │ - Transform  │     │ - S3        │
└────────────┘     └──────────────┘     └─────────────┘
```

---

## 6. Alerting Strategy

### Alert Design Principles

```
THE GOLDEN RULES OF ALERTING:

1. EVERY alert must be ACTIONABLE
   └── If you can't do anything about it → don't alert
   └── "CPU is at 82%" → Is that actionable? Maybe not.
   └── "Error rate > 1%" → YES, someone should investigate

2. Alert on SYMPTOMS, not CAUSES
   └── Symptom: "Users seeing errors" (what matters)
   └── Cause: "CPU is high" (might not affect users)
   └── Why: CPU can be high without any user impact

3. Use MULTIPLE evaluation periods
   └── Single breach → could be a blip (false alarm)
   └── 3/5 breaches → probably real (reduce noise)

4. Set MEANINGFUL thresholds
   └── Don't alert at 80% CPU "just because"
   └── Alert when the METRIC that matters (latency, errors) degrades
   └── Use anomaly detection for dynamic thresholds

5. TIER your alerts
   └── P1 (Page): Customer-facing outage RIGHT NOW
   └── P2 (Page): Degraded performance, approaching failure
   └── P3 (Ticket): Not urgent, fix during business hours
   └── P4 (Log): Informational, review weekly
```

### Alert Routing

```
┌─────────────────────────────────────────────────────────────┐
│  Alert Classification → Action                               │
│                                                              │
│  "Is the customer impacted RIGHT NOW?"                      │
│  ├── YES + Widespread → P1 PAGE (wake people up!)           │
│  │   └── Route: PagerDuty → On-call SRE + Eng Lead         │
│  │                                                           │
│  ├── YES + Minor/Degraded → P2 PAGE                         │
│  │   └── Route: PagerDuty → On-call SRE                    │
│  │                                                           │
│  ├── NO + Will be soon → P3 TICKET                          │
│  │   └── Route: Jira → Sprint backlog                       │
│  │                                                           │
│  └── NO + Informational → P4 LOG                            │
│      └── Route: Slack #monitoring → Weekly review            │
│                                                              │
│  TIME-BASED ROUTING:                                         │
│  ├── Business hours: All levels → appropriate channel       │
│  ├── After hours: P1+P2 only → page on-call                │
│  └── Weekends: P1 only → page on-call                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Structured Logging

### Why Structured Logs Matter

```
UNSTRUCTURED LOG (bad for machines):
"2024-06-15 10:30:00 ERROR Payment failed for user 123 amount $99.99 timeout after 30s"

STRUCTURED LOG (queryable, parseable):
{
  "timestamp": "2024-06-15T10:30:00.123Z",
  "level": "ERROR",
  "service": "payment-api",
  "message": "Payment failed",
  "trace_id": "abc-123-def",
  "span_id": "001122334455",
  "customer_id": "user-123",
  "amount": 99.99,
  "error_type": "TimeoutException",
  "duration_ms": 30000,
  "downstream_service": "stripe-api",
  "environment": "production"
}

WHY structured is better:
├── QUERYABLE: filter by customer_id, error_type, etc.
├── PARSEABLE: Machines can aggregate, alert, dashboard
├── CORRELATED: trace_id links to distributed trace
├── CONSISTENT: Same format across all services
└── EFFICIENT: Logs Insights queries are fast with structured data
```

### Log Levels Guide

```
FATAL/CRITICAL: System is unusable, immediate attention needed
├── Database connection pool exhausted
├── Out of memory
└── Security breach detected

ERROR: Something failed, but system continues
├── Payment processing failed for one customer
├── API call to downstream service returned 500
└── Database query timeout

WARN: Something unexpected, might become a problem
├── Retry attempt 2/3 for external API
├── Disk usage at 80%
├── Deprecated API endpoint still receiving traffic

INFO: Normal operations, significant business events
├── Order created successfully
├── User logged in
├── Deployment completed

DEBUG: Detailed technical information (NEVER in production!)
├── SQL query: "SELECT * FROM users WHERE id = 123"
├── Request payload: {...}
├── Cache hit/miss details

TRACE: Very detailed (development only)
├── Function entry/exit
├── Variable values
└── Loop iterations
```

---

## 8. Embedded Metrics Format (EMF)

### CloudWatch EMF: Metrics from Logs (Cost-Effective)

```
PROBLEM: Custom metrics cost $0.30/metric/month
         With 10 dimensions × 100 values = 1000 metrics = $300/month!

SOLUTION: EMF = Write a specially-formatted LOG LINE
         CloudWatch automatically EXTRACTS metrics from it
         Cost: Just log ingestion ($0.50/GB) — typically MUCH cheaper!

HOW IT WORKS:
You print a JSON log with "_aws" metadata block
CloudWatch Logs recognizes it → extracts as a metric automatically

Example:
{
  "_aws": {
    "Timestamp": 1718443800000,
    "CloudWatchMetrics": [{
      "Namespace": "MyApp/PaymentService",
      "Dimensions": [["Endpoint", "StatusCode"]],
      "Metrics": [
        {"Name": "RequestLatency", "Unit": "Milliseconds"},
        {"Name": "RequestCount", "Unit": "Count"}
      ]
    }]
  },
  "Endpoint": "/v1/payments",
  "StatusCode": "200",
  "RequestLatency": 145,
  "RequestCount": 1
}

Result: CloudWatch metric "RequestLatency" with dimensions appears
        WITHOUT calling PutMetricData API (cheaper, faster!)

USE CASES:
├── High-cardinality metrics (per-endpoint, per-customer-tier)
├── Business metrics embedded in application logs
├── Lambda functions (avoid PutMetricData API call overhead)
└── Microservices with many unique dimension combinations
```

---

## 9. Cross-Account Observability

### Centralized Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Centralized Observability (Monitoring Account)                  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Monitoring Account (central)                                ││
│  │                                                              ││
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐   ││
│  │  │ CloudWatch │  │ AMP        │  │ Grafana (AMG)      │   ││
│  │  │ Cross-Acct │  │ Workspace  │  │                    │   ││
│  │  │ Dashboard  │  │ (metrics)  │  │ Data Sources:      │   ││
│  │  └────────────┘  └────────────┘  │ - AMP              │   ││
│  │                                    │ - CloudWatch (all) │   ││
│  │                                    │ - X-Ray            │   ││
│  │                                    └────────────────────┘   ││
│  └─────────────────────────────────────────────────────────────┘│
│       ▲                    ▲                                     │
│       │ OAM Link           │ Remote Write                        │
│       │                    │                                     │
│  ┌────┴────────────────────┴────────────────────────────────┐   │
│  │  Source Accounts (workload accounts)                      │   │
│  │                                                           │   │
│  │  Account A          Account B          Account C         │   │
│  │  ├── CW Metrics    ├── CW Metrics    ├── CW Metrics    │   │
│  │  ├── CW Logs       ├── CW Logs       ├── CW Logs       │   │
│  │  ├── X-Ray traces  ├── X-Ray traces  ├── X-Ray traces  │   │
│  │  └── ADOT agent    └── ADOT agent    └── ADOT agent    │   │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

Setup: CloudWatch OAM (Observability Access Manager)
├── Create SINK in monitoring account
├── Create LINK in each source account
├── Metrics, Logs, and Traces flow automatically
└── Single pane of glass across ALL accounts
```

---

## 10. Production Observability Checklist

```yaml
infrastructure_monitoring:
  - [ ] EC2/ECS: CPU, Memory, Disk, Network (auto from CloudWatch)
  - [ ] RDS: Connections, ReadLatency, WriteLatency, FreeableMemory
  - [ ] ALB: RequestCount, TargetResponseTime, HTTP 5xx, HealthyHosts
  - [ ] Lambda: Invocations, Errors, Duration, Throttles
  - [ ] S3: NumberOfObjects, BucketSizeBytes, 4xxErrors
  - [ ] ElastiCache: CacheHits, CacheMisses, Evictions, Memory

application_monitoring:
  - [ ] Request rate (requests/second per endpoint)
  - [ ] Error rate (% of 5xx responses)
  - [ ] Latency (P50, P95, P99 per endpoint)
  - [ ] Saturation (queue depth, active connections, thread pool)
  - [ ] Business metrics (orders/min, revenue, signups)

logging:
  - [ ] Structured JSON format across all services
  - [ ] Trace ID correlation in every log line
  - [ ] Retention policies set (7d dev, 30d staging, 90d prod)
  - [ ] Log level: INFO in prod (never DEBUG!)
  - [ ] Metric filters for key error patterns
  - [ ] Subscription filter to security account for audit events

tracing:
  - [ ] X-Ray or OTEL tracing enabled for all services
  - [ ] Sampling configured (not 100% in production)
  - [ ] Custom annotations on key attributes (customer_id, order_id)
  - [ ] Downstream calls instrumented (HTTP, DB, cache)
  - [ ] Service map available (visual dependency graph)

alerting:
  - [ ] SLO-based alerts (error budget burn rate)
  - [ ] Composite alarms (reduce noise)
  - [ ] Tiered routing (P1→page, P3→ticket)
  - [ ] Runbook URL in every alert
  - [ ] Alert noise < 5 alerts/night for on-call
  - [ ] Monthly alert quality review

dashboards:
  - [ ] Executive dashboard (overall health, SLOs)
  - [ ] Service dashboard (per service: 4 golden signals)
  - [ ] Infrastructure dashboard (compute, storage, network)
  - [ ] On-call dashboard (active incidents, recent deploys)
  - [ ] Cost dashboard (daily spend trend, anomalies)
```

---

## 11. Learning Path

```
Beginner (Week 1-2):
├── Enable CloudWatch metrics for EC2/RDS
├── Create a CloudWatch Alarm (CPU > 80%)
├── Write structured logs, query with Logs Insights
├── Create a simple dashboard (3-4 widgets)
├── Understand: metrics, logs, alarms, dashboards
└── Set up log retention policies

Intermediate (Week 3-4):
├── Create custom metrics (PutMetricData or EMF)
├── Build composite alarms
├── Implement metric filters (logs → metrics)
├── Enable X-Ray tracing for one service
├── Deploy ADOT collector on EKS
├── Set up Prometheus + Grafana (AMP/AMG)
├── Write PromQL queries
└── Cross-account observability (OAM)

Advanced (Week 5-8):
├── Design alerting strategy (SLO-based, tiered)
├── Implement distributed tracing across 5+ services
├── OpenTelemetry instrumentation (all three signals)
├── Tail-based sampling with OTEL Collector
├── Anomaly detection alarms
├── Cost optimization ($50K → $15K CW bill)
├── Synthetic canaries for proactive monitoring
└── Incident correlation (metrics ↔ logs ↔ traces)

Expert (Month 3+):
├── Design observability platform for 200+ services
├── Implement observability as code (Terraform)
├── Build internal monitoring standards
├── Prometheus cardinality management at scale
├── Custom OTEL instrumentation libraries
├── ML-based anomaly detection
└── FinOps for observability costs
```
