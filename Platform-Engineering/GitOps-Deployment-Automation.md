# GitOps & Deployment Automation - Strategies and Patterns

## GitOps Principles

```
┌─────────────────────────────────────────────────────────────┐
│                    GitOps Core Principles                     │
│                                                              │
│  1. Declarative: Desired state defined in code (Git)        │
│  2. Versioned: Git = single source of truth + audit trail   │
│  3. Automated: Changes auto-applied by operators            │
│  4. Self-healing: Drift detected and corrected              │
└─────────────────────────────────────────────────────────────┘

Traditional CI/CD:                    GitOps:
  Developer → CI → CD → Cluster       Developer → Git → Operator → Cluster
  (Push-based)                         (Pull-based)
  
  Pipeline HAS cluster credentials     Operator runs IN the cluster
  Imperative deployment commands       Declarative desired state
  Hard to audit what's deployed        Git history = deployment history
```

## Deployment Strategies Comparison

| Strategy | Zero Downtime | Rollback Speed | Resource Cost | Complexity |
|----------|:---:|:---:|:---:|:---:|
| Rolling Update | ✅ | Slow (re-roll) | 1x | Low |
| Blue-Green | ✅ | Instant (DNS/LB) | 2x | Medium |
| Canary | ✅ | Fast (route shift) | 1.1x | High |
| A/B Testing | ✅ | Fast | 1.1x | Very High |
| Shadow/Mirror | ✅ | N/A | 2x | High |

## Blue-Green Deployment (ECS)

```hcl
# CodeDeploy Blue-Green with ECS
resource "aws_codedeploy_app" "main" {
  compute_platform = "ECS"
  name             = "payment-api"
}

resource "aws_codedeploy_deployment_group" "production" {
  app_name               = aws_codedeploy_app.main.name
  deployment_group_name  = "production"
  deployment_config_name = "CodeDeployDefault.ECSCanary10Percent5Minutes"
  service_role_arn       = aws_iam_role.codedeploy.arn
  
  ecs_service {
    cluster_name = aws_ecs_cluster.production.name
    service_name = aws_ecs_service.payment_api.name
  }
  
  # Blue-Green configuration
  blue_green_deployment_config {
    deployment_ready_option {
      action_on_timeout = "CONTINUE_DEPLOYMENT"
    }
    
    terminate_blue_instances_on_deployment_success {
      action                           = "TERMINATE"
      termination_wait_time_in_minutes = 60  # Keep old for 1hr rollback
    }
  }
  
  # Traffic routing
  load_balancer_info {
    target_group_pair_info {
      prod_traffic_route {
        listener_arns = [aws_lb_listener.https.arn]
      }
      
      test_traffic_route {
        listener_arns = [aws_lb_listener.test.arn]  # Test before live
      }
      
      target_group {
        name = aws_lb_target_group.blue.name
      }
      target_group {
        name = aws_lb_target_group.green.name
      }
    }
  }
  
  # Auto-rollback on alarm
  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }
  
  alarm_configuration {
    alarms  = ["payment-api-error-rate", "payment-api-p99-latency"]
    enabled = true
  }
}
```

## Canary Deployment (Lambda)

```hcl
# Lambda Canary with automatic rollback
resource "aws_lambda_alias" "production" {
  name             = "production"
  function_name    = aws_lambda_function.api.function_name
  function_version = aws_lambda_function.api.version
  
  routing_config {
    additional_version_weights = {
      # Route 10% to new version for canary testing
      (aws_lambda_function.api.version) = 0.1
    }
  }
}

resource "aws_codedeploy_deployment_group" "lambda" {
  app_name               = aws_codedeploy_app.lambda.name
  deployment_group_name  = "production"
  deployment_config_name = "CodeDeployDefault.LambdaCanary10Percent10Minutes"
  service_role_arn       = aws_iam_role.codedeploy.arn
  
  deployment_style {
    deployment_type   = "BLUE_GREEN"
    deployment_option = "WITH_TRAFFIC_CONTROL"
  }
  
  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }
  
  alarm_configuration {
    alarms  = [aws_cloudwatch_metric_alarm.lambda_errors.alarm_name]
    enabled = true
  }
}

# Pre/Post deployment hooks for validation
resource "aws_codedeploy_deployment_group" "lambda_validated" {
  # ...
  
  # BeforeAllowTraffic: Run integration tests against new version
  # AfterAllowTraffic: Validate metrics look good
}
```

## Feature Flags for Safe Deployments

