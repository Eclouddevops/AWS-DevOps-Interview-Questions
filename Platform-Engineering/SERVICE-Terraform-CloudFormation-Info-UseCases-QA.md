# Terraform & CloudFormation — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | Terraform | CloudFormation |
|-----------|-----------|----------------|
| **Provider** | HashiCorp | AWS |
| **Type** | Multi-cloud IaC | AWS-native IaC |
| **Language** | HCL (HashiCorp Configuration Language) | JSON / YAML |
| **State** | External (S3, Terraform Cloud) | AWS-managed (no state file!) |
| **Pricing** | Free (OSS) / Paid (Cloud/Enterprise) | Free (AWS service) |
| **Multi-cloud** | ✅ (AWS, Azure, GCP, 3000+ providers) | ❌ (AWS only) |
| **Drift Detection** | `terraform plan` (manual/scheduled) | Built-in drift detection |
| **Launched** | 2014 (HashiCorp) | 2011 (AWS) |

---

## 🎯 Use Cases

### Terraform Use Cases
1. **Multi-cloud infrastructure** — Same tool for AWS + Azure + GCP
2. **Multi-account AWS** — Terragrunt + modules for 100+ accounts
3. **Platform Engineering** — Reusable modules as internal products
4. **GitOps for infrastructure** — PR-based changes, plan in CI
5. **Existing resource adoption** — Import unmanaged resources
6. **Complex dependencies** — Cross-service resources with explicit graph

### CloudFormation Use Cases
1. **AWS-native automation** — StackSets across Organization (100+ accounts)
2. **Service Catalog** — Self-service products for teams
3. **Control Tower customizations** — Landing Zone baseline
4. **CDK-generated templates** — TypeScript/Python → CloudFormation
5. **Nested stacks** — Modular large-scale deployments
6. **Custom Resources** — Lambda-backed custom provisioning

---

## ❓ Interview Questions & Answers

### Q1: When would you choose Terraform over CloudFormation (and vice versa)?

**Answer:**

```
CHOOSE TERRAFORM:                        CHOOSE CLOUDFORMATION:
├── Multi-cloud (AWS + Azure + GCP)      ├── AWS-only, want zero tooling
├── Team already knows HCL               ├── Need StackSets (Org-wide)
├── Need modules marketplace             ├── Need Service Catalog
├── Want explicit state management       ├── Want AWS-managed state (none!)
├── Need import existing resources       ├── Building with CDK (generates CFN)
├── Complex cross-provider deps          ├── Need CloudFormation Hooks
├── Want Terragrunt for DRY              ├── Control Tower integration
├── Need plan preview in PRs             ├── Need drift detection built-in
├── Open-source community                ├── AWS Support covers it
└── Most common in job market!           └── Tight AWS IAM integration

MOST ENTERPRISES: Use BOTH!
├── Terraform: Application infrastructure (VPCs, ECS, RDS, etc.)
├── CloudFormation: Organization baseline (StackSets, Control Tower)
└── CDK: When team prefers TypeScript/Python for IaC
```

### Q2: Explain Terraform state. What happens if it gets corrupted or deleted?

**Answer:**

```
STATE FILE = Terraform's memory of what it created

Contains:
├── Resource IDs (vpc-abc123, i-def456)
├── Resource attributes (IP addresses, ARNs)
├── Dependencies between resources
├── Provider configuration
└── Sensitive values (DB passwords!) — ENCRYPT!

IF STATE IS CORRUPTED:
├── Terraform doesn't know what exists in AWS
├── `terraform plan` shows "create all" (wants to recreate everything!)
├── Running apply would DUPLICATE everything (disaster!)
└── Recovery: Restore from S3 versioning (that's why versioning is critical!)

IF STATE IS DELETED:
├── Same as corrupted — Terraform thinks nothing exists
├── Recovery options:
│   1. Restore from S3 version (best)
│   2. `terraform import` each resource manually (painful)
│   3. Recreate state from scratch with import blocks (Terraform 1.5+)
└── Prevention: S3 versioning + DynamoDB locking + encryption ALWAYS
```

### Q3: How do you handle secrets in Terraform without exposing them in state?

**Answer:**

```python
# BAD: Secret in plaintext in .tf file (visible in state + git!)
resource "aws_db_instance" "main" {
  password = "SuperSecret123!"  # NEVER DO THIS!
}

# GOOD: Reference from Secrets Manager (secret stays in SM, not state)
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
# NOTE: Password IS still in state file! Encrypt state with KMS!

# BEST: Use random_password + store in Secrets Manager
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}

resource "aws_db_instance" "main" {
  password = random_password.db.result
}
# State has the password but is encrypted (S3 SSE-KMS)
# Applications read from Secrets Manager at runtime (not from Terraform)
```

