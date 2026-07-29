# AWS CloudWatch, Alarms & SNS - Step-by-Step Configuration Guide

## Table of Contents
1. [CloudWatch Agent Installation & Setup](#1-cloudwatch-agent-installation--setup)
2. [Custom Metrics Configuration](#2-custom-metrics-configuration)
3. [CloudWatch Alarms Setup](#3-cloudwatch-alarms-setup)
4. [Composite Alarms](#4-composite-alarms)
5. [SNS Topics & Subscriptions](#5-sns-topics--subscriptions)
6. [CloudWatch Logs Configuration](#6-cloudwatch-logs-configuration)
7. [CloudWatch Dashboards](#7-cloudwatch-dashboards)
8. [Metric Filters & Insights](#8-metric-filters--insights)
9. [Anomaly Detection](#9-anomaly-detection)
10. [Production Monitoring Strategy](#10-production-monitoring-strategy)

---

## 1. CloudWatch Agent Installation & Setup

### Step 1.1: Install CloudWatch Agent


**Amazon Linux 2023 / Amazon Linux 2:**
```bash
# Install via yum
sudo yum install -y amazon-cloudwatch-agent

# Or download manually
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm
```

**Ubuntu/Debian:**
```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb
```

**Using SSM (Recommended for Fleet Management):**
```bash
# Install on multiple instances via SSM Run Command
aws ssm send-command \
  --document-name "AWS-ConfigureAWSPackage" \
  --targets '[{"Key":"tag:Environment","Values":["production"]}]' \
  --parameters '{
    "action": ["Install"],
    "installationType": ["Uninstall and reinstall"],
    "name": ["AmazonCloudWatchAgent"]
  }'
```

### Step 1.2: IAM Role for CloudWatch Agent

```bash
# Attach these policies to EC2 instance role
aws iam attach-role-policy \
  --role-name EC2-Production-Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name EC2-Production-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

### Step 1.3: CloudWatch Agent Configuration File

```bash
# Create configuration using the wizard
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Or create manually:
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc/
```

**Complete Production Configuration:**
```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "cwagent",
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "metrics": {
    "namespace": "Production/EC2",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "aggregation_dimensions": [
      ["AutoScalingGroupName"],
      ["InstanceId"],
      []
    ],
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system",
          "cpu_usage_steal"
        ],
        "metrics_collection_interval": 30,
        "totalcpu": true,
        "resources": ["*"]
      },
      "mem": {
        "measurement": [
          "mem_used_percent",
          "mem_available_percent",
          "mem_used",
          "mem_cached",
          "mem_buffered"
        ],
        "metrics_collection_interval": 30
      },
      "disk": {
        "measurement": [
          "disk_used_percent",
          "disk_free",
          "disk_inodes_free"
        ],
        "resources": ["/", "/opt/myapp"],
        "ignore_file_system_types": ["sysfs", "devtmpfs", "tmpfs"]
      },
      "diskio": {
        "measurement": [
          "diskio_reads",
          "diskio_writes",
          "diskio_read_bytes",
          "diskio_write_bytes",
          "diskio_io_time"
        ],
        "resources": ["nvme0n1"]
      },
      "net": {
        "measurement": [
          "net_bytes_sent",
          "net_bytes_recv",
          "net_packets_sent",
          "net_packets_recv",
          "net_err_in",
          "net_err_out"
        ],
        "resources": ["eth0"]
      },
      "netstat": {
        "measurement": [
          "netstat_tcp_established",
          "netstat_tcp_time_wait",
          "netstat_tcp_close_wait"
        ]
      },
      "processes": {
        "measurement": [
          "processes_running",
          "processes_sleeping",
          "processes_zombies",
          "processes_blocked"
        ]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/myapp/application.log",
            "log_group_name": "/production/myapp/application",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 30,
            "timestamp_format": "%Y-%m-%dT%H:%M:%S"
          },
          {
            "file_path": "/var/log/myapp/error.log",
            "log_group_name": "/production/myapp/errors",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 90,
            "timestamp_format": "%Y-%m-%dT%H:%M:%S"
          },
          {
            "file_path": "/var/log/myapp/access.log",
            "log_group_name": "/production/myapp/access",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 14,
            "timestamp_format": "%d/%b/%Y:%H:%M:%S"
          },
          {
            "file_path": "/var/log/syslog",
            "log_group_name": "/production/system/syslog",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 7
          }
        ]
      }
    },
    "force_flush_interval": 5
  }
}
```

### Step 1.4: Start CloudWatch Agent

```bash
# Start agent with configuration file
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a status

# Restart agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a stop
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a start

# View agent logs
tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### Step 1.5: Store Configuration in SSM Parameter Store

```bash
# Store config in Parameter Store (for fleet deployment)
aws ssm put-parameter \
  --name "/cloudwatch-agent/config/production" \
  --type String \
  --value file://amazon-cloudwatch-agent.json \
  --overwrite

# Fetch and apply from SSM
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c ssm:/cloudwatch-agent/config/production
```


---

## 2. Custom Metrics Configuration

### Step 2.1: Publish Custom Metrics (AWS CLI)

```bash
# Single metric
aws cloudwatch put-metric-data \
  --namespace "Production/MyApp" \
  --metric-name "OrdersProcessed" \
  --value 42 \
  --unit Count \
  --dimensions Service=OrderService,Environment=production

# Multiple metrics in one call
aws cloudwatch put-metric-data \
  --namespace "Production/MyApp" \
  --metric-data '[
    {
      "MetricName": "RequestLatency",
      "Value": 125.5,
      "Unit": "Milliseconds",
      "Dimensions": [
        {"Name": "Service", "Value": "APIGateway"},
        {"Name": "Endpoint", "Value": "/api/orders"}
      ]
    },
    {
      "MetricName": "ActiveConnections",
      "Value": 350,
      "Unit": "Count",
      "Dimensions": [
        {"Name": "Service", "Value": "WebSocket"}
      ]
    }
  ]'

# High-resolution metric (1-second interval)
aws cloudwatch put-metric-data \
  --namespace "Production/MyApp" \
  --metric-data '[{
    "MetricName": "TransactionsPerSecond",
    "Value": 1500,
    "Unit": "Count/Second",
    "StorageResolution": 1
  }]'
```

### Step 2.2: Publish Custom Metrics (Python/Boto3)

```python
import boto3
import time
from datetime import datetime

cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

def publish_application_metrics(metrics_data):
    """Publish custom application metrics to CloudWatch."""
    
    metric_data = []
    
    # Business metric: Orders per minute
    metric_data.append({
        'MetricName': 'OrdersPerMinute',
        'Timestamp': datetime.utcnow(),
        'Value': metrics_data['orders_count'],
        'Unit': 'Count',
        'Dimensions': [
            {'Name': 'Environment', 'Value': 'production'},
            {'Name': 'Service', 'Value': 'order-service'}
        ]
    })
    
    # Performance metric: API response time
    metric_data.append({
        'MetricName': 'APIResponseTime',
        'Timestamp': datetime.utcnow(),
        'Value': metrics_data['response_time_ms'],
        'Unit': 'Milliseconds',
        'Dimensions': [
            {'Name': 'Environment', 'Value': 'production'},
            {'Name': 'Endpoint', 'Value': metrics_data['endpoint']}
        ],
        'StatisticValues': {
            'SampleCount': metrics_data['sample_count'],
            'Sum': metrics_data['total_time'],
            'Minimum': metrics_data['min_time'],
            'Maximum': metrics_data['max_time']
        }
    })
    
    # Error metric
    metric_data.append({
        'MetricName': 'ErrorCount',
        'Timestamp': datetime.utcnow(),
        'Value': metrics_data['error_count'],
        'Unit': 'Count',
        'Dimensions': [
            {'Name': 'Environment', 'Value': 'production'},
            {'Name': 'ErrorType', 'Value': metrics_data['error_type']}
        ]
    })
    
    # Publish in batches of 20 (API limit)
    for i in range(0, len(metric_data), 20):
        batch = metric_data[i:i+20]
        cloudwatch.put_metric_data(
            Namespace='Production/MyApp',
            MetricData=batch
        )

# Usage
publish_application_metrics({
    'orders_count': 150,
    'response_time_ms': 95.5,
    'endpoint': '/api/v1/orders',
    'sample_count': 1000,
    'total_time': 95500,
    'min_time': 12,
    'max_time': 450,
    'error_count': 3,
    'error_type': 'TimeoutError'
})
```

### Step 2.3: Embedded Metric Format (EMF) - Logs to Metrics

```python
import json
import sys

def emit_metric_log(metric_name, value, unit, dimensions):
    """Emit CloudWatch EMF structured log that auto-creates metrics."""
    
    log_entry = {
        "_aws": {
            "Timestamp": int(time.time() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "Production/MyApp",
                "Dimensions": [list(dimensions.keys())],
                "Metrics": [{
                    "Name": metric_name,
                    "Unit": unit
                }]
            }]
        },
        metric_name: value,
        **dimensions
    }
    
    # Print to stdout (CloudWatch Logs will auto-extract metric)
    print(json.dumps(log_entry))
    sys.stdout.flush()

# Usage - This creates both a log AND a metric automatically
emit_metric_log(
    "PaymentProcessingTime", 
    250, 
    "Milliseconds",
    {"Service": "payment-service", "PaymentMethod": "credit_card"}
)
```

---

## 3. CloudWatch Alarms Setup

### Step 3.1: EC2/ASG Alarms

```bash
# High CPU Utilization
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-HighCPU-Warning" \
  --alarm-description "CPU utilization exceeds 80% for 10 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=AutoScalingGroupName,Value=production-api-asg \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings \
  --ok-actions arn:aws:sns:us-east-1:123456789012:production-resolved \
  --tags Key=Environment,Value=production Key=Severity,Value=warning

# Critical CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-HighCPU-Critical" \
  --alarm-description "CPU utilization exceeds 95% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 60 \
  --threshold 95 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 5 \
  --dimensions Name=AutoScalingGroupName,Value=production-api-asg \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical \
  --treat-missing-data breaching

# Memory utilization (from CloudWatch Agent)
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-HighMemory-Warning" \
  --alarm-description "Memory utilization exceeds 85%" \
  --metric-name mem_used_percent \
  --namespace Production/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=AutoScalingGroupName,Value=production-api-asg \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Disk space
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-DiskSpace-Critical" \
  --alarm-description "Disk usage exceeds 90%" \
  --metric-name disk_used_percent \
  --namespace Production/EC2 \
  --statistic Maximum \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=AutoScalingGroupName,Value=production-api-asg \
               Name=path,Value=/ \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical
```

### Step 3.2: ALB Alarms

```bash
# High 5xx error rate
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-ALB-5xxErrors" \
  --alarm-description "5xx errors exceed 50 per minute" \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 60 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=LoadBalancer,Value=app/production-api-alb/xxx \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical \
  --treat-missing-data notBreaching

# High latency (P99)
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-ALB-HighLatency-P99" \
  --alarm-description "P99 response time exceeds 3 seconds" \
  --metric-name TargetResponseTime \
  --namespace AWS/ApplicationELB \
  --extended-statistic p99 \
  --period 60 \
  --threshold 3 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=LoadBalancer,Value=app/production-api-alb/xxx \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Unhealthy targets
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-UnhealthyTargets" \
  --alarm-description "Unhealthy targets detected in target group" \
  --metric-name UnHealthyHostCount \
  --namespace AWS/ApplicationELB \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 2 \
  --dimensions Name=TargetGroup,Value=targetgroup/production-api-tg/xxx \
               Name=LoadBalancer,Value=app/production-api-alb/yyy \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Rejected connections (ALB capacity issue)
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-RejectedConnections" \
  --alarm-description "ALB rejecting connections" \
  --metric-name RejectedConnectionCount \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --dimensions Name=LoadBalancer,Value=app/production-api-alb/xxx \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical
```

### Step 3.3: RDS/Aurora Alarms

```bash
# Database CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-RDS-HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=DBClusterIdentifier,Value=production-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Database connections
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-RDS-HighConnections" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Maximum \
  --period 60 \
  --threshold 400 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=DBClusterIdentifier,Value=production-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# Replica lag
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-Aurora-ReplicaLag" \
  --metric-name AuroraReplicaLag \
  --namespace AWS/RDS \
  --statistic Maximum \
  --period 60 \
  --threshold 200 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=DBClusterIdentifier,Value=production-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Free storage space
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-RDS-LowStorage" \
  --metric-name FreeLocalStorage \
  --namespace AWS/RDS \
  --statistic Minimum \
  --period 300 \
  --threshold 5368709120 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DBClusterIdentifier,Value=production-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical
```


---

## 4. Composite Alarms

### Step 4.1: Create Composite Alarm

```bash
# Composite: Alert only when BOTH CPU AND Memory are high
aws cloudwatch put-composite-alarm \
  --alarm-name "Production-ResourceExhaustion-Critical" \
  --alarm-rule "ALARM(Production-HighCPU-Critical) AND ALARM(Production-HighMemory-Warning)" \
  --alarm-description "Both CPU and Memory are critical - resource exhaustion likely" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical \
  --ok-actions arn:aws:sns:us-east-1:123456789012:production-resolved \
  --tags Key=Environment,Value=production Key=Severity,Value=critical

# Composite: Service degradation (errors OR latency) but NOT maintenance
aws cloudwatch put-composite-alarm \
  --alarm-name "Production-ServiceDegraded" \
  --alarm-rule "(ALARM(Production-ALB-5xxErrors) OR ALARM(Production-ALB-HighLatency-P99)) AND NOT ALARM(MaintenanceWindow-Active)" \
  --alarm-description "Service is degraded and not in maintenance window" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# Composite: Complete outage detection
aws cloudwatch put-composite-alarm \
  --alarm-name "Production-CompleteOutage" \
  --alarm-rule "ALARM(Production-ALB-5xxErrors) AND ALARM(Production-UnhealthyTargets) AND ALARM(Production-HighCPU-Critical)" \
  --alarm-description "Complete service outage - all indicators critical" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-pagerduty
```

### Step 4.2: Maintenance Window Alarm (Suppression)

```bash
# Create a "maintenance mode" alarm that suppresses alerts
aws cloudwatch put-metric-alarm \
  --alarm-name "MaintenanceWindow-Active" \
  --metric-name MaintenanceMode \
  --namespace Production/Operations \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1

# Activate maintenance mode
aws cloudwatch put-metric-data \
  --namespace Production/Operations \
  --metric-name MaintenanceMode \
  --value 1

# Deactivate maintenance mode
aws cloudwatch put-metric-data \
  --namespace Production/Operations \
  --metric-name MaintenanceMode \
  --value 0
```

---

## 5. SNS Topics & Subscriptions

### Step 5.1: Create SNS Topics

```bash
# Critical alerts (PagerDuty)
aws sns create-topic --name production-critical \
  --tags Key=Environment,Value=production Key=Severity,Value=critical

# Warning alerts (Slack)
aws sns create-topic --name production-warnings \
  --tags Key=Environment,Value=production Key=Severity,Value=warning

# Resolved notifications
aws sns create-topic --name production-resolved \
  --tags Key=Environment,Value=production Key=Type,Value=recovery

# Deployment notifications
aws sns create-topic --name deployment-notifications \
  --tags Key=Environment,Value=production Key=Type,Value=deployment
```

### Step 5.2: Add Subscriptions

```bash
# Email subscription
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:production-critical \
  --protocol email \
  --notification-endpoint oncall-team@company.com

# SMS subscription
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:production-critical \
  --protocol sms \
  --notification-endpoint "+1234567890"

# HTTPS (PagerDuty/Webhook)
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:production-critical \
  --protocol https \
  --notification-endpoint "https://events.pagerduty.com/integration/xxx/enqueue"

# Lambda (for custom processing)
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:production-warnings \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:123456789012:function:slack-alert-handler

# SQS (for buffering/processing)
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:production-warnings \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:alert-processing-queue
```

### Step 5.3: SNS Message Filtering

```bash
# Subscribe with filter policy (only receive critical DB alerts)
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:production-critical \
  --protocol email \
  --notification-endpoint dba-team@company.com \
  --attributes '{
    "FilterPolicy": "{\"service\": [\"database\"], \"severity\": [\"critical\"]}"
  }'
```

### Step 5.4: SNS Access Policy (Cross-Account)

```bash
aws sns set-topic-attributes \
  --topic-arn arn:aws:sns:us-east-1:123456789012:production-critical \
  --attribute-name Policy \
  --attribute-value '{
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "cloudwatch.amazonaws.com"},
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789012:production-critical",
      "Condition": {
        "ArnLike": {"aws:SourceArn": "arn:aws:cloudwatch:us-east-1:123456789012:alarm:*"}
      }
    }]
  }'
```


---

## 6. CloudWatch Logs Configuration

### Step 6.1: Create Log Groups

```bash
# Create log groups with retention
aws logs create-log-group --log-group-name /production/myapp/application
aws logs put-retention-policy \
  --log-group-name /production/myapp/application \
  --retention-in-days 30

aws logs create-log-group --log-group-name /production/myapp/errors
aws logs put-retention-policy \
  --log-group-name /production/myapp/errors \
  --retention-in-days 90

aws logs create-log-group --log-group-name /production/myapp/access
aws logs put-retention-policy \
  --log-group-name /production/myapp/access \
  --retention-in-days 14

# Enable encryption with KMS
aws logs associate-kms-key \
  --log-group-name /production/myapp/application \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx-yyy-zzz
```

### Step 6.2: Log Subscription Filters (Stream to Other Services)

```bash
# Stream logs to Lambda for processing
aws logs put-subscription-filter \
  --log-group-name /production/myapp/errors \
  --filter-name "ErrorLogProcessor" \
  --filter-pattern "?ERROR ?CRITICAL ?FATAL" \
  --destination-arn arn:aws:lambda:us-east-1:123456789012:function:error-log-processor

# Stream to Elasticsearch/OpenSearch
aws logs put-subscription-filter \
  --log-group-name /production/myapp/application \
  --filter-name "ElasticsearchStream" \
  --filter-pattern "" \
  --destination-arn arn:aws:es:us-east-1:123456789012:domain/production-logs \
  --role-arn arn:aws:iam::123456789012:role/CWLtoElasticsearchRole

# Stream to Kinesis Firehose (for S3 archival)
aws logs put-subscription-filter \
  --log-group-name /production/myapp/application \
  --filter-name "S3Archive" \
  --filter-pattern "" \
  --destination-arn arn:aws:firehose:us-east-1:123456789012:deliverystream/logs-to-s3 \
  --role-arn arn:aws:iam::123456789012:role/CWLtoFirehoseRole
```

### Step 6.3: Query Logs with CloudWatch Logs Insights

```bash
# Find all errors in the last hour
aws logs start-query \
  --log-group-name /production/myapp/application \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message
    | filter @message like /ERROR/
    | sort @timestamp desc
    | limit 50'

# Get query results
aws logs get-query-results --query-id "query-id-from-above"

# Top error types in last 24 hours
aws logs start-query \
  --log-group-name /production/myapp/errors \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, error_type, message
    | stats count(*) as error_count by error_type
    | sort error_count desc
    | limit 20'

# P99 latency by endpoint
aws logs start-query \
  --log-group-name /production/myapp/access \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields endpoint, duration_ms
    | stats avg(duration_ms) as avg_latency, 
            pct(duration_ms, 95) as p95,
            pct(duration_ms, 99) as p99,
            count(*) as requests
      by endpoint
    | sort p99 desc'

# Find slow requests (>2 seconds)
aws logs start-query \
  --log-group-name /production/myapp/access \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, endpoint, duration_ms, user_id, trace_id
    | filter duration_ms > 2000
    | sort duration_ms desc
    | limit 50'
```

---

## 7. CloudWatch Dashboards

### Step 7.1: Create Production Dashboard

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "Production-Overview" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "API Request Rate & Errors",
          "metrics": [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/production-api-alb/xxx", {"stat": "Sum", "label": "Total Requests"}],
            ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "LoadBalancer", "app/production-api-alb/xxx", {"stat": "Sum", "label": "5xx Errors", "color": "#d13212"}],
            ["AWS/ApplicationELB", "HTTPCode_Target_4XX_Count", "LoadBalancer", "app/production-api-alb/xxx", {"stat": "Sum", "label": "4xx Errors", "color": "#ff7f0e"}]
          ],
          "period": 60,
          "region": "us-east-1",
          "view": "timeSeries"
        }
      },
      {
        "type": "metric",
        "x": 12, "y": 0, "width": 12, "height": 6,
        "properties": {
          "title": "Response Time (P50, P95, P99)",
          "metrics": [
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/production-api-alb/xxx", {"stat": "p50", "label": "P50"}],
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/production-api-alb/xxx", {"stat": "p95", "label": "P95", "color": "#ff7f0e"}],
            ["AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "app/production-api-alb/xxx", {"stat": "p99", "label": "P99", "color": "#d13212"}]
          ],
          "period": 60,
          "annotations": {
            "horizontal": [{"value": 2, "label": "SLO Target (2s)", "color": "#d13212"}]
          }
        }
      },
      {
        "type": "metric",
        "x": 0, "y": 6, "width": 8, "height": 6,
        "properties": {
          "title": "EC2 CPU Utilization (ASG)",
          "metrics": [
            ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "production-api-asg", {"stat": "Average", "label": "Average CPU"}],
            ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "production-api-asg", {"stat": "Maximum", "label": "Max CPU", "color": "#d13212"}]
          ],
          "period": 60,
          "annotations": {
            "horizontal": [
              {"value": 80, "label": "Warning", "color": "#ff7f0e"},
              {"value": 95, "label": "Critical", "color": "#d13212"}
            ]
          }
        }
      },
      {
        "type": "metric",
        "x": 8, "y": 6, "width": 8, "height": 6,
        "properties": {
          "title": "Memory & Disk Utilization",
          "metrics": [
            ["Production/EC2", "mem_used_percent", "AutoScalingGroupName", "production-api-asg", {"stat": "Average", "label": "Memory %"}],
            ["Production/EC2", "disk_used_percent", "AutoScalingGroupName", "production-api-asg", "path", "/", {"stat": "Maximum", "label": "Disk %"}]
          ],
          "period": 300
        }
      },
      {
        "type": "metric",
        "x": 16, "y": 6, "width": 8, "height": 6,
        "properties": {
          "title": "Database Performance",
          "metrics": [
            ["AWS/RDS", "CPUUtilization", "DBClusterIdentifier", "production-aurora-cluster", {"label": "DB CPU"}],
            ["AWS/RDS", "DatabaseConnections", "DBClusterIdentifier", "production-aurora-cluster", {"label": "Connections", "yAxis": "right"}]
          ],
          "period": 60
        }
      },
      {
        "type": "metric",
        "x": 0, "y": 12, "width": 12, "height": 6,
        "properties": {
          "title": "ASG Instance Count",
          "metrics": [
            ["AWS/AutoScaling", "GroupInServiceInstances", "AutoScalingGroupName", "production-api-asg", {"label": "In Service"}],
            ["AWS/AutoScaling", "GroupDesiredCapacity", "AutoScalingGroupName", "production-api-asg", {"label": "Desired"}],
            ["AWS/AutoScaling", "GroupMinSize", "AutoScalingGroupName", "production-api-asg", {"label": "Min"}],
            ["AWS/AutoScaling", "GroupMaxSize", "AutoScalingGroupName", "production-api-asg", {"label": "Max"}]
          ],
          "period": 60
        }
      },
      {
        "type": "alarm",
        "x": 12, "y": 12, "width": 12, "height": 6,
        "properties": {
          "title": "Active Alarms",
          "alarms": [
            "arn:aws:cloudwatch:us-east-1:123456789012:alarm:Production-HighCPU-Critical",
            "arn:aws:cloudwatch:us-east-1:123456789012:alarm:Production-ALB-5xxErrors",
            "arn:aws:cloudwatch:us-east-1:123456789012:alarm:Production-RDS-HighCPU",
            "arn:aws:cloudwatch:us-east-1:123456789012:alarm:Production-UnhealthyTargets"
          ]
        }
      }
    ]
  }'