```python
# Using AWS AppConfig for feature flags
import boto3
import json

class FeatureFlagManager:
    def __init__(self, app_id, env_id, config_profile_id):
        self.client = boto3.client('appconfigdata')
        self.app_id = app_id
        self.env_id = env_id
        self.config_profile_id = config_profile_id
        self._token = None
        self._flags = {}
        self._initialize_session()
    
    def _initialize_session(self):
        response = self.client.start_configuration_session(
            ApplicationIdentifier=self.app_id,
            EnvironmentIdentifier=self.env_id,
            ConfigurationProfileIdentifier=self.config_profile_id,
            RequiredMinimumPollIntervalInSeconds=15
        )
        self._token = response['InitialConfigurationToken']
        self._refresh_flags()
    
    def _refresh_flags(self):
        response = self.client.get_latest_configuration(
            ConfigurationToken=self._token
        )
        self._token = response['NextPollConfigurationToken']
        if response['Configuration']:
            self._flags = json.loads(response['Configuration'].read())
    
    def is_enabled(self, flag_name: str, context: dict = None) -> bool:
        """Check if feature flag is enabled."""
        flag = self._flags.get(flag_name, {})
        
        if not flag.get('enabled', False):
            return False
        
        # Percentage rollout
        if 'percentage' in flag:
            user_hash = hash(context.get('user_id', '')) % 100
            return user_hash < flag['percentage']
        
        # Targeting rules
        if 'rules' in flag and context:
            for rule in flag['rules']:
                if self._evaluate_rule(rule, context):
                    return rule.get('enabled', True)
        
        return flag.get('enabled', False)

# Usage in application:
flags = FeatureFlagManager(
    app_id='payment-api',
    env_id='production',
    config_profile_id='feature-flags'
)

@app.post("/v1/payments")
async def create_payment(payment: Payment, request: Request):
    # Gradually roll out new payment processor
    if flags.is_enabled('new_payment_processor', context={'user_id': payment.user_id}):
        result = await new_processor.charge(payment)
    else:
        result = await legacy_processor.charge(payment)
    
    return result
```

## Multi-Region Deployment Pipeline

```yaml
# Deployment order: Primary → Secondary (with validation gates)
name: Multi-Region Production Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - id: build
        run: |
          TAG="${{ github.sha }}"
          docker build -t $ECR_REPO:$TAG .
          docker push $ECR_REPO:$TAG
          echo "tag=$TAG" >> $GITHUB_OUTPUT
  
  deploy-primary:
    needs: build
    runs-on: ubuntu-latest
    environment: production-us-east-1
    steps:
      - name: Deploy to Primary (us-east-1)
        run: |
          ./scripts/deploy-ecs.sh \
            --cluster production \
            --service payment-api \
            --image $ECR_REPO:${{ needs.build.outputs.image_tag }} \
            --region us-east-1 \
            --strategy canary \
            --canary-percent 10
      
      - name: Validate Primary (10 minutes)
        run: |
          ./scripts/validate-deployment.sh \
            --region us-east-1 \
            --service payment-api \
            --duration 600 \
            --error-threshold 0.1
      
      - name: Promote to 100%
        run: |
          ./scripts/deploy-ecs.sh \
            --cluster production \
            --service payment-api \
            --image $ECR_REPO:${{ needs.build.outputs.image_tag }} \
            --region us-east-1 \
            --strategy canary \
            --canary-percent 100
  
  deploy-secondary:
    needs: [build, deploy-primary]
    runs-on: ubuntu-latest
    environment: production-eu-west-1
    steps:
      - name: Deploy to Secondary (eu-west-1)
        run: |
          ./scripts/deploy-ecs.sh \
            --cluster production \
            --service payment-api \
            --image $ECR_REPO:${{ needs.build.outputs.image_tag }} \
            --region eu-west-1 \
            --strategy canary \
            --canary-percent 10
      
      - name: Validate Secondary
        run: |
          ./scripts/validate-deployment.sh \
            --region eu-west-1 \
            --service payment-api \
            --duration 600
      
      - name: Promote to 100%
        run: |
          ./scripts/deploy-ecs.sh \
            --cluster production \
            --service payment-api \
            --image $ECR_REPO:${{ needs.build.outputs.image_tag }} \
            --region eu-west-1 \
            --strategy canary \
            --canary-percent 100
  
  deploy-tertiary:
    needs: [build, deploy-secondary]
    runs-on: ubuntu-latest
    environment: production-us-west-2
    steps:
      - name: Deploy to Tertiary (us-west-2)
        run: |
          # Same pattern as above
          ./scripts/deploy-ecs.sh \
            --cluster production \
            --service payment-api \
            --image $ECR_REPO:${{ needs.build.outputs.image_tag }} \
            --region us-west-2 \
            --strategy rolling
```

## Deployment Validation Patterns

