# AWS DevOps Engineer - Roles, Responsibilities & Industry-Level Interview Questions

## Roles & Responsibilities

AWS infrastructure, ensuring scalability, reliability, security, and high availability for mission-critical applications.

### Key Responsibilities
- Own and manage AWS infrastructure, including EC2, Auto Scaling Groups, and Application Load Balancers (ALB).
- Design, build, and maintain CI/CD deployment pipelines using AWS CodeDeploy.
- Set up monitoring, logging, and alerting using Amazon CloudWatch, CloudWatch Alarms, and SNS.
- Manage DNS and traffic routing using Amazon Route 53.
- Administer and optimize Elasticsearch clusters for production search and indexing.
- Manage Amazon S3 and Firebase Storage, including storage optimization, security policies, and cost management.
- Manage Amazon Aurora/RDS databases, including backups, scaling, monitoring, and performance tuning.
- Ensure high availability and resilience for real-time, high-concurrency production workloads.
- Drive infrastructure security, cost optimization, incident response, and production support.
- Continuously improve infrastructure through automation, process optimization, and modernization.
- Create and maintain infrastructure documentation, runbooks, disaster recovery plans, and post-incident reviews.
- Support engineering teams with backend deployments and RabbitMQ/Redis infrastructure.
- Build and maintain Infrastructure as Code (Terraform or CloudFormation).

### Required Skills
- 8+ years of hands-on AWS infrastructure and DevOps experience.
- Strong expertise in EC2, Auto Scaling, Application Load Balancer (ALB), CodeDeploy, CloudWatch, and Route 53.
- Experience managing Elasticsearch in production environments.
- Hands-on experience managing Amazon S3 and cloud storage at scale.
- Strong experience with Amazon Aurora/RDS, including backup, scaling, replication, and performance tuning.
- Experience supporting deployments for Python/Flask (or similar) backend applications.
- Strong production incident response and troubleshooting experience.
- Hands-on experience with Infrastructure as Code (Terraform or CloudFormation).
- Proven experience driving infrastructure automation, reliability improvements, and cost optimization.
- Experience managing RabbitMQ and Redis in production.
- Experience working on logistics, delivery, quick-commerce, or other real-time, high-concurrency platforms.

### Good to Have
- AWS Certifications (Solutions Architect, SysOps Administrator, DevOps Engineer, or equivalent).
- Experience with Docker, Kubernetes, or Amazon ECS.

---

## Industry-Level Interview Questions & Answers

---

## Section 1: EC2, Auto Scaling Groups & Application Load Balancers (ALB)

---


### Q1: Your Auto Scaling Group is launching instances but ALB health checks are failing. How do you troubleshoot this?

**Answer:**
This is a common production issue. Here's the systematic approach:

1. **Check Health Check Configuration:**
   - Verify the health check path (e.g., `/health`) returns HTTP 200.
   - Ensure the health check port matches the application port.
   - Review health check thresholds (healthy/unhealthy threshold counts and intervals).

2. **Security Group Analysis:**
   - Confirm the ALB security group allows outbound traffic to the instance port.
   - Confirm the instance security group allows inbound traffic from the ALB security group.

3. **Application-Level Checks:**
   - SSH into the instance and verify the application is running (`systemctl status`, `curl localhost:port/health`).
   - Check application logs for startup failures.
   - Verify the instance has completed its UserData/bootstrap script.

4. **Timing Issues:**
   - Increase the `HealthCheckGracePeriod` on the ASG if the application takes time to start.
   - Consider using lifecycle hooks to delay health checks until the app is ready.

5. **Target Group Configuration:**
   - Verify target group protocol matches (HTTP vs HTTPS).
   - Check if the target group is using the correct VPC and subnets.

**Industry Best Practice:** Implement a dedicated `/health` endpoint that checks database connectivity, cache availability, and critical dependencies before returning 200.

---

### Q2: How do you implement a zero-downtime deployment strategy with ALB and Auto Scaling Groups?

**Answer:**

**Rolling Deployment Approach:**
```
1. Create a new Launch Template version with updated AMI/configuration.
2. Update the ASG to use the new Launch Template version.
3. Set Instance Refresh with:
   - MinHealthyPercentage: 90%
   - InstanceWarmup: 300 seconds
4. ASG replaces instances in batches, maintaining capacity.
```

**Blue-Green Deployment Approach:**
```
1. Create a new ASG (Green) with the new application version.
2. Attach the Green ASG to the same Target Group or a new Target Group.
3. Use ALB weighted target groups to shift traffic gradually:
   - 10% to Green, 90% to Blue
   - 50/50
   - 100% to Green
4. Monitor error rates and latency during traffic shift.
5. Decommission the Blue ASG after validation.
```

**Key Considerations:**
- Connection draining: Set deregistration delay (default 300s) to allow in-flight requests to complete.
- Sticky sessions: If enabled, plan for session migration or use external session stores (Redis/ElastiCache).
- Pre-warming: For predictable traffic spikes, use Predictive Scaling or pre-warm ALB by contacting AWS support.

---

### Q3: An EC2 instance in your ASG is showing high CPU utilization (95%+) but ASG is not scaling out. What could be wrong?

**Answer:**

**Possible Root Causes:**

1. **Scaling Policy Configuration:**
   - Check if the scaling policy uses `Average` vs `Maximum` metric. If using Average across all instances, one hot instance may not trigger scaling.
   - Verify the CloudWatch alarm threshold and evaluation periods.
   - Confirm the cooldown period hasn't blocked scaling actions.

2. **ASG Limits:**
   - Check if the ASG has reached `MaxSize`. If MaxSize = current capacity, no scale-out occurs.
   - Verify account-level EC2 instance limits (service quotas).

3. **CloudWatch Metrics:**
   - Confirm detailed monitoring is enabled (1-minute intervals vs 5-minute).
   - Check if the alarm is in `ALARM` state or stuck in `INSUFFICIENT_DATA`.

4. **Scaling Activity:**
   - Review ASG Activity History for failed launch attempts.
   - Check if there are suspended scaling processes.

5. **Instance Type Availability:**
   - The specified instance type may not be available in the AZ.
   - Solution: Use mixed instance policy with multiple instance types.

**Resolution:**
```bash
# Check ASG scaling activities
aws autoscaling describe-scaling-activities --auto-scaling-group-name my-asg

# Check suspended processes
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-name my-asg --query 'AutoScalingGroups[].SuspendedProcesses'

# Verify CloudWatch alarm state
aws cloudwatch describe-alarms --alarm-names "HighCPU-alarm"
```

---

### Q4: How do you design an ALB architecture for a multi-tenant SaaS application handling 100,000+ concurrent connections?

**Answer:**

**Architecture Design:**

1. **ALB Configuration:**
   - Enable cross-zone load balancing for even distribution.
   - Configure idle timeout based on application needs (default 60s, increase for WebSocket).
   - Enable HTTP/2 for multiplexed connections.
   - Set up WAF integration for DDoS protection.

2. **Routing Strategy:**
   - Use host-based routing: `tenant1.app.com` → Target Group 1.
   - Use path-based routing: `/api/v1/*` → API Target Group, `/static/*` → Static Target Group.
   - Implement weighted routing for canary deployments.

3. **Target Group Design:**
   - Separate target groups per microservice.
   - Enable slow start mode (30-90s) for new instances.
   - Configure appropriate health check intervals (10s for critical services).

4. **High Availability:**
   - Deploy ALB across minimum 3 AZs.
   - Pre-warm ALB for expected traffic spikes (contact AWS support or use predictive patterns).
   - Implement connection draining with 30-60s deregistration delay.

5. **Security:**
   - Terminate TLS at ALB (ACM certificates).
   - Enable access logs to S3 for audit and troubleshooting.
   - Restrict backend security groups to ALB-only traffic.

**Scaling Considerations for 100K+ Connections:**
- ALB automatically scales, but sudden spikes need pre-warming.
- Use Global Accelerator for global traffic distribution.
- Implement request rate limiting at ALB using WAF rules.

---


## Section 2: CI/CD Pipelines & AWS CodeDeploy

---

### Q5: Design a CI/CD pipeline for deploying a Python/Flask application using CodeDeploy with rollback capability.

**Answer:**

**Pipeline Architecture:**
```
GitHub/CodeCommit → CodePipeline → CodeBuild (Test/Build) → CodeDeploy → EC2/ASG
```

**AppSpec.yml Configuration:**
```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /opt/myapp
hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
      timeout: 60
      runas: root
  AfterInstall:
    - location: scripts/install_dependencies.sh
      timeout: 120
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 60
      runas: root
  ValidateService:
    - location: scripts/health_check.sh
      timeout: 120
      runas: root
```

**Deployment Configuration:**
```
- Deployment Type: In-place or Blue/Green
- Deployment Config: CodeDeployDefault.OneAtATime (safest) or Custom (25% at a time)
- Auto Rollback: Enable on deployment failure AND alarm threshold breach
- Minimum Healthy Hosts: 75%
```

**Rollback Strategy:**
1. **Automatic Rollback Triggers:**
   - Deployment failure (any lifecycle hook fails)
   - CloudWatch alarm breach (5xx errors > 5%, latency > 2s)
2. **Manual Rollback:** Redeploy previous revision from CodeDeploy console/CLI.
3. **Blue/Green Rollback:** Simply reroute traffic back to original target group.

**Best Practices:**
- Use `ValidateService` hook to run smoke tests post-deployment.
- Implement canary deployments: deploy to 10% first, wait 10 minutes, then deploy to remaining.
- Store deployment artifacts in S3 with versioning enabled.
- Use parameter store for environment-specific configurations.

---

### Q6: How do you handle database migrations in a CI/CD pipeline without causing downtime?

**Answer:**

**Strategy: Expand-Contract Pattern**

**Phase 1 - Expand (Backward Compatible):**
```
1. Add new columns/tables without removing old ones.
2. Deploy application code that works with BOTH old and new schema.
3. Run migration: ALTER TABLE ADD COLUMN (non-blocking in Aurora/PostgreSQL).
4. Application writes to both old and new columns.
```

**Phase 2 - Migrate Data:**
```
1. Backfill existing data to new columns using batch scripts.
2. Validate data integrity.
3. Run reconciliation checks.
```

**Phase 3 - Contract (Remove Old):**
```
1. Deploy application code that only uses new schema.
2. Remove old columns/tables after verification period (7-14 days).
```

**Implementation in Pipeline:**
```yaml
# CodeBuild buildspec.yml
phases:
  pre_build:
    commands:
      - python manage.py db check  # Verify migration is backward compatible
  build:
    commands:
      - python manage.py db migrate  # Run migration
      - python manage.py db verify   # Validate migration
  post_build:
    commands:
      - python manage.py db smoke_test  # Smoke test against new schema
```

