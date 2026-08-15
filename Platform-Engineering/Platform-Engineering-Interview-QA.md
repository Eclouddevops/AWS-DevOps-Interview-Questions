# Platform Engineering - Tricky Production-Based Interview Questions & Answers

## IaC, GitOps, CI/CD Architecture

### Q1: Your Terraform state file got corrupted after two engineers ran `terraform apply` simultaneously on the same workspace. How do you recover and prevent this in production?

**Answer:**

**Immediate Recovery:**

```bash
# Step 1: Check if state lock was bypassed
# If using S3 backend with DynamoDB locking, this shouldn't happen
# But if it did (e.g., someone used -lock=false):

# Step 2: List state versions (S3 versioning MUST be enabled)
aws s3api list-object-versions \
  --bucket terraform-state-bucket \
  --prefix env/production/terraform.tfstate \
  --max-keys 10

# Step 3: Download the last known good version
aws s3api get-object \
  --bucket terraform-state-bucket \
  --key env/production/terraform.tfstate \
  --version-id "GOOD_VERSION_ID" \
  terraform.tfstate.backup

# Step 4: Verify the backup is valid
terraform show terraform.tfstate.backup

# Step 5: Upload recovered state
aws s3 cp terraform.tfstate.backup \
  s3://terraform-state-bucket/env/production/terraform.tfstate

# Step 6: Run terraform plan to verify no drift
terraform plan
```

**Prevention - Proper Backend Configuration:**

```hcl
# backend.tf - Production-grade state management
terraform {
  backend "s3" {
    bucket         = "company-terraform-state-prod"
    key            = "platform/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/terraform-state-key"
    
    # DynamoDB for state locking (CRITICAL)
    dynamodb_table = "terraform-state-locks"
    
    # Prevent accidental deletion
    # S3 bucket has versioning + MFA delete enabled
  }
}

# DynamoDB lock table
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  # Point-in-time recovery for lock table
  point_in_time_recovery { enabled = true }
  
  tags = { Purpose = "terraform-state-locking" }
}
```

**Prevention - CI/CD Only Applies:**

```yaml
# GitHub Actions - Only CI/CD can apply (no local applies)
name: Terraform Apply
on:
  push:
    branches: [main]  # Only after merge to main
    paths: ['infrastructure/**']

jobs:
  apply:
    runs-on: ubuntu-latest
    concurrency:
      group: terraform-${{ github.ref }}  # Only one apply at a time
      cancel-in-progress: false           # Don't cancel running applies!
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Terraform Init
        run: terraform init -backend-config=backend.hcl
      
      - name: Terraform Plan
        run: terraform plan -out=tfplan -lock=true -lock-timeout=300s
      
      - name: Terraform Apply
        run: terraform apply -lock=true tfplan
```

**Prevention - State Bucket Protection:**

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "company-terraform-state-prod"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

# Prevent deletion of state objects
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    id     = "keep-old-versions"
    status = "Enabled"
    noncurrent_version_expiration { noncurrent_days = 90 }
  }
}

# Bucket policy: Deny delete except by break-glass role
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "DenyDeleteExceptBreakGlass"
      Effect    = "Deny"
      Principal = "*"
      Action    = ["s3:DeleteObject", "s3:DeleteObjectVersion"]
      Resource  = "${aws_s3_bucket.terraform_state.arn}/*"
      Condition = {
        StringNotLike = {
          "aws:PrincipalArn" = "arn:aws:iam::*:role/BreakGlassRole"
        }
      }
    }]
  })
}
```

---

### Q2: You manage 500+ Terraform resources across 20 AWS accounts. Plans take 15+ minutes and are becoming unmanageable. How do you restructure?

**Answer:**

**Problem: Monolithic State Anti-Pattern**
```
BEFORE (Monolithic):
terraform-repo/
├── main.tf          # 5000+ lines
├── variables.tf     # 500+ variables
├── outputs.tf
└── terraform.tfstate  # 500+ resources, 15 min plan
```

**Solution: State Decomposition Strategy**

```
AFTER (Layered Architecture):
infrastructure/
├── layers/
│   ├── 01-foundation/          # Accounts, OUs, SCPs (rarely changes)
│   │   ├── main.tf
│   │   ├── accounts.tf
│   │   └── state: s3://state/01-foundation/
│   │
│   ├── 02-networking/          # VPCs, TGW, DNS (changes monthly)
│   │   ├── main.tf
│   │   ├── vpc.tf
│   │   ├── transit-gateway.tf
│   │   └── state: s3://state/02-networking/
│   │
│   ├── 03-security/            # IAM, KMS, GuardDuty (changes weekly)
│   │   ├── main.tf
│   │   ├── iam-roles.tf
│   │   └── state: s3://state/03-security/
│   │
│   ├── 04-data-platform/       # Databases, caches (changes frequently)
│   │   ├── main.tf
│   │   ├── rds.tf
│   │   ├── elasticache.tf
│   │   └── state: s3://state/04-data-platform/
│   │
│   └── 05-applications/        # ECS, Lambda (changes daily)
│       ├── team-a/
│       ├── team-b/
│       └── state: s3://state/05-apps/{team}/
│
├── modules/                    # Shared modules (versioned)
│   ├── vpc/
│   ├── ecs-service/
│   ├── rds-cluster/
│   └── lambda-function/
│
└── terragrunt.hcl             # DRY configuration
```

**Terragrunt for DRY Multi-Account:**

```hcl
# infrastructure/terragrunt.hcl (root)
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite"
  }
  config = {
    bucket         = "company-terraform-state-${get_aws_account_id()}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Generate provider configuration
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
provider "aws" {
  region = "${local.region}"
  
  default_tags {
    tags = {
      Environment = "${local.environment}"
      ManagedBy   = "terraform"
      Layer       = "${local.layer}"
    }
  }
}
EOF
}

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  env_vars     = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  
  environment = local.env_vars.locals.environment
  region      = local.region_vars.locals.region
  account_id  = local.account_vars.locals.account_id
}
```

```hcl
# infrastructure/production/us-east-1/networking/terragrunt.hcl
terraform {
  source = "../../../../modules//vpc"
}

include "root" {
  path = find_in_parent_folders()
}

# Cross-state reference (read networking state from foundation layer)
dependency "foundation" {
  config_path = "../foundation"
}

inputs = {
  vpc_cidr           = "10.1.0.0/16"
  transit_gateway_id = dependency.foundation.outputs.transit_gateway_id
  environment        = "production"
}
```

**Performance Optimization:**

```hcl
# Target specific resources when you know what changed
# terraform plan -target=module.ecs_service  (5 seconds vs 15 minutes)

