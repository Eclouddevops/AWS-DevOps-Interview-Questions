# CI/CD Architecture - Pipeline Design & Multi-Account Deployment

## Enterprise CI/CD Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Enterprise CI/CD Architecture                         │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Source Control (GitHub Enterprise)                            │   │
│  │  ├── Application Repos (per team)                             │   │
│  │  ├── Infrastructure Repo (platform team)                      │   │
│  │  ├── GitOps Config Repo (desired state)                       │   │
│  │  └── Shared Libraries (CI templates, modules)                 │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│  ┌───────────────────────────▼───────────────────────────────────┐   │
│  │  CI Layer (Build + Test + Scan)                                │   │
│  │                                                                │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │ Compile │→│ Unit    │→│ SAST/   │→│ Image   │           │   │
│  │  │ & Build │ │ Tests   │ │ DAST    │ │ Build   │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └────┬────┘           │   │
│  │                                             │                 │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐     │                 │   │
│  │  │Container│→│ SBOM    │→│ Sign    │◄────┘                 │   │
│  │  │ Scan    │ │ Gen     │ │ Image   │                        │   │
│  │  └─────────┘ └─────────┘ └─────────┘                        │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│  ┌───────────────────────────▼───────────────────────────────────┐   │
│  │  CD Layer (Deploy + Validate + Promote)                        │   │
│  │                                                                │   │
│  │  ┌────────────┐   ┌────────────┐   ┌────────────────────┐    │   │
│  │  │ Deploy Dev │──▶│ Deploy Stg │──▶│ Deploy Prod        │    │   │
│  │  │ (auto)     │   │ (auto)     │   │ (canary + manual)  │    │   │
│  │  └────────────┘   └────────────┘   └────────────────────┘    │   │
│  │       │                 │                    │                 │   │
│  │  ┌────▼────┐      ┌────▼────┐         ┌────▼────┐           │   │
│  │  │Validate │      │Validate │         │Validate │           │   │
│  │  │ Tests   │      │ E2E     │         │ Canary  │           │   │
│  │  └─────────┘      └─────────┘         └─────────┘           │   │
│  └───────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## GitHub Actions Reusable Workflows

### Shared CI Template:

```yaml
# .github/workflows/shared-ci.yml (in shared-workflows repo)
name: Standard CI Pipeline
on:
  workflow_call:
    inputs:
      language:
        required: true
        type: string
      service_name:
        required: true
        type: string
      ecr_repo:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Language
        uses: actions/setup-python@v5
        if: inputs.language == 'python'
        with:
          python-version: '3.11'
      
      - name: Install Dependencies
        run: |
          if [ "${{ inputs.language }}" = "python" ]; then
            pip install -r requirements.txt -r requirements-dev.txt
          fi
      
      - name: Lint
        run: |
          if [ "${{ inputs.language }}" = "python" ]; then
            ruff check . --output-format github
            mypy src/
          fi
      
      - name: Unit Tests
        run: |
          pytest tests/unit/ --cov=src --cov-report=xml --junitxml=test-results.xml
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          file: coverage.xml
          fail_ci_if_error: true
          threshold: 80%

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Dependency Vulnerability Scan
        run: |
          pip-audit --requirement requirements.txt --format json > audit.json
          # Fail on HIGH/CRITICAL
          jq '.[] | select(.vulns[].fix_versions | length > 0)' audit.json
      
      - name: SAST (Semgrep)
        uses: returntocorp/semgrep-action@v1
        with:
          config: "p/python p/owasp-top-ten p/security-audit"
      
      - name: Secrets Detection
        uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified

  build-and-push:
    needs: [lint-and-test, security-scan]
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      image_digest: ${{ steps.build.outputs.digest }}
    
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      
      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Docker Meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ inputs.ecr_repo }}
          tags: |
            type=sha,prefix=${{ inputs.service_name }}-
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and Push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Trivy Container Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ inputs.ecr_repo }}@${{ steps.build.outputs.digest }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      - name: Generate SBOM
        run: |
          syft ${{ inputs.ecr_repo }}@${{ steps.build.outputs.digest }} -o spdx-json > sbom.json
          aws s3 cp sbom.json s3://artifacts/sbom/${{ inputs.service_name }}/${{ github.sha }}/
      
      - name: Sign Image (Cosign)
        run: |
          cosign sign --yes \
            --key awskms:///alias/container-signing-key \
            ${{ inputs.ecr_repo }}@${{ steps.build.outputs.digest }}
```

### Calling the Shared Workflow:

```yaml
# .github/workflows/ci.yml (in application repo)
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: company/shared-workflows/.github/workflows/shared-ci.yml@v2
    with:
      language: python
      service_name: payment-api
      ecr_repo: 123456789012.dkr.ecr.us-east-1.amazonaws.com/payment-api
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
```

## AWS CodePipeline (Native)