**Critical Rules:**
- Never rename columns in a single deployment.
- Never drop columns that running code depends on.
- Use `pt-online-schema-change` or `gh-ost` for large table alterations in MySQL.
- For Aurora PostgreSQL, use `CREATE INDEX CONCURRENTLY` for non-blocking index creation.
- Always have a rollback migration script ready.

---

### Q7: Your CodeDeploy deployment is stuck in "In Progress" state for over 30 minutes. How do you diagnose and resolve?

**Answer:**

**Diagnostic Steps:**

1. **Check Deployment Events:**
```bash
aws deploy get-deployment --deployment-id d-XXXXXXXXX
aws deploy list-deployment-instances --deployment-id d-XXXXXXXXX
```

2. **Common Causes:**
   - **CodeDeploy Agent Not Running:** SSH to instance, check `sudo service codedeploy-agent status`.
   - **Lifecycle Hook Timeout:** A script in BeforeInstall/AfterInstall is hanging.
   - **IAM Role Issues:** Instance profile doesn't have permission to pull artifacts from S3.
   - **Network Issues:** Instance cannot reach CodeDeploy endpoints (check VPC endpoints/NAT gateway).
   - **Disk Space:** Instance ran out of disk space during artifact extraction.

3. **Log Analysis:**
```bash
# CodeDeploy agent logs
tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log

# Deployment logs
cat /opt/codedeploy-agent/deployment-root/<deployment-group-id>/<deployment-id>/logs/scripts.log
```

4. **Resolution:**
   - Stop the stuck deployment: `aws deploy stop-deployment --deployment-id d-XXX`
   - If agent is unresponsive, restart: `sudo service codedeploy-agent restart`
   - Fix the root cause and redeploy.

**Prevention:**
- Set appropriate timeouts in AppSpec hooks.
- Monitor CodeDeploy agent health with CloudWatch agent.
- Use lifecycle hook scripts that are idempotent.
- Implement proper error handling and logging in deployment scripts.

---


## Section 3: Amazon CloudWatch, Alarms & SNS

---

### Q8: Design a comprehensive monitoring strategy for a production e-commerce platform using CloudWatch.

**Answer:**

**Monitoring Layers:**

**1. Infrastructure Metrics (EC2/ASG):**
```
- CPUUtilization > 80% (Warning), > 95% (Critical)
- MemoryUtilization > 85% (requires CloudWatch Agent)
- DiskSpaceUtilization > 80%
- NetworkIn/NetworkOut anomaly detection
- StatusCheckFailed (immediate alert)
```

**2. Application Metrics (Custom):**
```python
import boto3
cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='MyApp/Production',
    MetricData=[
        {
            'MetricName': 'OrderProcessingTime',
            'Value': processing_time_ms,
            'Unit': 'Milliseconds',
            'Dimensions': [
                {'Name': 'Service', 'Value': 'OrderService'},
                {'Name': 'Environment', 'Value': 'Production'}
            ]
        },
        {
            'MetricName': 'FailedPayments',
            'Value': 1,
            'Unit': 'Count'
        }
    ]
)
```

**3. ALB Metrics:**
```
- HTTPCode_Target_5XX_Count > 10/minute (Critical)
- TargetResponseTime > 2s (Warning), > 5s (Critical)
- UnHealthyHostCount > 0 (Warning)
- ActiveConnectionCount (anomaly detection)
- RejectedConnectionCount > 0 (Critical)
```

**4. Database Metrics (Aurora/RDS):**
```
- CPUUtilization > 80%
- FreeableMemory < 1GB
- DatabaseConnections > 80% of max
- ReadLatency/WriteLatency > 20ms
- AuroraReplicaLag > 100ms
```

**SNS Alert Routing:**
```
Critical → PagerDuty (immediate page) + Slack #incidents
Warning  → Slack #monitoring + Email to on-call team
Info     → CloudWatch Dashboard only
```

**CloudWatch Dashboard Design:**
- Overview: Request rates, error rates, latency P50/P95/P99
- Infrastructure: CPU, Memory, Network across all ASGs
- Database: Connections, Latency, Replication lag
- Business: Orders/min, Revenue/hour, Cart abandonment rate

---

### Q9: How do you implement CloudWatch Composite Alarms to reduce alert fatigue?

**Answer:**

**Problem:** Individual alarms for CPU, Memory, and Disk generate too many alerts during maintenance windows or benign spikes.

**Solution: Composite Alarms**

```bash
# Create individual alarms
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-Production" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3

aws cloudwatch put-metric-alarm \
  --alarm-name "HighMemory-Production" \
  --metric-name mem_used_percent \
  --namespace CWAgent \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3

# Create composite alarm - only fires when BOTH conditions are true
aws cloudwatch put-composite-alarm \
  --alarm-name "Critical-Resource-Exhaustion" \
  --alarm-rule "ALARM(HighCPU-Production) AND ALARM(HighMemory-Production)" \
  --alarm-actions "arn:aws:sns:us-east-1:123456789:Critical-Alerts" \
  --alarm-description "Both CPU and Memory are critical - likely resource exhaustion"
```

**Advanced Pattern - Service Health Composite:**
```
Rule: (ALARM(5xxErrors) OR ALARM(HighLatency)) AND NOT ALARM(MaintenanceWindow)
```

This ensures alerts are suppressed during planned maintenance.

**Best Practices:**
- Use composite alarms for page-worthy alerts only.
- Implement maintenance window suppression alarms.
- Layer alarms: Individual → Composite → SNS Topic → PagerDuty/Slack.
- Use anomaly detection alarms for metrics without obvious thresholds.

---

### Q10: Your team is receiving 200+ CloudWatch alerts daily. How do you optimize the alerting strategy?

**Answer:**

**Assessment Phase:**
1. Categorize all alerts by: actionable vs. informational, frequency, service, and severity.
2. Identify noisy alarms (those that auto-resolve within 5 minutes).
3. Track Mean Time to Acknowledge (MTTA) and Mean Time to Resolve (MTTR).

**Optimization Strategy:**

**1. Alarm Tuning:**
```
Before: CPU > 80% for 1 period (5 min) → Alert
After:  CPU > 90% for 3 consecutive periods (15 min) → Alert
```

**2. Tiered Alert Routing:**
```
P1 (Page): Service down, data loss risk → PagerDuty
P2 (Urgent): Degraded performance, failing health checks → Slack #urgent
P3 (Warning): Approaching thresholds → Slack #monitoring
P4 (Info): Auto-healing events → Dashboard/Email digest
```

**3. Implement Auto-Remediation:**
```python
# Lambda triggered by CloudWatch Alarm via SNS
def lambda_handler(event, context):
    alarm_name = event['Records'][0]['Sns']['Subject']
    
    if 'DiskSpace' in alarm_name:
        # Auto-clean logs older than 7 days
        cleanup_old_logs(instance_id)
    elif 'UnhealthyHost' in alarm_name:
        # Terminate unhealthy instance (ASG will replace)
        terminate_instance(instance_id)
```

**4. Consolidation:**
- Replace 10 individual instance alarms with 1 ASG-level alarm using metric math.
- Use CloudWatch Metric Insights for cross-account/cross-region aggregation.

**Result Target:** Reduce from 200+ daily alerts to <20 actionable alerts.

---


## Section 4: Amazon Route 53 - DNS & Traffic Routing

---

### Q11: How do you implement a multi-region active-active architecture using Route 53?

**Answer:**

**Architecture Design:**

```
Users → Route 53 (Latency-based routing)
         ├── us-east-1: ALB → ASG → Aurora Global DB (Primary)
         └── eu-west-1: ALB → ASG → Aurora Global DB (Secondary)
```

**Route 53 Configuration:**

```bash
# Primary region record
aws route53 change-resource-record-sets --hosted-zone-id ZXXXXX --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "api.myapp.com",
      "Type": "A",
      "SetIdentifier": "us-east-1",
      "Region": "us-east-1",
      "AliasTarget": {
        "HostedZoneId": "Z35SXDOTRQ7X7K",
        "DNSName": "alb-us-east-1.elb.amazonaws.com",
        "EvaluateTargetHealth": true
      }
    }
  }]
}'
```

**Health Check Configuration:**
```
- Type: HTTPS
- Path: /health
- Request Interval: 10 seconds (Fast)
- Failure Threshold: 2
- Regions: All Route 53 health check regions
- Enable SNS notification on health check failure
```

**Failover Behavior:**
1. Route 53 health checks monitor each region's ALB endpoint.
2. If primary region fails health check, traffic automatically routes to healthy region.
3. TTL set to 60s for fast failover (balance between speed and DNS cache hit ratio).

**Key Considerations:**
- Use `EvaluateTargetHealth: true` for automatic failover.
- Implement cross-region data replication (Aurora Global Database, DynamoDB Global Tables).
- Consider data consistency models (eventual vs. strong consistency).
- Test failover regularly with Game Days.

---

### Q12: How do you migrate a domain from an external registrar to Route 53 with zero downtime?

**Answer:**

**Migration Steps:**

**Phase 1 - Preparation (Day 1-3):**
```
1. Export all DNS records from current provider.
2. Create hosted zone in Route 53.
3. Import all records (A, CNAME, MX, TXT, SRV, etc.).
4. Lower TTL on all records to 60-300 seconds (at current provider).
5. Wait 48 hours for TTL changes to propagate.
```

**Phase 2 - DNS Cutover (Day 4):**
```
1. Verify all records in Route 53 match current provider.
2. Update nameservers at the registrar to Route 53 NS records.
3. Monitor resolution using:
   - dig @ns-xxx.awsdns-xx.com api.myapp.com
   - Multiple DNS lookup tools (whatsmydns.net)
4. Verify MX records for email delivery.
```

**Phase 3 - Domain Transfer (Day 5-12):**
```
1. Unlock domain at current registrar.
2. Get authorization/EPP code.
3. Initiate transfer in Route 53.
4. Approve transfer confirmation email.
5. Wait for transfer completion (5-7 days).
```

**Phase 4 - Post-Migration:**
```
1. Restore TTL values to production levels (300-3600 seconds).
2. Enable DNSSEC if required.
3. Set up Route 53 health checks.
4. Configure CloudWatch alarms for DNS query failures.
```

**Zero Downtime Guarantee:**
- Keep old DNS records active until NS propagation completes.
- Both old and new nameservers serve correct records during transition.
- Monitor for DNS resolution failures across global checking points.

---

### Q13: Explain Route 53 routing policies and when to use each in production.

**Answer:**

| Routing Policy | Use Case | Production Example |
|---|---|---|
| **Simple** | Single resource | Internal tools, dev environments |
| **Weighted** | Traffic distribution/Canary | Send 5% traffic to new version |
| **Latency** | Multi-region performance | Serve users from nearest region |
| **Failover** | Active-Passive DR | Primary in us-east-1, DR in us-west-2 |
| **Geolocation** | Compliance/Content localization | EU users → EU servers (GDPR) |
| **Geoproximity** | Fine-tuned geo routing with bias | Shift traffic between regions gradually |
| **Multivalue Answer** | Simple load balancing with health checks | Distribute across multiple IPs |