### Q4: CloudFormation stack is stuck in UPDATE_ROLLBACK_FAILED. How do you recover?

**Answer:**

```bash
# 1. Find which resources failed:
aws cloudformation describe-stack-events --stack-name my-stack \
  --query "StackEvents[?ResourceStatus=='UPDATE_FAILED']"

# 2. Continue rollback, SKIPPING broken resources:
aws cloudformation continue-update-rollback \
  --stack-name my-stack \
  --resources-to-skip "BrokenResource1" "BrokenResource2"

# 3. After rollback completes → stack is in UPDATE_ROLLBACK_COMPLETE
# 4. Now you can update or delete the stack normally

# WHY IT HAPPENS:
# ├── Resource was manually deleted (CFN can't roll back to something gone)
# ├── IAM permissions removed during update
# ├── Dependency ordering issue
# └── Nested stack failure cascading to parent
```

### Q5: How do you implement Terraform modules at enterprise scale (200+ consumers)?

**Answer:**

```hcl
# Module versioning (pin consumers to specific versions):
module "vpc" {
  source  = "git::https://github.com/company/tf-modules.git//vpc?ref=v3.2.1"
  # OR: source = "app.terraform.io/company/vpc/aws" version = "~> 3.2"
}

# Enterprise module strategy:
# ├── Modules in separate repo (versioned with git tags)
# ├── Semantic versioning (v1.0.0 → v1.0.1 = safe, v2.0.0 = breaking)
# ├── Automated testing (Terratest in CI on every change)
# ├── Renovate/Dependabot auto-updates consumers
# ├── CHANGELOG.md documents every change
# └── Breaking changes require MIGRATION.md guide

# Tier system:
# ├── Resource modules: Thin wrappers (aws_vpc → opinionated defaults)
# ├── Component modules: Combine resources (VPC + subnets + routing)
# └── Platform modules: Full stack (VPC + ECS + RDS + monitoring)
```

### Q6: Compare Terraform workspaces vs Terragrunt for multi-environment management.

**Answer:**

```
TERRAFORM WORKSPACES:                    TERRAGRUNT:
├── Built-in feature                     ├── Wrapper tool (by Gruntwork)
├── Same code, different state           ├── DRY configuration (inherit)
├── Variable per workspace               ├── Separate directory per env
├── Simple (few envs, same resources)    ├── Complex (different per env)
├── Risk: Apply to wrong workspace       ├── Explicit (cd into env dir)
├── Limited: Can't have different         ├── Each env can have different
│   resources per workspace               │   resources/modules
└── Best for: Dev/staging/prod            └── Best for: Enterprise scale
    of IDENTICAL infrastructure                with multi-account/region

RECOMMENDATION:
├── < 5 environments, same infra → Workspaces
├── > 5 environments or different infra → Terragrunt (or directory structure)
└── Enterprise (100+ accounts) → Terragrunt + modules + CI/CD
```

### Q7: How do CloudFormation StackSets work? When would you use them?

**Answer:**

```
StackSets = Deploy ONE template to MANY accounts/regions simultaneously

Architecture:
┌─────────────────────┐
│ Management Account  │ ← StackSet defined here
│ (Administrator)     │
└─────────┬───────────┘
          │ Deploys to:
    ┌─────┼─────────────────────────────────┐
    ▼     ▼     ▼     ▼     ▼     ▼       ▼
 Acct1  Acct2  Acct3  ...  Acct100  Acct200
 us-e1  us-e1  us-e1       us-w2    eu-w1
 (Stack Instance in each target)

USE CASES:
├── Security baseline (Config rules, GuardDuty, CloudTrail) in ALL accounts
├── Networking baseline (VPC, DNS, VPC endpoints) in ALL accounts
├── IAM roles (deploy cross-account roles to all member accounts)
├── Compliance (deploy Config rules org-wide)
└── Any "deploy same thing everywhere" pattern

PERMISSION MODELS:
├── SELF_MANAGED: You create admin/execution roles
└── SERVICE_MANAGED: AWS Organizations integration (recommended!)
    └── Auto-deploys to new accounts when they join the OU!
```

### Q8: Your Terraform plan shows 50 resources will be destroyed. You only changed one line. What happened?

**Answer:**

