# Git / Bitbucket CI-CD Pipeline Automation & Security — Deep-Dive Interview Q&A

## Table of Contents
1. [Git Advanced Concepts](#git-advanced-concepts)
2. [Branching Strategies](#branching-strategies)
3. [CI/CD Pipeline Design](#cicd-pipeline-design)
4. [Pipeline Security](#pipeline-security)
5. [Tricky Scenarios](#tricky-scenarios)

---

## Git Advanced Concepts


**Q1: Explain `git rebase` vs `git merge`. When would you use each? What are the dangers?**

**A:**

```
MERGE (preserves history):
main:    A---B---C---F (merge commit)
              \     /
feature:       D---E

REBASE (linear history):
main:    A---B---C
feature:             D'---E' (replayed on top of C)
```

| Feature | Merge | Rebase |
|---------|-------|--------|
| History | Preserves branching | Linear, clean |
| Commits | Creates merge commit | Rewrites commit hashes |
| Conflicts | Resolve once | Resolve per commit |
| Safe for shared branches | Yes | NO (rewrites history!) |
| Use when | Merging to main, collaboration | Cleaning local feature branch |

**Golden rule**: NEVER rebase commits that have been pushed to a shared branch. It rewrites history — other developers' copies diverge.

**Safe rebase workflow:**
```bash
# On your LOCAL feature branch (before push):
git fetch origin
git rebase origin/main   # Replay your commits on top of latest main
# Resolve conflicts if any
git push origin feature   # Or force push if already pushed: git push --force-with-lease
```

**Tricky**: `--force-with-lease` is safer than `--force`. It fails if the remote has commits you haven't fetched (prevents overwriting others' work).

---

**Q2: Explain `git cherry-pick`, `git stash`, and `git bisect`. Real-world use cases.**

**A:**

**Cherry-pick** (apply specific commit to another branch):
```bash
# Hotfix: Apply one commit from develop to main
git checkout main
git cherry-pick abc123   # Apply commit abc123's changes
git cherry-pick abc123..def456  # Range of commits
git cherry-pick -n abc123  # Stage changes without committing (for modifications)
```
Use case: Hotfix needs to go to production but full branch isn't ready.

**Stash** (save uncommitted work temporarily):
```bash
git stash                     # Save working directory changes
git stash push -m "WIP auth"  # Named stash
git stash list                # View all stashes
git stash pop                 # Apply and remove latest stash
git stash apply stash@{2}    # Apply specific stash (keep in list)
git stash drop stash@{0}     # Delete specific stash
git stash branch new-branch   # Create branch from stash
```
Use case: Need to switch branches but have uncommitted changes.

**Bisect** (find commit that introduced a bug):
```bash
git bisect start
git bisect bad          # Current commit is broken
git bisect good v1.0    # This tag/commit was working
# Git checks out middle commit — you test it
git bisect good         # This commit is fine
git bisect bad          # This commit has the bug
# Repeat until Git identifies the exact commit
git bisect reset        # Exit bisect mode

# Automated bisect with test script:
git bisect run ./test.sh  # Runs test on each commit automatically
```

---

**Q3: How do you recover from common Git disasters?**

**A:**

```bash
# 1. Undo last commit (keep changes staged)
git reset --soft HEAD~1

# 2. Undo last commit (keep changes unstaged)
git reset --mixed HEAD~1  # Default

# 3. Completely discard last commit
git reset --hard HEAD~1

# 4. Recover accidentally deleted branch
git reflog  # Find the commit hash
git checkout -b recovered-branch abc123

# 5. Undo a pushed commit (safe — creates new commit)
git revert abc123
git push

# 6. Remove sensitive file from ALL history
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secrets.env' HEAD
# Or better: use BFG Repo-Cleaner
bfg --delete-files secrets.env

# 7. Fix commit message (not yet pushed)
git commit --amend -m "Correct message"

# 8. Add file to last commit
git add forgotten-file.txt
git commit --amend --no-edit

# 9. Recover after hard reset
git reflog  # Shows ALL head movements (even resets)
git reset --hard abc123  # Go back to the reflog entry
```

**Tricky**: `git reflog` is your safety net! It records every HEAD change for 90 days. Even `git reset --hard` can be undone if you haven't run `git gc`.

---

## Branching Strategies

**Q4: Compare GitFlow, GitHub Flow, and Trunk-Based Development.**

**A:**

| Strategy | Branches | Complexity | Release | Best For |
|----------|----------|-----------|---------|----------|
| GitFlow | main, develop, feature, release, hotfix | High | Scheduled releases | Large teams, versioned software |
| GitHub Flow | main + feature branches | Low | Continuous (merge = deploy) | SaaS, small teams |
| Trunk-Based | main (trunk) + short-lived branches | Lowest | Continuous | High-velocity teams, CI/CD |

**GitFlow:**
```
main ─────●─────────────●──────── (production releases)
           \           /
develop ────●───●───●──── (integration)
             \   \ /
feature/x ────●───● (merged to develop)
```

**Trunk-Based Development:**
```
main ──●──●──●──●──●──●──●── (always deployable)
        \  /  \  /
         ●     ●  (short-lived branches, < 1 day)
```

**Tricky**: GitFlow is OVERKILL for most modern teams. If you deploy continuously (SaaS), Trunk-Based with feature flags is preferred. GitFlow suits teams with scheduled releases (mobile apps, desktop software).

---

## CI/CD Pipeline Design

**Q5: Design a complete CI/CD pipeline for a microservices application. Include all stages.**

**A:**

```yaml
# Jenkinsfile / GitHub Actions / Bitbucket Pipelines
stages:
  1. Code Quality:
     ├── Lint (ESLint, pylint, golangci-lint)
     ├── Static Analysis (SonarQube)
     ├── Secret scanning (gitleaks, trufflehog)
     └── License compliance check

  2. Build:
     ├── Compile/transpile
     ├── Docker build (multi-stage)
     ├── Tag image (git SHA + branch)
     └── Push to ECR/registry

  3. Test:
     ├── Unit tests (fast, isolated)
     ├── Integration tests (with test DB)
     ├── Contract tests (Pact)
     └── Security scan (Trivy on image)

  4. Deploy to Staging:
     ├── Terraform plan/apply (infra)
     ├── ECS/K8s deployment
     ├── Database migrations
     └── Smoke tests

  5. Approval Gate:
     └── Manual approval for production

  6. Deploy to Production:
     ├── Blue/green or canary deployment
     ├── Health check validation
     ├── Synthetic monitoring
     └── Automatic rollback on failure

  7. Post-Deploy:
     ├── Notify team (Slack/Teams)
     ├── Update changelog
     ├── Tag release
     └── Monitor metrics (15 min bake time)
```

---

**Q6: How do you implement CI/CD for Terraform infrastructure changes?**

**A:**

```yaml
# GitHub Actions for Terraform
name: Infrastructure
on:
  pull_request:
    paths: ['terraform/**']
  push:
    branches: [main]
    paths: ['terraform/**']

jobs:
  validate:
    steps:
      - terraform fmt -check
      - terraform validate
      - tflint
      - checkov --directory terraform/  # Security/compliance scanning

  plan:
    needs: validate
    steps:
      - terraform init
      - terraform plan -out=plan.tfplan
      - # Post plan output as PR comment
      - # Upload plan artifact

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    environment: production  # Requires approval
    steps:
      - terraform apply plan.tfplan
      - # Verify with smoke tests
```

**Key practices:**
1. Plan on PR (automated), Apply on merge (with approval)
2. Post plan output as PR comment for review
3. Use saved plan file (plan reviewed = plan applied)
4. Lock state during pipeline execution
5. Checkov/tfsec for security scanning
6. Separate pipelines per environment
7. Never `terraform apply -auto-approve` without saved plan

---

**Q7: Explain Jenkins Pipeline vs GitHub Actions vs Bitbucket Pipelines. Key differences.**

**A:**

| Feature | Jenkins | GitHub Actions | Bitbucket Pipelines |
|---------|---------|---------------|-------------------|
| Hosting | Self-hosted (or Cloud) | GitHub-hosted | Atlassian-hosted |
| Config | Jenkinsfile (Groovy) | YAML (.github/workflows/) | YAML (bitbucket-pipelines.yml) |
| Plugins | 1800+ plugins | Actions marketplace | Pipes marketplace |
| Runners | Agents (any OS) | GitHub-hosted or self-hosted | Cloud or self-hosted runners |
| Parallelism | Stages/parallel blocks | Jobs run in parallel by default | Steps in parallel |
| Secrets | Credentials plugin | Repository/Org secrets | Repository variables |
| Matrix builds | Declarative matrix | Native matrix strategy | Manual definition |
| Cost | Free (self-hosted infra) | Free tier (2000 min/month) | Free tier (50 min/month) |
| Docker support | Docker pipeline plugin | Native | Native |

**Tricky**: Jenkins gives maximum flexibility but requires maintenance (plugins, security patches, scaling agents). GitHub Actions is simpler but limited for complex orchestration. Choose based on: team size, existing tooling, security requirements.

---

## Pipeline Security

**Q8: How do you secure a CI/CD pipeline? What are the attack vectors?**

**A:**

**Attack vectors:**
1. **Compromised dependencies**: Malicious npm/pip packages
2. **Secrets in code**: API keys, passwords committed
3. **Pipeline poisoning**: Malicious PR modifies pipeline config
4. **Container image vulnerabilities**: Base image has CVEs
5. **Build server compromise**: Attacker gains access to CI server
6. **Supply chain attacks**: Compromised GitHub Actions / plugins

**Security measures:**

```yaml
# 1. Pin action versions (prevent supply chain attacks)
- uses: actions/checkout@v4.1.1  # Pin to specific version, NOT @main

# 2. Use OIDC for cloud credentials (no static keys)
permissions:
  id-token: write
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/github-actions
      aws-region: us-east-1

# 3. Dependency scanning
- run: npm audit --audit-level=high
- run: snyk test

# 4. Secret scanning
- uses: trufflesecurity/trufflehog@main
  with:
    extra_args: --only-verified

# 5. Image scanning
- run: trivy image --severity HIGH,CRITICAL myapp:latest

# 6. Signed commits requirement
# Branch protection: Require signed commits

# 7. CODEOWNERS for pipeline files
# .github/CODEOWNERS:
# .github/workflows/ @security-team
# Jenkinsfile @platform-team
```

**Pipeline hardening:**
- Restrict who can modify pipeline files (CODEOWNERS)
- Require PR reviews for pipeline changes
- Use ephemeral build environments (no persistent state)
- Separate secrets per environment (dev can't access prod secrets)
- Audit trail for all deployments
- SLSA framework compliance for supply chain security

---

**Q9: How do you implement secrets management in CI/CD pipelines?**

**A:**

| Method | Security Level | Provider |
|--------|---------------|---------|
| Env vars in CI settings | Medium | All CI tools |
| Vault integration | High | HashiCorp Vault |
| AWS Secrets Manager | High | AWS-native |
| OIDC tokens | Highest (no static secrets) | GitHub, GitLab |
| Sealed Secrets | High (GitOps) | Kubernetes |

**Best practice — OIDC (no secrets stored):**
```yaml
# GitHub Actions — ZERO static credentials
jobs:
  deploy:
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456:role/GitHubActionsRole
          aws-region: us-east-1
      # AWS credentials are temporary (1 hour), no stored secrets!
```

**AWS Trust Policy for OIDC:**
```json
{
  "Effect": "Allow",
  "Principal": {"Federated": "arn:aws:iam::123456:oidc-provider/token.actions.githubusercontent.com"},
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
    }
  }
}
```

**Tricky**: Restrict OIDC trust to specific branches! Without the `sub` condition, ANY branch in the repo can assume the production role. A developer could push to a feature branch and access production secrets.

---

## Tricky Scenarios

**Q10: A deployment passed all tests but broke production. How do you prevent this and implement safe deployments?**

**A:**

**Prevention layers:**
1. **Feature flags**: Deploy code without activating it
2. **Canary deployments**: Route 5% traffic to new version first
3. **Blue/green**: Instant rollback by switching traffic
4. **Progressive delivery**: Gradual traffic shift with automated rollback

**Automated rollback criteria:**
```yaml
# Deploy with automatic rollback
canary:
  steps:
    - setWeight: 10   # 10% traffic
    - pause: {duration: 5m}
    - analysis:        # Check metrics
        metrics:
          - name: error-rate
            threshold: 0.01    # Rollback if >1% errors
          - name: latency-p99
            threshold: 500     # Rollback if p99 > 500ms
    - setWeight: 50
    - pause: {duration: 10m}
    - analysis: ...
    - setWeight: 100
  rollback:
    - setWeight: 0   # Instant rollback to previous version
```

**Post-incident checklist:**
- Why didn't tests catch it? (Add test for this scenario)
- Why didn't canary detect it? (Improve canary metrics)
- How long was MTTR? (Automate rollback faster)
- Was monitoring sufficient? (Add alerts)

---

**Q11: How do you handle database migrations in CI/CD without downtime?**

**A:**

**Expand-Contract Pattern:**
```
Phase 1 (Expand): Add new column/table (backward compatible)
  ├── Deploy migration: ALTER TABLE users ADD COLUMN email_v2 VARCHAR(255)
  ├── Old code still works (uses old column)
  └── New code writes to BOTH columns

Phase 2 (Migrate): Backfill data
  ├── Background job: Copy data from old column to new
  └── Both columns in sync

Phase 3 (Contract): Remove old column (after all services updated)
  ├── Deploy code that only uses new column
  └── DROP old column (separate deployment, days/weeks later)
```

**Rules for zero-downtime migrations:**
1. NEVER rename a column (add new, migrate, drop old)
2. NEVER drop a column in same deploy as code change
3. NEVER add NOT NULL without default value
4. Always make migrations backward compatible
5. Separate deployment of code and schema changes

**Tools:** Flyway, Liquibase, Django migrations, Rails migrations, Alembic (Python)

---

**Q12: Explain Bitbucket Pipelines caching, artifacts, and parallel steps.**

**A:**

```yaml
# bitbucket-pipelines.yml
definitions:
  caches:
    npm: ~/.npm
    docker: /var/lib/docker

pipelines:
  pull-requests:
    '**':
      - parallel:
          - step:
              name: Lint
              caches: [npm]
              script:
                - npm ci
                - npm run lint
          - step:
              name: Test
              caches: [npm]
              script:
                - npm ci
                - npm test
              artifacts:
                - coverage/**

  branches:
    main:
      - step:
          name: Build & Push
          caches: [docker]
          services: [docker]
          script:
            - docker build -t myapp:${BITBUCKET_COMMIT} .
            - pipe: atlassian/aws-ecr-push-image:2.0.0
              variables:
                IMAGE_NAME: myapp
                TAGS: ${BITBUCKET_COMMIT}
      - step:
          name: Deploy to Staging
          deployment: staging
          script:
            - pipe: atlassian/aws-ecs-deploy:1.0.0
              variables:
                CLUSTER_NAME: staging-cluster
                SERVICE_NAME: web-service
      - step:
          name: Deploy to Production
          deployment: production
          trigger: manual  # Manual gate!
          script:
            - pipe: atlassian/aws-ecs-deploy:1.0.0
              variables:
                CLUSTER_NAME: prod-cluster
                SERVICE_NAME: web-service
```

**Tricky**: Bitbucket Pipelines has a 4GB memory limit per step and 2-hour timeout. For large Docker builds, use external build services or multi-step builds. Also: caches expire after 7 days regardless of activity.



---

## Additional Scenario-Based Tricky Questions

---

**Q13: Two developers are working on the same file. Developer A merges their PR first. Developer B's PR now has merge conflicts. B resolves conflicts and merges. After merge, the feature A implemented is broken — A's code is gone. What happened and how do you prevent this?**

**A:**

**The "evil merge" problem:**
```
Timeline:
1. Both A and B branch from main at commit C0
2. A modifies function processOrder() → adds validation logic
3. B modifies function processOrder() → adds logging logic
4. A merges first → main has A's validation logic
5. B pulls main into their branch → CONFLICT in processOrder()
6. B resolves conflict by keeping THEIR version (removes A's validation)
7. B merges → A's validation code is gone!

Git shows no error — the conflict was "resolved" (just incorrectly).
This is called a "semantic merge conflict" — syntactically valid but logically broken.
```

**Prevention strategies:**
```yaml
# Strategy 1: Branch protection — require up-to-date branch
# GitHub:
branches:
  - name: main
    protection:
      required_status_checks:
        strict: true  # Branch must be up-to-date with main before merge
        # Forces B to rebase/merge main into branch BEFORE PR can merge
        # CI runs on the MERGED result, catching broken functionality

# Strategy 2: Require re-review after conflict resolution
# If PR has merge conflicts → resolve → require NEW approval
# GitHub doesn't do this natively, use required status checks:
- name: Check for conflict resolution
  run: |
    CONFLICT_COMMITS=$(git log --oneline --merges origin/main..HEAD | wc -l)
    if [ "$CONFLICT_COMMITS" -gt 0 ]; then
      echo "::warning::Branch contains merge commits — re-review recommended"
    fi

# Strategy 3: Merge queue (serializes merges)
# GitHub merge queue / Mergify:
# Only one PR merges at a time
# Each PR rebased on latest main before merge
# Eliminates race condition entirely
```

**Detection after the fact:**
```bash
# Find the commit that removed A's code
git log -p --follow -S "validateOrder" -- src/orders/processor.js
# -S flag finds commits that ADD or REMOVE the string "validateOrder"
# Shows exactly which commit removed A's validation logic

# Blame to find who resolved the conflict incorrectly
git blame src/orders/processor.js | grep -A5 -B5 "processOrder"
```

**Tricky**: `git merge` vs `git rebase` handles this differently. With rebase, B's commits are replayed on top of A's — conflicts are resolved per-commit (easier to see what's lost). With merge commit, all conflicts are resolved at once (easier to accidentally discard changes). GitHub's "Require linear history" forces rebase (no merge commits) — reduces evil merges but makes history more complex. The best prevention is merge queues that test the INTEGRATED result before merging.

---

**Q14: Your CI pipeline takes 45 minutes. Developers skip running it locally and push directly, causing many failed builds. How do you reduce pipeline time to under 10 minutes while keeping the same test coverage?**

**A:**

**Pipeline audit (find the bottleneck):**
```bash
# Typical 45-min pipeline breakdown:
# - Checkout + dependencies: 5 min (npm install, docker pull)
# - Build: 8 min (TypeScript compilation, Docker build)
# - Unit tests: 5 min
# - Integration tests: 15 min ← MAIN BOTTLENECK
# - E2E tests: 10 min ← SECOND BOTTLENECK
# - Security scan: 2 min
```

**Optimization strategies:**
```yaml
# Strategy 1: Parallelization (biggest win)
# Run independent stages concurrently:
jobs:
  unit-tests:          # 5 min
    runs-on: ubuntu-latest
    steps: ...
  integration-tests:   # 15 min → split into 3 parallel jobs = 5 min each
    strategy:
      matrix:
        shard: [1, 2, 3]
    steps:
      - run: npx jest --shard=${{ matrix.shard }}/3
  e2e-tests:           # 10 min → split into 4 parallel jobs = 2.5 min each
    strategy:
      matrix:
        spec: [auth, orders, payments, ui]
    steps:
      - run: cypress run --spec "cypress/e2e/${{ matrix.spec }}/**"
  security-scan:       # 2 min (parallel with tests)
    runs-on: ubuntu-latest

# Total: max(5, 5, 2.5, 2) = 5 min (vs 45 min sequential!)

# Strategy 2: Caching (eliminate repeated downloads)
- uses: actions/cache@v3
  with:
    path: |
      ~/.npm
      node_modules
      ~/.cache/Cypress
    key: deps-${{ hashFiles('package-lock.json') }}

# Strategy 3: Conditional execution (only test what changed)
- name: Detect changed files
  id: changes
  uses: dorny/paths-filter@v2
  with:
    filters: |
      backend:
        - 'src/api/**'
      frontend:
        - 'src/ui/**'

- name: Run backend tests
  if: steps.changes.outputs.backend == 'true'
  run: npm run test:backend

# Strategy 4: Test impact analysis
# Only run tests that are affected by changed code
# Using: jest --changedSince=origin/main
- run: npx jest --changedSince=origin/main --coverage
```

**Local developer experience:**
```bash
# Pre-commit hooks (catch issues BEFORE push)
# .husky/pre-commit:
#!/bin/sh
npx lint-staged  # Only lint changed files (2 seconds)

# .husky/pre-push:
#!/bin/sh
npx jest --changedSince=origin/main --bail  # Only affected tests (30 seconds)

# Result: Developer gets feedback in 30s locally
# CI still runs full suite but failures are rare (caught locally)
```

**Tricky**: Parallel test execution requires tests to be INDEPENDENT. If integration tests share a database, running them in parallel causes data conflicts and flaky tests. Fix: each parallel shard gets its own database schema (create schema per shard, drop after). For E2E tests: each parallel worker gets its own browser instance and test user. The parallelization investment pays off immediately but requires test isolation work upfront.

---

**Q15: Your team uses Bitbucket Pipelines. A production deployment pipeline failed halfway through — database migration ran successfully but application deployment failed. Now production has the new schema but old code. How do you handle this?**

**A:**

**Assessment:**
```
Current state:
- Database: NEW schema (migration applied ✓)
- Application: OLD code (deployment failed ✗)
- Risk: Old code may not work with new schema

Questions to answer immediately:
1. Is old code compatible with new schema? (expand-contract?)
2. Is the application currently serving errors?
3. Can we roll forward (fix deploy) faster than rolling back (reverse migration)?
```

**Decision tree:**
```
If old code works with new schema (expand-contract pattern):
  → No emergency. Fix the deployment issue and retry.
  
If old code BREAKS with new schema:
  → EMERGENCY. Must either:
    a) Force deploy new code (fix whatever failed)
    b) Reverse the migration (risky with data loss)
    c) Deploy a compatibility patch (old code + adapter for new schema)
```

**Handling each case:**
```bash
# Case A: Fix and retry deployment
# Check why deployment failed:
# - Docker image push failed? → retry push
# - K8s deployment timeout? → increase timeout, check resources
# - Health check failing? → check new code compatibility
# Fix the issue and re-run deployment step ONLY (not migration again)

# Case B: Reverse migration (DANGEROUS)
# Only possible if migration was backward-compatible (no data loss)
# Example: Added column → can drop column safely
# But: If migration MOVED data or DROPPED columns → data lost → can't reverse!
psql -h prod-db -c "
  -- Only if migration ADDED a column (safe to reverse):
  ALTER TABLE orders DROP COLUMN IF EXISTS new_column;
"

# Case C: Deploy compatibility shim
# Quick patch that makes old code work with new schema:
# Add column alias or view that maps new schema to old code expectations
psql -h prod-db -c "
  -- Create view that old code can query:
  CREATE OR REPLACE VIEW orders_compat AS
  SELECT *, new_column AS old_column_name FROM orders;
"
```

**Pipeline design to prevent this:**
```yaml
# Bitbucket pipeline with atomic deployment:
pipelines:
  branches:
    main:
      - step:
          name: Build & Test
          script:
            - npm run build
            - npm run test
          artifacts:
            - dist/**

      - step:
          name: Deploy (atomic)
          deployment: production
          script:
            # Step 1: Deploy new code FIRST (must handle both old and new schema)
            - kubectl set image deployment/myapp myapp=$IMAGE
            - kubectl rollout status deployment/myapp --timeout=300s

            # Step 2: Run migration ONLY after new code is deployed and healthy
            - kubectl run migration --image=$IMAGE --rm --restart=Never -- npm run migrate

            # If migration fails → new code still works with old schema
            # If deployment fails → old code still works with old schema
            # NO partial state!
```

**Tricky**: The safe order is: (1) Deploy new code that works with BOTH old and new schema, (2) Run migration, (3) Later: deploy code that ONLY works with new schema. This is the "expand-contract" pattern. If you run migration FIRST (as the failed pipeline did), any deployment failure leaves you in a dangerous partial state. Always make migrations backward-compatible and deploy code before migrations.

---
