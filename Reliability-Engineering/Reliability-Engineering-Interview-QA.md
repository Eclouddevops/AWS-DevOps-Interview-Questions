# Reliability Engineering - Tricky Production-Based Interview Questions & Answers

## Observability, SLAs/SLOs, Disaster Recovery, Resiliency

### Q1: Your production application's P99 latency spiked from 200ms to 2.5s during peak hours. CloudWatch shows no CPU/memory issues on EC2 instances. How do you systematically diagnose this?

**Answer:**

**Systematic Diagnosis Framework (USE + RED Method):**

**Step 1: Verify the signal is real**
```bash
# Check if it's a metric collection issue vs real latency
# Look at ALB metrics (independent of app instrumentation)
aws cloudwatch get-metric-data --metric-data-queries '[
  {
    "Id": "p99_latency",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/ApplicationELB",
        "MetricName": "TargetResponseTime",
        "Dimensions": [{"Name":"LoadBalancer","Value":"app/prod-alb/xxx"}]
      },
      "Period": 60,
      "Stat": "p99"
    }
  }
]' --start-time 2024-06-15T10:00:00Z --end-time 2024-06-15T12:00:00Z
```

**Step 2: Identify WHERE the latency is (decompose the request path):**
```
Client → CloudFront → ALB → Target Group → EC2/ECS → Database/Cache → External APIs
   5ms      20ms       2ms      ?ms          ?ms           ?ms            ?ms
```

Check at each layer:
- **ALB:** `TargetResponseTime` vs `RequestCount` (is it all requests or specific paths?)
- **Target Group:** `HealthyHostCount`, `UnHealthyHostCount`, connection errors
- **Application:** X-Ray traces (distributed tracing shows exact bottleneck)
- **Database:** RDS `ReadLatency`, `WriteLatency`, `DatabaseConnections`
- **Cache:** ElastiCache `CurrConnections`, `CacheMisses`, `Evictions`
- **External:** X-Ray subsegment latency for downstream calls

**Step 3: Common hidden culprits (not CPU/memory):**

1. **Connection pool exhaustion:**
```python
# Check application connection pools
# Symptom: Threads waiting for DB connection
# CloudWatch won't show CPU spike - threads are WAITING, not computing

# Fix: Increase pool size or add connection timeout
DATABASE_URL = "postgresql://user:pass@rds-host/db?pool_size=50&pool_timeout=10"
```

2. **DNS resolution timeout:**
```bash
# Check Route 53 Resolver metrics
# If DNS is timing out, every new connection adds 5+ seconds
dig +trace your-service.internal.company.com
```

3. **Garbage Collection pauses (JVM):**
```bash
# Check GC logs even if CPU looks fine (GC pauses don't show as CPU usage)
# Enable JVM GC logging:
-XX:+UseG1GC -XX:MaxGCPauseMillis=100 -Xlog:gc*:file=/var/log/gc.log
```

4. **Noisy Neighbor (EBS throttling):**
```bash
# gp2/gp3 volumes have burst credit limits
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeQueueLength \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --statistics Average --period 60
# If QueueLength > 1, I/O is queuing (throttled)
```

5. **NAT Gateway throughput limit:**
```bash
# NAT GW: 45 Gbps per AZ, but 55,000 simultaneous connections limit
# Check: ErrorPortAllocation metric
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name ErrorPortAllocation \
  --dimensions Name=NatGatewayId,Value=nat-xxx
```

**Step 4: Use X-Ray for root cause:**
```python
# X-Ray trace showing actual bottleneck
# Trace map reveals: App → ElastiCache (2400ms!)
# Root cause: ElastiCache evictions causing cache misses
# → Every request hitting RDS instead of cache
# → RDS connection pool saturated → cascade failure

# Fix: Increase ElastiCache node size + implement circuit breaker
```

---

### Q2: Define SLIs, SLOs, and SLAs for a payment processing API. How would you implement error budgets and what happens when the budget is exhausted?

**Answer:**

**SLI/SLO/SLA Framework:**

```
┌─────────────────────────────────────────────────────────────┐
│  Service: Payment Processing API                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  SLI (Service Level Indicator) - What we MEASURE:           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Availability: % of requests returning non-5xx          │ │
│  │ Latency: % of requests completing < threshold          │ │
│  │ Correctness: % of transactions processed accurately    │ │
│  │ Throughput: Requests/sec handled without degradation    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  SLO (Service Level Objective) - What we TARGET:            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Availability: 99.95% (measured over 30-day window)     │ │
│  │ Latency P50: < 100ms (99% of requests)                │ │
│  │ Latency P99: < 500ms (99.9% of requests)              │ │
│  │ Correctness: 99.999% (financial accuracy)              │ │
│  │ Throughput: 10,000 TPS sustained                       │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  SLA (Service Level Agreement) - What we GUARANTEE:         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Availability: 99.9% (contractual)                      │ │
│  │ Latency P99: < 1000ms                                  │ │
│  │ Financial penalty: 10% credit per 0.1% below SLA      │ │
│  │ Note: SLO is TIGHTER than SLA (buffer for safety)      │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Error Budget Calculation:**

```python
# Error Budget = 1 - SLO
# For 99.95% availability SLO over 30 days:

error_budget_percentage = 1 - 0.9995  # = 0.05%
minutes_in_30_days = 30 * 24 * 60     # = 43,200 minutes
error_budget_minutes = minutes_in_30_days * error_budget_percentage  # = 21.6 minutes

# Per month: 21.6 minutes of allowed downtime
# Per day: ~43 seconds of allowed downtime (averaged)
```

**Implementation with CloudWatch:**

```hcl
# CloudWatch composite alarm for SLO tracking
resource "aws_cloudwatch_metric_alarm" "slo_availability" {
  alarm_name          = "payment-api-slo-availability-breach"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 5
  threshold           = 99.95
  alarm_description   = "Payment API availability SLO breach"
  
  metric_query {
    id          = "availability"
    expression  = "100 - (errors / total * 100)"
    label       = "Availability %"
    return_data = true
  }
  
  metric_query {
    id = "errors"
    metric {
      metric_name = "HTTPCode_Target_5XX_Count"
      namespace   = "AWS/ApplicationELB"
      period      = 300
      stat        = "Sum"
      dimensions  = { LoadBalancer = "app/payment-api-alb/xxx" }
    }
  }
  
  metric_query {
    id = "total"
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 300
      stat        = "Sum"
      dimensions  = { LoadBalancer = "app/payment-api-alb/xxx" }
    }
  }
  
  alarm_actions = [aws_sns_topic.sre_oncall.arn]
}

