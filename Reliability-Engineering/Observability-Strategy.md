# Observability Strategy - Metrics, Logs, Traces, Dashboards

## Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────┐
│                   OBSERVABILITY STACK                         │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │   METRICS    │  │    LOGS      │  │    TRACES        │  │
│  │              │  │              │  │                  │  │
│  │ CloudWatch   │  │ CloudWatch   │  │ X-Ray            │  │
│  │ Prometheus   │  │ Logs         │  │ (Distributed     │  │
│  │ (via AMP)   │  │ OpenSearch   │  │  Tracing)        │  │
│  │              │  │              │  │                  │  │
│  │ What is      │  │ Why it       │  │ Where the       │  │
│  │ happening?   │  │ happened?    │  │ bottleneck is?  │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ DASHBOARDS & ALERTING                                 │   │
│  │ CloudWatch Dashboards + Grafana (AMG)                 │   │
│  │ → Real-time operational visibility                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Amazon Managed Prometheus (AMP) + Grafana (AMG)

### Architecture:
```
┌────────────────────────┐
│  EKS / ECS Cluster     │
│  ┌──────────────────┐  │
│  │ Application Pods │  │
│  │ /metrics endpoint│  │
│  └────────┬─────────┘  │
│           │             │
│  ┌────────▼─────────┐  │
│  │ ADOT Collector   │  │  (AWS Distro for OpenTelemetry)
│  │ (DaemonSet)      │  │
│  └────────┬─────────┘  │
└───────────┼─────────────┘
            │ Remote Write
            ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│ Amazon Managed          │────▶│ Amazon Managed          │
│ Prometheus (AMP)        │     │ Grafana (AMG)           │
│                         │     │                         │
│ - Multi-AZ storage      │     │ - Pre-built dashboards  │
│ - 150-day retention     │     │ - Alerting rules        │
│ - PromQL compatible     │     │ - Team workspaces       │
└─────────────────────────┘     └─────────────────────────┘
```

### ADOT (OpenTelemetry) Configuration:
```yaml
# ADOT Collector ConfigMap for EKS
apiVersion: v1
kind: ConfigMap
metadata:
  name: adot-collector-config
data:
  config.yaml: |
    receivers:
      prometheus:
        config:
          scrape_configs:
            - job_name: 'kubernetes-pods'
              kubernetes_sd_configs:
                - role: pod
              relabel_configs:
                - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
                  action: keep
                  regex: true
                - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
                  action: replace
                  target_label: __metrics_path__
                  regex: (.+)
      
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    
    processors:
      batch:
        timeout: 30s
        send_batch_size: 1000
      
      memory_limiter:
        limit_mib: 512
        spike_limit_mib: 128
        check_interval: 5s
    
    exporters:
      prometheusremotewrite:
        endpoint: https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-xxx/api/v1/remote_write
        auth:
          authenticator: sigv4auth
      
      awsxray:
        region: us-east-1
      
      awscloudwatchlogs:
        log_group_name: "/aws/ecs/application"
        log_stream_name: "otel-traces"
        region: us-east-1
    
    extensions:
      sigv4auth:
        region: us-east-1
        service: aps
    
    service:
      extensions: [sigv4auth]
      pipelines:
        metrics:
          receivers: [prometheus, otlp]
          processors: [batch, memory_limiter]
          exporters: [prometheusremotewrite]
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsxray]
```

## CloudWatch Embedded Metrics Format (EMF)

```python
# High-cardinality custom metrics without cost explosion
import json
import time

def emit_metric(metric_name, value, dimensions, unit="None"):
    """Emit CloudWatch metric via EMF (structured log that becomes a metric)."""
    metric_log = {
        "_aws": {
            "Timestamp": int(time.time() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "Application/PaymentService",
                "Dimensions": [list(dimensions.keys())],
                "Metrics": [{
                    "Name": metric_name,
                    "Unit": unit
                }]
            }]
        },
        metric_name: value,
        **dimensions
    }
    # Print as structured JSON → CloudWatch Logs → Auto-extracted as metric
    print(json.dumps(metric_log))

# Usage: High-cardinality metrics (per customer, per endpoint)
emit_metric(
    "RequestLatency",
    value=145.5,
    dimensions={
        "Service": "payment-api",
        "Endpoint": "/v1/charge",
        "StatusCode": "200",
        "Region": "us-east-1"
    },
    unit="Milliseconds"
)

emit_metric(
    "OrderValue",
    value=99.99,
    dimensions={
        "Service": "payment-api",
        "PaymentMethod": "credit_card",
        "Currency": "USD"
    },
    unit="None"
)
```

