# AWS Docker & ECS - Step-by-Step Configuration Guide

## Table of Contents
1. [ECR Repository Setup](#1-ecr-repository-setup)
2. [Dockerfile Best Practices](#2-dockerfile-best-practices)
3. [Build & Push Images](#3-build--push-images)
4. [ECS Cluster Creation](#4-ecs-cluster-creation)
5. [Task Definition](#5-task-definition)
6. [ECS Service Configuration](#6-ecs-service-configuration)
7. [Auto Scaling](#7-auto-scaling)
8. [CI/CD for ECS](#8-cicd-for-ecs)
9. [Monitoring & Logging](#9-monitoring--logging)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. ECR Repository Setup

### Step 1.1: Create ECR Repository


```bash
# Create repository
aws ecr create-repository \
  --repository-name myapp/api \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=KMS,kmsKey=arn:aws:kms:us-east-1:123456789012:key/xxx \
  --tags Key=Environment,Value=production

# Create lifecycle policy (keep last 20 images)
aws ecr put-lifecycle-policy \
  --repository-name myapp/api \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Keep last 20 tagged images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v", "release"],
          "countType": "imageCountMoreThan",
          "countNumber": 20
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "Remove untagged after 7 days",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 7
        },
        "action": {"type": "expire"}
      }
    ]
  }'
```

---

## 2. Dockerfile Best Practices

### Step 2.1: Production Python/Flask Dockerfile

```dockerfile
# Multi-stage build
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM python:3.11-slim
WORKDIR /app

# Security: Non-root user
RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser

# Copy dependencies
COPY --from=builder /root/.local /home/appuser/.local
ENV PATH=/home/appuser/.local/bin:$PATH

# Copy application
COPY --chown=appuser:appuser . .

# Security: Read-only filesystem support
RUN mkdir -p /app/tmp && chown appuser:appuser /app/tmp

USER appuser

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", \
     "--workers", "4", "--threads", "2", \
     "--timeout", "120", "--graceful-timeout", "30", \
     "--keep-alive", "65", \
     "--max-requests", "1000", "--max-requests-jitter", "50", \
     "--access-logfile", "-", "--error-logfile", "-", \
     "app:create_app()"]
```

---

## 3. Build & Push Images

### Step 3.1: Authenticate and Push

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build image
docker build -t myapp/api:v1.2.3 .
docker tag myapp/api:v1.2.3 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/api:v1.2.3
docker tag myapp/api:v1.2.3 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/api:latest

# Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/api:v1.2.3
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/api:latest

# Verify scan results
aws ecr describe-image-scan-findings \
  --repository-name myapp/api \
  --image-id imageTag=v1.2.3 \
  --query 'imageScanFindings.findingSeverityCounts'
```

---

## 4. ECS Cluster Creation

### Step 4.1: Create Fargate Cluster

```bash
aws ecs create-cluster \
  --cluster-name production-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy '[
    {"capacityProvider": "FARGATE", "weight": 70, "base": 2},
    {"capacityProvider": "FARGATE_SPOT", "weight": 30}
  ]' \
  --configuration '{
    "executeCommandConfiguration": {
      "logging": "OVERRIDE",
      "logConfiguration": {
        "cloudWatchLogGroupName": "/ecs/exec-logs",
        "cloudWatchEncryptionEnabled": true
      }
    }
  }' \
  --settings '[{"name": "containerInsights", "value": "enabled"}]' \
  --tags Key=Environment,Value=production
```

---

## 5. Task Definition

### Step 5.1: Create Task Definition

```bash
aws ecs register-task-definition \
  --family myapp-api \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu "1024" \
  --memory "2048" \
  --execution-role-arn arn:aws:iam::123456789012:role/ecsTaskExecutionRole \
  --task-role-arn arn:aws:iam::123456789012:role/ecsTaskRole \
  --runtime-platform '{"cpuArchitecture": "X86_64", "operatingSystemFamily": "LINUX"}' \
  --container-definitions '[
    {
      "name": "api",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/api:v1.2.3",
      "essential": true,
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "environment": [
        {"name": "ENVIRONMENT", "value": "production"},
        {"name": "PORT", "value": "8080"}
      ],
      "secrets": [
        {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:production/database/url"},
        {"name": "REDIS_URL", "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/production/myapp/redis_url"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "api",
          "awslogs-create-group": "true"
        }
      },
      "ulimits": [{"name": "nofile", "softLimit": 65535, "hardLimit": 65535}],
      "linuxParameters": {
        "initProcessEnabled": true
      }
    }
  ]'
```

### Step 5.2: IAM Roles for ECS

```bash
# Task Execution Role (ECR pull, logs, secrets)
aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Statement": [{"Effect": "Allow", "Principal": {"Service": "ecs-tasks.amazonaws.com"}, "Action": "sts:AssumeRole"}]
  }'
aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Task Role (what the container can do)
aws iam create-role --role-name ecsTaskRole \
  --assume-role-policy-document '{
    "Statement": [{"Effect": "Allow", "Principal": {"Service": "ecs-tasks.amazonaws.com"}, "Action": "sts:AssumeRole"}]
  }'
```

---

## 6. ECS Service Configuration

### Step 6.1: Create Service with ALB

```bash
aws ecs create-service \
  --cluster production-cluster \
  --service-name myapp-api-service \
  --task-definition myapp-api:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --platform-version LATEST \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-private-az1", "subnet-private-az2", "subnet-private-az3"],
      "securityGroups": ["sg-ecs-app123456"],
      "assignPublicIp": "DISABLED"
    }
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/myapp-api-tg/xxx",
    "containerName": "api",
    "containerPort": 8080
  }]' \
  --deployment-configuration '{
    "deploymentCircuitBreaker": {"enable": true, "rollback": true},
    "maximumPercent": 200,
    "minimumHealthyPercent": 100
  }' \
  --deployment-controller '{"type": "ECS"}' \
  --enable-execute-command \
  --health-check-grace-period-seconds 120 \
  --tags Key=Environment,Value=production
```

### Step 6.2: Blue/Green Deployment (CodeDeploy)

```bash
aws ecs create-service \
  --cluster production-cluster \
  --service-name myapp-api-bluegreen \
  --task-definition myapp-api:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --deployment-controller '{"type": "CODE_DEPLOY"}' \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-private-az1", "subnet-private-az2", "subnet-private-az3"],
      "securityGroups": ["sg-ecs-app123456"],
      "assignPublicIp": "DISABLED"
    }
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/blue-tg/xxx",
    "containerName": "api",
    "containerPort": 8080
  }]'