# Error budget burn rate alarm (fast burn = urgent)
resource "aws_cloudwatch_metric_alarm" "error_budget_fast_burn" {
  alarm_name          = "payment-api-error-budget-fast-burn"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  threshold           = 14.4  # 14.4x burn rate = budget exhausted in 2 hours
  
  metric_query {
    id          = "burn_rate"
    expression  = "(1 - (success_1h / total_1h)) / (1 - 0.9995) "
    label       = "Error Budget Burn Rate"
    return_data = true
  }
  # ... metric queries for success_1h and total_1h
  
  alarm_actions = [aws_sns_topic.sre_page.arn]  # Page immediately
}
```

**Error Budget Policy (What happens when exhausted):**

```yaml
# Error Budget Policy Document
error_budget_policy:
  service: payment-processing-api
  slo: 99.95% availability (30-day rolling)
  
  actions_by_budget_remaining:
    budget_100_to_50_percent:
      # Business as usual
      - Normal feature velocity
      - Standard deployment cadence
      - Regular maintenance windows
    
    budget_50_to_25_percent:
      # Caution mode
      - Reduce deployment frequency (daily → twice weekly)
      - Require SRE review for all production changes
      - Prioritize reliability work in sprint
      - Cancel non-essential maintenance windows
    
    budget_25_to_0_percent:
      # Reliability mode
      - Feature freeze (no new features deployed)
      - Only reliability improvements and critical bug fixes
      - Mandatory rollback plan for every change
      - Increase monitoring and alerting sensitivity
    
    budget_exhausted:
      # Emergency mode
      - Complete production change freeze
      - All engineering effort on reliability
      - Incident review for every error budget spend
      - Executive escalation
      - Resume features only after budget recovers above 25%
```

---

### Q3: Your multi-region active-active application uses DynamoDB Global Tables. During a regional failover, you notice data inconsistencies. How do you handle this?

**Answer:**

**Understanding DynamoDB Global Tables Consistency:**

```
┌──────────────────────┐          ┌──────────────────────┐
│  us-east-1 (Region A)│          │  eu-west-1 (Region B)│
│                      │          │                      │
│  DynamoDB Table      │◄────────▶│  DynamoDB Table      │
│  (Replica)           │  Async   │  (Replica)           │
│                      │  Repl.   │                      │
│  Last Writer Wins    │  ~1 sec  │  Last Writer Wins    │
└──────────────────────┘          └──────────────────────┘

Conflict Resolution: Last Writer Wins (LWW)
Replication Lag: Usually < 1 second, but CAN be seconds during:
- Regional events
- High write throughput
- Network partition
```

**Inconsistency Scenarios:**

1. **Split-brain during network partition:**
   - User updates item in us-east-1 AND eu-west-1 simultaneously
   - When partition heals, LWW picks one (potentially wrong one)

2. **Failover during replication lag:**
   - Write to us-east-1 at T=0
   - us-east-1 fails at T=0.5s
   - Failover to eu-west-1 which hasn't received the write yet
   - Write appears "lost"

**Solutions:**

**Solution 1: Conflict-free data modeling:**
```python
# Instead of overwriting values, use APPEND/ADD operations
# These are commutative and order-independent

# BAD: Last writer wins (data loss risk)
table.update_item(
    Key={'order_id': '123'},
    UpdateExpression='SET #status = :status',
    ExpressionAttributeValues={':status': 'shipped'}
)

# GOOD: Append events (conflict-free)
table.update_item(
    Key={'order_id': '123'},
    UpdateExpression='SET #events = list_append(if_not_exists(#events, :empty), :new_event)',
    ExpressionAttributeValues={
        ':new_event': [{'status': 'shipped', 'timestamp': '2024-06-15T10:00:00Z', 'region': 'us-east-1'}],
        ':empty': []
    }
)
# Application reads all events and derives current state
```

**Solution 2: Region-aware writes with version vectors:**
```python
import time
import uuid

def update_with_conflict_detection(table, key, updates, region):
    """Update with region-aware versioning."""
    version_key = f"version_{region}"
    
    try:
        response = table.update_item(
            Key=key,
            UpdateExpression=f'SET {updates}, #{version_key} = :new_version, #last_modified = :now',
            ConditionExpression=f'attribute_not_exists(#{version_key}) OR #{version_key} < :new_version',
            ExpressionAttributeNames={
                f'#{version_key}': version_key,
                '#last_modified': 'last_modified'
            },
            ExpressionAttributeValues={
                ':new_version': int(time.time() * 1000),
                ':now': datetime.utcnow().isoformat()
            },
            ReturnValues='ALL_NEW'
        )
        return response
    except table.meta.client.exceptions.ConditionalCheckFailedException:
        # Conflict detected - implement merge logic
        return handle_conflict(table, key, updates, region)
```

**Solution 3: Reconciliation pipeline:**
```python
# Lambda triggered by DynamoDB Streams in BOTH regions
# Detects and resolves conflicts post-replication

def reconcile_handler(event, context):
    for record in event['Records']:
        if record['eventName'] in ['MODIFY', 'INSERT']:
            new_image = record['dynamodb']['NewImage']
            
            # Check for conflict markers
            if has_conflict_marker(new_image):
                resolve_conflict(
                    item_key=record['dynamodb']['Keys'],
                    region_a_version=get_region_version(new_image, 'us-east-1'),
                    region_b_version=get_region_version(new_image, 'eu-west-1'),
                    resolution_strategy='business_rules'  # Not just LWW
                )

def resolve_conflict(item_key, region_a_version, region_b_version, resolution_strategy):
    """Apply business-specific conflict resolution."""
    if resolution_strategy == 'business_rules':
        # Example: For order status, use state machine priority
        status_priority = {'cancelled': 4, 'shipped': 3, 'confirmed': 2, 'pending': 1}
        
        winner = max([region_a_version, region_b_version], 
                     key=lambda v: status_priority.get(v.get('status', ''), 0))
        
        # Write resolved version to both regions
        write_resolved_item(item_key, winner)