## Distributed Tracing with X-Ray

### Application Instrumentation:
```python
from aws_xray_sdk.core import xray_recorder, patch_all
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware
import boto3

# Patch all AWS SDK calls and HTTP libraries
patch_all()

# Custom subsegment for business logic
@xray_recorder.capture('process_payment')
def process_payment(payment_data):
    # X-Ray captures timing automatically
    
    # Add annotations (searchable in X-Ray console)
    xray_recorder.current_subsegment().put_annotation('customer_id', payment_data['customer_id'])
    xray_recorder.current_subsegment().put_annotation('amount', payment_data['amount'])
    
    # Add metadata (not searchable, but visible in trace)
    xray_recorder.current_subsegment().put_metadata('payment_details', payment_data)
    
    # Call downstream (automatically traced)
    result = payment_gateway.charge(payment_data)
    
    return result

# X-Ray Groups for filtering
# Create group for slow traces
xray_client = boto3.client('xray')
xray_client.create_group(
    GroupName='SlowPayments',
    FilterExpression='service("payment-api") AND responsetime > 2'
)

# X-Ray Insights (anomaly detection)
xray_client.create_group(
    GroupName='PaymentErrors',
    FilterExpression='service("payment-api") AND fault = true',
    InsightsConfiguration={
        'InsightsEnabled': True,
        'NotificationsEnabled': True
    }
)
```

## Structured Logging Strategy

### Log Format Standard:
```python
import json
import logging
import time
from contextvars import ContextVar

# Request context (propagated through async calls)
request_id_var: ContextVar[str] = ContextVar('request_id', default='unknown')
trace_id_var: ContextVar[str] = ContextVar('trace_id', default='unknown')

class StructuredLogger:
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.logger = logging.getLogger(service_name)
    
    def _format(self, level: str, message: str, **kwargs):
        log_entry = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%S.000Z", time.gmtime()),
            "level": level,
            "service": self.service_name,
            "message": message,
            "request_id": request_id_var.get(),
            "trace_id": trace_id_var.get(),
            **kwargs
        }
        return json.dumps(log_entry)
    
    def info(self, message: str, **kwargs):
        self.logger.info(self._format("INFO", message, **kwargs))
    
    def error(self, message: str, error: Exception = None, **kwargs):
        extra = {}
        if error:
            extra["error_type"] = type(error).__name__
            extra["error_message"] = str(error)
            extra["stack_trace"] = traceback.format_exc()
        self.logger.error(self._format("ERROR", message, **extra, **kwargs))

logger = StructuredLogger("payment-service")

# Usage:
logger.info("Payment processed", 
    customer_id="cust-123",
    amount=99.99,
    payment_method="credit_card",
    processing_time_ms=145
)

# Output (to CloudWatch Logs):
# {"timestamp":"2024-06-15T10:30:00.000Z","level":"INFO","service":"payment-service",
#  "message":"Payment processed","request_id":"req-abc","trace_id":"1-xxx",
#  "customer_id":"cust-123","amount":99.99,"payment_method":"credit_card",
#  "processing_time_ms":145}
```

### CloudWatch Logs Insights Queries:
```sql
-- Find slow requests
fields @timestamp, @message
| filter service = "payment-service"
| filter processing_time_ms > 1000
| sort processing_time_ms desc
| limit 50

-- Error rate by endpoint
fields @timestamp, @message
| filter level = "ERROR"
| stats count(*) as error_count by endpoint
| sort error_count desc

-- P95 latency per service
fields @timestamp, processing_time_ms, service
| stats percentile(processing_time_ms, 95) as p95,
        percentile(processing_time_ms, 99) as p99,
        avg(processing_time_ms) as avg_latency
  by bin(5m), service

-- Trace slow request through microservices
fields @timestamp, service, message, processing_time_ms
| filter trace_id = "1-xxx-yyy"
| sort @timestamp asc
```

