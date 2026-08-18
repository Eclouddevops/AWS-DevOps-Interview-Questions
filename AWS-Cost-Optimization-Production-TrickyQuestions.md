# AWS Cost Optimization — Production Tricky Interview Q&A

## Table of Contents
1. [Compute Cost Optimization](#compute-cost-optimization)
2. [Storage & Data Transfer Costs](#storage--data-transfer-costs)
3. [Kubernetes Cost Management](#kubernetes-cost-management)
4. [Architecture Decisions That Save Money](#architecture-decisions-that-save-money)
5. [Cost Monitoring & Accountability](#cost-monitoring--accountability)

---

## Compute Cost Optimization

---

**Q1: Your AWS bill jumped from $50K to $120K/month. Management wants it back to $50K within 30 days. Where do you start?**

**A:**

**Immediate analysis (Day 1):**
```bash
# AWS Cost Explorer — group by service
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# Typical breakdown for compute-heavy workloads:
# EC2:           45% ($54K)  ← Look here first
# RDS:           20% ($24K)
# Data Transfer: 15% ($18K)  ← Often overlooked
# S3:             8% ($9.6K)
# NAT Gateway:    5% ($6K)   ← Hidden cost
# Other:          7% ($8.4K)
```

**Quick wins (save 40-60% in first week):**
```bash
# WIN 1: Right-size over-provisioned instances
# Find instances with <20% avg CPU
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUUtilization --period 86400 --statistics Average \
  --start-time $(date -d '7 days ago' --iso-8601) --end-time $(date --iso-8601) \
  --dimensions Name=InstanceId,Value=i-xxx

# AWS Compute Optimizer gives recommendations:
aws compute-optimizer get-ec2-instance-recommendations --instance-arns \
  arn:aws:ec2:us-east-1:123456:instance/i-xxx

# WIN 2: Delete forgotten resources
# Unattached EBS volumes:
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].[VolumeId,Size,CreateTime]' --output table
# 50 forgotten 500GB volumes = 50×$0.10×500 = $2,500/month!

# Unused Elastic IPs:
aws ec2 describe-addresses --query 'Addresses[?AssociationId==null]'
# $3.60/month each, but more importantly: indicates orphaned infrastructure

# Old snapshots:
aws ec2 describe-snapshots --owner-ids self --query \
  'Snapshots[?StartTime<`2023-06-01`].[SnapshotId,VolumeSize,StartTime]'

# WIN 3: Reserved Instances / Savings Plans for steady-state
# If running 20× m5.xlarge 24/7, you're paying On-Demand ($3,504/month each)
# 1-year RI: $2,340/month (33% savings)
# 3-year RI: $1,577/month (55% savings)
# 20 instances × savings = $20K-$38K/month saved!
```

**Tricky**: The biggest cost waste is usually NOT "expensive resources" — it's "forgotten resources." Dev/test environments left running, failed deployments that left ELBs/NAT GWs behind, snapshots that accumulate forever, and Lambda functions with provisioned concurrency that nobody uses. A weekly "garbage collection" job that finds untagged or unused resources saves more than any architectural change.

---

**Q2: You run 100 EC2 instances in production (steady workload). Your manager says "use Spot instances to save 70%." Is this a good idea? What's the production-safe approach?**

**A:**

**Spot reality check:**
```
Spot Instances:
✓ 60-90% cheaper than On-Demand
✗ Can be terminated with 2-MINUTE warning
✗ Specific instance types can be unavailable for hours/days
✗ Interruption rates vary: c5.large (5-10%), m5.xlarge (2-5%), p3.2xlarge (15%+)

For PRODUCTION (steady workload):
- DON'T use 100% Spot
- DO use a MIX: On-Demand + Reserved + Spot
```

**Production-safe Spot strategy:**
```yaml
# EKS with Karpenter — mixed instance types + purchase options
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
spec:
  requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: ["on-demand", "spot"]
  - key: node.kubernetes.io/instance-type
    operator: In
    values:  # DIVERSE instance types (avoid single-pool interruption)
    - m5.large
    - m5a.large
    - m5n.large
    - m5zn.large
    - m6i.large
    - m6a.large
    - c5.large
    - c5a.large
    - c6i.large
  # Karpenter automatically picks cheapest available

# Separate node groups by reliability needs:
# Critical (payments, auth): 100% On-Demand / Reserved
# Standard (API, web): 70% On-Demand, 30% Spot
# Batch (reports, ETL): 100% Spot with retries
```

**Handling Spot interruption gracefully:**
```bash
# AWS Node Termination Handler (DaemonSet)
# Detects: Spot interruption, scheduled maintenance, rebalance recommendation
# Action: Cordon node → drain pods (30s) → pods reschedule on healthy nodes

# Pod disruption budget ensures minimum availability during drain:
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: "80%"
  selector:
    matchLabels:
      app: myservice

# Application-level: handle SIGTERM gracefully
# Stop accepting new requests → finish in-flight → exit
```

**Cost calculation for 100 instances (m5.xlarge):**
| Strategy | Monthly Cost | Savings | Risk |
|----------|-------------|---------|------|
| 100% On-Demand | $14,016 | Baseline | None |
| 100% 1yr Reserved | $9,384 | 33% | Commit risk |
| 70 Reserved + 30 Spot | $7,566 | 46% | Low (Spot only for non-critical) |
| 50 Reserved + 30 OD + 20 Spot | $9,720 | 31% | Minimal |
| Savings Plan (Compute) | $8,750 | 38% | Flexible commitment |

**Tricky**: Spot instances work great for Kubernetes because K8s already handles pod rescheduling. BUT: if your pods have long startup times (>2 min), Spot interruptions cause extended degradation. Also, `r5.large` Spot in `us-east-1a` might be interrupted, but `r5.large` in `us-east-1b` is fine. Diversify across BOTH instance types AND AZs. Karpenter does this automatically; ASG-based solutions need multiple Launch Templates.

---

## Storage & Data Transfer Costs

---

**Q3: Your monthly AWS data transfer bill is $25,000. Where is the money going and how do you reduce it to under $5,000?**

**A:**

**Data transfer pricing (the hidden killer):**
```
FREE:
- Inbound from internet → AWS: FREE
- Same AZ, same VPC: FREE
- S3 to CloudFront (same region): FREE
- VPC endpoints (Gateway type for S3/DynamoDB): FREE

COSTS MONEY:
- Cross-AZ: $0.01/GB each direction ($0.02 round trip)
- Internet outbound: $0.09/GB (first 10TB), $0.085/GB (next 40TB)
- Cross-region: $0.02/GB
- NAT Gateway processing: $0.045/GB
- VPC peering cross-region: $0.02/GB
- CloudFront to internet: $0.085/GB (US/EU)
```

**Finding the $25K:**
```bash
# Enable VPC Flow Logs with traffic analysis
# Use AWS Cost Explorer → Group by "Usage Type"
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE","Values":["DataTransfer-Out-Bytes"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# Typical breakdown:
# Cross-AZ traffic (EKS): $8,000 (pods talk across AZs)
# NAT Gateway processing: $6,000 (S3/ECR through NAT)
# Internet outbound (API responses): $5,000
# Cross-region replication: $3,000
# VPC peering: $3,000
```

**Fixes:**
```bash
# FIX 1: Cross-AZ traffic — topology-aware routing ($8K → $2K)
# Kubernetes: prefer same-AZ pod communication
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/topology-mode: Auto  # Route to same-AZ pods first

# FIX 2: NAT Gateway → VPC Endpoints ($6K → $0.5K)
# S3 Gateway Endpoint = FREE (no data processing charge)
# ECR, CloudWatch, STS Interface Endpoints = small hourly fee but no per-GB charge
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 --vpc-endpoint-type Gateway

# FIX 3: CloudFront for API responses ($5K → $1.5K)
# Even for dynamic APIs, CloudFront reduces data transfer cost
# CloudFront egress: $0.085/GB vs EC2 egress: $0.09/GB
# Plus: caching reduces origin requests by 30-60%

# FIX 4: Compress responses (reduce bytes transferred)
# Enable gzip/brotli on ALB or application
# API responses typically compress 70-80%
# $5K × 0.25 = $1.25K after compression

# FIX 5: Cross-region → single region or S3 Replication class ($3K → $1K)
# Use S3 Replication Time Control only for critical data
# Batch replicate non-critical data during off-peak
```

**Tricky**: Cross-AZ data transfer in Kubernetes is the sneakiest cost. If you have 3 AZs with pods spread evenly, roughly 66% of pod-to-pod traffic crosses AZs (by probability). With 100TB/month internal traffic: 66TB × $0.02 = $1,320/month just for pods talking to each other! Topology-aware routing (`topology.kubernetes.io/zone`) keeps traffic in the same AZ when possible. But be careful — same-AZ routing reduces HA. If an AZ goes down, all traffic for that AZ's pods is lost. Balance cost vs resilience.

---

**Q4: Your company stores 500TB in S3 Standard. Monthly S3 bill is $12,000 for storage alone. How do you reduce it?**

**A:**

**S3 Storage class comparison:**
| Class | Cost/GB/month | Retrieval | Use Case |
|-------|--------------|-----------|----------|
| Standard | $0.023 | Instant | Hot data (accessed daily) |
| Intelligent-Tiering | $0.023 (+ $0.0025 monitoring) | Instant | Unknown access patterns |
| Standard-IA | $0.0125 | Instant (+ retrieval fee) | Accessed monthly |
| One Zone-IA | $0.01 | Instant (one AZ, less durable) | Reproducible data |
| Glacier Instant | $0.004 | Milliseconds | Quarterly access |
| Glacier Flexible | $0.0036 | 1-12 hours | Annual access |
| Glacier Deep Archive | $0.00099 | 12-48 hours | Compliance/archive |

**Analysis of 500TB:**
```bash
# Use S3 Storage Lens or S3 Analytics to understand access patterns
aws s3api put-bucket-analytics-configuration --bucket my-bucket \
  --id full-analysis --analytics-configuration '{
    "StorageClassAnalysis": {"DataExport": {"Destination": {"S3BucketDestination": {
      "Bucket": "arn:aws:s3:::analytics-output", "Format": "CSV"
    }}}}
  }'

# Typical findings:
# 50TB (10%): accessed daily → Standard ✓
# 100TB (20%): accessed weekly → Standard-IA
# 150TB (30%): not accessed in 30+ days → Glacier Instant
# 200TB (40%): not accessed in 180+ days → Glacier Deep Archive
```

**Lifecycle policy (automate transitions):**
```json
{
  "Rules": [{
    "ID": "OptimizeStorage",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER_IR"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "Expiration": {"Days": 2555}
  }]
}
```

**Cost after optimization:**
```
Before: 500TB × $0.023 = $11,500/month

After:
  50TB × $0.023 (Standard)      = $1,150
  100TB × $0.0125 (Standard-IA) = $1,250
  150TB × $0.004 (Glacier Inst) = $600
  200TB × $0.00099 (Deep Archive)= $198
  TOTAL = $3,198/month

SAVINGS: $8,302/month ($99,624/year!)
```

**Tricky**: Moving to Glacier saves on storage but RETRIEVAL costs can be brutal. Glacier Flexible retrieval: $0.03/GB for Expedited (minutes), $0.01/GB for Standard (hours). If you accidentally move 100TB of frequently-accessed data to Glacier and need it back: 100,000GB × $0.01 = $1,000 per retrieval! Always use S3 Analytics (wait 30 days for data) before creating lifecycle policies. Also, S3 has a 128KB minimum billable size for IA/Glacier — millions of tiny files (<128KB) in IA cost MORE than Standard.

---

## Kubernetes Cost Management

---

**Q5: Your EKS cluster runs 80 nodes (m5.xlarge) costing $28K/month. Actual CPU utilization averages 15%. Memory averages 35%. How do you optimize without impacting reliability?**

**A:**

**The waste analysis:**
```
80 × m5.xlarge: 4 vCPU, 16GB RAM each
Total capacity: 320 vCPU, 1280 GB RAM
Actual usage: 48 vCPU (15%), 448 GB RAM (35%)

You're paying for 320 vCPU but using 48 → 272 vCPU wasted
You're paying for 1280 GB but using 448 → 832 GB wasted
```

**Why utilization is so low (common causes):**
```
1. Over-requested resources:
   Pod requests: cpu=1000m, memory=2Gi
   Pod actual:   cpu=100m,  memory=400Mi
   → 10x CPU over-requested, 5x memory over-requested
   
2. Scaling for peak (running peak capacity 24/7):
   Peak traffic: 9AM-6PM (needs 60 nodes)
   Off-peak: 6PM-9AM (needs 20 nodes)
   Running 80 nodes 24/7 = 60 wasted node-hours per night

3. Non-production in same cluster:
   Dev/staging namespaces consuming production-grade resources
```

**Optimization strategy:**
```bash
# Step 1: Right-size pod requests using VPA recommendations
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml

# VPA in "recommend" mode (doesn't change anything, just suggests)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myservice-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myservice
  updatePolicy:
    updateMode: "Off"  # Just recommend, don't auto-apply

# Check recommendations:
kubectl describe vpa myservice-vpa
# "Target: cpu=150m, memory=512Mi" vs current request "cpu=1000m, memory=2Gi"

# Step 2: Implement Cluster Autoscaler scale-down
# cluster-autoscaler/config:
--scale-down-enabled=true
--scale-down-utilization-threshold=0.5  # Scale down if node <50% utilized
--scale-down-unneeded-time=10m          # Wait 10 min before removing node
--skip-nodes-with-system-pods=false

# Step 3: Time-based scaling for predictable patterns
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  behavior:
    scaleDown:
      policies:
      - type: Percent
        value: 50
        periodSeconds: 300

# Or use KEDA for time-based:
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: "0 9 * * *"    # Scale up at 9AM
      end: "0 18 * * *"     # Scale down at 6PM
      desiredReplicas: "50"

# Step 4: Use smaller, right-sized node types
# Instead of m5.xlarge (4CPU, 16GB): $140/month
# Use m5.large (2CPU, 8GB): $70/month
# More smaller nodes = better bin-packing = less waste
```

**Expected savings:**
```
Current: 80 × m5.xlarge = $28,000/month

After optimization:
- Right-size requests → need 40 nodes instead of 80
- Use m5.large instead of xlarge → $70 vs $140
- Time-based scaling → 50% off-peak reduction (20 nodes at night)
- Mix in Spot for non-critical (30% of fleet)

Optimized: ~30 On-Demand m5.large + 10 Spot = ~$3,000/month
SAVINGS: $25,000/month!
```

**Tricky**: Right-sizing pod requests is the SINGLE biggest lever for Kubernetes cost. If pods request 1 CPU but use 0.1 CPU, you need 10x more nodes than necessary. But reducing requests too aggressively causes throttling (CPU) or OOMKills (memory). The safe approach: set requests = p95 actual usage + 20% buffer. Use VPA in "Off" mode for 2 weeks to collect data before making changes. Never right-size memory below p99 — a single OOMKill is worse than paying 10% extra.

---

## Cost Monitoring & Accountability

---

**Q6: Your company has 15 teams sharing one AWS account. Nobody takes responsibility for cost because they can't see their own spend. How do you implement cost accountability?**

**A:**

**Tagging strategy (the foundation):**
```bash
# Required tags for ALL resources (enforced via SCP/AWS Config):
# - Team: "payments", "search", "infrastructure"
# - Environment: "production", "staging", "development"
# - Service: "api-gateway", "user-service", "recommendation-engine"
# - CostCenter: "CC-1234"

# Enforce tagging via AWS Organizations SCP:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "RequireTags",
    "Effect": "Deny",
    "Action": ["ec2:RunInstances", "rds:CreateDBInstance"],
    "Resource": "*",
    "Condition": {
      "Null": {"aws:RequestTag/Team": "true"}
    }
  }]
}

# AWS Config rule to find untagged resources:
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "required-tags",
  "Source": {"Owner": "AWS", "SourceIdentifier": "REQUIRED_TAGS"},
  "InputParameters": "{\"tag1Key\":\"Team\",\"tag2Key\":\"Environment\"}"
}'
```

**Cost allocation for Kubernetes (per-team):**
```bash
# Use Kubecost or OpenCost for per-namespace/label cost breakdown
# Install Kubecost:
helm install kubecost kubecost/cost-analyzer --namespace kubecost

# Query per-team costs:
curl http://kubecost:9090/model/allocation?window=lastMonth&aggregate=namespace
# Returns: { "payments-ns": "$3,200", "search-ns": "$5,400", ... }

# Label-based allocation for shared namespaces:
curl http://kubecost:9090/model/allocation?window=lastMonth&aggregate=label:team
```

**Weekly cost reports (automated):**
```python
# Lambda function: sends weekly cost report per team to Slack
import boto3
import json
import requests

def lambda_handler(event, context):
    ce = boto3.client('ce')
    
    response = ce.get_cost_and_usage(
        TimePeriod={'Start': '2024-01-01', 'End': '2024-01-31'},
        Granularity='MONTHLY',
        Metrics=['BlendedCost'],
        GroupBy=[{'Type': 'TAG', 'Key': 'Team'}]
    )
    
    message = "📊 *Monthly AWS Cost by Team:*\n"
    for group in response['ResultsByTime'][0]['Groups']:
        team = group['Keys'][0].replace('Team$', '')
        cost = float(group['Metrics']['BlendedCost']['Amount'])
        message += f"• *{team}*: ${cost:,.0f}\n"
    
    # Send to Slack
    requests.post(SLACK_WEBHOOK, json={"text": message})
```

**Budget alerts with auto-action:**
```bash
# Per-team budgets with automatic notification + action
aws budgets create-budget --account-id 123456 --budget '{
  "BudgetName": "payments-team-monthly",
  "BudgetLimit": {"Amount": "5000", "Unit": "USD"},
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {"TagKeyValue": ["user:Team$payments"]}
}' --notifications-with-subscribers '[{
  "Notification": {
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 80
  },
  "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "payments-team@company.com"}]
}]'
```

**Tricky**: AWS Cost Allocation Tags take 24-48 hours to appear in Cost Explorer after activation. If you tag a resource today, you won't see cost data attributed to that tag until 2 days later. Also, not all resources support tagging at creation — some require a separate `tag-resource` API call. The biggest gap: data transfer costs are nearly impossible to attribute to specific teams because network traffic isn't tagged. You need VPC Flow Logs + custom analysis to allocate network costs.

---