```

---

## 7. Auto Scaling

### Step 7.1: Target Tracking Scaling

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production-cluster/myapp-api-service \
  --min-capacity 3 \
  --max-capacity 30

# CPU-based scaling
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production-cluster/myapp-api-service \
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

# Memory-based scaling
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production-cluster/myapp-api-service \
  --policy-name memory-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 75.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'

# Request count scaling
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/production-cluster/myapp-api-service \
  --policy-name request-count-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/production-api-alb/xxx/targetgroup/myapp-api-tg/yyy"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

---

## 8. CI/CD for ECS

### Step 8.1: Deployment Script

```bash
#!/bin/bash
# deploy-ecs.sh - Deploy new image to ECS

set -e

CLUSTER="production-cluster"
SERVICE="myapp-api-service"
TASK_FAMILY="myapp-api"
IMAGE_TAG="${1:-latest}"
ECR_REPO="123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp/api"

echo "Deploying ${ECR_REPO}:${IMAGE_TAG} to ${CLUSTER}/${SERVICE}"

# Get current task definition
TASK_DEF=$(aws ecs describe-task-definition --task-definition $TASK_FAMILY --query 'taskDefinition')

# Update image in task definition
NEW_TASK_DEF=$(echo $TASK_DEF | jq --arg IMAGE "${ECR_REPO}:${IMAGE_TAG}" \
  '.containerDefinitions[0].image = $IMAGE |
   del(.taskDefinitionArn, .revision, .status, .requiresAttributes, .compatibilities, .registeredAt, .registeredBy)')

# Register new task definition
NEW_REVISION=$(aws ecs register-task-definition --cli-input-json "$NEW_TASK_DEF" \
  --query 'taskDefinition.taskDefinitionArn' --output text)

echo "New task definition: $NEW_REVISION"

# Update service
aws ecs update-service \
  --cluster $CLUSTER \
  --service $SERVICE \
  --task-definition $NEW_REVISION \
  --force-new-deployment

# Wait for deployment
echo "Waiting for deployment to complete..."
aws ecs wait services-stable --cluster $CLUSTER --services $SERVICE

echo "Deployment complete!"
```

---

## 9. Monitoring & Logging

### Step 9.1: ECS Alarms

```bash
# Service CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-API-HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ClusterName,Value=production-cluster Name=ServiceName,Value=myapp-api-service \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Running task count
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-API-LowTaskCount" \
  --metric-name RunningTaskCount \
  --namespace ECS/ContainerInsights \
  --statistic Minimum \
  --period 60 \
  --threshold 2 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ClusterName,Value=production-cluster Name=ServiceName,Value=myapp-api-service \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical
```

---

## 10. Production Best Practices

### Step 10.1: ECS Exec (Debug Containers)

```bash
# Enable ECS Exec on service (already done in service creation)
# Access container shell
aws ecs execute-command \
  --cluster production-cluster \
  --task arn:aws:ecs:us-east-1:123456789012:task/production-cluster/abc123 \
  --container api \
  --interactive \
  --command "/bin/sh"
```

### Step 10.2: Quick Reference Commands

```bash
# List services
aws ecs list-services --cluster production-cluster

# Describe service
aws ecs describe-services --cluster production-cluster --services myapp-api-service

# List running tasks
aws ecs list-tasks --cluster production-cluster --service-name myapp-api-service

# Force new deployment (restart all tasks)
aws ecs update-service --cluster production-cluster --service myapp-api-service --force-new-deployment

# Scale service manually
aws ecs update-service --cluster production-cluster --service myapp-api-service --desired-count 10

# View task logs
aws logs tail /ecs/myapp-api --follow --since 5m

# Stop a specific task
aws ecs stop-task --cluster production-cluster --task arn:aws:ecs:...:task/xxx --reason "Manual stop for debugging"

# Deregister old task definitions
aws ecs deregister-task-definition --task-definition myapp-api:1
```

---