```

---

### Q4: Design a disaster recovery strategy for a mission-critical application with RPO < 5 minutes and RTO < 15 minutes across AWS regions.

**Answer:**

**DR Strategy: Warm Standby with Automated Failover**

```
┌────────────────────────────────────────────────────────────────────┐
│                     DR Architecture                                  │
│                                                                      │
│  ┌──────────────────────────┐    ┌──────────────────────────┐      │
│  │   PRIMARY (us-east-1)    │    │   SECONDARY (us-west-2)   │      │
│  │                          │    │                           │      │
│  │  Route 53 (Active)       │    │  Route 53 (Standby)      │      │
│  │  ┌────────────────────┐  │    │  ┌────────────────────┐  │      │
│  │  │ ALB (full scale)   │  │    │  │ ALB (min scale)    │  │      │
│  │  │ ECS: 10 tasks      │  │    │  │ ECS: 2 tasks       │  │      │
│  │  │ RDS: Multi-AZ      │──┼────┼──│ RDS: Read Replica  │  │      │
│  │  │ ElastiCache: 3 nds │  │    │  │ ElastiCache: 1 nd  │  │      │
│  │  │ DynamoDB: Global   │──┼────┼──│ DynamoDB: Global   │  │      │
│  │  │ S3: CRR enabled    │──┼────┼──│ S3: Replica bucket │  │      │
│  │  └────────────────────┘  │    │  └────────────────────┘  │      │
│  └──────────────────────────┘    └──────────────────────────┘      │
│                                                                      │
│  RPO: < 5 min (async replication lag)                               │
│  RTO: < 15 min (automated failover)                                 │
└────────────────────────────────────────────────────────────────────┘
```

**Automated Failover Implementation:**

```python
# Step Function: Automated DR Failover Orchestration
failover_state_machine = {
    "Comment": "Automated DR Failover",
    "StartAt": "DetectFailure",
    "States": {
        "DetectFailure": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-west-2:123456789:function:validate-primary-health",
            "Next": "ConfirmFailover",
            "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 3}]
        },
        "ConfirmFailover": {
            "Type": "Choice",
            "Choices": [{
                "Variable": "$.primaryHealthy",
                "BooleanEquals": False,
                "Next": "ExecuteFailover"
            }],
            "Default": "AbortFailover"
        },
        "ExecuteFailover": {
            "Type": "Parallel",
            "Branches": [
                {
                    "StartAt": "PromoteRDSReplica",
                    "States": {
                        "PromoteRDSReplica": {
                            "Type": "Task",
                            "Resource": "arn:aws:lambda:us-west-2:123456789:function:promote-rds-replica",
                            "End": True
                        }
                    }
                },
                {
                    "StartAt": "ScaleUpSecondary",
                    "States": {
                        "ScaleUpSecondary": {
                            "Type": "Task",
                            "Resource": "arn:aws:lambda:us-west-2:123456789:function:scale-up-ecs",
                            "End": True
                        }
                    }
                },
                {
                    "StartAt": "ScaleUpCache",
                    "States": {
                        "ScaleUpCache": {
                            "Type": "Task",
                            "Resource": "arn:aws:lambda:us-west-2:123456789:function:scale-up-elasticache",
                            "End": True
                        }
                    }
                }
            ],
            "Next": "UpdateDNS"
        },
        "UpdateDNS": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-west-2:123456789:function:failover-route53",
            "Next": "ValidateSecondary"
        },
        "ValidateSecondary": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-west-2:123456789:function:validate-secondary-health",
            "Next": "NotifyTeam"
        },
        "NotifyTeam": {
            "Type": "Task",
            "Resource": "arn:aws:sns:us-west-2:123456789:dr-failover-notifications",
            "End": True
        },
        "AbortFailover": {
            "Type": "Succeed"
        }
    }
}
```

**Route 53 Health Check + Failover:**

```hcl
# Primary health check
resource "aws_route53_health_check" "primary" {
  fqdn              = "api-primary.company.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health/deep"
  failure_threshold = 3
  request_interval  = 10  # Fast detection (10 sec)
  
  regions = ["us-east-1", "us-west-2", "eu-west-1"]  # Check from 3 regions
  
  tags = { Name = "primary-health-check" }
}

# Composite health check (multiple signals)
resource "aws_route53_health_check" "primary_composite" {
  type                   = "CALCULATED"
  child_health_threshold = 2  # 2 of 3 must be healthy
  child_healthchecks     = [
    aws_route53_health_check.primary.id,
    aws_route53_health_check.primary_db.id,
    aws_route53_health_check.primary_app.id
  ]
}

# DNS failover record
resource "aws_route53_record" "api" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.company.com"
  type    = "A"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary_composite.id
  
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.company.com"
  type    = "A"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  set_identifier = "secondary"
  
  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

**RDS Cross-Region Replica Promotion:**

```python
import boto3
import time

def promote_rds_replica(event, context):
    """Promote RDS read replica to standalone in DR region."""
    rds = boto3.client('rds', region_name='us-west-2')
    
    # Step 1: Promote replica
    rds.promote_read_replica_db_cluster(
        DBClusterIdentifier='app-db-replica-usw2'
    )
    
    # Step 2: Wait for promotion (typically 1-5 minutes)
    waiter = rds.get_waiter('db_cluster_available')
    waiter.wait(
        DBClusterIdentifier='app-db-replica-usw2',
        WaiterConfig={'Delay': 10, 'MaxAttempts': 60}
    )
    
    # Step 3: Enable Multi-AZ on promoted instance (for HA)
    rds.modify_db_cluster(
        DBClusterIdentifier='app-db-replica-usw2',
        DeletionProtection=True
    )
    
    return {'status': 'promoted', 'endpoint': get_cluster_endpoint('app-db-replica-usw2')}
```

**DR Testing (Game Days):**

```yaml
# Quarterly DR test plan
dr_test_plan:
  frequency: quarterly
  type: full_failover
  
  pre_test:
    - Notify stakeholders 48 hours in advance
    - Verify secondary region health
    - Confirm backups are current (< 5 min old)
    - Disable non-essential alerting
    - Brief incident commander
  
  test_execution:
    - T+0: Simulate primary failure (block Route 53 health checks)
    - T+1min: Verify automated failover triggers
    - T+5min: Validate DNS propagation
    - T+10min: Run synthetic transaction tests
    - T+15min: Verify data consistency
    - T+30min: Run full integration test suite
    - T+60min: Measure actual RPO and RTO
  
  post_test:
    - Failback to primary (controlled)
    - Verify data reconciliation
    - Document gaps and improvements
    - Update runbooks
    
  success_criteria:
    - RTO achieved: < 15 minutes
    - RPO achieved: < 5 minutes
    - All synthetic transactions pass
    - No data corruption
    - Alert pipeline functional
```