```hcl
resource "aws_codepipeline" "main" {
  name     = "payment-api-pipeline"
  role_arn = aws_iam_role.codepipeline.arn
  
  artifact_store {
    location = aws_s3_bucket.artifacts.bucket
    type     = "S3"
    encryption_key {
      id   = aws_kms_key.pipeline.arn
      type = "KMS"
    }
  }
  
  # Stage 1: Source
  stage {
    name = "Source"
    action {
      name             = "GitHub"
      category         = "Source"
      owner            = "ThirdParty"
      provider         = "GitHub"
      version          = "2"
      output_artifacts = ["source"]
      configuration = {
        ConnectionArn    = aws_codestarconnections_connection.github.arn
        FullRepositoryId = "company/payment-api"
        BranchName       = "main"
      }
    }
  }
  
  # Stage 2: Build
  stage {
    name = "Build"
    action {
      name             = "BuildAndTest"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      version          = "1"
      input_artifacts  = ["source"]
      output_artifacts = ["build_output"]
      configuration = {
        ProjectName = aws_codebuild_project.build.name
      }
    }
  }
  
  # Stage 3: Deploy to Staging
  stage {
    name = "DeployStaging"
    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "ECS"
      version         = "1"
      input_artifacts = ["build_output"]
      role_arn        = "arn:aws:iam::STAGING_ACCOUNT:role/CICDDeployRole"
      configuration = {
        ClusterName = "staging"
        ServiceName = "payment-api"
        FileName    = "imagedefinitions.json"
      }
    }
  }
  
  # Stage 4: Integration Tests
  stage {
    name = "IntegrationTests"
    action {
      name            = "RunTests"
      category        = "Build"
      owner           = "AWS"
      provider        = "CodeBuild"
      version         = "1"
      input_artifacts = ["source"]
      configuration = {
        ProjectName = aws_codebuild_project.integration_tests.name
        EnvironmentVariables = jsonencode([{
          name  = "TARGET_ENV"
          value = "staging"
          type  = "PLAINTEXT"
        }])
      }
    }
  }
  
  # Stage 5: Manual Approval
  stage {
    name = "Approval"
    action {
      name     = "ManualApproval"
      category = "Approval"
      owner    = "AWS"
      provider = "Manual"
      version  = "1"
      configuration = {
        NotificationArn = aws_sns_topic.approvals.arn
        CustomData      = "Review staging deployment before production"
      }
    }
  }
  
  # Stage 6: Deploy to Production (Blue-Green)
  stage {
    name = "DeployProduction"
    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "CodeDeployToECS"
      version         = "1"
      input_artifacts = ["build_output"]
      role_arn        = "arn:aws:iam::PROD_ACCOUNT:role/CICDDeployRole"
      configuration = {
        ApplicationName                = "payment-api"
        DeploymentGroupName            = "production"
        TaskDefinitionTemplateArtifact = "build_output"
        AppSpecTemplateArtifact        = "build_output"
        AppSpecTemplatePath            = "appspec.yaml"
      }
    }
  }
}
```

## Pipeline Security Best Practices

### Supply Chain Security:

```yaml
# Secure pipeline configuration
pipeline_security:
  source_integrity:
    - Signed commits required (GPG)
    - Branch protection rules (2 reviewers)
    - CODEOWNERS file for critical paths
    - Dependabot/Renovate for dependency updates
  
  build_security:
    - Hermetic builds (reproducible)
    - Pinned dependencies (lock files)
    - No internet access during build (VPC-isolated CodeBuild)
    - SBOM generation for every artifact
  
  artifact_security:
    - Image signing (cosign + KMS)
    - Immutable tags in ECR
    - Vulnerability scanning (Trivy/Grype)
    - Signature verification before deployment
  
  deployment_security:
    - OIDC for cross-account access (no stored keys)
    - Least-privilege IAM roles per stage
    - Deployment approval gates
    - Audit trail (CloudTrail)
    - Automated rollback on failure
```

### SLSA (Supply-chain Levels for Software Artifacts):

```yaml
# Level 3 compliance checklist:
slsa_compliance:
  source:
    - Version controlled: ✅ (Git)
    - Verified history: ✅ (signed commits)
    - Two-person review: ✅ (branch protection)
    - Retained indefinitely: ✅ (never force push)
  
  build:
    - Scripted build: ✅ (Dockerfile + CI config)
    - Build service: ✅ (CodeBuild / GitHub Actions)
    - Ephemeral environment: ✅ (containers destroyed after build)
    - Isolated: ✅ (VPC-isolated, no internet)
    - Parameterless: ✅ (build from source only)
  
  provenance:
    - Available: ✅ (SBOM + build metadata in S3)
    - Authenticated: ✅ (signed with KMS key)
    - Non-forgeable: ✅ (build service generates, not developer)
```

## Monitoring CI/CD Pipelines

### Pipeline Metrics Dashboard:

