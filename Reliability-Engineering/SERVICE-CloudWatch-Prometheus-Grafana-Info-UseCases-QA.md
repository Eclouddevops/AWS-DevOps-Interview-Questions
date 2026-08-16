# CloudWatch & Prometheus/Grafana — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | CloudWatch | Prometheus (AMP) | Grafana (AMG) |
|-----------|-----------|-----------------|---------------|
| **Type** | Metrics + Logs + Alarms | Time-series metrics DB | Visualization + Alerting |
| **Managed** | Fully (AWS) | Fully (AMP) | Fully (AMG) |
| **Query Language** | CloudWatch Metrics Insights | PromQL | N/A (dashboards) |
| **Pricing** | Per-metric + per-GB logs | Per-sample ingested | Per-user/month |
| **Retention** | 15 months (metrics) | 150 days (AMP) | N/A |
| **Best for** | AWS-native services | Kubernetes/app metrics | Unified visualization |
| **Multi-cloud** | AWS only | Yes (open standard) | Yes (data source agnostic) |

---

## 🎯 Use Cases

### CloudWatch
1. **AWS service monitoring** — EC2, RDS, ALB metrics (auto, free)
2. **Log aggregation** — Centralized logs from all services
3. **Alarming** — Threshold + anomaly detection + composite alarms
4. **Dashboards** — Operational visibility (cross-account)
5. **Synthetics** — Proactive endpoint monitoring (canaries)
6. **Contributor Insights** — Top-N analysis (highest error producers)

### Prometheus + Grafana (AMP + AMG)
1. **Kubernetes observability** — Pod/container/service metrics
2. **High-cardinality metrics** — Per-endpoint, per-customer metrics
3. **SLO tracking** — Error budgets with PromQL precision
4. **Multi-source dashboards** — CW + Prometheus + X-Ray in one view
5. **Advanced alerting** — AlertManager (grouping, inhibition, routing)
6. **Application metrics** — Custom business metrics (orders/sec, revenue)

---

## ❓ Interview Questions & Answers

### Q1: When should you use CloudWatch vs Prometheus for monitoring?

**Answer:**

```
USE CLOUDWATCH:                          USE PROMETHEUS:
├── AWS service metrics (EC2, RDS, ALB)  ├── Kubernetes workloads
├── Simple alerting needs                ├── High-cardinality app metrics
├── Don't want to manage infra           ├── Complex alerting (AlertManager)
├── Lambda/serverless monitoring         ├── PromQL queries needed
├── Small team (< 10 engineers)          ├── Multi-cloud/hybrid
├── Budget not sensitive to volume       ├── Large scale (cost at volume)
└── Native AWS integration preferred     └── Grafana dashboards desired

COST COMPARISON (at scale):
├── 100K custom CW metrics: $30,000/month (ouch!)
├── 100K Prometheus series (AMP): ~$3,000/month
├── Savings: 90% at high cardinality!
└── But CW is FREE for AWS service metrics (EC2, RDS, etc.)

BEST PRACTICE: HYBRID
├── CloudWatch for: AWS service metrics (free), logs, basic alarms
├── Prometheus for: App metrics, Kubernetes, high-cardinality, SLOs
├── Grafana: Unified dashboards querying BOTH (single pane of glass)
└── ADOT Collector: Single agent for all telemetry routing
```

### Q2: Explain CloudWatch Embedded Metrics Format (EMF). Why use it instead of PutMetricData?

**Answer:**

```python
# PutMetricData API:
# ├── Explicit API call ($0.01 per 1000 calls)
# ├── Rate limited (150 calls/sec by default)
# ├── Separate from logging (two code paths)
# └── Fixed cost per metric ($0.30/metric/month)

# EMF (Embedded Metrics Format):
# ├── Just a specially-formatted LOG LINE
# ├── CloudWatch auto-extracts metrics from it
# ├── Cost: Only log ingestion ($0.50/GB) — often CHEAPER!
# ├── No API rate limits (just log writes)
# └── Metrics appear in CloudWatch automatically

import json

def emit_emf_metric(endpoint, status_code, latency_ms):
    """Print a log line that CloudWatch converts to a metric."""
    print(json.dumps({
        "_aws": {
            "Timestamp": int(time.time() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "MyApp",
                "Dimensions": [["Endpoint", "StatusCode"]],
                "Metrics": [
                    {"Name": "Latency", "Unit": "Milliseconds"},
                    {"Name": "RequestCount", "Unit": "Count"}
                ]
            }]
        },
        "Endpoint": endpoint,
        "StatusCode": str(status_code),
        "Latency": latency_ms,
        "RequestCount": 1
    }))

# WHY better:
# ├── High cardinality is CHEAP (log cost, not $0.30/metric)
# ├── Per-endpoint + per-status metrics without custom metric cost explosion
# ├── Works in Lambda perfectly (no API call overhead)
# └── Data is BOTH a log AND a metric (query both ways!)
```