**Real-World Scenario - Quick Commerce Platform:**
```
1. Geolocation: Route Indian users to ap-south-1, US users to us-east-1
2. Within each region, use Weighted routing:
   - 95% → Production (stable)
   - 5% → Canary (new release)
3. Failover configured as backup:
   - If primary region health check fails → route to DR region
```

**Advanced Pattern - Blue/Green with Route 53:**
```bash
# Shift traffic gradually
aws route53 change-resource-record-sets --change-batch '{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "blue",
        "Weight": 0,
        "AliasTarget": {"DNSName": "alb-blue.elb.amazonaws.com", ...}
      }
    },
    {
      "Action": "UPSERT", 
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "green",
        "Weight": 100,
        "AliasTarget": {"DNSName": "alb-green.elb.amazonaws.com", ...}
      }
    }
  ]
}'
```

---


## Section 5: Elasticsearch Cluster Management

---

### Q14: Your Elasticsearch cluster is experiencing high latency during peak hours. How do you diagnose and resolve?

**Answer:**

**Diagnostic Framework:**

**1. Cluster Health Check:**
```bash
GET _cluster/health
GET _cluster/stats
GET _nodes/stats
GET _cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent
```

**2. Common Causes & Solutions:**

| Symptom | Cause | Solution |
|---|---|---|
| Yellow/Red cluster status | Unassigned shards | Increase nodes, fix disk space |
| High search latency | Too many shards per index | Reduce shard count, use rollover |
| High indexing latency | Bulk queue full | Increase refresh interval, optimize mapping |
| High JVM heap usage | Large aggregations, field data | Increase heap (max 30.5GB), use doc_values |
| Disk I/O bottleneck | Too many merges | Use SSD (gp3/io2), reduce merge policy |

**3. Performance Optimization:**
```json
// Index settings optimization
PUT /my-index/_settings
{
  "index": {
    "refresh_interval": "30s",
    "number_of_replicas": 1,
    "translog.durability": "async",
    "translog.sync_interval": "30s"
  }
}
```

**4. Shard Strategy:**
- Target shard size: 20-50 GB per shard.
- Shards per node: Keep below 20 shards per GB of heap.
- Use Index Lifecycle Management (ILM) for time-series data.

**5. Query Optimization:**
- Use `filter` context instead of `query` context where scoring isn't needed.
- Implement query result caching.
- Avoid deep pagination (use `search_after` instead of `from/size`).
- Use `routing` for tenant-specific queries to reduce shard fan-out.

---

### Q15: Design an Elasticsearch architecture for a logistics platform processing 50,000 events/second.

**Answer:**

**Cluster Architecture:**

```
Data Nodes:    6x r5.2xlarge (Hot) + 3x r5.xlarge (Warm) + 2x r5.large (Cold)
Master Nodes:  3x m5.large (Dedicated)
Coordinator:   2x c5.xlarge (Dedicated for searches)
```

**Index Strategy:**
```
# Time-based indices with ILM
delivery-events-2024.01.15  (Hot - SSD, 1 primary + 1 replica)
delivery-events-2024.01.14  (Warm - HDD, 1 primary + 1 replica, force-merged)
delivery-events-2024.01.01  (Cold - Frozen, searchable snapshots)
```

**ILM Policy:**
```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "40gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "shrink": {"number_of_shards": 1},
          "forcemerge": {"max_num_segments": 1},
          "allocate": {"require": {"data": "warm"}}
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {"require": {"data": "cold"}}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {"delete": {}}
      }
    }
  }
}
```

**Ingestion Pipeline:**
```
Application → Kafka/RabbitMQ → Logstash/Vector → Elasticsearch
                                    ↓
                            (Batch: 5000 docs, Flush: 5s)
```

**Key Design Decisions:**
- Use dedicated master nodes to prevent split-brain.
- Dedicated coordinator nodes for heavy search workloads.
- Hot-Warm-Cold architecture for cost optimization.
- Kafka as buffer to handle back-pressure during indexing spikes.
- Cross-cluster replication (CCR) for disaster recovery.

---

### Q16: How do you handle Elasticsearch cluster upgrades in production with zero downtime?

**Answer:**

**Rolling Upgrade Process:**

**Pre-Upgrade Checklist:**
```bash
# 1. Check cluster health
GET _cluster/health  # Must be GREEN

# 2. Disable shard allocation
PUT _cluster/settings
{"persistent": {"cluster.routing.allocation.enable": "primaries"}}

# 3. Stop non-essential indexing (if possible)
# 4. Flush all indices
POST _flush/synced

# 5. Check deprecation API
GET _migration/deprecations
```

**Upgrade Steps (Per Node):**
```
1. Disable shard allocation (already done).
2. Stop the node.
3. Upgrade Elasticsearch on the node.
4. Start the node and verify it joins the cluster.
5. Re-enable shard allocation.
6. Wait for cluster to return to GREEN.
7. Repeat for next node.
```

**Node Upgrade Order:**
```
1. Non-master-eligible data nodes first.
2. Master-eligible nodes (one at a time).
3. Dedicated master nodes last.
```

**Post-Upgrade Validation:**
```bash
GET _cluster/health
GET _cat/recovery?active_only=true
GET _nodes  # Verify all nodes on new version
# Run search and indexing smoke tests
```

**Rollback Plan:**
- If upgrade fails on a node, reinstall previous version.
- If cluster issues arise, use snapshot/restore.
- Always take a full snapshot before starting upgrade.

---


## Section 6: Amazon S3 & Storage Management

---

### Q17: Design a cost-optimized S3 storage strategy for a platform handling 500TB of data with varying access patterns.

**Answer:**

**Storage Tier Strategy:**

| Data Type | Storage Class | Access Pattern | Cost Savings |
|---|---|---|---|
| Active uploads (0-30 days) | S3 Standard | Frequent | Baseline |
| Recent data (30-90 days) | S3 Standard-IA | Infrequent | 45% |
| Archive data (90-365 days) | S3 Glacier Instant Retrieval | Rare, fast access needed | 68% |
| Compliance data (1-7 years) | S3 Glacier Deep Archive | Almost never | 95% |
| Unknown patterns | S3 Intelligent-Tiering | Automatic | Variable |

**Lifecycle Policy:**
```json
{
  "Rules": [
    {
      "ID": "OptimizeStorageCosts",
      "Status": "Enabled",
      "Filter": {"Prefix": "uploads/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "NoncurrentVersionTransitions": [
        {"NoncurrentDays": 7, "StorageClass": "STANDARD_IA"},
        {"NoncurrentDays": 30, "StorageClass": "GLACIER"}
      ],
      "NoncurrentVersionExpiration": {"NoncurrentDays": 90},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
```

**Cost Optimization Techniques:**
1. **Multipart Upload Cleanup:** Abort incomplete uploads after 7 days (saves hidden costs).
2. **Versioning Strategy:** Enable versioning but expire old versions aggressively.
3. **Requester Pays:** For data sharing scenarios.
4. **S3 Analytics:** Enable Storage Class Analysis to identify optimization opportunities.
5. **Compression:** Compress before upload (gzip/zstd can save 60-80% on logs).

**Security Configuration:**
```json
{
  "BlockPublicAccess": {
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  },
  "Encryption": "aws:kms",
  "BucketPolicy": "Deny non-SSL requests",
  "AccessLogging": "Enabled",
  "ObjectLock": "Enabled for compliance data"
}
```

---

### Q18: How do you troubleshoot S3 performance issues when handling 10,000+ requests/second?

**Answer:**

**S3 Performance Limits:**
- 5,500 GET/HEAD requests per second per prefix.
- 3,500 PUT/COPY/POST/DELETE requests per second per prefix.

**Problem: All objects under single prefix `/uploads/`**

**Solution 1 - Prefix Distribution:**
```
Before: s3://bucket/uploads/file1.jpg, s3://bucket/uploads/file2.jpg
After:  s3://bucket/uploads/a1/file1.jpg, s3://bucket/uploads/b2/file2.jpg
```
Use hash-based prefix distribution to spread load across partitions.

**Solution 2 - CloudFront Integration:**
```
S3 → CloudFront Distribution
- Cache hot objects at edge locations
- Reduces S3 request load by 80-90%
- Use Origin Access Control (OAC) for security
```

**Solution 3 - S3 Transfer Acceleration:**
```bash
# Enable transfer acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket my-bucket \
  --accelerate-configuration Status=Enabled

# Use accelerated endpoint
aws s3 cp file.zip s3://my-bucket/file.zip --endpoint-url https://my-bucket.s3-accelerate.amazonaws.com
```

**Solution 4 - Multipart Upload for Large Files:**
```python
import boto3
from boto3.s3.transfer import TransferConfig

config = TransferConfig(
    multipart_threshold=100 * 1024 * 1024,  # 100 MB
    max_concurrency=10,
    multipart_chunksize=100 * 1024 * 1024,
    use_threads=True
)

s3 = boto3.client('s3')
s3.upload_file('large_file.zip', 'my-bucket', 'large_file.zip', Config=config)
```

**Monitoring:**
- Enable S3 Request Metrics (CloudWatch).
- Monitor `4xxErrors`, `5xxErrors`, `FirstByteLatency`, `TotalRequestLatency`.
- Set alarms on `503 SlowDown` errors (throttling indicator).

---

### Q19: How do you implement cross-account S3 access securely for a multi-team organization?

**Answer:**

**Approach 1: Bucket Policy (Simple Cross-Account Access)**
```json
{
  "Statement": [
    {
      "Sid": "CrossAccountAccess",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::ACCOUNT-B:root"},
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::shared-bucket",
        "arn:aws:s3:::shared-bucket/*"
      ],
      "Condition": {
        "StringEquals": {"aws:PrincipalOrgID": "o-xxxxxxxxxxxxx"}
      }
    }
  ]
}
```

**Approach 2: Cross-Account IAM Role (Recommended)**
```json
// Account A: Trust Policy for role
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::ACCOUNT-B:role/DataTeamRole"},
    "Action": "sts:AssumeRole",
    "Condition": {"StringEquals": {"sts:ExternalId": "UniqueExternalId"}}
  }]
}
```

**Approach 3: S3 Access Points (Multi-Team)**
```bash
# Create access point per team
aws s3control create-access-point \
  --account-id 111111111111 \
  --name data-team-ap \
  --bucket shared-bucket \
  --vpc-configuration VpcId=vpc-xxxxx

# Each access point has its own policy
# data-team can only access /data-team/* prefix
# ml-team can only access /ml-models/* prefix
```