```hcl
resource "aws_cloudwatch_dashboard" "cicd_metrics" {
  dashboard_name = "CICD-Platform-Metrics"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        properties = {
          title  = "Deployment Frequency (DORA)"
          view   = "timeSeries"
          metrics = [
            ["CICD", "DeploymentsPerDay", "Environment", "production"]
          ]
          period = 86400
          stat   = "Sum"
          annotations = {
            horizontal = [{ value = 1, label = "Target: Daily" }]
          }
        }
      },
      {
        type = "metric"
        properties = {
          title  = "Lead Time for Changes (DORA)"
          view   = "timeSeries"
          metrics = [
            ["CICD", "LeadTimeMinutes", "Environment", "production"]
          ]
          period = 86400
          stat   = "Average"
          annotations = {
            horizontal = [{ value = 60, label = "Target: < 1 hour" }]
          }
        }
      },
      {
        type = "metric"
        properties = {
          title  = "Change Failure Rate (DORA)"
          view   = "singleValue"
          metrics = [
            [{
              expression = "failures / deployments * 100"
              label = "Failure Rate %"
            }],
            ["CICD", "FailedDeployments", "Environment", "production", { id = "failures", visible = false }],
            ["CICD", "TotalDeployments", "Environment", "production", { id = "deployments", visible = false }]
          ]
        }
      },
      {
        type = "metric"
        properties = {
          title  = "Mean Time to Recovery (DORA)"
          view   = "timeSeries"
          metrics = [
            ["CICD", "MTTRMinutes", "Environment", "production"]
          ]
          period = 86400
          stat   = "Average"
          annotations = {
            horizontal = [{ value = 60, label = "Target: < 1 hour" }]
          }
        }
      }
    ]
  })
}
```

### DORA Metrics Collection:

```python
# Lambda to track DORA metrics
import boto3
import time
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def track_deployment(event, context):
    """Track deployment metrics for DORA calculations."""
    
    deployment = event['detail']
    
    # Deployment Frequency
    cloudwatch.put_metric_data(
        Namespace='CICD',
        MetricData=[{
            'MetricName': 'DeploymentsPerDay',
            'Value': 1,
            'Unit': 'Count',
            'Dimensions': [
                {'Name': 'Environment', 'Value': deployment['environment']},
                {'Name': 'Service', 'Value': deployment['service_name']}
            ]
        }]
    )
    
    # Lead Time (commit to deploy)
    commit_time = datetime.fromisoformat(deployment['commit_timestamp'])
    deploy_time = datetime.utcnow()
    lead_time_minutes = (deploy_time - commit_time).total_seconds() / 60
    
    cloudwatch.put_metric_data(
        Namespace='CICD',
        MetricData=[{
            'MetricName': 'LeadTimeMinutes',
            'Value': lead_time_minutes,
            'Unit': 'Count',
            'Dimensions': [
                {'Name': 'Environment', 'Value': deployment['environment']},
                {'Name': 'Service', 'Value': deployment['service_name']}
            ]
        }]
    )
    
    # Change Failure Rate
    if deployment.get('status') == 'FAILED' or deployment.get('rolled_back'):
        cloudwatch.put_metric_data(
            Namespace='CICD',
            MetricData=[{
                'MetricName': 'FailedDeployments',
                'Value': 1,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'Environment', 'Value': deployment['environment']}
                ]
            }]
        )
```

## Multi-Account CI/CD Patterns

### Hub-and-Spoke Deployment:

```
┌────────────────────────────────────────────────┐
│  Shared Services Account (CI/CD Hub)           │
│                                                 │
│  CodePipeline / GitHub Actions                  │
│  ├── Build artifacts (ECR, S3)                  │
│  ├── Assume roles in target accounts            │
│  └── Centralized pipeline monitoring            │
│                                                 │
│  Secrets:                                       │
│  ├── OIDC trust (no stored credentials)         │
│  ├── Cross-account KMS for artifact encryption  │
│  └── Secrets Manager (deploy-time only)         │
└────────────────┬────────────────────────────────┘
                 │ AssumeRole (OIDC)
    ┌────────────┼────────────┬────────────┐
    ▼            ▼            ▼            ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ Dev Acct │ │ Stg Acct │ │ Prod-A   │ │ Prod-B   │
│          │ │          │ │ (Primary)│ │ (DR)     │
│ Auto-    │ │ Auto +   │ │ Canary + │ │ After    │
│ deploy   │ │ E2E test │ │ Approval │ │ Primary  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### Cross-Account Artifact Sharing:

```hcl
# ECR Repository Policy (allow target accounts to pull)
resource "aws_ecr_repository_policy" "cross_account" {
  repository = aws_ecr_repository.app.name
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "CrossAccountPull"
      Effect    = "Allow"
      Principal = {
        AWS = [
          "arn:aws:iam::DEV_ACCOUNT:root",
          "arn:aws:iam::STAGING_ACCOUNT:root",
          "arn:aws:iam::PROD_ACCOUNT:root"
        ]
      }
      Action = [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }]
  })
}

# ECR Replication (images available in DR region)
resource "aws_ecr_replication_configuration" "cross_region" {
  replication_configuration {
    rule {
      destination {
        region      = "us-west-2"
        registry_id = data.aws_caller_identity.current.account_id
      }
      destination {
        region      = "eu-west-1"
        registry_id = data.aws_caller_identity.current.account_id
      }
    }
  }
}
```
