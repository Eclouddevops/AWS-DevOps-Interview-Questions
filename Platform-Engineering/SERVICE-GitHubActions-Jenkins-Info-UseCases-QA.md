# GitHub Actions & Jenkins — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | GitHub Actions | Jenkins |
|-----------|---------------|---------|
| **Type** | Cloud-native CI/CD | Self-hosted CI/CD |
| **Hosting** | GitHub-managed runners + self-hosted | 100% self-hosted |
| **Config** | YAML (.github/workflows/) | Groovy (Jenkinsfile) |
| **Pricing** | Free (public), $0.008/min (private) | Free OSS + infrastructure cost |
| **Marketplace** | 15,000+ actions | 1,800+ plugins |
| **Scaling** | Auto (managed runners) | Manual (add agents) |
| **Best for** | GitHub-hosted code, modern CI/CD | Enterprise, complex pipelines, on-prem |

---

## 🎯 Use Cases

### GitHub Actions
1. **CI for every PR** — Build, test, lint on every pull request
2. **CD to AWS** — Deploy containers to ECS/EKS via OIDC
3. **Release automation** — Tag → build → push to ECR → deploy
4. **Infrastructure PRs** — Terraform plan comment on PR, apply on merge
5. **Security scanning** — SAST/DAST/dependency scanning in pipeline
6. **Scheduled jobs** — Nightly tests, weekly dependency updates

### Jenkins
1. **Enterprise multi-branch** — Complex branching strategies at scale
2. **On-premises CI/CD** — Air-gapped environments, compliance
3. **Multi-SCM** — GitHub + GitLab + Bitbucket in one Jenkins
4. **Custom toolchains** — Proprietary build tools, legacy systems
5. **Long-running builds** — Hours-long ML training, hardware tests
6. **Shared libraries** — Organization-wide pipeline templates

---

## ❓ Interview Questions & Answers

### Q1: How does OIDC authentication work in GitHub Actions for AWS? Why is it better than stored secrets?

**Answer:**

```yaml
# Traditional (DANGEROUS):
# Store AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY as GitHub secrets
# Problems: Keys never expire, any workflow can access, if GitHub breached → keys exposed

# OIDC (SECURE):
permissions:
  id-token: write  # GitHub generates signed JWT

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/GitHubDeployRole
      aws-region: us-east-1
      # NO secrets stored anywhere!

# HOW IT WORKS:
# 1. GitHub generates JWT token (signed by GitHub, contains repo+branch+run info)
# 2. JWT sent to AWS STS (AssumeRoleWithWebIdentity)
# 3. AWS validates JWT signature against GitHub's OIDC provider
# 4. If trust policy matches (correct repo + branch) → temp credentials issued
# 5. Credentials expire in 15 min - 1 hour (auto-rotated per run)

# AWS Trust Policy (restricts which repos/branches can assume):
{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myapp:ref:refs/heads/main"
    }
  }
}
# ONLY myorg/myapp on main branch can assume this role!
# Fork PRs, feature branches, other repos → DENIED
```

### Q2: Your GitHub Actions workflow takes 40 minutes. How do you optimize to < 10 minutes?

**Answer:**

```yaml
# KEY OPTIMIZATIONS:
# 1. Cache dependencies (save 5-8 min)
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('package-lock.json') }}

# 2. Parallel jobs (cut time by 50-70%)
jobs:
  lint:    # 2 min
  test:    # 5 min (run in parallel with lint!)
    strategy:
      matrix:
        shard: [1, 2, 3, 4]  # Test sharding!
  build:
    needs: [lint, test]  # Only waits for both

# 3. Cancel in-progress (don't waste minutes)
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

# 4. Skip unchanged (paths filter)
on:
  push:
    paths: ['src/**']  # Only run when source changes
    paths-ignore: ['**.md', 'docs/**']

# 5. Docker layer caching
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max

# 6. Selective testing (only test what changed)
- run: npx jest --changedSince=origin/main
```

### Q3: How do you implement deployment approval gates in GitHub Actions?

**Answer:**

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # ← THIS enables approval gates!
    # GitHub pauses here until designated reviewers approve
    # Configured in: Settings → Environments → production → Required reviewers
    steps:
      - run: ./deploy.sh production

# Environment protection rules:
# ├── Required reviewers (1-6 people must approve)
# ├── Wait timer (delay N minutes before allowing)
# ├── Deployment branches (only main can deploy to prod)
# └── Secrets scoped to environment (prod secrets ≠ staging secrets)
```

### Q4: Compare Jenkins Declarative vs Scripted Pipeline. When use each?

**Answer:**

```groovy
// DECLARATIVE (structured, opinionated — RECOMMENDED):
pipeline {
    agent { kubernetes { yaml '...' } }
    stages {
        stage('Build') { steps { sh 'make build' } }
        stage('Test')  { steps { sh 'make test' } }
        stage('Deploy') {
            when { branch 'main' }
            input { message "Deploy to production?" }
            steps { sh 'make deploy' }
        }
    }
    post {
        failure { slackSend "Build failed!" }
    }
}