# Use refresh=false when you trust state
# terraform plan -refresh=false  (skip API calls to verify state)

# Parallelism for large applies
# terraform apply -parallelism=20  (default is 10)
```

**Module Versioning:**

```hcl
# Pin module versions (prevent breaking changes)
module "vpc" {
  source  = "git::https://github.com/company/terraform-modules.git//vpc?ref=v3.2.1"
  # OR using private registry:
  source  = "app.terraform.io/company/vpc/aws"
  version = "~> 3.2"
}
```

---

### Q3: How do you implement policy-as-code to prevent engineers from deploying non-compliant infrastructure? Give specific examples for a financial services company.

**Answer:**

**Multi-Layer Policy Enforcement:**

```
┌─────────────────────────────────────────────────────────────┐
│                Policy Enforcement Layers                      │
│                                                              │
│  Layer 1: Pre-commit (Developer IDE)                        │
│  ├── tflint (syntax/best practices)                         │
│  ├── checkov (misconfigurations)                            │
│  └── tfsec (security scanning)                              │
│                                                              │
│  Layer 2: CI Pipeline (Pull Request)                        │
│  ├── OPA/Conftest (custom business rules)                   │
│  ├── Sentinel (HashiCorp Terraform Cloud)                   │
│  └── Checkov with custom policies                           │
│                                                              │
│  Layer 3: Deployment (terraform apply)                      │
│  ├── CloudFormation Hooks (proactive controls)              │
│  └── Terraform Cloud Sentinel policies                      │
│                                                              │
│  Layer 4: Runtime (AWS Config)                              │
│  ├── AWS Config Rules (detective)                           │
│  ├── SCPs (preventive)                                      │
│  └── Auto-remediation Lambda                                │
└─────────────────────────────────────────────────────────────┘
```

**OPA (Open Policy Agent) Policies:**

```rego
# policy/terraform/encryption.rego
package terraform.encryption

# Deny unencrypted S3 buckets
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket"
    resource.change.after.server_side_encryption_configuration == null
    msg := sprintf("S3 bucket '%s' must have encryption enabled", [resource.address])
}

# Deny unencrypted RDS instances
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.after.storage_encrypted != true
    msg := sprintf("RDS instance '%s' must have storage encryption enabled", [resource.address])
}

# Deny unencrypted EBS volumes
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_ebs_volume"
    resource.change.after.encrypted != true
    msg := sprintf("EBS volume '%s' must be encrypted", [resource.address])
}
```

```rego
# policy/terraform/networking.rego
package terraform.networking

# Deny public S3 buckets
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_public_access_block"
    after := resource.change.after
    any([
        after.block_public_acls != true,
        after.block_public_policy != true,
        after.ignore_public_acls != true,
        after.restrict_public_buckets != true
    ])
    msg := sprintf("S3 bucket '%s' must block all public access", [resource.address])
}

# Deny security groups with 0.0.0.0/0 ingress (except port 443)
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group_rule"
    resource.change.after.type == "ingress"
    resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
    resource.change.after.from_port != 443
    msg := sprintf("Security group rule '%s' allows unrestricted access on port %d", 
        [resource.address, resource.change.after.from_port])
}

# Deny RDS instances that are publicly accessible
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.after.publicly_accessible == true
    msg := sprintf("RDS instance '%s' must not be publicly accessible", [resource.address])
}
```

```rego
# policy/terraform/compliance.rego
package terraform.compliance

# Mandatory tags for PCI-DSS compliance
required_tags := {"Environment", "DataClassification", "Owner", "CostCenter", "Compliance"}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_instance"
    tags := object.get(resource.change.after, "tags", {})
    missing := required_tags - {key | tags[key]}
    count(missing) > 0
    msg := sprintf("EC2 instance '%s' is missing required tags: %v", [resource.address, missing])
}

# Deny instances without IMDSv2
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_instance"
    metadata := object.get(resource.change.after, "metadata_options", [{}])[0]
    metadata.http_tokens != "required"
    msg := sprintf("EC2 instance '%s' must enforce IMDSv2 (http_tokens=required)", [resource.address])
}
```

**CI Pipeline Integration:**

```yaml
# .github/workflows/terraform-compliance.yml
name: Terraform Compliance Check
on: [pull_request]

jobs:
  policy-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0
      
      - name: Terraform Init & Plan
        run: |
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json
      
      - name: OPA Policy Check
        run: |
          opa eval \
            --data policy/ \
            --input tfplan.json \
            --format pretty \
            "data.terraform.encryption.deny" | tee results.txt
          
          # Fail if any denies
          if grep -q "deny" results.txt; then
            echo "❌ Policy violations found!"
            exit 1
          fi
      
      - name: Checkov Security Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          check: CKV_AWS_18,CKV_AWS_19,CKV_AWS_21,CKV_AWS_145
          soft_fail: false
      
      - name: tfsec Security Analysis
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false
          additional_args: --minimum-severity HIGH
```

**Checkov Custom Policy:**

```python
# custom_policies/ensure_backup_tags.py
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from checkov.common.models.enums import CheckResult, CheckCategories

