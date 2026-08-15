# AWS Security (IAM, WAF, Shield) - Step-by-Step Configuration Guide

## Table of Contents
1. [IAM Roles & Policies](#1-iam-roles--policies)
2. [AWS WAF Configuration](#2-aws-waf-configuration)
3. [AWS Shield & DDoS Protection](#3-aws-shield--ddos-protection)
4. [Secrets Manager & Parameter Store](#4-secrets-manager--parameter-store)
5. [KMS Encryption](#5-kms-encryption)
6. [Security Monitoring (GuardDuty, Config)](#6-security-monitoring)
7. [VPC Security](#7-vpc-security)
8. [Certificate Manager (ACM)](#8-certificate-manager)
9. [Compliance & Auditing](#9-compliance--auditing)
10. [Incident Response Automation](#10-incident-response-automation)

---

## 1. IAM Roles & Policies

### Step 1.1: Create Service Role (Least Privilege)


```bash
# Create application service role
aws iam create-role \
  --role-name AppServiceRole \
  --assume-role-policy-document '{
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole",
      "Condition": {"StringEquals": {"aws:RequestedRegion": "us-east-1"}}
    }]
  }'

# Create least-privilege policy
aws iam create-policy \
  --policy-name AppServicePolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "S3Access",
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject"],
        "Resource": "arn:aws:s3:::myapp-production-data/app-data/*"
      },
      {
        "Sid": "SecretsAccess",
        "Effect": "Allow",
        "Action": ["secretsmanager:GetSecretValue"],
        "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:production/*"
      },
      {
        "Sid": "SQSAccess",
        "Effect": "Allow",
        "Action": ["sqs:SendMessage", "sqs:ReceiveMessage", "sqs:DeleteMessage"],
        "Resource": "arn:aws:sqs:us-east-1:123456789012:production-*"
      },
      {
        "Sid": "CloudWatchLogs",
        "Effect": "Allow",
        "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
        "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/production/*"
      },
      {
        "Sid": "CloudWatchMetrics",
        "Effect": "Allow",
        "Action": ["cloudwatch:PutMetricData"],
        "Resource": "*",
        "Condition": {"StringEquals": {"cloudwatch:namespace": "Production/MyApp"}}
      }
    ]
  }'

# Attach policy to role
aws iam attach-role-policy \
  --role-name AppServiceRole \
  --policy-arn arn:aws:iam::123456789012:policy/AppServicePolicy
```

### Step 1.2: Service Control Policies (SCPs)

```bash
# Deny actions outside allowed regions
aws organizations create-policy \
  --name "RestrictRegions" \
  --type SERVICE_CONTROL_POLICY \
  --description "Restrict to approved AWS regions" \
  --content '{
    "Statement": [{
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
        },
        "ArnNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/OrganizationAdmin"
        }
      }
    }]
  }'
```

### Step 1.3: IAM Access Analyzer

```bash
# Create Access Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name production-analyzer \
  --type ACCOUNT \
  --tags Environment=production

# List findings (external access)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/production-analyzer \
  --filter '{"status": {"eq": ["ACTIVE"]}}'
```

---

## 2. AWS WAF Configuration

### Step 2.1: Create Web ACL

```bash
aws wafv2 create-web-acl \
  --name production-web-acl \
  --scope REGIONAL \
  --default-action '{"Allow": {}}' \
  --visibility-config '{
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "ProductionWebACL"
  }' \
  --rules '[
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "OverrideAction": {"None": {}},
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CommonRules"
      }
    },
    {
      "Name": "AWSManagedRulesSQLiRuleSet",
      "Priority": 2,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesSQLiRuleSet"
        }
      },
      "OverrideAction": {"None": {}},
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "SQLiRules"
      }
    },
    {
      "Name": "RateLimitRule",
      "Priority": 3,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "RateLimit"
      }
    },
    {
      "Name": "GeoBlockRule",
      "Priority": 4,
      "Statement": {
        "GeoMatchStatement": {
          "CountryCodes": ["CN", "RU", "KP"]
        }
      },
      "Action": {"Block": {}},
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "GeoBlock"
      }
    }
  ]' \
  --region us-east-1
```

### Step 2.2: Associate WAF with ALB

```bash
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456789012:regional/webacl/production-web-acl/xxx \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/production-api-alb/xxx
```

### Step 2.3: Custom WAF Rules

```bash
# Block specific user agents (bots)
aws wafv2 create-regex-pattern-set \
  --name "BlockedBots" \
  --scope REGIONAL \
  --regular-expression-list '[{"RegexString": "(?i)(scrapy|bot|crawler|spider)"}]'

# IP Blacklist
aws wafv2 create-ip-set \
  --name "BlockedIPs" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses '["1.2.3.4/32", "5.6.7.0/24"]'
```

---

## 3. AWS Shield & DDoS Protection

### Step 3.1: Enable Shield Advanced

```bash
# Subscribe to Shield Advanced
aws shield create-subscription

# Create protection for ALB
aws shield create-protection \
  --name "ProductionALB-Protection" \
  --resource-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/production-api-alb/xxx

# Create protection for CloudFront
aws shield create-protection \
  --name "CloudFront-Protection" \
  --resource-arn arn:aws:cloudfront::123456789012:distribution/E1234567890

# Enable proactive engagement
aws shield enable-proactive-engagement

# Set DRT access role
aws shield associate-drt-role \
  --role-arn arn:aws:iam::123456789012:role/AWSDDoSResponseTeamRole
```

---

## 4. Secrets Manager & Parameter Store

### Step 4.1: Store and Rotate Secrets

```bash
# Create secret with auto-rotation
aws secretsmanager create-secret \
  --name production/database/credentials \
  --description "Production database credentials" \
  --secret-string '{"username":"app_user","password":"InitialP@ss123!","host":"cluster.rds.amazonaws.com","port":"5432","dbname":"myapp"}' \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx

# Enable rotation (every 30 days)
aws secretsmanager rotate-secret \
  --secret-id production/database/credentials \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsRotationFunction \
  --rotation-rules AutomaticallyAfterDays=30
```

### Step 4.2: Parameter Store (Configuration)

```bash
# Store configuration parameters
aws ssm put-parameter \
  --name "/production/myapp/api_url" \
  --type String \
  --value "https://api.myapp.com/v1"

aws ssm put-parameter \
  --name "/production/myapp/redis_host" \
  --type SecureString \
  --value "production-redis.xxxxx.cache.amazonaws.com" \
  --key-id arn:aws:kms:us-east-1:123456789012:key/xxx

# Get parameter in application
aws ssm get-parameter --name "/production/myapp/redis_host" --with-decryption
```

---

## 5. KMS Encryption

### Step 5.1: Create KMS Key

```bash
aws kms create-key \
  --description "Production data encryption key" \
  --key-usage ENCRYPT_DECRYPT \
  --key-spec SYMMETRIC_DEFAULT \
  --policy '{
    "Statement": [
      {
        "Sid": "RootAccess",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
        "Action": "kms:*",
        "Resource": "*"
      },
      {
        "Sid": "AppServiceAccess",
        "Effect": "Allow",
        "Principal": {"AWS": "arn:aws:iam::123456789012:role/AppServiceRole"},
        "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
        "Resource": "*"
      }
    ]
  }'

# Create alias
aws kms create-alias \
  --alias-name alias/production-data-key \
  --target-key-id arn:aws:kms:us-east-1:123456789012:key/xxx

# Enable automatic key rotation
aws kms enable-key-rotation --key-id arn:aws:kms:us-east-1:123456789012:key/xxx
```

---

## 6. Security Monitoring

### Step 6.1: Enable GuardDuty

```bash
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --data-sources '{
    "S3Logs": {"Enable": true},
    "Kubernetes": {"AuditLogs": {"Enable": true}},
    "MalwareProtection": {"ScanEc2InstanceWithFindings": {"EbsVolumes": true}}
  }'

# Create filter for high-severity findings
aws guardduty create-filter \
  --detector-id xxx \
  --name "HighSeverityFindings" \
  --finding-criteria '{
    "Criterion": {"severity": {"Gte": 7}}
  }' \
  --action ARCHIVE
```

### Step 6.2: AWS Config Rules

```bash
# Enable Config
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::123456789012:role/ConfigRole \
  --recording-group allSupported=true,includeGlobalResourceTypes=true

# Add compliance rules
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "encrypted-volumes",
  "Source": {"Owner": "AWS", "SourceIdentifier": "ENCRYPTED_VOLUMES"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "restricted-ssh",
  "Source": {"Owner": "AWS", "SourceIdentifier": "INCOMING_SSH_DISABLED"}
}'

aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "s3-bucket-ssl-requests-only",
  "Source": {"Owner": "AWS", "SourceIdentifier": "S3_BUCKET_SSL_REQUESTS_ONLY"}
}'
```

### Step 6.3: CloudTrail

```bash
aws cloudtrail create-trail \
  --name production-audit-trail \
  --s3-bucket-name myapp-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx \
  --include-global-service-events

aws cloudtrail start-logging --name production-audit-trail
```

---

## 7. VPC Security

### Step 7.1: VPC Flow Logs

```bash
aws ec2 create-flow-log \
  --resource-type VPC \
  --resource-ids vpc-0123456789abcdef0 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /production/vpc-flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/VPCFlowLogsRole \
  --max-aggregation-interval 60
```

### Step 7.2: Network ACLs (Extra Layer)

```bash
# Block known malicious CIDRs at NACL level
aws ec2 create-network-acl-entry \
  --network-acl-id acl-xxx \
  --rule-number 50 \
  --protocol -1 \
  --rule-action deny \
  --cidr-block "1.2.3.0/24" \
  --ingress
```

---

## 8. Certificate Manager

### Step 8.1: Request SSL Certificate

```bash
# Request certificate (DNS validation)
aws acm request-certificate \
  --domain-name "myapp.com" \
  --subject-alternative-names "*.myapp.com" "api.myapp.com" \
  --validation-method DNS \
  --tags Key=Environment,Value=production

# Get DNS validation records
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/xxx \
  --query 'Certificate.DomainValidationOptions[].ResourceRecord'

# Add validation CNAME to Route 53, then verify:
aws acm wait certificate-validated \
  --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/xxx
```

---

## 9. Compliance & Auditing

### Step 9.1: Security Hub

```bash
# Enable Security Hub
aws securityhub enable-security-hub --enable-default-standards

# Get compliance score
aws securityhub get-findings \
  --filters '{"ComplianceStatus": [{"Value": "FAILED", "Comparison": "EQUALS"}]}' \
  --max-items 20
```

---

## 10. Incident Response Automation

### Step 10.1: Auto-Remediate Compromised Key

```python
import boto3

def handle_compromised_key(event, context):
    """Lambda triggered by GuardDuty finding."""
    iam = boto3.client('iam')
    finding = event['detail']

    access_key_id = finding['resource']['accessKeyDetails']['accessKeyId']
    username = finding['resource']['accessKeyDetails']['userName']

    # 1. Disable the key
    iam.update_access_key(
        UserName=username,
        AccessKeyId=access_key_id,
        Status='Inactive'
    )

    # 2. Attach deny-all policy
    iam.attach_user_policy(
        UserName=username,
        PolicyArn='arn:aws:iam::123456789012:policy/DenyAll'
    )

    # 3. Alert security team
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:security-incidents',
        Subject=f'CRITICAL: Compromised key detected - {username}',
        Message=f'Access key {access_key_id} has been disabled. Immediate investigation required.'
    )
```

### Quick Reference Commands

```bash
# Check IAM credential report
aws iam generate-credential-report
aws iam get-credential-report --output text --query Content | base64 --decode

# List unused access keys (>90 days)
aws iam list-access-keys --user-name xxx
aws iam get-access-key-last-used --access-key-id AKIAXXXXXXXX

# Check Security Hub findings
aws securityhub get-findings --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}'

# List WAF blocked requests
aws wafv2 get-sampled-requests \
  --web-acl-arn arn:aws:wafv2:...:webacl/production-web-acl/xxx \
  --rule-metric-name RateLimit --scope REGIONAL \
  --time-window StartTime=$(date -d '1 hour ago' +%s),EndTime=$(date +%s) \
  --max-items 100
```

---
