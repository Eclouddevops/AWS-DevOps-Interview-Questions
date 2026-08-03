# AWS Load Balancing, Autoscaling & ASG Policies — Deep-Dive Interview Q&A

## Table of Contents
1. [Load Balancer Types](#load-balancer-types)
2. [ALB Deep Dive](#alb-deep-dive)
3. [NLB Deep Dive](#nlb-deep-dive)
4. [Auto Scaling Groups](#auto-scaling-groups)
5. [Scaling Policies](#scaling-policies)
6. [Tricky Scenarios](#tricky-scenarios)

---

## Load Balancer Types

**Q1: Compare ALB, NLB, CLB, and GWLB. When would you choose each?**

**A:**

| Feature | ALB | NLB | CLB (Legacy) | GWLB |
|---------|-----|-----|------|------|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) | 4/7 | 3 (IP) |
| Performance | Good | Ultra-high (millions RPS) | Moderate | High |
| Static IP | No (use Global Accelerator) | Yes (Elastic IP per AZ) | No | No |
| WebSocket | Yes | Yes (passthrough) | No | N/A |
| Path routing | Yes | No | No | N/A |
| Host routing | Yes | No | No | N/A |
| gRPC | Yes | Yes (passthrough) | No | N/A |
| Source IP preservation | Via X-Forwarded-For | Yes (native) | Via X-Forwarded-For | Yes |
| SSL termination | Yes | Yes (TLS) | Yes | N/A |
| Cross-zone | Default on (free) | Default off (charges apply) | Default on | N/A |
| Use case | Web apps, microservices | Gaming, IoT, extreme perf | Legacy | Third-party firewalls |

**Tricky**: NLB preserves source IP natively (target sees real client IP). ALB inserts `X-Forwarded-For` header (target sees ALB IP). This matters for IP-based ACLs in your application!

---

**Q2: Explain ALB target types and when to use each.**

**A:**

| Target Type | Description | Use Case |
|-------------|-------------|----------|
| Instance | EC2 instance ID | Traditional EC2 deployments |
| IP | Private IP address | ECS (awsvpc), on-premises (via DX/VPN), cross-VPC |
| Lambda | Lambda function ARN | Serverless HTTP APIs |
| ALB | Another ALB | Chain ALBs (Global Accelerator → ALB → ALB) |

```
# IP targets can reach:
├── ECS tasks (awsvpc networking)
├── Instances in peered VPCs
├── On-premises servers (via Direct Connect)
├── Fargate containers
└── Any IP in RFC1918 range within VPC
```

**Tricky**: With IP targets, health checks come FROM the ALB's IP (not the node). Security groups on targets must allow health check traffic from the ALB subnet CIDR, not just the ALB security group.

---

## ALB Deep Dive

**Q3: Explain ALB routing rules — priority, conditions, and actions.**

**A:**

```
Listener (port 443)
├── Rule 1 (priority 1): IF path=/api/* AND header=X-Version:v2 → Target Group: api-v2
├── Rule 2 (priority 2): IF path=/api/* → Target Group: api-v1
├── Rule 3 (priority 3): IF host=admin.example.com → Target Group: admin
├── Rule 4 (priority 4): IF source-ip=10.0.0.0/8 → Fixed Response: 403
└── Default Rule: → Target Group: web-default
```

**Condition types:**
- `host-header`: Route by domain name
- `path-pattern`: Route by URL path
- `http-header`: Route by any header value
- `http-request-method`: GET, POST, etc.
- `query-string`: Route by query parameters
- `source-ip`: Route by client IP CIDR

**Action types:**
- `forward`: Send to target group (with optional stickiness/weights)
- `redirect`: HTTP redirect (301/302)
- `fixed-response`: Return static response (maintenance pages)
- `authenticate-oidc`: Redirect to IdP for authentication
- `authenticate-cognito`: Redirect to Cognito for authentication

**Weighted target groups (canary):**
```json
{
  "Type": "forward",
  "ForwardConfig": {
    "TargetGroups": [
      {"TargetGroupArn": "arn:...stable", "Weight": 90},
      {"TargetGroupArn": "arn:...canary", "Weight": 10}
    ]
  }
}
```

---

**Q4: How does ALB handle sticky sessions? What are the problems?**

**A:**

**Two types:**
1. **Duration-based** (AWSALB cookie): ALB generates cookie, routes to same target for duration
2. **Application-based** (custom cookie): App generates cookie, ALB respects it

```
# Enable on target group
Stickiness: enabled
Type: app_cookie
Cookie name: SESSIONID
Duration: 86400 (seconds)
```

**Problems with sticky sessions:**
1. **Uneven load distribution**: One target gets all "stuck" users
2. **Scaling issues**: New targets get no traffic from existing sessions
3. **Failover**: If target dies, session is lost (user re-authenticates)
4. **Auto-scaling**: Scaling in terminates sessions prematurely

**Better alternatives:**
- Externalized session store (ElastiCache Redis/DynamoDB)
- JWT tokens (stateless — no server session needed)
- Consistent hashing at application level

**Tricky**: If you use sticky sessions with weighted target groups (canary), stickiness overrides the weight! Once a user is "stuck" to v1, they stay there even if you shift weight to v2. Must wait for cookie expiry or use different cookie names per version.

---

## NLB Deep Dive

**Q5: When must you use NLB instead of ALB? What are NLB-specific features?**

**A:**

**Must use NLB when:**
1. Need static IP / Elastic IP (whitelisting by partners)
2. Need to handle millions of requests per second
3. TCP/UDP protocol (non-HTTP) — databases, gaming, IoT
4. Need source IP preservation without headers
5. Need ultra-low latency (~100μs vs ALB's ~400μs)
6. PrivateLink (exposing service to other VPCs/accounts)
7. Need to handle volatile traffic patterns (NLB scales instantly)

**NLB + ALB combo pattern:**
```
Internet → NLB (static IP) → ALB (path routing) → Targets

Why: Partners whitelist NLB's static IPs, ALB provides L7 routing
```

**Tricky**: NLB passes through TCP connections. It does NOT terminate TLS by default (unless you configure TLS listener). This means:
- Target sees real client IP (no X-Forwarded-For needed)
- Target must handle TLS termination
- Security group on target must allow client IPs directly

---

## Auto Scaling Groups

**Q6: Explain ASG lifecycle hooks. How do you use them for graceful deployments?**

**A:**

```
Launch lifecycle:
Pending → Pending:Wait (hook) → Pending:Proceed → InService

Terminate lifecycle:
InService → Terminating → Terminating:Wait (hook) → Terminating:Proceed → Terminated
```

**Use cases:**
- **Launch hook**: Install software, register with config management, warm cache
- **Terminate hook**: Drain connections, deregister from service discovery, backup logs

```json
{
  "LifecycleHookName": "drain-connections",
  "AutoScalingGroupName": "web-asg",
  "LifecycleTransition": "autoscaling:EC2_INSTANCE_TERMINATING",
  "HeartbeatTimeout": 300,
  "DefaultResult": "CONTINUE"
}
```

**Lambda handling lifecycle hook:**
```python
def handler(event, context):
    instance_id = event['detail']['EC2InstanceId']
    hook_name = event['detail']['LifecycleHookName']

    # Drain connections from load balancer
    elbv2.deregister_targets(TargetGroupArn=tg_arn, Targets=[{'Id': instance_id}])

    # Wait for connections to drain
    waiter = elbv2.get_waiter('target_deregistered')
    waiter.wait(TargetGroupArn=tg_arn, Targets=[{'Id': instance_id}])

    # Signal completion
    autoscaling.complete_lifecycle_action(
        LifecycleHookName=hook_name,
        AutoScalingGroupName=asg_name,
        InstanceId=instance_id,
        LifecycleActionResult='CONTINUE'
    )
```

---

**Q7: Explain Launch Templates vs Launch Configurations. Why are Launch Configurations deprecated?**

**A:**

| Feature | Launch Template | Launch Configuration |
|---------|----------------|---------------------|
| Versioning | Yes (multiple versions) | No (immutable) |
| Modify after creation | Yes (new version) | No (must recreate) |
| Mixed instances | Yes | No |
| Spot + On-Demand mix | Yes | No |
| Inheritance | Yes (partial overrides) | No |
| T2/T3 unlimited | Yes | Limited |
| EBS optimization | Configurable | Limited |
| Status | Current | Deprecated (no new features) |

**Mixed instances policy (cost optimization):**
```json
{
  "LaunchTemplate": {"LaunchTemplateId": "lt-xxx", "Version": "$Latest"},
  "Overrides": [
    {"InstanceType": "m5.large"},
    {"InstanceType": "m5a.large"},
    {"InstanceType": "m4.large"},
    {"InstanceType": "r5.large"}
  ],
  "InstancesDistribution": {
    "OnDemandBaseCapacity": 2,
    "OnDemandPercentageAboveBaseCapacity": 30,
    "SpotAllocationStrategy": "capacity-optimized"
  }
}
```

---

## Scaling Policies

**Q8: Compare all ASG scaling policy types. When to use each?**

**A:**

| Policy | How It Works | Reaction Time | Use Case |
|--------|-------------|---------------|----------|
| Target Tracking | Maintains metric at target value | ~60s (1 data point) | Most common (CPU at 60%) |
| Step Scaling | Different actions at different thresholds | Configurable (1-5 min) | Aggressive scaling at high load |
| Simple Scaling | One action per alarm | Cooldown period | Legacy (avoid) |
| Scheduled | Time-based scale | Exact time | Predictable patterns (9am scale up) |
| Predictive | ML-based forecasting | Proactive (before load) | Repeating patterns |

**Target Tracking example:**
```json
{
  "TargetTrackingConfiguration": {
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 60.0,
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300,
    "DisableScaleIn": false
  }
}
```

**Step Scaling example:**
```
CPU 60-70%: Add 1 instance
CPU 70-85%: Add 3 instances
CPU 85-100%: Add 5 instances (aggressive scale-out)
```

**Tricky**: Target Tracking creates and manages CloudWatch alarms automatically. Don't delete them manually! Also, you can have MULTIPLE scaling policies on one ASG — ASG takes the action that results in the MOST capacity change.

---

**Q9: How does predictive scaling work? When does it fail?**

**A:**

**How it works:**
1. Analyzes 14 days of historical CloudWatch data
2. Uses ML to identify patterns (daily, weekly cycles)
3. Proactively scales BEFORE traffic arrives
4. Combines with reactive scaling for unexpected spikes

**When it works well:**
- Daily traffic patterns (e-commerce: peak at noon)
- Weekly patterns (B2B: low on weekends)
- Consistent, repeating workloads

**When it fails:**
- Random/unpredictable traffic (viral events)
- New applications with < 24 hours of history
- Traffic shaped by external events (product launches)
- Applications with long boot times (need warm pool instead)

**Warm Pools** (complement to predictive scaling):
```
Warm Pool: Pre-initialized instances in Stopped state
├── No compute charges (only EBS storage)
├── Launches in seconds (vs minutes for cold launch)
├── Good for: Java apps, large AMIs, long bootstrapping
└── States: Stopped | Running | Hibernated
```

---

**Q10: How do you handle scaling with connection draining and deployment?**

**A:**

**Connection draining (deregistration delay):**
```
Target Group setting: deregistration_delay = 300 seconds (default)

When target is deregistered:
1. ALB stops sending NEW connections to target
2. Existing connections allowed to complete (up to 300s)
3. After timeout: remaining connections forcefully closed
4. Target removed from target group
```

**Deployment with ASG (rolling update):**
```json
{
  "AutoScalingRollingUpdate": {
    "MinInstancesInService": 2,
    "MaxBatchSize": 1,
    "PauseTime": "PT5M",
    "WaitOnResourceSignals": true,
    "SuspendProcesses": ["HealthCheck", "ReplaceUnhealthy", "AZRebalance"]
  }
}
```

**Tricky interaction**: If ASG scales IN during deployment:
1. New instances launch with new AMI
2. ASG decides old instance should be terminated (scale-in)
3. Connection draining starts (300s)
4. Health check marks draining instance as unhealthy
5. ASG launches ANOTHER instance (replace unhealthy) — infinite loop!

**Fix**: Suspend `ReplaceUnhealthy` during deployments, or use proper blue-green with separate ASGs.

---

## Tricky Scenarios

**Q11: Your ASG keeps launching and terminating instances in a loop (flapping). What's happening?**

**A:**

**Common causes:**

1. **Health check too aggressive**: Instance boots → passes ELB health check → gets traffic → overwhelmed → fails health check → terminated → repeat
   - Fix: Increase health check grace period, use startup probes

2. **Scaling cooldown too short**: Scale out → metric drops → scale in → metric rises → scale out
   - Fix: Increase scale-in cooldown (300-600s)

3. **Target tracking target too low**: Target CPU=30% → always trying to scale out, then metrics drop after scaling → scales in
   - Fix: Increase target to 60-70%

4. **AZ Rebalance**: Uneven AZ distribution → ASG terminates in over-provisioned AZ → launches in under-provisioned AZ
   - Shows as constant churn in CloudTrail

5. **Spot interruptions**: Spot instances terminated → ASG replaces → gets interrupted again
   - Fix: Use capacity-optimized allocation, diversify instance types

**Debugging:**
```bash
# Check scaling activities
aws autoscaling describe-scaling-activities --auto-scaling-group-name my-asg --max-items 20

# Check which process is causing termination
# Look at "Cause" field in scaling activities
```

---

**Q12: How do you implement blue/green deployments with ALB and ASG?**

**A:**

```
Blue ASG (current v1):
├── Target Group: blue-tg (registered with ALB listener)
├── Desired: 4 instances
└── Status: serving traffic

Green ASG (new v2):
├── Target Group: green-tg (not yet in listener)
├── Desired: 4 instances
└── Status: warming up, health checks passing

Cutover:
1. Switch ALB listener rule to point to green-tg
2. Monitor metrics (error rate, latency)
3. If OK → terminate blue ASG after bake time
4. If NOT OK → switch listener back to blue-tg (instant rollback)
```

**Weighted switching (canary):**
```bash
aws elbv2 modify-rule --rule-arn arn:... --actions '[{
  "Type": "forward",
  "ForwardConfig": {
    "TargetGroups": [
      {"TargetGroupArn": "blue-tg", "Weight": 90},
      {"TargetGroupArn": "green-tg", "Weight": 10}
    ]
  }
}]'
```

**Tricky**: If using sticky sessions, some users stay on blue even after switching to green. Must expire cookies or use different cookie names per version.

---

**Q13: ALB returns 502 Bad Gateway intermittently. How do you troubleshoot?**

**A:**

**502 means**: ALB received invalid response from target (or couldn't connect).

**Causes:**
1. **Target closed connection**: App closed keepalive before ALB expected
   - Fix: Set app keepalive > ALB idle timeout (default 60s)

2. **Target not listening on configured port**: Deployment in progress, port not ready
   - Check: Security group allows ALB → target on the port

3. **Target returning malformed HTTP**: Response not valid HTTP/1.1
   - Check: Target returning proper HTTP headers

4. **Health check passing but app failing**: Health endpoint works but other routes crash
   - Fix: More comprehensive health check

5. **SSL/TLS mismatch**: ALB sends HTTPS to target expecting HTTP (or vice versa)

**Debugging:**
```bash
# Check ALB access logs (enable if not already)
# Fields: request_processing_time, target_processing_time, response_processing_time

# If target_processing_time = -1 → ALB couldn't connect to target
# If response_processing_time = -1 → Target closed connection early

# Check target health
aws elbv2 describe-target-health --target-group-arn arn:...
```

---

**Q14: Explain cross-zone load balancing. What happens without it?**

**A:**

```
WITH cross-zone (ALB default: ON):
AZ-a (2 targets): Each gets 25% traffic
AZ-b (2 targets): Each gets 25% traffic
→ Even distribution regardless of AZ

WITHOUT cross-zone (NLB default: OFF):
AZ-a (2 targets): 50% traffic split between 2 = 25% each
AZ-b (6 targets): 50% traffic split between 6 = 8.3% each
→ AZ-a targets overloaded!
```

**Why NLB defaults to OFF:**
- Cross-zone creates inter-AZ traffic (data transfer charges)
- NLB handles millions of packets — inter-AZ cost can be significant
- For some use cases (regional failover), you WANT AZ-isolated routing

**Tricky**: When using NLB with cross-zone OFF, if one AZ loses all targets, the DNS record for that AZ is removed (zonal DNS). But DNS caching means some clients still send traffic to that AZ for a while → connection timeout.

---

**Q15: You need an ASG that scales based on SQS queue depth. How do you configure it?**

**A:**

```hcl
# Custom metric: messages per instance
resource "aws_cloudwatch_metric_alarm" "queue_depth" {
  alarm_name          = "sqs-backlog-per-instance"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  threshold           = 100

  metric_query {
    id          = "e1"
    expression  = "visible / instances"
    label       = "Backlog Per Instance"
    return_data = true
  }
  metric_query {
    id = "visible"
    metric {
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      period      = 60
      stat        = "Average"
      dimensions  = { QueueName = "my-queue" }
    }
  }
  metric_query {
    id = "instances"
    metric {
      metric_name = "GroupInServiceInstances"
      namespace   = "AWS/AutoScaling"
      period      = 60
      stat        = "Average"
      dimensions  = { AutoScalingGroupName = "my-asg" }
    }
  }
}
```

**Formula**: `Backlog per instance = ApproximateNumberOfMessagesVisible / InServiceInstances`
- Target: ~100 messages per instance (based on processing time)
- Scale out: When backlog per instance > 100
- Scale to zero: When queue is empty (set min capacity = 0)

**Tricky**: `ApproximateNumberOfMessagesVisible` updates every 1-5 minutes (not real-time). For faster scaling, publish a custom metric from your application that tracks actual processing rate.



---

## Additional Scenario-Based Tricky Questions

---

**Q16: Your ALB target group shows all targets as "healthy" but customers report intermittent 503 Service Unavailable errors. ALB access logs confirm 503s being returned. What's causing this?**

**A:**

**ALB 503 causes when targets are healthy:**
```
1. ALL TARGETS DEREGISTERING simultaneously:
   - During deployment, old targets deregister + new targets registering
   - Brief window where NO healthy targets exist
   - ALB returns 503 "Service Unavailable"
   
2. TARGET GROUP HAS NO REGISTERED TARGETS:
   - ASG scaled to 0 (scaling policy bug or manual error)
   - ALB has no targets → 503
   
3. CONNECTION DRAINING EXHAUSTED:
   - Deregistration delay: 300s (default)
   - During this time, target is "draining" — health check says healthy
   - But target is refusing new connections → 503 for new requests

4. ALB CAPACITY INSUFFICIENT (rare):
   - Sudden traffic spike → ALB hasn't scaled internally yet
   - ALB nodes overwhelmed → returns 503
   - Usually resolves in 1-5 minutes (ALB auto-scales)
   - Fix: Pre-warm ALB before expected spikes (contact AWS support)

5. LAMBDA TARGET THROTTLING:
   - ALB with Lambda target → Lambda throttled → 503
```

**Debugging:**
```bash
# Check ALB access logs for 503 patterns
aws s3 cp s3://alb-logs/AWSLogs/123456/elasticloadbalancing/us-east-1/2024/01/20/ ./logs/
zcat *.log.gz | awk '$9 == 503 {print $0}' | head -20
# Look at: target_status_code vs elb_status_code
# If target returns 503 → application problem
# If ELB returns 503 (no target response) → no healthy targets available

# Check target registration timeline
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Reason]'

# Check ASG desired/running counts
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names prod-asg \
  --query 'AutoScalingGroups[].[DesiredCapacity,Instances[].HealthStatus]'
```

**Fix for deployment-related 503s:**
```yaml
# Use minimum healthy percent + slow start
TargetGroup:
  Attributes:
    slow_start.duration_seconds: 60  # New targets get gradual traffic
    deregistration_delay.timeout_seconds: 30  # Faster drain (reduce window)

AutoScalingGroup:
  UpdatePolicy:
    MinInstancesInService: 2        # Always keep 2 healthy during deploy
    MaxBatchSize: 1                 # Replace 1 at a time
    PauseTime: PT5M                 # Wait 5 min between batches
    WaitOnResourceSignals: true     # Wait for app to signal ready
```

**Tricky**: ALB health checks and ASG health checks are DIFFERENT! ALB might say "healthy" (HTTP 200 on /health) while ASG says "unhealthy" (EC2 status check failed). Or vice versa. When ALB says healthy but returns 503, check if the target was just deregistered (draining state). During draining, the target still passes health checks (existing connections work) but new connections fail. The 503 window is the gap between "start draining" and "new target is ready."

---

**Q17: Your Auto Scaling Group keeps launching and terminating instances in a loop — scaling up then down every 5 minutes. CloudWatch CPU alarm triggers at 70%, ASG adds instances, CPU drops to 30%, ASG removes instances, CPU goes back to 70%. How do you stop this "flapping"?**

**A:**

**The oscillation problem:**
```
T0: 3 instances, CPU avg = 75% → alarm: SCALE UP
T1: 5 instances launched (total 8), CPU drops to 30%
T2: Cooldown ends → CPU avg = 30% < 40% → alarm: SCALE DOWN
T3: 5 instances terminated (back to 3), CPU avg = 75%
T4: Repeat forever...

This is classic control system oscillation — too aggressive scaling.
```

**Multi-layered fix:**
```bash
# Fix 1: Use Target Tracking instead of Simple/Step scaling
aws autoscaling put-scaling-policy --auto-scaling-group-name prod-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 60.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60,
    "DisableScaleIn": false
  }'
# Target tracking automatically handles oscillation
# Maintains CPU around 60% without aggressive add/remove cycles

# Fix 2: If using step scaling — add proper cooldowns
# Scale-out cooldown: 60s (respond quickly to load)
# Scale-in cooldown: 300s (wait before removing — prevent flap)

# Fix 3: Use predictive scaling for known patterns
aws autoscaling put-scaling-policy --auto-scaling-group-name prod-asg \
  --policy-name predictive-scaling \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [{
      "TargetValue": 60,
      "PredefinedMetricPairSpecification": {
        "PredefinedMetricType": "ASGCPUUtilization"
      }
    }],
    "Mode": "ForecastAndScale",
    "SchedulingBufferTime": 300
  }'
```

**Understanding cooldown vs stabilization:**
```
COOLDOWN (Simple/Step scaling):
- After scale-out: ignore alarms for X seconds
- Prevents: launching too many instances at once
- Default: 300 seconds

SCALE-IN PROTECTION:
- Instance can't be terminated for X minutes after launch
- Prevents: killing instances before they finish bootstrapping

WARM-UP:
- New instance's metrics excluded from aggregation for X seconds
- Prevents: new instance (0% CPU) from dragging average down → premature scale-in
```

**Tricky**: The root cause of flapping is usually that the ASG OVER-PROVISIONS on scale-out. If you need 5 instances at 70% CPU, and scaling adds 5 more (total 10), CPU drops to 35%. Then it removes 5, back to 70%. Fix: use step scaling with smaller steps (add 2, not 5) or target tracking which calculates the exact number needed. Also, `InstanceWarmup` in target tracking policies (default 300s) tells ASG to ignore new instance metrics until warm — without it, a cold instance shows 0% CPU → average drops → premature scale-in.

---

**Q18: You have a microservices architecture with 10 ALBs (one per service). Monthly ALB cost is $3,500. Your manager asks you to reduce networking costs. What are your options?**

**A:**

**Cost breakdown per ALB:**
```
ALB hourly charge: $0.0225/hour = $16.20/month
LCU charges: Based on new connections, active connections, bandwidth, rules
10 ALBs: $162/month (fixed) + LCU charges (~$3,300)

Main cost is LCU, not hourly charge.
LCU pricing: $0.008/LCU-hour
1 LCU = 25 new connections/s OR 3000 active connections OR 1GB/hour
```

**Cost reduction strategies:**
```bash
# Strategy 1: Consolidate ALBs using path-based routing
# Instead of 10 ALBs, use 1-2 ALBs with listener rules:
# /api/users/* → user-service target group
# /api/orders/* → order-service target group
# /api/payments/* → payment-service target group
# Savings: 8 ALBs × $16.20 = $130/month + shared LCU efficiency

# Strategy 2: Use NLB for non-HTTP services
# NLB: $0.006/NLCU-hour (25% cheaper than ALB LCU)
# For gRPC, TCP, or simple pass-through: NLB is cheaper
# ALB is only needed for: path routing, host routing, HTTP/2, WAF integration

# Strategy 3: Use API Gateway for low-traffic services
# API Gateway: $3.50 per million requests
# If service gets <1M requests/month: API Gateway < ALB
# Break-even: ~2M requests/month (below this, API GW is cheaper)

# Strategy 4: Kubernetes Ingress Controller (if using EKS)
# Single ALB + Ingress Controller handles ALL services
# AWS Load Balancer Controller creates ONE ALB with dynamic rules
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-alb  # Groups into single ALB!
spec:
  rules:
  - host: users.api.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: user-service
            port: 
              number: 80
```

**Comparison for 10 microservices:**
| Approach | Monthly Cost | Complexity |
|----------|-------------|------------|
| 10 separate ALBs | $3,500 | Low |
| 2 ALBs (consolidated) | $1,200 | Medium |
| 1 ALB + Ingress Controller | $800 | Medium |
| API Gateway (low traffic) | $200-500 | Low |

**Tricky**: Consolidating to fewer ALBs means one ALB handles more services. If that ALB has issues, ALL services are affected (increased blast radius). Production best practice: consolidate by criticality tier. Tier-1 (payment, auth): dedicated ALB. Tier-2 (catalog, search): shared ALB. Tier-3 (internal tools): API Gateway. Also, ALB listener rules are limited to 100 per ALB (default). With 10 services × 10 path rules each = 100 rules. Request increase if needed.

---