// SCRIPTED (full Groovy — maximum flexibility):
node('docker') {
    try {
        stage('Build') { sh 'make build' }
        if (env.BRANCH_NAME == 'main') {
            stage('Deploy') {
                input "Deploy?"
                sh 'make deploy'
            }
        }
    } catch (e) {
        slackSend "Failed: ${e}"
        throw e
    }
}

// USE DECLARATIVE: 95% of cases (simpler, safer, better UX)
// USE SCRIPTED: Complex conditional logic, dynamic stage generation
```

### Q5: How do you prevent credential leakage in Jenkins pipelines?

**Answer:**

```groovy
// PROTECTION LAYERS:

// 1. Ephemeral agents (no stale state between builds)
agent { kubernetes { yaml '...' } }  // Pod destroyed after build

// 2. Credentials binding (scoped to block, masked in logs)
withCredentials([usernamePassword(credentialsId: 'db', 
    usernameVariable: 'USER', passwordVariable: 'PASS')]) {
    sh 'deploy.sh'  // $USER and $PASS available only here
    // Auto-masked in console output (shows ****)
}

// 3. Folder-level credential isolation
// Team A credentials in /TeamA/ folder — invisible to Team B

// 4. NEVER allow arbitrary shell in Jenkinsfile from untrusted PRs
// Use: "Pipeline script from SCM" → only main branch Jenkinsfile

// 5. Network policies: Build pods can't reach internet (block exfiltration)
// 6. Audit: Log all credential access attempts
```

### Q6: How do you implement a multi-account AWS deployment pipeline in GitHub Actions?

**Answer:**

```yaml
name: Deploy Multi-Account
on:
  push:
    branches: [main]

jobs:
  deploy-dev:
    environment: development
    permissions: { id-token: write, contents: read }
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::DEV_ACCOUNT:role/DeployRole
      - run: ./deploy.sh dev

  deploy-staging:
    needs: deploy-dev
    environment: staging
    permissions: { id-token: write, contents: read }
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::STAGING_ACCOUNT:role/DeployRole
      - run: ./deploy.sh staging
      - run: ./run-integration-tests.sh

  deploy-production:
    needs: deploy-staging
    environment: production  # Manual approval required
    permissions: { id-token: write, contents: read }
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::PROD_ACCOUNT:role/DeployRole
      - run: ./deploy.sh production --strategy canary --percent 10
      - run: sleep 300 && ./validate-canary.sh
      - run: ./deploy.sh production --strategy canary --percent 100

# Each account has its own OIDC trust policy allowing only specific branch
```

### Q7: What are GitHub Actions reusable workflows? How do they enable platform engineering?

**Answer:**

```yaml
# SHARED WORKFLOW (in .github/workflows repo):
# company/shared-workflows/.github/workflows/standard-ci.yml
name: Standard CI
on:
  workflow_call:  # Makes it callable from other repos
    inputs:
      language: { required: true, type: string }
      service_name: { required: true, type: string }
    secrets:
      AWS_ROLE: { required: true }

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: ./build-${{ inputs.language }}.sh
      - run: ./test.sh
      - run: ./security-scan.sh
      - run: ./push-image.sh ${{ inputs.service_name }}

# CONSUMER (in application repo — 3 lines!):
# myapp/.github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  ci:
    uses: company/shared-workflows/.github/workflows/standard-ci.yml@v2
    with:
      language: python
      service_name: payment-api
    secrets:
      AWS_ROLE: ${{ secrets.AWS_DEPLOY_ROLE }}

# BENEFIT: 50 repos use SAME pipeline. Update once → all repos updated.
# Platform team owns the workflow. App teams just call it.
```

---

## 🏆 Key Takeaways

```
GitHub Actions:
├── OIDC > stored secrets (always!)
├── Concurrency control prevents deployment conflicts
├── Matrix builds + caching = fast feedback
├── Environments for approval gates
├── Reusable workflows for platform standardization
└── paths-ignore to skip unnecessary runs

Jenkins:
├── Ephemeral agents (Kubernetes plugin) = no stale state
├── Shared libraries = organization-wide pipeline standards
├── Folder-level credential isolation per team
├── Declarative pipeline for 95% of cases
├── Pipeline replay for debugging (re-run with changes)
└── Blue Ocean UI for better visualization
```