class EnsureBackupTagExists(BaseResourceCheck):
    def __init__(self):
        name = "Ensure all databases have BackupTier tag"
        id = "CUSTOM_AWS_001"
        supported_resources = ['aws_db_instance', 'aws_rds_cluster', 'aws_dynamodb_table']
        categories = [CheckCategories.BACKUP_AND_RECOVERY]
        super().__init__(name=name, id=id, categories=categories, supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        tags = conf.get("tags", [{}])
        if isinstance(tags, list):
            tags = tags[0] if tags else {}
        
        if "BackupTier" in tags:
            return CheckResult.PASSED
        return CheckResult.FAILED

check = EnsureBackupTagExists()
```

---

### Q4: Design a CI/CD pipeline architecture that deploys to 20+ AWS accounts across 3 regions with proper security controls. How do you handle secrets, permissions, and rollback?

**Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                   CI/CD Pipeline Architecture                     │
│                                                                   │
│  ┌───────────┐    ┌────────────────────────────────────────┐    │
│  │ GitHub    │───▶│  Shared Services Account               │    │
│  │ (Source)  │    │  (CI/CD Tools)                          │    │
│  └───────────┘    │                                         │    │
│                   │  ┌─────────────────────────────────┐   │    │
│                   │  │  CodePipeline / GitHub Actions    │   │    │
│                   │  │  ┌──────┐ ┌──────┐ ┌──────────┐│   │    │
│                   │  │  │Build ├─│Test  ├─│Security  ││   │    │
│                   │  │  │      │ │      │ │Scan      ││   │    │
│                   │  │  └──────┘ └──────┘ └──────────┘│   │    │
│                   │  └─────────────────┬───────────────┘   │    │
│                   │                    │                     │    │
│                   │  ┌─────────────────▼───────────────┐   │    │
│                   │  │  Deployment Orchestrator         │   │    │
│                   │  │  (Step Functions / CodeDeploy)   │   │    │
│                   │  └─────────────┬───────────────────┘   │    │
│                   └────────────────┼────────────────────────┘    │
│                                    │                              │
│              ┌─────────────────────┼──────────────────────┐      │
│              │                     │                      │      │
│  ┌───────────▼──────┐  ┌──────────▼────────┐  ┌─────────▼──┐  │
│  │  Dev Accounts     │  │  Staging Accounts  │  │  Prod Accts │  │
│  │  (us-east-1)     │  │  (us-east-1,       │  │  (us-east-1 │  │
│  │                   │  │   eu-west-1)       │  │   us-west-2 │  │
│  │  Auto-deploy     │  │                    │  │   eu-west-1)│  │
│  │  on merge        │  │  Auto-deploy       │  │             │  │
│  └───────────────────┘  │  after dev passes  │  │  Manual     │  │
│                          └────────────────────┘  │  approval + │  │
│                                                   │  canary     │  │
│                                                   └─────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Cross-Account IAM Setup:**

```hcl
# In Shared Services Account - Pipeline Role
resource "aws_iam_role" "pipeline" {
  name = "cicd-pipeline-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "codepipeline.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_policy" "pipeline_cross_account" {
  name = "cicd-cross-account-deploy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "sts:AssumeRole"
      Resource = [
        "arn:aws:iam::DEV_ACCOUNT:role/CICDDeployRole",
        "arn:aws:iam::STAGING_ACCOUNT:role/CICDDeployRole",
        "arn:aws:iam::PROD_ACCOUNT:role/CICDDeployRole"
      ]
    }]
  })
}

# In Target Account (e.g., Production) - Deploy Role
resource "aws_iam_role" "deploy" {
  name = "CICDDeployRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::SHARED_SERVICES_ACCOUNT:role/cicd-pipeline-role"
      }
      Action = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "aws:PrincipalOrgID" = var.org_id
          "sts:ExternalId" = "cicd-deploy-prod"
        }
      }
    }]
  })
  
  # Least privilege - only what's needed for deployment
  inline_policy {
    name = "deploy-permissions"
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
        {
          Effect = "Allow"
          Action = [
            "ecs:UpdateService",
            "ecs:DescribeServices",
            "ecs:RegisterTaskDefinition",
            "ecs:DescribeTaskDefinition"
          ]
          Resource = "*"
          Condition = {
            StringEquals = { "aws:ResourceTag/ManagedBy" = "cicd" }
          }
        },
        {
          Effect = "Allow"
          Action = ["ecr:GetAuthorizationToken", "ecr:BatchGetImage", "ecr:GetDownloadUrlForLayer"]
          Resource = "*"
        }
      ]
    })
  }
}
```

**Secret Management:**

```hcl
# AWS Secrets Manager with cross-account access
resource "aws_secretsmanager_secret" "app_secrets" {
  name                    = "prod/payment-api/secrets"
  kms_key_id              = aws_kms_key.secrets.arn
  recovery_window_in_days = 7
  
  # Automatic rotation
  rotation_rules {
    automatically_after_days = 30
  }
}

# NEVER put secrets in pipeline environment variables!
# Instead, fetch at deploy time:
```

```yaml
# GitHub Actions with OIDC (no long-lived credentials!)
name: Deploy to Production
on:
  push:
    branches: [main]

permissions:
  id-token: write  # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    
    steps:
      - uses: actions/checkout@v4
      
      # OIDC - No secrets stored in GitHub!
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::SHARED_SERVICES:role/GitHubActionsRole
          aws-region: us-east-1
          role-session-name: github-actions-deploy
      
      # Fetch secrets at deploy time (not stored anywhere)
      - name: Get Secrets
        run: |
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id prod/payment-api/secrets \
            --query SecretString --output text)
          # Use in memory only, never echo/log
          echo "::add-mask::$SECRET"
      
      - name: Deploy with Canary
        run: |
          # Deploy to 10% first
          ./scripts/canary-deploy.sh --percentage 10 --region us-east-1
          
          # Wait and validate
          sleep 300
          ./scripts/validate-canary.sh
          
          # If passed, deploy to 100%
          ./scripts/canary-deploy.sh --percentage 100 --region us-east-1
```

**OIDC Configuration (Eliminates stored credentials):**

```hcl
# GitHub OIDC Provider in AWS
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = ["sts.amazonaws.com"]
  
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# Role that GitHub Actions can assume
resource "aws_iam_role" "github_actions" {
  name = "GitHubActionsRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # Only specific repos/branches can assume this role
          "token.actions.githubusercontent.com:sub" = [
            "repo:company/infra-repo:ref:refs/heads/main",
            "repo:company/app-repo:ref:refs/heads/main"
          ]
        }
      }
    }]
  })
}
```

**Automated Rollback:**

```python
# CodeDeploy with automatic rollback on CloudWatch alarm
import boto3

def setup_deployment_with_rollback():
    codedeploy = boto3.client('codedeploy')
    
    codedeploy.create_deployment_group(
        applicationName='payment-api',
        deploymentGroupName='production',
        deploymentConfigName='CodeDeployDefault.ECSCanary10Percent5Minutes',
        
        ecsServices=[{
            'serviceName': 'payment-api',
            'clusterName': 'production'
        }],
        
        # Auto-rollback configuration
        autoRollbackConfiguration={
            'enabled': True,
            'events': [
                'DEPLOYMENT_FAILURE',
                'DEPLOYMENT_STOP_ON_ALARM'  # Rollback when alarm fires
            ]
        },
        
        # CloudWatch alarms that trigger rollback
        alarmConfiguration={
            'enabled': True,
            'alarms': [
                {'name': 'payment-api-5xx-rate-high'},
                {'name': 'payment-api-p99-latency-high'},
                {'name': 'payment-api-error-budget-burn'}
            ]
        },
        
        blueGreenDeploymentConfiguration={
            'terminateBlueInstancesOnDeploymentSuccess': {
                'action': 'TERMINATE',
                'terminationWaitTimeInMinutes': 60  # Keep old version for 1 hour
            },
            'deploymentReadyOption': {
                'actionOnTimeout': 'CONTINUE_DEPLOYMENT',
                'waitTimeInMinutes': 0
            }
        }
    )
