# CI/CD & GitOps — Complete Knowledge Guide

> **Purpose**: Understand continuous integration, continuous delivery, and GitOps from fundamentals through enterprise patterns. Learn how modern deployment pipelines work, when to use each strategy, and how to build reliable delivery systems.

---

## 1. CI/CD Fundamentals (The Big Picture)

### What CI/CD Actually Means

```
┌─────────────────────────────────────────────────────────────────────┐
│  The Delivery Pipeline                                               │
│                                                                       │
│  Continuous Integration (CI):                                        │
│  "Every code change is automatically built and tested"               │
│  ├── Developer pushes code                                           │
│  ├── Automated build runs                                            │
│  ├── Unit tests execute                                              │
│  ├── Code quality checks                                             │
│  └── Result: "This code compiles and tests pass" ✓                  │
│                                                                       │
│  Continuous Delivery (CD):                                           │
│  "Code is ALWAYS ready to deploy (but human approves)"              │
│  ├── CI passes                                                       │
│  ├── Deploy to staging automatically                                 │
│  ├── Integration tests pass                                          │
│  ├── Human clicks "Deploy to Production"                             │
│  └── Result: "We CAN deploy anytime" ✓                              │
│                                                                       │
│  Continuous Deployment (full CD):                                    │
│  "Every change that passes tests goes to production automatically"  │
│  ├── CI passes                                                       │
│  ├── Deploy to staging → tests pass                                  │
│  ├── Deploy to production AUTOMATICALLY (no human)                   │
│  ├── Monitoring validates (auto-rollback if errors)                  │
│  └── Result: "Push to main = live in production in minutes" ✓       │
│                                                                       │
│  Most teams: CI + Continuous Delivery (manual prod approval)         │
│  Top teams: CI + Continuous Deployment (fully automated)             │
└─────────────────────────────────────────────────────────────────────┘
```

### The Pipeline Stages

```
┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
│ SOURCE │──▶│ BUILD  │──▶│ TEST   │──▶│ STAGE  │──▶│ PROD   │
└────────┘   └────────┘   └────────┘   └────────┘   └────────┘
     │            │            │            │            │
  git push     compile      unit test    deploy to    deploy to
  webhook      package      lint         staging      production
               docker       SAST/DAST    e2e tests    canary/bg
               build        coverage     smoke test   monitoring

Time:  0       2 min       5 min        10 min       15 min
       ──────────────── Total: ~15 minutes ──────────────────
```

---

## 2. GitHub Actions (Deep Understanding)

### How GitHub Actions Works

```
┌─────────────────────────────────────────────────────────────────┐
│  GitHub Actions Architecture                                     │
│                                                                   │
│  WORKFLOW (.github/workflows/*.yml)                               │
│  └── Triggered by: push, PR, schedule, manual, webhook          │
│                                                                   │
│  JOB (runs on a RUNNER — a fresh VM every time)                 │
│  ├── Each job gets a CLEAN environment (no leftover state!)     │
│  ├── Jobs run in PARALLEL by default                            │
│  ├── Use "needs:" to create dependencies (sequential)           │
│  └── Each job can use a different runner (ubuntu, windows, mac) │
│                                                                   │
│  STEP (individual commands within a job)                         │
│  ├── Runs sequentially within the job                           │
│  ├── Can be: shell command (run:) OR action (uses:)            │
│  ├── Shares filesystem with other steps in same job             │
│  └── Shares environment variables within the job                │
│                                                                   │
│  ACTION (reusable unit of automation)                            │
│  ├── Pre-built: actions/checkout, actions/setup-node, etc.      │
│  ├── Custom: Your own composite or Docker actions               │
│  └── Marketplace: Thousands of community actions                │
└─────────────────────────────────────────────────────────────────┘
```

### Workflow Syntax Deep Dive

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# TRIGGERS: When does this workflow run?
on:
  push:
    branches: [main, 'release/**']  # Only these branches
    paths:
      - 'src/**'                     # Only when source changes
      - 'package.json'               # Or dependencies change
    paths-ignore:
      - '**.md'                      # Never for docs changes
  
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
  
  schedule:
    - cron: '0 6 * * 1'             # Every Monday at 6 AM UTC
  
  workflow_dispatch:                  # Manual trigger button
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

