# Amazon MSK & Streaming — Complete Knowledge Guide

> **Purpose**: Understand event streaming from fundamentals — how Kafka works internally, when to use MSK vs Kinesis, consumer patterns, exactly-once semantics, and production operations.

---

## 1. Kafka Fundamentals (The Mental Model)

### What Problem Does Kafka Solve?

```
WITHOUT Event Streaming:
┌─────────┐     ┌─────────┐     ┌─────────┐
│Service A│────▶│Service B│────▶│Service C│
└─────────┘     └─────────┘     └─────────┘
Problems:
- A must KNOW about B (tight coupling)
- If B is down, A's message is LOST
- Adding Service D means changing A's code
- No replay (if C had a bug, data is gone)

WITH Event Streaming (Kafka):
┌─────────┐     ┌───────────────────────┐     ┌─────────┐
│Service A│────▶│  Kafka Topic           │◀────│Service B│
└─────────┘     │  (persistent log)      │◀────│Service C│
                │                        │◀────│Service D│ (added later!)
                │  Retains messages       │
                │  for days/weeks/forever │
                └───────────────────────┘
Benefits:
- A doesn't know/care who reads
- Messages survive consumer failures (replay!)
- New consumers added without changing producers
- Time travel: reprocess from any point in history
```

### Kafka's Core Data Structure: The Append-Only Log

```
A Kafka TOPIC is split into PARTITIONS.
Each partition is an ordered, immutable sequence of messages.

Topic: "order-events" (3 partitions)

Partition 0: [msg0][msg1][msg2][msg3][msg4][msg5] → offset 5 (latest)
Partition 1: [msg0][msg1][msg2][msg3] → offset 3
Partition 2: [msg0][msg1][msg2][msg3][msg4][msg5][msg6] → offset 6

Key concepts:
├── OFFSET: Position of a message within a partition (like array index)
│   └── Offsets are sequential per partition, NOT global
├── PARTITION: Unit of parallelism (more partitions = more consumers)
│   └── Messages within a partition are ORDERED
│   └── Messages across partitions have NO ordering guarantee
├── KEY: Determines which partition a message goes to
│   └── Same key → always same partition → guaranteed ordering for that key
│   └── No key → round-robin across partitions
└── RETENTION: How long messages are kept (time or size based)
    └── Default: 7 days. Can be: hours, weeks, or FOREVER
```

### Producers, Consumers, and Consumer Groups

```
┌───────────────────────────────────────────────────────────────────┐
│  PRODUCERS (write messages)                                        │
│                                                                     │
│  Producer → Topic:partition (determined by message key)             │
│                                                                     │
│  Acks settings (durability vs speed tradeoff):                     │
│  ├── acks=0: Fire and forget (fastest, may lose messages)          │
│  ├── acks=1: Leader acknowledges (balanced)                        │
│  └── acks=all: All replicas acknowledge (safest, slowest)          │
└───────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────┐
│  CONSUMERS (read messages)                                         │
│                                                                     │
│  Consumer Group: Set of consumers that SHARE the work              │
│                                                                     │
│  Topic with 4 partitions + Consumer Group with 2 consumers:        │
│                                                                     │
│  Partition 0 ──┐                                                    │
│  Partition 1 ──┼──▶ Consumer A (reads P0 + P1)                     │
│  Partition 2 ──┐                                                    │
│  Partition 3 ──┼──▶ Consumer B (reads P2 + P3)                     │
│                                                                     │
│  Rules:                                                             │
│  ├── Each partition assigned to EXACTLY ONE consumer in a group    │
│  ├── One consumer can read MULTIPLE partitions                     │
│  ├── More consumers than partitions = some consumers IDLE          │
│  ├── Different groups = independent (each gets ALL messages)       │
│  └── Consumer dies → partitions reassigned to surviving consumers  │
└───────────────────────────────────────────────────────────────────┘

Multiple Consumer Groups (independent processing):

Topic: "orders"
├── Consumer Group: "payment-processor" → Reads ALL messages → charges cards
├── Consumer Group: "analytics-pipeline" → Reads ALL messages → builds reports
├── Consumer Group: "notification-sender" → Reads ALL messages → sends emails
└── Consumer Group: "fraud-detector" → Reads ALL messages → flags suspicious

Each group has its OWN offset tracking (independent progress)
```

