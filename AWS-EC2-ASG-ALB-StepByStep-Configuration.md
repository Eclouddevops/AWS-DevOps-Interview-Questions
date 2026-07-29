# AWS EC2, Auto Scaling Groups & Application Load Balancer - Step-by-Step Configuration Guide

## Table of Contents
1. [EC2 Instance Setup](#1-ec2-instance-setup)
2. [Security Groups Configuration](#2-security-groups-configuration)
3. [Launch Template Creation](#3-launch-template-creation)
4. [Target Group Configuration](#4-target-group-configuration)
5. [Application Load Balancer Setup](#5-application-load-balancer-setup)
6. [Auto Scaling Group Configuration](#6-auto-scaling-group-configuration)
7. [Scaling Policies](#7-scaling-policies)
8. [Production Best Practices](#8-production-best-practices)

---

## 1. EC2 Instance Setup

### Step 1.1: Choose AMI (Amazon Machine Image)


**AWS Console:**
```
EC2 Dashboard → Launch Instance → Step 1: Choose AMI
├── Amazon Linux 2023 (Recommended for production)
├── Ubuntu Server 22.04 LTS
├── Red Hat Enterprise Linux 9
└── Custom AMI (Pre-baked with application)
```

**AWS CLI:**
```bash
# List available AMIs (Amazon Linux 2023)
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-*" \
            "Name=architecture,Values=x86_64" \
            "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text

# Result: ami-0abcdef1234567890
```

### Step 1.2: Choose Instance Type

| Use Case | Instance Type | vCPUs | Memory | Network |
|---|---|---|---|---|
| Web/API Server | t3.medium | 2 | 4 GB | Up to 5 Gbps |
| Application Server | m6i.xlarge | 4 | 16 GB | Up to 12.5 Gbps |
| Compute Intensive | c6i.2xlarge | 8 | 16 GB | Up to 12.5 Gbps |
| Memory Intensive | r6i.2xlarge | 8 | 64 GB | Up to 12.5 Gbps |
| Cost-Optimized | t3a.large | 2 | 8 GB | Up to 5 Gbps |

**AWS CLI:**
```bash
# Launch EC2 instance
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.large \
  --key-name my-key-pair \
  --subnet-id subnet-0123456789abcdef0 \
  --security-group-ids sg-0123456789abcdef0 \
  --iam-instance-profile Name=EC2-Production-Role \
  --block-device-mappings '[{
    "DeviceName": "/dev/xvda",
    "Ebs": {
      "VolumeSize": 50,
      "VolumeType": "gp3",
      "Iops": 3000,
      "Throughput": 125,
      "Encrypted": true,
      "DeleteOnTermination": true
    }
  }]' \
  --tag-specifications 'ResourceType=instance,Tags=[
    {Key=Name,Value=web-server-prod-01},
    {Key=Environment,Value=production},
    {Key=Team,Value=platform},
    {Key=ManagedBy,Value=terraform}
  ]' \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=1,HttpEndpoint=enabled" \
  --user-data file://userdata.sh
```

### Step 1.3: User Data Script (Bootstrap)

```bash
#!/bin/bash
# userdata.sh - Production EC2 Bootstrap Script

set -e

# Update system
yum update -y

# Install CloudWatch Agent
yum install -y amazon-cloudwatch-agent

# Install CodeDeploy Agent
yum install -y ruby wget
cd /home/ec2-user
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x ./install
./install auto
systemctl enable codedeploy-agent
systemctl start codedeploy-agent

# Install application dependencies
yum install -y python3.11 python3.11-pip nginx
pip3.11 install gunicorn flask

# Configure CloudWatch Agent
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "metrics": {
    "namespace": "Production/EC2",
    "metrics_collected": {
      "mem": {"measurement": ["mem_used_percent"]},
      "disk": {"measurement": ["disk_used_percent"], "resources": ["/"]}
    },
    "append_dimensions": {"InstanceId": "${aws:InstanceId}"}
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {"file_path": "/var/log/myapp/*.log", "log_group_name": "/production/myapp", "log_stream_name": "{instance_id}"}
        ]
      }
    }
  }
}
EOF

/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Signal success
echo "Bootstrap completed successfully" > /var/log/bootstrap-status.log
```

### Step 1.4: Create Key Pair

```bash
# Create key pair
aws ec2 create-key-pair \
  --key-name production-key-2024 \
  --key-type ed25519 \
  --query 'KeyMaterial' \
  --output text > production-key-2024.pem

chmod 400 production-key-2024.pem
```

---

## 2. Security Groups Configuration

### Step 2.1: ALB Security Group

```bash
# Create ALB Security Group
aws ec2 create-security-group \
  --group-name alb-production-sg \
  --description "ALB Security Group - Production" \
  --vpc-id vpc-0123456789abcdef0

# Allow inbound HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id sg-alb123456 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Allow inbound HTTP (redirect to HTTPS)
aws ec2 authorize-security-group-ingress \
  --group-id sg-alb123456 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

### Step 2.2: Application Security Group

```bash
# Create Application Security Group
aws ec2 create-security-group \
  --group-name app-production-sg \
  --description "Application Security Group - Production" \
  --vpc-id vpc-0123456789abcdef0

# Allow inbound ONLY from ALB Security Group
aws ec2 authorize-security-group-ingress \
  --group-id sg-app123456 \
  --protocol tcp \
  --port 8080 \
  --source-group sg-alb123456

# Allow SSH only from bastion/VPN (NOT 0.0.0.0/0)
aws ec2 authorize-security-group-ingress \
  --group-id sg-app123456 \
  --protocol tcp \
  --port 22 \
  --source-group sg-bastion123456
```

### Step 2.3: Database Security Group

```bash
# Create Database Security Group
aws ec2 create-security-group \
  --group-name db-production-sg \
  --description "Database Security Group - Production" \
  --vpc-id vpc-0123456789abcdef0

# Allow inbound ONLY from Application Security Group
aws ec2 authorize-security-group-ingress \
  --group-id sg-db123456 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app123456
```

**Security Group Architecture:**
```
Internet → ALB (sg-alb: 80, 443 from 0.0.0.0/0)
              → App (sg-app: 8080 from sg-alb only)
                  → DB (sg-db: 5432 from sg-app only)
```

---

## 3. Launch Template Creation

### Step 3.1: Create Launch Template

**AWS Console Path:**
```
EC2 → Launch Templates → Create Launch Template
```

**AWS CLI:**
```bash
aws ec2 create-launch-template \
  --launch-template-name production-web-lt \
  --version-description "v1.0 - Initial production template" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "t3.large",
    "KeyName": "production-key-2024",
    "SecurityGroupIds": ["sg-app123456"],
    "IamInstanceProfile": {
      "Arn": "arn:aws:iam::123456789012:instance-profile/EC2-Production-Role"
    },
    "BlockDeviceMappings": [{
      "DeviceName": "/dev/xvda",
      "Ebs": {
        "VolumeSize": 50,
        "VolumeType": "gp3",
        "Iops": 3000,
        "Throughput": 125,
        "Encrypted": true,
        "DeleteOnTermination": true
      }
    }],
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpPutResponseHopLimit": 1,
      "HttpEndpoint": "enabled"
    },
    "Monitoring": {"Enabled": true},
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [
        {"Key": "Name", "Value": "web-prod"},
        {"Key": "Environment", "Value": "production"},
        {"Key": "ManagedBy", "Value": "ASG"}
      ]
    }],
    "UserData": "BASE64_ENCODED_USERDATA_SCRIPT"
  }'
```

### Step 3.2: Create New Version (For Updates)

```bash
# Create new version with updated AMI
aws ec2 create-launch-template-version \
  --launch-template-name production-web-lt \
  --version-description "v2.0 - Updated AMI with security patches" \
  --source-version 1 \
  --launch-template-data '{"ImageId": "ami-0newami1234567890"}'

# Set new version as default
aws ec2 modify-launch-template \
  --launch-template-name production-web-lt \
  --default-version 2
```

---

## 4. Target Group Configuration

### Step 4.1: Create Target Group

**AWS Console Path:**
```
EC2 → Target Groups → Create Target Group
```

**AWS CLI:**
```bash
aws elbv2 create-target-group \
  --name production-api-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-0123456789abcdef0 \
  --target-type instance \
  --health-check-enabled \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-port 8080 \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 2 \
  --matcher '{"HttpCode": "200"}' \
  --tags Key=Environment,Value=production Key=Service,Value=api
```

### Step 4.2: Configure Target Group Attributes

```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456:targetgroup/production-api-tg/xxx \
  --attributes \
    Key=deregistration_delay.timeout_seconds,Value=60 \
    Key=slow_start.duration_seconds,Value=90 \
    Key=stickiness.enabled,Value=false \
    Key=load_balancing.algorithm.type,Value=least_outstanding_requests
```

**Attribute Explanation:**
| Attribute | Value | Purpose |
|---|---|---|
| deregistration_delay | 60s | Allow in-flight requests to complete before removing target |
| slow_start | 90s | Gradually increase traffic to new instances |
| stickiness | false | Stateless app, use Redis for sessions |
| algorithm | least_outstanding_requests | Better distribution for varying request durations |

---

## 5. Application Load Balancer Setup

### Step 5.1: Create ALB

**AWS Console Path:**
```
EC2 → Load Balancers → Create Load Balancer → Application Load Balancer
```

**AWS CLI:**
```bash
aws elbv2 create-load-balancer \
  --name production-api-alb \
  --type application \
  --scheme internet-facing \
  --ip-address-type ipv4 \
  --subnets subnet-public-az1 subnet-public-az2 subnet-public-az3 \
  --security-groups sg-alb123456 \
  --tags Key=Environment,Value=production Key=Service,Value=api
```

### Step 5.2: Configure HTTPS Listener (Port 443)

```bash
# Create HTTPS listener with ACM certificate
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456:loadbalancer/app/production-api-alb/xxx \
  --protocol HTTPS \
  --port 443 \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:123456:certificate/xxx-yyy-zzz \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456:targetgroup/production-api-tg/xxx
```

### Step 5.3: Configure HTTP to HTTPS Redirect (Port 80)

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456:loadbalancer/app/production-api-alb/xxx \
  --protocol HTTP \
  --port 80 \
  --default-actions '[{
    "Type": "redirect",
    "RedirectConfig": {
      "Protocol": "HTTPS",
      "Port": "443",
      "StatusCode": "HTTP_301"
    }
  }]'
```

### Step 5.4: Add Path-Based Routing Rules

```bash
# Rule: /api/* → API Target Group
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/xxx \
  --priority 10 \
  --conditions '[{"Field": "path-pattern", "PathPatternConfig": {"Values": ["/api/*"]}}]' \
  --actions '[{"Type": "forward", "TargetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/api-tg/xxx"}]'

# Rule: /admin/* → Admin Target Group (with IP restriction)
aws elbv2 create-rule \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/xxx \
  --priority 20 \
  --conditions '[
    {"Field": "path-pattern", "PathPatternConfig": {"Values": ["/admin/*"]}},
    {"Field": "source-ip", "SourceIpConfig": {"Values": ["10.0.0.0/8", "172.16.0.0/12"]}}
  ]' \
  --actions '[{"Type": "forward", "TargetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/admin-tg/xxx"}]'
```

### Step 5.5: Configure ALB Attributes

```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/production-api-alb/xxx \
  --attributes \
    Key=idle_timeout.timeout_seconds,Value=120 \
    Key=routing.http2.enabled,Value=true \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-alb-access-logs \
    Key=access_logs.s3.prefix,Value=production/api-alb \
    Key=routing.http.drop_invalid_header_fields.enabled,Value=true \
    Key=deletion_protection.enabled,Value=true
```

### Step 5.6: Enable Access Logs

```bash
# Create S3 bucket for ALB logs (must have correct policy)
aws s3api create-bucket --bucket my-alb-access-logs --region us-east-1

# Add bucket policy for ALB to write logs
aws s3api put-bucket-policy --bucket my-alb-access-logs --policy '{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::127311923021:root"},
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-alb-access-logs/production/api-alb/AWSLogs/123456789012/*"
  }]
}'
```

---

## 6. Auto Scaling Group Configuration

### Step 6.1: Create Auto Scaling Group

**AWS Console Path:**
```
EC2 → Auto Scaling Groups → Create Auto Scaling Group
```

**AWS CLI:**
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name production-api-asg \
  --launch-template LaunchTemplateName=production-web-lt,Version='$Latest' \
  --min-size 3 \
  --max-size 20 \
  --desired-capacity 6 \
  --vpc-zone-identifier "subnet-private-az1,subnet-private-az2,subnet-private-az3" \
  --target-group-arns "arn:aws:elasticloadbalancing:...:targetgroup/production-api-tg/xxx" \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --default-cooldown 300 \
  --termination-policies '["OldestLaunchTemplate", "OldestInstance"]' \
  --new-instances-protected-from-scale-in \
  --tags '[
    {"Key": "Name", "Value": "web-prod", "PropagateAtLaunch": true},
    {"Key": "Environment", "Value": "production", "PropagateAtLaunch": true}
  ]'
```

### Step 6.2: Configure Mixed Instances Policy (Cost Optimization)

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name production-workers-asg \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "production-worker-lt",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "c5.2xlarge"},
        {"InstanceType": "c5a.2xlarge"},
        {"InstanceType": "c5n.2xlarge"},
        {"InstanceType": "c6i.2xlarge"},
        {"InstanceType": "m5.2xlarge"}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 30,
      "SpotAllocationStrategy": "capacity-optimized",
      "SpotMaxPrice": ""
    }
  }' \
  --min-size 3 \
  --max-size 30 \
  --desired-capacity 6 \
  --vpc-zone-identifier "subnet-private-az1,subnet-private-az2,subnet-private-az3"
```

### Step 6.3: Configure Instance Refresh (Rolling Updates)

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name production-api-asg \
  --strategy Rolling \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300,
    "MaxHealthyPercentage": 120,
    "SkipMatching": true
  }'

# Check refresh status
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name production-api-asg
```

### Step 6.4: Configure Lifecycle Hooks

```bash
# Hook before instance terminates (for graceful shutdown)
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name graceful-shutdown-hook \
  --auto-scaling-group-name production-api-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
  --heartbeat-timeout 300 \
  --default-result CONTINUE \
  --notification-target-arn arn:aws:sns:us-east-1:123456:asg-lifecycle-events

# Hook after instance launches (for validation)
aws autoscaling put-lifecycle-hook \
  --lifecycle-hook-name launch-validation-hook \
  --auto-scaling-group-name production-api-asg \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
  --heartbeat-timeout 600 \
  --default-result ABANDON
```

---

## 7. Scaling Policies

### Step 7.1: Target Tracking Scaling (Recommended)

```bash
# Scale based on average CPU utilization
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name production-api-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 70.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'

# Scale based on ALB request count per target
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name production-api-asg \
  --policy-name request-count-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/production-api-alb/xxx/targetgroup/production-api-tg/yyy"
    },
    "TargetValue": 1000.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

### Step 7.2: Step Scaling (For Aggressive Scaling)

```bash
# Create CloudWatch Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-ScaleOut" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=AutoScalingGroupName,Value=production-api-asg \
  --alarm-actions arn:aws:autoscaling:us-east-1:123456:scalingPolicy:xxx

# Step Scaling Policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name production-api-asg \
  --policy-name aggressive-scale-out \
  --policy-type StepScaling \
  --adjustment-type PercentChangeInCapacity \
  --step-adjustments '[
    {"MetricIntervalLowerBound": 0, "MetricIntervalUpperBound": 20, "ScalingAdjustment": 25},
    {"MetricIntervalLowerBound": 20, "MetricIntervalUpperBound": 40, "ScalingAdjustment": 50},
    {"MetricIntervalLowerBound": 40, "ScalingAdjustment": 100}
  ]' \
  --min-adjustment-magnitude 1
```

### Step 7.3: Predictive Scaling (For Known Patterns)

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name production-api-asg \
  --policy-name predictive-scaling \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications": [{
      "TargetValue": 70.0,
      "PredefinedMetricPairSpecification": {
        "PredefinedMetricType": "ASGCPUUtilization"
      }
    }],
    "Mode": "ForecastAndScale",
    "SchedulingBufferTime": 300,
    "MaxCapacityBreachBehavior": "IncreaseMaxCapacity",
    "MaxCapacityBuffer": 20
  }'