```

---

### Q5: Your team uses Terraform modules shared across 15 teams. A breaking change in a shared module caused outages in 3 teams' infrastructure. How do you implement safe module evolution?

**Answer:**

**Module Versioning Strategy:**

```
terraform-modules/                    # Shared module repository
├── modules/
│   ├── vpc/
│   │   ├── v1/                      # Legacy (deprecated, still supported)
│   │   ├── v2/                      # Current stable
│   │   └── v3/                      # Next (breaking changes)
│   ├── ecs-service/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── CHANGELOG.md             # Document ALL changes
│   │   ├── MIGRATION.md             # Migration guide for breaking changes
│   │   └── examples/
│   │       ├── basic/
│   │       └── advanced/
│   └── rds-cluster/
├── tests/                           # Automated tests for every module
│   ├── vpc_test.go
│   ├── ecs_service_test.go
│   └── rds_cluster_test.go
└── .github/
    └── workflows/
        └── test-and-release.yml
```

**Semantic Versioning for Modules:**

```hcl
# Consumer usage - pinned to MINOR version
module "vpc" {
  source  = "git::https://github.com/company/terraform-modules.git//modules/vpc?ref=v2.3.1"
  # OR with private registry:
  source  = "app.terraform.io/company/vpc/aws"
  version = "~> 2.3"  # Allows 2.3.x but not 2.4.0
}

# Versioning rules:
# v1.0.0 → v1.0.1: Bug fix (patch) - backward compatible
# v1.0.0 → v1.1.0: New feature (minor) - backward compatible
# v1.0.0 → v2.0.0: Breaking change (major) - requires migration
```

**Automated Module Testing (Terratest):**

```go
// tests/ecs_service_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
)

func TestEcsServiceModule(t *testing.T) {
    t.Parallel()
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/ecs-service/examples/basic",
        Vars: map[string]interface{}{
            "service_name": "test-service",
            "container_image": "nginx:latest",
            "desired_count": 2,
        },
    })
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    // Verify outputs
    serviceArn := terraform.Output(t, terraformOptions, "service_arn")
    assert.Contains(t, serviceArn, "arn:aws:ecs")
    
    // Verify the ECS service is running
    cluster := terraform.Output(t, terraformOptions, "cluster_name")
    services := aws.GetEcsServices(t, "us-east-1", cluster)
    assert.Equal(t, 1, len(services))
}