---

### Q5: Your CloudWatch alarms are too noisy - the on-call engineer gets 200+ alerts per night, most are false positives. How do you redesign the alerting strategy?

**Answer:**

**Alert Quality Framework:**

```
┌─────────────────────────────────────────────────────────┐
│  Alert Classification (Before Redesign):                 │
│                                                          │
│  200 alerts/night                                        │
│  ├── 150 non-actionable (monitoring metrics, INFO)       │
│  ├── 30 transient (auto-recover within 5 min)           │
│  ├── 15 duplicate (same issue, multiple signals)        │
│  └── 5 actionable (real issues requiring response)       │
│                                                          │
│  Target: < 5 actionable alerts that require response     │
└─────────────────────────────────────────────────────────┘
```

**Solution 1: Alert Tiering:**

```hcl
# Tier 1: PAGE (wake someone up) - Symptom-based, customer-facing
resource "aws_cloudwatch_metric_alarm" "page_availability" {
  alarm_name          = "P1-payment-api-availability-critical"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 3        # 3 consecutive periods
  datapoints_to_alarm = 3        # All 3 must breach
  period              = 60       # 1-minute periods
  threshold           = 99.5     # Below SLO
  treat_missing_data  = "notBreaching"  # Don't alarm on missing data
  
  alarm_actions = [aws_sns_topic.pagerduty_high.arn]
}

# Tier 2: TICKET (fix during business hours) - Cause-based
resource "aws_cloudwatch_metric_alarm" "ticket_high_cpu" {
  alarm_name          = "P3-web-servers-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 10       # 10 consecutive periods (50 min)
  datapoints_to_alarm = 8        # 8 of 10 must breach (allows transients)
  period              = 300      # 5-minute periods
  threshold           = 85
  treat_missing_data  = "notBreaching"
  
  alarm_actions = [aws_sns_topic.jira_ticket.arn]
}

# Tier 3: LOG (visibility only) - Informational
resource "aws_cloudwatch_metric_alarm" "log_disk_usage" {
  alarm_name          = "INFO-disk-usage-elevated"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 6
  period              = 3600     # Hourly check
  threshold           = 70
  
  alarm_actions = [aws_sns_topic.slack_ops_channel.arn]
}
```

**Solution 2: Composite Alarms (Reduce noise):**

```hcl
# Instead of alerting on every individual metric,
# create composite alarms that represent actual incidents

resource "aws_cloudwatch_composite_alarm" "service_degraded" {
  alarm_name = "P1-payment-service-degraded"
  
  # Alert ONLY when BOTH conditions are true:
  # High error rate AND high latency (confirms real issue, not transient)
  alarm_rule = <<-RULE
    ALARM(${aws_cloudwatch_metric_alarm.error_rate_high.alarm_name})
    AND
    ALARM(${aws_cloudwatch_metric_alarm.latency_p99_high.alarm_name})
  RULE
  
  alarm_actions = [aws_sns_topic.pagerduty.arn]
  
  # Suppress for 5 minutes after resolution (prevent flapping)
  actions_suppressor       = aws_cloudwatch_metric_alarm.suppressor.arn
  actions_suppressor_wait_period    = 300
  actions_suppressor_extension_period = 300
}

# Another example: Alert only if MULTIPLE AZs are affected
resource "aws_cloudwatch_composite_alarm" "multi_az_failure" {
  alarm_name = "P1-multi-az-failure"
  
  alarm_rule = <<-RULE
    (
      ALARM(${aws_cloudwatch_metric_alarm.az_a_unhealthy.alarm_name})
      AND
      ALARM(${aws_cloudwatch_metric_alarm.az_b_unhealthy.alarm_name})
    )
    OR
    ALARM(${aws_cloudwatch_metric_alarm.all_targets_unhealthy.alarm_name})
  RULE
  
  alarm_actions = [aws_sns_topic.pagerduty_critical.arn]
}
```

**Solution 3: Anomaly Detection (Eliminate static thresholds):**

```hcl
resource "aws_cloudwatch_metric_alarm" "anomaly_request_count" {
  alarm_name          = "P2-request-count-anomaly"
  comparison_operator = "LessThanLowerOrGreaterThanUpperThreshold"
  evaluation_periods  = 3
  
  # Uses ML-based anomaly detection band instead of static threshold
  threshold_metric_id = "anomaly_band"
  
  metric_query {
    id          = "actual"
    return_data = true
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 300
      stat        = "Sum"
    }
  }
  
  metric_query {
    id          = "anomaly_band"
    expression  = "ANOMALY_DETECTION_BAND(actual, 3)"  # 3 standard deviations
    label       = "Expected Range"
    return_data = true
  }
}
```

**Solution 4: Alert Routing and Deduplication:**

```python
# EventBridge → Lambda → Smart routing
def route_alert(event):
    """Intelligent alert routing with deduplication."""
    alarm = event['detail']
    alarm_name = alarm['alarmName']
    
    # Check if this is a duplicate of an existing incident
    incidents = get_active_incidents()
    for incident in incidents:
        if is_related_alarm(alarm_name, incident):
            # Add as context to existing incident, don't create new page
            add_context_to_incident(incident['id'], alarm)
            return
    
    # Apply time-based routing
    current_hour = datetime.now().hour
    if current_hour >= 22 or current_hour < 7:
        # Night: Only P1 alerts page
        if not alarm_name.startswith('P1-'):
            queue_for_morning(alarm)
            return
    
    # Route based on service ownership
    team = get_team_for_alarm(alarm_name)
    route_to_team(team, alarm)
```

**Alerting Best Practices:**

| Principle | Implementation |
|-----------|---------------|
| Alert on symptoms, not causes | "Error rate > 1%" not "CPU > 80%" |
| Every alert must be actionable | If no runbook exists, it's not an alert |
| Use evaluation periods > 1 | Prevents transient spikes from paging |
| Treat missing data as "not breaching" | Avoid alerts during deploys |
| Use composite alarms | Correlate signals before alerting |
| Review alert quality monthly | Track: actionable %, false positive % |