---

## 2. Amazon MSK: Managed Kafka on AWS

### What MSK Manages For You

```
┌─────────────────────────────────────────────────────────────────┐
│  What YOU manage:              │  What AWS manages:              │
│                                │                                 │
│  ├── Topics & partitions      │  ├── Broker provisioning        │
│  ├── Producer/Consumer code   │  ├── ZooKeeper (or KRaft)      │
│  ├── Schema design            │  ├── OS patching                │
│  ├── Consumer group strategy  │  ├── Broker replacement (AZ)   │
│  ├── Retention settings       │  ├── Storage expansion          │
│  ├── Monitoring & alerting    │  ├── Encryption at rest/transit │
│  └── Performance tuning       │  ├── Multi-AZ deployment        │
│                                │  ├── Backup & recovery         │
│                                │  └── Minor version upgrades     │
└────────────────────────────────┴─────────────────────────────────┘
```

### MSK vs MSK Serverless vs Kinesis Data Streams

```
┌─────────────────────────────────────────────────────────────────────┐
│  Feature           │ MSK Provisioned │ MSK Serverless│ Kinesis DS   │
├────────────────────┼─────────────────┼───────────────┼──────────────┤
│ Management         │ Choose brokers  │ Fully managed │ Fully managed│
│ Scaling            │ Manual/planned  │ Automatic     │ On-demand    │
│ Protocol           │ Kafka native    │ Kafka native  │ AWS SDK/HTTP │
│ Ecosystem          │ Full Kafka      │ Full Kafka    │ AWS native   │
│ Retention          │ Unlimited       │ 24 hours      │ 1-365 days   │
│ Throughput         │ GB+/sec         │ Auto-scales   │ Per-shard    │
│ Ordering           │ Per partition   │ Per partition  │ Per shard    │
│ Consumer model     │ Pull (poll)     │ Pull (poll)   │ Pull + Push  │
│ Max message size   │ 10MB (config)   │ 8MB           │ 1MB          │
│ Cross-region       │ MirrorMaker     │ No            │ No (manual)  │
│ Cost model         │ Per-broker-hour │ Per-data      │ Per-shard-hr │
│ Kafka Connect      │ Yes (managed)   │ Yes           │ No           │
│ Schema Registry    │ Yes (Glue)      │ Yes           │ No           │
│ Best for           │ High-throughput, │ Variable     │ AWS-native,  │
│                    │ complex, Kafka   │ workloads    │ simple, low  │
│                    │ expertise       │              │ volume       │
└────────────────────┴─────────────────┴───────────────┴──────────────┘

Decision Guide:
├── "We know Kafka, need full control" → MSK Provisioned
├── "We know Kafka, want less ops" → MSK Serverless
├── "We're AWS-native, simple streams" → Kinesis Data Streams
├── "Lambda consumers, small messages" → Kinesis Data Streams
└── "Complex event processing, high volume" → MSK Provisioned
```

---

## 3. MSK Architecture & Sizing

### Cluster Design

```
Production MSK Cluster:

┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  Availability Zone A    AZ B              AZ C                   │
│  ┌──────────────┐      ┌──────────────┐  ┌──────────────┐      │
│  │ Broker 1     │      │ Broker 2     │  │ Broker 3     │      │
│  │ kafka.m5.2xl │      │ kafka.m5.2xl │  │ kafka.m5.2xl │      │
│  │ 2TB EBS gp3  │      │ 2TB EBS gp3  │  │ 2TB EBS gp3  │      │
│  └──────────────┘      └──────────────┘  └──────────────┘      │
│  ┌──────────────┐      ┌──────────────┐  ┌──────────────┐      │
│  │ Broker 4     │      │ Broker 5     │  │ Broker 6     │      │
│  │ kafka.m5.2xl │      │ kafka.m5.2xl │  │ kafka.m5.2xl │      │
│  │ 2TB EBS gp3  │      │ 2TB EBS gp3  │  │ 2TB EBS gp3  │      │
│  └──────────────┘      └──────────────┘  └──────────────┘      │
│                                                                   │
│  Total: 6 brokers across 3 AZs (2 per AZ)                       │
│  Replication Factor: 3 (data on 3 different brokers)             │
│  Min In-Sync Replicas: 2 (tolerate 1 broker failure)            │
│                                                                   │
│  ZooKeeper (managed by AWS): 3 nodes across 3 AZs               │
└─────────────────────────────────────────────────────────────────┘
```