# ENVIRONMENT VARIABLES: Available to all jobs
env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

# CONCURRENCY: Prevent parallel runs
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true  # Cancel older runs on same branch

# JOBS: Units of work
jobs:
  # Job 1: Quick feedback (runs first)
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'  # Cache node_modules between runs
      
      - run: npm ci  # Clean install (deterministic)
      - run: npm run lint
      - run: npm run typecheck

  # Job 2: Tests (parallel with lint)
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    strategy:
      matrix:
        shard: [1, 2, 3]  # Run tests in 3 parallel shards
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --shard=${{ matrix.shard }}/3

  # Job 3: Build (only after lint + test pass)
  build:
    needs: [lint, test]  # Waits for both to succeed
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/myapp
          tags: |
            type=sha,prefix=
            type=semver,pattern={{version}}
      
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Job 4: Deploy (only on main, requires approval)
  deploy:
    needs: [build]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval
    steps:
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service myapp \
            --force-new-deployment
```

### Key Concepts to Master

```yaml
# SECRETS (never in code, encrypted at rest)
steps:
  - env:
      API_KEY: ${{ secrets.API_KEY }}  # From repo/org settings
    run: ./deploy.sh

# OUTPUTS (pass data between jobs)
jobs:
  build:
    outputs:
      version: ${{ steps.ver.outputs.version }}
    steps:
      - id: ver
        run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT
  
  deploy:
    needs: build
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.version }}"

# ENVIRONMENTS (approval gates + deployment tracking)
jobs:
  deploy-prod:
    environment:
      name: production
      url: https://app.company.com
    # GitHub shows: "Waiting for approval"
    # Designated reviewers must approve before job runs

# ARTIFACTS (share files between jobs)
jobs:
  build:
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
  
  deploy:
    needs: build
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist

# CACHING (speed up repeated workflows)
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: npm-${{ hashFiles('package-lock.json') }}
      restore-keys: npm-  # Partial match fallback

# MATRIX (run same job with different configurations)
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20, 22]
      fail-fast: false  # Don't cancel other matrix jobs on failure
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

---

## 3. Jenkins (Enterprise CI/CD)

### Jenkins Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Jenkins Architecture                                            │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  CONTROLLER (Master)                                        │ │
│  │  ├── Stores job configurations                             │ │
│  │  ├── Schedules builds                                      │ │
│  │  ├── Serves the web UI                                     │ │
│  │  ├── Manages plugins                                       │ │
│  │  └── Distributes work to agents                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│       │                                                          │
│       │ Dispatches jobs                                          │
│       ▼                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Agent 1  │  │ Agent 2  │  │ Agent 3  │  │ Agent N  │      │
│  │ (EC2)    │  │ (K8s Pod)│  │ (Docker) │  │ (ECS)    │      │
│  │          │  │          │  │          │  │          │      │
│  │ Executes │  │ Executes │  │ Executes │  │ Executes │      │
│  │ builds   │  │ builds   │  │ builds   │  │ builds   │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│                                                                   │
│  Best Practice: Controller does NO builds (only orchestration)   │
│  Best Practice: Agents are EPHEMERAL (created per-build, destroyed after) │
└─────────────────────────────────────────────────────────────────┘
```

### Declarative Jenkinsfile

```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    // WHERE to run
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: builder
      image: node:20-alpine
      command: ['sleep', 'infinity']
    - name: docker
      image: docker:24-dind
      securityContext:
        privileged: true