---

### Q6: Your ECS Fargate service experiences intermittent 502 errors during deployments. Users report brief outages during every release. How do you achieve zero-downtime deployments?

**Answer:**

**Root Cause Analysis:**

```
502 errors during deployment happen because:
1. New tasks start receiving traffic before they're ready
2. Old tasks are killed before draining connections
3. Health check passes too early (container up, app not ready)
4. ALB deregistration delay too short
```

**Solution: Complete Zero-Downtime Deployment:**

```hcl
# ECS Service with proper deployment configuration
resource "aws_ecs_service" "app" {
  name            = "payment-api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 4
  launch_type     = "FARGATE"
  
  # Key: Rolling update with proper health checks
  deployment_configuration {
    maximum_percent         = 200  # Double capacity during deploy
    minimum_healthy_percent = 100  # Never go below desired count
    
    deployment_circuit_breaker {
      enable   = true
      rollback = true  # Auto-rollback on failure
    }
  }
  
  # Key: Health check grace period
  health_check_grace_period_seconds = 120  # Wait 2 min before checking health
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 8080
  }
  
  deployment_controller {
    type = "ECS"  # Or "CODE_DEPLOY" for blue/green
  }
}

# Target Group with proper health check
resource "aws_lb_target_group" "app" {
  name        = "payment-api-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"
  
  health_check {
    enabled             = true
    path                = "/health/ready"  # Readiness check (not liveness!)
    port                = "traffic-port"
    protocol            = "HTTP"
    healthy_threshold   = 3       # 3 consecutive passes to be healthy
    unhealthy_threshold = 3       # 3 consecutive fails to be unhealthy
    interval            = 10      # Check every 10 seconds
    timeout             = 5
    matcher             = "200"
  }
  
  # Key: Deregistration delay for connection draining
  deregistration_delay = 120  # 120 seconds to drain connections
  
  # Slow start for new targets (gradual traffic shift)
  slow_start = 60  # 60 seconds ramp-up period
}
```

**Application-Level Readiness:**

```python
# Health check endpoints in the application
from fastapi import FastAPI, Response

app = FastAPI()
_ready = False  # Not ready until fully initialized

@app.on_event("startup")
async def startup_event():
    """Full initialization before accepting traffic."""
    global _ready
    # Wait for DB connection pool to be established
    await init_database_pool()
    # Wait for cache connections
    await init_cache_connections()
    # Warm up application caches
    await warm_caches()
    # Load ML models if applicable
    await load_models()
    _ready = True

@app.get("/health/live")
async def liveness():
    """Is the process alive? (for ECS task health)"""
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness():
    """Is the app ready to receive traffic? (for ALB health check)"""
    if not _ready:
        return Response(status_code=503, content="Not ready")
    # Also check dependencies
    if not await check_db_connection():
        return Response(status_code=503, content="DB unavailable")
    return {"status": "ready"}

# Graceful shutdown - handle SIGTERM
import signal
import asyncio

async def graceful_shutdown(sig):
    """Handle SIGTERM from ECS (stop accepting new requests, finish existing)."""
    global _ready
    _ready = False  # Stop passing health checks (ALB stops sending traffic)
    
    # Wait for in-flight requests to complete (matches deregistration_delay)
    await asyncio.sleep(30)
    
    # Close database connections gracefully
    await close_database_pool()
    await close_cache_connections()

signal.signal(signal.SIGTERM, lambda s, f: asyncio.create_task(graceful_shutdown(s)))
```

**ECS Task Definition with Stop Timeout:**

```hcl
resource "aws_ecs_task_definition" "app" {
  family                   = "payment-api"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 1024
  memory                   = 2048
  
  container_definitions = jsonencode([{
    name  = "app"
    image = "${aws_ecr_repository.app.repository_url}:${var.image_tag}"
    
    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]
    
    # Key: Give container time to gracefully shutdown
    stopTimeout = 120  # 120 seconds to handle SIGTERM before SIGKILL
    
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:8080/health/live || exit 1"]
      interval    = 10
      timeout     = 5
      retries     = 3
      startPeriod = 60  # Don't check health for first 60 seconds
    }
    
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/payment-api"
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "app"
      }
    }
  }])
}
```

---

### Q7: Your RDS Aurora cluster experiences a failover. The application reconnects within 30 seconds, but some transactions are lost. How do you prevent this?

**Answer:**

**Understanding Aurora Failover:**

```
Normal Operation:         During Failover:
┌─────────────────┐      ┌─────────────────┐
│ Writer Instance │      │ Writer (FAILING) │ ← Connection drops
│ (primary)       │      │                  │
└────────┬────────┘      └─────────────────┘
         │
         │ Replication              (1-2 seconds)
         │ (synchronous             ↓
         │  within cluster)   ┌─────────────────┐
         ▼                    │ Reader promoted  │ ← New writer
┌─────────────────┐          │ to Writer        │
│ Reader Instance │          └─────────────────┘
│ (replica)       │
└─────────────────┘          DNS update: 5-30 seconds
                              (cluster endpoint)
```

**Why Transactions Are Lost:**

1. **In-flight transactions:** Uncommitted transactions on the failed writer are lost
2. **Application connection pool:** Holds stale connections to old writer
3. **DNS caching:** Application resolves old IP even after failover
4. **Connection timeout:** Application hangs waiting for dead connection

**Solution: Application-Level Resilience:**