**Security Best Practices:**
1. Use AWS Organizations SCPs to restrict cross-account access.
2. Enable CloudTrail data events for audit trail.
3. Use VPC endpoints for private access (no internet traversal).
4. Implement S3 Object Lock for regulatory compliance.
5. Use condition keys: `aws:PrincipalOrgID`, `aws:SourceVpc`, `s3:prefix`.

---


## Section 7: Amazon Aurora/RDS - Database Management

---

### Q20: Your Aurora PostgreSQL cluster has a replica lag of 500ms during peak traffic. How do you resolve this?

**Answer:**

**Diagnosis:**
```sql
-- Check replica lag
SELECT server_id, session_id, 
       EXTRACT(EPOCH FROM (now() - replay_lag)) AS lag_seconds
FROM aurora_replica_status();

-- Check for long-running transactions blocking replication
SELECT pid, age(clock_timestamp(), xact_start), query 
FROM pg_stat_activity 
WHERE state = 'active' AND xact_start < now() - interval '5 minutes';
```

**Root Causes & Solutions:**

**1. Write-Heavy Workload:**
```
Solution: 
- Scale up writer instance (r5.2xlarge → r5.4xlarge).
- Batch writes instead of individual inserts.
- Use asynchronous writes where possible.
```

**2. Large Transactions:**
```
Solution:
- Break large transactions into smaller batches.
- Use advisory locks instead of long table locks.
- Implement optimistic locking patterns.
```

**3. Replica Instance Too Small:**
```
Solution:
- Match replica instance size to writer (or larger for read-heavy).
- Add more replicas to distribute read load.
- Use Aurora Serverless v2 for automatic scaling.
```

**4. Network Throughput:**
```
Solution:
- Use instances with enhanced networking (r5 family).
- Check VPC flow logs for network bottlenecks.
- Ensure writer and replicas are in the same region.
```

**Aurora-Specific Optimizations:**
- Aurora replicas share storage layer (not logical replication), so lag is usually <100ms.
- If lag exceeds 100ms consistently, it's likely the replay process on the replica.
- Use `aurora_replica_read_consistency` parameter for session-level read-after-write consistency.

**Monitoring Setup:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "AuroraReplicaLag-Critical" \
  --metric-name AuroraReplicaLag \
  --namespace AWS/RDS \
  --statistic Maximum \
  --period 60 \
  --threshold 200 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456:DBAlerts
```

---

### Q21: Design a disaster recovery strategy for Aurora with RPO < 1 minute and RTO < 5 minutes.

**Answer:**

**Architecture: Aurora Global Database**

```
Primary Region (us-east-1):
  └── Aurora Cluster (Writer + 2 Readers)
       ├── Writer: r5.4xlarge
       ├── Reader 1: r5.2xlarge
       └── Reader 2: r5.2xlarge

Secondary Region (us-west-2):
  └── Aurora Global DB Secondary Cluster
       ├── Reader 1: r5.2xlarge (can be promoted to writer)
       └── Reader 2: r5.2xlarge
```

**RPO Achievement (< 1 minute):**
- Aurora Global Database provides replication lag typically < 1 second.
- Storage-level replication (not logical), so RPO ≈ 1 second.
- Monitor with `AuroraGlobalDBReplicationLag` CloudWatch metric.

**RTO Achievement (< 5 minutes):**

**Automated Failover Process:**
```python
# Lambda function triggered by Route 53 health check failure
import boto3

def failover_handler(event, context):
    rds = boto3.client('rds', region_name='us-west-2')
    
    # 1. Detach secondary from global cluster (makes it standalone writer)
    rds.remove_from_global_cluster(
        GlobalClusterIdentifier='my-global-cluster',
        DbClusterIdentifier='arn:aws:rds:us-west-2:xxx:cluster:my-secondary'
    )
    
    # 2. Update Route 53 to point to new region
    update_route53_to_secondary_region()
    
    # 3. Notify on-call team
    notify_team("Database failover executed to us-west-2")
```

**Failover Timeline:**
```
0:00 - Health check failure detected
0:30 - Route 53 health check confirms failure (2 consecutive failures)
1:00 - Lambda triggered, detach secondary cluster
2:00 - Secondary promoted to standalone writer
3:00 - Application DNS updated (Route 53 failover record)
4:00 - Connections established to new writer
4:30 - Service restored
```

**Backup Strategy (Belt and Suspenders):**
```
- Continuous backups: Aurora automatic backups (retained 35 days)
- Point-in-time recovery: Available within 5-minute granularity
- Cross-region snapshots: Every 6 hours for additional protection
- Logical backups: pg_dump weekly for table-level recovery
```

**Testing:**
- Monthly DR drills (promote secondary, validate application connectivity).
- Chaos engineering: Inject failures using AWS Fault Injection Simulator.
- Measure actual RTO/RPO during drills and track improvements.

---

### Q22: How do you optimize Aurora PostgreSQL performance for a high-concurrency application with 10,000+ database connections?

**Answer:**

**Problem:** Direct connections don't scale well. PostgreSQL has overhead per connection (memory, process).

**Solution Architecture:**
```
Application (10,000 connections) → RDS Proxy → Aurora (500 max connections)
```

**RDS Proxy Configuration:**
```bash
aws rds create-db-proxy \
  --db-proxy-name my-app-proxy \
  --engine-family POSTGRESQL \
  --auth Description="proxy auth",AuthScheme=SECRETS,SecretArn=arn:aws:secretsmanager:xxx,IAMAuth=REQUIRED \
  --role-arn arn:aws:iam::xxx:role/RDSProxyRole \
  --vpc-subnet-ids subnet-xxx subnet-yyy \
  --require-tls
```

**Connection Pooling Benefits:**
- Multiplexes thousands of application connections into hundreds of database connections.
- Reduces connection churn (expensive CREATE/DESTROY operations).
- Provides transparent failover (application doesn't see disconnects during failover).

**Database-Level Optimization:**

```sql
-- Optimize for high concurrency
ALTER SYSTEM SET max_connections = 500;
ALTER SYSTEM SET shared_buffers = '16GB';  -- 25% of RAM
ALTER SYSTEM SET effective_cache_size = '48GB';  -- 75% of RAM
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM SET maintenance_work_mem = '2GB';
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 8;

-- Connection settings
ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';
ALTER SYSTEM SET statement_timeout = '30s';
```

**Query Optimization:**
```sql
-- Identify slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Identify missing indexes
SELECT schemaname, relname, seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 100 AND seq_scan > idx_scan
ORDER BY seq_tup_read DESC;
```

**Read Scaling:**
- Route read queries to Aurora Replicas using reader endpoint.
- Use application-level read/write splitting.
- Implement caching (Redis/ElastiCache) for frequently accessed data.
- Use Aurora Serverless v2 replicas for burst read capacity.

---


## Section 8: Infrastructure as Code (Terraform & CloudFormation)

---

### Q23: How do you structure a Terraform codebase for managing multi-environment, multi-region AWS infrastructure?

**Answer:**

**Directory Structure:**
```
terraform/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf (EC2, ASG, ALB)
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf (Aurora, ElastiCache)
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf (CloudWatch, SNS)
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── production/
│       ├── us-east-1/
│       │   ├── main.tf
│       │   ├── terraform.tfvars
│       │   └── backend.tf
│       └── eu-west-1/
│           ├── main.tf
│           ├── terraform.tfvars
│           └── backend.tf
└── global/
    ├── iam/
    ├── route53/
    └── s3/
```

**State Management:**
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "production/us-east-1/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**Module Usage:**
```hcl
# environments/production/us-east-1/main.tf
module "networking" {
  source = "../../../modules/networking"
  
  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones
  environment        = "production"
  region             = "us-east-1"
}

module "compute" {
  source = "../../../modules/compute"
  
  vpc_id          = module.networking.vpc_id
  subnet_ids      = module.networking.private_subnet_ids
  instance_type   = "r5.2xlarge"
  min_size        = 3
  max_size        = 20
  desired_capacity = 6
}
```

**Best Practices:**
- Use remote state with S3 + DynamoDB for locking.
- Implement `terraform plan` in CI before `apply`.
- Use `terragrunt` for DRY configuration across environments.
- Pin module versions and provider versions.
- Use `moved` blocks for safe refactoring.

---

### Q24: Your Terraform state file is corrupted/lost. How do you recover?

**Answer:**

**Prevention (Before Disaster):**
```hcl
# S3 backend with versioning
resource "aws_s3_bucket_versioning" "state" {
  bucket = "terraform-state-bucket"
  versioning_configuration { status = "Enabled" }
}

# Enable MFA delete for state bucket
# Enable cross-region replication for state bucket
# DynamoDB table for state locking
```

**Recovery Scenarios:**

**Scenario 1: State in S3 with Versioning (Best Case)**
```bash
# List previous versions
aws s3api list-object-versions --bucket terraform-state --prefix production/terraform.tfstate

# Restore previous version
aws s3api get-object --bucket terraform-state \
  --key production/terraform.tfstate \
  --version-id "versionId123" \
  restored-terraform.tfstate

# Upload as current
aws s3 cp restored-terraform.tfstate s3://terraform-state/production/terraform.tfstate
```

**Scenario 2: State Completely Lost (Worst Case)**
```bash
# Import existing resources back into state
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_db_instance.main my-database
terraform import aws_lb.main arn:aws:elasticloadbalancing:...

# For large infrastructure, use tools:
# - terraformer: Auto-generates tf files + state from existing infrastructure
# - terracognita: Similar import tool

# Verify imported state matches reality
terraform plan  # Should show no changes if import is complete
```

**Scenario 3: State Lock Stuck (DynamoDB)**
```bash
# Force unlock (use with caution)
terraform force-unlock LOCK-ID

# Or manually delete from DynamoDB
aws dynamodb delete-item \
  --table-name terraform-locks \
  --key '{"LockID": {"S": "terraform-state/production/terraform.tfstate-md5"}}'
```

**Post-Recovery Validation:**
```bash
terraform plan  # Verify no unexpected changes
terraform state list  # Verify all resources present
terraform show  # Review full state
```

---

### Q25: How do you implement drift detection and remediation in your Terraform workflow?

**Answer:**

**Automated Drift Detection Pipeline:**

```yaml
# GitHub Actions / Jenkins Pipeline
name: Terraform Drift Detection
on:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours

jobs:
  drift-check:
    steps:
      - name: Terraform Plan
        run: |
          terraform init
          terraform plan -detailed-exitcode -out=plan.out 2>&1 | tee plan.log
          EXIT_CODE=$?
          if [ $EXIT_CODE -eq 2 ]; then
            echo "DRIFT DETECTED"
            # Send alert to Slack/PagerDuty
            curl -X POST $SLACK_WEBHOOK -d '{"text":"Terraform drift detected in production!"}'
          fi