'''
        }
    }
    
    // CONFIGURATION
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }
    
    environment {
        REGISTRY = 'ecr.us-east-1.amazonaws.com'
        IMAGE = "${REGISTRY}/myapp"
    }
    
    // STAGES (sequential)
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install') {
            steps {
                container('builder') {
                    sh 'npm ci'
                }
            }
        }
        
        stage('Test') {
            parallel {  // Run in parallel within this stage
                stage('Unit Tests') {
                    steps {
                        container('builder') {
                            sh 'npm test -- --coverage'
                        }
                    }
                }
                stage('Lint') {
                    steps {
                        container('builder') {
                            sh 'npm run lint'
                        }
                    }
                }
            }
        }
        
        stage('Build Image') {
            steps {
                container('docker') {
                    sh "docker build -t ${IMAGE}:${GIT_COMMIT} ."
                    sh "docker push ${IMAGE}:${GIT_COMMIT}"
                }
            }
        }
        
        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                sh "./deploy.sh staging ${GIT_COMMIT}"
            }
        }
        
        stage('Deploy to Production') {
            when { branch 'main' }
            input {
                message "Deploy to production?"
                ok "Yes, deploy!"
            }
            steps {
                sh "./deploy.sh production ${GIT_COMMIT}"
            }
        }
    }
    
    // ALWAYS runs (cleanup, notifications)
    post {
        success {
            slackSend(color: 'good', message: "✅ ${env.JOB_NAME} - Success")
        }
        failure {
            slackSend(color: 'danger', message: "❌ ${env.JOB_NAME} - Failed")
        }
        always {
            cleanWs()  // Clean workspace
            junit '**/test-results/*.xml'  // Publish test results
        }
    }
}
```

### GitHub Actions vs Jenkins

```
┌──────────────────────────────────────────────────────────────────┐
│  Feature           │ GitHub Actions      │ Jenkins              │
├────────────────────┼─────────────────────┼──────────────────────┤
│ Hosting            │ GitHub-managed      │ Self-hosted          │
│ Config format      │ YAML               │ Groovy (Jenkinsfile) │
│ Runners            │ Managed + self-host │ Must provide agents  │
│ Marketplace        │ 15,000+ actions    │ 1,800+ plugins       │
│ Secrets            │ Built-in           │ Credentials plugin   │
│ Matrix builds      │ Native             │ Plugin-based         │
│ Container support  │ services: blocks   │ Kubernetes plugin    │
│ Cost               │ Free (public repos)│ Infrastructure cost  │
│ Scale              │ Auto               │ Must manage agents   │
│ Approval gates     │ Environments       │ input {} step        │
│ Best for           │ GitHub-hosted code │ Enterprise, complex  │
│                    │ Simple-medium      │ On-prem, compliance  │
└────────────────────┴─────────────────────┴──────────────────────┘

Choose GitHub Actions when:
├── Code is on GitHub
├── Team wants simplicity
├── Standard web/API applications
└── Don't want to manage infrastructure

Choose Jenkins when:
├── Enterprise with complex requirements
├── Need on-premises CI/CD (compliance)
├── Have existing Jenkins investment
├── Need extreme customization
└── Multi-SCM (GitHub + GitLab + Bitbucket)
```

---

## 4. GitOps (The Modern Deployment Model)

### What GitOps Means

```
Traditional CI/CD (Push-based):
  Developer → CI builds → CD PUSHES to cluster
  ├── Pipeline has cluster credentials
  ├── Pipeline decides WHEN and WHAT to deploy
  └── If pipeline breaks, deployments stop

GitOps (Pull-based):
  Developer → Commits desired state to Git → Operator PULLS from Git
  ├── Git is the SINGLE source of truth
  ├── Operator (ArgoCD/Flux) runs IN the cluster
  ├── Continuously reconciles: cluster state == git state
  └── If operator restarts, it re-syncs from Git (self-healing)

The 4 Principles of GitOps:
1. DECLARATIVE: System described declaratively (YAML manifests)
2. VERSIONED: Desired state stored in Git (history, audit, rollback)
3. AUTOMATED: Changes auto-applied by operators (no manual kubectl)
4. SELF-HEALING: Agents detect drift and correct it continuously
```

### GitOps Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  GitOps Flow                                                         │
│                                                                       │
│  ┌──────────────┐      ┌──────────────────┐                        │
│  │ Application  │      │  CI Pipeline      │                        │
│  │ Repo         │─────▶│  (Build + Test)   │                        │
│  │ (source code)│      │                   │                        │
│  └──────────────┘      └────────┬──────────┘                        │
│                                  │ Pushes new image tag              │
│                                  ▼                                    │
│  ┌──────────────────────────────────────────────┐                   │
│  │  GitOps Config Repo (desired state)          │                   │
│  │                                               │                   │
│  │  manifests/                                   │                   │
│  │  ├── production/                              │                   │
│  │  │   ├── deployment.yaml (image: v1.2.3)    │                   │
│  │  │   ├── service.yaml                        │                   │
│  │  │   └── hpa.yaml                            │                   │
│  │  └── staging/                                 │                   │
│  │      └── deployment.yaml (image: v1.2.4)    │                   │
│  └─────────────────────┬────────────────────────┘                   │
│                         │                                            │
│                         │ ArgoCD/Flux watches this repo              │
│                         ▼                                            │
│  ┌──────────────────────────────────────────────┐                   │
│  │  Kubernetes Cluster                           │                   │
│  │                                               │                   │
│  │  ┌──────────────────────────────────────┐    │                   │
│  │  │  ArgoCD (GitOps Operator)            │    │                   │
│  │  │                                       │    │                   │
│  │  │  Continuously:                        │    │                   │
│  │  │  1. Polls Git repo for changes       │    │                   │
│  │  │  2. Compares: Git state vs Cluster   │    │                   │
│  │  │  3. If different → Apply changes     │    │                   │
│  │  │  4. If someone does kubectl edit →   │    │                   │
│  │  │     Reverts to Git state (self-heal) │    │                   │
│  │  └──────────────────────────────────────┘    │                   │
│  │                                               │                   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │                   │
│  │  │ App v1.2 │  │ App v1.2 │  │ App v1.2 │  │                   │
│  │  │ (pod)    │  │ (pod)    │  │ (pod)    │  │                   │
│  │  └──────────┘  └──────────┘  └──────────┘  │                   │
│  └──────────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

### ArgoCD Core Concepts

```yaml
# ArgoCD Application (what to deploy and where)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
spec:
  # WHERE the desired state lives (Git)
  source:
    repoURL: https://github.com/company/gitops-config.git
    path: manifests/production/payment-service
    targetRevision: main
  
  # WHERE to deploy (Kubernetes cluster + namespace)
  destination:
    server: https://kubernetes.default.svc
    namespace: payment
  
  # HOW to sync
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Revert manual changes
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

# ArgoCD ApplicationSet (deploy same app to multiple environments)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: payment-service-set
spec:
  generators:
    - list:
        elements:
          - cluster: staging
            namespace: payment-staging
          - cluster: production
            namespace: payment-prod
  template:
    metadata:
      name: 'payment-{{cluster}}'
    spec:
      source:
        repoURL: https://github.com/company/gitops-config.git
        path: 'manifests/{{cluster}}/payment-service'
      destination:
        server: '{{cluster}}'
        namespace: '{{namespace}}'
```

### GitOps vs Traditional CI/CD

```
┌──────────────────────────────────────────────────────────────────┐
│  Aspect              │ Traditional CI/CD    │ GitOps             │
├──────────────────────┼──────────────────────┼────────────────────┤
│ Source of truth      │ Pipeline config      │ Git repository     │
│ Deploy mechanism     │ Push (pipeline→k8s)  │ Pull (agent→git)   │
│ Cluster credentials  │ In pipeline (risky!) │ In cluster only    │
│ Drift detection      │ None                 │ Continuous         │
│ Rollback             │ Re-run old pipeline  │ git revert         │
│ Audit trail          │ Pipeline logs        │ Git history        │
│ Self-healing         │ No                   │ Yes (auto-revert)  │
│ Multi-cluster        │ Complex              │ Natural (per-repo) │
│ Approval            │ Pipeline gates       │ PR reviews         │
│ Disaster recovery    │ Rebuild pipeline     │ Re-point to Git    │
└──────────────────────┴──────────────────────┴────────────────────┘
```

---

## 5. Deployment Strategies (When to Use Each)

### Strategy Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│  Strategy      │ Downtime │ Risk    │ Rollback │ Cost    │ Use  │
├────────────────┼──────────┼─────────┼──────────┼─────────┼──────┤
│ Recreate       │ YES      │ High    │ Slow     │ 1x      │ Dev  │
│ Rolling        │ No       │ Medium  │ Slow     │ 1-1.5x  │ Most │
│ Blue-Green     │ No       │ Low     │ Instant  │ 2x      │ Crit │
│ Canary         │ No       │ Lowest  │ Fast     │ 1.1x    │ Crit │
│ Shadow/Mirror  │ No       │ None    │ N/A      │ 2x      │ Test │
└────────────────┴──────────┴─────────┴──────────┴─────────┴──────┘
```

### Rolling Update (Default for Most Services)

```
State 1: All running v1
[v1] [v1] [v1] [v1]

State 2: Replace one at a time
[v2] [v1] [v1] [v1]  ← v2 starts, v1 stops

State 3: Continue rolling
[v2] [v2] [v1] [v1]

State 4: Complete
[v2] [v2] [v2] [v2]  ← All v2, zero downtime

Kubernetes:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # At most 1 extra pod during update
      maxUnavailable: 0  # Never go below desired count

Pros: Simple, no extra infrastructure, zero downtime
Cons: Both versions run simultaneously (must be compatible!)
      Rollback is slow (roll forward to v1 again)
```

### Blue-Green Deployment

```
                    Load Balancer
                         │
            ┌────────────┼────────────┐
            │            │            │
         ┌──▼──┐     ┌──▼──┐     ┌──▼──┐
BLUE:    │ v1  │     │ v1  │     │ v1  │  ← Currently serving traffic
         └─────┘     └─────┘     └─────┘
         
         ┌─────┐     ┌─────┐     ┌─────┐
GREEN:   │ v2  │     │ v2  │     │ v2  │  ← New version (idle, tested)
         └─────┘     └─────┘     └─────┘

CUTOVER: Switch load balancer from Blue → Green (instant!)
ROLLBACK: Switch back from Green → Blue (instant!)

Implementation (AWS):
├── Two ECS services (or two target groups)
├── ALB listener rule switches between target groups
├── CodeDeploy manages the traffic shift
└── Old version kept running for 1 hour (rollback window)

Pros: Instant rollback, full testing of new version before live
Cons: Double the resources during deployment (expensive)
      Database schema must be compatible with both versions
```

### Canary Deployment

```
Step 1: Deploy v2 to small subset (canary)
┌────────────────────────────────────────────────┐
│  Traffic split: 95% v1, 5% v2                  │
│                                                  │
│  [v1][v1][v1][v1][v1][v1][v1][v1][v1]  [v2]   │
│  ────────── 95% ──────────────────    ── 5% ── │
└────────────────────────────────────────────────┘

Step 2: Monitor canary metrics (errors, latency)
├── Error rate same as v1? → Continue
├── Latency acceptable? → Continue
└── Any regression? → ABORT (shift 100% back to v1)

Step 3: Gradually increase (10% → 25% → 50% → 100%)
┌────────────────────────────────────────────────┐
│  [v1][v1][v1][v1][v1]  [v2][v2][v2][v2][v2]   │
│  ────── 50% ─────────  ────── 50% ───────────  │
└────────────────────────────────────────────────┘

Step 4: Complete (100% v2)
┌────────────────────────────────────────────────┐
│  [v2][v2][v2][v2][v2][v2][v2][v2][v2][v2]     │
│  ──────────────── 100% ────────────────────    │
└────────────────────────────────────────────────┘

Implementation:
├── AWS: CodeDeploy ECSCanary10Percent5Minutes
├── Kubernetes: Argo Rollouts with analysis
├── Manual: ALB weighted target groups
└── Automated: CloudWatch alarms trigger rollback

Pros: Lowest risk (only small % affected if v2 is bad)
      Can validate with REAL traffic
Cons: More complex (need traffic splitting + metric analysis)
      Both versions must be live simultaneously
```

---

## 6. Pipeline Security

### The Supply Chain Security Problem

```
Attack Surface in CI/CD:

Source Code:
├── Malicious dependency (supply chain attack)
├── Compromised developer account
└── Unauthorized code in PR

Build Process:
├── Compromised build environment
├── Malicious CI configuration
├── Secret leakage in logs
└── Build tool vulnerability

Artifacts:
├── Tampered container image
├── Unsigned artifacts
├── Image tag override (mutable tags)
└── Vulnerable base images

Deployment:
├── Stolen deployment credentials
├── Unauthorized deployment
├── Configuration injection
└── Missing approval gates
```

### Security Controls at Each Stage

```yaml
source_security:
  - Branch protection (require reviews, no force push)
  - Signed commits (GPG)
  - CODEOWNERS for sensitive files
  - Dependency scanning (Dependabot/Renovate)
  - Secret scanning (pre-commit hooks)

build_security:
  - Ephemeral build environments (no state between builds)
  - OIDC authentication (no stored credentials)
  - Minimal permissions (least privilege for CI role)
  - Hermetic builds (reproducible, deterministic)
  - SBOM generation (track all dependencies)

artifact_security:
  - Immutable image tags (never :latest in production!)
  - Image signing (cosign + KMS)
  - Vulnerability scanning (Trivy, Grype)
  - ECR lifecycle policies (remove old images)
  - Private registry (no public pulls in prod)

deployment_security:
  - Environment approval gates
  - Automated rollback on failure
  - Deployment windows (no deploys Friday night!)
  - Audit trail (who deployed what when)
  - Network isolation (CI can't access everything)
```

### OIDC Authentication (No Stored Secrets!)

```yaml
# The OLD way (dangerous):
# Store AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY as GitHub secrets
# These never expire, can be exfiltrated, not scoped to branch

# The NEW way (OIDC — zero stored credentials):
permissions:
  id-token: write  # GitHub generates a JWT token

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/GitHubDeployRole
      aws-region: us-east-1
      # No credentials stored! GitHub proves identity to AWS via JWT
      # AWS gives back TEMPORARY credentials (15 min - 1 hour)
      # Credentials are scoped to this specific workflow run

# HOW IT WORKS:
# 1. GitHub Actions gets a JWT (signed by GitHub)
# 2. JWT contains: repo name, branch, workflow, run ID
# 3. AWS IAM validates the JWT against GitHub's OIDC provider
# 4. If trust policy matches (repo + branch) → issue temp credentials
# 5. Credentials expire after the job completes

# WHY IT'S BETTER:
# - Nothing to steal (no long-lived keys)
# - Scoped to specific repo + branch (can't use from fork)
# - Auditable (CloudTrail shows GitHub run ID)
# - Auto-rotates (new credentials every run)
```

---

## 7. AWS Native CI/CD Services

### AWS CodePipeline Ecosystem

```
┌─────────────────────────────────────────────────────────────────┐
│  AWS CI/CD Services                                              │
│                                                                   │
│  CodeCommit ──▶ CodeBuild ──▶ CodeDeploy ──▶ CodePipeline       │
│  (Git repo)     (Build/Test)  (Deploy)       (Orchestrator)     │
│                                                                   │
│  CodeCommit: AWS-hosted Git (like GitHub but within AWS)         │
│  ├── Encrypted at rest                                           │
│  ├── IAM-based access control                                    │
│  ├── Integrates with CodePipeline                               │
│  └── Mostly superseded by GitHub/GitLab                         │
│                                                                   │
│  CodeBuild: Managed build service                                │
│  ├── Serverless (no agents to manage)                           │
│  ├── Pay per build minute                                        │
│  ├── Docker-based build environments                            │
│  ├── buildspec.yml defines build steps                          │
│  └── Can run in VPC (access private resources)                  │
│                                                                   │
│  CodeDeploy: Deployment automation                               │
│  ├── EC2, ECS, Lambda deployment strategies                     │
│  ├── Blue-green, canary, rolling                                │
│  ├── Automatic rollback on alarm                                │
│  └── AppSpec file defines deployment steps                      │
│                                                                   │
│  CodePipeline: Orchestrates the full pipeline                    │
│  ├── Visual pipeline builder                                    │
│  ├── Stages with actions (source, build, deploy)               │
│  ├── Manual approval gates                                      │
│  ├── Cross-account deployment                                   │
│  └── EventBridge integration for triggers                       │
└─────────────────────────────────────────────────────────────────┘
```

### CodeBuild buildspec.yml

```yaml
# buildspec.yml (in repo root)
version: 0.2

env:
  variables:
    APP_NAME: "payment-api"
  secrets-manager:
    DOCKER_USER: "prod/dockerhub:username"
    DOCKER_PASS: "prod/dockerhub:password"

phases:
  install:
    runtime-versions:
      nodejs: 20
    commands:
      - npm ci
  
  pre_build:
    commands:
      - echo "Logging in to ECR..."
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
  
  build:
    commands:
      - echo "Running tests..."
      - npm test
      - echo "Building Docker image..."
      - docker build -t $ECR_REPO:$IMAGE_TAG .
      - docker push $ECR_REPO:$IMAGE_TAG
  
  post_build:
    commands:
      - echo "Creating deployment artifacts..."
      - printf '[{"name":"%s","imageUri":"%s"}]' $APP_NAME $ECR_REPO:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json

cache:
  paths:
    - 'node_modules/**/*'

reports:
  test-results:
    files:
      - 'test-results/**/*'
    file-format: JUNITXML
```

---

## 8. DORA Metrics (Measuring Pipeline Effectiveness)

### The Four Key Metrics

```
┌─────────────────────────────────────────────────────────────────┐
│  DORA Metrics: Measuring Software Delivery Performance          │
│                                                                   │
│  1. DEPLOYMENT FREQUENCY                                         │
│     "How often do you deploy to production?"                    │
│     ├── Elite: Multiple times per day                           │
│     ├── High: Once per day to once per week                     │
│     ├── Medium: Once per week to once per month                 │
│     └── Low: Once per month to once every 6 months              │
│                                                                   │
│  2. LEAD TIME FOR CHANGES                                        │
│     "Time from code commit to running in production"            │
│     ├── Elite: Less than 1 hour                                 │
│     ├── High: 1 day to 1 week                                   │
│     ├── Medium: 1 week to 1 month                               │
│     └── Low: 1 month to 6 months                                │
│                                                                   │
│  3. CHANGE FAILURE RATE                                          │
│     "% of deployments that cause a failure in production"       │
│     ├── Elite: 0-15%                                            │
│     ├── High: 16-30%                                            │
│     └── Low: 31-45%+                                            │
│                                                                   │
│  4. MEAN TIME TO RECOVERY (MTTR)                                 │
│     "How long to restore service after a failure?"              │
│     ├── Elite: Less than 1 hour                                 │
│     ├── High: Less than 1 day                                    │
│     ├── Medium: 1 day to 1 week                                 │
│     └── Low: More than 1 week                                   │
│                                                                   │
│  KEY INSIGHT: Speed and stability are NOT tradeoffs!            │
│  Elite teams deploy FASTER with FEWER failures.                 │
│  (Because: small changes = less risk = easier rollback)         │
└─────────────────────────────────────────────────────────────────┘
```

### How to Improve Each Metric

```yaml
improve_deployment_frequency:
  - Automate everything (no manual steps)
  - Trunk-based development (small, frequent merges)
  - Feature flags (deploy code without enabling features)
  - Microservices (deploy independently)
  - Remove approval bottlenecks (automated gates instead)

improve_lead_time:
  - Parallelize CI stages (lint, test, build simultaneously)
  - Cache dependencies (npm, Docker layers)
  - Fast tests (< 10 min total, use test sharding)
  - Auto-merge when checks pass (for low-risk changes)
  - Pre-built base images (don't install OS packages every build)

improve_change_failure_rate:
  - Better testing (unit, integration, e2e)
  - Canary deployments (test with real traffic)
  - Feature flags (dark launch, gradual rollout)
  - Smaller changes (less risk per deployment)
  - Code review quality (catch issues before merge)

improve_mttr:
  - Automated rollback (CloudWatch alarm → rollback)
  - Blue-green deployment (instant rollback capability)
  - Good observability (find problems fast)
  - Runbooks for common failures
  - Blameless postmortems (learn and prevent)
```

---

## 9. Multi-Environment Pipeline Patterns

### Environment Promotion

```
┌────────────────────────────────────────────────────────────────┐
│  Promotion Pattern: Same artifact, different configs            │
│                                                                  │
│  Build Once → Promote Through Environments:                     │
│                                                                  │
│  ┌────────┐    ┌─────────┐    ┌──────────┐    ┌────────────┐ │
│  │  Build │───▶│   Dev   │───▶│  Staging │───▶│ Production │ │
│  │ image  │    │ (auto)  │    │ (auto)   │    │ (approval) │ │
│  │ v1.2.3 │    │         │    │          │    │            │ │
│  └────────┘    └─────────┘    └──────────┘    └────────────┘ │
│                                                                  │
│  SAME Docker image (v1.2.3) in ALL environments                │
│  DIFFERENT config (env vars, secrets) per environment          │
│                                                                  │
│  WHY: "Works in staging" = "Will work in prod" (same binary)  │
│  ANTI-PATTERN: Building different image per environment        │
└────────────────────────────────────────────────────────────────┘
```

### Configuration Management

```yaml
# How to handle different configs per environment:

# Option 1: Environment variables (simple)
# Set via ECS task definition, K8s ConfigMap, Lambda env vars
DATABASE_URL: "{{ssm:/prod/database/url}}"  # From SSM Parameter Store

# Option 2: AWS AppConfig (feature flags + config)
# Deploy config changes independently of code
# Gradual rollout, instant rollback, validation

# Option 3: Helm values files (Kubernetes)
# values-dev.yaml, values-staging.yaml, values-prod.yaml
helm upgrade myapp ./chart -f values-prod.yaml

# Option 4: Kustomize overlays (Kubernetes + GitOps)
# base/ (common) + overlays/prod/ (environment-specific patches)
```

---

## 10. Production Best Practices Checklist

```yaml
pipeline_best_practices:
  speed:
    - [ ] Total pipeline time < 15 minutes
    - [ ] Parallelize independent stages
    - [ ] Cache dependencies between runs
    - [ ] Use Docker layer caching
    - [ ] Test sharding for large test suites
    - [ ] Incremental builds (only rebuild what changed)
    
  reliability:
    - [ ] Idempotent deployments (safe to retry)
    - [ ] Automated rollback on failure
    - [ ] Health checks before marking deployment complete
    - [ ] Deployment concurrency control (one at a time)
    - [ ] Pipeline as code (version controlled, reviewable)
    - [ ] No manual steps in production path
    
  security:
    - [ ] OIDC for cloud credentials (no stored keys)
    - [ ] Image scanning in pipeline (fail on critical CVEs)
    - [ ] Secret scanning (prevent credential commits)
    - [ ] Signed artifacts (cosign for containers)
    - [ ] Immutable tags (never overwrite production image)
    - [ ] Branch protection (reviews for main branch)
    - [ ] Ephemeral build environments
    
  observability:
    - [ ] Pipeline duration metrics (track trends)
    - [ ] Failure rate by stage (find bottlenecks)
    - [ ] DORA metrics dashboard
    - [ ] Alert on pipeline failures (Slack/PagerDuty)
    - [ ] Deployment tracking (what version is where?)
    
  team_practices:
    - [ ] Trunk-based development (short-lived branches)
    - [ ] Feature flags for incomplete features
    - [ ] Small, frequent deployments (not big bang releases)
    - [ ] Blameless postmortems for deployment failures
    - [ ] Shared ownership of the pipeline (not one person)
```

---

## 11. Learning Path

```
Beginner (Week 1-2):
├── Create GitHub Actions workflow (lint + test + build)
├── Understand triggers, jobs, steps, actions
├── Use secrets and environment variables
├── Deploy a simple app to AWS (ECS or Lambda)
└── Understand: CI vs CD vs Continuous Deployment

Intermediate (Week 3-4):
├── Multi-stage pipeline with approval gates
├── Docker build + push to ECR
├── Matrix builds and test sharding
├── Caching for speed optimization
├── OIDC authentication (no stored keys)
├── Blue-green deployment with CodeDeploy
└── Branch protection and PR workflows

Advanced (Week 5-8):
├── GitOps with ArgoCD (full setup)
├── Canary deployments with automated analysis
├── Multi-account, multi-region deployments
├── Pipeline security (SLSA, signing, SBOM)
├── Reusable workflows and custom actions
├── Jenkins Pipeline (Kubernetes agents)
├── DORA metrics collection and dashboards
└── Rollback automation (alarm-triggered)

Expert (Month 3+):
├── Design CI/CD for 50+ microservices
├── Internal Developer Platform (self-service deploys)
├── Progressive delivery (Argo Rollouts)
├── Cross-team pipeline standardization
├── Migration between CI/CD systems
├── Cost optimization at scale
└── Compliance automation (SOC2, PCI in pipelines)
```