func TestEcsServiceModuleBreakingChange(t *testing.T) {
    t.Parallel()
    
    // Test that v1 configuration still works with v2 module
    // This catches breaking changes before release
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/ecs-service/examples/v1-compatible",
    })
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
}
```

**CI Pipeline for Module Releases:**

```yaml
# .github/workflows/module-release.yml
name: Module Test and Release
on:
  pull_request:
    paths: ['modules/**']
  push:
    tags: ['v*']

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      modules: ${{ steps.changes.outputs.modules }}
    steps:
      - uses: actions/checkout@v4
      - id: changes
        run: |
          # Detect which modules changed
          MODULES=$(git diff --name-only HEAD~1 | grep "^modules/" | cut -d'/' -f2 | sort -u | jq -R -s 'split("\n")[:-1]')
          echo "modules=$MODULES" >> $GITHUB_OUTPUT
  
  test:
    needs: detect-changes
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: ${{ fromJson(needs.detect-changes.outputs.modules) }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
      
      - name: Terraform Validate
        run: |
          cd modules/${{ matrix.module }}
          terraform init -backend=false
          terraform validate
      
      - name: tfsec Security Scan
        run: tfsec modules/${{ matrix.module }} --minimum-severity HIGH
      
      - name: Checkov Compliance
        run: checkov -d modules/${{ matrix.module }} --framework terraform
      
      - name: Terratest
        run: |
          cd tests
          go test -v -run "Test.*${{ matrix.module }}.*" -timeout 30m
  
  release:
    needs: [test]
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - name: Create Release
        uses: actions/create-release@v1
        with:
          tag_name: ${{ github.ref_name }}
          release_name: ${{ github.ref_name }}
          body: "See CHANGELOG.md for details"
      
      - name: Notify Teams of New Version
        run: |
          # Post to Slack with migration guide if major version
          VERSION="${{ github.ref_name }}"
          if [[ "$VERSION" =~ ^v[0-9]+\.0\.0$ ]]; then
            ./scripts/notify-breaking-change.sh "$VERSION"
          fi
```

**Migration Path for Breaking Changes:**

```hcl
# modules/vpc/variables.tf

# DEPRECATED: Use vpc_config instead
variable "cidr_block" {
  type        = string
  default     = null
  description = "DEPRECATED in v3.0 - Use vpc_config.cidr_block instead"
  
  validation {
    condition     = var.cidr_block == null
    error_message = "The cidr_block variable is deprecated. Use vpc_config instead. See MIGRATION.md"
  }
}

# NEW in v3.0
variable "vpc_config" {
  type = object({
    cidr_block          = string
    secondary_cidr_blocks = optional(list(string), [])
    enable_ipv6         = optional(bool, false)
  })
  description = "VPC configuration block (replaces individual variables in v3.0)"
}

# Backward compatibility shim (supports both old and new interface)
locals {
  effective_cidr = coalesce(
    try(var.vpc_config.cidr_block, null),
    var.cidr_block,
    "10.0.0.0/16"  # default
  )
}
```

---

### Q6: How do you implement GitOps for Kubernetes (EKS) deployments with ArgoCD while maintaining security and compliance in a multi-team environment?

**Answer:**

**GitOps Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitOps Architecture                            │
│                                                                   │
│  ┌──────────────┐     ┌──────────────────────────────────────┐  │
│  │ App Repo     │     │ Config Repo (GitOps State)           │  │
│  │              │     │                                       │  │
│  │ src/         │     │ clusters/                             │  │
│  │ Dockerfile   │────▶│ ├── production/                       │  │
│  │ .github/     │ CI  │ │   ├── apps/                        │  │
│  │              │     │ │   │   ├── payment-api/              │  │
│  └──────────────┘     │ │   │   │   ├── kustomization.yaml   │  │
│                        │ │   │   │   ├── deployment.yaml      │  │
│        CI pushes       │ │   │   │   └── hpa.yaml             │  │
│        new image tag   │ │   │   └── order-service/            │  │
│        to config repo  │ │   ├── platform/                    │  │
│                        │ │   │   ├── istio/                   │  │
│                        │ │   │   ├── monitoring/              │  │
│                        │ │   │   └── cert-manager/            │  │
│                        │ │   └── argocd/                      │  │
│                        │ │       └── applicationsets.yaml     │  │
│                        │ └── staging/                         │  │
│                        │     └── (same structure)             │  │
│                        └──────────────────┬───────────────────┘  │
│                                           │                      │
│                                           │ ArgoCD syncs         │
│                                           ▼                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              EKS Cluster                                     ││
│  │  ┌──────────────────────────────────────────────────────┐   ││
│  │  │  ArgoCD (management namespace)                        │   ││
│  │  │  - ApplicationSets (multi-team)                       │   ││
│  │  │  - App of Apps pattern                                │   ││
│  │  │  - RBAC per team                                      │   ││
│  │  └──────────────────────────────────────────────────────┘   ││
│  │                                                              ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ││
│  │  │team-a ns │  │team-b ns │  │platform  │  │monitoring│  ││
│  │  │          │  │          │  │namespace │  │namespace │  ││
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

**ArgoCD ApplicationSet (Multi-Team):**

```yaml
# clusters/production/argocd/applicationset-teams.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: team-applications
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/company/gitops-config.git
        revision: main
        directories:
          - path: "clusters/production/apps/*"
  
  template:
    metadata:
      name: "{{path.basename}}"
      labels:
        team: "{{path.basename}}"
    spec:
      project: "{{path.basename}}-project"  # ArgoCD project per team
      source:
        repoURL: https://github.com/company/gitops-config.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
      
      syncPolicy:
        automated:
          prune: true       # Remove resources not in Git
          selfHeal: true    # Revert manual changes
          allowEmpty: false # Don't sync if source is empty
        
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
          - ApplyOutOfSyncOnly=true
        
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m
```

**ArgoCD RBAC (Per-Team Isolation):**

```yaml
# ArgoCD ConfigMap for RBAC
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # Team A can manage only their applications
    p, role:team-a, applications, get, team-a-project/*, allow
    p, role:team-a, applications, sync, team-a-project/*, allow
    p, role:team-a, applications, override, team-a-project/*, deny
    p, role:team-a, applications, delete, team-a-project/*, deny
    p, role:team-a, logs, get, team-a-project/*, allow
    
    # Team B similar
    p, role:team-b, applications, get, team-b-project/*, allow
    p, role:team-b, applications, sync, team-b-project/*, allow
    
    # Platform team can manage everything
    p, role:platform-admin, applications, *, */*, allow
    p, role:platform-admin, clusters, *, *, allow
    p, role:platform-admin, repositories, *, *, allow
    
    # Map SSO groups to roles
    g, team-a-developers, role:team-a
    g, team-b-developers, role:team-b
    g, platform-engineers, role:platform-admin
  
  policy.default: role:readonly
```

**ArgoCD Project (Namespace Isolation):**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a-project
  namespace: argocd
spec:
  description: "Team A applications"
  
  # Only allow deployments to team-a namespace
  destinations:
    - namespace: team-a
      server: https://kubernetes.default.svc
    - namespace: team-a-*
      server: https://kubernetes.default.svc
  
  # Only allow specific Git repos
  sourceRepos:
    - https://github.com/company/gitops-config.git
    - https://github.com/company/team-a-apps.git
  
  # Restrict what K8s resources teams can deploy
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  
  namespaceResourceWhitelist:
    - group: "apps"
      kind: Deployment
    - group: "apps"
      kind: StatefulSet
    - group: ""
      kind: Service
    - group: ""
      kind: ConfigMap
    - group: ""
      kind: Secret
    - group: "networking.k8s.io"
      kind: Ingress
    - group: "autoscaling"
      kind: HorizontalPodAutoscaler
  
  # DENY dangerous resources
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota  # Only platform team can set quotas
    - group: "rbac.authorization.k8s.io"
      kind: Role
    - group: "rbac.authorization.k8s.io"
      kind: RoleBinding
  
  # Sync windows (prevent deploys during maintenance)
  syncWindows:
    - kind: deny
      schedule: "0 22 * * 5"  # No deploys Friday 10 PM
      duration: 58h            # Until Monday 8 AM
      applications: ["*"]
      
    - kind: allow
      schedule: "0 9 * * 1-5"  # Allow deploys weekdays 9 AM - 10 PM
      duration: 13h
      applications: ["*"]
```

**Progressive Delivery with Argo Rollouts:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
  namespace: team-a
spec:
  replicas: 10
  
  strategy:
    canary:
      canaryService: payment-api-canary
      stableService: payment-api-stable
      
      # Traffic management via Istio/ALB
      trafficRouting:
        istio:
          virtualServices:
            - name: payment-api-vsvc
              routes:
                - primary
      
      steps:
        # Step 1: 5% traffic to canary
        - setWeight: 5
        - pause: { duration: 5m }
        
        # Step 2: Automated analysis
        - analysis:
            templates:
              - templateName: success-rate
              - templateName: latency-p99
            args:
              - name: service-name
                value: payment-api-canary
        
        # Step 3: 25% traffic
        - setWeight: 25
        - pause: { duration: 10m }
        
        # Step 4: Another analysis
        - analysis:
            templates:
              - templateName: success-rate
        
        # Step 5: 50% traffic
        - setWeight: 50
        - pause: { duration: 10m }
        
        # Step 6: Full rollout
        - setWeight: 100
      
      # Auto-rollback on failure
      abortScaleDownDelaySeconds: 30
      
---
# Analysis Template - Prometheus query
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}", code=~"2.."}[5m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

---

### Q7: Your Terraform plan shows 50+ resources will be destroyed and recreated due to a provider upgrade. How do you safely handle this in production without downtime?

**Answer:**

**Diagnosis:**

```bash
# Step 1: Understand WHY resources are being recreated
terraform plan -out=tfplan 2>&1 | grep "must be replaced"
# Common causes:
# - Provider changed resource schema
# - Computed attribute changed default
# - Resource renamed/moved in module

# Step 2: Use terraform show to see exact changes
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions | contains(["delete"]))'
```

**Strategy: Phased Migration**

