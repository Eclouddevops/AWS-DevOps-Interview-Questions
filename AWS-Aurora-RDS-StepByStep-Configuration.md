# AWS Aurora/RDS - Step-by-Step Configuration Guide

## Table of Contents
1. [Subnet Group & Security Setup](#1-subnet-group--security-setup)
2. [Aurora Cluster Creation](#2-aurora-cluster-creation)
3. [RDS Proxy Configuration](#3-rds-proxy-configuration)
4. [Backup & Recovery](#4-backup--recovery)
5. [Read Replicas & Scaling](#5-read-replicas--scaling)
6. [Aurora Global Database](#6-aurora-global-database)
7. [Performance Tuning](#7-performance-tuning)
8. [Monitoring & Alarms](#8-monitoring--alarms)
9. [Maintenance & Upgrades](#9-maintenance--upgrades)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. Subnet Group & Security Setup

### Step 1.1: Create DB Subnet Group


**AWS Console Path:**
```
RDS → Subnet Groups → Create DB Subnet Group
```

**AWS CLI:**
```bash
# Create subnet group across 3 AZs (private subnets only)
aws rds create-db-subnet-group \
  --db-subnet-group-name production-db-subnet-group \
  --db-subnet-group-description "Production database subnet group - private subnets" \
  --subnet-ids subnet-private-az1 subnet-private-az2 subnet-private-az3 \
  --tags Key=Environment,Value=production
```

### Step 1.2: Create Security Group for Database

```bash
# Create security group
aws ec2 create-security-group \
  --group-name production-aurora-sg \
  --description "Aurora PostgreSQL Production - Allow from app layer only" \
  --vpc-id vpc-0123456789abcdef0

# Allow PostgreSQL (5432) ONLY from application security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-aurora123456 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app123456

# Allow from RDS Proxy security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-aurora123456 \
  --protocol tcp \
  --port 5432 \
  --source-group sg-rdsproxy123456
```

### Step 1.3: Create Parameter Groups

```bash
# Create cluster parameter group
aws rds create-db-cluster-parameter-group \
  --db-cluster-parameter-group-name production-aurora-pg15-cluster \
  --db-parameter-group-family aurora-postgresql15 \
  --description "Production Aurora PostgreSQL 15 cluster parameters"

# Create instance parameter group
aws rds create-db-parameter-group \
  --db-parameter-group-name production-aurora-pg15-instance \
  --db-parameter-group-family aurora-postgresql15 \
  --description "Production Aurora PostgreSQL 15 instance parameters"

# Set cluster parameters
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name production-aurora-pg15-cluster \
  --parameters \
    "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot" \
    "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate" \
    "ParameterName=log_statement,ParameterValue=ddl,ApplyMethod=immediate" \
    "ParameterName=idle_in_transaction_session_timeout,ParameterValue=60000,ApplyMethod=immediate" \
    "ParameterName=statement_timeout,ParameterValue=30000,ApplyMethod=immediate"

# Set instance parameters
aws rds modify-db-parameter-group \
  --db-parameter-group-name production-aurora-pg15-instance \
  --parameters \
    "ParameterName=shared_buffers,ParameterValue={DBInstanceClassMemory/4},ApplyMethod=pending-reboot" \
    "ParameterName=effective_cache_size,ParameterValue={DBInstanceClassMemory*3/4},ApplyMethod=pending-reboot" \
    "ParameterName=work_mem,ParameterValue=65536,ApplyMethod=immediate" \
    "ParameterName=maintenance_work_mem,ParameterValue=2097152,ApplyMethod=immediate" \
    "ParameterName=max_parallel_workers_per_gather,ParameterValue=4,ApplyMethod=immediate"
```

---

## 2. Aurora Cluster Creation

### Step 2.1: Create Aurora PostgreSQL Cluster

**AWS Console Path:**
```
RDS → Databases → Create Database
├── Engine: Amazon Aurora
├── Edition: Aurora (PostgreSQL Compatible)
├── Version: 15.4
├── Template: Production
├── Cluster: production-aurora-cluster
└── Instance: db.r6g.2xlarge
```

**AWS CLI:**
```bash
# Create Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier production-aurora-cluster \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username admin_user \
  --manage-master-user-password \
  --db-subnet-group-name production-db-subnet-group \
  --vpc-security-group-ids sg-aurora123456 \
  --db-cluster-parameter-group-name production-aurora-pg15-cluster \
  --backup-retention-period 35 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:05:00-sun:06:00" \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx-yyy-zzz \
  --enable-cloudwatch-logs-exports '["postgresql", "upgrade"]' \
  --deletion-protection \
  --copy-tags-to-snapshot \
  --tags Key=Environment,Value=production Key=Team,Value=platform
```

### Step 2.2: Create Writer Instance

```bash
aws rds create-db-instance \
  --db-instance-identifier production-aurora-writer \
  --db-cluster-identifier production-aurora-cluster \
  --db-instance-class db.r6g.2xlarge \
  --engine aurora-postgresql \
  --db-parameter-group-name production-aurora-pg15-instance \
  --availability-zone us-east-1a \
  --auto-minor-version-upgrade \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --performance-insights-kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx \
  --tags Key=Environment,Value=production Key=Role,Value=writer
```

### Step 2.3: Create Reader Instances

```bash
# Reader 1 (AZ-b)
aws rds create-db-instance \
  --db-instance-identifier production-aurora-reader-1 \
  --db-cluster-identifier production-aurora-cluster \
  --db-instance-class db.r6g.2xlarge \
  --engine aurora-postgresql \
  --db-parameter-group-name production-aurora-pg15-instance \
  --availability-zone us-east-1b \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --tags Key=Environment,Value=production Key=Role,Value=reader

# Reader 2 (AZ-c)
aws rds create-db-instance \
  --db-instance-identifier production-aurora-reader-2 \
  --db-cluster-identifier production-aurora-cluster \
  --db-instance-class db.r6g.xlarge \
  --engine aurora-postgresql \
  --db-parameter-group-name production-aurora-pg15-instance \
  --availability-zone us-east-1c \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
  --enable-performance-insights \
  --performance-insights-retention-period 731 \
  --tags Key=Environment,Value=production Key=Role,Value=reader
```

### Step 2.4: Get Connection Endpoints

```bash
# Get cluster endpoints
aws rds describe-db-clusters \
  --db-cluster-identifier production-aurora-cluster \
  --query 'DBClusters[0].{
    WriterEndpoint: Endpoint,
    ReaderEndpoint: ReaderEndpoint,
    Port: Port,
    Status: Status
  }'

# Output:
# WriterEndpoint: production-aurora-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com
# ReaderEndpoint: production-aurora-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com
# Port: 5432
```


---

## 3. RDS Proxy Configuration

### Step 3.1: Create Secrets Manager Secret

```bash
# Store database credentials in Secrets Manager
aws secretsmanager create-secret \
  --name production/aurora/credentials \
  --description "Production Aurora database credentials" \
  --secret-string '{"username":"app_user","password":"SecurePassword123!"}'
```

### Step 3.2: Create RDS Proxy IAM Role

```bash
# Create role for RDS Proxy
aws iam create-role \
  --role-name RDSProxyRole \
  --assume-role-policy-document '{
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "rds.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach policy to access Secrets Manager
aws iam put-role-policy \
  --role-name RDSProxyRole \
  --policy-name RDSProxySecretsAccess \
  --policy-document '{
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:production/aurora/*"
    }, {
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/xxx-yyy-zzz"
    }]
  }'
```

### Step 3.3: Create RDS Proxy

```bash
aws rds create-db-proxy \
  --db-proxy-name production-aurora-proxy \
  --engine-family POSTGRESQL \
  --auth '[{
    "Description": "Production app credentials",
    "AuthScheme": "SECRETS",
    "SecretArn": "arn:aws:secretsmanager:us-east-1:123456789012:secret:production/aurora/credentials",
    "IAMAuth": "REQUIRED"
  }]' \
  --role-arn arn:aws:iam::123456789012:role/RDSProxyRole \
  --vpc-subnet-ids subnet-private-az1 subnet-private-az2 subnet-private-az3 \
  --vpc-security-group-ids sg-rdsproxy123456 \
  --require-tls \
  --idle-client-timeout 1800 \
  --debug-logging \
  --tags Key=Environment,Value=production

# Register target (Aurora cluster)
aws rds register-db-proxy-targets \
  --db-proxy-name production-aurora-proxy \
  --db-cluster-identifiers production-aurora-cluster
```

### Step 3.4: Create Proxy Endpoints (Read/Write Separation)

```bash
# Create read-only endpoint
aws rds create-db-proxy-endpoint \
  --db-proxy-name production-aurora-proxy \
  --db-proxy-endpoint-name production-proxy-reader \
  --vpc-subnet-ids subnet-private-az1 subnet-private-az2 subnet-private-az3 \
  --vpc-security-group-ids sg-rdsproxy123456 \
  --target-role READ_ONLY

# Get proxy endpoints
aws rds describe-db-proxy-endpoints \
  --db-proxy-name production-aurora-proxy \
  --query 'DBProxyEndpoints[].{Name:DBProxyEndpointName,Endpoint:Endpoint,Role:TargetRole}'
```

---

## 4. Backup & Recovery

### Step 4.1: Automated Backups (Already configured in cluster creation)

```bash
# Modify backup retention
aws rds modify-db-cluster \
  --db-cluster-identifier production-aurora-cluster \
  --backup-retention-period 35 \
  --preferred-backup-window "03:00-04:00" \
  --apply-immediately
```

### Step 4.2: Manual Snapshot

```bash
# Create manual cluster snapshot
aws rds create-db-cluster-snapshot \
  --db-cluster-identifier production-aurora-cluster \
  --db-cluster-snapshot-identifier production-manual-snap-$(date +%Y%m%d) \
  --tags Key=Purpose,Value=pre-migration Key=Environment,Value=production

# List snapshots
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier production-aurora-cluster \
  --query 'DBClusterSnapshots[].{Id:DBClusterSnapshotIdentifier,Status:Status,Created:SnapshotCreateTime}' \
  --output table
```

### Step 4.3: Point-in-Time Recovery (PITR)

```bash
# Restore to specific point in time
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier production-aurora-cluster \
  --db-cluster-identifier production-aurora-pitr-recovery \
  --restore-to-time "2024-01-15T10:30:00Z" \
  --db-subnet-group-name production-db-subnet-group \
  --vpc-security-group-ids sg-aurora123456

# Or restore to latest restorable time
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier production-aurora-cluster \
  --db-cluster-identifier production-aurora-pitr-recovery \
  --use-latest-restorable-time \
  --db-subnet-group-name production-db-subnet-group \
  --vpc-security-group-ids sg-aurora123456

# Create instance in restored cluster
aws rds create-db-instance \
  --db-instance-identifier production-aurora-pitr-instance \
  --db-cluster-identifier production-aurora-pitr-recovery \
  --db-instance-class db.r6g.2xlarge \
  --engine aurora-postgresql
```

### Step 4.4: Cross-Region Snapshot Copy

```bash
# Copy snapshot to another region for DR
aws rds copy-db-cluster-snapshot \
  --source-db-cluster-snapshot-identifier arn:aws:rds:us-east-1:123456789012:cluster-snapshot:production-manual-snap-20240115 \
  --target-db-cluster-snapshot-identifier production-dr-snap-20240115 \
  --kms-key-id arn:aws:kms:us-west-2:123456789012:key/dr-key-xxx \
  --region us-west-2

# Automate with Lambda (scheduled every 6 hours)
```

### Step 4.5: Export Snapshot to S3

```bash
# Export snapshot data to S3 (for analytics)
aws rds start-export-task \
  --export-task-identifier production-export-20240115 \
  --source-arn arn:aws:rds:us-east-1:123456789012:cluster-snapshot:production-manual-snap-20240115 \
  --s3-bucket-name myapp-db-exports \
  --s3-prefix aurora-exports/ \
  --iam-role-arn arn:aws:iam::123456789012:role/RDSExportRole \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxx
```


---

## 5. Read Replicas & Scaling

### Step 5.1: Aurora Auto Scaling (Read Replicas)

```bash
# Register Aurora cluster as scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace rds \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --resource-id cluster:production-aurora-cluster \
  --min-capacity 2 \
  --max-capacity 8

# Create scaling policy (target CPU 70%)
aws application-autoscaling put-scaling-policy \
  --service-namespace rds \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --resource-id cluster:production-aurora-cluster \
  --policy-name aurora-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "RDSReaderAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 120
  }'

# Create scaling policy based on connections
aws application-autoscaling put-scaling-policy \
  --service-namespace rds \
  --scalable-dimension rds:cluster:ReadReplicaCount \
  --resource-id cluster:production-aurora-cluster \
  --policy-name aurora-connections-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 300.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "RDSReaderAverageDatabaseConnections"
    },
    "ScaleInCooldown": 600,
    "ScaleOutCooldown": 120
  }'
```

### Step 5.2: Aurora Serverless v2 (Auto-Scaling Capacity)

```bash
# Add Serverless v2 reader for burst capacity
aws rds create-db-instance \
  --db-instance-identifier production-aurora-serverless-reader \
  --db-cluster-identifier production-aurora-cluster \
  --db-instance-class db.serverless \
  --engine aurora-postgresql

# Configure serverless scaling range
aws rds modify-db-cluster \
  --db-cluster-identifier production-aurora-cluster \
  --serverless-v2-scaling-configuration MinCapacity=2,MaxCapacity=64
```

### Step 5.3: Failover Priority Configuration

```bash
# Set failover priority (lower number = higher priority)
aws rds modify-db-instance \
  --db-instance-identifier production-aurora-reader-1 \
  --promotion-tier 1  # First to be promoted

aws rds modify-db-instance \
  --db-instance-identifier production-aurora-reader-2 \
  --promotion-tier 2  # Second priority

# Manual failover (for testing)
aws rds failover-db-cluster \
  --db-cluster-identifier production-aurora-cluster \
  --target-db-instance-identifier production-aurora-reader-1
```

---

## 6. Aurora Global Database

### Step 6.1: Create Global Database

```bash
# Step 1: Create global database from existing cluster
aws rds create-global-cluster \
  --global-cluster-identifier production-global-aurora \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:production-aurora-cluster

# Step 2: Add secondary region cluster
aws rds create-db-cluster \
  --db-cluster-identifier production-aurora-dr-cluster \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --global-cluster-identifier production-global-aurora \
  --db-subnet-group-name dr-db-subnet-group \
  --vpc-security-group-ids sg-aurora-dr-123456 \
  --region us-west-2

# Step 3: Add instance in secondary cluster
aws rds create-db-instance \
  --db-instance-identifier production-aurora-dr-reader-1 \
  --db-cluster-identifier production-aurora-dr-cluster \
  --db-instance-class db.r6g.2xlarge \
  --engine aurora-postgresql \
  --region us-west-2
```

### Step 6.2: Global Database Failover

```bash
# Planned failover (managed switchover - no data loss)
aws rds switchover-global-cluster \
  --global-cluster-identifier production-global-aurora \
  --target-db-cluster-identifier arn:aws:rds:us-west-2:123456789012:cluster:production-aurora-dr-cluster

# Unplanned failover (detach secondary - possible minimal data loss)
aws rds remove-from-global-cluster \
  --global-cluster-identifier production-global-aurora \
  --db-cluster-identifier arn:aws:rds:us-west-2:123456789012:cluster:production-aurora-dr-cluster \
  --region us-west-2

# Monitor replication lag
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraGlobalDBReplicationLag \
  --dimensions Name=DBClusterIdentifier,Value=production-aurora-dr-cluster \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Average Maximum \
  --region us-west-2
```

---

## 7. Performance Tuning

### Step 7.1: Enable Performance Insights

```bash
# Already enabled during instance creation, verify:
aws rds describe-db-instances \
  --db-instance-identifier production-aurora-writer \
  --query 'DBInstances[0].{
    PerformanceInsightsEnabled: PerformanceInsightsEnabled,
    RetentionPeriod: PerformanceInsightsRetentionPeriod
  }'

# Query Performance Insights data
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-XXXXXXXXXXXXXXXXXXXXX \
  --metric-queries '[{
    "Metric": "db.load.avg",
    "GroupBy": {"Group": "db.sql", "Limit": 10}
  }]' \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period-in-seconds 60
```

### Step 7.2: Slow Query Analysis

```sql
-- Enable pg_stat_statements (in parameter group)
-- Then query top slow queries:

SELECT
  query,
  calls,
  round(mean_exec_time::numeric, 2) as avg_time_ms,
  round(total_exec_time::numeric, 2) as total_time_ms,
  round((100 * total_exec_time / sum(total_exec_time) OVER())::numeric, 2) as percent_total
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find missing indexes
SELECT
  schemaname, relname,
  seq_scan, seq_tup_read,
  idx_scan, idx_tup_fetch,
  CASE WHEN seq_scan > 0 THEN round(seq_tup_read::numeric / seq_scan, 0) ELSE 0 END as avg_rows_per_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100 AND seq_scan > idx_scan
ORDER BY seq_tup_read DESC
LIMIT 20;

-- Check table bloat
SELECT
  schemaname, tablename,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
  n_dead_tup,
  n_live_tup,
  round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) as dead_ratio_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- Active connections and their state
SELECT
  state, count(*),
  round(avg(EXTRACT(EPOCH FROM (now() - state_change)))::numeric, 0) as avg_duration_sec
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
GROUP BY state;
```

### Step 7.3: Connection Management

```python
# Application connection pool configuration (SQLAlchemy)
from sqlalchemy import create_engine

# Writer connection
writer_engine = create_engine(
    "postgresql://user:pass@writer-endpoint:5432/mydb",
    pool_size=20,              # Base connections
    max_overflow=30,           # Additional connections under load
    pool_timeout=30,           # Wait time for connection
    pool_recycle=1800,         # Recycle connections every 30 min
    pool_pre_ping=True,        # Verify connection before use
    connect_args={
        "connect_timeout": 5,
        "options": "-c statement_timeout=30000"
    }
)

# Reader connection (for read-heavy queries)
reader_engine = create_engine(
    "postgresql://user:pass@reader-endpoint:5432/mydb",
    pool_size=40,
    max_overflow=60,
    pool_timeout=30,
    pool_recycle=1800,
    pool_pre_ping=True
)
```

---

## 8. Monitoring & Alarms

### Step 8.1: Critical Aurora Alarms

```bash
# CPU Utilization
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-Writer-HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=DBInstanceIdentifier,Value=production-aurora-writer \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Freeable Memory
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-Writer-LowMemory" \
  --metric-name FreeableMemory \
  --namespace AWS/RDS \
  --statistic Minimum \
  --period 300 \
  --threshold 1073741824 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=DBInstanceIdentifier,Value=production-aurora-writer \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# Database Connections
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-Cluster-HighConnections" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Maximum \
  --period 60 \
  --threshold 400 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=DBClusterIdentifier,Value=production-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# Replica Lag
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-ReplicaLag-High" \
  --metric-name AuroraReplicaLag \
  --namespace AWS/RDS \
  --statistic Maximum \
  --period 60 \
  --threshold 200 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=DBClusterIdentifier,Value=production-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Write Latency
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-Writer-HighWriteLatency" \
  --metric-name WriteLatency \
  --namespace AWS/RDS \
  --statistic Average \
  --period 60 \
  --threshold 0.020 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 5 \
  --dimensions Name=DBInstanceIdentifier,Value=production-aurora-writer \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Deadlocks
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-Deadlocks" \
  --metric-name Deadlocks \
  --namespace AWS/RDS \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DBInstanceIdentifier,Value=production-aurora-writer \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings
```

---

## 9. Maintenance & Upgrades

### Step 9.1: Minor Version Upgrade

```bash
# Check available upgrades
aws rds describe-db-engine-versions \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --query 'DBEngineVersions[0].ValidUpgradeTarget[].EngineVersion'

# Apply minor upgrade
aws rds modify-db-cluster \
  --db-cluster-identifier production-aurora-cluster \
  --engine-version 15.5 \
  --apply-immediately
# Note: For production, use --no-apply-immediately to apply during maintenance window
```

### Step 9.2: Major Version Upgrade

```bash
# Step 1: Test in staging/clone first
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier production-aurora-cluster \
  --db-cluster-identifier upgrade-test-cluster \
  --use-latest-restorable-time \
  --engine-version 16.1

# Step 2: Run application tests against upgraded clone

# Step 3: Upgrade production (during maintenance window)
aws rds modify-db-cluster \
  --db-cluster-identifier production-aurora-cluster \
  --engine-version 16.1 \
  --allow-major-version-upgrade \
  --db-cluster-parameter-group-name production-aurora-pg16-cluster \
  --no-apply-immediately
```

### Step 9.3: Instance Class Modification (Vertical Scaling)

```bash
# Modify writer instance class
aws rds modify-db-instance \
  --db-instance-identifier production-aurora-writer \
  --db-instance-class db.r6g.4xlarge \
  --apply-immediately
# Note: This causes a brief interruption (failover to reader, then modify)

# Zero-downtime approach:
# 1. Add a new larger reader
# 2. Failover to the new reader (becomes writer)
# 3. Modify or remove the old smaller instance
```

---

## 10. Production Best Practices

### Step 10.1: Connection String Configuration

```bash
# Application environment variables
DATABASE_WRITER_URL="postgresql://app_user:password@production-aurora-proxy.proxy-xxxxx.us-east-1.rds.amazonaws.com:5432/mydb?sslmode=require"
DATABASE_READER_URL="postgresql://app_user:password@production-proxy-reader.endpoint.proxy-xxxxx.us-east-1.rds.amazonaws.com:5432/mydb?sslmode=require"
```

### Step 10.2: Quick Reference Commands

```bash
# List all clusters
aws rds describe-db-clusters \
  --query 'DBClusters[].{Cluster:DBClusterIdentifier,Status:Status,Engine:Engine,Endpoint:Endpoint}'

# Check cluster instances
aws rds describe-db-instances \
  --filters Name=db-cluster-id,Values=production-aurora-cluster \
  --query 'DBInstances[].{Instance:DBInstanceIdentifier,Class:DBInstanceClass,AZ:AvailabilityZone,Status:DBInstanceStatus}'

# Force failover
aws rds failover-db-cluster \
  --db-cluster-identifier production-aurora-cluster

# Check events
aws rds describe-events \
  --source-identifier production-aurora-cluster \
  --source-type db-cluster \
  --duration 1440

# Modify maintenance window
aws rds modify-db-cluster \
  --db-cluster-identifier production-aurora-cluster \
  --preferred-maintenance-window "sun:05:00-sun:06:00"

# Create cluster clone (fast copy for testing)
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier production-aurora-cluster \
  --db-cluster-identifier production-clone-testing \
  --restore-type copy-on-write \
  --use-latest-restorable-time
```

---