### Sizing Formula

```
Given:
- Ingestion rate: 100 MB/s
- Replication factor: 3
- Retention: 7 days
- Consumer read rate: 200 MB/s (2 consumer groups reading at 100 MB/s each)

Calculate NETWORK:
Total broker network = Produce + Replication + Consumer reads
= 100 + (100 × 2 replication) + 200 = 500 MB/s across cluster

If each broker handles 100 MB/s network:
Brokers needed (network) = 500 / 100 = 5 brokers minimum

Calculate STORAGE:
Daily ingestion = 100 MB/s × 86400s = 8.64 TB/day
With replication (×3) = 25.92 TB/day
7-day retention = 181.4 TB total
Per broker (6 brokers) = 30.2 TB each

Recommendation: 6 brokers, 4TB EBS each (with headroom for growth)

Calculate PARTITIONS:
Rule: 1 partition ≈ 10 MB/s throughput (producer)
For 100 MB/s: minimum 10 partitions per topic
For parallelism: partitions >= max consumers you'll ever need
Recommendation: 30-50 partitions for high-traffic topics
```

---

## 4. Kafka Internals You Must Understand

### Replication: How Data Survives Failures

```
Topic: "payments", Partition 0, Replication Factor 3

Leader (Broker 1): [msg0][msg1][msg2][msg3][msg4]  ← Producers write HERE
Follower (Broker 2): [msg0][msg1][msg2][msg3][msg4]  ← Catches up via replication
Follower (Broker 3): [msg0][msg1][msg2][msg3]         ← Slightly behind (lag)

ISR (In-Sync Replicas) = {Broker 1, Broker 2}
  └── Broker 3 is behind → NOT in ISR

min.insync.replicas = 2

What happens when Leader (Broker 1) dies?
├── Broker 2 becomes new Leader (was in ISR, fully caught up)
├── Producers now write to Broker 2
├── Broker 3 catches up to become ISR member
├── When Broker 1 recovers, it becomes a follower
└── NO DATA LOSS (because ISR replica had all data)

What if min.insync.replicas = 2 and only 1 replica is in sync?
├── Producer with acks=all will get an error
├── Kafka REFUSES to accept writes (protecting data)
└── This is "availability sacrifice for durability" (correct behavior)
```

### Consumer Offsets: How Kafka Tracks Progress

```
Consumer Group "payment-processor":

Partition 0: Messages [0][1][2][3][4][5][6][7][8][9]
                                     ↑
                              committed offset = 5
                              
Meaning: Consumer has PROCESSED messages 0-4
         Messages 5-9 are available but not yet processed
         
LAG = Latest offset - Committed offset = 9 - 5 = 4 messages behind

Offset Storage: __consumer_offsets (internal Kafka topic)

Commit Strategies:
├── Auto-commit (enable.auto.commit=true)
│   ├── Commits every 5 seconds automatically
│   ├── Risk: Commit BEFORE processing → message lost on crash
│   └── Use for: Non-critical data, analytics
│
├── Manual commit AFTER processing
│   ├── consumer.commitSync() after successful processing
│   ├── Risk: Crash AFTER processing but BEFORE commit → duplicate
│   └── Use for: Most production workloads (at-least-once)
│
└── Transactional commit (exactly-once)
    ├── Commit offset as part of produce transaction
    ├── Atomic: either both succeed or both fail
    └── Use for: Financial transactions, critical data
```

### Consumer Rebalancing (The Hidden Performance Killer)

```
REBALANCE happens when:
├── New consumer joins the group
├── Existing consumer leaves/crashes
├── Topic partitions are added
└── Consumer heartbeat timeout (session.timeout.ms exceeded)

During rebalance:
├── ALL consumers in the group STOP processing
├── Partitions are redistributed
├── Can take 10-60 seconds (or more for large groups)
└── This causes latency spikes!

STRATEGIES to minimize rebalance impact:

1. Cooperative Sticky Assignor (Kafka 2.4+):
   ├── Only reassigns partitions that NEED to move
   ├── Other consumers continue processing
   └── partition.assignment.strategy=CooperativeStickyAssignor

2. Static Group Membership:
   ├── Assign each consumer a stable "group.instance.id"
   ├── Consumer rejoins with same ID → gets same partitions back
   ├── No rebalance on short disconnects
   └── session.timeout.ms can be higher (5+ minutes)

3. Incremental Rebalancing:
   ├── New in Kafka 3.x
   ├── Consumers don't need to revoke all partitions
   └── Only changed assignments cause interruption
```