### Q3: How do you implement SLO-based alerting with Prometheus?

**Answer:**

```yaml
# Multi-window, multi-burn-rate SLO alerting:
# SLO: 99.9% availability (error budget = 0.1%)

groups:
  - name: slo_burn_rate
    rules:
      # FAST BURN: Budget gone in 2 hours → PAGE immediately
      - alert: HighBurnRate_Fast
        expr: |
          (
            sum(rate(http_requests_total{code=~"5.."}[5m]))
            / sum(rate(http_requests_total[5m]))
          ) > (14.4 * 0.001)
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "Error budget burning 14.4x → exhausted in 2 hours!"

      # SLOW BURN: Budget gone in 10 days → ticket
      - alert: HighBurnRate_Slow
        expr: |
          (
            sum(rate(http_requests_total{code=~"5.."}[1h]))
            / sum(rate(http_requests_total[1h]))
          ) > (3 * 0.001)
        for: 1h
        labels: { severity: warning }

# WHY multi-window:
# ├── Short window (5m): Catches sudden spikes FAST
# ├── Long window (1h): Confirms it's sustained (not just a blip)
# ├── Both must be true → very few false positives!
# └── Different severities = appropriate response
```

### Q4: Your CloudWatch bill jumped from $5K to $40K. How do you diagnose?

**Answer:**

```bash
# Step 1: Identify cost category
aws ce get-cost-and-usage --granularity DAILY --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["AmazonCloudWatch"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# COMMON CULPRITS:
# ├── CW:MetricMonitorUsage → Custom metrics explosion
# │   Fix: Use EMF, reduce cardinality, remove unused metrics
# ├── CW:DataProcessing-Bytes → Log ingestion spike
# │   Fix: Set retention, filter noise, reduce log level
# ├── CW:GMD-Metrics → Dashboard API calls (auto-refresh too fast)
# │   Fix: Set refresh to 5 min (not 10 seconds!)
# └── CW:DataStorage-Bytes → No retention set (logs stored forever!)
#     Fix: Set retention policy on ALL log groups

# QUICK WINS:
# 1. Set log retention on all groups (7d dev, 30d staging, 90d prod)
# 2. Remove high-cardinality custom metrics (request_id, user_id labels)
# 3. Use Metric Filters instead of PutMetricData (free from logs!)
# 4. VPC Endpoints for S3/CW (eliminate NAT transfer costs)
# 5. Dashboard refresh interval: 5 min (not 10 sec)
```

### Q5: How does ADOT (AWS Distro for OpenTelemetry) simplify observability?

**Answer:**

```
BEFORE ADOT: Multiple agents needed
├── CloudWatch Agent (for metrics + logs to CW)
├── X-Ray SDK (for tracing to X-Ray)  
├── Prometheus exporter (for metrics to Prometheus)
└── Custom code for each backend

AFTER ADOT: One agent, multiple destinations
┌──────────────────────────────────────────────┐
│  Application → OTEL SDK → ADOT Collector     │
│                               │               │
│                    ┌──────────┼──────────┐   │
│                    ▼          ▼          ▼   │
│               CloudWatch  Prometheus   X-Ray │
│               (logs+metrics) (AMP)    (traces)│
└──────────────────────────────────────────────┘

BENEFIT:
├── Instrument ONCE (OpenTelemetry SDK)
├── Route to ANY backend (change config, not code!)
├── Vendor-neutral (switch from Datadog to AMP = config change)
├── All three signals (metrics + traces + logs) from ONE SDK
└── Future-proof (OTEL is industry standard)
```

### Q6: Explain Prometheus cardinality. Why is it the #1 performance/cost killer?

**Answer:**