```

**AWS Config for Real-Time Drift Detection:**
```hcl
resource "aws_config_config_rule" "required_tags" {
  name = "required-tags"
  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }
  input_parameters = jsonencode({
    tag1Key   = "Environment"
    tag2Key   = "Owner"
    tag3Key   = "ManagedBy"
  })
}
```

**CloudFormation Drift Detection (for CF-managed resources):**
```bash
aws cloudformation detect-stack-drift --stack-name my-stack
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id xxx
```

**Remediation Strategies:**

1. **Auto-Remediate:** Run `terraform apply` automatically for non-critical drift.
2. **Alert & Manual:** For production, alert team and require manual approval.
3. **Prevent Drift:** Use IAM policies to restrict manual changes.

```json
// IAM policy to prevent manual changes to Terraform-managed resources
{
  "Statement": [{
    "Effect": "Deny",
    "Action": ["ec2:*", "rds:*"],
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:PrincipalArn": "arn:aws:iam::xxx:role/TerraformRole"
      },
      "StringEquals": {
        "aws:ResourceTag/ManagedBy": "terraform"
      }
    }
  }]
}
```

---


## Section 9: RabbitMQ & Redis Infrastructure

---

### Q26: How do you design a highly available RabbitMQ cluster for a quick-commerce platform processing 50,000 messages/second?

**Answer:**

**Cluster Architecture:**
```
                    ┌─── RabbitMQ Node 1 (AZ-a) ───┐
ALB/NLB ──────────├─── RabbitMQ Node 2 (AZ-b) ───├── Mirrored Queues
                    └─── RabbitMQ Node 3 (AZ-c) ───┘
```

**Amazon MQ for RabbitMQ (Managed):**
```bash
aws mq create-broker \
  --broker-name production-rabbitmq \
  --deployment-mode CLUSTER_MULTI_AZ \
  --engine-type RABBITMQ \
  --engine-version 3.11.20 \
  --host-instance-type mq.m5.large \
  --publicly-accessible false
```

**Self-Managed Cluster Configuration:**
```ini
# rabbitmq.conf
cluster_formation.peer_discovery_backend = aws
cluster_formation.aws.region = us-east-1
cluster_formation.aws.use_autoscaling_group = true

# Queue mirroring policy
ha-mode = exactly
ha-params = 2
ha-sync-mode = automatic

# Performance tuning
vm_memory_high_watermark.relative = 0.7
disk_free_limit.absolute = 5GB
channel_max = 2048
```

**Queue Design for 50K msgs/sec:**
```
1. Use Quorum Queues (not Classic Mirrored) for durability + performance.
2. Implement sharded queues: order-queue-{0..9} with consistent hashing.
3. Prefetch count: Set to 20-50 (not unlimited) for fair dispatch.
4. Message TTL: 60 seconds for real-time delivery events.
5. Dead Letter Exchange: Capture failed messages for analysis.
```

**Monitoring:**
```
Critical Metrics:
- Queue depth > 10,000 → Alert (consumers not keeping up)
- Memory usage > 70% → Warning, > 85% → Critical
- File descriptors > 80% of limit → Warning
- Consumer count = 0 → Critical (no consumers processing)
- Message publish rate vs consume rate (should be balanced)
```

**Disaster Recovery:**
- Use Shovel or Federation plugin for cross-region replication.
- Implement publisher confirms for guaranteed delivery.
- Consumer acknowledgments with requeue on failure.
- Regular backup of definitions (exchanges, queues, policies).

---

### Q27: Your Redis cluster is experiencing memory pressure and evicting keys. How do you handle this in production?

**Answer:**

**Immediate Diagnosis:**
```bash
# Check memory usage
redis-cli INFO memory
# Key metrics: used_memory, used_memory_peak, maxmemory, mem_fragmentation_ratio

# Check eviction policy
redis-cli CONFIG GET maxmemory-policy
# Recommended: allkeys-lru or volatile-lru

# Find big keys consuming memory
redis-cli --bigkeys
redis-cli MEMORY USAGE key_name

# Check key expiration
redis-cli INFO keyspace
```

**Immediate Resolution:**

**1. Identify and Remove Unnecessary Data:**
```bash
# Find keys by pattern and check their TTL
redis-cli --scan --pattern "session:*" | head -100 | xargs -I {} redis-cli TTL {}

# Set TTL on keys without expiration
redis-cli EXPIRE large_key 3600
```

**2. Memory Optimization:**
```
# Redis configuration optimization
maxmemory-policy allkeys-lru
activedefrag yes
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes

# Data structure optimization:
- Use Hashes instead of individual keys for objects (ziplist encoding)
- Use bitmaps for boolean data
- Use HyperLogLog for cardinality counting
- Use compressed serialization (msgpack instead of JSON)
```

**3. Scaling Strategy:**

**Vertical Scaling (Quick Fix):**
```bash
# ElastiCache: Modify node type
aws elasticache modify-replication-group \
  --replication-group-id my-redis-cluster \
  --cache-node-type cache.r6g.2xlarge \
  --apply-immediately
```

**Horizontal Scaling (Long-term):**
```bash
# Enable cluster mode for sharding
aws elasticache modify-replication-group \
  --replication-group-id my-redis-cluster \
  --resharding-configuration '{"NodeGroupCount": 6}'
```

**Application-Level Solutions:**
- Implement multi-tier caching (L1: In-memory/Caffeine, L2: Redis).
- Use Redis Cluster with hash slots for data distribution.
- Implement cache-aside pattern with appropriate TTLs.
- Use `OBJECT ENCODING key` to check encoding and optimize data structures.

---

### Q28: Design a Redis caching strategy for a high-concurrency delivery tracking system.

**Answer:**

**Architecture:**
```
Application → Redis (ElastiCache Cluster Mode Enabled)
               ├── Shard 1: Driver locations (Geospatial)
               ├── Shard 2: Order status (Hash)
               ├── Shard 3: Session data (String)
               └── Shard 4: Rate limiting (Sorted Set)
```

**ElastiCache Configuration:**
```bash
aws elasticache create-replication-group \
  --replication-group-id delivery-redis \
  --description "Delivery tracking cache" \
  --num-node-groups 4 \
  --replicas-per-node-group 2 \
  --cache-node-type cache.r6g.xlarge \
  --cluster-mode-enabled \
  --automatic-failover-enabled \
  --multi-az-enabled \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled
```

**Use Case Implementations:**

**1. Real-Time Driver Location (Geospatial):**
```python
# Store driver location
redis.geoadd("drivers:active", longitude, latitude, f"driver:{driver_id}")

# Find drivers within 3km of customer
nearby = redis.geosearch("drivers:active", 
    longitude=customer_lng, latitude=customer_lat, 
    radius=3, unit="km", sort="ASC", count=10)
```

**2. Order Status Cache (Hash):**
```python
# Store order status
redis.hset(f"order:{order_id}", mapping={
    "status": "out_for_delivery",
    "driver_id": "D123",
    "eta_minutes": "12",
    "last_updated": "2024-01-15T10:30:00Z"
})
redis.expire(f"order:{order_id}", 86400)  # 24h TTL

# Pub/Sub for real-time updates
redis.publish(f"order_updates:{order_id}", json.dumps(update))
```

**3. Rate Limiting (Sliding Window):**
```python
def is_rate_limited(user_id, limit=100, window=60):
    key = f"ratelimit:{user_id}"
    now = time.time()
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zadd(key, {str(now): now})
    pipe.zcard(key)
    pipe.expire(key, window)
    results = pipe.execute()
    return results[2] > limit
```

**High Availability Pattern:**
- Cluster Mode Enabled: 4 shards x 3 nodes (1 primary + 2 replicas) = 12 nodes.
- Multi-AZ deployment with automatic failover.
- Connection pooling at application level (max 50 connections per node).
- Implement circuit breaker: fallback to database if Redis is unavailable.

---


## Section 10: Incident Response & Production Support

---

### Q29: Walk through your incident response process for a complete production outage affecting thousands of users.

**Answer:**

**Incident Response Framework (OODA Loop):**

**Phase 1 - Detection (0-2 minutes):**
```
Automated:
- CloudWatch alarms trigger → SNS → PagerDuty page on-call
- Synthetic monitoring (Route 53 health checks) detects failure
- APM tools show 5xx spike