```python
# Python/SQLAlchemy with Aurora-aware retry logic
from sqlalchemy import create_engine, event
from sqlalchemy.orm import sessionmaker
from sqlalchemy.exc import OperationalError, DisconnectionError
import time

# Key 1: Use cluster endpoint (not instance endpoint)
# The cluster endpoint DNS updates during failover
WRITER_ENDPOINT = "app-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com"
READER_ENDPOINT = "app-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com"

engine = create_engine(
    f"postgresql://user:pass@{WRITER_ENDPOINT}:5432/appdb",
    pool_size=20,
    max_overflow=10,
    pool_timeout=10,
    pool_recycle=300,      # Key 2: Recycle connections every 5 min
    pool_pre_ping=True,    # Key 3: Test connection before use
    connect_args={
        "connect_timeout": 5,          # Key 4: Short connect timeout
        "options": "-c statement_timeout=30000"  # 30s query timeout
    }
)

# Key 5: Handle disconnect events
@event.listens_for(engine, "engine_connect")
def receive_engine_connect(conn, branch):
    """Verify connection on checkout from pool."""
    if branch:
        return
    # Force DNS re-resolution on new connections
    conn.execute("SELECT 1")

# Key 6: Retry decorator for transient failures
import functools

def retry_on_failover(max_retries=3, initial_delay=1):
    """Retry transactions that fail due to Aurora failover."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except OperationalError as e:
                    last_exception = e
                    error_code = getattr(e.orig, 'pgcode', '')
                    
                    # Aurora failover error codes
                    if error_code in ('08003', '08006', '57P01', '08001'):
                        delay = initial_delay * (2 ** attempt)
                        logger.warning(f"Failover detected, retry {attempt+1}/{max_retries} in {delay}s")
                        time.sleep(delay)
                        
                        # Dispose all pooled connections (force reconnect)
                        engine.dispose()
                        continue
                    raise
            raise last_exception
        return wrapper
    return decorator

# Usage:
@retry_on_failover(max_retries=3)
def process_payment(payment_data):
    """Process payment with failover resilience."""
    with Session() as session:
        try:
            payment = Payment(**payment_data)
            session.add(payment)
            session.commit()
            return payment.id
        except Exception:
            session.rollback()
            raise
```

**Aurora-Specific Best Practices:**

```hcl
# Aurora Cluster with fast failover
resource "aws_rds_cluster" "app" {
  cluster_identifier     = "app-cluster"
  engine                 = "aurora-postgresql"
  engine_version         = "15.4"
  master_username        = "admin"
  master_password        = var.db_password
  
  # Fast failover settings
  preferred_backup_window      = "03:00-04:00"
  backup_retention_period      = 7
  
  # Enable enhanced monitoring for failover detection
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  # Key: Enable cluster-level failover priority
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.fast_failover.name
}

resource "aws_rds_cluster_parameter_group" "fast_failover" {
  name   = "fast-failover-params"
  family = "aurora-postgresql15"
  
  parameter {
    name  = "tcp_keepalives_idle"
    value = "5"  # Detect dead connections faster
  }
  parameter {
    name  = "tcp_keepalives_interval"
    value = "1"
  }
  parameter {
    name  = "tcp_keepalives_count"
    value = "3"  # 3 missed keepalives = dead
  }
}

# RDS Proxy for connection pooling and fast failover
resource "aws_db_proxy" "app" {
  name                   = "app-db-proxy"
  debug_logging          = false
  engine_family          = "POSTGRESQL"
  idle_client_timeout    = 1800
  require_tls            = true
  role_arn               = aws_iam_role.rds_proxy.arn
  vpc_security_group_ids = [aws_security_group.rds_proxy.id]
  vpc_subnet_ids         = var.private_subnet_ids
  
  auth {
    auth_scheme = "SECRETS"
    iam_auth    = "REQUIRED"
    secret_arn  = aws_secretsmanager_secret.db_creds.arn
  }
}

# RDS Proxy handles failover transparently!
# Applications connect to proxy endpoint instead of cluster endpoint
# Proxy detects failover and redirects connections within seconds
```

---

### Q8: How do you implement chaos engineering on AWS to proactively find reliability issues before they impact production?

**Answer:**

**AWS Fault Injection Simulator (FIS) Implementation:**

```hcl
# Experiment: Test application resilience to AZ failure
resource "aws_fis_experiment_template" "az_failure" {
  description = "Simulate AZ failure - verify application maintains SLO"
  role_arn    = aws_iam_role.fis.arn
  
  stop_condition {
    source = "aws:cloudwatch:alarm"
    value  = aws_cloudwatch_metric_alarm.slo_critical.arn
  }
  
  # Action 1: Terminate EC2 instances in one AZ
  action {
    name        = "terminate-instances-az-a"
    action_id   = "aws:ec2:terminate-instances"
    description = "Terminate all instances in us-east-1a"
    
    target {
      key   = "Instances"
      value = "instances-az-a"
    }
  }
  
  # Action 2: Disrupt ECS tasks
  action {
    name        = "stop-ecs-tasks-az-a"
    action_id   = "aws:ecs:stop-task"
    description = "Stop ECS tasks in AZ-a"
    
    target {
      key   = "Tasks"
      value = "ecs-tasks-az-a"
    }
    
    start_after = ["terminate-instances-az-a"]
  }
  
  # Action 3: Network disruption (packet loss)
  action {
    name        = "network-disruption"
    action_id   = "aws:ssm:send-command"
    description = "Inject 50% packet loss on target instances"
    
    parameter {
      key   = "documentArn"
      value = "arn:aws:ssm:us-east-1::document/AWSFIS-Run-Network-Packet-Loss"
    }
    parameter {
      key   = "documentParameters"
      value = jsonencode({
        "DurationSeconds" = "300",
        "LossPercent"     = "50",
        "Interface"       = "eth0"
      })
    }
    
    target {
      key   = "Instances"
      value = "instances-az-b"
    }
  }
  
  # Targets
  target {
    name           = "instances-az-a"
    resource_type  = "aws:ec2:instance"
    selection_mode = "ALL"
    
    resource_tag {
      key   = "Environment"
      value = "production"
    }
    resource_tag {
      key   = "AvailabilityZone"
      value = "us-east-1a"
    }
  }
  
  target {
    name           = "ecs-tasks-az-a"
    resource_type  = "aws:ecs:task"
    selection_mode = "PERCENT(50)"
    
    filter {
      path   = "State.Name"
      values = ["RUNNING"]
    }
  }
}
```

**Chaos Engineering Program:**