```
COMMON CAUSES:

1. PROVIDER UPGRADE forced resource replacement:
   ├── New provider version changed a resource's schema
   ├── Attribute now "ForceNew" that wasn't before
   └── Fix: Pin provider version, upgrade carefully

2. count → for_each migration (index shift):
   ├── Resource address changed: aws_subnet.web[0] → aws_subnet.web["az1"]
   ├── Terraform sees: old one DELETE, new one CREATE
   └── Fix: Use `moved` blocks or `terraform state mv`

3. Module source/version change:
   ├── Module internals changed resource names
   ├── Terraform can't correlate old ↔ new
   └── Fix: Use `moved` blocks in module

4. Backend/workspace change:
   ├── State file changed (different backend, different workspace)
   ├── New state is empty → Terraform wants to create everything
   └── Fix: Verify backend config, check terraform.tfstate

5. Lifecycle rule missing:
   resource "aws_instance" "app" {
     ami = var.ami_id  # AMI changed → instance replacement!
     lifecycle {
       ignore_changes = [ami]  # This prevents destroy on AMI change
     }
   }

DEBUGGING:
├── terraform plan -target=specific_resource (isolate the change)
├── terraform state list (verify resources are in state)
├── terraform show (compare state vs config)
└── Check: Did someone run terraform with different backend/workspace?
```

### Q9: How do you implement policy-as-code with Terraform to prevent non-compliant infrastructure?

**Answer:**

```
MULTI-LAYER ENFORCEMENT:

Layer 1: terraform validate + tflint (syntax/best practices)
Layer 2: OPA/Conftest on terraform plan output (custom rules)
Layer 3: Checkov/tfsec on .tf files (security scanning)
Layer 4: Sentinel (Terraform Cloud — enterprise policy engine)
Layer 5: AWS Config rules (runtime compliance detection)

EXAMPLE OPA POLICY:
```

```rego
# policy/deny_public_s3.rego
package terraform.aws

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  not has_encryption(resource)
  msg := sprintf("S3 bucket '%s' must have encryption", [resource.address])
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group_rule"
  resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
  resource.change.after.from_port != 443
  msg := sprintf("SG '%s' allows 0.0.0.0/0 on non-443 port", [resource.address])
}
```

```yaml
# CI pipeline enforces policies:
- name: Terraform Plan
  run: terraform plan -out=tfplan && terraform show -json tfplan > plan.json

- name: OPA Policy Check
  run: |
    opa eval --data policy/ --input plan.json "data.terraform.aws.deny"
    # FAILS pipeline if any policy violation detected!
```

### Q10: Explain CloudFormation Hooks and when you'd use them over SCPs.

**Answer:**

```
CloudFormation Hooks = PROACTIVE controls that validate resources BEFORE creation

SCPs: "You can't call this API at all" (broad, all-or-nothing)
Hooks: "You can call the API, but only if the resource matches rules" (granular)

EXAMPLE:
├── SCP: "Deny ec2:RunInstances" → No one can create ANY EC2 instance!
├── Hook: "Allow ec2:RunInstances only if instance has:
│          - encryption enabled
│          - IMDSv2 required
│          - approved instance type
│          - required tags present"
└── Result: Users CAN create EC2, but ONLY compliant ones!

Hook Target Types:
├── AWS::EC2::Instance (validate before creation)
├── AWS::S3::Bucket (ensure encryption/versioning)
├── AWS::RDS::DBInstance (require encryption + VPC)
└── Any CloudFormation resource type!

USE HOOKS WHEN:
├── Need attribute-level validation (not just API-level blocking)
├── Want to allow the action but enforce HOW it's done
├── Using CloudFormation/CDK for deployments
└── Need guardrails that are more specific than SCPs

USE SCPs WHEN:
├── Want to block entire services/regions
├── Need broad organizational guardrails
├── Must work regardless of deployment tool (Terraform, console, CLI)
└── Simpler rule (deny/allow entire action)
```

---

## 🏆 Key Takeaways for Interviews

```
Terraform:
├── State is everything (protect it: S3 + DynamoDB + KMS)
├── Modules = reusability at scale (version and test them!)
├── Plan before apply ALWAYS (CI/CD enforces this)
├── for_each > count (stable addressing!)
├── Remote backend + locking = team collaboration
└── OIDC for CI/CD (no stored credentials)

CloudFormation:
├── No state file (AWS manages it — simpler!)
├── StackSets = Organization-wide deployment (killer feature)
├── CDK generates CloudFormation (best of both: code + CFN)
├── Nested stacks for modularity
├── Drift detection built-in (Terraform needs external tooling)
└── Hooks for proactive compliance (before resource creation)
```