```

### Step 7.4: Scheduled Scaling (For Predictable Patterns)

```bash
# Scale up for business hours (Mon-Fri 8 AM)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name production-api-asg \
  --scheduled-action-name business-hours-scale-up \
  --recurrence "0 8 * * 1-5" \
  --min-size 6 \
  --max-size 20 \
  --desired-capacity 8

# Scale down for off-hours (Mon-Fri 8 PM)
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name production-api-asg \
  --scheduled-action-name off-hours-scale-down \
  --recurrence "0 20 * * 1-5" \
  --min-size 3 \
  --max-size 10 \
  --desired-capacity 3
```

---

## 8. Production Best Practices

### Step 8.1: Enable Deletion Protection

```bash
# ALB deletion protection
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:...:loadbalancer/app/production-api-alb/xxx \
  --attributes Key=deletion_protection.enabled,Value=true
```

### Step 8.2: Cross-Zone Load Balancing

```bash
# Ensure even distribution across AZs
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/production-api-tg/xxx \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

### Step 8.3: Connection Draining Configuration

```bash
# Set deregistration delay
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/production-api-tg/xxx \
  --attributes Key=deregistration_delay.timeout_seconds,Value=60
```

### Step 8.4: Health Check Best Practices

```python
# Application health endpoint (/health)
@app.route('/health')
def health_check():
    checks = {
        "database": check_database_connection(),
        "redis": check_redis_connection(),
        "disk_space": check_disk_space(),
        "memory": check_memory_usage()
    }
    
    all_healthy = all(checks.values())
    status_code = 200 if all_healthy else 503
    
    return jsonify({
        "status": "healthy" if all_healthy else "unhealthy",
        "checks": checks,
        "timestamp": datetime.utcnow().isoformat()
    }), status_code
```