```


---

## 8. Metric Filters & Insights

### Step 8.1: Create Metric Filters from Logs

```bash
# Count ERROR occurrences
aws logs put-metric-filter \
  --log-group-name /production/myapp/application \
  --filter-name "ErrorCount" \
  --filter-pattern "ERROR" \
  --metric-transformations '[{
    "metricName": "ApplicationErrors",
    "metricNamespace": "Production/MyApp",
    "metricValue": "1",
    "defaultValue": 0
  }]'

# Count specific exceptions
aws logs put-metric-filter \
  --log-group-name /production/myapp/application \
  --filter-name "DatabaseTimeouts" \
  --filter-pattern '{ $.error_type = "DatabaseTimeout" }' \
  --metric-transformations '[{
    "metricName": "DatabaseTimeouts",
    "metricNamespace": "Production/MyApp",
    "metricValue": "1",
    "defaultValue": 0
  }]'

# Extract latency values from logs
aws logs put-metric-filter \
  --log-group-name /production/myapp/access \
  --filter-name "RequestLatency" \
  --filter-pattern '{ $.duration_ms = * }' \
  --metric-transformations '[{
    "metricName": "RequestDuration",
    "metricNamespace": "Production/MyApp",
    "metricValue": "$.duration_ms",
    "unit": "Milliseconds"
  }]'