```yaml
# Progressive chaos testing maturity
chaos_engineering_program:
  level_1_basic:
    environment: staging
    experiments:
      - name: "Single instance failure"
        tool: FIS
        action: Terminate 1 EC2 instance
        expected: Auto-scaling replaces, no user impact
        frequency: weekly
      
      - name: "Cache failure"
        tool: FIS
        action: Reboot ElastiCache node
        expected: Cache miss handled gracefully, slight latency increase
        frequency: weekly
  
  level_2_intermediate:
    environment: production (off-peak)
    experiments:
      - name: "AZ failure simulation"
        tool: FIS
        action: Terminate all instances in 1 AZ
        expected: Multi-AZ architecture handles, SLO maintained
        frequency: monthly
      
      - name: "Dependency failure"
        tool: FIS + SSM
        action: Block network to downstream API
        expected: Circuit breaker activates, graceful degradation
        frequency: monthly
  
  level_3_advanced:
    environment: production (peak hours)
    experiments:
      - name: "Region failure simulation"
        tool: FIS + custom
        action: Simulate regional outage
        expected: DR failover completes within RTO
        frequency: quarterly
      
      - name: "Data corruption scenario"
        tool: Custom
        action: Inject bad data into pipeline
        expected: Data quality checks catch and quarantine
        frequency: quarterly

  guardrails:
    - Automated stop conditions (SLO breach → abort experiment)
    - Blast radius limits (never affect > 10% of traffic)
    - Communication plan (stakeholders notified)
    - Rollback procedure documented and tested
    - Only during business hours with full team available
```

---

### Q9: Your microservices architecture has cascading failures - when one service slows down, it brings down 5 other services. How do you prevent this?

**Answer:**

**Cascading Failure Pattern:**
```
Service A (slow) ← Service B (waiting) ← Service C (timeout)
      ↑                    ↑                     ↑
  Thread pool          Thread pool          Thread pool
  exhausted            filling up           starting to fill

Result: One slow service → ALL services become unavailable
```

**Solution: Multi-Layer Resilience:**

**Layer 1: Circuit Breaker Pattern:**
```python
# Using resilience4j-inspired pattern (or AWS SDK retry)
import time
from enum import Enum
from dataclasses import dataclass
from threading import Lock

class CircuitState(Enum):
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Failing fast (no calls to downstream)
    HALF_OPEN = "half_open" # Testing if downstream recovered

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5       # Failures before opening
    recovery_timeout: float = 30.0   # Seconds before half-open
    success_threshold: int = 3       # Successes before closing
    timeout: float = 5.0             # Call timeout

class CircuitBreaker:
    def __init__(self, name: str, config: CircuitBreakerConfig):
        self.name = name
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = 0
        self.lock = Lock()
    
    def call(self, func, *args, **kwargs):
        with self.lock:
            if self.state == CircuitState.OPEN:
                if time.time() - self.last_failure_time > self.config.recovery_timeout:
                    self.state = CircuitState.HALF_OPEN
                    self.success_count = 0
                else:
                    raise CircuitOpenError(f"Circuit {self.name} is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        with self.lock:
            if self.state == CircuitState.HALF_OPEN:
                self.success_count += 1
                if self.success_count >= self.config.success_threshold:
                    self.state = CircuitState.CLOSED
                    self.failure_count = 0
            elif self.state == CircuitState.CLOSED:
                self.failure_count = 0
    
    def _on_failure(self):
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.OPEN
            elif self.failure_count >= self.config.failure_threshold:
                self.state = CircuitState.OPEN

# Usage
payment_circuit = CircuitBreaker("payment-service", CircuitBreakerConfig(
    failure_threshold=5,
    recovery_timeout=30,
    timeout=3.0
))

def get_payment_status(payment_id):
    try:
        return payment_circuit.call(payment_client.get_status, payment_id)
    except CircuitOpenError:
        # Fallback: Return cached/default response
        return get_cached_payment_status(payment_id)
```

**Layer 2: Bulkhead Pattern (Isolate failures):**
```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

# Separate thread pools per downstream service
# If one service exhausts its pool, others are unaffected
class BulkheadedClient:
    def __init__(self):
        # Each service gets its own isolated resource pool
        self.payment_pool = ThreadPoolExecutor(max_workers=20, thread_name_prefix="payment")
        self.inventory_pool = ThreadPoolExecutor(max_workers=15, thread_name_prefix="inventory")
        self.notification_pool = ThreadPoolExecutor(max_workers=10, thread_name_prefix="notification")
    
    async def get_payment(self, payment_id):
        """Payment calls limited to 20 concurrent."""
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            self.payment_pool,
            payment_client.get, payment_id
        )
    
    async def check_inventory(self, product_id):
        """Inventory calls limited to 15 concurrent."""
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            self.inventory_pool,
            inventory_client.check, product_id
        )
```

**Layer 3: Timeout + Retry with Backoff:**
```python
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

# Configure HTTP client with aggressive timeouts
client = httpx.AsyncClient(
    timeout=httpx.Timeout(
        connect=2.0,     # Connection timeout: 2 seconds
        read=5.0,        # Read timeout: 5 seconds
        write=5.0,       # Write timeout: 5 seconds
        pool=3.0         # Pool connection timeout: 3 seconds
    ),
    limits=httpx.Limits(
        max_connections=100,
        max_keepalive_connections=20,
        keepalive_expiry=30
    )
)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type((httpx.TimeoutException, httpx.NetworkError)),
    before_sleep=lambda retry_state: logger.warning(f"Retrying: attempt {retry_state.attempt_number}")
)
async def call_downstream_service(endpoint: str, payload: dict):
    response = await client.post(endpoint, json=payload)
    response.raise_for_status()
    return response.json()
```

**Layer 4: Load Shedding (Reject excess traffic):**
```python
# Adaptive rate limiting based on service health
from fastapi import FastAPI, Request, Response
import time

app = FastAPI()

class AdaptiveRateLimiter:
    def __init__(self, max_rps: int = 1000):
        self.max_rps = max_rps
        self.current_rps = 0
        self.window_start = time.time()
        self.health_factor = 1.0  # 1.0 = healthy, 0.5 = degraded
    
    def should_accept(self) -> bool:
        now = time.time()
        if now - self.window_start >= 1.0:
            self.current_rps = 0
            self.window_start = now
        
        self.current_rps += 1
        effective_limit = self.max_rps * self.health_factor
        return self.current_rps <= effective_limit
    
    def degrade(self):
        """Called when downstream is slow - reduce accepted traffic."""
        self.health_factor = max(0.1, self.health_factor - 0.1)
    
    def recover(self):
        """Called when downstream recovers."""
        self.health_factor = min(1.0, self.health_factor + 0.05)

limiter = AdaptiveRateLimiter(max_rps=5000)

@app.middleware("http")
async def load_shedding_middleware(request: Request, call_next):
    if not limiter.should_accept():
        return Response(
            status_code=503,
            content="Service temporarily overloaded",
            headers={"Retry-After": "5"}
        )
    return await call_next(request)
```