### Step 8.5: Complete Architecture Diagram

```
                    Internet
                       │
                 Route 53 (DNS)
                       │
                 CloudFront (CDN)
                       │
                   WAF + Shield
                       │
              ┌────── ALB ──────┐
              │  (3 AZs, HTTPS) │
              └────────┬────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
    ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
    │ AZ-1a   │   │ AZ-1b   │   │ AZ-1c   │
    │ EC2 x2  │   │ EC2 x2  │   │ EC2 x2  │
    └────┬────┘   └────┬────┘   └────┬────┘
         │             │             │
         └─────────────┼─────────────┘
                       │
              ┌────────┴────────┐
              │  Aurora/RDS     │
              │  (Multi-AZ)    │
              └─────────────────┘
```

---

## Quick Reference: Common CLI Commands

```bash
# Check ASG status
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-name production-api-asg

# Check instance health in target group
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/xxx

# Check ALB status
aws elbv2 describe-load-balancers --names production-api-alb

# View scaling activities
aws autoscaling describe-scaling-activities --auto-scaling-group-name production-api-asg --max-items 10

# Manually set desired capacity (emergency)
aws autoscaling set-desired-capacity --auto-scaling-group-name production-api-asg --desired-capacity 10

# Detach instance for debugging
aws autoscaling detach-instances --auto-scaling-group-name production-api-asg \
  --instance-ids i-1234567890abcdef0 --should-decrement-desired-capacity
```

---