## Cross-Account Observability

### CloudWatch Cross-Account Observability:
```hcl
# Monitoring Account (centralized)
resource "aws_oam_sink" "central" {
  name = "central-observability-sink"
  
  tags = {
    Environment = "monitoring"
  }
}

resource "aws_oam_sink_policy" "allow_org" {
  sink_identifier = aws_oam_sink.central.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::*:root" }
      Action    = ["oam:CreateLink", "oam:UpdateLink"]
      Resource  = "*"
      Condition = {
        "ForAnyValue:StringEquals" = {
          "aws:PrincipalOrgID" = var.org_id
        }
      }
    }]
  })
}

# Source Account (workload)
resource "aws_oam_link" "source" {
  label_template  = "$AccountName"
  resource_types  = ["AWS::CloudWatch::Metric", "AWS::Logs::LogGroup", "AWS::XRay::Trace"]
  sink_identifier = var.monitoring_sink_arn
}
```

## Operational Dashboards

### Executive Dashboard:
```hcl
resource "aws_cloudwatch_dashboard" "executive" {
  dashboard_name = "Executive-Platform-Health"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          title  = "Overall Platform Availability"
          view   = "singleValue"
          metrics = [
            [{
              expression = "100 - (m1/m2*100)"
              label      = "Availability %"
              id         = "availability"
            }],
            ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "LoadBalancer", "app/main/xxx", { id = "m1", visible = false }],
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/main/xxx", { id = "m2", visible = false }]
          ]
          period = 86400  # 24 hour view
        }
      },
      {
        type = "metric"
        properties = {
          title  = "Error Budget Remaining"
          view   = "gauge"
          yAxis  = { left = { min = 0, max = 100 } }
          metrics = [
            [{
              expression = "((0.0005 * 43200) - m1) / (0.0005 * 43200) * 100"
              label      = "Budget %"
              id         = "budget"
            }],
            ["Custom/SRE", "ErrorMinutes", "Service", "platform", { id = "m1", visible = false, stat = "Sum", period = 2592000 }]
          ]
        }
      },
      {
        type = "metric"
        properties = {
          title  = "P99 Latency Trend"
          view   = "timeSeries"
          metrics = [
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/main/xxx", { stat = "p99" }]
          ]
          period = 300
          annotations = {
            horizontal = [{ value = 0.5, label = "SLO (500ms)" }]
          }
        }
      }
    ]
  })
}
```

## Alerting Decision Tree

```
Is the customer impacted RIGHT NOW?
├── YES → Is it a widespread outage?
│   ├── YES → P1 PAGE (SRE + Engineering Lead + Comms)
│   └── NO  → Is it degraded performance?
│       ├── YES → P2 PAGE (On-call SRE)
│       └── NO  → P3 TICKET (next business day)
└── NO → Will it become customer-impacting soon?
    ├── YES → P3 TICKET (proactive fix)
    └── NO  → LOG (visibility only)
```

### Alert Runbook Template:
```markdown
## Alert: P1-payment-api-availability-critical

### What
Payment API availability dropped below 99.5% SLO

### Impact
Users cannot complete purchases. Revenue loss: ~$X per minute.

### Immediate Actions
1. Check ALB target health: `aws elbv2 describe-target-health --target-group-arn $TG_ARN`
2. Check ECS service events: `aws ecs describe-services --cluster prod --services payment-api`
3. Check recent deployments: `aws ecs describe-services --cluster prod --services payment-api | jq '.services[0].deployments'`
4. Check RDS status: `aws rds describe-db-clusters --db-cluster-identifier payment-db`

### Rollback
If caused by recent deployment:
```bash
aws ecs update-service --cluster prod --service payment-api \
  --task-definition payment-api:PREVIOUS_VERSION --force-new-deployment
```

### Escalation
- After 15 min: Escalate to Engineering Lead
- After 30 min: Escalate to VP Engineering
- After 60 min: Executive communication required
```