```python
# scripts/validate-deployment.sh (Python equivalent)
import boto3
import time
import sys

def validate_deployment(region, service_name, duration_seconds, error_threshold):
    """Validate deployment health over a time window."""
    cloudwatch = boto3.client('cloudwatch', region_name=region)
    
    start_time = time.time()
    errors_detected = 0
    checks_performed = 0
    
    while time.time() - start_time < duration_seconds:
        # Check error rate
        response = cloudwatch.get_metric_data(
            MetricDataQueries=[
                {
                    'Id': 'error_rate',
                    'Expression': 'm1 / m2 * 100',
                    'Label': 'Error Rate %'
                },
                {
                    'Id': 'm1',
                    'MetricStat': {
                        'Metric': {
                            'Namespace': 'AWS/ApplicationELB',
                            'MetricName': 'HTTPCode_Target_5XX_Count',
                            'Dimensions': [{'Name': 'TargetGroup', 'Value': f'targetgroup/{service_name}/xxx'}]
                        },
                        'Period': 60,
                        'Stat': 'Sum'
                    },
                    'ReturnData': False
                },
                {
                    'Id': 'm2',
                    'MetricStat': {
                        'Metric': {
                            'Namespace': 'AWS/ApplicationELB',
                            'MetricName': 'RequestCount',
                            'Dimensions': [{'Name': 'TargetGroup', 'Value': f'targetgroup/{service_name}/xxx'}]
                        },
                        'Period': 60,
                        'Stat': 'Sum'
                    },
                    'ReturnData': False
                }
            ],
            StartTime=time.time() - 120,
            EndTime=time.time()
        )
        
        for result in response['MetricDataResults']:
            if result['Id'] == 'error_rate' and result['Values']:
                current_error_rate = result['Values'][0]
                checks_performed += 1
                
                if current_error_rate > error_threshold:
                    errors_detected += 1
                    if errors_detected >= 3:
                        print(f"❌ Validation FAILED: Error rate {current_error_rate}% exceeds threshold {error_threshold}%")
                        sys.exit(1)
        
        time.sleep(30)  # Check every 30 seconds
    
    print(f"✅ Validation PASSED: {checks_performed} checks, {errors_detected} warnings")
    return True
```

## Rollback Strategies

### Instant Rollback (Blue-Green):
```bash
# Switch ALB back to old target group
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,TargetGroupArn=$OLD_TARGET_GROUP_ARN
```

### ECS Rollback (Previous Task Definition):
```bash
# Get previous task definition
PREV_TD=$(aws ecs describe-services \
  --cluster production \
  --services payment-api \
  --query 'services[0].deployments[?status==`PRIMARY`].taskDefinition' \
  --output text)

# Rollback by deploying previous version
aws ecs update-service \
  --cluster production \
  --service payment-api \
  --task-definition $PREV_TD \
  --force-new-deployment
```

### Database Rollback (Migration Strategy):
```python
# Always write backward-compatible migrations
# Pattern: Expand-Contract for schema changes

# Step 1: EXPAND (add new column, keep old)
# Deploy: Migration adds new_column, code writes to BOTH columns
ALTER TABLE orders ADD COLUMN status_v2 VARCHAR(50);

# Step 2: MIGRATE (backfill data)
UPDATE orders SET status_v2 = CASE status WHEN 0 THEN 'pending' WHEN 1 THEN 'active' END;

# Step 3: CODE SWITCH (read from new, write to both)
# Deploy: Code reads from status_v2

# Step 4: CONTRACT (remove old column - separate deploy)
# Only after confirming everything works
ALTER TABLE orders DROP COLUMN status;
ALTER TABLE orders RENAME COLUMN status_v2 TO status;
```

## Deployment Windows & Change Management

```yaml
# Deployment calendar configuration
deployment_policy:
  environments:
    production:
      allowed_windows:
        - days: [monday, tuesday, wednesday, thursday]
          hours: "09:00-16:00 UTC"
          description: "Standard deployment window"
        - days: [tuesday, thursday]
          hours: "02:00-05:00 UTC"
          description: "Maintenance window (database migrations)"
      
      blocked_periods:
        - start: "2024-11-28"
          end: "2024-12-02"
          reason: "Black Friday / Cyber Monday freeze"
        - start: "2024-12-20"
          end: "2025-01-03"
          reason: "Holiday code freeze"
      
      emergency_deploy:
        requires: "P1 incident ticket + VP approval"
        approvers: ["sre-lead", "eng-vp"]
    
    staging:
      allowed_windows:
        - days: [monday, tuesday, wednesday, thursday, friday]
          hours: "00:00-23:59 UTC"
          description: "Anytime on weekdays"
```