# Count 5xx responses in application logs
aws logs put-metric-filter \
  --log-group-name /production/myapp/access \
  --filter-name "5xxResponses" \
  --filter-pattern '{ $.status_code >= 500 }' \
  --metric-transformations '[{
    "metricName": "Server5xxErrors",
    "metricNamespace": "Production/MyApp",
    "metricValue": "1",
    "defaultValue": 0
  }]'

# Create alarm on metric filter
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-AppErrors-High" \
  --metric-name ApplicationErrors \
  --namespace Production/MyApp \
  --statistic Sum \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings
```

### Step 8.2: CloudWatch Metric Math

```bash
# Error rate percentage using metric math
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-ErrorRate-Percent" \
  --evaluation-periods 3 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --metrics '[
    {
      "Id": "errors",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "HTTPCode_Target_5XX_Count",
          "Dimensions": [{"Name": "LoadBalancer", "Value": "app/production-api-alb/xxx"}]
        },
        "Period": 60,
        "Stat": "Sum"
      },
      "ReturnData": false
    },
    {
      "Id": "requests",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "RequestCount",
          "Dimensions": [{"Name": "LoadBalancer", "Value": "app/production-api-alb/xxx"}]
        },
        "Period": 60,
        "Stat": "Sum"
      },
      "ReturnData": false
    },
    {
      "Id": "error_rate",
      "Expression": "(errors/requests)*100",
      "Label": "Error Rate %",
      "ReturnData": true
    }
  ]' \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical
```

---

## 9. Anomaly Detection

### Step 9.1: Create Anomaly Detection Alarm

```bash
# Anomaly detection on request count
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-RequestCount-Anomaly" \
  --evaluation-periods 3 \
  --comparison-operator LessThanLowerOrGreaterThanUpperThreshold \
  --threshold-metric-id "anomaly_band" \
  --metrics '[
    {
      "Id": "requests",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "RequestCount",
          "Dimensions": [{"Name": "LoadBalancer", "Value": "app/production-api-alb/xxx"}]
        },
        "Period": 300,
        "Stat": "Sum"
      },
      "ReturnData": true
    },
    {
      "Id": "anomaly_band",
      "Expression": "ANOMALY_DETECTION_BAND(requests, 2)",
      "Label": "Expected Range",
      "ReturnData": true
    }
  ]' \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings \
  --alarm-description "Request count is outside expected range (anomaly detected)"

# Anomaly detection on latency
aws cloudwatch put-metric-alarm \
  --alarm-name "Production-Latency-Anomaly" \
  --evaluation-periods 3 \
  --comparison-operator GreaterThanUpperThreshold \
  --threshold-metric-id "anomaly_band" \
  --metrics '[
    {
      "Id": "latency",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "TargetResponseTime",
          "Dimensions": [{"Name": "LoadBalancer", "Value": "app/production-api-alb/xxx"}]
        },
        "Period": 300,
        "Stat": "Average"
      },
      "ReturnData": true
    },
    {
      "Id": "anomaly_band",
      "Expression": "ANOMALY_DETECTION_BAND(latency, 3)",
      "Label": "Expected Latency Range",
      "ReturnData": true
    }
  ]' \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings
```

---

## 10. Production Monitoring Strategy

### Step 10.1: Complete Alarm Hierarchy

```
Tier 1 - PAGE (PagerDuty - Immediate Response):
├── Complete service outage (all health checks failing)
├── Error rate > 10% sustained for 5 minutes
├── Database failover triggered
├── Security breach detected (GuardDuty high-severity)
└── Data loss risk (backup failures)

Tier 2 - URGENT (Slack #incidents - 15 min response):
├── Error rate > 5% sustained for 5 minutes
├── P99 latency > 5 seconds
├── Unhealthy targets > 2
├── Database CPU > 90%
├── Disk space > 85%
└── Replica lag > 500ms

Tier 3 - WARNING (Slack #monitoring - Business hours):
├── CPU > 80% sustained
├── Memory > 85%
├── Error rate > 2%
├── Deployment failures
└── Certificate expiring within 14 days

Tier 4 - INFO (Dashboard / Daily digest):
├── Auto-scaling events
├── Instance launches/terminations
├── Backup completion status
└── Cost anomalies
```

### Step 10.2: Auto-Remediation with Lambda

```python
# Lambda function triggered by CloudWatch Alarm via SNS
import boto3
import json

ec2 = boto3.client('ec2')
autoscaling = boto3.client('autoscaling')
ssm = boto3.client('ssm')

def lambda_handler(event, context):
    """Auto-remediate based on alarm type."""
    
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    
    if 'DiskSpace' in alarm_name:
        return remediate_disk_space(message)
    elif 'UnhealthyTargets' in alarm_name:
        return remediate_unhealthy_instance(message)
    elif 'HighMemory' in alarm_name:
        return remediate_memory_pressure(message)
    
    return {'statusCode': 200, 'body': 'No remediation action defined'}

def remediate_disk_space(message):
    """Clean old logs and temp files."""
    instance_id = extract_instance_id(message)
    
    ssm.send_command(
        InstanceIds=[instance_id],
        DocumentName='AWS-RunShellScript',
        Parameters={
            'commands': [
                'find /var/log -name "*.gz" -mtime +7 -delete',
                'find /tmp -mtime +3 -delete',
                'journalctl --vacuum-time=3d',
                'docker system prune -f 2>/dev/null || true'
            ]
        }
    )
    return {'action': 'disk_cleanup', 'instance': instance_id}

def remediate_unhealthy_instance(message):
    """Terminate unhealthy instance (ASG will replace)."""
    instance_id = extract_instance_id(message)
    
    autoscaling.terminate_instance_in_auto_scaling_group(
        InstanceId=instance_id,
        ShouldDecrementDesiredCapacity=False
    )
    return {'action': 'instance_terminated', 'instance': instance_id}
```

### Step 10.3: Quick Reference Commands

```bash
# List all alarms in ALARM state
aws cloudwatch describe-alarms --state-value ALARM \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Reason:StateReason}' \
  --output table

# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=production-api-asg \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average Maximum

# List all dashboards
aws cloudwatch list-dashboards

# Describe log groups
aws logs describe-log-groups --log-group-name-prefix /production/

# Tail logs in real-time
aws logs tail /production/myapp/errors --follow --since 5m

# List metric filters
aws logs describe-metric-filters --log-group-name /production/myapp/application

# Test metric filter pattern
aws logs test-metric-filter \
  --log-group-name /production/myapp/application \
  --filter-pattern "ERROR" \
  --log-event-messages '["INFO: Request completed","ERROR: Database timeout","ERROR: Connection refused"]'
```

---