**Layer 5: AWS-Native Resilience:**
```hcl
# App Mesh for service-level circuit breaking and retries
resource "aws_appmesh_virtual_node" "payment" {
  name      = "payment-service"
  mesh_name = aws_appmesh_mesh.main.name
  
  spec {
    listener {
      port_mapping {
        port     = 8080
        protocol = "http"
      }
      
      timeout {
        http {
          idle {
            unit  = "s"
            value = 60
          }
          per_request {
            unit  = "s"
            value = 5  # 5 second timeout per request
          }
        }
      }
      
      outlier_detection {
        base_ejection_duration {
          unit  = "s"
          value = 30
        }
        interval {
          unit  = "s"
          value = 10
        }
        max_ejection_percent = 50
        max_server_errors    = 5  # Circuit opens after 5 errors
      }
    }
    
    backend {
      virtual_service {
        virtual_service_name = "inventory.local"
        client_policy {
          tls { enforce = true }
        }
      }
    }
  }
}
```

---

### Q10: Your application must handle a 10x traffic spike during a flash sale event (Black Friday). Current architecture handles 5,000 RPS, you need 50,000 RPS. What's your preparation plan?

**Answer:**

**Capacity Planning & Scaling Strategy:**

```
┌─────────────────────────────────────────────────────────────┐
│               Flash Sale Architecture                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Edge Layer (Absorb 70% of traffic)                    │   │
│  │ CloudFront + WAF + Shield Advanced                    │   │
│  │ - Static content cached at edge                       │   │
│  │ - API responses cached (5-30 sec TTL)                │   │
│  │ - Rate limiting per IP                                │   │
│  └──────────────────────────────────────────────────────┘   │
│                        │ 30% passes through                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Application Layer (Pre-scaled)                        │   │
│  │ ALB + ECS Fargate (pre-warmed)                       │   │
│  │ - 50 tasks pre-provisioned (not reactive scaling)     │   │
│  │ - Fargate Spot for cost optimization                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                        │                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Data Layer (Write-behind pattern)                     │   │
│  │ ElastiCache (Redis) for hot data                     │   │
│  │ - Product catalog: cached 5 min                      │   │
│  │ - Inventory: atomic decrements in Redis              │   │
│  │ - Order queue: SQS for async processing              │   │
│  │                                                       │   │
│  │ DynamoDB (on-demand) for order storage               │   │
│  │ Aurora (provisioned) for post-processing              │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Pre-Event Preparation Checklist:**

```yaml
t_minus_30_days:
  - Load test at 2x expected peak (100K RPS)
  - Identify bottlenecks and fix
  - Request service limit increases:
    - EC2: instance limits per region
    - ALB: connections per LB
    - NAT Gateway: bandwidth
    - DynamoDB: table throughput (if provisioned)
    - Lambda: concurrent executions
    - SQS: throughput
  - Pre-provision Fargate capacity
  - Enable DynamoDB on-demand (or reserved capacity)

t_minus_7_days:
  - Final load test at expected peak
  - Pre-warm ALB (request AWS Support for pre-warming)
  - Pre-warm CloudFront (prime cache with synthetic traffic)
  - Scale up ElastiCache nodes
  - Scale up RDS reader instances
  - Deploy canary monitoring
  - Brief incident response team
  - Document rollback procedures

t_minus_1_day:
  - Scale ECS to peak capacity (don't rely on auto-scaling speed)
  - Verify all alarms and dashboards
  - Disable non-critical cron jobs
  - Freeze all deployments
  - Enable enhanced monitoring (1-second granularity)
  - Notify AWS TAM/Support of event

during_event:
  - War room: SRE team monitoring dashboards
  - Pre-authorized to scale beyond plan if needed
  - Circuit breakers configured for non-essential features
  - Feature flags ready to disable resource-heavy features

post_event:
  - Gradual scale-down (don't rush)
  - Review performance data
  - Document lessons learned
  - Calculate actual cost vs budget
```

**Implementation Details:**

```hcl
# Pre-provisioned ECS capacity (don't rely on reactive auto-scaling)
resource "aws_ecs_service" "flash_sale" {
  name            = "product-api"
  desired_count   = 50  # Pre-scaled for peak
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 7
    base              = 20  # 20 guaranteed on-demand
  }
  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight            = 3  # 30% on spot for cost
  }
}

# ElastiCache (Redis) for inventory management
resource "aws_elasticache_replication_group" "flash_sale" {
  replication_group_id = "flash-sale-cache"
  node_type            = "cache.r6g.2xlarge"
  num_cache_clusters   = 6  # 3 shards × 2 replicas
  
  parameter_group_name = aws_elasticache_parameter_group.optimized.name
  
  # Key: Read from replicas for read-heavy workload
  automatic_failover_enabled = true
  multi_az_enabled           = true
}

# SQS for order processing (decouple and buffer)
resource "aws_sqs_queue" "orders" {
  name                       = "flash-sale-orders"
  visibility_timeout_seconds = 300
  message_retention_seconds  = 86400  # 24 hours
  receive_wait_time_seconds  = 20     # Long polling
  
  # Scale consumers based on queue depth
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3
  })
}
```

```python
# Inventory management with Redis (atomic operations)
import redis

redis_client = redis.Redis(host='flash-sale-cache.xxx.cache.amazonaws.com', port=6379)

def purchase_item(product_id: str, quantity: int = 1) -> bool:
    """Atomic inventory decrement - handles 50K+ concurrent requests."""
    # DECRBY is atomic - no race conditions
    remaining = redis_client.decrby(f"inventory:{product_id}", quantity)
    
    if remaining < 0:
        # Oversold! Roll back
        redis_client.incrby(f"inventory:{product_id}", quantity)
        return False  # Out of stock
    
    # Enqueue order for async processing
    order = {"product_id": product_id, "quantity": quantity, "timestamp": time.time()}
    sqs.send_message(QueueUrl=ORDER_QUEUE_URL, MessageBody=json.dumps(order))
    
    return True

# Pre-load inventory into Redis before event
def warm_inventory_cache():
    """Load inventory counts from DB into Redis."""
    products = db.query("SELECT product_id, stock_count FROM products WHERE flash_sale = true")
    pipeline = redis_client.pipeline()
    for product in products:
        pipeline.set(f"inventory:{product.product_id}", product.stock_count)
    pipeline.execute()
```
