# Terraform — Complete Knowledge Guide

> **Purpose**: Master Terraform from fundamentals through enterprise-scale patterns. Understand how it works internally, best practices for production, and how to avoid common pitfalls.

---

## 1. What Terraform Actually Does (The Mental Model)

### The Core Loop

```
YOU write:              Terraform does:
┌──────────────┐       ┌─────────────────────────────────────────┐
│ .tf files    │       │ 1. INIT: Download providers & modules    │
│ (desired     │──────▶│ 2. PLAN: Compare desired vs actual state │
│  state)      │       │ 3. APPLY: Make API calls to reach desired│
└──────────────┘       │ 4. STATE: Record what was created        │
                       └─────────────────────────────────────────┘

The KEY insight:
├── Terraform is DECLARATIVE (you describe WHAT, not HOW)
├── You never say "create this EC2" — you say "this EC2 should exist"
├── Terraform figures out the actions needed (create/update/delete)
└── STATE file is the source of truth for "what currently exists"
```

### The Three Files You Must Understand

```
┌─────────────────────────────────────────────────────────────────┐
│  1. CONFIGURATION (.tf files) — What you WANT                    │
│     "I want a VPC with CIDR 10.0.0.0/16 and 3 subnets"         │
│                                                                   │
│  2. STATE (terraform.tfstate) — What EXISTS right now            │
│     "VPC vpc-abc123 exists with CIDR 10.0.0.0/16"              │
│     "Subnet subnet-111 exists in us-east-1a"                    │
│                                                                   │
│  3. REALITY (actual AWS resources) — What's REALLY there        │
│     The actual resources in your AWS account                     │
│                                                                   │
│  Terraform compares: Configuration ↔ State ↔ Reality            │
│                                                                   │
│  terraform plan:                                                  │
│  - Refreshes: State ↔ Reality (detects drift)                   │
│  - Compares: Configuration ↔ Refreshed State (finds diff)       │
│  - Shows: What changes are needed                                │
│                                                                   │
│  terraform apply:                                                 │
│  - Makes API calls to change Reality to match Configuration      │
│  - Updates State to reflect new Reality                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Terraform Internals

### How `terraform plan` Works Step by Step

```
Step 1: READ all .tf files in current directory
        → Build a dependency graph of all resources

Step 2: READ the state file (from backend: S3, local, etc.)
        → Know what resources Terraform previously created

Step 3: REFRESH state (call AWS APIs to check real status)
        → For each resource in state, verify it still exists
        → Update state with current attributes (e.g., IP addresses)

Step 4: COMPARE desired (config) vs current (refreshed state)
        → Resource in config but NOT in state → CREATE
        → Resource in state but NOT in config → DESTROY
        → Resource in BOTH but attributes differ → UPDATE or REPLACE

Step 5: Generate execution plan
        → Order operations based on dependency graph
        → Show user what will happen
```

### The Dependency Graph

```hcl
# Terraform automatically builds a DAG (Directed Acyclic Graph)

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "web" {
  vpc_id     = aws_vpc.main.id  # ← DEPENDS ON vpc
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "app" {
  subnet_id = aws_subnet.web.id  # ← DEPENDS ON subnet
  ami       = "ami-12345"
  instance_type = "t3.micro"
}

# Dependency Graph (Terraform figures this out automatically):
#   aws_vpc.main
#       │
#       ▼
#   aws_subnet.web
#       │
#       ▼
#   aws_instance.app
#
# Create order: VPC first → Subnet second → Instance third
# Destroy order: Instance first → Subnet second → VPC last (reverse!)
```

### State File: The Heart of Terraform

```json
// terraform.tfstate (simplified)
{
  "version": 4,
  "serial": 42,
  "resources": [
    {
      "type": "aws_vpc",
      "name": "main",
      "instances": [
        {
          "attributes": {
            "id": "vpc-abc123",
            "cidr_block": "10.0.0.0/16",
            "arn": "arn:aws:ec2:us-east-1:123456789:vpc/vpc-abc123",
            "tags": {"Name": "production-vpc"}
          }
        }
      ]
    }
  ]
}

// CRITICAL RULES about state:
// ├── State contains SECRETS (database passwords, access keys)
// ├── NEVER commit state to git
// ├── ALWAYS use remote backend with encryption
// ├── ALWAYS enable state locking (DynamoDB)
// ├── ALWAYS enable versioning on S3 bucket (recovery!)
// └── State is the SINGLE source of what Terraform manages
```

---

## 3. HCL Language Essentials

### Variables and Types

```hcl
# VARIABLE TYPES:
variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type"
}