```
CARDINALITY = unique combinations of metric name + ALL label values

Example:
http_requests_total{method="GET", path="/users/123", pod="pod-abc"}

If you have:
├── 5 methods × 10,000 paths × 200 pods = 10,000,000 time series!
├── Each series = ~2 bytes/sample × 15 sec interval × 86400 sec/day
├── = ~11.5 KB/day/series × 10M = 115 GB/day of metric data!
└── AMP cost: 10M × 8640 samples/day × 30 days × $0.003/10K = $77,760/month!

FIX:
├── Path labels: Normalize! /users/123 → /users/{id} (bounded)
├── Pod labels: DROP! Aggregate at service level (not pod)
├── Use recording rules: Pre-aggregate, store summary only
└── Target: < 100K active series (manageable, fast, affordable)

# Relabeling (drop high-cardinality before ingestion):
metric_relabel_configs:
  - source_labels: [path]
    regex: '/users/[0-9a-f-]+'
    replacement: '/users/{id}'
    target_label: path
  - action: labeldrop
    regex: '(pod|pod_template_hash)'  # Remove pod-level granularity
```

### Q7: How do you set up cross-account observability in AWS?

**Answer:**

```
ARCHITECTURE:
┌────────────────────────────────────────────────────┐
│  Monitoring Account (central)                       │
│  ├── CloudWatch OAM Sink (receives from all)      │
│  ├── AMP Workspace (all Prometheus metrics)        │
│  ├── Grafana (AMG) — unified dashboards           │
│  └── X-Ray traces from all accounts               │
└─────────────────────┬──────────────────────────────┘
                      │ OAM Links
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
   Account A     Account B     Account C
   (source)      (source)      (source)

SETUP (CloudWatch OAM):
1. Monitoring account: Create OAM Sink
2. Source accounts: Create OAM Link → connects to Sink
3. Result: Monitoring account sees ALL metrics/logs/traces

# Terraform:
resource "aws_oam_sink" "central" { name = "central-sink" }  # Monitoring acct
resource "aws_oam_link" "source" {                            # Each source acct
  label_template = "$AccountName"
  resource_types = ["AWS::CloudWatch::Metric", "AWS::Logs::LogGroup", "AWS::XRay::Trace"]
  sink_identifier = var.monitoring_sink_arn
}
```

### Q8: Compare CloudWatch Alarms, Composite Alarms, and Anomaly Detection.

**Answer:**

```
STANDARD ALARM:
├── Single metric threshold
├── "Alert when CPU > 80% for 5 minutes"
├── Simple but generates noise (brief spikes trigger it)
└── Use: Basic health monitoring

COMPOSITE ALARM:
├── Combines multiple alarms with logic (AND/OR)
├── "Alert when CPU > 80% AND Error Rate > 1%"
├── Reduces false positives dramatically
├── Use: Real incidents require MULTIPLE signals
├── Example: AZ failure = service_A_down AND service_B_down AND service_C_down

ANOMALY DETECTION:
├── ML-based dynamic thresholds (learns patterns)
├── "Alert when metric is 3 standard deviations from normal"
├── Adapts to: daily patterns, weekly cycles, seasonal trends
├── No manual threshold setting needed
├── Use: Metrics with variable baselines (traffic, latency)
├── Example: Traffic at 2 AM is normally low — don't alert
│            Traffic at 2 PM is normally high — alert if it drops!

RECOMMENDATION:
├── P1 alerts: Composite (multiple signals confirm real incident)
├── P2 alerts: Standard with smart thresholds (3 of 5 datapoints)
├── P3 alerts: Anomaly detection (catches unusual but non-critical)
└── NEVER: Alert on single metric breach for single period (too noisy!)
```

---

## 🏆 Key Takeaways

```
CloudWatch:
├── FREE for AWS service metrics (use them!)
├── EMF for custom metrics (cheaper than PutMetricData at scale)
├── Composite Alarms reduce noise (AND/OR logic)
├── Log Insights for searching (SQL-like, fast)
├── Set retention on ALL log groups (prevent cost explosion!)
└── Cross-Account OAM for centralized monitoring

Prometheus/Grafana:
├── PromQL is industry-standard (learn it well!)
├── Cardinality is enemy #1 (normalize labels, drop high-card)
├── Recording rules for performance (pre-compute expensive queries)
├── AMP = zero-ops Prometheus (AWS manages storage + HA)
├── AMG = zero-ops Grafana (pre-integrated with CW, AMP, X-Ray)
└── AlertManager for advanced routing (grouping, inhibition, silence)

Together (Hybrid — Best Practice):
├── ADOT: Single collector for all telemetry
├── CloudWatch: AWS metrics + logs
├── AMP: Application metrics + Kubernetes
├── X-Ray: Distributed tracing
├── AMG: Single pane of glass (queries all sources)
└── All correlated via trace_id (logs ↔ metrics ↔ traces)
```
