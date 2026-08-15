# AWS S3 & Storage Management - Step-by-Step Configuration Guide

## Table of Contents
1. [Bucket Creation & Configuration](#1-bucket-creation--configuration)
2. [Security & Access Control](#2-security--access-control)
3. [Lifecycle Policies](#3-lifecycle-policies)
4. [Versioning & Replication](#4-versioning--replication)
5. [Performance Optimization](#5-performance-optimization)
6. [Encryption Configuration](#6-encryption-configuration)
7. [Event Notifications](#7-event-notifications)
8. [Monitoring & Logging](#8-monitoring--logging)
9. [Cost Optimization](#9-cost-optimization)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. Bucket Creation & Configuration

### Step 1.1: Create Production Bucket


**AWS Console Path:**
```
S3 → Create Bucket → Configure settings
```

**AWS CLI:**
```bash
# Create bucket (us-east-1 doesn't need LocationConstraint)
aws s3api create-bucket \
  --bucket myapp-production-data \
  --region us-east-1

# Create bucket in other regions
aws s3api create-bucket \
  --bucket myapp-production-data-eu \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1

# Add tags
aws s3api put-bucket-tagging \
  --bucket myapp-production-data \
  --tagging '{
    "TagSet": [
      {"Key": "Environment", "Value": "production"},
      {"Key": "Team", "Value": "platform"},
      {"Key": "CostCenter", "Value": "engineering"},
      {"Key": "DataClassification", "Value": "confidential"}
    ]
  }'
```

### Step 1.2: Block Public Access (CRITICAL)

```bash
# Block ALL public access (default for new buckets, but verify)
aws s3api put-public-access-block \
  --bucket myapp-production-data \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'

# Verify
aws s3api get-public-access-block --bucket myapp-production-data
```

### Step 1.3: Enable Versioning

```bash
aws s3api put-bucket-versioning \
  --bucket myapp-production-data \
  --versioning-configuration Status=Enabled

# Check versioning status
aws s3api get-bucket-versioning --bucket myapp-production-data
```

---

## 2. Security & Access Control

### Step 2.1: Bucket Policy (Deny Non-SSL)

```bash
aws s3api put-bucket-policy \
  --bucket myapp-production-data \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyNonSSLRequests",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": [
          "arn:aws:s3:::myapp-production-data",
          "arn:aws:s3:::myapp-production-data/*"
        ],
        "Condition": {
          "Bool": {"aws:SecureTransport": "false"}
        }
      },
      {
        "Sid": "DenyIncorrectEncryptionHeader",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::myapp-production-data/*",
        "Condition": {
          "StringNotEquals": {"s3:x-amz-server-side-encryption": "aws:kms"}
        }
      },
      {
        "Sid": "AllowOnlyFromVPC",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": [
          "arn:aws:s3:::myapp-production-data",
          "arn:aws:s3:::myapp-production-data/*"
        ],
        "Condition": {
          "StringNotEquals": {"aws:sourceVpc": "vpc-0123456789abcdef0"}
        }
      }
    ]
  }'
```

### Step 2.2: Cross-Account Access

```bash
# Allow another AWS account read access
aws s3api put-bucket-policy \
  --bucket myapp-shared-data \
  --policy '{
    "Statement": [{
      "Sid": "CrossAccountRead",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::987654321098:role/DataAnalyticsRole"},
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::myapp-shared-data",
        "arn:aws:s3:::myapp-shared-data/analytics/*"
      ],
      "Condition": {
        "StringEquals": {"aws:PrincipalOrgID": "o-xxxxxxxxxxxxx"}
      }
    }]
  }'
```

### Step 2.3: S3 Access Points

```bash
# Create access point for specific team
aws s3control create-access-point \
  --account-id 123456789012 \
  --name data-team-access \
  --bucket myapp-production-data \
  --vpc-configuration VpcId=vpc-0123456789abcdef0

# Set access point policy
aws s3control put-access-point-policy \
  --account-id 123456789012 \
  --name data-team-access \
  --policy '{
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/DataTeamRole"},
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:us-east-1:123456789012:accesspoint/data-team-access/object/data-team/*"
    }]
  }'
```

### Step 2.4: Presigned URLs (Temporary Access)

```python
import boto3
from datetime import datetime

s3_client = boto3.client('s3', region_name='us-east-1')

# Generate presigned URL for upload (PUT)
upload_url = s3_client.generate_presigned_url(
    'put_object',
    Params={
        'Bucket': 'myapp-production-data',
        'Key': f'uploads/{datetime.now().strftime("%Y/%m/%d")}/file.jpg',
        'ContentType': 'image/jpeg',
        'ServerSideEncryption': 'aws:kms'
    },
    ExpiresIn=3600  # 1 hour
)

# Generate presigned URL for download (GET)
download_url = s3_client.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'myapp-production-data',
        'Key': 'reports/monthly-report-2024-01.pdf'
    },
    ExpiresIn=86400  # 24 hours
)
```

---

## 3. Lifecycle Policies

### Step 3.1: Complete Lifecycle Configuration

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket myapp-production-data \
  --lifecycle-configuration '{
    "Rules": [
      {
        "ID": "TransitionUploads",
        "Status": "Enabled",
        "Filter": {"Prefix": "uploads/"},
        "Transitions": [
          {"Days": 30, "StorageClass": "STANDARD_IA"},
          {"Days": 90, "StorageClass": "GLACIER_IR"},
          {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
        ],
        "NoncurrentVersionTransitions": [
          {"NoncurrentDays": 7, "StorageClass": "STANDARD_IA"},
          {"NoncurrentDays": 30, "StorageClass": "GLACIER"}
        ],
        "NoncurrentVersionExpiration": {"NoncurrentDays": 90}
      },
      {
        "ID": "ExpireTempFiles",
        "Status": "Enabled",
        "Filter": {"Prefix": "tmp/"},
        "Expiration": {"Days": 7}
      },
      {
        "ID": "CleanupIncompleteMultipart",
        "Status": "Enabled",
        "Filter": {"Prefix": ""},
        "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
      },
      {
        "ID": "LogsRetention",
        "Status": "Enabled",
        "Filter": {"Prefix": "logs/"},
        "Transitions": [
          {"Days": 7, "StorageClass": "STANDARD_IA"},
          {"Days": 30, "StorageClass": "GLACIER"}
        ],
        "Expiration": {"Days": 365}
      },
      {
        "ID": "IntelligentTieringForUnknown",
        "Status": "Enabled",
        "Filter": {"Prefix": "user-content/"},
        "Transitions": [
          {"Days": 0, "StorageClass": "INTELLIGENT_TIERING"}
        ]
      }
    ]
  }'
```

### Step 3.2: S3 Intelligent-Tiering Configuration

```bash
# Configure Intelligent-Tiering archive access tiers
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket myapp-production-data \
  --id "archive-config" \
  --intelligent-tiering-configuration '{
    "Id": "archive-config",
    "Status": "Enabled",
    "Filter": {"Prefix": "user-content/"},
    "Tierings": [
      {"AccessTier": "ARCHIVE_ACCESS", "Days": 90},
      {"AccessTier": "DEEP_ARCHIVE_ACCESS", "Days": 180}
    ]
  }'
```

---

## 4. Versioning & Replication

### Step 4.1: Cross-Region Replication (CRR)

```bash
# Step 1: Create destination bucket with versioning
aws s3api create-bucket \
  --bucket myapp-production-data-replica \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1

aws s3api put-bucket-versioning \
  --bucket myapp-production-data-replica \
  --versioning-configuration Status=Enabled

# Step 2: Create replication IAM role
aws iam create-role \
  --role-name S3-CRR-Role \
  --assume-role-policy-document '{
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "s3.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Step 3: Attach replication policy
aws iam put-role-policy \
  --role-name S3-CRR-Role \
  --policy-name S3ReplicationPolicy \
  --policy-document '{
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:GetReplicationConfiguration", "s3:ListBucket"],
        "Resource": "arn:aws:s3:::myapp-production-data"
      },
      {
        "Effect": "Allow",
        "Action": ["s3:GetObjectVersionForReplication", "s3:GetObjectVersionAcl", "s3:GetObjectVersionTagging"],
        "Resource": "arn:aws:s3:::myapp-production-data/*"
      },
      {
        "Effect": "Allow",
        "Action": ["s3:ReplicateObject", "s3:ReplicateDelete", "s3:ReplicateTags"],
        "Resource": "arn:aws:s3:::myapp-production-data-replica/*"
      }
    ]
  }'

# Step 4: Enable replication
aws s3api put-bucket-replication \
  --bucket myapp-production-data \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789012:role/S3-CRR-Role",
    "Rules": [{
      "ID": "ReplicateAll",
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {"Prefix": ""},
      "Destination": {
        "Bucket": "arn:aws:s3:::myapp-production-data-replica",
        "StorageClass": "STANDARD_IA",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {"Minutes": 15}
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {"Minutes": 15}
        }
      },
      "DeleteMarkerReplication": {"Status": "Enabled"}
    }]
  }'
```

---

## 5. Performance Optimization

### Step 5.1: Transfer Acceleration

```bash
# Enable transfer acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket myapp-production-data \
  --accelerate-configuration Status=Enabled

# Use accelerated endpoint for uploads
aws s3 cp large-file.zip s3://myapp-production-data/uploads/ \
  --endpoint-url https://myapp-production-data.s3-accelerate.amazonaws.com
```

### Step 5.2: Multipart Upload Configuration

```python
import boto3
from boto3.s3.transfer import TransferConfig

s3 = boto3.client('s3')

# Configure multipart upload settings
config = TransferConfig(
    multipart_threshold=100 * 1024 * 1024,  # 100 MB threshold
    max_concurrency=10,                       # 10 concurrent parts
    multipart_chunksize=100 * 1024 * 1024,   # 100 MB per part
    use_threads=True
)

# Upload large file with multipart
s3.upload_file(
    'large-backup.tar.gz',
    'myapp-production-data',
    'backups/large-backup.tar.gz',
    Config=config,
    ExtraArgs={
        'ServerSideEncryption': 'aws:kms',
        'StorageClass': 'STANDARD_IA'
    }
)
```

### Step 5.3: S3 Batch Operations

```bash
# Create inventory configuration (needed for batch operations)
aws s3api put-bucket-inventory-configuration \
  --bucket myapp-production-data \
  --id "weekly-inventory" \
  --inventory-configuration '{
    "Id": "weekly-inventory",
    "IsEnabled": true,
    "Destination": {
      "S3BucketDestination": {
        "Bucket": "arn:aws:s3:::myapp-inventory-reports",
        "Format": "CSV",
        "Prefix": "inventory/"
      }
    },
    "Schedule": {"Frequency": "Weekly"},
    "IncludedObjectVersions": "Current",
    "OptionalFields": ["Size", "LastModifiedDate", "StorageClass", "ETag"]
  }'
```

---

## 6. Encryption Configuration

### Step 6.1: Default Encryption (SSE-KMS)

```bash
# Set default encryption with KMS
aws s3api put-bucket-encryption \
  --bucket myapp-production-data \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/xxx-yyy-zzz"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

### Step 6.2: S3 Object Lock (Compliance/WORM)

```bash
# Enable object lock (must be done at bucket creation)
aws s3api create-bucket \
  --bucket myapp-compliance-data \
  --object-lock-enabled-for-bucket

# Set default retention
aws s3api put-object-lock-configuration \
  --bucket myapp-compliance-data \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Years": 7
      }
    }
  }'
```

---

## 7. Event Notifications

### Step 7.1: S3 Event to Lambda

```bash
aws s3api put-bucket-notification-configuration \
  --bucket myapp-production-data \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [
      {
        "Id": "ProcessNewUploads",
        "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:process-upload",
        "Events": ["s3:ObjectCreated:*"],
        "Filter": {
          "Key": {"FilterRules": [{"Name": "prefix", "Value": "uploads/"}, {"Name": "suffix", "Value": ".jpg"}]}
        }
      }
    ],
    "QueueConfigurations": [
      {
        "Id": "LargeFileProcessing",
        "QueueArn": "arn:aws:sqs:us-east-1:123456789012:large-file-queue",
        "Events": ["s3:ObjectCreated:CompleteMultipartUpload"],
        "Filter": {
          "Key": {"FilterRules": [{"Name": "prefix", "Value": "large-files/"}]}
        }
      }
    ],
    "TopicConfigurations": [
      {
        "Id": "DeletionAlert",
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:s3-deletion-alerts",
        "Events": ["s3:ObjectRemoved:*"]
      }
    ]
  }'
```

---

## 8. Monitoring & Logging

### Step 8.1: Server Access Logging

```bash
# Create logging bucket
aws s3api create-bucket --bucket myapp-s3-access-logs

# Grant S3 log delivery permission
aws s3api put-bucket-acl \
  --bucket myapp-s3-access-logs \
  --grant-write URI=http://acs.amazonaws.com/groups/s3/LogDelivery \
  --grant-read-acp URI=http://acs.amazonaws.com/groups/s3/LogDelivery

# Enable access logging
aws s3api put-bucket-logging \
  --bucket myapp-production-data \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "myapp-s3-access-logs",
      "TargetPrefix": "production-data-logs/"
    }
  }'
```

### Step 8.2: S3 Request Metrics (CloudWatch)

```bash
# Enable request metrics for entire bucket
aws s3api put-bucket-metrics-configuration \
  --bucket myapp-production-data \
  --id "EntireBucket" \
  --metrics-configuration '{"Id": "EntireBucket"}'

# Metrics for specific prefix
aws s3api put-bucket-metrics-configuration \
  --bucket myapp-production-data \
  --id "UploadsPrefix" \
  --metrics-configuration '{
    "Id": "UploadsPrefix",
    "Filter": {"Prefix": "uploads/"}
  }'

# Create alarms on S3 metrics
aws cloudwatch put-metric-alarm \
  --alarm-name "S3-4xxErrors-High" \
  --metric-name 4xxErrors \
  --namespace AWS/S3 \
  --statistic Sum \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=BucketName,Value=myapp-production-data \
               Name=FilterId,Value=EntireBucket \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings
```

---

## 9. Cost Optimization

### Step 9.1: Storage Class Analysis

```bash
# Enable storage class analysis
aws s3api put-bucket-analytics-configuration \
  --bucket myapp-production-data \
  --id "StorageAnalysis" \
  --analytics-configuration '{
    "Id": "StorageAnalysis",
    "Filter": {"Prefix": "uploads/"},
    "StorageClassAnalysis": {
      "DataExport": {
        "OutputSchemaVersion": "V_1",
        "Destination": {
          "S3BucketDestination": {
            "Bucket": "arn:aws:s3:::myapp-analytics-reports",
            "Prefix": "storage-analysis/",
            "Format": "CSV"
          }
        }
      }
    }
  }'
```

### Step 9.2: Cost Monitoring Commands

```bash
# Check bucket size and object count
aws s3api list-objects-v2 --bucket myapp-production-data \
  --query 'length(Contents)' --output text

# Get bucket metrics from CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name BucketSizeBytes \
  --dimensions Name=BucketName,Value=myapp-production-data \
               Name=StorageType,Value=StandardStorage \
  --start-time $(date -u -d '1 day ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Average

# List incomplete multipart uploads (hidden costs!)
aws s3api list-multipart-uploads --bucket myapp-production-data
```

---

## 10. Production Best Practices

### Step 10.1: CORS Configuration (For Web Apps)

```bash
aws s3api put-bucket-cors \
  --bucket myapp-production-data \
  --cors-configuration '{
    "CORSRules": [{
      "AllowedHeaders": ["*"],
      "AllowedMethods": ["GET", "PUT", "POST"],
      "AllowedOrigins": ["https://myapp.com", "https://www.myapp.com"],
      "ExposeHeaders": ["ETag", "x-amz-request-id"],
      "MaxAgeSeconds": 3600
    }]
  }'
```

### Step 10.2: Quick Reference Commands

```bash
# Sync local directory to S3
aws s3 sync ./build/ s3://myapp-production-data/static/ --delete

# Copy with storage class
aws s3 cp file.log s3://myapp-production-data/logs/ --storage-class STANDARD_IA

# Recursive delete with filter
aws s3 rm s3://myapp-production-data/tmp/ --recursive --include "*.tmp"

# Get object metadata
aws s3api head-object --bucket myapp-production-data --key uploads/file.jpg

# Restore from Glacier
aws s3api restore-object \
  --bucket myapp-production-data \
  --key archives/old-data.zip \
  --restore-request '{"Days": 7, "GlacierJobParameters": {"Tier": "Standard"}}'

# List object versions
aws s3api list-object-versions --bucket myapp-production-data --prefix uploads/important-file.txt
```

---