---

## 5. Delivery Guarantees Explained

### At-Most-Once, At-Least-Once, Exactly-Once

```
AT-MOST-ONCE (may lose, never duplicate):
┌──────────┐     ┌─────────┐     ┌──────────────┐
│ Producer │────▶│  Kafka  │────▶│  Consumer    │
│ acks=0   │     │         │     │ auto-commit  │
└──────────┘     └─────────┘     │ before proc  │
                                  └──────────────┘
When: Log data, metrics, non-critical events
Risk: Message lost if broker fails before replication

AT-LEAST-ONCE (never lose, may duplicate):
┌──────────┐     ┌─────────┐     ┌──────────────┐
│ Producer │────▶│  Kafka  │────▶│  Consumer    │
│ acks=all │     │         │     │ commit after │
│ retries  │     │         │     │ processing   │
└──────────┘     └─────────┘     └──────────────┘
When: Most production workloads (default choice)
Risk: Duplicate if consumer crashes after processing but before commit
Fix: Make consumer idempotent (process duplicates safely)

EXACTLY-ONCE (never lose, never duplicate):
┌──────────┐     ┌─────────┐     ┌──────────────┐
│ Producer │────▶│  Kafka  │────▶│  Consumer    │
│ idempotent│    │ (txn)   │     │ read_committed│
│ transactional│ │         │     │ txn commit   │
└──────────┘     └─────────┘     └──────────────┘
When: Financial transactions, billing, critical state changes
How: Kafka Transactions (atomic produce + offset commit)
Cost: ~3-5% throughput overhead
```

### Implementing Exactly-Once (End-to-End)

```python
# Producer: Idempotent + Transactional
producer_config = {
    'bootstrap.servers': BROKERS,
    'enable.idempotence': True,              # Prevents duplicate produces
    'transactional.id': 'payment-proc-001',  # For transactions
    'acks': 'all',
    'retries': 2147483647,
    'max.in.flight.requests.per.connection': 5,  # OK with idempotence
}

# Consumer: Read only committed messages
consumer_config = {
    'bootstrap.servers': BROKERS,
    'group.id': 'payment-processor',
    'isolation.level': 'read_committed',  # Skip uncommitted txn messages
    'enable.auto.commit': False,           # Manual commit inside txn
}

# The pattern: Consume → Process → Produce + Commit (atomically)
producer.init_transactions()

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    
    try:
        producer.begin_transaction()
        
        # Process
        result = process_payment(msg.value())
        
        # Produce result (part of transaction)
        producer.produce('payment-results', value=result)
        
        # Commit consumer offset (part of SAME transaction)
        producer.send_offsets_to_transaction(
            consumer.position(consumer.assignment()),
            consumer.consumer_group_metadata()
        )
        
        # Atomic commit: produce + offset commit succeed OR both fail
        producer.commit_transaction()
        
    except Exception:
        producer.abort_transaction()
        # Message will be reprocessed (offset not committed)
```

---

## 6. MSK Connect (Managed Kafka Connect)

### What Kafka Connect Does

```
Kafka Connect = Move data IN and OUT of Kafka without writing code

┌────────────┐     ┌───────────────────┐     ┌────────────┐
│ SOURCE     │────▶│  KAFKA TOPIC      │────▶│ SINK       │
│ Connector  │     │                   │     │ Connector  │
│            │     │                   │     │            │
│ Examples:  │     │                   │     │ Examples:  │
│ - Database │     │                   │     │ - S3       │
│ - File     │     │                   │     │ - Redshift │
│ - API      │     │                   │     │ - OpenSearch│
│ - CDC      │     │                   │     │ - Database │
└────────────┘     └───────────────────┘     └────────────┘

Popular Connectors:
├── Debezium (CDC from MySQL, PostgreSQL, MongoDB)
├── S3 Sink (write Kafka data to S3 in Parquet/JSON/Avro)
├── JDBC Source/Sink (read/write databases)
├── Elasticsearch Sink (index Kafka events)
└── HTTP Sink (call APIs with Kafka data)
```