```bash
# Option 1: terraform state mv (resource renamed but same infra)
# If the resource address changed but the actual cloud resource hasn't:
terraform state mv 'module.old_name.aws_instance.web' 'module.new_name.aws_instance.web'

# Option 2: Import existing resources into new state
terraform import 'module.vpc.aws_vpc.main' vpc-12345678

# Option 3: Use moved blocks (Terraform 1.1+)
```

```hcl
# moved.tf - Declarative state moves (preferred)
moved {
  from = module.legacy_vpc.aws_vpc.main
  to   = module.vpc_v2.aws_vpc.main
}

moved {
  from = aws_security_group.web_sg
  to   = module.security.aws_security_group.web
}

# After running terraform plan with moved blocks:
# Terraform shows "move" instead of "destroy + create"
```

**For True Schema Changes (provider forces replacement):**

```hcl
# Strategy: Create new resource before destroying old one

# Step 1: Use create_before_destroy lifecycle
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = "m5.xlarge"
  
  lifecycle {
    create_before_destroy = true  # New instance up before old goes down
  }
}

# Step 2: For resources that can't use create_before_destroy,
# use blue-green pattern:

# Phase 1: Deploy new resources alongside old (don't destroy yet)
resource "aws_lb_target_group" "app_new" {
  name     = "app-tg-v2"
  # ... new configuration
}

# Phase 2: Shift traffic to new resources
resource "aws_lb_listener_rule" "weighted" {
  # Gradually shift traffic from old to new target group
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_new.arn
    # Move from old: aws_lb_target_group.app_old.arn
  }
}

# Phase 3: After validation, remove old resources
# (separate PR/apply)
```

**Provider Upgrade Procedure:**

```yaml
# Runbook: Provider Upgrade in Production
steps:
  1_preparation:
    - Pin current provider version in lock file
    - Take state backup: aws s3 cp s3://state/prod/terraform.tfstate ./backup/
    - Document current resource count: terraform state list | wc -l
    - Run plan with new provider in staging first
  
  2_staging_validation:
    - Upgrade provider in staging
    - Run terraform plan - document all changes
    - Apply in staging, verify no issues
    - Run integration tests against staging
    - Document any resources that were force-replaced
  
  3_production_approach:
    - If < 5 resources force-replaced: Apply during maintenance window
    - If 5-20 resources force-replaced: Blue-green with traffic shift
    - If > 20 resources force-replaced: 
      - Split into multiple applies
      - Use -target for critical resources first
      - Consider upgrading provider in stages (minor versions)
  
  4_execution:
    - Notify stakeholders
    - Apply with -parallelism=5 (slower but safer)
    - Monitor CloudWatch during apply
    - Verify health checks after each batch
    - Keep old state backup for 7 days
  
  5_rollback:
    - If issues: restore state from backup
    - Revert provider version in lock file
    - Apply to restore old configuration
```

---

### Q8: Design an Internal Developer Platform (IDP) that allows 50 engineering teams to self-service deploy applications on AWS without deep infrastructure knowledge.

**Answer:**

**IDP Architecture:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                Internal Developer Platform (IDP)                      │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Developer Interface Layer                                      │  │
│  │                                                                  │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌────────────────────────┐ │  │
│  │  │ Backstage   │  │ CLI Tool    │  │ GitHub PR Interface    │ │  │
│  │  │ Portal      │  │ (platform   │  │ (template PR +        │ │  │
│  │  │ (Self-serve)│  │  deploy)    │  │  auto-merge)          │ │  │
│  │  └─────────────┘  └─────────────┘  └────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                │                                      │
│  ┌────────────────────────────▼───────────────────────────────────┐  │
│  │  Orchestration Layer                                            │  │
│  │                                                                  │  │
│  │  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐ │  │
│  │  │ Service      │  │ Environment      │  │ Pipeline         │ │  │
│  │  │ Catalog      │  │ Provisioner      │  │ Generator        │ │  │
│  │  │ (templates)  │  │ (Crossplane/TF)  │  │ (CI/CD factory)  │ │  │
│  │  └──────────────┘  └──────────────────┘  └──────────────────┘ │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                │                                      │
│  ┌────────────────────────────▼───────────────────────────────────┐  │
│  │  Platform Services Layer                                        │  │
│  │                                                                  │  │
│  │  ┌──────────┐ ┌────────┐ ┌─────────┐ ┌────────┐ ┌──────────┐│  │
│  │  │ EKS/ECS  │ │ RDS    │ │ Redis   │ │ S3     │ │ SQS/SNS  ││  │
│  │  │ Cluster  │ │ Aurora │ │ Cache   │ │ Bucket │ │ Queues   ││  │
│  │  └──────────┘ └────────┘ └─────────┘ └────────┘ └──────────┘│  │
│  │  ┌──────────┐ ┌────────┐ ┌─────────┐ ┌────────┐            │  │
│  │  │ Secrets  │ │ DNS    │ │ TLS     │ │ Logs   │            │  │
│  │  │ Manager  │ │ Route53│ │ Certs   │ │ Groups │            │  │
│  │  └──────────┘ └────────┘ └─────────┘ └────────┘            │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**Service Template (What developers fill out):**

```yaml
# service-manifest.yaml (developer creates this)
apiVersion: platform.company.io/v1
kind: Service
metadata:
  name: payment-processor
  team: payments
  tier: tier-1  # Determines SLO, DR strategy, on-call requirements
  
spec:
  # What to deploy
  runtime: container  # container | lambda | static-site
  language: python
  framework: fastapi
  
  # Where to deploy
  environments:
    - name: development
      account: dev
      regions: [us-east-1]
    - name: staging
      account: staging
      regions: [us-east-1]
    - name: production
      account: production
      regions: [us-east-1, us-west-2]  # Multi-region for tier-1
  
  # What it needs
  resources:
    database:
      type: aurora-postgresql
      size: small  # small | medium | large | xlarge
      multiAz: true
    cache:
      type: redis
      size: small
    queue:
      type: sqs
      dlq: true
    storage:
      type: s3
      encryption: kms
  
  # How it scales
  scaling:
    min: 2
    max: 20
    targetCpuPercent: 70
  
  # Networking
  networking:
    public: true  # Gets ALB + WAF
    internalDependencies:
      - service: user-service
      - service: notification-service
  
  # Observability (auto-configured based on tier)
  observability:
    metrics: true
    tracing: true
    customDashboard: true
    alerts:
      - type: availability
        threshold: 99.9%
      - type: latency-p99
        threshold: 500ms
```