Manual:
- Customer reports flood support channels
- Status page auto-updates based on alarm state
```

**Phase 2 - Triage (2-5 minutes):**
```
1. Acknowledge the incident in PagerDuty.
2. Open incident Slack channel (#inc-YYYY-MM-DD-description).
3. Quick blast radius assessment:
   - Which services affected?
   - Which regions?
   - How many users impacted?
4. Assign roles: Incident Commander, Technical Lead, Communications Lead.
5. Set severity: SEV1 (complete outage), SEV2 (degraded), SEV3 (minor impact).
```

**Phase 3 - Mitigation (5-30 minutes):**
```
Decision Tree:
├── Recent deployment? → Rollback immediately
├── Database issue? → Failover to replica/DR
├── Single AZ failure? → Evacuate AZ (remove from ALB)
├── DDoS attack? → Engage AWS Shield, enable WAF rules
├── Third-party dependency? → Enable circuit breaker, serve cached data
└── Unknown? → Increase capacity while investigating
```

**Phase 4 - Resolution & Communication:**
```
- Update status page every 15 minutes.
- Internal communication to leadership every 30 minutes.
- Post-resolution: Verify all services healthy, run smoke tests.
- Keep incident channel open for 24 hours for monitoring.
```

**Phase 5 - Post-Incident Review (Within 48 hours):**
```markdown
## Post-Incident Review Template
- **Incident ID:** INC-2024-0115
- **Duration:** 23 minutes
- **Impact:** 15,000 users affected, $XX revenue impact
- **Timeline:** Detailed minute-by-minute breakdown
- **Root Cause:** Configuration change to ASG removed instances from ALB
- **Contributing Factors:** Missing integration test, no canary deployment
- **Action Items:**
  - [ ] Add integration test for ALB attachment (Owner: @engineer, Due: Jan 22)
  - [ ] Implement canary deployments (Owner: @devops, Due: Feb 1)
  - [ ] Add runbook for ASG misconfiguration (Owner: @sre, Due: Jan 20)
```

---

### Q30: Your application is experiencing intermittent 504 Gateway Timeouts. How do you systematically troubleshoot?

**Answer:**

**504 Gateway Timeout Chain:**
```
Client → CloudFront → ALB → Target (EC2/Container) → Backend (DB/API)
                                                        ↑
                                                 Timeout occurs here
```

**Systematic Investigation:**

**Layer 1 - ALB/CloudFront:**
```bash
# Check ALB access logs
aws s3 cp s3://alb-logs/2024/01/15/ ./logs/ --recursive
grep "504" ./logs/* | awk '{print $13, $14, $15}'
# Fields: target_processing_time, response_processing_time, request_processing_time

# If target_processing_time = -1, target never responded
# If target_processing_time > idle_timeout, adjust ALB idle timeout
```

**Layer 2 - Target Health:**
```bash
# Check target group health
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:...

# Check if targets are draining (deregistering)
# Check connection count per target
```

**Layer 3 - Application:**
```bash
# Check application threads/workers
# Python/Flask: Check if all Gunicorn workers are busy
ps aux | grep gunicorn
# Check for thread pool exhaustion

# Check for connection pool exhaustion
netstat -an | grep ESTABLISHED | wc -l
netstat -an | grep TIME_WAIT | wc -l
```

**Layer 4 - Backend Dependencies:**
```bash
# Database connection pool exhausted?
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';

# Redis connection timeout?
redis-cli --latency

# External API timeout?
# Check circuit breaker states
```

**Resolution Checklist:**
| Cause | Solution |
|---|---|
| ALB idle timeout too low | Increase from 60s to 120s |
| Target slow to respond | Scale up instances, optimize code |
| DB connection pool full | Increase pool size, add RDS Proxy |
| Worker/thread exhaustion | Increase workers, implement async |
| DNS resolution delay | Use VPC DNS caching |
| Upstream API slow | Implement circuit breaker + timeout |
| Keep-alive misconfiguration | App keep-alive > ALB idle timeout |

**Key Fix:**
```
Application keep-alive timeout MUST be greater than ALB idle timeout.
ALB idle timeout: 60s → Application keep-alive: 65s
```

---

### Q31: How do you implement a chaos engineering program to improve production resilience?

**Answer:**

**AWS Fault Injection Simulator (FIS) Experiments:**

**Experiment 1: AZ Failure**
```json
{
  "description": "Simulate AZ failure",
  "targets": {
    "ec2-instances": {
      "resourceType": "aws:ec2:instance",
      "selectionMode": "ALL",
      "filters": [
        {"path": "Placement.AvailabilityZone", "values": ["us-east-1a"]}
      ]
    }
  },
  "actions": {
    "stop-instances": {
      "actionId": "aws:ec2:stop-instances",
      "targets": {"Instances": "ec2-instances"},
      "duration": "PT10M"
    }
  },
  "stopConditions": [
    {"source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:...:alarm:ErrorRate-Critical"}
  ]
}
```

**Experiment 2: Network Latency Injection**
```json
{
  "actions": {
    "inject-latency": {
      "actionId": "aws:ssm:send-command",
      "parameters": {
        "documentArn": "arn:aws:ssm:...:document/InjectNetworkLatency",
        "documentParameters": "{\"latencyMs\": \"500\", \"interface\": \"eth0\", \"duration\": \"300\"}"
      }
    }
  }
}
```

**Experiment 3: Database Failover**
```bash
# Force Aurora failover
aws rds failover-db-cluster --db-cluster-identifier production-cluster

# Measure:
# - Connection recovery time
# - Error rate during failover
# - Data consistency post-failover
```

**Chaos Engineering Process:**
```
1. Define steady state (normal error rate, latency P99, throughput)
2. Form hypothesis: "System recovers from AZ failure within 2 minutes"
3. Run experiment in staging first, then production
4. Monitor blast radius with stop conditions
5. Analyze results and create improvement items
6. Repeat monthly with increasing severity
```

**Safety Guidelines:**
- Always have stop conditions (CloudWatch alarms that auto-stop experiments).
- Start with small blast radius (1 instance, not entire AZ).
- Run during business hours with team standing by.
- Never test in production without testing in staging first.
- Maintain a "big red button" to immediately stop all experiments.

---


## Section 11: Security & Cost Optimization

---

### Q32: How do you implement a defense-in-depth security strategy for AWS infrastructure?

**Answer:**

**Security Layers:**

```
Layer 1: Edge (CloudFront + WAF + Shield)
Layer 2: Network (VPC, Security Groups, NACLs, VPC Endpoints)
Layer 3: Identity (IAM, SSO, MFA, Service Control Policies)
Layer 4: Application (Secrets Manager, Parameter Store, Encryption)
Layer 5: Data (KMS, S3 encryption, RDS encryption, backup encryption)
Layer 6: Monitoring (CloudTrail, GuardDuty, Security Hub, Config)
```

**Implementation Details:**

**Network Security:**
```hcl
# VPC with private subnets (no internet access for application tier)
resource "aws_subnet" "private" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = false
}

# VPC Endpoints for AWS services (no internet needed)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"
  route_table_ids = [aws_route_table.private.id]
}

# Security Group: Least privilege
resource "aws_security_group" "app" {
  ingress {
    from_port       = 8080
    to_port         = 8080
    security_groups = [aws_security_group.alb.id]  # Only ALB can reach app
  }
  egress {
    from_port   = 5432
    to_port     = 5432
    security_groups = [aws_security_group.db.id]  # App can only reach DB
  }
}
```

**IAM Best Practices:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/app-data/*",
      "Condition": {
        "StringEquals": {"aws:RequestedRegion": "us-east-1"},
        "Bool": {"aws:SecureTransport": "true"}
      }
    }
  ]
}
```

**Secrets Management:**
```python
# Use Secrets Manager with automatic rotation
import boto3

def get_db_credentials():
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId='production/database')
    return json.loads(response['SecretString'])

# Enable automatic rotation every 30 days
# Never hardcode credentials in code or environment variables
```

**Security Monitoring:**
```bash
# Enable GuardDuty for threat detection
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# Security Hub for centralized findings
aws securityhub enable-security-hub --enable-default-standards

# CloudTrail for audit logging
aws cloudtrail create-trail --name production-trail \
  --s3-bucket-name audit-logs \
  --is-multi-region-trail \
  --enable-log-file-validation
```

---

### Q33: You need to reduce AWS costs by 30% without impacting performance. What's your approach?

**Answer:**

**Cost Optimization Framework:**

**Phase 1 - Visibility (Week 1):**
```
1. Enable AWS Cost Explorer with hourly granularity.
2. Implement resource tagging (Environment, Team, Service, CostCenter).
3. Set up AWS Budgets with alerts at 80%, 90%, 100% of target.
4. Use Cost Anomaly Detection for unexpected spend spikes.
5. Review Trusted Advisor cost optimization recommendations.
```

**Phase 2 - Quick Wins (Week 2-3):**

| Action | Typical Savings |
|---|---|
| Right-size overprovisioned EC2 instances | 20-30% |
| Purchase Reserved Instances/Savings Plans | 30-60% |
| Terminate idle resources (unused EBS, EIPs) | 5-10% |
| S3 lifecycle policies (IA, Glacier) | 40-70% on storage |
| Delete unattached EBS volumes/old snapshots | 5-15% |

**Phase 3 - Architecture Optimization (Week 4-8):**

```bash
# Right-sizing with CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-xxxxx \
  --start-time 2024-01-01 --end-time 2024-01-15 \
  --period 3600 --statistics Average Maximum

# If max CPU < 40% consistently → downsize
# If avg CPU < 10% → consider spot/serverless
```

**Savings Plans Strategy:**
```
Compute Savings Plans (most flexible):
- 1-year No Upfront: 20% savings
- 1-year All Upfront: 30% savings
- 3-year All Upfront: 50% savings

Coverage target: 70% baseline with Savings Plans, 30% On-Demand for burst
```

**Spot Instances for Non-Critical:**
```hcl
# ASG with mixed instances policy
resource "aws_autoscaling_group" "workers" {
  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = 2
      on_demand_percentage_above_base_capacity = 20
      spot_allocation_strategy                 = "capacity-optimized"
    }
    launch_template {
      override {
        instance_type = "c5.2xlarge"
      }
      override {
        instance_type = "c5a.2xlarge"
      }
      override {
        instance_type = "c5n.2xlarge"
      }
    }
  }
}
```

**Phase 4 - Ongoing Governance:**
- Monthly cost review meetings with engineering leads.
- Automated cleanup Lambda for untagged/idle resources.
- Implement showback/chargeback per team.
- Set team-level budgets with enforcement.

---

### Q34: How do you implement infrastructure security scanning and compliance automation?

**Answer:**

**Compliance Stack:**
```
AWS Config Rules → Security Hub → Custom Remediation Lambdas
      ↓                              ↓
  Non-compliant detection → Automatic or manual remediation
```

**AWS Config Rules for Common Compliance:**
```hcl
# Ensure all EBS volumes are encrypted
resource "aws_config_config_rule" "ebs_encryption" {
  name = "ebs-encrypted"
  source {
    owner             = "AWS"
    source_identifier = "ENCRYPTED_VOLUMES"
  }
}

# Ensure security groups don't allow 0.0.0.0/0 on SSH
resource "aws_config_config_rule" "restricted_ssh" {
  name = "restricted-ssh"
  source {
    owner             = "AWS"
    source_identifier = "INCOMING_SSH_DISABLED"
  }
}

# Auto-remediation: Revoke public security group rules
resource "aws_config_remediation_configuration" "revoke_sg" {
  config_rule_name = aws_config_config_rule.restricted_ssh.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-DisablePublicAccessForSecurityGroup"
  automatic        = true
  
  parameter {
    name         = "GroupId"
    resource_value = "RESOURCE_ID"
  }
}
```

**Infrastructure Scanning Pipeline:**
```yaml
# Pre-deployment security checks
steps:
  - name: Terraform Security Scan
    run: |
      # tfsec for Terraform misconfigurations
      tfsec . --format json --out tfsec-results.json
      
      # checkov for policy-as-code
      checkov -d . --output json > checkov-results.json
      
      # terrascan for compliance
      terrascan scan -i terraform -d . -o json > terrascan-results.json

  - name: Container Image Scan
    run: |
      # Trivy for vulnerability scanning
      trivy image --severity HIGH,CRITICAL myapp:latest
      
      # ECR native scanning
      aws ecr start-image-scan --repository-name myapp --image-id imageTag=latest
```

**Compliance Dashboard:**
- AWS Security Hub score target: 90%+
- AWS Config conformance packs for CIS Benchmarks, PCI DSS.
- Custom dashboards showing compliance trends over time.
- Weekly compliance reports to security team and leadership.

---


## Section 12: High Availability & Real-Time Systems (Quick Commerce/Logistics)

---

### Q35: Design a highly available architecture for a quick-commerce platform that guarantees 99.99% uptime.

**Answer:**

**Architecture Overview:**
```
                        Route 53 (Latency + Failover)
                              │
               ┌──────────────┼──────────────┐
               ▼                              ▼
         CloudFront                     CloudFront
         (Edge Cache)                   (Edge Cache)
               │                              │
         WAF + Shield                   WAF + Shield
               │                              │
     ┌─────── ALB (us-east-1) ────┐    ALB (us-west-2) [DR]
     │              │              │
  ASG (API)   ASG (Workers)   ASG (WebSocket)
     │              │              │
     └──────────────┼──────────────┘
                    │
     ┌──────────────┼──────────────┐
     │              │              │
 Aurora Global   ElastiCache    Amazon MQ
 (Multi-AZ)     (Redis Cluster) (RabbitMQ)
```

**99.99% Uptime Calculation:**
```
99.99% = 52.6 minutes of downtime per year
= 4.38 minutes per month
= 8.6 seconds per day

To achieve this:
- Multi-AZ within region: Survives single AZ failure
- Multi-Region: Survives entire region failure
- No single points of failure
- Automated failover at every layer
```

**Key Design Decisions:**

**1. Stateless Application Layer:**
```
- Sessions stored in Redis (not local memory)
- File uploads go directly to S3 (presigned URLs)
- WebSocket state backed by Redis Pub/Sub
- Any instance can handle any request
```

**2. Database Layer:**
```
- Aurora Global Database: <1s replication to DR region
- Read replicas for read scaling (80% reads, 20% writes)
- RDS Proxy for connection pooling (handle 10K+ connections)
- Automated failover: <30 seconds for same-region, <1 minute cross-region
```

**3. Caching Layer:**
```
- ElastiCache Redis Cluster Mode: 6 shards, 2 replicas each
- Multi-AZ with automatic failover
- Read-through cache for product catalog
- Write-behind cache for order updates
- Circuit breaker: Serve stale data if Redis unavailable
```

**4. Message Queue:**
```
- Amazon MQ (RabbitMQ): Cluster multi-AZ deployment
- Critical flows: Order placement, driver assignment, delivery updates
- Dead letter queues with alarming for failed messages
- Maximum retry: 3 attempts with exponential backoff
```

**5. Auto-Healing:**
```python
# Lambda auto-remediation triggered by CloudWatch
def auto_heal(event):
    alarm = event['detail']['alarmName']
    
    if 'UnhealthyHost' in alarm:
        terminate_and_replace_instance()
    elif 'HighErrorRate' in alarm:
        rollback_last_deployment()
    elif 'DatabaseConnectionExhausted' in alarm:
        scale_up_rds_proxy()
```

---

### Q36: How do you handle a traffic spike of 10x normal load during a flash sale event?

**Answer:**

**Pre-Event Preparation (T-7 days):**
```
1. Load Testing:
   - Run load tests at 12x expected peak (headroom)
   - Identify bottlenecks: DB connections, cache capacity, queue depth
   
2. Pre-Scale Infrastructure:
   - Increase ASG min/desired capacity to expected peak
   - Scale up Aurora writer instance (r5.4xl → r5.8xl)
   - Add Redis replicas
   - Pre-warm ALB (contact AWS support or generate synthetic traffic)
   
3. Feature Flags:
   - Disable non-critical features during spike
   - Enable graceful degradation modes
   - Prepare circuit breakers for external dependencies
```

**Architecture for Flash Sale:**
```
                    CloudFront (Cache static assets)
                         │
                    WAF (Rate limiting per IP)
                         │
                    ALB (Pre-warmed)
                         │
              ┌──────────┼──────────┐
              │          │          │
         ASG (API)  ASG (Order)  ASG (Search)
         min:20     min:15       min:10
         max:100    max:80       max:50
              │          │          │
              ▼          ▼          ▼
         Redis      SQS Queue    Elasticsearch
         (Rate      (Order       (Pre-warmed
          Limit)     Buffer)      indices)
```

**Traffic Management:**

**1. Request Queuing:**
```python
# SQS-based order queuing during extreme load
def place_order(order_data):
    if current_load > threshold:
        # Queue order instead of direct processing
        sqs.send_message(
            QueueUrl=ORDER_QUEUE_URL,
            MessageBody=json.dumps(order_data),
            MessageAttributes={
                'Priority': {'StringValue': 'high', 'DataType': 'String'}
            }
        )
        return {"status": "queued", "message": "Order received, processing shortly"}
    else:
        return process_order_directly(order_data)
```

**2. Rate Limiting at Multiple Levels:**
```
- WAF: 1000 requests/5-min per IP
- ALB: Connection limits per target
- Application: Redis-based rate limiting per user
- Database: RDS Proxy connection pooling
```

**3. Auto Scaling Policies:**
```hcl
resource "aws_autoscaling_policy" "flash_sale" {
  name                      = "flash-sale-scaling"
  autoscaling_group_name    = aws_autoscaling_group.api.name
  policy_type               = "TargetTrackingScaling"
  
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.main.arn_suffix}/${aws_lb_target_group.api.arn_suffix}"
    }
    target_value = 1000  # requests per target
    scale_in_cooldown  = 300
    scale_out_cooldown = 60  # Fast scale-out
  }
}
```

**Post-Event:**
- Gradually scale down over 2-4 hours (don't scale in immediately).
- Review metrics: peak requests, error rates, latency percentiles.
- Document lessons learned for next event.

---

### Q37: How do you implement observability for a microservices architecture on AWS?

**Answer:**

**Three Pillars of Observability:**

**1. Metrics (CloudWatch + Custom):**
```
Infrastructure: CPU, Memory, Network, Disk (CloudWatch Agent)
Application: Request rate, Error rate, Duration (RED method)
Business: Orders/min, Revenue/hour, Active users (Custom metrics)

Key Dashboards:
- Service Overview: All microservices health at a glance
- Deep Dive: Per-service metrics with P50, P95, P99 latency
- Business: Real-time business KPIs
```

**2. Logs (CloudWatch Logs + OpenSearch):**
```json
// Structured logging format (JSON)
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "abc-123-def",
  "span_id": "xyz-789",
  "message": "Payment processing failed",
  "error": "TimeoutException",
  "user_id": "U12345",
  "order_id": "ORD-67890",
  "duration_ms": 5023,
  "metadata": {
    "payment_provider": "stripe",
    "retry_count": 3
  }
}
```

**Log Pipeline:**
```
Application → CloudWatch Logs → Subscription Filter → 
  ├── Lambda → OpenSearch (for search/analysis)
  ├── Kinesis Firehose → S3 (long-term archive)
  └── CloudWatch Metric Filter → Alarm (error counting)
```

**3. Distributed Tracing (AWS X-Ray):**
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

app = Flask(__name__)
XRayMiddleware(app, xray_recorder)

@app.route('/orders', methods=['POST'])
@xray_recorder.capture('create_order')
def create_order():
    # Trace propagated across services
    subsegment = xray_recorder.begin_subsegment('validate_payment')
    payment_result = payment_service.validate(order)
    xray_recorder.end_subsegment()
    
    subsegment = xray_recorder.begin_subsegment('save_to_db')
    db.save(order)
    xray_recorder.end_subsegment()
```

**Service Map & Dependencies:**
```
X-Ray Service Map shows:
Order Service → Payment Service (avg 200ms, 0.1% errors)
Order Service → Inventory Service (avg 50ms, 0% errors)
Order Service → Aurora DB (avg 5ms, 0% errors)
Order Service → Redis Cache (avg 1ms, 0% errors)
```

**Alerting Strategy:**
```
SLO-Based Alerting:
- SLO: 99.9% of requests < 500ms
- Burn rate alert: If error budget consumed at 10x rate → Page
- Burn rate alert: If error budget consumed at 2x rate → Ticket

Multi-Window:
- 5-minute window: Catch sudden spikes
- 1-hour window: Catch sustained degradation
- 24-hour window: Catch slow burns
```

---


## Section 13: Docker, Kubernetes & ECS (Good to Have)

---

### Q38: How do you containerize a Python/Flask application and deploy it to Amazon ECS with auto-scaling?

**Answer:**

**Dockerfile (Production-Grade):**
```dockerfile
# Multi-stage build for minimal image size
FROM python:3.11-slim as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim
WORKDIR /app

# Security: Run as non-root user
RUN useradd -m -r appuser && chown appuser:appuser /app
USER appuser

# Copy dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local
ENV PATH=/home/appuser/.local/bin:$PATH

# Copy application
COPY --chown=appuser:appuser . .

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "--threads", "2", "--timeout", "120", "app:create_app()"]
```

**ECS Task Definition:**
```json
{
  "family": "flask-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::xxx:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::xxx:role/ecsTaskRole",
  "containerDefinitions": [{
    "name": "flask-app",
    "image": "xxx.dkr.ecr.us-east-1.amazonaws.com/flask-app:latest",
    "portMappings": [{"containerPort": 8000, "protocol": "tcp"}],
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 60
    },
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/flask-app",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "secrets": [
      {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:...:production/db-url"}
    ]
  }]
}
```

**Auto-Scaling Configuration:**
```bash
# Target tracking on CPU
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-cluster/flask-app \
  --min-capacity 3 --max-capacity 50

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/my-cluster/flask-app \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

---

### Q39: How do you implement a Kubernetes rolling update strategy with zero downtime?

**Answer:**

**Deployment Configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  namespace: production
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # Only 1 pod down at a time
      maxSurge: 2            # Allow 2 extra pods during update
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
        version: v2.1.0
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: flask-app
        image: xxx.dkr.ecr.us-east-1.amazonaws.com/flask-app:v2.1.0
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]  # Allow in-flight requests to complete
```

**Key Zero-Downtime Elements:**
1. **Readiness Probe:** New pod only receives traffic when healthy.
2. **preStop Hook:** Gives time for load balancer to deregister pod.
3. **terminationGracePeriodSeconds:** Allows graceful shutdown.
4. **PodDisruptionBudget:** Ensures minimum availability during disruptions.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: flask-app-pdb
spec:
  minAvailable: 4  # Always keep at least 4 pods running
  selector:
    matchLabels:
      app: flask-app
```

---

### Q40: How do you secure container images in ECR and implement vulnerability scanning?

**Answer:**

**ECR Security Configuration:**
```hcl
resource "aws_ecr_repository" "app" {
  name                 = "flask-app"
  image_tag_mutability = "IMMUTABLE"  # Prevent tag overwriting
  
  image_scanning_configuration {
    scan_on_push = true  # Automatic vulnerability scanning
  }
  
  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }
}

# Lifecycle policy: Keep only last 10 images
resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name
  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 10 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 10
      }
      action = { type = "expire" }
    }]
  })
}
```

**CI/CD Security Pipeline:**
```yaml
steps:
  - name: Build Image
    run: docker build -t flask-app:$COMMIT_SHA .

  - name: Scan with Trivy
    run: |
      trivy image --severity HIGH,CRITICAL \
        --exit-code 1 \
        flask-app:$COMMIT_SHA
      # Fail pipeline if HIGH/CRITICAL vulnerabilities found

  - name: Push to ECR
    run: |
      aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URL
      docker push $ECR_URL/flask-app:$COMMIT_SHA

  - name: Verify ECR Scan Results
    run: |
      aws ecr wait image-scan-complete --repository-name flask-app --image-id imageTag=$COMMIT_SHA
      FINDINGS=$(aws ecr describe-image-scan-findings --repository-name flask-app --image-id imageTag=$COMMIT_SHA)
      # Block deployment if critical findings exist
```

**Runtime Security:**
- Use read-only root filesystem in containers.
- Run as non-root user (securityContext: runAsNonRoot: true).
- Drop all capabilities, add only what's needed.
- Use Pod Security Standards (restricted profile).
- Implement network policies to restrict pod-to-pod communication.

---


## Section 14: Scenario-Based Questions (Industry Real-World)

---

### Q41: Your quick-commerce platform is experiencing a cascading failure during peak dinner hours. Orders are failing, drivers can't update status, and the customer app is unresponsive. Walk through your response.

**Answer:**

**Minute 0-2: Detection & Acknowledgment**
```
- PagerDuty alerts fire: 5xx rate > 50%, order failures > 100/min
- On-call engineer acknowledges, opens #incident-20240115 Slack channel
- Status page auto-updated to "Investigating"
```

**Minute 2-5: Rapid Assessment**
```bash
# Quick health check across services
curl -s https://api.myapp.com/health | jq .
# Check: Which services are DOWN vs DEGRADED?

# CloudWatch dashboard: Identify the origin
# Pattern: DB connections exhausted → queries queuing → timeouts cascading

# Key finding: Aurora writer at 100% CPU, 500 active connections
```

**Minute 5-10: Containment**
```
Immediate Actions:
1. Enable circuit breaker for non-critical services (recommendations, analytics)
2. Activate rate limiting: 50 requests/sec per user
3. Scale out read replicas (add 2 more Aurora replicas)
4. Increase RDS Proxy max connections
5. Flush Redis cache for stale data that might be causing cache stampede
```

**Minute 10-20: Root Cause Identification**
```sql
-- Found: A new feature released 30 min ago has N+1 query problem
SELECT query, calls, mean_exec_time 
FROM pg_stat_statements 
ORDER BY total_exec_time DESC LIMIT 5;

-- Result: SELECT * FROM deliveries WHERE driver_id = ? 
-- Called 500,000 times in last 30 minutes (should be batched)
```

**Minute 20-25: Resolution**
```
1. Rollback the problematic deployment via CodeDeploy
2. Verify order processing resuming
3. Monitor error rate dropping
4. Clear the backed-up message queues
```

**Minute 25-30: Verification & Recovery**
```
- Error rate back to < 0.1%
- Order processing caught up (queue depth = 0)
- Driver app responsive
- All health checks green
- Status page: "Resolved"
```

**Post-Incident Actions:**
- Fix N+1 query with batch loading
- Add query performance regression tests to CI/CD
- Implement slow query alerting (queries > 100ms)
- Add canary deployment for this service

---

### Q42: Your company is expanding from India to Southeast Asia. Design the infrastructure expansion strategy.

**Answer:**

**Phase 1: Requirements Analysis**
```
- Data residency: Singapore (ap-southeast-1) for SEA compliance
- Latency target: <100ms API response for SEA users
- Data sovereignty: User PII must stay in region
- Scale: Start at 20% of India traffic, grow to 100% in 6 months
```

**Phase 2: Multi-Region Architecture**
```
India (ap-south-1):           SEA (ap-southeast-1):
├── ALB + ASG (API)           ├── ALB + ASG (API)
├── Aurora Primary            ├── Aurora Global DB (Secondary)
├── ElastiCache Redis         ├── ElastiCache Redis (separate)
├── Elasticsearch             ├── Elasticsearch (separate)
├── S3 (User data)            ├── S3 (User data - SEA)
└── RabbitMQ                  └── RabbitMQ (separate)

Shared:
├── Route 53 (Geolocation routing: SEA → ap-southeast-1)
├── CloudFront (Global CDN)
├── S3 (Shared assets, cross-region replicated)
└── Centralized Monitoring (us-east-1)
```

**Phase 3: Data Strategy**
```
User Data: Region-local (PDPA compliance)
  - Singapore users → ap-southeast-1 Aurora + S3
  - Indian users → ap-south-1 Aurora + S3

Shared Data (Product catalog, configurations):
  - DynamoDB Global Tables or
  - Aurora Global Database with read-local writes

Analytics:
  - Replicate to centralized data lake (us-east-1)
  - Redact PII before cross-region transfer
```

**Phase 4: Terraform Implementation**
```hcl
# Use the same modules, different tfvars per region
module "sea_infrastructure" {
  source = "../modules/platform"
  
  region             = "ap-southeast-1"
  environment        = "production"
  vpc_cidr           = "10.2.0.0/16"  # Non-overlapping with India
  instance_types     = ["c6g.xlarge", "c6g.2xlarge"]  # Graviton for cost
  aurora_instance    = "db.r6g.2xlarge"
  min_capacity       = 3
  max_capacity       = 30
}
```

**Phase 5: Traffic Migration**
```
Week 1: 5% SEA traffic → new region (canary)
Week 2: 25% → monitor latency, errors
Week 3: 50% → validate database performance
Week 4: 100% → full cutover for SEA
```

---

### Q43: How do you handle a security breach where an IAM access key has been compromised?

**Answer:**

**Immediate Response (First 15 minutes):**

**Step 1: Disable the compromised key:**
```bash
aws iam update-access-key \
  --access-key-id AKIAXXXXXXXXXXXXXXXX \
  --status Inactive \
  --user-name compromised-user
```

**Step 2: Assess blast radius:**
```bash
# Check CloudTrail for actions taken with the key
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAXXXXXXXXXXXXXXXX \
  --start-time "2024-01-14T00:00:00Z" \
  --end-time "2024-01-15T23:59:59Z"

# Check what permissions this key has
aws iam list-user-policies --user-name compromised-user
aws iam list-attached-user-policies --user-name compromised-user
```

**Step 3: Containment:**
```json
// Attach deny-all policy immediately
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*"
  }]
}
```

**Step 4: Investigate unauthorized actions:**
```bash
# Look for:
# - New IAM users/roles created
# - New EC2 instances launched (crypto mining)
# - S3 data exfiltration
# - Security group changes
# - Lambda functions created (backdoors)

# Check for persistence mechanisms
aws iam list-users  # Any new unknown users?
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"  # Unknown instances?
aws lambda list-functions  # Unknown functions?
```

**Step 5: Remediation:**
```
1. Delete compromised access key
2. Rotate all secrets the compromised user had access to
3. Revoke any temporary credentials (STS sessions)
4. Review and remove any resources created by attacker
5. Patch the vulnerability that led to key exposure
```

**Step 6: Post-Incident:**
```
- Enable AWS GuardDuty if not already active
- Implement access key rotation policy (90 days max)
- Enable MFA for all IAM users
- Use IAM Roles instead of long-lived access keys
- Enable AWS Config rule: access-keys-rotated
- Review all access keys: aws iam generate-credential-report
```

---

## Section 15: Behavioral & Design Questions

---

### Q44: Tell me about a time you improved infrastructure reliability that had a measurable business impact.

**Answer Framework (STAR Method):**

**Situation:**
"Our delivery platform was experiencing 2-3 outages per month, each lasting 15-30 minutes. This was causing $50K+ revenue loss per incident and damaging customer trust."

**Task:**
"I was tasked with improving platform reliability from 99.5% to 99.95% uptime within 3 months."

**Action:**
```
1. Implemented comprehensive monitoring with SLOs:
   - Defined SLIs: Availability, Latency (P99), Error Rate
   - Set SLOs: 99.95% availability, P99 < 500ms
   - Created error budget dashboards

2. Eliminated single points of failure:
   - Migrated from single-AZ to multi-AZ for all services
   - Implemented Aurora Global Database for DR
   - Added circuit breakers for all external dependencies

3. Improved deployment safety:
   - Implemented canary deployments (5% → 25% → 100%)
   - Added automated rollback on error rate spike
   - Created staging environment that mirrors production

4. Built chaos engineering practice:
   - Monthly game days with AZ failure simulations
   - Automated chaos experiments with FIS
   - Identified and fixed 12 resilience gaps
```

**Result:**
```
- Uptime improved from 99.5% to 99.97% (exceeded target)
- Incidents reduced from 2-3/month to 1/quarter
- Mean Time to Recovery (MTTR) reduced from 25 min to 4 min
- Estimated revenue saved: $400K/year
- Customer satisfaction (NPS) improved by 15 points
```

---

### Q45: How do you make the decision between building vs. buying (managed services)?

**Answer:**

**Decision Framework:**

| Factor | Build (Self-Managed) | Buy (Managed Service) |
|---|---|---|
| Control needed | Full customization | Standard config sufficient |
| Team expertise | Strong domain expertise | Limited expertise |
| Scale | Massive scale (cost at scale) | Moderate scale |
| Compliance | Specific compliance needs | Standard compliance |
| Time to market | Can wait | Need it now |
| Operational cost | Have dedicated ops team | Minimal ops team |

**Real Example - Elasticsearch:**
```
Option A: Self-managed on EC2
- Pro: Full control, custom plugins, cost at scale ($3K/month for 10-node cluster)
- Con: Operational burden (upgrades, monitoring, scaling), need dedicated engineer

Option B: Amazon OpenSearch Service
- Pro: Managed upgrades, automated snapshots, built-in monitoring ($5K/month)
- Con: Slightly higher cost, limited plugin support, version lag

Decision: Started with OpenSearch (speed to market), migrated to self-managed 
after reaching 50+ nodes (30% cost savings justified operational investment)
```

**Decision Criteria Checklist:**
```
1. Does the managed service meet 90%+ of our requirements?
2. Is the cost premium acceptable vs. engineering time?
3. Do we have the expertise to operate this at scale?
4. Are there compliance/security requirements that prevent managed services?
5. Is this a core competency or undifferentiated heavy lifting?
```

**General Rule:** Buy for undifferentiated work, build for competitive advantage.

---

## Summary: Key Topics Quick Reference

| Topic | Key Concepts to Master |
|---|---|
| EC2/ASG/ALB | Health checks, scaling policies, zero-downtime deployments |
| CodeDeploy | AppSpec, lifecycle hooks, rollback strategies, Blue/Green |
| CloudWatch | Custom metrics, composite alarms, dashboards, anomaly detection |
| Route 53 | Routing policies, failover, health checks, DNS migration |
| Elasticsearch | Cluster sizing, ILM, shard strategy, rolling upgrades |
| S3 | Lifecycle policies, performance optimization, security, cross-account |
| Aurora/RDS | Replication, failover, performance tuning, Global Database |
| Terraform | State management, modules, drift detection, disaster recovery |
| RabbitMQ/Redis | HA clustering, memory management, scaling strategies |
| Security | Defense-in-depth, IAM, encryption, compliance automation |
| Incident Response | OODA loop, runbooks, post-incident reviews, chaos engineering |
| Cost Optimization | Right-sizing, Savings Plans, Spot instances, lifecycle policies |

---

*This document covers industry-level interview questions aligned with the AWS DevOps Engineer role responsibilities for mission-critical, high-concurrency platforms.*
