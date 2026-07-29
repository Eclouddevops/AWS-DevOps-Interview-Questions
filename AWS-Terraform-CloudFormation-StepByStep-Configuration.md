# AWS Terraform & CloudFormation - Step-by-Step Configuration Guide

## Table of Contents
1. [Terraform Setup & Backend](#1-terraform-setup--backend)
2. [Project Structure](#2-project-structure)
3. [VPC & Networking Module](#3-vpc--networking-module)
4. [Compute Module (EC2/ASG/ALB)](#4-compute-module)
5. [Database Module (Aurora)](#5-database-module)
6. [State Management](#6-state-management)
7. [CI/CD for Terraform](#7-cicd-for-terraform)
8. [CloudFormation Basics](#8-cloudformation-basics)
9. [Drift Detection & Compliance](#9-drift-detection--compliance)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. Terraform Setup & Backend

### Step 1.1: Install Terraform


```bash
# Install Terraform (Linux)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Verify
terraform version

# Install tfenv (version manager - recommended)
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
tfenv install 1.7.0
tfenv use 1.7.0
```

### Step 1.2: Configure S3 Backend with DynamoDB Locking

```bash
# Create S3 bucket for state
aws s3api create-bucket --bucket company-terraform-state --region us-east-1
aws s3api put-bucket-versioning --bucket company-terraform-state --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket company-terraform-state \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms"}}]}'

# Create DynamoDB table for locking
aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=Purpose,Value=terraform-state-locking
```

**backend.tf:**
```hcl
terraform {
  required_version = ">= 1.7.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }

  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "production/us-east-1/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Team        = "platform"
      Project     = "myapp"
    }
  }
}
```

---

## 2. Project Structure

```
terraform/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   │   ├── main.tf
│   │   ├── asg.tf
│   │   ├── alb.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf
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
│       └── us-west-2/
│           ├── main.tf
│           ├── terraform.tfvars
│           └── backend.tf
├── global/
│   ├── iam/
│   ├── route53/
│   └── s3/
└── scripts/
    ├── plan.sh
    └── apply.sh
```

---

## 3. VPC & Networking Module

### modules/networking/main.tf

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${var.environment}-vpc" }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.environment}-igw" }
}

# Public Subnets (for ALB)
resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = { Name = "${var.environment}-public-${var.availability_zones[count.index]}" }
}

# Private Subnets (for EC2/Application)
resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + length(var.availability_zones))
  availability_zone = var.availability_zones[count.index]

  tags = { Name = "${var.environment}-private-${var.availability_zones[count.index]}" }
}

# Database Subnets (for RDS)
resource "aws_subnet" "database" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + 2 * length(var.availability_zones))
  availability_zone = var.availability_zones[count.index]

  tags = { Name = "${var.environment}-database-${var.availability_zones[count.index]}" }
}

# NAT Gateway (one per AZ for HA)
resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
  tags   = { Name = "${var.environment}-nat-eip-${count.index}" }
}

resource "aws_nat_gateway" "main" {
  count         = length(var.availability_zones)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = { Name = "${var.environment}-nat-${var.availability_zones[count.index]}" }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = { Name = "${var.environment}-public-rt" }
}

resource "aws_route_table" "private" {
  count  = length(var.availability_zones)
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  tags = { Name = "${var.environment}-private-rt-${count.index}" }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(var.availability_zones)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# VPC Endpoints (reduce NAT costs)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.region}.s3"
  route_table_ids = concat(
    [aws_route_table.public.id],
    aws_route_table.private[*].id
  )
}

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

---

## 4. Compute Module

### modules/compute/main.tf

```hcl
# Launch Template
resource "aws_launch_template" "app" {
  name_prefix   = "${var.environment}-app-"
  image_id      = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name

  iam_instance_profile { arn = var.instance_profile_arn }

  vpc_security_group_ids = [aws_security_group.app.id]

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 50
      volume_type           = "gp3"
      iops                  = 3000
      throughput            = 125
      encrypted             = true
      delete_on_termination = true
    }
  }

  metadata_options {
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
    http_endpoint               = "enabled"
  }

  monitoring { enabled = true }

  user_data = base64encode(templatefile("${path.module}/userdata.sh.tpl", {
    environment = var.environment
    region      = var.region
  }))

  tag_specifications {
    resource_type = "instance"
    tags = { Name = "${var.environment}-app" }
  }

  lifecycle {
    create_before_destroy = true
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "${var.environment}-app-asg"
  vpc_zone_identifier = var.private_subnet_ids
  min_size            = var.asg_min_size
  max_size            = var.asg_max_size
  desired_capacity    = var.asg_desired_capacity
  health_check_type   = "ELB"
  health_check_grace_period = 300
  target_group_arns   = [aws_lb_target_group.app.arn]
  termination_policies = ["OldestLaunchTemplate", "OldestInstance"]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
      instance_warmup        = 300
    }
  }

  tag {
    key                 = "Name"
    value               = "${var.environment}-app"
    propagate_at_launch = true
  }
}

# Target Tracking Scaling Policy
resource "aws_autoscaling_policy" "cpu" {
  name                   = "${var.environment}-cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value     = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}

# ALB
resource "aws_lb" "app" {
  name               = "${var.environment}-app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = var.environment == "production"
  idle_timeout               = 120

  access_logs {
    bucket  = var.alb_logs_bucket
    prefix  = "${var.environment}/app-alb"
    enabled = true
  }
}

resource "aws_lb_target_group" "app" {
  name     = "${var.environment}-app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    path                = "/health"
    port                = 8080
    healthy_threshold   = 3
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 15
    matcher             = "200"
  }

  deregistration_delay = 60
  slow_start           = 90
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.app.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

## 5. Database Module

### modules/database/main.tf

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-aurora-subnet-group"
  subnet_ids = var.database_subnet_ids
}

resource "aws_rds_cluster" "main" {
  cluster_identifier     = "${var.environment}-aurora-cluster"
  engine                 = "aurora-postgresql"
  engine_version         = var.engine_version
  database_name          = var.database_name
  master_username        = var.master_username
  manage_master_user_password = true
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]
  
  backup_retention_period      = 35
  preferred_backup_window      = "03:00-04:00"
  preferred_maintenance_window = "sun:05:00-sun:06:00"
  
  storage_encrypted = true
  kms_key_id        = var.kms_key_arn
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  deletion_protection             = var.environment == "production"
  copy_tags_to_snapshot           = true
  skip_final_snapshot             = var.environment != "production"
  final_snapshot_identifier       = var.environment == "production" ? "${var.environment}-aurora-final-${formatdate("YYYYMMDDHHmmss", timestamp())}" : null
}

resource "aws_rds_cluster_instance" "writer" {
  identifier         = "${var.environment}-aurora-writer"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = var.writer_instance_class
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version

  monitoring_interval          = 60
  monitoring_role_arn          = var.monitoring_role_arn
  performance_insights_enabled = true
  performance_insights_retention_period = 731

  tags = { Role = "writer" }
}

resource "aws_rds_cluster_instance" "readers" {
  count              = var.reader_count
  identifier         = "${var.environment}-aurora-reader-${count.index + 1}"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = var.reader_instance_class
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version
  promotion_tier     = count.index + 1

  monitoring_interval          = 60
  monitoring_role_arn          = var.monitoring_role_arn
  performance_insights_enabled = true

  tags = { Role = "reader" }
}
```

---

## 6. State Management

### Step 6.1: Common State Operations

```bash
# Initialize Terraform
terraform init

# Plan changes
terraform plan -out=plan.tfplan -var-file=terraform.tfvars

# Apply changes
terraform apply plan.tfplan

# List resources in state
terraform state list

# Show specific resource
terraform state show aws_instance.app

# Move resource (rename without destroy/create)
terraform state mv aws_instance.old_name aws_instance.new_name

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0

# Remove from state (without destroying)
terraform state rm aws_instance.temp

# Force unlock state
terraform force-unlock LOCK-ID
```

---

## 7. CI/CD for Terraform

### GitHub Actions Pipeline

```yaml
name: Terraform CI/CD
on:
  pull_request:
    paths: ['terraform/**']
  push:
    branches: [main]
    paths: ['terraform/**']

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: {terraform_version: "1.7.0"}

      - name: Terraform Init
        run: terraform init
        working-directory: terraform/environments/production/us-east-1

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -out=plan.tfplan -no-color
        working-directory: terraform/environments/production/us-east-1

      - name: Security Scan (tfsec)
        uses: aquasecurity/tfsec-action@v1.0.0
        with: {working_directory: terraform/}

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Apply
        run: |
          terraform init
          terraform apply -auto-approve -var-file=terraform.tfvars
        working-directory: terraform/environments/production/us-east-1
```

---

## 8. CloudFormation Basics

### Step 8.1: VPC CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Production VPC with 3 AZs

Parameters:
  Environment:
    Type: String
    Default: production
  VpcCidr:
    Type: String
    Default: '10.0.0.0/16'

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-igw'

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

Outputs:
  VpcId:
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-VpcId'
```

### Step 8.2: CloudFormation CLI Commands

```bash
# Create stack
aws cloudformation create-stack \
  --stack-name production-vpc \
  --template-body file://vpc.yaml \
  --parameters ParameterKey=Environment,ParameterValue=production

# Update stack
aws cloudformation update-stack \
  --stack-name production-vpc \
  --template-body file://vpc.yaml

# Create change set (preview changes)
aws cloudformation create-change-set \
  --stack-name production-vpc \
  --change-set-name update-cidr \
  --template-body file://vpc.yaml

# Execute change set
aws cloudformation execute-change-set \
  --stack-name production-vpc \
  --change-set-name update-cidr

# Detect drift
aws cloudformation detect-stack-drift --stack-name production-vpc
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id xxx
```

---

## 9. Drift Detection & Compliance

```bash
# Terraform: Check for drift
terraform plan -detailed-exitcode
# Exit code 0 = no changes, 2 = drift detected

# Schedule drift detection (cron job or Lambda)
# terraform plan output saved and compared

# tfsec security scanning
tfsec . --format json --out results.json

# checkov policy scanning
checkov -d . --framework terraform --output json

# Infracost (cost estimation)
infracost breakdown --path .
```

---

## 10. Production Best Practices

```hcl
# Pin provider versions
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.30" }
  }
}

# Use data sources for existing resources
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Use locals for computed values
locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
  name_prefix = "${var.environment}-${var.project}"
}

# Prevent accidental destruction
resource "aws_rds_cluster" "main" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}

# Use moved blocks for refactoring
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

### Quick Reference Commands

```bash
terraform init              # Initialize working directory
terraform plan              # Preview changes
terraform apply             # Apply changes
terraform destroy           # Destroy all resources
terraform fmt -recursive    # Format all files
terraform validate          # Validate configuration
terraform output            # Show outputs
terraform graph | dot -Tpng > graph.png  # Visualize dependencies
terraform workspace list    # List workspaces
terraform workspace new dev # Create workspace
```

---