**Platform Generates Everything:**

```python
# Platform controller processes service manifest and generates:

def provision_service(manifest):
    """Generate all infrastructure and pipelines from service manifest."""
    
    # 1. Generate Terraform for infrastructure
    generate_terraform(manifest)  # VPC, RDS, Redis, S3, IAM roles
    
    # 2. Generate Kubernetes manifests
    generate_k8s_manifests(manifest)  # Deployment, Service, HPA, NetworkPolicy
    
    # 3. Generate CI/CD pipeline
    generate_pipeline(manifest)  # GitHub Actions workflow
    
    # 4. Generate monitoring
    generate_dashboards(manifest)  # CloudWatch/Grafana dashboards
    generate_alerts(manifest)      # Alarms based on tier SLO
    
    # 5. Register in service catalog
    register_in_backstage(manifest)
    
    # 6. Configure DNS
    configure_dns(manifest)  # Route 53 records
    
    # 7. Set up secrets
    create_secrets_structure(manifest)  # Secrets Manager paths
    
    # 8. Configure network policies
    configure_network_policies(manifest)  # Security groups, NACLs

def generate_terraform(manifest):
    """Generate Terraform using pre-approved modules."""
    template = f"""
module "{manifest.name}_database" {{
  source  = "company/rds-aurora/aws"
  version = "~> 3.0"
  
  cluster_name     = "{manifest.name}-{manifest.environment}"
  engine           = "aurora-postgresql"
  instance_class   = "{size_to_instance_class(manifest.resources.database.size)}"
  multi_az         = {manifest.resources.database.multiAz}
  
  vpc_id           = data.terraform_remote_state.networking.outputs.vpc_id
  subnet_ids       = data.terraform_remote_state.networking.outputs.data_subnet_ids
  
  tags = {{
    Service     = "{manifest.name}"
    Team        = "{manifest.team}"
    Tier        = "{manifest.tier}"
    Environment = "{manifest.environment}"
  }}
}}
"""
    return template
```

**Developer Experience:**

```bash
# Developer workflow (simplified)

# 1. Create new service from template
$ platform service create \
  --name my-api \
  --template fastapi-service \
  --team payments

# Output:
# ✅ Repository created: github.com/company/my-api
# ✅ CI/CD pipeline generated
# ✅ Development environment provisioning...
# ✅ Service registered in Backstage
# 🔗 Dashboard: https://backstage.company.com/catalog/my-api
# 🔗 Pipeline: https://github.com/company/my-api/actions

# 2. Deploy
$ git push origin main
# Pipeline automatically: build → test → scan → deploy to dev

# 3. Promote to production
$ platform promote my-api --from staging --to production
# Triggers canary deployment with automatic rollback

# 4. Check status
$ platform status my-api
# Service: my-api
# Environments:
#   dev:        ✅ healthy (v1.2.3)
#   staging:    ✅ healthy (v1.2.3)
#   production: ✅ healthy (v1.2.2) → deploying v1.2.3 (canary 25%)
```

---

### Q9: Your container images are 2GB+, taking 5+ minutes to pull during scaling events. How do you optimize the container lifecycle for production ECS/EKS?

**Answer:**

**Multi-Stage Docker Build Optimization:**

```dockerfile
# BEFORE: 2.1GB image
FROM python:3.11
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]

# AFTER: 89MB image (96% reduction)
# Stage 1: Build dependencies
FROM python:3.11-slim AS builder
WORKDIR /build

# Install build dependencies separately (cached layer)
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2: Runtime image
FROM python:3.11-slim AS runtime
WORKDIR /app

# Copy only installed packages (no build tools)
COPY --from=builder /install /usr/local

# Copy only application code
COPY src/ ./src/
COPY config/ ./config/

# Non-root user
RUN useradd -r -s /bin/false appuser
USER appuser

# Health check
HEALTHCHECK --interval=10s --timeout=3s --start-period=30s \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**ECR Lifecycle and Scanning:**

```hcl
resource "aws_ecr_repository" "app" {
  name                 = "payment-api"
  image_tag_mutability = "IMMUTABLE"  # Prevent tag overwriting
  
  image_scanning_configuration {
    scan_on_push = true  # Scan for vulnerabilities on push
  }
  
  encryption_configuration {
    encryption_type = "KMS"
    kms_key         = aws_kms_key.ecr.arn
  }
}

# Lifecycle policy - keep images lean
resource "aws_ecr_lifecycle_policy" "app" {
  repository = aws_ecr_repository.app.name
  
  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Keep last 10 production images"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["prod-"]
          countType     = "imageCountMoreThan"
          countNumber   = 10
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 2
        description  = "Remove untagged images after 1 day"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 1
        }
        action = { type = "expire" }
      },
      {
        rulePriority = 3
        description  = "Remove dev images after 7 days"
        selection = {
          tagStatus     = "tagged"
          tagPrefixList = ["dev-", "feature-"]
          countType     = "sinceImagePushed"
          countUnit     = "days"
          countNumber   = 7
        }
        action = { type = "expire" }
      }
    ]
  })
}
```

**Image Pull Optimization for EKS:**

```yaml
# Use ECR pull-through cache for public images
# Instead of pulling from Docker Hub (rate limited + slow)
# aws ecr create-pull-through-cache-rule --ecr-repository-prefix ecr-public --upstream-registry-url public.ecr.aws

# Use SOCI (Seekable OCI) for lazy loading on EKS
# Images start in ~1 second instead of waiting for full pull
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
spec:
  template:
    spec:
      containers:
        - name: app
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/payment-api:prod-v1.2.3
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
      
      # Node selector for Bottlerocket (optimized for containers)
      nodeSelector:
        kubernetes.io/os: linux
        node.kubernetes.io/instance-type: m6g.xlarge
      
      # Topology spread for HA
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
```

**CI Pipeline for Image Security:**

```yaml
# .github/workflows/container-build.yml
jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Image
        run: docker build -t $ECR_REPO:${{ github.sha }} .
      
      - name: Trivy Vulnerability Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.ECR_REPO }}:${{ github.sha }}"
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'  # Fail pipeline on HIGH/CRITICAL
      
      - name: Grype SBOM Generation
        run: |
          grype $ECR_REPO:${{ github.sha }} -o json > sbom.json
          # Store SBOM for compliance
          aws s3 cp sbom.json s3://security-artifacts/sbom/${{ github.sha }}/
      
      - name: Sign Image (Cosign)
        run: |
          cosign sign --key awskms:///$KMS_KEY_ARN \
            $ECR_REPO:${{ github.sha }}
      
      - name: Push to ECR
        run: |
          docker push $ECR_REPO:${{ github.sha }}
          # Tag as production candidate
          docker tag $ECR_REPO:${{ github.sha }} $ECR_REPO:prod-v${{ env.VERSION }}
          docker push $ECR_REPO:prod-v${{ env.VERSION }}
