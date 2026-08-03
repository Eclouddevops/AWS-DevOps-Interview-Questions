# Terraform — Deep-Dive Interview Q&A (Modules & Workspaces)

## Table of Contents
1. [Core Concepts & State](#core-concepts--state)
2. [Modules](#modules)
3. [Workspaces](#workspaces)
4. [State Management](#state-management)
5. [Advanced Patterns](#advanced-patterns)
6. [Tricky Scenarios](#tricky-scenarios)

---

## Core Concepts & State


**Q1: Explain the Terraform workflow and what happens during each phase.**

**A:**

```
terraform init → terraform plan → terraform apply → terraform destroy

init:
├── Downloads provider plugins
├── Initializes backend (state storage)
├── Downloads modules from registry/source
└── Creates .terraform/ directory and .terraform.lock.hcl

plan:
├── Reads current state from backend
├── Refreshes state (queries real infrastructure)
├── Compares desired config vs current state
├── Generates execution plan (create/update/destroy)
└── Shows diff without making changes

apply:
├── Executes the plan (or generates new one if no saved plan)
├── Creates/modifies/destroys resources in dependency order
├── Updates state file after each resource operation
└── Parallel operations where no dependencies exist

destroy:
├── Generates plan to remove ALL managed resources
├── Destroys in reverse dependency order
└── Clears state file
```

**Tricky**: `terraform apply` without a saved plan file will generate a NEW plan. Infrastructure could have changed between `plan` and `apply`. For production: `terraform plan -out=plan.tfplan && terraform apply plan.tfplan`

---

**Q2: What is Terraform state? Why is it necessary? What problems does it solve?**

**A:**

State file (`terraform.tfstate`) is a JSON mapping between your config and real-world resources.

**Why it's necessary:**
1. **Mapping**: Config `aws_instance.web` → real resource `i-0abc123def`
2. **Metadata**: Dependencies between resources (for destroy ordering)
3. **Performance**: Caches resource attributes (avoids querying every resource on every plan)
4. **Collaboration**: Tracks who manages what (prevents conflicts)

**What's in state:**
```json
{
  "resources": [{
    "type": "aws_instance",
    "name": "web",
    "instances": [{
      "attributes": {
        "id": "i-0abc123def",
        "ami": "ami-12345",
        "public_ip": "54.1.2.3"
      }
    }]
  }]
}
```

**Tricky**: State contains SENSITIVE data (DB passwords, private keys) in plaintext! Always use encrypted backend (S3 with SSE, Terraform Cloud). NEVER commit state to Git.

---

**Q3: Explain `terraform import` vs `terraform state mv` vs `terraform state rm`.**

**A:**

| Command | Purpose | Effect on Real Infrastructure |
|---------|---------|-------------------------------|
| `import` | Bring existing resource under Terraform management | None (resource already exists) |
| `state mv` | Rename/move resource in state | None |
| `state rm` | Remove resource from state (stop managing it) | None (resource stays, Terraform forgets it) |

```bash
# Import existing resource into state
terraform import aws_instance.web i-0abc123def
# NOTE: You must write the config block manually first!

# Rename a resource (refactoring)
terraform state mv aws_instance.old_name aws_instance.new_name

# Move resource to a module
terraform state mv aws_instance.web module.compute.aws_instance.web

# Stop managing without destroying
terraform state rm aws_instance.web
# Resource continues to exist in AWS, Terraform just forgets it
```

**Tricky**: `terraform import` in older versions doesn't generate config. You must write the HCL manually, then import. Terraform 1.5+ has `import` blocks for config generation:
```hcl
import {
  to = aws_instance.web
  id = "i-0abc123def"
}
```

---

## Modules

**Q4: Explain Terraform modules. What is the difference between root module and child modules?**

**A:**

- **Root module**: The directory where you run `terraform apply` (contains `main.tf`, etc.)
- **Child module**: Any module called by another module using `module` block

```
infrastructure/
├── main.tf          ← Root module
├── variables.tf
├── outputs.tf
├── modules/
│   ├── vpc/         ← Child module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ec2/         ← Child module
│   └── rds/         ← Child module
```

**Module sources:**
```hcl
# Local path
module "vpc" {
  source = "./modules/vpc"
}

# Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"
}

# GitHub
module "vpc" {
  source = "github.com/org/terraform-modules//vpc?ref=v1.0.0"
}

# S3 bucket
module "vpc" {
  source = "s3::https://bucket.s3.amazonaws.com/modules/vpc.zip"
}
```

**Tricky**: Module `source` cannot use variables or expressions! It must be a literal string. This is because modules are resolved during `init`, before variables are evaluated.

---

**Q5: How do you pass data between modules? Explain inputs, outputs, and data flow.**

**A:**

```hcl
# modules/vpc/variables.tf (input)
variable "cidr_block" {
  type        = string
  description = "VPC CIDR block"
}

# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
}

# modules/vpc/outputs.tf (output)
output "vpc_id" {
  value = aws_vpc.main.id
}
output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

# Root module - passing data between modules
module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}

module "ec2" {
  source     = "./modules/ec2"
  vpc_id     = module.vpc.vpc_id           # Output from vpc → input to ec2
  subnet_ids = module.vpc.private_subnet_ids
}
```

**Tricky**: You CANNOT reference resources across modules directly. You MUST use outputs. `module.vpc.aws_vpc.main.id` is INVALID. Use `module.vpc.vpc_id` (via output).

---

**Q6: What are module best practices? How do you version and publish modules?**

**A:**

**Structure best practices:**
```
modules/vpc/
├── main.tf          # Resources
├── variables.tf     # All input variables
├── outputs.tf       # All outputs
├── versions.tf      # Required providers and versions
├── README.md        # Documentation
├── examples/        # Usage examples
│   └── complete/
│       └── main.tf
└── tests/           # Terraform test files
    └── vpc_test.tftest.hcl
```

**Best practices:**
1. **Pin module versions**: Never use `main` branch in production
2. **Minimal inputs**: Sensible defaults, only require what's necessary
3. **Explicit outputs**: Expose everything consumers might need
4. **Validation**: Use variable validation blocks
5. **Composition over inheritance**: Small, focused modules composed together
6. **No hardcoded values**: Everything configurable via variables

```hcl
variable "instance_type" {
  type    = string
  default = "t3.medium"
  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3 instance types are allowed."
  }
}
```

**Publishing to registry:**
- Repository must be named `terraform-<PROVIDER>-<NAME>`
- Must have tagged releases (semantic versioning)
- Must include `main.tf`, `variables.tf`, `outputs.tf`
- GitHub repository linked to Terraform Registry

---

**Q7: Explain `for_each` vs `count` in modules. When does each cause problems?**

**A:**

```hcl
# count - index-based (positional)
module "server" {
  count  = 3
  source = "./modules/ec2"
  name   = "server-${count.index}"
}
# State: module.server[0], module.server[1], module.server[2]

# for_each - key-based (named)
module "server" {
  for_each = toset(["web", "api", "worker"])
  source   = "./modules/ec2"
  name     = each.key
}
# State: module.server["web"], module.server["api"], module.server["worker"]
```

**Why `for_each` is preferred:**

| Scenario | count | for_each |
|----------|-------|----------|
| Remove middle item | ALL subsequent items recreated (index shifts!) | Only that item removed |
| Add item to beginning | ALL items recreated | Only new item created |
| Reference specific | `module.server[0]` (fragile) | `module.server["web"]` (stable) |

**Tricky**: `for_each` CANNOT use values that aren't known until apply time! If the map depends on a resource attribute that doesn't exist yet, you get: "The for_each value depends on resource attributes that cannot be determined until apply."

**Fix**: Use `depends_on` or split into separate apply stages, or use `-target`.

---

## Workspaces

**Q8: Explain Terraform workspaces. What's the difference between CLI workspaces and Terraform Cloud workspaces?**

**A:**

**CLI Workspaces** (OSS Terraform):
- Multiple state files from same configuration
- State stored in `terraform.tfstate.d/<workspace>/`
- Switch with `terraform workspace select <name>`
- Share same backend, variables, and code
- Use case: Lightweight environment separation (dev/staging/prod)

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
terraform workspace select prod
terraform workspace list
# * default
#   dev
#   staging
#   prod
```

**Terraform Cloud Workspaces:**
- Full environment isolation (own state, variables, credentials, policies)
- VCS-connected (each workspace can track different branch)
- RBAC, Sentinel policies, cost estimation
- Run history, state versioning, locking

**Key differences:**

| Feature | CLI Workspace | TF Cloud Workspace |
|---------|--------------|-------------------|
| State isolation | Different state file, same backend | Fully independent |
| Variable isolation | NONE (must use workspace-aware logic) | Full per-workspace variables |
| Code | Same | Can be different (different branches/repos) |
| Access control | None | RBAC per workspace |
| Use case | Simple multi-env | Enterprise multi-team |

**Tricky**: CLI workspaces share the same `terraform.tfvars`. You must use `terraform.workspace` to differentiate:
```hcl
locals {
  env_config = {
    dev     = { instance_type = "t3.micro", count = 1 }
    staging = { instance_type = "t3.small", count = 2 }
    prod    = { instance_type = "t3.large", count = 3 }
  }
  config = local.env_config[terraform.workspace]
}
```

---

**Q9: When should you NOT use Terraform workspaces? What are the alternatives?**

**A:**

**Don't use workspaces when:**
1. Environments need different providers/credentials
2. Environments have significantly different infrastructure (not just sizes)
3. Team needs access control per environment
4. Blast radius concerns (one mistake affects all environments)
5. Different approval workflows per environment

**Alternatives:**

1. **Directory-based separation** (most common in production):
```
environments/
├── dev/
│   ├── main.tf
│   ├── terraform.tfvars
│   └── backend.tf  # Different state file
├── staging/
│   ├── main.tf
│   └── ...
└── prod/
    ├── main.tf
    └── ...
```

2. **Terragrunt** (DRY configuration):
```
live/
├── terragrunt.hcl  # Common config
├── dev/
│   └── terragrunt.hcl  # include + env-specific vars
├── staging/
└── prod/
```

3. **Terraform Cloud workspaces** (enterprise)

**Tricky**: If someone runs `terraform destroy` in the wrong workspace, they destroy that environment. There's no built-in protection. At least with separate directories, the `backend.tf` points to different state files, making accidental cross-environment destruction harder.

---

## State Management

**Q10: How do you configure remote state with locking? Explain the S3 + DynamoDB pattern.**

**A:**

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
    kms_key_id     = "arn:aws:kms:us-east-1:123456:key/abcdef"
  }
}
```

**How locking works:**
1. `terraform plan`/`apply` → Creates lock entry in DynamoDB
2. DynamoDB item: `LockID = "bucket/key"`, includes who locked, when, operation
3. If another user tries to run → Gets "Error acquiring the state lock"
4. On completion → Lock deleted from DynamoDB
5. Force unlock (if stuck): `terraform force-unlock <LOCK_ID>`

**DynamoDB table schema:**
```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Tricky**: The S3 backend itself doesn't support locking! DynamoDB provides it. Without the `dynamodb_table` parameter, two people can `apply` simultaneously and corrupt state!

---

**Q11: How do you share data between Terraform configurations (cross-state reference)?**

**A:**

```hcl
# Configuration A outputs VPC ID
# (in its own state file)
output "vpc_id" {
  value = aws_vpc.main.id
}

# Configuration B reads Configuration A's state
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the remote state data
resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.vpc.outputs.private_subnet_ids[0]
}
```

**Alternative (preferred): Use data sources instead of remote state:**
```hcl
# More decoupled - doesn't need access to other state
data "aws_vpc" "main" {
  tags = { Name = "production-vpc" }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = { Tier = "private" }
}
```

**Why data sources are preferred:**
- No coupling to other team's state backend
- No need for cross-account state access permissions
- Works even if the VPC was created manually or by another tool

---

**Q12: State file is corrupted/lost. What do you do?**

**A:**

**If using S3 backend with versioning:**
```bash
# List state versions
aws s3api list-object-versions --bucket terraform-state --prefix prod/terraform.tfstate

# Restore previous version
aws s3api get-object --bucket terraform-state \
  --key prod/terraform.tfstate \
  --version-id "previous-version-id" \
  terraform.tfstate.backup

# Replace current state
aws s3 cp terraform.tfstate.backup s3://terraform-state/prod/terraform.tfstate
```

**If state is truly lost:**
```bash
# Option 1: Import all resources back
terraform import aws_vpc.main vpc-123456
terraform import aws_subnet.private[0] subnet-789
# ... for every resource (painful but accurate)

# Option 2: Use terraformer to generate state from existing infra
terraformer import aws --resources=vpc,subnet --regions=us-east-1

# Option 3: Write config + import blocks (Terraform 1.5+)
import {
  to = aws_instance.web
  id = "i-0abc123"
}
```

**Prevention:**
- S3 versioning enabled (ALWAYS)
- DynamoDB locking (prevent concurrent writes)
- State backups before risky operations
- Terraform Cloud (automatic state versioning and recovery)

---

## Advanced Patterns

**Q13: Explain `depends_on`, implicit vs explicit dependencies. When is explicit needed?**

**A:**

```hcl
# IMPLICIT dependency (Terraform figures it out from references)
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id  # Terraform knows: subnet must exist first
}

# EXPLICIT dependency (for hidden/behavioral dependencies)
resource "aws_instance" "web" {
  ami           = "ami-123"
  subnet_id     = aws_subnet.main.id
  depends_on    = [aws_iam_role_policy.s3_access]
  # No reference to the policy, but instance NEEDS it to function
}
```

**When explicit `depends_on` is needed:**
1. Resource needs IAM permissions that are created separately
2. Resource depends on a VPC endpoint being available
3. Module-level dependencies (module B needs module A completed)
4. AWS resources that have eventual consistency issues

**Tricky pitfalls:**
- `depends_on` on a module makes ALL resources in that module dependencies
- `depends_on` prevents parallelism (everything waits)
- Using `depends_on` with data sources forces them to read during apply (not plan)
- Overuse of `depends_on` = slow applies

---

**Q14: Explain `lifecycle` blocks. What do `prevent_destroy`, `create_before_destroy`, and `ignore_changes` do?**

**A:**

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    # Don't destroy this resource (terraform destroy will error)
    prevent_destroy = true

    # Create replacement before destroying old one (zero downtime)
    create_before_destroy = true

    # Ignore changes to these attributes (external modifications allowed)
    ignore_changes = [tags, ami]

    # Replace resource when any of these change (force recreation)
    replace_triggered_by = [null_resource.trigger.id]
  }
}
```

**Use cases:**
- `prevent_destroy`: Databases, state buckets (prevent accidental deletion)
- `create_before_destroy`: Load-balanced instances (maintain availability during replacement)
- `ignore_changes`: Tags managed by external tools (AWS Config auto-tagging), AMI updated by ASG

**Tricky**: `create_before_destroy` can fail if there's a unique constraint (e.g., same name). Old resource exists while new one is being created. Use `name_prefix` instead of `name` for resources that need this lifecycle.

---

**Q15: How do you handle secrets in Terraform? What are the options?**

**A:**

| Method | Security Level | Complexity | State Exposure |
|--------|---------------|-----------|----------------|
| `terraform.tfvars` (plaintext) | Low | Low | Yes |
| Environment variables | Medium | Low | Yes |
| AWS Secrets Manager data source | High | Medium | Yes (in state!) |
| Vault provider | High | High | Configurable |
| SOPS + encrypted files | High | Medium | No (encrypted) |
| Terraform Cloud variables (sensitive) | High | Low | Marked sensitive |

**Key insight**: Even when using Secrets Manager, the secret VALUE appears in state!

```hcl
# Secret is in state after apply!
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db-password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # This password is NOW in terraform.tfstate in plaintext!
}
```

**Best practice for databases:**
```hcl
# Generate random password, store in Secrets Manager
resource "random_password" "db" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret_version" "db" {
  secret_id     = aws_secretsmanager_secret.db.id
  secret_string = random_password.db.result
}

# Mark as sensitive (hides from plan output, still in state)
output "db_password" {
  value     = random_password.db.result
  sensitive = true
}
```

---

**Q16: Explain Terraform `moved` blocks and refactoring without destroying resources.**

**A:**

```hcl
# Old config: resource at top level
# resource "aws_instance" "web" { ... }

# New config: moved into a module
module "compute" {
  source = "./modules/compute"
}

# Tell Terraform this is a RENAME, not delete+create
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

# Rename a resource
moved {
  from = aws_security_group.allow_http
  to   = aws_security_group.web_ingress
}

# Move from count to for_each
moved {
  from = aws_instance.server[0]
  to   = aws_instance.server["web"]
}
```

**When to use `moved`:**
- Refactoring resource names
- Moving resources into/out of modules
- Converting `count` to `for_each`
- Splitting one module into multiple

**Tricky**: `moved` blocks should be kept in config for at least one apply cycle (so all team members pick it up). Can be removed after everyone has applied. Terraform Cloud: keep until all workspaces have applied.

---

## Tricky Scenarios

**Q17: `terraform plan` shows no changes but `terraform apply` fails. What are possible causes?**

**A:**

1. **Stale state**: State says resource exists, but it was deleted outside Terraform
   - Fix: `terraform refresh` or `terraform apply -refresh-only`

2. **API rate limiting**: Plan succeeds (cached), apply hits API limits
   - Fix: Add retry logic or reduce parallelism: `terraform apply -parallelism=5`

3. **IAM eventual consistency**: Policy created in plan, not propagated by apply time
   - Fix: Add `depends_on` or `sleep` via `time_sleep` resource

4. **Provider bug**: Provider reports no diff but apply sends incompatible API call
   - Fix: Update provider, check GitHub issues

5. **Conditional resource**: `count = condition ? 1 : 0` — condition changed between plan and apply
   - Fix: Use saved plan file

6. **State lock stolen**: Another apply started between your plan and apply
   - Fix: Always use `terraform plan -out=plan.tfplan && terraform apply plan.tfplan`

---

**Q18: How do you handle Terraform in a CI/CD pipeline securely?**

**A:**

```yaml
# GitHub Actions example
name: Terraform
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: hashicorp/setup-terraform@v3

    - name: Terraform Init
      run: terraform init

    - name: Terraform Plan
      run: terraform plan -out=plan.tfplan
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

    - name: Upload Plan
      uses: actions/upload-artifact@v4
      with:
        name: plan
        path: plan.tfplan

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    environment: production  # Requires approval
    runs-on: ubuntu-latest
    steps:
    - name: Download Plan
      uses: actions/download-artifact@v4
    - name: Terraform Apply
      run: terraform apply plan.tfplan
```

**Security considerations:**
- Use OIDC for cloud credentials (no static keys)
- Plan on PR, apply on merge (with approval gate)
- Store plan artifact (apply exact plan that was reviewed)
- Use Sentinel/OPA for policy enforcement
- Lock state during pipeline runs
- Never expose plan output publicly (contains sensitive values)

---

**Q19: You need to rename an S3 bucket managed by Terraform. What happens and how do you handle it?**

**A:**

**Problem**: S3 buckets cannot be renamed. Changing the `bucket` argument forces destruction and recreation. This means:
- All objects in the bucket are DELETED
- Any references to the bucket name break
- DNS/CloudFront/policies all need updating

**Solutions:**

1. **Create new + migrate (safest):**
```hcl
# Create new bucket
resource "aws_s3_bucket" "new" {
  bucket = "new-bucket-name"
}

# Sync data: aws s3 sync s3://old-bucket s3://new-bucket

# Remove old from state (don't destroy)
# terraform state rm aws_s3_bucket.old

# Import new bucket if created outside Terraform
```

2. **Prevent accidental destruction:**
```hcl
resource "aws_s3_bucket" "data" {
  bucket = var.bucket_name
  lifecycle {
    prevent_destroy = true
  }
}
```

3. **If you MUST change the name (accept downtime):**
```bash
# Backup data
aws s3 sync s3://old-bucket ./backup/
# Let Terraform destroy and create
terraform apply
# Restore data
aws s3 sync ./backup/ s3://new-bucket/
```

---

**Q20: Explain `terraform taint` vs `terraform apply -replace`. Why was taint deprecated?**

**A:**

Both force resource recreation on next apply.

```bash
# Old way (deprecated in 1.5+)
terraform taint aws_instance.web
# Marks resource as "tainted" in state
# Next plan shows: destroy + create

# New way (recommended)
terraform apply -replace="aws_instance.web"
# Inline with apply command, no state modification needed
```

**Why taint was deprecated:**
1. Modifies state directly (dangerous in team environments)
2. State can drift between taint and apply
3. `-replace` is atomic (plan + destroy + create in one operation)
4. `taint` can be accidentally committed to state without applying
5. `-replace` works with plan files: `terraform plan -replace="aws_instance.web" -out=plan.tfplan`

**Tricky**: If you `taint` a resource and someone else runs `apply` before you, THEY destroy and recreate it (surprise!). `-replace` avoids this because it's specified at plan/apply time.



---

## Additional Scenario-Based Tricky Questions

---

**Q21: You ran `terraform apply` and it created half the resources, then failed on one resource. Now your state is partially applied. Some resources exist in AWS, some don't, and the state file is a mess. How do you recover?**

**A:**

**Understanding partial apply:**
```
Terraform applies resources in dependency order.
If resource #7 out of 15 fails:
- Resources 1-6: Created in AWS ✓, recorded in state ✓
- Resource 7: May or may not exist in AWS, may or may not be in state
- Resources 8-15: Not created, not in state

Your state IS consistent for resources 1-6.
The problem is resource 7 (potentially inconsistent).
```

**Recovery procedure:**
```bash
# Step 1: Check current state vs reality
terraform plan
# This will show you what Terraform THINKS vs what EXISTS
# It should show: resources 8-15 will be created, resource 7 depends on situation

# Step 2: Handle the failed resource (#7)
# Case A: Resource was created in AWS but NOT recorded in state
# → Import it
terraform import aws_rds_instance.main prod-database-instance

# Case B: Resource was NOT created in AWS but IS in state (rare)
# → Remove from state
terraform state rm aws_rds_instance.main

# Case C: Resource was partially created (e.g., security group without rules)
# → Let terraform apply fix it (it will detect drift and update)

# Step 3: Re-run apply (idempotent — won't recreate existing resources)
terraform apply
# Resources 1-6: "No changes"
# Resource 7: "Update in-place" or "Create" (depending on Case A/B/C)
# Resources 8-15: "Create"
```

**Prevention:**
```hcl
# Use -target for risky resources (apply in stages)
terraform apply -target=aws_vpc.main -target=aws_subnet.private
terraform apply -target=aws_rds_instance.main  # Apply DB separately
terraform apply  # Apply remaining

# Use create_before_destroy for zero-downtime
lifecycle {
  create_before_destroy = true
}

# Use precondition/postcondition (Terraform 1.2+)
resource "aws_instance" "web" {
  lifecycle {
    precondition {
      condition     = data.aws_ami.latest.id != ""
      error_message = "AMI not found — aborting before partial apply"
    }
  }
}
```

**Tricky**: Terraform's apply is NOT atomic. There's no transaction rollback. If resource 7 fails, resources 1-6 remain. This is why `terraform plan -out=plan.tfplan` followed by `terraform apply plan.tfplan` is critical — you validate the EXACT plan before applying. Also, some resources have long creation times (RDS: 10+ min, CloudFront: 15+ min). A timeout during creation means the resource may actually be created but Terraform lost connection. Always check AWS Console before assuming "not created."

---

**Q22: Two team members ran `terraform apply` simultaneously (DynamoDB locking wasn't configured). Now the state file shows a resource that was deleted by Team A but Team B's apply still references it. How do you untangle this?**

**A:**

**The double-apply disaster:**
```
Timeline:
T0: Both read same state (resource X exists)
T1: Team A deletes resource X, updates state (X removed)
T2: Team B's apply tries to modify resource X
T3: Team B gets error: "resource not found" 
T4: Team B's apply partially fails, state is corrupted
    State says X exists (Team B's stale copy) but it doesn't in AWS
```

**Recovery:**
```bash
# Step 1: STOP everyone from running Terraform
# Communicate to team: "State corruption — nobody touch Terraform"

# Step 2: Pull the latest state
terraform state pull > current_state.json
cat current_state.json | jq '.resources[] | .type + "." + .name'

# Step 3: Identify ghost resources (in state but not in AWS)
terraform plan
# Resources that show "will be destroyed" but you didn't intend to destroy
# OR resources that show error "not found" during refresh

# Step 4: Remove ghost resources from state
terraform state rm aws_instance.deleted_by_team_a
terraform state rm aws_security_group.deleted_by_team_a

# Step 5: Import any resources that exist in AWS but not in state
terraform import aws_instance.new_one i-0abc123def

# Step 6: Verify clean state
terraform plan
# Should show only INTENDED changes (no ghosts, no surprises)

# Step 7: NOW configure locking (prevent recurrence)
```

**Proper locking setup:**
```hcl
terraform {
  backend "s3" {
    bucket         = "company-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # THIS prevents concurrent access
  }
}

# DynamoDB table for locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Tricky**: Even WITH locking, if someone uses `terraform force-unlock` incorrectly (lock was legitimate, not stale), they can cause the same concurrent apply issue. Also, `terraform refresh` (implicit in plan/apply) can itself update state based on what it finds in AWS. If Team A deleted a resource and Team B runs `terraform plan`, the refresh step will notice the resource is gone and UPDATE the state to remove it — but only if Team B's code still references it, Terraform will then try to recreate it. Understanding this refresh behavior is key to debugging state issues.

---

**Q23: Your Terraform module is used by 20 teams. You need to make a breaking change (rename a variable). How do you do this without breaking all 20 teams simultaneously?**

**A:**

**The breaking change problem:**
```hcl
# Current module interface (used by 20 teams):
module "vpc" {
  source      = "git::https://github.com/company/tf-modules//vpc?ref=v2.0.0"
  vpc_cidr    = "10.0.0.0/16"  # ← This variable name needs to change
  subnet_bits = 8
}

# You want to rename: vpc_cidr → cidr_block (more consistent)
# If you just rename it → 20 teams' configs break on next terraform init
```

**Backward-compatible migration strategy:**
```hcl
# Step 1: Add new variable, keep old one (v2.1.0 — non-breaking)
variable "cidr_block" {
  description = "VPC CIDR block (preferred)"
  type        = string
  default     = null
}

variable "vpc_cidr" {
  description = "DEPRECATED: Use cidr_block instead"
  type        = string
  default     = null
}

locals {
  # Use new variable if set, fall back to old
  actual_cidr = coalesce(var.cidr_block, var.vpc_cidr)
}

# Validation: at least one must be set
variable "validate_cidr" {
  type    = string
  default = ""
  validation {
    condition     = var.cidr_block != null || var.vpc_cidr != null
    error_message = "Either cidr_block or vpc_cidr must be provided."
  }
}

resource "aws_vpc" "main" {
  cidr_block = local.actual_cidr
}

# Step 2: Notify teams (deprecation notice in CHANGELOG, Slack, email)
# "vpc_cidr is deprecated. Switch to cidr_block before v3.0.0"

# Step 3: Monitor adoption (check who still uses old variable)
# grep -r "vpc_cidr" across all team repos

# Step 4: Remove old variable in next MAJOR version (v3.0.0)
# Semantic versioning: breaking changes = major version bump
# Teams pin to "?ref=v2.1.0" — won't break until they upgrade
```

**Module versioning best practice:**
```hcl
# Teams should ALWAYS pin module versions:
module "vpc" {
  source = "git::https://github.com/company/tf-modules//vpc?ref=v2.1.0"
  # NOT: ?ref=main (breaking changes hit immediately)
  # NOT: no ref (uses default branch = unpredictable)
}

# Or use Terraform Registry with version constraints:
module "vpc" {
  source  = "company/vpc/aws"
  version = "~> 2.1"  # Allows 2.1.x but not 3.0.0
}
```

**Tricky**: `moved` blocks (Terraform 1.1+) can help with resource renames inside modules without forcing teams to import/state-rm:
```hcl
# In the module code:
moved {
  from = aws_vpc.this
  to   = aws_vpc.main
}
```
This tells Terraform "the resource was renamed, don't destroy and recreate it." But `moved` blocks only help with RESOURCE renames, not variable renames. For variable renames, the deprecation pattern above is the only safe approach.

---