### MSK Connect Configuration

```json
// S3 Sink Connector (write Kafka → S3 as Parquet)
{
  "connector.class": "io.confluent.connect.s3.S3SinkConnector",
  "tasks.max": "8",
  "topics": "order-events,payment-events",
  
  // S3 destination
  "s3.bucket.name": "datalake-bronze",
  "s3.region": "us-east-1",
  
  // File format
  "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
  "parquet.codec": "snappy",
  
  // File rotation (when to create new file)
  "flush.size": "100000",           // Every 100K records
  "rotate.interval.ms": "300000",   // Or every 5 minutes
  
  // Partitioning in S3
  "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
  "path.format": "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH",
  "locale": "en-US",
  "timezone": "UTC",
  "partition.duration.ms": "3600000",  // Hourly partitions
  
  // Schema
  "schema.compatibility": "BACKWARD",
  "value.converter": "io.confluent.connect.avro.AvroConverter",
  "value.converter.schema.registry.url": "https://glue-schema-registry"
}
```

---

## 7. Schema Management (Glue Schema Registry)

### Why Schema Registry Matters

```
WITHOUT Schema Registry:
Producer sends: {"user_id": "123", "amount": 99.99}
Then producer changes to: {"userId": "123", "total": 99.99}
Consumer breaks! Field names changed without warning.

WITH Schema Registry:
Producer registers schema → Registry validates compatibility
├── Schema change "add optional field" → ALLOWED (backward compatible)
├── Schema change "rename field" → REJECTED (breaks consumers!)
└── Schema change "remove field" → REJECTED (breaks consumers!)

Compatibility Modes:
├── BACKWARD: New schema can READ old data (add optional fields OK)
├── FORWARD: Old schema can READ new data (remove optional fields OK)
├── FULL: Both backward AND forward compatible
├── NONE: No compatibility checks (dangerous!)
└── Recommendation: BACKWARD for most use cases
```

```python
# Using Glue Schema Registry with MSK
from aws_schema_registry import DataAndSchema, SchemaRegistryClient
from aws_schema_registry.avro import AvroSchema

# Define schema
order_schema = {
    "type": "record",
    "name": "Order",
    "namespace": "com.company.events",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "string"},
        {"name": "amount", "type": "double"},
        {"name": "timestamp", "type": "long"},
        # New optional field (backward compatible!)
        {"name": "promo_code", "type": ["null", "string"], "default": None}
    ]
}

# Producer with schema validation
client = SchemaRegistryClient(
    registry_name='data-platform-registry'
)

# Register schema (fails if not compatible with previous version)
schema_version = client.register_schema(
    schema_name='order-events',
    schema_definition=json.dumps(order_schema),
    data_format='AVRO',
    compatibility='BACKWARD'
)
```

---

## 8. Stream Processing Patterns

### Pattern 1: Event Enrichment

```
Raw Event:     {"user_id": "123", "product_id": "456", "action": "purchase"}
                    │
                    ▼ (join with user profile from cache/DB)
Enriched Event: {"user_id": "123", "user_name": "John", "tier": "gold",
                 "product_id": "456", "product_name": "Widget", 
                 "action": "purchase", "enriched_at": "2024-06-15T10:00:00Z"}
```

### Pattern 2: Windowed Aggregation

```
Input: Stream of click events

5-minute TUMBLING window (non-overlapping):
[10:00-10:05] → count=150, unique_users=45
[10:05-10:10] → count=180, unique_users=52
[10:10-10:15] → count=120, unique_users=38

5-minute SLIDING window (overlapping, every 1 minute):
[10:00-10:05] → count=150
[10:01-10:06] → count=160
[10:02-10:07] → count=175
... (smoother but more expensive)

SESSION window (gap-based):
User A: [event][event][event]...........[event][event]
         ←── session 1 ──→  30min gap   ←─ session 2 ─→
         Duration: 5 min                  Duration: 3 min
```

### Pattern 3: Event Sourcing + CQRS

