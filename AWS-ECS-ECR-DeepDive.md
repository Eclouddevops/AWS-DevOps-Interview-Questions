# AWS ECS & ECR — Deep-Dive Interview Q&A

## Table of Contents
1. [ECS Architecture](#ecs-architecture)
2. [Task Definitions & Services](#task-definitions--services)
3. [ECS Networking](#ecs-networking)
4. [ECR](#ecr)
5. [Scaling & Deployment](#scaling--deployment)
6. [Tricky Scenarios](#tricky-scenarios)

---

## ECS Architecture

**Q1: Explain ECS architecture — Cluster, Service, Task, Task Definition. How do they relate?**

**A:**

```
ECS Cluster (logical grouping)
├── Service A (manages desired count, LB integration, deployment)
│   ├── Task 1 (running instance of task definition)
│   │   ├── Container: app (main application)
│   │   └── Container: sidecar (logging/monitoring)
│   └── Task 2
├── Service B
│   └── Task 3
└── Standalone Task (not managed by service — batch jobs)

Infrastructure:
├── Fargate (serverless — AWS manages hosts)
└── EC2 (you manage EC2 instances with ECS agent)
```

**Key differences:**
- **Task Definition**: Blueprint (like Dockerfile for containers) — image, CPU, memory, ports, env vars
- **Task**: Running instance of a task definition (like a pod in K8s)
- **Service**: Ensures desired number of tasks running, handles deployments, LB registration
- **Cluster**: Logical boundary (like a namespace)

**Tricky**: A Task can have multiple containers that share network namespace (like a K8s pod). They communicate via `localhost`. But unlike K8s, ECS tasks don't have built-in service discovery between tasks — you need Cloud Map or ALB.

---

**Q2: Compare ECS Fargate vs EC2 launch type. When would you choose each?**

**A:**

| Feature | Fargate | EC2 |
|---------|---------|-----|
| Infrastructure management | None (serverless) | You manage instances |
| Pricing | Per vCPU/GB per second | EC2 instance cost + ECS free |
| Cost efficiency | Higher per-unit cost | Lower (reserved instances, spot) |
| GPU support | No | Yes |
| Startup time | 30-60 seconds | Depends on instance pool |
| Max resources/task | 16 vCPU, 120 GB | Instance-limited |
| EBS volumes | Ephemeral (20GB default, up to 200GB) | Full EBS support |
| Docker socket | No access | Full access |
| Privileged mode | No | Yes |
| Custom AMI | No | Yes |
| Placement constraints | Limited (AZ) | Full (instance type, attribute) |

**Choose Fargate when:** Small-medium workloads, variable traffic, team doesn't want to manage infrastructure, security isolation needed.

**Choose EC2 when:** GPU workloads, need privileged containers, cost optimization with Reserved Instances/Spot, need EBS persistent volumes, high memory/CPU requirements.

---

**Q3: Explain ECS task networking modes — bridge, host, awsvpc.**

**A:**

| Mode | Description | Use Case |
|------|-------------|----------|
| bridge | Docker bridge network, port mapping | Legacy, multiple tasks sharing host |
| host | Container uses host network directly | Performance (no NAT), port conflicts possible |
| awsvpc | Each task gets its own ENI (own IP) | Fargate (required), security groups per task |

**awsvpc (recommended for new workloads):**
```
Task gets:
├── Own private IP (from subnet CIDR)
├── Own security group (task-level isolation!)
├── Own ENI (Elastic Network Interface)
└── Can use VPC features (flow logs, NACLs)
```

**Tricky**: awsvpc on EC2 has ENI limits! Each EC2 instance has a max number of ENIs. A c5.large supports ~3 ENIs = max 3 tasks per instance. Use ENI trunking (ECS managed):
```bash
# Enable ENI trunking for more tasks per instance
aws ecs put-account-setting --name awsvpcTrunking --value enabled
```

---

## Task Definitions & Services

**Q4: What are the critical task definition parameters? Explain CPU/memory allocation.**

**A:**

```json
{
  "family": "web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.2",
      "cpu": 768,
      "memory": 1536,
      "memoryReservation": 1024,
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/web-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "app"
        }
      },
      "secrets": [
        {"name": "DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:db-pass"}
      ],
      "essential": true
    }
  ],
  "executionRoleArn": "arn:aws:iam::123:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123:role/ecsTaskRole"
}
```

**CPU/Memory in Fargate (valid combinations):**
```
CPU (units) | Memory (MB)
256         | 512, 1024, 2048
512         | 1024-4096
1024        | 2048-8192
2048        | 4096-16384
4096        | 8192-30720
8192        | 16384-61440
16384       | 32768-122880
```

**Tricky**: `executionRoleArn` vs `taskRoleArn`:
- **Execution Role**: Used by ECS agent to pull images from ECR, push logs to CloudWatch, get secrets
- **Task Role**: Used by YOUR application code to access AWS services (S3, DynamoDB, etc.)
- Common mistake: putting application permissions on execution role instead of task role

---

**Q5: Explain ECS deployment strategies — rolling, blue/green, external.**

**A:**

**Rolling Update (default):**
```json
{
  "deploymentConfiguration": {
    "maximumPercent": 200,
    "minimumHealthyPercent": 100,
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    }
  }
}
```
- Replaces tasks gradually (new tasks start → old tasks drain)
- `minimumHealthyPercent: 100` = always maintain full capacity
- `maximumPercent: 200` = can temporarily have 2x tasks during deployment

**Blue/Green (via CodeDeploy):**
```
Blue TG (current v1) ← ALB listener
Green TG (new v2) ← Test listener

Steps:
1. Deploy v2 tasks → register in Green TG
2. Test listener validates Green TG
3. Shift production listener: Blue → Green
4. Monitor (automatic rollback on alarm)
5. Terminate Blue tasks
```

**Tricky**: Circuit breaker with rollback automatically detects failed deployments (tasks can't reach RUNNING state) and rolls back. Without it, a bad image keeps failing and ECS retries forever.

---

## ECR

**Q6: How do you secure ECR? Explain image scanning, lifecycle policies, and access control.**

**A:**

**Image Scanning:**
```bash
# Enable scan-on-push
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Enhanced scanning (Inspector integration — continuous)
# Detects: OS vulnerabilities, programming language vulnerabilities
# Scans continuously (not just on push)
```

**Lifecycle Policies (auto-cleanup):**
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged after 1 day",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": {"type": "expire"}
    }
  ]
}
```

**Cross-account access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::OTHER_ACCOUNT:root"},
    "Action": ["ecr:GetDownloadUrlForLayer", "ecr:BatchGetImage"]
  }]
}
```

**Tricky**: ECR authentication tokens expire after 12 hours. In CI/CD pipelines, always refresh: `aws ecr get-login-password | docker login --username AWS --password-stdin <ecr-url>`

---

## Scaling & Deployment

**Q7: How does ECS Service Auto Scaling work? What metrics should you scale on?**

**A:**

**Scaling options:**
1. **Target Tracking**: Maintain metric at target value
2. **Step Scaling**: Different actions at different thresholds
3. **Scheduled Scaling**: Time-based

**Best metrics to scale on:**
| Metric | Use When |
|--------|----------|
| CPU Utilization | CPU-bound workloads |
| Memory Utilization | Memory-bound workloads |
| ALB Request Count per Target | Web services (most accurate for HTTP) |
| SQS Queue Depth / Tasks | Queue processors |
| Custom CloudWatch Metric | Business-specific metrics |

```json
{
  "TargetTrackingScalingPolicyConfiguration": {
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/my-alb/123/targetgroup/my-tg/456"
    },
    "TargetValue": 1000,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }
}
```

**Tricky**: For Fargate, scaling adds NEW tasks (30-60s cold start). For burst traffic, you may need to over-provision or combine with CloudFront caching. Also: ECS scaling only changes task count — for EC2 launch type, you also need ASG to scale the underlying instances (Capacity Providers solve this).

---

**Q8: Explain ECS Capacity Providers. How do they automate EC2 scaling for ECS?**

**A:**

```
Without Capacity Providers:
- ECS wants to place task → No capacity → Task stuck in PENDING
- You must manually ensure ASG has enough instances

With Capacity Providers:
- ECS wants to place task → Asks Capacity Provider → ASG scales automatically
- Target: Keep 80% capacity utilization (efficient)
```

```json
{
  "capacityProviders": [
    {
      "name": "on-demand-cp",
      "autoScalingGroupProvider": {
        "autoScalingGroupArn": "arn:aws:autoscaling:...",
        "managedScaling": {
          "status": "ENABLED",
          "targetCapacity": 80,
          "minimumScalingStepSize": 1,
          "maximumScalingStepSize": 10
        },
        "managedTerminationProtection": "ENABLED"
      }
    },
    {
      "name": "spot-cp",
      "autoScalingGroupProvider": {
        "autoScalingGroupArn": "arn:aws:autoscaling:spot-asg..."
      }
    }
  ],
  "defaultCapacityProviderStrategy": [
    {"capacityProvider": "on-demand-cp", "weight": 1, "base": 2},
    {"capacityProvider": "spot-cp", "weight": 3}
  ]
}
```

**Result**: First 2 tasks always on On-Demand (base=2), remaining split 75% Spot / 25% On-Demand.

---

## Tricky Scenarios

**Q9: ECS tasks keep failing with "CannotPullContainerError". What are all possible causes?**

**A:**

1. **ECR permissions**: Task execution role missing `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`
2. **Network**: Task in private subnet without NAT Gateway or VPC endpoint for ECR
3. **Image doesn't exist**: Wrong tag, wrong repository name
4. **ECR endpoint**: Need both `ecr.api` AND `ecr.dkr` VPC endpoints (or `com.amazonaws.region.ecr.api` + `com.amazonaws.region.ecr.dkr` + S3 gateway endpoint for image layers)
5. **Rate limiting**: Docker Hub rate limits (100 pulls/6h anonymous, 200 authenticated)
6. **Security group**: Outbound blocked on port 443
7. **Image too large + timeout**: Fargate has a pull timeout

**Fix checklist:**
```bash
# Check execution role has ECR permissions
# Check VPC endpoints or NAT gateway
# Check security group allows outbound 443
# Check image exists: aws ecr describe-images --repository-name myapp
# Check for S3 gateway endpoint (ECR stores layers in S3!)
```

---

**Q10: How do you handle secrets and sensitive configuration in ECS tasks?**

**A:**

```json
{
  "containerDefinitions": [{
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:prod/db:password::"
      },
      {
        "name": "API_KEY",
        "valueFrom": "arn:aws:ssm:us-east-1:123:parameter/prod/api-key"
      }
    ],
    "environment": [
      {"name": "APP_ENV", "value": "production"}
    ]
  }]
}
```

**Sources:**
- Secrets Manager: `arn:aws:secretsmanager:region:account:secret:name:json-key:version-stage:version-id`
- SSM Parameter Store: `arn:aws:ssm:region:account:parameter/name`

**Requirements:**
- Execution role needs `secretsmanager:GetSecretValue` and/or `ssm:GetParameters`
- KMS decrypt permission if secrets are KMS-encrypted

**Tricky**: Secrets are injected at task LAUNCH time. If you rotate a secret, running tasks still have the OLD value. Must redeploy (force new deployment) to pick up new secrets. Use Secrets Manager rotation Lambda + ECS deployment trigger for automatic refresh.



---

## Additional Scenario-Based Tricky Questions

---

**Q11: Your ECS Fargate task starts and immediately exits with "CannotPullContainerError." ECR image exists and you can pull it locally. The task ran fine yesterday. What changed?**

**A:**

**Common causes (ordered by frequency):**
```
1. VPC ENDPOINT OR NAT GATEWAY ISSUE:
   - Fargate tasks in private subnet need NAT GW or VPC endpoints to reach ECR
   - NAT GW was deleted/misconfigured
   - Or: VPC endpoint for ECR was removed
   
2. ECR REPOSITORY POLICY CHANGED:
   - Someone modified the ECR repo policy → task role no longer has pull access
   - Or: cross-account pull permission revoked

3. IMAGE TAG OVERWRITTEN WITH BROKEN IMAGE:
   - Someone pushed a new image with same tag (:latest or :v1.2.3)
   - New image is corrupt or wrong architecture (amd64 vs arm64)

4. ECR LIFECYCLE POLICY DELETED THE IMAGE:
   - Lifecycle rule: "keep only last 10 images"
   - 11th image pushed → old tag deleted
   - Task definition references deleted tag

5. STS TOKEN EXPIRED (cross-account):
   - Task role assumes cross-account role to pull from ECR in another account
   - Role trust policy changed → AssumeRole fails → can't authenticate to ECR
```

**Debugging:**
```bash
# Check stopped task details
aws ecs describe-tasks --cluster prod --tasks <task-arn> \
  --query 'tasks[].containers[].{reason:reason,lastStatus:lastStatus}'

# Check if image exists in ECR
aws ecr describe-images --repository-name myapp --image-ids imageTag=v1.2.3

# Check task execution role permissions
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456:role/ecsTaskExecutionRole \
  --action-names ecr:GetAuthorizationToken ecr:GetDownloadUrlForLayer ecr:BatchGetImage

# Check VPC endpoint / NAT connectivity
# For private subnets, you need EITHER:
# - NAT Gateway (route 0.0.0.0/0 → nat-gw)
# - VPC endpoints for: ecr.api, ecr.dkr, AND s3 (ECR uses S3 for layers!)
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.us-east-1.ecr.dkr
```

**Fix:**
```bash
# Create required VPC endpoints for private Fargate tasks
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.ecr.api --vpc-endpoint-type Interface \
  --subnet-ids subnet-a subnet-b --security-group-ids sg-xxx

aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.ecr.dkr --vpc-endpoint-type Interface \
  --subnet-ids subnet-a subnet-b --security-group-ids sg-xxx

# Don't forget S3 endpoint (ECR layers stored in S3!)
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 --vpc-endpoint-type Gateway \
  --route-table-ids rtb-private
```

**Tricky**: ECR stores image layers in S3. When Fargate pulls an image, it authenticates with ECR (ecr.dkr endpoint) then downloads layers from S3. If you create VPC endpoints for ECR but forget the S3 Gateway Endpoint, pulls will PARTIALLY work (auth succeeds) but then TIMEOUT downloading layers. The error message is unhelpful: "CannotPullContainerError: context deadline exceeded." Always create all three endpoints: ecr.api, ecr.dkr, AND s3.

---

**Q12: Your ECS service keeps cycling between RUNNING and STOPPED states. Task starts, runs for 30 seconds, then stops. ECS launches a new task, same thing happens. Container logs show the application starting successfully. What's killing it?**

**A:**

**Investigation:**
```bash
# Check stopped task reason
aws ecs describe-tasks --cluster prod --tasks $(aws ecs list-tasks --cluster prod \
  --service-name myservice --desired-status STOPPED --query 'taskArns[0]' --output text) \
  --query 'tasks[].{stopCode:stopCode,stoppedReason:stoppedReason,containers:containers[].{exitCode:exitCode,reason:reason}}'

# Common stoppedReason values:
# "Essential container in task exited" → container crashed
# "Task failed ELB health checks" → ALB health check failed
# "Scaling activity initiated by..." → desired count reduced
# "DAEMON task stopped" → daemon service constraints
```

**The "app starts fine but dies after 30s" causes:**
```
1. ALB HEALTH CHECK FAILING:
   - Task starts → registers with target group → health check begins
   - Health check path: /health (returns 200 locally)
   - But in ECS: app listens on port 8080, health check configured for port 80
   - 30 seconds = health check timeout → ALB marks unhealthy → ECS stops task
   
2. CONTAINER HEALTH CHECK FAILING:
   - Task definition has healthCheck configured
   - Command: ["CMD-SHELL", "curl -f http://localhost:8080/health"]
   - But container doesn't have curl installed! → health check always fails
   - After startPeriod + (retries × interval) = task marked unhealthy

3. GRACEFUL SHUTDOWN SIGNAL HANDLING:
   - ECS sends SIGTERM when stopping tasks
   - App doesn't handle SIGTERM → doesn't shut down within stopTimeout (30s)
   - ECS sends SIGKILL → exit code 137
   - ECS replaces task → same cycle

4. RESOURCE LIMITS:
   - Memory limit: 512MB
   - App uses 512MB at startup then spikes to 520MB → OOMKilled
   - Exit code: 137 (SIGKILL from OOM)
```

**Fix for health check issues:**
```json
// Task definition — proper health check configuration
{
  "healthCheck": {
    "command": ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1"],
    "interval": 10,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 60
  }
}
// startPeriod: 60 = give app 60 seconds to start before health checks begin
// Use wget instead of curl (alpine images have wget but not curl)
```

**Tricky**: ECS health check and ALB health check are SEPARATE. If BOTH are configured, EITHER can kill your task. ECS container health check failure → task marked UNHEALTHY → service scheduler replaces it. ALB health check failure → target deregistered → ECS sees it and may replace. Disable one of them to avoid double-kill scenarios. Best practice: use ALB health check only (set `healthCheck` in task definition to null) and configure ALB health check with generous `HealthyThresholdCount` and `HealthCheckGracePeriodSeconds` on the ECS service.

---

**Q13: You need to deploy a new ECS service version with zero downtime. The new version requires a database migration that takes 2 minutes. During migration, the old version must continue serving traffic. How do you orchestrate this?**

**A:**

**Blue-Green deployment with migration:**
```
Timeline:
T0: Old version (v1) serving traffic ✓
T1: Run database migration (backward-compatible)
T2: Deploy new version (v2) alongside v1
T3: Health checks pass on v2
T4: Shift traffic from v1 → v2
T5: Drain v1 (deregistration delay)
T6: Stop v1 tasks
```

**Implementation with ECS + CodeDeploy Blue/Green:**
```bash
# Step 1: Migration must be BACKWARD COMPATIBLE
# v1 must still work after migration runs (expand-contract pattern)
# Migration adds new columns, doesn't remove/rename old ones

# Step 2: ECS Blue/Green deployment
aws deploy create-deployment --application-name myapp \
  --deployment-group-name prod-bg \
  --revision '{
    "revisionType": "AppSpecContent",
    "appSpecContent": {
      "content": "{\"version\":1,\"Resources\":[{\"TargetService\":{\"Type\":\"AWS::ECS::Service\",\"Properties\":{\"TaskDefinition\":\"arn:aws:ecs:...:task-definition/myapp:42\",\"LoadBalancerInfo\":{\"ContainerName\":\"myapp\",\"ContainerPort\":8080}}}}],\"Hooks\":[{\"BeforeAllowTraffic\":\"arn:aws:lambda:...:function:run-migration\"},{\"AfterAllowTraffic\":\"arn:aws:lambda:...:function:smoke-test\"}]}"
    }
  }'

# CodeDeploy lifecycle hooks:
# BeforeInstall: (create new task set)
# AfterInstall: (new tasks running, not receiving traffic)
# BeforeAllowTraffic: RUN MIGRATION HERE ← Lambda runs DB migration
# AllowTraffic: (shift traffic to new tasks)
# AfterAllowTraffic: RUN SMOKE TESTS ← Lambda verifies new version works
```

**Alternative: ECS rolling update with pre-deployment Job:**
```yaml
# Step 1: Run migration as ECS task (one-off, not service)
aws ecs run-task --cluster prod \
  --task-definition myapp-migration:5 \
  --network-configuration '...' \
  --overrides '{"containerOverrides":[{"name":"migrate","command":["python","manage.py","migrate"]}]}'

# Wait for migration task to complete
aws ecs wait tasks-stopped --cluster prod --tasks $MIGRATION_TASK_ARN

# Step 2: Update service with new task definition (rolling)
aws ecs update-service --cluster prod --service myapp \
  --task-definition myapp:42 \
  --deployment-configuration '{
    "minimumHealthyPercent": 100,
    "maximumPercent": 200
  }'
# minimumHealthyPercent=100: old tasks stay until new ones are healthy
# maximumPercent=200: allows doubling capacity during transition
```

**Tricky**: The `minimumHealthyPercent: 100` + `maximumPercent: 200` pattern means ECS launches new tasks FIRST (doubling capacity), waits for them to be healthy, then drains old tasks. But this doubles your Fargate cost during deployment (running 2x tasks). For large services, use `minimumHealthyPercent: 50` to allow removing some old tasks before all new ones are ready — faster but riskier. Also, set `healthCheckGracePeriodSeconds` on the service (default 0) to give new tasks time to start before ALB health checks begin failing them.

---