variable "port_list" {
  type    = list(number)
  default = [80, 443, 8080]
}

variable "tags" {
  type = map(string)
  default = {
    Environment = "production"
    Team        = "platform"
  }
}

variable "vpc_config" {
  type = object({
    cidr_block    = string
    enable_dns    = bool
    subnet_cidrs  = list(string)
  })
}

# VALIDATION (catch errors early):
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "cidr" {
  type = string
  validation {
    condition     = can(cidrhost(var.cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

### Expressions and Functions

```hcl
# CONDITIONAL:
resource "aws_instance" "app" {
  instance_type = var.environment == "prod" ? "m5.xlarge" : "t3.micro"
}

# FOR EACH (create multiple resources from a map):
variable "subnets" {
  default = {
    "web-az1" = { cidr = "10.0.1.0/24", az = "us-east-1a" }
    "web-az2" = { cidr = "10.0.2.0/24", az = "us-east-1b" }
    "app-az1" = { cidr = "10.0.3.0/24", az = "us-east-1a" }
  }
}

resource "aws_subnet" "this" {
  for_each          = var.subnets
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  tags              = { Name = each.key }
}

# COUNT (create N identical resources):
resource "aws_instance" "worker" {
  count         = var.worker_count
  ami           = "ami-12345"
  instance_type = "t3.medium"
  tags          = { Name = "worker-${count.index}" }
}

# DYNAMIC BLOCKS (generate repeated nested blocks):
resource "aws_security_group" "app" {
  name = "app-sg"

  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
  }
}

# USEFUL FUNCTIONS:
locals {
  # String manipulation
  upper_env = upper(var.environment)              # "PROD"
  name      = "${var.project}-${var.environment}" # "myapp-prod"
  
  # Collection manipulation
  subnet_ids    = [for s in aws_subnet.this : s.id]
  private_cidrs = [for k, v in var.subnets : v.cidr if startswith(k, "app")]
  
  # Networking
  subnet_cidr = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
  
  # Encoding
  user_data = base64encode(templatefile("userdata.sh", { env = var.environment }))
  
  # Lookup
  instance_type = lookup(var.instance_types, var.environment, "t3.micro")
}
```

### Lifecycle Rules

```hcl
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    # Prevent accidental destruction
    prevent_destroy = true
    
    # Create replacement before destroying old (zero-downtime)
    create_before_destroy = true
    
    # Ignore changes made outside Terraform (e.g., auto-scaling)
    ignore_changes = [
      tags["LastModified"],
      instance_type,  # Don't revert if someone manually resized
    ]
    
    # Replace resource when these change (force recreation)
    replace_triggered_by = [
      aws_ami.app.id  # New AMI → recreate instance
    ]
  }
}
```

---

## 4. State Management (Production)

### Remote Backend Configuration

```hcl
# backend.tf — Store state in S3 with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "production/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:alias/terraform"
    dynamodb_table = "terraform-state-lock"
    
    # Recommended: Enable S3 versioning for state recovery
  }
}
```

### State Operations You Must Know

```bash
# LIST all resources in state
terraform state list
# Output:
# aws_vpc.main
# aws_subnet.web["az1"]
# aws_instance.app[0]

# SHOW details of one resource
terraform state show aws_vpc.main
# Shows all attributes Terraform knows about

# MOVE a resource (rename without destroy/recreate)
terraform state mv aws_instance.old_name aws_instance.new_name
# Use when: Refactoring code, moving into modules

# REMOVE from state (Terraform forgets about it, resource stays in AWS)
terraform state rm aws_instance.legacy
# Use when: Resource will be managed by another tool/team

# IMPORT existing resource INTO state
terraform import aws_instance.existing i-1234567890abcdef0
# Use when: Adopting existing resources into Terraform management

# PULL state to local file (for inspection)
terraform state pull > state_backup.json

# PUSH state (dangerous! Use only for recovery)
terraform state push fixed_state.json
```

### Moved Blocks (Terraform 1.1+)

```hcl
# Instead of manual state mv, declare moves in code:
# (This is tracked in version control and works for the whole team)

# Moved a resource into a module:
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

# Renamed a resource:
moved {
  from = aws_instance.web_server
  to   = aws_instance.application
}

# Moved from count to for_each:
moved {
  from = aws_subnet.private[0]
  to   = aws_subnet.private["us-east-1a"]
}
```

---

## 5. Modules: Reusable Infrastructure

### Module Structure

```
modules/
├── vpc/
│   ├── main.tf          # Resources
│   ├── variables.tf     # Inputs
│   ├── outputs.tf       # Outputs (what callers can reference)
│   ├── versions.tf      # Required providers and versions
│   ├── README.md        # Documentation (auto-generated by terraform-docs)
│   └── examples/
│       ├── basic/
│       │   └── main.tf  # Example usage
│       └── complete/
│           └── main.tf
```

### Writing a Good Module

```hcl
# modules/vpc/variables.tf — Module inputs
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "environment" {
  type        = string
  description = "Environment name (used for naming and tagging)"
}

variable "private_subnet_cidrs" {
  type        = list(string)
  description = "List of CIDR blocks for private subnets"
  default     = []
}

# modules/vpc/main.tf — Module resources
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_subnet" "private" {
  for_each = { for idx, cidr in var.private_subnet_cidrs : 
               "az${idx}" => {
                 cidr = cidr
                 az   = data.aws_availability_zones.available.names[idx]
               }
             }
  
  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  
  tags = { Name = "${var.environment}-private-${each.key}" }
}

# modules/vpc/outputs.tf — What callers can reference
output "vpc_id" {
  value       = aws_vpc.this.id
  description = "ID of the created VPC"
}

output "private_subnet_ids" {
  value       = [for s in aws_subnet.private : s.id]
  description = "List of private subnet IDs"
}
```

### Using Modules

```hcl
# Root module (calling the vpc module):
module "networking" {
  source = "./modules/vpc"
  # OR from git:
  # source = "git::https://github.com/company/tf-modules.git//vpc?ref=v2.1.0"
  # OR from registry:
  # source  = "company/vpc/aws"
  # version = "~> 2.1"
  
  vpc_cidr             = "10.0.0.0/16"
  environment          = "production"
  private_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

# Reference module outputs:
resource "aws_instance" "app" {
  subnet_id = module.networking.private_subnet_ids[0]
  # ...
}
```

### Module Composition Pattern

```hcl
# Compose small modules into larger ones:
module "platform" {
  source = "./modules/platform"
  # This module internally calls:
  # - module "vpc" (networking)
  # - module "ecs_cluster" (compute)
  # - module "rds" (database)
  # - module "monitoring" (observability)
  
  environment = "production"
  team        = "payments"
}

# Output: Everything a team needs, from one module call
# module.platform.cluster_id
# module.platform.database_endpoint
# module.platform.vpc_id
```

---

## 6. Providers and Data Sources

### Provider Configuration

```hcl
# versions.tf — Pin provider versions (CRITICAL for stability)
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"  # Allows 5.30.x but not 5.31.0
    }
  }
}

# provider.tf — Configure the provider
provider "aws" {
  region = "us-east-1"
  
  # Default tags applied to ALL resources
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "terraform"
      Team        = var.team
    }
  }
  
  # Assume a role for cross-account management
  assume_role {
    role_arn     = "arn:aws:iam::TARGET_ACCOUNT:role/TerraformRole"
    session_name = "terraform-${var.environment}"
  }
}

# Multiple providers (multi-region, multi-account):
provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
  assume_role {
    role_arn = "arn:aws:iam::EU_ACCOUNT:role/TerraformRole"
  }
}

resource "aws_s3_bucket" "dr_backup" {
  provider = aws.us_west  # Use the us-west-2 provider
  bucket   = "dr-backup-bucket"
}
```

### Data Sources (Read Existing Resources)

```hcl
# Data sources READ existing resources (don't create anything)

# Look up the latest Amazon Linux AMI:
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Look up existing VPC by tags:
data "aws_vpc" "existing" {
  tags = { Name = "production-vpc" }
}

# Look up current account info:
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Use data source values:
resource "aws_instance" "app" {
  ami       = data.aws_ami.amazon_linux.id
  subnet_id = data.aws_vpc.existing.id
  
  tags = {
    Account = data.aws_caller_identity.current.account_id
    Region  = data.aws_region.current.name
  }
}
```

---

## 7. Terraform Workflow (Team Collaboration)

### The Git Workflow

```
Feature Branch Workflow:

1. Developer creates branch: git checkout -b feature/add-redis
2. Makes Terraform changes
3. Runs locally: terraform plan (validates changes)
4. Pushes branch, creates Pull Request
5. CI runs: terraform plan (shows changes in PR comment)
6. Team reviews the PLAN output (not just the code!)
7. PR approved and merged to main
8. CI/CD runs: terraform apply (on main branch ONLY)

RULES:
├── NEVER run terraform apply locally in production
├── Only CI/CD applies to production (auditable, repeatable)
├── terraform plan on every PR (shows impact before merge)
├── Lock file (.terraform.lock.hcl) committed to git
└── State file NEVER in git (always remote backend)
```

### CI/CD Pipeline for Terraform

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  pull_request:
    paths: ['infrastructure/**']
  push:
    branches: [main]
    paths: ['infrastructure/**']

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0
      
      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/
      
      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        working-directory: infrastructure/
      
      - name: Comment Plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const plan = `${{ steps.plan.outputs.stdout }}`
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Terraform Plan\n\`\`\`\n${plan}\n\`\`\``
            })

  apply:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    concurrency:
      group: terraform-apply
      cancel-in-progress: false  # NEVER cancel a running apply!
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      
      - run: terraform init
        working-directory: infrastructure/
      
      - run: terraform apply -auto-approve
        working-directory: infrastructure/
```

---

## 8. Common Patterns and Anti-Patterns

### ✅ Best Practices

```yaml
code_organization:
  - One state per environment per layer (not monolithic)
  - Pin provider and module versions
  - Use variables for anything that changes between environments
  - Use locals for computed values and expressions
  - Name resources descriptively (aws_instance.web_server not aws_instance.a)
  
state_management:
  - Always use remote backend with locking
  - Enable versioning on state bucket (disaster recovery)
  - Encrypt state at rest (contains secrets!)
  - Use workspaces OR separate state files (not both!)
  
security:
  - Never hardcode secrets in .tf files
  - Use data sources to reference secrets from Secrets Manager/Parameter Store
  - Mark sensitive outputs: sensitive = true
  - Use OIDC for CI/CD (no stored credentials)
  
team_practices:
  - Run terraform fmt before committing
  - Use terraform validate in pre-commit hooks
  - Require plan review in PR before merge
  - Only CI/CD runs apply (never local applies to prod)
```

### ❌ Anti-Patterns

```yaml
dont_do_this:
  - Store state in git (exposes secrets, causes conflicts)
  - Run apply without plan review
  - Use count when for_each is better (count index shifts!)
  - Hardcode account IDs, regions, or AMI IDs
  - Create one massive state file (slow, risky, conflicts)
  - Skip provider version pinning (breaks on upgrade)
  - Use terraform destroy in production without extreme care
  - Modify state file manually (use state mv/rm/import)
  - Use local-exec for things that should be resources
  - Forget lifecycle prevent_destroy on databases
```

### Count vs For_Each (Common Mistake)

```hcl
# BAD: Using count (fragile — indexes shift when you remove items)
variable "subnets" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "bad" {
  count      = length(var.subnets)
  cidr_block = var.subnets[count.index]
}
# If you remove the FIRST subnet, ALL remaining subnets shift index!
# Terraform wants to destroy and recreate everything. DANGEROUS!

# GOOD: Using for_each (stable — each resource has a key)
variable "subnets" {
  default = {
    "az1" = "10.0.1.0/24"
    "az2" = "10.0.2.0/24"
    "az3" = "10.0.3.0/24"
  }
}

resource "aws_subnet" "good" {
  for_each   = var.subnets
  cidr_block = each.value
  tags       = { Name = "subnet-${each.key}" }
}
# Address: aws_subnet.good["az1"], aws_subnet.good["az2"]
# Remove "az2"? Only az2 is destroyed. Others are untouched!

# RULE: Use count ONLY for identical resources (replicas)
#       Use for_each for DISTINCT resources (different configs)
```

---

## 9. Advanced Patterns

### Terragrunt (DRY Multi-Environment)

```
Problem: Same infrastructure in dev, staging, prod
         → Copy-paste .tf files? NO!
         
Solution: Terragrunt wraps Terraform with DRY configurations

Directory structure:
infrastructure/
├── modules/              # Shared Terraform modules
│   ├── vpc/
│   ├── ecs/
│   └── rds/
├── environments/
│   ├── terragrunt.hcl   # Root config (backend, provider)
│   ├── dev/
│   │   ├── terragrunt.hcl  # Include root + set variables
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl  # Source = modules/vpc
│   │   └── ecs/
│   │       └── terragrunt.hcl
│   ├── staging/
│   │   ├── terragrunt.hcl
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl
│   │   └── ecs/
│   │       └── terragrunt.hcl
│   └── prod/
│       └── ... (same structure)
```

```hcl
# environments/prod/vpc/terragrunt.hcl
terraform {
  source = "../../../modules//vpc"
}

include "root" {
  path = find_in_parent_folders()
}

inputs = {
  vpc_cidr    = "10.0.0.0/16"
  environment = "production"
  # Different from dev (which uses "10.100.0.0/16")
}
```

### Import and Adopt Existing Resources

```hcl
# Terraform 1.5+ : Import blocks (declarative imports!)
import {
  to = aws_instance.existing_server
  id = "i-0123456789abcdef0"
}

resource "aws_instance" "existing_server" {
  ami           = "ami-12345"
  instance_type = "t3.large"
  subnet_id     = "subnet-abc123"
  # Must match the actual resource's configuration
}

# Run: terraform plan
# Shows: "aws_instance.existing_server will be imported"
# Run: terraform apply
# Result: Resource is now in state, managed by Terraform

# BEFORE Terraform 1.5 (CLI import):
# terraform import aws_instance.existing_server i-0123456789abcdef0
```

### Workspaces vs Directory Structure

```
WORKSPACES (built-in, simple):
├── Same code, different state per workspace
├── terraform workspace new staging
├── terraform workspace select production
├── Access workspace name: terraform.workspace
├── Good for: Simple projects, same config different params
└── Bad for: Different resources per environment

DIRECTORY STRUCTURE (recommended for production):
├── Separate directories per environment
├── Each has own backend config (different state file)
├── Can have different resources per environment
├── Explicit and visible (no hidden workspace magic)
└── Good for: Teams, complex environments, different configs
```

---

## 10. Terraform at Enterprise Scale

### State Decomposition Strategy

```
Small project (< 50 resources): 1 state file
Medium project (50-200): 3-5 states (by layer)
Large enterprise (200+): 10+ states (by layer + team + account)

Layers (change frequency):
├── Foundation (yearly): Organizations, SCPs, IAM baseline
├── Networking (monthly): VPCs, TGW, DNS, VPN
├── Security (monthly): KMS keys, Config rules, GuardDuty
├── Data (weekly): RDS, ElastiCache, S3 buckets
├── Compute (weekly): ECS clusters, ALBs, Auto Scaling
└── Applications (daily): ECS services, Lambda, API Gateway

Cross-state references:
├── terraform_remote_state (read another state's outputs)
├── data sources (query AWS directly)
└── SSM Parameters (write outputs to Parameter Store, read elsewhere)
```

### Drift Detection and Remediation

```bash
# Detect drift (run nightly in CI):
terraform plan -detailed-exitcode
# Exit code 0 = no changes (no drift)
# Exit code 1 = error
# Exit code 2 = changes detected (DRIFT!)

# Common drift causes:
# ├── Manual console changes (educate team!)
# ├── Auto-scaling modifying desired_count
# ├── AWS service updates (security group rules added by ELB)
# └── Another tool managing same resources

# Fix: For expected drift, use ignore_changes
resource "aws_ecs_service" "app" {
  desired_count = 3
  lifecycle {
    ignore_changes = [desired_count]  # Auto-scaling manages this
  }
}
```

---

## 11. Troubleshooting Common Issues

```
ERROR: "state is locked"
├── Another terraform operation is running
├── Fix: Wait for it to finish, OR force-unlock:
└── terraform force-unlock LOCK_ID (dangerous! Verify no other apply running)

ERROR: "resource already exists"
├── Resource was created outside Terraform
├── Fix: terraform import <address> <id>
└── Then add matching resource block to .tf code

ERROR: "cycle detected"
├── Two resources depend on each other (circular)
├── Fix: Break the cycle with depends_on or restructure
└── Common: Security group A references B, B references A

ERROR: "provider produced inconsistent result"
├── Provider bug or API returned unexpected data
├── Fix: Run apply again (usually transient)
└── If persistent: pin to older provider version

ERROR: "Error acquiring state lock"
├── Previous operation crashed without releasing lock
├── Fix: terraform force-unlock <LOCK-ID>
└── Prevention: Never kill terraform apply mid-run!
```

---

## 12. Learning Path

```
Beginner (Week 1-2):
├── Install Terraform, create first resource (S3 bucket)
├── Understand: terraform init, plan, apply, destroy
├── Learn variables, outputs, locals
├── Understand state (what it is, why it matters)
└── Create VPC + subnet + EC2 instance

Intermediate (Week 3-4):
├── Remote backend (S3 + DynamoDB)
├── Modules (write and use)
├── for_each, dynamic blocks, conditionals
├── Data sources (reference existing resources)
├── Import existing resources
├── Workspaces vs directory structure
└── CI/CD pipeline for Terraform

Advanced (Week 5-8):
├── State decomposition (multi-state architecture)
├── Terragrunt for DRY multi-environment
├── Policy-as-code (OPA, Sentinel, Checkov)
├── Custom providers and provisioners
├── Module versioning and publishing
├── Cross-account/cross-region patterns
├── Testing (Terratest, terraform test)
└── Drift detection automation

Expert (Month 3+):
├── Enterprise Terraform at scale (500+ resources)
├── Custom provider development
├── Terraform Cloud/Enterprise features
├── Module factory pattern (generate modules from specs)
├── Self-service platform with Terraform
├── Migration from CloudFormation/CDK to Terraform
└── Performance optimization (parallelism, targets)
```

---

## 13. Quick Reference Card

```hcl
# ESSENTIAL COMMANDS:
terraform init          # Download providers, configure backend
terraform plan          # Preview changes (dry run)
terraform apply         # Execute changes
terraform destroy       # Delete all resources
terraform fmt           # Format code
terraform validate      # Syntax check
terraform output        # Show output values
terraform state list    # List resources in state
terraform import        # Import existing resource
terraform taint         # Mark resource for recreation (deprecated)
terraform refresh       # Update state from reality (rarely needed)

# ESSENTIAL FLAGS:
terraform plan -out=tfplan          # Save plan to file
terraform apply tfplan              # Apply saved plan
terraform plan -target=module.vpc   # Plan only VPC module
terraform apply -auto-approve       # Skip interactive approval (CI/CD)
terraform plan -var="env=prod"      # Override variable
terraform plan -var-file=prod.tfvars # Use variable file
terraform init -upgrade             # Upgrade providers to latest
terraform state mv old.name new.name # Rename resource in state
terraform plan -refresh=false       # Skip refresh (faster, less safe)

# ESSENTIAL FILES:
main.tf           # Primary resources
variables.tf      # Input variables
outputs.tf        # Output values
versions.tf       # Required providers and versions
terraform.tfvars  # Variable values (per environment)
backend.tf        # State backend configuration
.terraform.lock.hcl  # Provider lock file (COMMIT THIS!)
```
