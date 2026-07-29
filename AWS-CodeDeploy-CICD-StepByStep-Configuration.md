# AWS CodeDeploy & CI/CD Pipeline - Step-by-Step Configuration Guide

## Table of Contents
1. [Prerequisites & IAM Setup](#1-prerequisites--iam-setup)
2. [CodeDeploy Application Setup](#2-codedeploy-application-setup)
3. [Deployment Group Configuration](#3-deployment-group-configuration)
4. [AppSpec File Configuration](#4-appspec-file-configuration)
5. [Deployment Scripts (Lifecycle Hooks)](#5-deployment-scripts-lifecycle-hooks)
6. [In-Place Deployment](#6-in-place-deployment)
7. [Blue/Green Deployment](#7-bluegreen-deployment)
8. [CodePipeline Integration](#8-codepipeline-integration)
9. [Rollback Configuration](#9-rollback-configuration)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. Prerequisites & IAM Setup

### Step 1.1: Create CodeDeploy Service Role


**AWS Console Path:**
```
IAM → Roles → Create Role → AWS Service → CodeDeploy
```

**AWS CLI:**
```bash
# Create trust policy file
cat > codedeploy-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "codedeploy.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name CodeDeployServiceRole \
  --assume-role-policy-document file://codedeploy-trust-policy.json

# Attach the CodeDeploy policy
aws iam attach-role-policy \
  --role-name CodeDeployServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole
```


### Step 1.2: Create EC2 Instance Profile for CodeDeploy Agent

```bash
# Trust policy for EC2
cat > ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create role
aws iam create-role \
  --role-name EC2-CodeDeploy-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

# Attach policies
aws iam attach-role-policy \
  --role-name EC2-CodeDeploy-Role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforAWSCodeDeploy

aws iam attach-role-policy \
  --role-name EC2-CodeDeploy-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam attach-role-policy \
  --role-name EC2-CodeDeploy-Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Create instance profile
aws iam create-instance-profile --instance-profile-name EC2-CodeDeploy-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-CodeDeploy-Profile \
  --role-name EC2-CodeDeploy-Role
```


### Step 1.3: Install CodeDeploy Agent on EC2

```bash
#!/bin/bash
# Install CodeDeploy Agent - Amazon Linux 2023 / Amazon Linux 2

# Install dependencies
sudo yum install -y ruby wget

# Download and install agent (change region as needed)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
cd /home/ec2-user
wget https://aws-codedeploy-${REGION}.s3.${REGION}.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto

# Verify installation
sudo systemctl status codedeploy-agent
sudo systemctl enable codedeploy-agent

# Check agent logs
tail -f /var/log/aws/codedeploy-agent/codedeploy-agent.log
```

**For Ubuntu/Debian:**
```bash
sudo apt-get install -y ruby-full wget
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
cd /home/ubuntu
wget https://aws-codedeploy-${REGION}.s3.${REGION}.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo systemctl start codedeploy-agent
sudo systemctl enable codedeploy-agent
```

---

## 2. CodeDeploy Application Setup

### Step 2.1: Create CodeDeploy Application

**AWS Console Path:**
```
CodeDeploy → Applications → Create Application
├── Application Name: my-production-app
├── Compute Platform: EC2/On-premises
└── Click "Create Application"
```

**AWS CLI:**
```bash
# Create application for EC2
aws deploy create-application \
  --application-name my-production-app \
  --compute-platform Server \
  --tags Key=Environment,Value=production Key=Team,Value=platform

# Verify
aws deploy get-application --application-name my-production-app
```


---

## 3. Deployment Group Configuration

### Step 3.1: Create Deployment Group (In-Place)

**AWS Console Path:**
```
CodeDeploy → Applications → my-production-app → Create Deployment Group
```

**AWS CLI:**
```bash
aws deploy create-deployment-group \
  --application-name my-production-app \
  --deployment-group-name production-fleet \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --auto-scaling-groups production-api-asg \
  --deployment-style '{
    "deploymentType": "IN_PLACE",
    "deploymentOption": "WITH_TRAFFIC_CONTROL"
  }' \
  --load-balancer-info '{
    "targetGroupInfoList": [{
      "name": "production-api-tg"
    }]
  }' \
  --auto-rollback-configuration '{
    "enabled": true,
    "events": ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }' \
  --alarm-configuration '{
    "enabled": true,
    "ignorePollAlarmFailure": false,
    "alarms": [
      {"name": "HighErrorRate-Production"},
      {"name": "HighLatency-Production"}
    ]
  }' \
  --trigger-configurations '[{
    "triggerName": "deployment-notifications",
    "triggerTargetArn": "arn:aws:sns:us-east-1:123456789012:deployment-alerts",
    "triggerEvents": [
      "DeploymentStart",
      "DeploymentSuccess",
      "DeploymentFailure",
      "DeploymentRollback"
    ]
  }]'
```

### Step 3.2: Create Deployment Group (Blue/Green)

```bash
aws deploy create-deployment-group \
  --application-name my-production-app \
  --deployment-group-name production-blue-green \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole \
  --deployment-config-name CodeDeployDefault.AllAtOnce \
  --auto-scaling-groups production-api-asg \
  --deployment-style '{
    "deploymentType": "BLUE_GREEN",
    "deploymentOption": "WITH_TRAFFIC_CONTROL"
  }' \
  --blue-green-deployment-configuration '{
    "terminateBlueInstancesOnDeploymentSuccess": {
      "action": "TERMINATE",
      "terminationWaitTimeInMinutes": 60
    },
    "deploymentReadyOption": {
      "actionOnTimeout": "CONTINUE_DEPLOYMENT",
      "waitTimeInMinutes": 0
    },
    "greenFleetProvisioningOption": {
      "action": "COPY_AUTO_SCALING_GROUP"
    }
  }' \
  --load-balancer-info '{
    "targetGroupInfoList": [{"name": "production-api-tg"}]
  }' \
  --auto-rollback-configuration '{
    "enabled": true,
    "events": ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }'
```

### Step 3.3: Deployment Configurations

```bash
# Custom deployment config: 25% at a time
aws deploy create-deployment-config \
  --deployment-config-name Production-25Percent \
  --minimum-healthy-hosts type=FLEET_PERCENT,value=75

# Custom: Deploy to 1 instance at a time
aws deploy create-deployment-config \
  --deployment-config-name Production-OneAtATime \
  --minimum-healthy-hosts type=HOST_COUNT,value=0

# Built-in configs:
# CodeDeployDefault.OneAtATime - Safest (1 at a time)
# CodeDeployDefault.HalfAtATime - Moderate (50% at a time)
# CodeDeployDefault.AllAtOnce - Fastest (all at once)
```


---

## 4. AppSpec File Configuration

### Step 4.1: Basic AppSpec.yml (EC2/On-Premises)

```yaml
# appspec.yml - Place in root of application source
version: 0.0
os: linux

files:
  # Copy entire source to destination
  - source: /
    destination: /opt/myapp
    overwrite: true

permissions:
  - object: /opt/myapp
    owner: appuser
    group: appuser
    mode: "755"
    type:
      - directory
  - object: /opt/myapp/scripts
    owner: appuser
    group: appuser
    mode: "755"
    pattern: "*.sh"

hooks:
  # Run before files are copied
  ApplicationStop:
    - location: scripts/stop_application.sh
      timeout: 60
      runas: root

  # Run before the new files are copied
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 120
      runas: root

  # Run after files are copied
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 120
      runas: root

  # Start the application
  ApplicationStart:
    - location: scripts/start_application.sh
      timeout: 60
      runas: root

  # Validate the deployment
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 120
      runas: root
```

### Step 4.2: AppSpec Lifecycle Event Order

```
In-Place Deployment Lifecycle:
═══════════════════════════════
1. ApplicationStop        → Stop the running application
2. DownloadBundle         → (Built-in) Download revision from S3/GitHub
3. BeforeInstall          → Pre-installation tasks (backup, cleanup)
4. Install                → (Built-in) Copy files to destination
5. AfterInstall           → Post-installation tasks (permissions, config)
6. ApplicationStart       → Start the application
7. ValidateService        → Health checks, smoke tests

Blue/Green Additional Events:
═══════════════════════════════
1. BeforeBlockTraffic     → Pre-deregistration tasks
2. BlockTraffic           → (Built-in) Deregister from Load Balancer
3. AfterBlockTraffic      → Post-deregistration tasks
4. ApplicationStop        → Stop old application
5. BeforeInstall          → Prepare new environment
6. Install                → (Built-in) Copy files
7. AfterInstall           → Configure new environment
8. ApplicationStart       → Start new application
9. ValidateService        → Validate new environment
10. BeforeAllowTraffic    → Pre-registration tasks
11. AllowTraffic          → (Built-in) Register with Load Balancer
12. AfterAllowTraffic     → Post-registration validation
```


---

## 5. Deployment Scripts (Lifecycle Hooks)

### Step 5.1: stop_application.sh

```bash
#!/bin/bash
# scripts/stop_application.sh
set -e

echo "[$(date)] Stopping application..."

# Stop Gunicorn/Flask application
if systemctl is-active --quiet myapp; then
    systemctl stop myapp
    echo "[$(date)] Application stopped successfully"
else
    echo "[$(date)] Application was not running"
fi

# Wait for connections to drain
sleep 5

# Verify port is free
if lsof -i :8080 > /dev/null 2>&1; then
    echo "[$(date)] WARNING: Port 8080 still in use, force killing..."
    fuser -k 8080/tcp || true
fi

echo "[$(date)] Application stop complete"
```

### Step 5.2: before_install.sh

```bash
#!/bin/bash
# scripts/before_install.sh
set -e

echo "[$(date)] Running pre-installation tasks..."

# Create application user if not exists
if ! id -u appuser > /dev/null 2>&1; then
    useradd -m -r -s /bin/bash appuser
    echo "[$(date)] Created appuser"
fi

# Backup current deployment
BACKUP_DIR="/opt/myapp-backups/$(date +%Y%m%d_%H%M%S)"
if [ -d /opt/myapp ]; then
    mkdir -p "$BACKUP_DIR"
    cp -r /opt/myapp/* "$BACKUP_DIR/" 2>/dev/null || true
    echo "[$(date)] Backup created at $BACKUP_DIR"
fi

# Clean old backups (keep last 5)
ls -dt /opt/myapp-backups/*/ | tail -n +6 | xargs rm -rf 2>/dev/null || true

# Remove old application files
rm -rf /opt/myapp/*

# Create necessary directories
mkdir -p /opt/myapp
mkdir -p /var/log/myapp
mkdir -p /opt/myapp/tmp

# Set ownership
chown -R appuser:appuser /opt/myapp
chown -R appuser:appuser /var/log/myapp

echo "[$(date)] Pre-installation tasks complete"
```

### Step 5.3: after_install.sh

```bash
#!/bin/bash
# scripts/after_install.sh
set -e

echo "[$(date)] Running post-installation tasks..."

cd /opt/myapp

# Install Python dependencies
if [ -f requirements.txt ]; then
    pip3 install --no-cache-dir -r requirements.txt
    echo "[$(date)] Python dependencies installed"
fi

# Pull configuration from Parameter Store
ENVIRONMENT="production"
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)

aws ssm get-parameters-by-path \
  --path "/myapp/${ENVIRONMENT}/" \
  --with-decryption \
  --region "$REGION" \
  --query 'Parameters[*].[Name,Value]' \
  --output text | while IFS=$'\t' read -r name value; do
    key=$(basename "$name")
    echo "export ${key}='${value}'" >> /opt/myapp/.env
done

# Set permissions
chown -R appuser:appuser /opt/myapp
chmod 600 /opt/myapp/.env

# Configure systemd service
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Flask Application
After=network.target

[Service]
Type=notify
User=appuser
Group=appuser
WorkingDirectory=/opt/myapp
EnvironmentFile=/opt/myapp/.env
ExecStart=/usr/local/bin/gunicorn \
    --bind 0.0.0.0:8080 \
    --workers 4 \
    --threads 2 \
    --timeout 120 \
    --graceful-timeout 30 \
    --keep-alive 65 \
    --max-requests 1000 \
    --max-requests-jitter 50 \
    --access-logfile /var/log/myapp/access.log \
    --error-logfile /var/log/myapp/error.log \
    "app:create_app()"
ExecReload=/bin/kill -s HUP $MAINPID
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload

# Configure log rotation
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    copytruncate
}
EOF

echo "[$(date)] Post-installation tasks complete"
```


### Step 5.4: start_application.sh

```bash
#!/bin/bash
# scripts/start_application.sh
set -e

echo "[$(date)] Starting application..."

# Start the application
systemctl start myapp

# Wait for application to be ready
MAX_RETRIES=30
RETRY_COUNT=0

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    if curl -sf http://localhost:8080/health > /dev/null 2>&1; then
        echo "[$(date)] Application started successfully"
        exit 0
    fi
    RETRY_COUNT=$((RETRY_COUNT + 1))
    echo "[$(date)] Waiting for application to start... (attempt $RETRY_COUNT/$MAX_RETRIES)"
    sleep 2
done

echo "[$(date)] ERROR: Application failed to start within timeout"
journalctl -u myapp --no-pager -n 50
exit 1
```

### Step 5.5: validate_service.sh

```bash
#!/bin/bash
# scripts/validate_service.sh
set -e

echo "[$(date)] Validating service..."

# Check 1: Application is responding
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health)
if [ "$HTTP_CODE" != "200" ]; then
    echo "[$(date)] ERROR: Health check returned $HTTP_CODE"
    exit 1
fi
echo "[$(date)] ✓ Health check passed (HTTP $HTTP_CODE)"

# Check 2: Application can connect to database
DB_CHECK=$(curl -s http://localhost:8080/health | python3 -c "import sys,json; print(json.load(sys.stdin).get('database','unknown'))")
if [ "$DB_CHECK" != "healthy" ]; then
    echo "[$(date)] ERROR: Database connection check failed: $DB_CHECK"
    exit 1
fi
echo "[$(date)] ✓ Database connection verified"

# Check 3: Application can connect to Redis
REDIS_CHECK=$(curl -s http://localhost:8080/health | python3 -c "import sys,json; print(json.load(sys.stdin).get('redis','unknown'))")
if [ "$REDIS_CHECK" != "healthy" ]; then
    echo "[$(date)] WARNING: Redis connection check failed (non-critical)"
fi
echo "[$(date)] ✓ Redis connection verified"

# Check 4: Critical endpoint responding
RESPONSE=$(curl -sf http://localhost:8080/api/v1/status 2>&1)
if [ $? -ne 0 ]; then
    echo "[$(date)] ERROR: Critical endpoint /api/v1/status not responding"
    exit 1
fi
echo "[$(date)] ✓ Critical API endpoint verified"

# Check 5: Response time is acceptable
RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" http://localhost:8080/api/v1/status)
THRESHOLD="2.0"
if (( $(echo "$RESPONSE_TIME > $THRESHOLD" | bc -l) )); then
    echo "[$(date)] ERROR: Response time ${RESPONSE_TIME}s exceeds threshold ${THRESHOLD}s"
    exit 1
fi
echo "[$(date)] ✓ Response time acceptable (${RESPONSE_TIME}s)"

echo "[$(date)] ═══════════════════════════════"
echo "[$(date)] ✓ All validations passed!"
echo "[$(date)] ═══════════════════════════════"
```

---

## 6. In-Place Deployment

### Step 6.1: Create S3 Bucket for Artifacts

```bash
# Create artifact bucket
aws s3api create-bucket \
  --bucket my-codedeploy-artifacts \
  --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-codedeploy-artifacts \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket my-codedeploy-artifacts \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "aws:kms"}}]
  }'
```

### Step 6.2: Push Application to S3

```bash
# Package the application (from project root)
aws deploy push \
  --application-name my-production-app \
  --s3-location s3://my-codedeploy-artifacts/releases/myapp-v1.2.3.zip \
  --source . \
  --description "Release v1.2.3 - Bug fixes and performance improvements"
```

### Step 6.3: Create Deployment

```bash
aws deploy create-deployment \
  --application-name my-production-app \
  --deployment-group-name production-fleet \
  --s3-location bucket=my-codedeploy-artifacts,key=releases/myapp-v1.2.3.zip,bundleType=zip \
  --description "Production deployment v1.2.3" \
  --file-exists-behavior OVERWRITE

# Output: {"deploymentId": "d-XXXXXXXXX"}
```

### Step 6.4: Monitor Deployment

```bash
# Get deployment status
aws deploy get-deployment --deployment-id d-XXXXXXXXX \
  --query 'deploymentInfo.{Status:status,Creator:creator,StartTime:createTime}'

# List deployment instances
aws deploy list-deployment-instances \
  --deployment-id d-XXXXXXXXX \
  --instance-status-filter Succeeded InProgress Failed

# Get instance deployment details
aws deploy get-deployment-instance \
  --deployment-id d-XXXXXXXXX \
  --instance-id i-0123456789abcdef0
```


---

## 7. Blue/Green Deployment

### Step 7.1: Architecture Overview

```
                        ALB
                         │
            ┌────────────┼────────────┐
            │                         │
    Blue Target Group          Green Target Group
    (Current - Live)           (New - Staging)
            │                         │
    ┌───────┴───────┐         ┌───────┴───────┐
    │  ASG (Blue)   │         │  ASG (Green)  │
    │  Instance 1   │         │  Instance 1   │
    │  Instance 2   │         │  Instance 2   │
    │  Instance 3   │         │  Instance 3   │
    └───────────────┘         └───────────────┘

Step 1: Green ASG created (copy of Blue)
Step 2: New code deployed to Green
Step 3: Validation on Green
Step 4: Traffic shifted Blue → Green
Step 5: Blue terminated (after wait period)
```

### Step 7.2: Configure Blue/Green Deployment

```bash
# Create deployment with Blue/Green strategy
aws deploy create-deployment \
  --application-name my-production-app \
  --deployment-group-name production-blue-green \
  --s3-location bucket=my-codedeploy-artifacts,key=releases/myapp-v2.0.0.zip,bundleType=zip \
  --description "Blue/Green deployment v2.0.0" \
  --target-instances '{
    "autoScalingGroups": ["production-api-asg"]
  }' \
  --file-exists-behavior OVERWRITE
```

### Step 7.3: Manual Traffic Rerouting (If Configured)

```bash
# If deploymentReadyOption.actionOnTimeout = "STOP_DEPLOYMENT"
# You must manually continue after validation

# Continue the deployment (shift traffic)
aws deploy continue-deployment \
  --deployment-id d-XXXXXXXXX \
  --deployment-wait-type READY_WAIT

# Or stop and rollback
aws deploy stop-deployment \
  --deployment-id d-XXXXXXXXX \
  --auto-rollback-enabled
```

---

## 8. CodePipeline Integration

### Step 8.1: Create Complete CI/CD Pipeline

**AWS Console Path:**
```
CodePipeline → Create Pipeline → Pipeline Settings
```

**Pipeline Architecture:**
```
Source (GitHub/CodeCommit)
    → Build (CodeBuild - Test & Package)
        → Deploy-Staging (CodeDeploy - Staging)
            → Manual Approval
                → Deploy-Production (CodeDeploy - Blue/Green)
```

### Step 8.2: CodeBuild Project (buildspec.yml)

```yaml
# buildspec.yml
version: 0.2

env:
  variables:
    PYTHON_VERSION: "3.11"
  parameter-store:
    DB_HOST: "/myapp/production/db_host"
    REDIS_HOST: "/myapp/production/redis_host"

phases:
  install:
    runtime-versions:
      python: 3.11
    commands:
      - pip install --upgrade pip
      - pip install -r requirements.txt
      - pip install -r requirements-dev.txt

  pre_build:
    commands:
      - echo "Running linting..."
      - flake8 . --max-line-length=120 --exclude=venv
      - echo "Running security scan..."
      - bandit -r . -x tests
      - echo "Running unit tests..."
      - pytest tests/unit/ -v --cov=app --cov-report=xml

  build:
    commands:
      - echo "Running integration tests..."
      - pytest tests/integration/ -v
      - echo "Building application package..."
      - pip install -r requirements.txt -t ./vendor
      - echo "Build completed on $(date)"

  post_build:
    commands:
      - echo "Creating deployment artifact..."
      - zip -r deployment-package.zip . -x "*.git*" "tests/*" "*.pyc" "__pycache__/*" "venv/*"

artifacts:
  files:
    - '**/*'
  exclude-paths:
    - 'tests/**'
    - '*.pyc'
    - '__pycache__/**'
    - 'venv/**'
    - '.git/**'

reports:
  pytest-reports:
    files:
      - 'coverage.xml'
    file-format: 'COBERTURAXML'

cache:
  paths:
    - '/root/.cache/pip/**/*'
```

### Step 8.3: Create CodePipeline (CLI)

```bash
# Create pipeline
aws codepipeline create-pipeline --pipeline '{
  "name": "my-production-pipeline",
  "roleArn": "arn:aws:iam::123456789012:role/CodePipelineServiceRole",
  "artifactStore": {
    "type": "S3",
    "location": "my-codepipeline-artifacts"
  },
  "stages": [
    {
      "name": "Source",
      "actions": [{
        "name": "SourceAction",
        "actionTypeId": {
          "category": "Source",
          "owner": "ThirdParty",
          "provider": "GitHub",
          "version": "1"
        },
        "outputArtifacts": [{"name": "SourceOutput"}],
        "configuration": {
          "Owner": "myorg",
          "Repo": "myapp",
          "Branch": "main",
          "OAuthToken": "{{resolve:secretsmanager:github-token}}"
        }
      }]
    },
    {
      "name": "Build",
      "actions": [{
        "name": "BuildAction",
        "actionTypeId": {
          "category": "Build",
          "owner": "AWS",
          "provider": "CodeBuild",
          "version": "1"
        },
        "inputArtifacts": [{"name": "SourceOutput"}],
        "outputArtifacts": [{"name": "BuildOutput"}],
        "configuration": {
          "ProjectName": "my-production-build"
        }
      }]
    },
    {
      "name": "Deploy-Staging",
      "actions": [{
        "name": "DeployStaging",
        "actionTypeId": {
          "category": "Deploy",
          "owner": "AWS",
          "provider": "CodeDeploy",
          "version": "1"
        },
        "inputArtifacts": [{"name": "BuildOutput"}],
        "configuration": {
          "ApplicationName": "my-production-app",
          "DeploymentGroupName": "staging-fleet"
        }
      }]
    },
    {
      "name": "Approval",
      "actions": [{
        "name": "ManualApproval",
        "actionTypeId": {
          "category": "Approval",
          "owner": "AWS",
          "provider": "Manual",
          "version": "1"
        },
        "configuration": {
          "NotificationArn": "arn:aws:sns:us-east-1:123456789012:pipeline-approvals",
          "CustomData": "Please review staging deployment before production"
        }
      }]
    },
    {
      "name": "Deploy-Production",
      "actions": [{
        "name": "DeployProduction",
        "actionTypeId": {
          "category": "Deploy",
          "owner": "AWS",
          "provider": "CodeDeploy",
          "version": "1"
        },
        "inputArtifacts": [{"name": "BuildOutput"}],
        "configuration": {
          "ApplicationName": "my-production-app",
          "DeploymentGroupName": "production-blue-green"
        }
      }]
    }
  ]
}'
```


---

## 9. Rollback Configuration

### Step 9.1: Automatic Rollback Setup

```bash
# Update deployment group with auto-rollback
aws deploy update-deployment-group \
  --application-name my-production-app \
  --current-deployment-group-name production-fleet \
  --auto-rollback-configuration '{
    "enabled": true,
    "events": [
      "DEPLOYMENT_FAILURE",
      "DEPLOYMENT_STOP_ON_ALARM",
      "DEPLOYMENT_STOP_ON_REQUEST"
    ]
  }'
```

### Step 9.2: CloudWatch Alarm-Based Rollback

```bash
# Create alarm that triggers rollback
aws cloudwatch put-metric-alarm \
  --alarm-name "HighErrorRate-Production" \
  --metric-name "HTTPCode_Target_5XX_Count" \
  --namespace "AWS/ApplicationELB" \
  --statistic Sum \
  --period 60 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=TargetGroup,Value=targetgroup/production-api-tg/xxx \
               Name=LoadBalancer,Value=app/production-api-alb/yyy

# Create latency alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "HighLatency-Production" \
  --metric-name "TargetResponseTime" \
  --namespace "AWS/ApplicationELB" \
  --statistic p99 \
  --period 60 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=TargetGroup,Value=targetgroup/production-api-tg/xxx \
               Name=LoadBalancer,Value=app/production-api-alb/yyy
```

### Step 9.3: Manual Rollback

```bash
# Stop current deployment
aws deploy stop-deployment \
  --deployment-id d-XXXXXXXXX \
  --auto-rollback-enabled

# Or redeploy previous revision
# List previous deployments
aws deploy list-deployments \
  --application-name my-production-app \
  --deployment-group-name production-fleet \
  --include-only-statuses Succeeded

# Redeploy last successful revision
LAST_SUCCESSFUL=$(aws deploy get-deployment --deployment-id d-LASTGOOD \
  --query 'deploymentInfo.revision')

aws deploy create-deployment \
  --application-name my-production-app \
  --deployment-group-name production-fleet \
  --revision "$LAST_SUCCESSFUL" \
  --description "Rollback to previous version"
```

---

## 10. Production Best Practices

### Step 10.1: Canary Deployment Configuration

```bash
# Create canary deployment config (10% first, then remaining)
aws deploy create-deployment-config \
  --deployment-config-name Canary-10Percent-5Minutes \
  --traffic-routing-config '{
    "type": "TimeBasedCanary",
    "timeBasedCanary": {
      "canaryPercentage": 10,
      "canaryInterval": 5
    }
  }' \
  --compute-platform Server
```

### Step 10.2: Linear Deployment Configuration

```bash
# Linear: 25% every 5 minutes
aws deploy create-deployment-config \
  --deployment-config-name Linear-25Percent-5Minutes \
  --traffic-routing-config '{
    "type": "TimeBasedLinear",
    "timeBasedLinear": {
      "linearPercentage": 25,
      "linearInterval": 5
    }
  }' \
  --compute-platform Server
```

### Step 10.3: Deployment Notifications (SNS + Slack)

```bash
# Create SNS topic
aws sns create-topic --name deployment-notifications

# Subscribe Lambda for Slack integration
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:deployment-notifications \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:123456789012:function:slack-notifier
```

**Lambda Slack Notifier:**
```python
import json
import urllib3

SLACK_WEBHOOK = "https://hooks.slack.com/services/xxx/yyy/zzz"

def lambda_handler(event, context):
    message = json.loads(event['Records'][0]['Sns']['Message'])
    
    deployment_id = message.get('deploymentId', 'Unknown')
    status = message.get('status', 'Unknown')
    app_name = message.get('applicationName', 'Unknown')
    
    color = "#36a64f" if status == "SUCCEEDED" else "#ff0000"
    
    slack_message = {
        "attachments": [{
            "color": color,
            "title": f"CodeDeploy: {app_name}",
            "fields": [
                {"title": "Deployment ID", "value": deployment_id, "short": True},
                {"title": "Status", "value": status, "short": True}
            ]
        }]
    }
    
    http = urllib3.PoolManager()
    http.request('POST', SLACK_WEBHOOK, 
                 body=json.dumps(slack_message),
                 headers={'Content-Type': 'application/json'})
```

### Step 10.4: Troubleshooting Checklist

```bash
# 1. Check CodeDeploy agent status
sudo systemctl status codedeploy-agent

# 2. Check agent logs
tail -100 /var/log/aws/codedeploy-agent/codedeploy-agent.log

# 3. Check deployment logs
ls /opt/codedeploy-agent/deployment-root/
cat /opt/codedeploy-agent/deployment-root/<deployment-group>/<deployment-id>/logs/scripts.log

# 4. Check instance can reach CodeDeploy endpoint
curl -I https://codedeploy.us-east-1.amazonaws.com

# 5. Check IAM permissions
aws sts get-caller-identity
aws s3 ls s3://my-codedeploy-artifacts/

# 6. Restart agent if needed
sudo systemctl restart codedeploy-agent
```

### Step 10.5: Complete Project Structure

```
my-flask-app/
├── appspec.yml                  # CodeDeploy configuration
├── buildspec.yml                # CodeBuild configuration
├── scripts/
│   ├── stop_application.sh      # ApplicationStop hook
│   ├── before_install.sh        # BeforeInstall hook
│   ├── after_install.sh         # AfterInstall hook
│   ├── start_application.sh     # ApplicationStart hook
│   └── validate_service.sh      # ValidateService hook
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── health.py
│   └── ...
├── requirements.txt
├── requirements-dev.txt
├── Dockerfile
├── .env.example
└── README.md
```

---

## Quick Reference: Common Commands

```bash
# List applications
aws deploy list-applications

# List deployment groups
aws deploy list-deployment-groups --application-name my-production-app

# Get deployment status
aws deploy get-deployment --deployment-id d-XXXXXXXXX

# List recent deployments
aws deploy list-deployments --application-name my-production-app \
  --deployment-group-name production-fleet --max-items 5

# Stop a deployment
aws deploy stop-deployment --deployment-id d-XXXXXXXXX

# Delete old revisions
aws deploy list-application-revisions --application-name my-production-app \
  --sort-by registerTime --sort-order descending

# Check pipeline status
aws codepipeline get-pipeline-state --name my-production-pipeline

# Retry failed stage
aws codepipeline retry-stage-execution \
  --pipeline-name my-production-pipeline \
  --stage-name Deploy-Production \
  --pipeline-execution-id xxx-yyy-zzz \
  --retry-mode FAILED_ACTIONS
```

---