```

---

### Q10: How do you implement infrastructure drift detection and auto-remediation across 100+ AWS accounts managed by Terraform?

**Answer:**

**Drift Detection Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                Drift Detection & Remediation                     │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Scheduled Detection (EventBridge → Lambda)               │   │
│  │  Runs every 4 hours across all accounts                   │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                              │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │  Drift Analyzer                                           │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌───────────┐  │   │
│  │  │ Terraform Plan │  │ AWS Config     │  │ CloudTrail│  │   │
│  │  │ (detect drift) │  │ (detect drift) │  │ (who did) │  │   │
│  │  └────────────────┘  └────────────────┘  └───────────┘  │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                              │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │  Decision Engine                                          │   │
│  │                                                           │   │
│  │  Critical drift (security) → Auto-remediate immediately   │   │
│  │  Standard drift → Create ticket + notify team             │   │
│  │  Cosmetic drift → Log only                                │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                              │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │  Remediation Actions                                      │   │
│  │  ┌────────────────┐  ┌─────────────┐  ┌──────────────┐  │   │
│  │  │ Auto-apply     │  │ Create PR   │  │ Alert + Log  │  │   │
│  │  │ (critical sec) │  │ (standard)  │  │ (cosmetic)   │  │   │
│  │  └────────────────┘  └─────────────┘  └──────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation:**

```python
# Lambda: Drift detection across accounts
import boto3
import json
import subprocess
import os

def detect_drift(event, context):
    """Run terraform plan across all state files to detect drift."""
    
    accounts = get_all_accounts()
    drift_results = []
    
    for account in accounts:
        # Assume role in each account
        credentials = assume_cross_account_role(account['id'])
        
        # List all Terraform states for this account
        states = list_terraform_states(account['id'])
        
        for state in states:
            drift = run_terraform_plan(state, credentials)
            if drift['has_changes']:
                drift_results.append({
                    'account_id': account['id'],
                    'state_key': state,
                    'changes': drift['changes'],
                    'severity': classify_drift_severity(drift['changes'])
                })
    
    # Process results
    for result in drift_results:
        if result['severity'] == 'critical':
            auto_remediate(result)
        elif result['severity'] == 'standard':
            create_drift_ticket(result)
        else:
            log_drift(result)
    
    return {'drifts_detected': len(drift_results)}

def classify_drift_severity(changes):
    """Classify drift based on security impact."""
    critical_patterns = [
        'security_group.*ingress.*0.0.0.0/0',
        's3_bucket_public_access',
        'iam_policy.*Allow.*\\*',
        'encryption.*false',
        'cloudtrail.*stopped'
    ]
    
    for change in changes:
        for pattern in critical_patterns:
            if re.search(pattern, json.dumps(change)):
                return 'critical'
    
    if any(c['type'] == 'delete' for c in changes):
        return 'standard'
    
    return 'cosmetic'

def auto_remediate(drift_result):
    """Auto-remediate critical security drift."""
    # Only for pre-approved remediations
    approved_remediations = {
        'security_group_open': revert_security_group,
        'public_s3_bucket': block_public_access,
        'encryption_disabled': enable_encryption,
    }
    
    drift_type = identify_drift_type(drift_result)
    if drift_type in approved_remediations:
        approved_remediations[drift_type](drift_result)
        notify_team(f"Auto-remediated: {drift_type} in {drift_result['account_id']}")
    else:
        # Unknown critical drift - page on-call
        page_oncall(drift_result)
```

**AWS Config for Continuous Compliance:**

```hcl
# Detect AND auto-remediate common drift
resource "aws_config_remediation_configuration" "s3_encryption" {
  config_rule_name = aws_config_config_rule.s3_encryption.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-EnableS3BucketEncryption"
  
  parameter {
    name         = "BucketName"
    resource_value = "RESOURCE_ID"
  }
  parameter {
    name         = "SSEAlgorithm"
    static_value = "aws:kms"
  }
  
  automatic                  = true
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60
}

resource "aws_config_remediation_configuration" "sg_open" {
  config_rule_name = aws_config_config_rule.restricted_ssh.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-DisablePublicAccessForSecurityGroup"
  
  parameter {
    name         = "GroupId"
    resource_value = "RESOURCE_ID"
  }
  
  automatic                  = true
  maximum_automatic_attempts = 5
  retry_attempt_seconds      = 30
}
```

**Drift Prevention (Proactive):**

```hcl
# CloudTrail + EventBridge → Alert on manual changes
resource "aws_cloudwatch_event_rule" "manual_changes" {
  name        = "detect-manual-infrastructure-changes"
  description = "Alert when resources are modified outside Terraform"
  
  event_pattern = jsonencode({
    source      = ["aws.ec2", "aws.rds", "aws.s3", "aws.iam"]
    detail-type = ["AWS API Call via CloudTrail"]
    detail = {
      eventSource = ["ec2.amazonaws.com", "rds.amazonaws.com", "s3.amazonaws.com"]
      eventName   = [
        "AuthorizeSecurityGroupIngress",
        "ModifyDBInstance",
        "PutBucketPolicy",
        "CreateRole",
        "AttachRolePolicy"
      ]
      # Exclude: Changes made by Terraform role
      userIdentity = {
        sessionContext = {
          sessionIssuer = {
            arn = [{ "anything-but" = "arn:aws:iam::*:role/TerraformRole" }]
          }
        }
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "manual_changes_alert" {
  rule      = aws_cloudwatch_event_rule.manual_changes.name
  target_id = "notify-platform-team"
  arn       = aws_sns_topic.platform_alerts.arn
  
  input_transformer {
    input_paths = {
      account = "$.detail.userIdentity.accountId"
      user    = "$.detail.userIdentity.principalId"
      action  = "$.detail.eventName"
      resource = "$.detail.requestParameters"
    }
    input_template = "\"⚠️ Manual infrastructure change detected!\\nAccount: <account>\\nUser: <user>\\nAction: <action>\\nResource: <resource>\\n\\nThis change will cause Terraform drift. Please update Terraform code.\""
  }
}
```
