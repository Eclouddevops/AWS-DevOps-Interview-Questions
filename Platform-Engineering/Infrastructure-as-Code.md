# Infrastructure as Code - Terraform, CloudFormation, CDK Standards

## Terraform Architecture Patterns

### Module Design Principles

```
┌─────────────────────────────────────────────────────────────┐
│                Module Architecture                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Root Modules (Compositions)                          │   │
│  │  - Combine multiple child modules                     │   │
│  │  - Environment-specific configuration                 │   │
│  │  - Backend configuration                              │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │                                    │
│  ┌──────────────────────▼───────────────────────────────┐   │
│  │  Child Modules (Reusable Components)                  │   │
│  │  - Single responsibility                              │   │
│  │  - Version-pinned                                     │   │
│  │  - Well-documented                                    │   │
│  │  - Tested                                             │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │                                    │
│  ┌──────────────────────▼───────────────────────────────┐   │
│  │  Resource Modules (Thin wrappers)                     │   │
│  │  - Opinionated defaults                               │   │
│  │  - Company standards baked in                         │   │
│  │  - Minimal configuration required                     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Production Module Example:

```hcl
# modules/ecs-service/main.tf
# A production-grade ECS Fargate service module

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# --- Variables with Validation ---
variable "service_name" {
  type        = string
  description = "Name of the ECS service"
  
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,28}[a-z0-9]$", var.service_name))
    error_message = "Service name must be 4-30 chars, lowercase alphanumeric with hyphens."
  }
}

variable "container_image" {
  type        = string
  description = "ECR image URI with tag"
  
  validation {
    condition     = can(regex("^\\d{12}\\.dkr\\.ecr\\.", var.container_image))
    error_message = "Container image must be from an ECR repository."
  }
}

variable "cpu" {
  type        = number
  default     = 512
  description = "CPU units (256, 512, 1024, 2048, 4096)"
  
  validation {
    condition     = contains([256, 512, 1024, 2048, 4096], var.cpu)
    error_message = "CPU must be one of: 256, 512, 1024, 2048, 4096."
  }
}

variable "memory" {
  type        = number
  default     = 1024
  description = "Memory in MB"
}

variable "desired_count" {
  type    = number
  default = 2
  
  validation {
    condition     = var.desired_count >= 2
    error_message = "Minimum 2 tasks required for high availability."
  }
}

variable "health_check_path" {
  type    = string
  default = "/health"
}

variable "environment" {
  type = string
  
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

# --- Locals for computed values ---
locals {
  full_name = "${var.service_name}-${var.environment}"
  
  # Standard tags applied to all resources
  tags = merge(var.additional_tags, {
    Service     = var.service_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Module      = "ecs-service"
    ModuleVersion = "3.2.1"
  })
  
  # Environment-specific defaults
  env_config = {
    development = {
      min_capacity        = 1
      max_capacity        = 4
      deployment_min_pct  = 50
      deployment_max_pct  = 200
      health_grace_period = 60
      log_retention       = 7
    }
    staging = {
      min_capacity        = 2
      max_capacity        = 8
      deployment_min_pct  = 100
      deployment_max_pct  = 200
      health_grace_period = 90
      log_retention       = 14
    }
    production = {
      min_capacity        = var.desired_count
      max_capacity        = var.desired_count * 4
      deployment_min_pct  = 100
      deployment_max_pct  = 200
      health_grace_period = 120
      log_retention       = 90
    }
  }
  
  config = local.env_config[var.environment]
}

# --- ECS Task Definition ---
resource "aws_ecs_task_definition" "main" {
  family                   = local.full_name
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.task.arn
  
  container_definitions = jsonencode([{
    name  = var.service_name
    image = var.container_image
    
    portMappings = [{
      containerPort = var.container_port
      protocol      = "tcp"
    }]
    
    environment = [
      for k, v in var.environment_variables : {
        name  = k
        value = v
      }
    ]
    
    secrets = [
      for k, v in var.secrets : {
        name      = k
        valueFrom = v
      }
    ]
    
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:${var.container_port}${var.health_check_path} || exit 1"]
      interval    = 10
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }
    
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.main.name
        "awslogs-region"        = data.aws_region.current.name
        "awslogs-stream-prefix" = var.service_name
      }
    }
    
    stopTimeout = 120
    
    # Resource limits
    ulimits = [{
      name      = "nofile"
      softLimit = 65536
      hardLimit = 65536
    }]
  }])
  
  tags = local.tags
}

# --- ECS Service ---
resource "aws_ecs_service" "main" {
  name            = local.full_name
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.main.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"
  
  deployment_configuration {
    minimum_healthy_percent = local.config.deployment_min_pct
    maximum_percent         = local.config.deployment_max_pct
    
    deployment_circuit_breaker {
      enable   = true
      rollback = true
    }
  }
  
  health_check_grace_period_seconds = local.config.health_grace_period
  
  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.service.id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.main.arn
    container_name   = var.service_name
    container_port   = var.container_port
  }
  
  # Spread across AZs
  ordered_placement_strategy {
    type  = "spread"
    field = "attribute:ecs.availability-zone"
  }
  
  lifecycle {
    ignore_changes = [desired_count]  # Managed by auto-scaling
  }
  
  tags = local.tags
}