```
Command Side (write):
┌──────────┐     ┌────────────────┐
│ API      │────▶│ Kafka Topic    │  ← Events are the source of truth
│ "Create  │     │ "order-events" │  ← Immutable event log
│  Order"  │     │                │
└──────────┘     └────────┬───────┘
                           │
Query Side (read):         ▼
┌──────────────────────────────────────────┐
│ Consumer builds MATERIALIZED VIEWS:       │
├── DynamoDB: Current order status (latest)│
├── Redshift: Analytics aggregations       │
├── ElastiCache: Hot query cache           │
└── OpenSearch: Full-text search index     │
└──────────────────────────────────────────┘

Benefits:
├── Write and read models are INDEPENDENT
├── Can rebuild any view by replaying events
├── Audit trail is built-in (event log = history)
└── Add new projections without changing producers
```

---

## 9. MSK Operations & Monitoring

### Key Metrics to Monitor

```
BROKER HEALTH:
├── UnderReplicatedPartitions: Should be 0 (data at risk if > 0!)
├── OfflinePartitionsCount: Should be 0 (data unavailable!)
├── ActiveControllerCount: Should be exactly 1
├── BrokerStorageUtilization: Alert at 70%, critical at 85%
└── BytesInPerSec / BytesOutPerSec: Throughput (capacity check)

PRODUCER METRICS:
├── ProduceMessageConversions: Should be 0 (format mismatch!)
├── ProduceRequestsPerSec: Traffic volume
└── FailedProduceRequestsPerSec: Should be 0

CONSUMER METRICS:
├── SumOffsetLag: Messages behind (THE most important metric!)
│   └── Growing lag = consumer can't keep up = problem
├── MaxOffsetLag: Worst-case partition lag
├── FetchRequestsPerSec: Consumer activity
└── ConsumerGroupState: Should be "Stable"

STORAGE:
├── KafkaDataLogsDiskUsed: Actual disk usage
├── KafkaAppLogsDiskUsed: Application log disk
└── EstimatedTimeToDiskFull: Emergency alert threshold
```

### CloudWatch Alarms for MSK

```hcl
# CRITICAL: Under-replicated partitions (data durability risk)
resource "aws_cloudwatch_metric_alarm" "under_replicated" {
  alarm_name          = "msk-under-replicated-CRITICAL"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "UnderReplicatedPartitions"
  namespace           = "AWS/Kafka"
  period              = 60
  statistic           = "Maximum"
  threshold           = 0
  alarm_actions       = [aws_sns_topic.pagerduty.arn]
  
  dimensions = { "Cluster Name" = var.cluster_name }
}

# WARNING: Consumer lag growing
resource "aws_cloudwatch_metric_alarm" "consumer_lag" {
  alarm_name          = "msk-consumer-lag-WARNING"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5
  metric_name         = "SumOffsetLag"
  namespace           = "AWS/Kafka"
  period              = 60
  statistic           = "Maximum"
  threshold           = 100000  # 100K messages behind
  alarm_actions       = [aws_sns_topic.slack_alerts.arn]
  
  dimensions = {
    "Cluster Name"   = var.cluster_name
    "Consumer Group" = "payment-processor"
  }
}

# CRITICAL: Storage running out
resource "aws_cloudwatch_metric_alarm" "storage_high" {
  alarm_name          = "msk-storage-CRITICAL"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "KafkaDataLogsDiskUsed"
  namespace           = "AWS/Kafka"
  period              = 300
  statistic           = "Maximum"
  threshold           = 85  # 85% full
  alarm_actions       = [aws_sns_topic.pagerduty.arn]
}
```

---

## 10. Common Operational Tasks

### Scaling MSK

```bash
# Horizontal scaling (add brokers) — requires partition rebalancing
aws kafka update-broker-count \
  --cluster-arn $ARN \
  --target-number-of-broker-nodes 9  # Was 6, now 9

# After adding brokers, reassign partitions to new brokers:
kafka-reassign-partitions.sh --bootstrap-server $BROKER \
  --reassignment-json-file new-assignment.json \
  --execute

# Vertical scaling (bigger instances) — rolling restart
aws kafka update-broker-type \
  --cluster-arn $ARN \
  --target-instance-type kafka.m5.4xlarge

# Storage scaling (more disk) — online, no restart
aws kafka update-broker-storage \
  --cluster-arn $ARN \
  --target-broker-ebs-volume-info '[{"KafkaBrokerNodeId":"ALL","VolumeSizeGB":4000}]'
```

