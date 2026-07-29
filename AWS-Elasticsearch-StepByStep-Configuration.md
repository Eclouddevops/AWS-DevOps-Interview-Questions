# AWS Elasticsearch (OpenSearch) - Step-by-Step Configuration Guide

## Table of Contents
1. [Domain Creation & Setup](#1-domain-creation--setup)
2. [Index Management & ILM](#2-index-management--ilm)
3. [Cluster Sizing & Scaling](#3-cluster-sizing--scaling)
4. [Security Configuration](#4-security-configuration)
5. [Monitoring & Performance](#5-monitoring--performance)
6. [Backup & Recovery](#6-backup--recovery)
7. [Production Best Practices](#7-production-best-practices)

---

## 1. Domain Creation & Setup

### Step 1.1: Create OpenSearch Domain (VPC)


**AWS CLI:**
```bash
aws opensearch create-domain \
  --domain-name production-search \
  --engine-version OpenSearch_2.11 \
  --cluster-config '{
    "InstanceType": "r6g.2xlarge.search",
    "InstanceCount": 6,
    "DedicatedMasterEnabled": true,
    "DedicatedMasterType": "m6g.large.search",
    "DedicatedMasterCount": 3,
    "ZoneAwarenessEnabled": true,
    "ZoneAwarenessConfig": {"AvailabilityZoneCount": 3},
    "WarmEnabled": true,
    "WarmType": "ultrawarm1.large.search",
    "WarmCount": 3
  }' \
  --ebs-options '{
    "EBSEnabled": true,
    "VolumeType": "gp3",
    "VolumeSize": 500,
    "Iops": 3000,
    "Throughput": 125
  }' \
  --vpc-options '{
    "SubnetIds": ["subnet-private-az1", "subnet-private-az2", "subnet-private-az3"],
    "SecurityGroupIds": ["sg-opensearch123456"]
  }' \
  --encryption-at-rest-options Enabled=true,KmsKeyId=arn:aws:kms:us-east-1:123456789012:key/xxx \
  --node-to-node-encryption-options Enabled=true \
  --domain-endpoint-options '{
    "EnforceHTTPS": true,
    "TLSSecurityPolicy": "Policy-Min-TLS-1-2-PFS-2023-10"
  }' \
  --advanced-security-options '{
    "Enabled": true,
    "InternalUserDatabaseEnabled": true,
    "MasterUserOptions": {
      "MasterUserName": "admin",
      "MasterUserPassword": "SecureP@ss123!"
    }
  }' \
  --auto-tune-options '{
    "DesiredState": "ENABLED",
    "MaintenanceSchedules": [{
      "StartAt": "2024-01-20T03:00:00Z",
      "Duration": {"Value": 2, "Unit": "HOURS"},
      "CronExpressionForRecurrence": "cron(0 3 ? * SUN *)"
    }]
  }' \
  --tags Key=Environment,Value=production Key=Team,Value=platform
```

### Step 1.2: Security Group for OpenSearch

```bash
aws ec2 create-security-group \
  --group-name production-opensearch-sg \
  --description "OpenSearch Production - Allow from app layer" \
  --vpc-id vpc-0123456789abcdef0

# Allow HTTPS from application security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-opensearch123456 \
  --protocol tcp \
  --port 443 \
  --source-group sg-app123456
```

### Step 1.3: Access Policy

```bash
aws opensearch update-domain-config \
  --domain-name production-search \
  --access-policies '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/AppServiceRole"},
      "Action": "es:*",
      "Resource": "arn:aws:es:us-east-1:123456789012:domain/production-search/*"
    }]
  }'
```

---

## 2. Index Management & ILM

### Step 2.1: Create Index Template

```bash
# Create index template for time-series data
curl -XPUT "https://vpc-production-search-xxx.us-east-1.es.amazonaws.com/_index_template/delivery-events-template" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{
    "index_patterns": ["delivery-events-*"],
    "template": {
      "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1,
        "refresh_interval": "30s",
        "index.lifecycle.name": "delivery-events-policy",
        "index.lifecycle.rollover_alias": "delivery-events"
      },
      "mappings": {
        "properties": {
          "@timestamp": {"type": "date"},
          "order_id": {"type": "keyword"},
          "driver_id": {"type": "keyword"},
          "status": {"type": "keyword"},
          "location": {"type": "geo_point"},
          "duration_ms": {"type": "integer"},
          "message": {"type": "text", "analyzer": "standard"}
        }
      }
    },
    "priority": 100
  }'
```

### Step 2.2: Index Lifecycle Management (ISM/ILM)

```bash
# Create ISM policy (OpenSearch)
curl -XPUT "https://vpc-production-search-xxx.us-east-1.es.amazonaws.com/_plugins/_ism/policies/delivery-events-policy" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{
    "policy": {
      "description": "Delivery events lifecycle - hot/warm/cold/delete",
      "default_state": "hot",
      "states": [
        {
          "name": "hot",
          "actions": [
            {"rollover": {"min_size": "40gb", "min_index_age": "1d"}}
          ],
          "transitions": [{"state_name": "warm", "conditions": {"min_index_age": "3d"}}]
        },
        {
          "name": "warm",
          "actions": [
            {"replica_count": {"number_of_replicas": 1}},
            {"force_merge": {"max_num_segments": 1}},
            {"allocation": {"require": {"data": "warm"}}}
          ],
          "transitions": [{"state_name": "cold", "conditions": {"min_index_age": "30d"}}]
        },
        {
          "name": "cold",
          "actions": [
            {"replica_count": {"number_of_replicas": 0}},
            {"allocation": {"require": {"data": "cold"}}}
          ],
          "transitions": [{"state_name": "delete", "conditions": {"min_index_age": "90d"}}]
        },
        {
          "name": "delete",
          "actions": [{"delete": {}}]
        }
      ],
      "ism_template": [{"index_patterns": ["delivery-events-*"], "priority": 100}]
    }
  }'
```

### Step 2.3: Create Initial Index with Alias

```bash
# Create the first index and alias
curl -XPUT "https://vpc-production-search-xxx.us-east-1.es.amazonaws.com/delivery-events-000001" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{
    "aliases": {
      "delivery-events": {"is_write_index": true}
    }
  }'
```

---

## 3. Cluster Sizing & Scaling

### Step 3.1: Sizing Guidelines

```
Production Cluster Sizing Formula:
═══════════════════════════════════
Data Nodes:
  Storage needed = Source data × (1 + Replicas) × 1.45 (overhead)
  Example: 500GB source × 2 (1 replica) × 1.45 = 1,450 GB
  Nodes needed: 1,450 GB / 500 GB per node = 3 nodes (minimum 3 for HA)

Instance Selection:
  - Search-heavy: r6g family (memory-optimized)
  - Ingest-heavy: c6g family (compute-optimized)
  - Balanced: m6g family (general purpose)

Shard Sizing:
  - Target: 20-50 GB per shard
  - Max shards per node: 20 shards per GB of JVM heap
  - Example: 30.5 GB heap → max 610 shards per node
```

### Step 3.2: Scale Cluster (Add Nodes)

```bash
# Scale out data nodes
aws opensearch update-domain-config \
  --domain-name production-search \
  --cluster-config '{
    "InstanceType": "r6g.2xlarge.search",
    "InstanceCount": 9,
    "DedicatedMasterEnabled": true,
    "DedicatedMasterType": "m6g.large.search",
    "DedicatedMasterCount": 3,
    "ZoneAwarenessEnabled": true,
    "ZoneAwarenessConfig": {"AvailabilityZoneCount": 3}
  }'

# Scale storage (increase EBS volume)
aws opensearch update-domain-config \
  --domain-name production-search \
  --ebs-options '{
    "EBSEnabled": true,
    "VolumeType": "gp3",
    "VolumeSize": 1000,
    "Iops": 6000,
    "Throughput": 250
  }'
```

---

## 4. Security Configuration

### Step 4.1: Fine-Grained Access Control (Roles)

```bash
# Create read-only role
curl -XPUT "https://vpc-production-search-xxx.us-east-1.es.amazonaws.com/_plugins/_security/api/roles/read_only_role" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{
    "cluster_permissions": ["cluster_composite_ops_ro"],
    "index_permissions": [{
      "index_patterns": ["delivery-events-*"],
      "allowed_actions": ["read", "search"]
    }]
  }'

# Map IAM role to OpenSearch role
curl -XPUT "https://vpc-production-search-xxx.us-east-1.es.amazonaws.com/_plugins/_security/api/rolesmapping/read_only_role" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{
    "backend_roles": ["arn:aws:iam::123456789012:role/AnalyticsReadOnlyRole"]
  }'
```

---

## 5. Monitoring & Performance

### Step 5.1: Key Metrics & Alarms

```bash
# Cluster health (Red status)
aws cloudwatch put-metric-alarm \
  --alarm-name "OpenSearch-ClusterRed" \
  --metric-name ClusterStatus.red \
  --namespace AWS/ES \
  --statistic Maximum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DomainName,Value=production-search Name=ClientId,Value=123456789012 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# JVM Memory Pressure
aws cloudwatch put-metric-alarm \
  --alarm-name "OpenSearch-HighJVMMemory" \
  --metric-name JVMMemoryPressure \
  --namespace AWS/ES \
  --statistic Maximum \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=DomainName,Value=production-search Name=ClientId,Value=123456789012 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings

# Free Storage Space
aws cloudwatch put-metric-alarm \
  --alarm-name "OpenSearch-LowStorage" \
  --metric-name FreeStorageSpace \
  --namespace AWS/ES \
  --statistic Minimum \
  --period 300 \
  --threshold 25000 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DomainName,Value=production-search Name=ClientId,Value=123456789012 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical

# Search Latency
aws cloudwatch put-metric-alarm \
  --alarm-name "OpenSearch-HighSearchLatency" \
  --metric-name SearchLatency \
  --namespace AWS/ES \
  --statistic Average \
  --period 300 \
  --threshold 500 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --dimensions Name=DomainName,Value=production-search Name=ClientId,Value=123456789012 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-warnings
```

### Step 5.2: Cluster Health Commands

```bash
# Cluster health
curl -s "https://vpc-production-search-xxx.us-east-1.es.amazonaws.com/_cluster/health?pretty" -u admin:pass

# Node stats
curl -s "https://.../_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent" -u admin:pass

# Index stats
curl -s "https://.../_cat/indices?v&s=store.size:desc&h=index,health,status,pri,rep,docs.count,store.size" -u admin:pass

# Shard allocation
curl -s "https://.../_cat/shards?v&s=store:desc" -u admin:pass

# Pending tasks
curl -s "https://.../_cluster/pending_tasks?pretty" -u admin:pass
```

---

## 6. Backup & Recovery

### Step 6.1: Automated Snapshots (Built-in)

```bash
# Automated snapshots are taken daily (1-hour window, 14-day retention)
# Configure snapshot window:
aws opensearch update-domain-config \
  --domain-name production-search \
  --snapshot-options AutomatedSnapshotStartHour=3
```

### Step 6.2: Manual Snapshots (S3)

```bash
# Register snapshot repository
curl -XPUT "https://vpc-production-search-xxx.us-east-1.es.amazonaws.com/_snapshot/my-s3-repo" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "myapp-opensearch-snapshots",
      "region": "us-east-1",
      "role_arn": "arn:aws:iam::123456789012:role/OpenSearchSnapshotRole"
    }
  }'

# Take manual snapshot
curl -XPUT "https://.../_snapshot/my-s3-repo/snapshot-$(date +%Y%m%d)" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{"indices": "delivery-events-*", "ignore_unavailable": true}'

# Restore from snapshot
curl -XPOST "https://.../_snapshot/my-s3-repo/snapshot-20240115/_restore" \
  -H "Content-Type: application/json" \
  -u admin:SecureP@ss123! \
  -d '{"indices": "delivery-events-000010", "rename_pattern": "(.+)", "rename_replacement": "restored-$1"}'
```

---

## 7. Production Best Practices

### Step 7.1: Query Optimization

```bash
# Use filter context for non-scoring queries (cached)
curl -XGET "https://.../_search" -d '{
  "query": {
    "bool": {
      "filter": [
        {"term": {"status": "delivered"}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  },
  "size": 100,
  "sort": [{"@timestamp": {"order": "desc"}}]
}'

# Use search_after for deep pagination (avoid from/size)
curl -XGET "https://.../_search" -d '{
  "query": {"match_all": {}},
  "size": 100,
  "sort": [{"@timestamp": "desc"}, {"_id": "asc"}],
  "search_after": ["2024-01-15T10:30:00Z", "doc123"]
}'
```

### Step 7.2: Quick Reference Commands

```bash
# Check domain status
aws opensearch describe-domain --domain-name production-search \
  --query 'DomainStatus.{Endpoint:Endpoints,Status:Processing,Engine:EngineVersion}'

# List all domains
aws opensearch list-domain-names

# Get domain config
aws opensearch describe-domain-config --domain-name production-search

# Check upgrade compatibility
aws opensearch get-compatible-versions --domain-name production-search

# Upgrade domain
aws opensearch upgrade-domain \
  --domain-name production-search \
  --target-version OpenSearch_2.13
```

---