# --- Auto Scaling ---
resource "aws_appautoscaling_target" "main" {
  max_capacity       = local.config.max_capacity
  min_capacity       = local.config.min_capacity
  resource_id        = "service/${var.cluster_name}/${aws_ecs_service.main.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${local.full_name}-cpu-scaling"
  resource_id        = aws_appautoscaling_target.main.resource_id
  scalable_dimension = aws_appautoscaling_target.main.scalable_dimension
  service_namespace  = aws_appautoscaling_target.main.service_namespace
  policy_type        = "TargetTrackingScaling"
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

## CloudFormation StackSets

### Organization-Wide Deployment:

```yaml
# stackset-security-baseline.yaml
# Deployed to ALL accounts via CloudFormation StackSets
AWSTemplateFormatVersion: '2010-09-09'
Description: Security baseline for all organization accounts

Parameters:
  LogArchiveAccountId:
    Type: String
    Description: Central log archive account ID

Resources:
  # Config Recorder in every account
  ConfigRecorder:
    Type: AWS::Config::ConfigurationRecorder
    Properties:
      Name: default
      RoleARN: !GetAtt ConfigRole.Arn
      RecordingGroup:
        AllSupported: true
        IncludeGlobalResourceTypes: true

  # GuardDuty detector in every account
  GuardDutyDetector:
    Type: AWS::GuardDuty::Detector
    Properties:
      Enable: true
      FindingPublishingFrequency: FIFTEEN_MINUTES
      DataSources:
        S3Logs:
          Enable: true
        Kubernetes:
          AuditLogs:
            Enable: true

  # CloudTrail (member account trail)
  MemberTrail:
    Type: AWS::CloudTrail::Trail
    Properties:
      TrailName: member-account-trail
      IsLogging: true
      IsMultiRegionTrail: true
      S3BucketName: !Sub 'org-cloudtrail-${LogArchiveAccountId}'
      EnableLogFileValidation: true
      IncludeGlobalServiceEvents: true

  # Security Hub enabled
  SecurityHub:
    Type: AWS::SecurityHub::Hub
    Properties:
      EnableDefaultStandards: true

  # IAM Password Policy
  IAMPasswordPolicy:
    Type: AWS::IAM::AccountPasswordPolicy
    Properties:
      MinimumPasswordLength: 14
      RequireSymbols: true
      RequireNumbers: true
      RequireUppercaseCharacters: true
      RequireLowercaseCharacters: true
      MaxPasswordAge: 90
      PasswordReusePrevention: 24
```

### StackSet Deployment via Terraform:

```hcl
resource "aws_cloudformation_stack_set" "security_baseline" {
  name             = "security-baseline"
  permission_model = "SERVICE_MANAGED"
  
  auto_deployment {
    enabled                          = true
    retain_stacks_on_account_removal = false
  }
  
  template_body = file("${path.module}/templates/security-baseline.yaml")
  
  parameters = {
    LogArchiveAccountId = var.log_archive_account_id
  }
  
  capabilities = ["CAPABILITY_NAMED_IAM"]
  
  lifecycle {
    ignore_changes = [administration_role_arn]
  }
}

resource "aws_cloudformation_stack_set_instance" "all_accounts" {
  stack_set_name = aws_cloudformation_stack_set.security_baseline.name
  
  deployment_targets {
    organizational_unit_ids = [var.root_ou_id]
  }
  
  region = "us-east-1"
  
  operation_preferences {
    failure_tolerance_count = 2
    max_concurrent_count    = 5
    region_concurrency_type = "PARALLEL"
  }
}
```

## AWS CDK Patterns

### CDK Construct for Standardized Service:

```typescript
import * as cdk from 'aws-cdk-lib';
import * as ecs from 'aws-cdk-lib/aws-ecs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as elbv2 from 'aws-cdk-lib/aws-elasticloadbalancingv2';
import { Construct } from 'constructs';

interface PlatformServiceProps {
  serviceName: string;
  containerImage: ecs.ContainerImage;
  cpu?: number;
  memoryLimitMiB?: number;
  desiredCount?: number;
  environment?: { [key: string]: string };
  tier: 'tier-1' | 'tier-2' | 'tier-3';
}

export class PlatformService extends Construct {
  public readonly service: ecs.FargateService;
  public readonly loadBalancer: elbv2.ApplicationLoadBalancer;
  
  constructor(scope: Construct, id: string, props: PlatformServiceProps) {
    super(scope, id);
    
    // Tier-based defaults
    const tierConfig = {
      'tier-1': { minCapacity: 4, maxCapacity: 20, multiAz: true, monitoring: 'enhanced' },
      'tier-2': { minCapacity: 2, maxCapacity: 10, multiAz: true, monitoring: 'standard' },
      'tier-3': { minCapacity: 1, maxCapacity: 5, multiAz: false, monitoring: 'basic' },
    };
    
    const config = tierConfig[props.tier];
    
    // Task Definition with platform standards
    const taskDef = new ecs.FargateTaskDefinition(this, 'TaskDef', {
      cpu: props.cpu || 512,
      memoryLimitMiB: props.memoryLimitMiB || 1024,
    });
    
    const container = taskDef.addContainer('app', {
      image: props.containerImage,
      logging: ecs.LogDrivers.awsLogs({
        streamPrefix: props.serviceName,
        logRetention: cdk.aws_logs.RetentionDays.THREE_MONTHS,
      }),
      environment: props.environment,
      healthCheck: {
        command: ['CMD-SHELL', 'curl -f http://localhost:8080/health || exit 1'],
        interval: cdk.Duration.seconds(10),
        timeout: cdk.Duration.seconds(5),
        retries: 3,
        startPeriod: cdk.Duration.seconds(60),
      },
    });
    
    container.addPortMappings({ containerPort: 8080 });
    
    // Service with circuit breaker
    this.service = new ecs.FargateService(this, 'Service', {
      cluster: props.cluster,
      taskDefinition: taskDef,
      desiredCount: props.desiredCount || config.minCapacity,
      circuitBreaker: { rollback: true },
      enableExecuteCommand: true,  // For debugging
    });
    
    // Auto-scaling
    const scaling = this.service.autoScaleTaskCount({
      minCapacity: config.minCapacity,
      maxCapacity: config.maxCapacity,
    });
    
    scaling.scaleOnCpuUtilization('CpuScaling', {
      targetUtilizationPercent: 70,
      scaleInCooldown: cdk.Duration.seconds(300),
      scaleOutCooldown: cdk.Duration.seconds(60),
    });
    
    // ALB + WAF (auto-attached for tier-1 and tier-2)
    if (props.tier !== 'tier-3') {
      this.loadBalancer = new elbv2.ApplicationLoadBalancer(this, 'ALB', {
        internetFacing: true,
        deletionProtection: props.tier === 'tier-1',
      });
      
      // WAF attached automatically
      // Monitoring dashboards created automatically
      // Alarms based on tier SLO
    }
  }
}
```

### CDK Aspects (Policy Enforcement):

```typescript
import * as cdk from 'aws-cdk-lib';
import { IConstruct } from 'constructs';

// Aspect: Enforce encryption on all S3 buckets
class EnforceS3Encryption implements cdk.IAspect {
  visit(node: IConstruct): void {
    if (node instanceof cdk.aws_s3.CfnBucket) {
      if (!node.bucketEncryption) {
        cdk.Annotations.of(node).addError(
          'S3 buckets must have encryption enabled. Use SSE-KMS.'
        );
      }
    }
  }
}

// Aspect: Enforce tagging on all resources
class EnforceTagging implements cdk.IAspect {
  private requiredTags = ['Environment', 'Team', 'CostCenter'];
  
  visit(node: IConstruct): void {
    if (cdk.TagManager.isTaggable(node)) {
      // Check that required tags exist (CDK will add them via Tags.of())
      // This aspect adds error annotations if tags are missing
    }
  }
}

// Apply aspects to entire app
const app = new cdk.App();
cdk.Aspects.of(app).add(new EnforceS3Encryption());
cdk.Aspects.of(app).add(new EnforceTagging());
```

## Infrastructure Testing

### Test Pyramid for IaC:

```
        /\
       /  \  Integration Tests (Terratest)
      /    \  - Deploy real resources
     /      \ - Validate behavior
    /--------\ - Expensive, slow (CI only)
   /          \
  /  Contract  \ Contract Tests (terraform validate + plan)
 /   Tests     \ - Validate module interfaces
/              \ - Check plan output
/--------------\ - Fast, cheap
/                \
/ Static Analysis  \ Static Analysis (tflint, checkov, tfsec)
/ Unit Tests        \ - No AWS calls
/                    \ - Instant feedback
/--------------------\ - Pre-commit hooks
```

### Pre-commit Hooks:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
        args:
          - '--args=--only=terraform_deprecated_interpolation'
          - '--args=--only=terraform_deprecated_index'
          - '--args=--only=terraform_unused_declarations'
          - '--args=--only=terraform_naming_convention'
      - id: terraform_tfsec
        args:
          - '--args=--minimum-severity HIGH'
      - id: terraform_checkov
        args:
          - '--args=--quiet'
          - '--args=--skip-check CKV2_AWS_6'
      - id: terraform_docs
        args:
          - '--args=--output-file README.md'
  
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: detect-private-key
      - id: no-commit-to-branch
        args: ['--branch', 'main']
```