### Topic Management

```bash
# Create topic with optimal settings
kafka-topics.sh --create \
  --bootstrap-server $BROKER \
  --topic payment-events \
  --partitions 30 \
  --replication-factor 3 \
  --config retention.ms=604800000 \      # 7 days
  --config cleanup.policy=delete \
  --config min.insync.replicas=2 \
  --config compression.type=lz4

# Increase partitions (CANNOT decrease!)
kafka-topics.sh --alter \
  --bootstrap-server $BROKER \
  --topic payment-events \
  --partitions 60  # Doubled from 30

# Check consumer lag
kafka-consumer-groups.sh \
  --bootstrap-server $BROKER \
  --describe \
  --group payment-processor

# Reset consumer offset (reprocess from beginning)
kafka-consumer-groups.sh \
  --bootstrap-server $BROKER \
  --group payment-processor \
  --topic payment-events \
  --reset-offsets --to-earliest \
  --execute
```

---

## 11. Production Best Practices

```yaml
topic_design:
  naming_convention: "{domain}.{entity}.{event_type}"
  examples:
    - "payments.orders.created"
    - "payments.orders.updated"
    - "users.profiles.changed"
  
  partitioning:
    key_selection: "Use natural business key (order_id, user_id)"
    partition_count: "Start with 3× expected consumer count"
    rule: "NEVER reduce partitions (only increase)"
    
  retention:
    default: "7 days (enough for replay and debugging)"
    compliance: "Longer if regulatory requirement"
    compacted_topics: "Keep latest value per key (forever)"

producer_best_practices:
  - Always set a message key (for ordering guarantee)
  - Use acks=all for critical data
  - Enable idempotence (prevent duplicates on retry)
  - Batch messages (linger.ms=5, batch.size=16384)
  - Compress messages (compression.type=lz4 or zstd)
  - Handle send failures with retry + dead letter

consumer_best_practices:
  - Use manual offset commit (after successful processing)
  - Make processing idempotent (handle duplicates gracefully)
  - Set session.timeout.ms appropriately (default 45s)
  - Use cooperative sticky assignor (minimize rebalances)
  - Process in batches (max.poll.records=500)
  - Monitor consumer lag (alert on growing lag)
  - Handle poison pills (bad messages that crash consumer)

operational_best_practices:
  - Always use 3+ brokers across 3 AZs
  - Set replication factor = 3, min.insync.replicas = 2
  - Monitor: UnderReplicatedPartitions, consumer lag, disk usage
  - Enable TLS encryption in transit
  - Use IAM authentication (not SASL/PLAIN)
  - Set up dead letter topics for failed messages
  - Regular testing: broker failure, AZ failure, consumer failure
```

---

## 12. Learning Path

```
Beginner (Week 1-2):
├── Understand topics, partitions, offsets, consumer groups
├── Create MSK Serverless cluster
├── Write producer and consumer in Python
├── Observe message ordering within partitions
└── Understand acks settings and delivery guarantees

Intermediate (Week 3-4):
├── Deploy MSK Provisioned cluster (multi-AZ)
├── Configure MSK Connect (S3 Sink)
├── Implement consumer group with multiple consumers
├── Handle consumer rebalancing gracefully
├── Set up Schema Registry (Glue)
├── Monitor: consumer lag, broker health metrics
└── Implement at-least-once with idempotent consumer

Advanced (Week 5-8):
├── Implement exactly-once semantics (transactions)
├── Stream processing with Flink / Kafka Streams
├── Windowed aggregations (tumbling, sliding, session)
├── Event sourcing + CQRS pattern
├── Cross-region replication (MirrorMaker 2)
├── Performance tuning (throughput optimization)
├── Partition reassignment and cluster scaling
└── Disaster recovery planning for MSK

Expert (Month 3+):
├── Design multi-tenant streaming platform
├── Implement change data capture at scale (Debezium)
├── Build real-time ML feature pipeline
├── Optimize for 1M+ messages/second
├── Implement event mesh across multiple domains
└── Cost optimization ($50K+ monthly clusters)
```
