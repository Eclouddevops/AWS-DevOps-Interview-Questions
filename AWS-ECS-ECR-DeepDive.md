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
