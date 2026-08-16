# Amazon MSK — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | Details |
|-----------|---------|
| **Service Name** | Amazon MSK (Managed Streaming for Apache Kafka) |
| **Category** | Analytics / Streaming & Messaging |
| **Type** | Managed Apache Kafka |
| **Engine** | Apache Kafka (versions 2.6 – 3.6+) |
| **Launched** | May 2019 (GA) |
| **Pricing Model** | Per-broker-hour + storage (Provisioned) or per-data (Serverless) |
| **Key Differentiator** | Fully compatible Kafka on AWS with ZooKeeper/KRaft managed |

---

## 🏗️ What Amazon MSK Does

MSK is a **managed Apache Kafka service** — it runs real Kafka clusters on AWS without you managing brokers, ZooKeeper, patching, or storage scaling.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AMAZON MSK                                     │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  MSK PROVISIONED (you choose broker type + count)               ││
│  │                                                                   ││
│  │  AZ-a              AZ-b              AZ-c                        ││
│  │  ┌──────────┐     ┌──────────┐     ┌──────────┐               ││
│  │  │Broker 1  │     │Broker 2  │     │Broker 3  │               ││
│  │  │m5.2xlarge│     │m5.2xlarge│     │m5.2xlarge│               ││
│  │  │2TB EBS   │     │2TB EBS   │     │2TB EBS   │               ││
│  │  └──────────┘     └──────────┘     └──────────┘               ││
│  │                                                                   ││
│  │  ZooKeeper (AWS-managed, 3 nodes, multi-AZ, transparent)        ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  MSK SERVERLESS (no brokers to manage, auto-scales)             ││
│  │                                                                   ││
│  │  ├── No broker selection (AWS handles)                           ││
│  │  ├── Auto-scales partitions and throughput                       ││
│  │  ├── Pay per GB ingested + stored                                ││
│  │  ├── Max retention: 24 hours                                     ││
│  │  └── Best for: variable/bursty workloads                        ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  MSK CONNECT (Managed Kafka Connect)                             ││
│  │                                                                   ││
│  │  ├── Source connectors: Pull data INTO Kafka                    ││
│  │  ├── Sink connectors: Push data OUT of Kafka                    ││
│  │  ├── Auto-scaling workers                                        ││
│  │  └── Pre-built: S3 Sink, Debezium CDC, JDBC, OpenSearch        ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## 💰 Pricing

### MSK Provisioned

| Component | Cost |
|-----------|------|
| kafka.m5.large (2 vCPU, 8GB) | $0.21/hr per broker |
| kafka.m5.xlarge (4 vCPU, 16GB) | $0.42/hr per broker |
| kafka.m5.2xlarge (8 vCPU, 32GB) | $0.84/hr per broker |
| kafka.m5.4xlarge (16 vCPU, 64GB) | $1.68/hr per broker |
| kafka.m5.8xlarge (32 vCPU, 128GB) | $3.36/hr per broker |
| kafka.m5.12xlarge (48 vCPU, 192GB) | $5.04/hr per broker |
| kafka.m5.16xlarge (64 vCPU, 256GB) | $6.72/hr per broker |
| kafka.m5.24xlarge (96 vCPU, 384GB) | $10.08/hr per broker |
| Storage (EBS gp3) | $0.10/GB/month |
| Provisioned throughput | $0.05/MiB/s/month |

**Cost Example — Production cluster:**
```
6 × kafka.m5.4xlarge brokers (3 AZs, 2 per AZ):
= 6 × $1.68/hr × 730 hours = $7,358/month

Storage: 6 × 2TB = 12TB:
= 12,000 GB × $0.10 = $1,200/month

Total: $8,558/month
```

### MSK Serverless

| Component | Cost |
|-----------|------|
| Cluster hours | $0.75/hr per cluster |
| Partition hours | $0.0015/hr per partition |
| Data ingested | $0.10/GB |
| Data stored | $0.023/GB/month |

**Cost Example — Variable workload:**
```
100 partitions, 50GB/day ingested, 24h retention:
Cluster: $0.75 × 730 = $547.50/month
Partitions: 100 × $0.0015 × 730 = $109.50/month
Ingestion: 50 × 30 × $0.10 = $150/month
Storage: 50GB × $0.023 = $1.15/month

Total: $808/month (vs $8,558 for provisioned!)
```

### MSK Connect

| Component | Cost |
|-----------|------|
| MCU (MSK Connect Unit) | $0.11/hr per MCU |
| Workers | 1-8 MCUs per worker, auto-scales |

---

## 📊 Key Service Limits

| Limit | Value |
|-------|-------|
| Max brokers per cluster | 90 (Provisioned) |
| Min brokers | 2 (but 3 recommended for HA) |
| Max storage per broker | 16 TB (EBS) |
| Max partitions per broker | 4,000 (recommended) |
| Max message size | 10 MB (configurable) |
| Max retention | Unlimited (Provisioned), 24h (Serverless) |
| Max clusters per region | 90 |
| Max topics per cluster | No hard limit (partition limit applies) |
| Replication factor | Min 2, Max = broker count |
| MSK Connect max workers | 10 per connector |
| Serverless max partitions | 120 per topic |

---

## 🎯 Real-World Use Cases

### Use Case 1: Real-Time Event Streaming Platform

```
Microservices → MSK Topics → Multiple Consumers
├── Order Service → "orders" topic       ├── Analytics pipeline (S3)
├── Payment Service → "payments" topic   ├── Fraud detection (Flink)
├── User Service → "user-events" topic   ├── Notification service
└── Inventory → "stock-updates" topic    ├── Search indexing (OpenSearch)
                                          └── Data warehouse (Redshift)

Pattern: Event-driven architecture
Benefit: Services decoupled, independent scaling, replay capability
```

**Why MSK:** Full Kafka compatibility, existing Kafka expertise, multi-consumer patterns.

---

### Use Case 2: Change Data Capture (CDC) Pipeline

```
Production Databases    →    MSK (via Debezium)    →    Data Lake
├── MySQL (orders)          Topic: cdc.orders           S3 (Iceberg)
├── PostgreSQL (users)      Topic: cdc.users            Redshift
├── MongoDB (sessions)      Topic: cdc.sessions         OpenSearch
└── Oracle (legacy)         Topic: cdc.legacy           Analytics

Debezium captures:
├── INSERT → {"op": "c", "after": {...}}
├── UPDATE → {"op": "u", "before": {...}, "after": {...}}
└── DELETE → {"op": "d", "before": {...}}
```

**Why MSK:** Debezium is Kafka-native, exactly-once semantics for CDC.

---

### Use Case 3: Log Aggregation & Processing

```
Application Logs (thousands of instances)
├── App Server 1 → Kafka producer → "app-logs" topic
├── App Server 2 → Kafka producer → "app-logs" topic
├── App Server N → Kafka producer → "app-logs" topic

Consumers:
├── Real-time alerting (< 5 sec latency)
├── OpenSearch indexing (for search/dashboards)
├── S3 archival (via MSK Connect S3 Sink)
└── Anomaly detection (ML pipeline)

Why MSK over CloudWatch Logs:
├── Higher throughput (100K+ msg/sec easily)
├── Lower cost at scale (vs $0.50/GB CW ingestion)
├── Multiple consumers from same stream
└── Custom processing logic (not just search)
```

---

### Use Case 4: Real-Time Fraud Detection

```
Payment Events → MSK → Flink (CEP) → Decisions
                         │
                         ├── Rule: >5 transactions in 1 minute → FLAG
                         ├── Rule: Transaction > $5000 from new device → FLAG
                         ├── Rule: Location impossible travel → BLOCK
                         └── ML model scoring (SageMaker endpoint)

Requirements:
├── Latency: < 500ms end-to-end
├── Throughput: 50,000 transactions/second
├── Ordering: Per-customer (partition by customer_id)
├── Exactly-once: Cannot charge twice or miss a fraud
└── 24/7: Zero downtime tolerance
```

---

### Use Case 5: IoT Data Ingestion

```
IoT Devices (millions)  →  IoT Core  →  MSK  →  Processing
├── Sensors                   Rules          ├── Real-time dashboards
├── Vehicles                  Engine         ├── Anomaly detection
├── Smart meters                             ├── Time-series DB
└── Wearables                                └── S3 archive

Why MSK (not Kinesis):
├── Message size: IoT payloads can be > 1MB (Kinesis limit)
├── Retention: Need > 24h for replay (Kinesis limited)
├── Ecosystem: Many IoT platforms have native Kafka support
└── Throughput: Millions of messages/sec per topic
```

---

### Use Case 6: Microservices Event Bus (Domain Events)

```
Domain Events Architecture:
├── Order Domain
│   ├── Publishes: OrderCreated, OrderShipped, OrderCancelled
│   └── Topic: "domain.orders" (partition by order_id)
│
├── Customer Domain
│   ├── Publishes: CustomerRegistered, CustomerUpdated
│   └── Topic: "domain.customers" (partition by customer_id)
│
├── Inventory Domain
│   ├── Publishes: StockUpdated, StockDepleted
│   └── Topic: "domain.inventory" (partition by product_id)
│
└── Consumers (cross-domain):
    ├── Order Service subscribes to: domain.inventory (stock check)
    ├── Notification Service subscribes to: domain.orders (send emails)
    └── Analytics subscribes to: ALL domains (build reports)

Schema Registry (Glue):
├── Enforces backward compatibility
├── Producers register schema before publishing
└── Consumers can evolve independently
```

---

## 🔑 Key Features to Know

### 1. MSK Provisioned vs Serverless

```
┌────────────────────────────────────────────────────────────────────┐
│  Feature          │ Provisioned         │ Serverless              │
├───────────────────┼─────────────────────┼─────────────────────────┤
│ Broker selection  │ You choose          │ AWS manages             │
│ Scaling           │ Manual (add brokers)│ Automatic               │
│ Storage           │ EBS (configurable)  │ Managed                 │
│ Retention         │ Unlimited           │ 24 hours max            │
│ Max throughput    │ Very high (custom)  │ Auto-scales             │
│ Max partitions    │ 4000/broker ×N      │ 120/topic               │
│ Authentication    │ IAM, SASL, TLS      │ IAM only                │
│ Kafka Connect     │ Yes                 │ Yes                     │
│ Custom config     │ Full control        │ Limited                 │
│ Cost model        │ Per-broker-hour     │ Per-GB processed        │
│ Idle cost         │ Yes (always on)     │ Minimal (cluster hour)  │
│ Best for          │ High-volume, 24/7,  │ Variable, dev/test,     │
│                   │ long retention       │ simple use cases        │
└───────────────────┴─────────────────────┴─────────────────────────┘
```

### 2. Authentication Methods

```
1. IAM Access Control (recommended for AWS-native):
   ├── Producer/consumer authenticate via IAM role
   ├── Topic-level permissions via IAM policies
   ├── No passwords to manage
   └── Works with IRSA (EKS pods)

2. SASL/SCRAM (username/password):
   ├── Stored in AWS Secrets Manager
   ├── Per-user access control
   ├── Compatible with existing Kafka clients
   └── Use for: Non-AWS clients, legacy applications

3. Mutual TLS (mTLS):
   ├── Certificate-based authentication
   ├── Client certificate required
   ├── Strongest authentication
   └── Use for: Cross-organization, highest security

4. Unauthenticated (PLAINTEXT):
   ├── No authentication
   ├── Only for development/testing
   └── NEVER in production!
```

### 3. Encryption

```
At Rest:
├── EBS volumes encrypted with KMS (aws/msk or custom CMK)
├── Always enabled (cannot disable)
└── No performance impact (hardware-accelerated)

In Transit:
├── TLS between brokers (inter-broker): Always on
├── TLS between client and broker: Configurable
│   ├── TLS (encrypted): Recommended for production
│   └── PLAINTEXT (unencrypted): Only for dev/high-throughput testing
└── Client → broker TLS has ~5% throughput overhead
```

### 4. MSK Connect (Managed Kafka Connect)

```
Source Connectors (INTO Kafka):
├── Debezium MySQL/PostgreSQL/MongoDB (CDC)
├── JDBC Source (poll-based database extraction)
├── S3 Source (read files from S3)
└── Custom connectors (bring your own JAR)

Sink Connectors (OUT of Kafka):
├── S3 Sink (Parquet/JSON/Avro to S3 — most common!)
├── OpenSearch Sink (index events for search)
├── JDBC Sink (write to databases)
├── Redshift Sink (via S3 staging)
└── Custom connectors

Key configuration:
├── Auto-scaling: 1-8 MCU per worker, 1-10 workers
├── Scale-out at: CPU > 80% or connector lag growing
├── Scale-in at: CPU < 20%
├── Dead letter queue: Failed messages routed to DLQ topic
└── Exactly-once: Supported for Sink connectors
```

### 5. Monitoring (CloudWatch Metrics)

```
CRITICAL METRICS (alert on these):
├── UnderReplicatedPartitions: MUST be 0 (data durability risk!)
├── OfflinePartitionsCount: MUST be 0 (data unavailable!)
├── ActiveControllerCount: MUST be 1 (cluster health)

PERFORMANCE METRICS:
├── BytesInPerSec / BytesOutPerSec (throughput)
├── MessagesInPerSec (message rate)
├── FetchMessageConversionsPerSec (format mismatch — should be 0)
├── ProduceMessageConversionsPerSec (format mismatch)
└── RequestHandlerAvgIdlePercent (< 30% = overloaded!)

CONSUMER METRICS:
├── SumOffsetLag (THE most important consumer metric!)
│   └── Growing = consumer can't keep up = PROBLEM
├── MaxOffsetLag (worst-case partition)
└── EstimatedTimeLag (how far behind in time)

STORAGE METRICS:
├── KafkaDataLogsDiskUsed (% of provisioned storage)
├── BurstBalance (EBS burst credits remaining)
└── PercentDiskUsed (alert at 70%, critical at 85%)
```

### 6. Tiered Storage (New Feature)

```
Without Tiered Storage:
├── All data on EBS (expensive for long retention)
├── 7 days × 1 GB/sec = 605 TB of EBS needed!
└── Cost: 605,000 GB × $0.10 = $60,500/month (just storage!)

With Tiered Storage:
├── Recent data: EBS (fast access) — configured retention
├── Older data: S3 (cheap storage) — extended retention
├── Transparent to consumers (same Kafka API)
├── Cost: ~80% reduction for long-retention topics
└── Infinite retention at S3 cost ($0.023/GB vs $0.10/GB)

Configuration:
├── remote.storage.enable=true (topic-level)
├── local.retention.ms=86400000 (keep 1 day on EBS)
├── retention.ms=-1 (infinite retention on S3)
└── Consumers read from EBS (recent) or S3 (historical) transparently
```

---

## ❓ Interview Questions & Answers

### Q1: When would you choose MSK over Kinesis Data Streams? Give specific scenarios.

**Answer:**

```
CHOOSE MSK WHEN:                         CHOOSE KINESIS WHEN:
├── Existing Kafka expertise             ├── Serverless/zero-ops preferred
├── Message size > 1MB                    ├── Messages < 1MB
├── Need Kafka ecosystem (Connect,       ├── AWS-native integrations
│   Streams, Schema Registry)            │   (Lambda, Firehose, Analytics)
├── Multi-consumer patterns (groups)     ├── Few consumers
├── Long retention (weeks/months)        ├── Short retention (1-7 days)
├── High throughput (GB/sec)             ├── Moderate throughput
├── Need exactly-once (transactions)     ├── At-least-once sufficient
├── Cross-platform (same code on/off AWS)├── AWS-only deployment
├── Debezium CDC integration             ├── Simple event capture
├── Topic compaction needed              ├── Not needed
└── Budget for dedicated brokers         └── Want pay-per-request

SPECIFIC SCENARIOS:

MSK:
1. Migrating on-prem Kafka to AWS (same code, same configs)
2. CDC pipeline with Debezium (Kafka-native connector)
3. Event sourcing with infinite retention + compaction
4. 1M+ messages/sec sustained throughput
5. Multi-team event bus with Schema Registry enforcement

Kinesis:
1. Lambda-triggered event processing (native integration)
2. IoT data → Firehose → S3 (zero code pipeline)
3. Real-time analytics with Kinesis Data Analytics
4. Simple fan-out to 2-3 consumers
5. Startup with no Kafka expertise (lower learning curve)
```

---

### Q2: Your MSK cluster handles 500K messages/sec. Suddenly, consumer lag starts growing for one consumer group. Walk through troubleshooting.

**Answer:**

```
STEP 1: IDENTIFY THE SCOPE
├── Is lag growing for ALL consumer groups or JUST one?
│   ├── ALL groups → Broker-side issue (storage/network/CPU)
│   └── ONE group → Consumer-side issue (processing bottleneck)
├── Is lag growing for ALL partitions or specific ones?
│   ├── ALL partitions → Consumer capacity issue
│   └── SPECIFIC partitions → Data skew or stuck consumer

STEP 2: CHECK CONSUMER HEALTH
```

```bash
# Check consumer group status
kafka-consumer-groups.sh --bootstrap-server $BROKER \
  --describe --group payment-processor

# Output:
# TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG    CONSUMER-ID
# payments 0          1000000         1000500         500    consumer-1
# payments 1          1000000         1005000         5000   consumer-2  ← HIGH LAG!
# payments 2          1000000         1000200         200    consumer-3

# Partition 1 has 10x more lag → either:
# A) Consumer-2 is slow (processing bottleneck)
# B) Partition 1 has more data (skew)
# C) Consumer-2 is dead/disconnected
```

```
STEP 3: DIAGNOSE ROOT CAUSE

Cause A: Consumer processing too slow
├── Symptom: Consumer CPU high, processing time increasing
├── Fix: Increase consumers (add more to group) or optimize processing
├── Fix: Use batch processing (poll multiple messages, process in batch)
└── Fix: Async processing (process → commit offset → don't block)

Cause B: Data skew (one partition gets majority of data)
├── Symptom: Lag only on specific partitions, producer metrics uneven
├── Fix: Better partition key (more evenly distributed)
├── Fix: Increase partition count (spread data more)
└── Diagnosis: Check bytes per partition in producer metrics

Cause C: Consumer crashes/rebalances
├── Symptom: Consumer group state = "Rebalancing" frequently
├── Fix: Increase session.timeout.ms (prevent false timeouts)
├── Fix: Use static group membership (group.instance.id)
├── Fix: Use cooperative sticky assignor (minimize disruption)
└── Diagnosis: Consumer logs show "Member X has been removed"

Cause D: Broker overloaded
├── Symptom: RequestHandlerAvgIdlePercent < 30%
├── Fix: Add more brokers
├── Fix: Increase broker instance type
├── Fix: Move partitions to less-loaded brokers
└── Diagnosis: CloudWatch BytesInPerSec near instance network limit

Cause E: Storage throttling
├── Symptom: BurstBalance depleted, VolumeQueueLength high
├── Fix: Enable provisioned throughput on EBS
├── Fix: Increase storage (more IOPS baseline)
└── Diagnosis: CloudWatch EBS metrics show throttling
```

```python
# QUICK FIX: Scale consumers to match partition count
# If you have 30 partitions but only 5 consumers:
# Each consumer handles 6 partitions → might be overloaded

# Increase to 15 consumers:
# Each handles 2 partitions → 3x less work per consumer

# In consumer configuration:
consumer_config = {
    'group.id': 'payment-processor',
    'max.poll.records': 500,          # Process 500 per poll (reduce if slow)
    'max.poll.interval.ms': 300000,    # 5 min max processing time
    'session.timeout.ms': 45000,       # 45 sec heartbeat timeout
    'fetch.min.bytes': 50000,          # Batch fetching (efficiency)
    'fetch.max.wait.ms': 500,          # Max wait for batch
}
```

---

### Q3: Design an MSK cluster for a system that needs to handle 1 million messages per second with 7-day retention. Provide sizing calculations.

**Answer:**

```
REQUIREMENTS:
├── Throughput: 1,000,000 messages/sec
├── Average message size: 1 KB
├── Retention: 7 days
├── Replication factor: 3
├── Availability: Multi-AZ (3 AZs)

CALCULATION:

Step 1: Data rate
├── Ingestion: 1M msg/sec × 1 KB = 1 GB/sec = 3.6 TB/hour
├── With replication (×3): 3 GB/sec inter-broker traffic
└── Total broker network: ~4 GB/sec (produce + replicate + consume)

Step 2: Storage
├── Daily: 1 GB/sec × 86,400 sec = 86.4 TB/day
├── 7 days: 86.4 × 7 = 604.8 TB (before replication on disk)
├── With replication: 604.8 × 3 = 1,814 TB total (distributed)
├── Per broker (if 12 brokers): 1,814 / 12 = ~151 TB per broker
└── Recommendation: 12 brokers × 16 TB storage each (with growth room)

Step 3: Network capacity
├── Instance network limit: kafka.m5.8xlarge = 10 Gbps = 1.25 GB/sec
├── Per broker traffic: ~4 GB/sec total / 12 brokers = 333 MB/sec per broker
├── Well within 1.25 GB/sec limit ✓
└── But add headroom: kafka.m5.12xlarge for safety (25 Gbps)

Step 4: Partitions
├── Rule: 1 partition ≈ 10 MB/sec producer throughput
├── 1 GB/sec ÷ 10 MB/sec = 100 partitions minimum
├── For consumer parallelism: More partitions = more consumers
├── Recommendation: 300 partitions (3× for growth + parallelism)
└── Per broker: 300 / 12 = 25 partitions (well under 4000 limit)

FINAL CLUSTER DESIGN:
├── Brokers: 12 × kafka.m5.12xlarge (48 vCPU, 192 GB RAM)
├── Distribution: 4 per AZ (3 AZs)
├── Storage: 16 TB EBS gp3 per broker (provisioned IOPS)
├── Partitions per topic: 300 (high-traffic), 30 (low-traffic)
├── Replication factor: 3
├── min.insync.replicas: 2
└── Tiered storage: Enable for cost optimization (old data → S3)

MONTHLY COST:
├── Brokers: 12 × $5.04/hr × 730 = $44,150
├── Storage: 12 × 16,000 GB × $0.10 = $19,200
├── Provisioned throughput: 12 × 250 MB/s × $0.05 = $150
├── Total: ~$63,500/month
├── With tiered storage: ~$45,000/month (30% savings on storage)
└── Per message: $63,500 / (1M × 86400 × 30) = $0.0000000246/message
```

---

### Q4: Explain exactly-once semantics in MSK. How does it work end-to-end and what are the limitations?

**Answer:**

```
EXACTLY-ONCE = Each message processed once and ONLY once

THREE LAYERS REQUIRED:

Layer 1: IDEMPOTENT PRODUCER (prevents duplicate produces)
├── Problem: Producer sends message → broker acks → ack lost in network
│   → Producer retries → DUPLICATE message on broker!
├── Solution: enable.idempotence=true
│   → Broker tracks producer sequence numbers per partition
│   → Duplicate sequences silently dropped
└── Scope: Per producer session, per partition

Layer 2: KAFKA TRANSACTIONS (atomic multi-partition writes)
├── Problem: Write to partition A succeeds, write to B fails
│   → Partial write (inconsistent!)
├── Solution: Transactional producer
│   → beginTransaction() → produce A → produce B → commitTransaction()
│   → Either ALL succeed or ALL fail (atomic)
└── Consumer must use: isolation.level=read_committed
    (only sees committed messages, not in-progress transactions)

Layer 3: CONSUME-TRANSFORM-PRODUCE (EOS pattern)
├── Problem: Consumer reads → processes → produces output
│   → If consumer crashes after produce but before offset commit
│   → Message reprocessed → DUPLICATE output!
├── Solution: Include offset commit IN the transaction
│   → Transaction = (produce output + commit consumer offset)
│   → Atomic: either both happen or neither
└── This is the gold standard for stream processing

LIMITATIONS:
├── Performance: ~3-5% throughput overhead (transaction coordination)
├── Latency: Slightly higher (wait for transaction commit)
├── Complexity: More code, more failure modes to handle
├── Cross-system: Only within Kafka! External sinks need idempotent writes
├── Timeout: Transactions have max timeout (default 1 min)
└── Producer restart: New producer.id → old transactions may hang
    (set transactional.id.expiration.ms appropriately)
```

```python
# COMPLETE EXACTLY-ONCE IMPLEMENTATION:
from confluent_kafka import Consumer, Producer

# Producer (idempotent + transactional)
producer = Producer({
    'bootstrap.servers': BROKERS,
    'enable.idempotence': True,
    'transactional.id': 'eos-processor-001',  # Unique per instance
    'acks': 'all',
    'max.in.flight.requests.per.connection': 5,
})
producer.init_transactions()

# Consumer (read_committed)
consumer = Consumer({
    'bootstrap.servers': BROKERS,
    'group.id': 'eos-group',
    'isolation.level': 'read_committed',  # Only committed messages
    'enable.auto.commit': False,           # We commit in transaction
})
consumer.subscribe(['input-topic'])

# Processing loop (exactly-once)
while True:
    msg = consumer.poll(1.0)
    if msg is None or msg.error():
        continue
    
    try:
        # Begin atomic transaction
        producer.begin_transaction()
        
        # Process message
        result = transform(msg.value())
        
        # Produce output (part of transaction)
        producer.produce('output-topic', value=result, key=msg.key())
        
        # Commit consumer offset (SAME transaction!)
        producer.send_offsets_to_transaction(
            consumer.position(consumer.assignment()),
            consumer.consumer_group_metadata()
        )
        
        # Atomic commit (output + offset)
        producer.commit_transaction()
        
    except Exception as e:
        producer.abort_transaction()
        # offset not committed → message will be reprocessed
        logger.error(f"Transaction aborted: {e}")
```

---

### Q5: Your MSK cluster has 6 brokers and 3 go down simultaneously (AZ failure). What happens and how do you recover?

**Answer:**

```
SCENARIO: 6 brokers (2 per AZ), AZ-b fails → Brokers 3 & 4 down

WHAT HAPPENS IMMEDIATELY:
├── Partitions with leaders on Brokers 3/4: Leader election happens
│   ├── If replica on Broker 1,2,5,6 is IN-SYNC → new leader elected (seconds)
│   ├── If NO in-sync replica → partition OFFLINE (data unavailable!)
│   └── With replication=3 across 3 AZs: Usually at least one replica survives
├── Producers:
│   ├── acks=all with min.insync.replicas=2:
│   │   └── Partitions with < 2 ISR → REJECT writes (data safety!)
│   └── acks=1: Writes continue to new leaders (may lose uncommitted)
└── Consumers:
    ├── Consumer group rebalances (partitions reassigned)
    └── Processing continues from surviving brokers

CONFIGURATION FOR AZ RESILIENCE:
```

```properties
# Cluster configuration (CRITICAL for AZ failure survival):
min.insync.replicas=2
default.replication.factor=3
unclean.leader.election.enable=false  # NEVER promote out-of-sync replica!

# These ensure:
# - Data replicated to 3 brokers (ideally in 3 AZs)
# - At least 2 must acknowledge writes (tolerates 1 AZ loss)
# - Never elect a lagging replica as leader (prevents data loss)
```

```
RECOVERY PROCESS:

Automatic (MSK handles):
├── Detects broker failure (health check)
├── Replaces failed brokers in same AZ (when AZ recovers)
├── New brokers rejoin cluster
├── Under-replicated partitions replicate to new brokers
└── Cluster returns to full health (minutes to hours depending on data)

Manual intervention (if prolonged AZ outage):
├── Option A: Wait for AZ recovery (AWS usually restores within hours)
├── Option B: Add brokers in surviving AZs (expand cluster)
│   ├── aws kafka update-broker-count --target 8 (add 2 more)
│   ├── Reassign partitions to new brokers
│   └── Takes time (data replication)
└── Option C: If data loss occurred (unlikely with proper replication)
    ├── Reset consumer offsets to last known good position
    ├── Replay from source systems if needed
    └── Investigate why ISR was insufficient

PREVENTION (Design for AZ failure):
├── ALWAYS 3+ brokers across 3 AZs (minimum)
├── replication.factor=3 (one replica per AZ)
├── min.insync.replicas=2 (survive 1 AZ loss)
├── rack.awareness: broker.rack=AZ-ID (ensures replicas span AZs)
└── Test: Regularly simulate AZ failure (chaos engineering)
```

---

### Q6: How do you migrate from a self-managed Kafka cluster (on EC2 or on-prem) to MSK with zero downtime?

**Answer:**

```
MIGRATION STRATEGY: MirrorMaker 2 (MM2) — Live Replication

┌──────────────┐    MirrorMaker 2    ┌──────────────┐
│ Source Kafka │ ──────────────────▶ │ Target MSK   │
│ (on-prem/EC2)│    (real-time       │ (new cluster)│
│              │     replication)    │              │
│ Topics:      │                    │ Topics:      │
│ orders       │ ──── mirrors ────▶ │ orders       │
│ payments     │ ──── mirrors ────▶ │ payments     │
│ events       │ ──── mirrors ────▶ │ events       │
└──────────────┘                    └──────────────┘
       ▲                                   ▲
       │ (producers write here)            │ (after cutover, producers write here)
       │                                   │
  [Consumers]                         [Consumers]
  (read from source)                  (switch to MSK after validation)

MIGRATION STEPS:

Phase 1: Setup (Days 1-3)
├── Create MSK cluster matching source specs
├── Configure same topics (partitions, retention, configs)
├── Deploy MirrorMaker 2 (MSK Connect or standalone)
├── Start replication (source → MSK)
└── Verify: Messages appearing in MSK, lag minimal

Phase 2: Validation (Days 4-10)
├── Run consumers against MSK (shadow/read-only)
├── Compare message counts: source offset vs MSK offset
├── Verify: Schema compatibility, message format
├── Test: Consumer logic works identically on MSK
└── Monitor: Replication lag < 100ms consistently

Phase 3: Consumer migration (Days 11-15)
├── Switch consumers one-by-one to read from MSK
├── Start with non-critical consumers first
├── Verify processing is correct (no duplicates, no gaps)
├── Monitor: Consumer lag on MSK cluster
└── Rollback plan: Switch consumer back to source if issues

Phase 4: Producer migration (Days 16-20)
├── Switch producers one-by-one to write to MSK
├── Verify: Messages appearing in MSK topics
├── MirrorMaker 2 still running (catches any stragglers)
├── Dual-write period: Some producers on source, some on MSK
└── After all producers migrated → disable MM2

Phase 5: Decommission (Day 21+)
├── Stop MirrorMaker 2
├── Verify all consumers reading from MSK only
├── Keep source cluster for 7 days (rollback safety)
├── Decommission source cluster
└── Update DNS/connection strings to MSK endpoints
```

```python
# MirrorMaker 2 configuration (via MSK Connect)
mm2_config = {
    "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "clusters": "source,target",
    "source.cluster.alias": "source",
    "target.cluster.alias": "target",
    "source.cluster.bootstrap.servers": "source-broker-1:9092,source-broker-2:9092",
    "target.cluster.bootstrap.servers": "msk-broker-1:9098,msk-broker-2:9098",
    
    # What to replicate
    "topics": "orders,payments,events,user-activity",  # Or ".*" for all
    "groups": ".*",  # Replicate consumer group offsets too!
    
    # Replication settings
    "replication.factor": "3",
    "sync.topic.configs.enabled": "true",   # Copy topic configs
    "sync.topic.acls.enabled": "false",      # MSK uses IAM, not ACLs
    "emit.checkpoints.enabled": "true",      # Track replication progress
    "emit.heartbeats.enabled": "true",
    
    # Performance
    "tasks.max": "10",
    "producer.max.request.size": "10485760",  # 10MB max message
    "offset-syncs.topic.replication.factor": "3"
}
```

---

### Q7: Explain MSK topic compaction. When would you use it and how does it work?

**Answer:**

```
TOPIC COMPACTION = Keep only the LATEST value for each key

Normal retention (delete):
[K:A,V:1] [K:B,V:2] [K:A,V:3] [K:C,V:4] [K:B,V:5]
After retention expires: (all deleted)

Compacted retention:
[K:A,V:1] [K:B,V:2] [K:A,V:3] [K:C,V:4] [K:B,V:5]
After compaction:
                      [K:A,V:3] [K:C,V:4] [K:B,V:5]
Only LATEST value per key survives!

Key insight: Compacted topics = infinite retention of CURRENT STATE
├── Every unique key has at least its latest value preserved
├── Old values for same key are garbage-collected
├── New consumers see FULL current state (like a database snapshot)
└── Storage bounded by: number_of_keys × latest_value_size

WHEN TO USE COMPACTED TOPICS:
├── Changelogs (database state as events)
│   └── Key = row primary key, Value = current row state
├── Configuration distribution
│   └── Key = config name, Value = current config value
├── User profiles (keep latest state)
│   └── Key = user_id, Value = latest profile JSON
├── Inventory levels
│   └── Key = product_id, Value = current stock count
└── Session state (keep active sessions)
    └── Key = session_id, Value = session data (null = deleted)

HOW DELETION WORKS (Tombstones):
├── To "delete" a key: Produce message with key=X, value=null
├── This is a TOMBSTONE (marker for deletion)
├── After compaction, key X is removed entirely
└── Consumers that read from beginning won't see deleted keys
```

```python
# Create compacted topic
# kafka-topics.sh --create --topic user-profiles \
#   --partitions 30 \
#   --replication-factor 3 \
#   --config cleanup.policy=compact \
#   --config min.cleanable.dirty.ratio=0.5 \
#   --config delete.retention.ms=86400000 \
#   --config segment.ms=604800000

# Use case: Distribute user profile changes to all services
producer.produce(
    topic='user-profiles',
    key=user_id.encode(),            # Compaction key
    value=json.dumps(user_data),     # Latest state
)

# Consumer reads from beginning → gets FULL current state of all users
# Then continues to get real-time updates
consumer.subscribe(['user-profiles'])
# initial_position = EARLIEST → reads all current profiles (snapshot)
# then continues streaming new changes (live updates)
```

---

### Q8: How do you implement dead letter queues (DLQ) in MSK for handling poison pill messages?

**Answer:**

```
PROBLEM: "Poison pill" = message that crashes consumer repeatedly
├── Consumer reads message → processing fails → no commit
├── Consumer re-reads same message → fails again → infinite loop!
├── Entire consumer group stuck on one bad message
└── Result: ALL messages behind the poison pill are delayed

SOLUTION: Dead Letter Queue pattern

┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Input   │────▶│  Consumer    │────▶│  Output      │
│  Topic   │     │  (processes) │     │  Topic       │
│          │     │              │     │              │
│          │     │  IF FAILS:   │     │              │
│          │     │  retry 3x    │     │              │
│          │     │  then → DLQ  │     │              │
└──────────┘     └──────┬───────┘     └──────────────┘
                        │
                        ▼ (failed messages)
                 ┌──────────────┐
                 │  DLQ Topic   │ → Alert → Manual review
                 │  (dead.letter│     or
                 │   .queue)    │ → Auto-retry after fix
                 └──────────────┘
```

```python
import json
import time
from confluent_kafka import Consumer, Producer, KafkaError

class ResilientConsumer:
    """Consumer with dead letter queue for poison pill handling."""
    
    def __init__(self, input_topic, dlq_topic, max_retries=3):
        self.consumer = Consumer({
            'bootstrap.servers': BROKERS,
            'group.id': 'payment-processor',
            'enable.auto.commit': False,
            'max.poll.interval.ms': 300000,
        })
        self.producer = Producer({'bootstrap.servers': BROKERS})
        self.input_topic = input_topic
        self.dlq_topic = dlq_topic
        self.max_retries = max_retries
        self.consumer.subscribe([input_topic])
    
    def process_messages(self):
        while True:
            msg = self.consumer.poll(1.0)
            if msg is None:
                continue
            if msg.error():
                if msg.error().code() == KafkaError._PARTITION_EOF:
                    continue
                raise Exception(msg.error())
            
            # Try processing with retries
            success = self._process_with_retry(msg)
            
            if success:
                self.consumer.commit(msg)
            else:
                # Send to DLQ after all retries exhausted
                self._send_to_dlq(msg)
                self.consumer.commit(msg)  # Commit to move past poison pill!
    
    def _process_with_retry(self, msg, attempt=0):
        """Retry with exponential backoff."""
        try:
            # Your business logic here
            process_payment(json.loads(msg.value()))
            return True
        except RetryableError as e:
            if attempt < self.max_retries:
                time.sleep(min(2 ** attempt, 30))  # Exponential backoff
                return self._process_with_retry(msg, attempt + 1)
            return False
        except NonRetryableError as e:
            # Don't retry (bad data, schema error, etc.)
            return False
    
    def _send_to_dlq(self, original_msg):
        """Send failed message to dead letter queue with metadata."""
        dlq_message = {
            'original_topic': original_msg.topic(),
            'original_partition': original_msg.partition(),
            'original_offset': original_msg.offset(),
            'original_key': original_msg.key().decode() if original_msg.key() else None,
            'original_value': original_msg.value().decode(),
            'failure_reason': str(self._last_error),
            'failure_time': time.time(),
            'retry_count': self.max_retries,
            'consumer_group': 'payment-processor'
        }
        
        self.producer.produce(
            topic=self.dlq_topic,
            key=original_msg.key(),
            value=json.dumps(dlq_message).encode(),
            headers=[('error-type', str(type(self._last_error).__name__).encode())]
        )
        self.producer.flush()
        
        # Alert on DLQ messages
        alert(f"Message sent to DLQ: {original_msg.topic()}:{original_msg.partition()}:{original_msg.offset()}")

# DLQ monitoring and replay
class DLQManager:
    """Monitor DLQ and replay messages after fixes."""
    
    def replay_messages(self, dlq_topic, target_topic, filter_func=None):
        """Replay DLQ messages back to original topic."""
        consumer = Consumer({
            'bootstrap.servers': BROKERS,
            'group.id': 'dlq-replayer',
            'auto.offset.reset': 'earliest'
        })
        producer = Producer({'bootstrap.servers': BROKERS})
        consumer.subscribe([dlq_topic])
        
        while True:
            msg = consumer.poll(1.0)
            if msg is None:
                break
            
            dlq_data = json.loads(msg.value())
            
            # Optional filter (only replay certain errors)
            if filter_func and not filter_func(dlq_data):
                continue
            
            # Replay to original topic
            producer.produce(
                topic=target_topic,
                key=dlq_data['original_key'].encode() if dlq_data['original_key'] else None,
                value=dlq_data['original_value'].encode()
            )
        
        producer.flush()
```

---

### Q9: How do you handle schema evolution in MSK without breaking consumers?

**Answer:**

```
PROBLEM: Producer changes message format → consumers break!

Example:
V1: {"user_id": "123", "name": "John", "email": "john@x.com"}
V2: {"userId": "123", "fullName": "John Doe", "email": "john@x.com", "phone": "555-0123"}
   ↑ field renamed      ↑ field renamed                              ↑ new field

Consumer expecting V1 format: CRASHES on V2!

SOLUTION: Schema Registry + Compatibility Rules
```

```python
# AWS Glue Schema Registry with MSK

import boto3

glue = boto3.client('glue')

# Step 1: Create registry
glue.create_registry(
    RegistryName='msk-schemas',
    Description='Schemas for MSK topics'
)

# Step 2: Register schema with BACKWARD compatibility
glue.create_schema(
    RegistryName='msk-schemas',
    SchemaName='user-events',
    DataFormat='AVRO',
    Compatibility='BACKWARD',  # New schema MUST read old data
    SchemaDefinition=json.dumps({
        "type": "record",
        "name": "UserEvent",
        "namespace": "com.company.events",
        "fields": [
            {"name": "user_id", "type": "string"},
            {"name": "name", "type": "string"},
            {"name": "email", "type": "string"}
        ]
    })
)

# Step 3: Evolve schema (add optional field — backward compatible!)
glue.register_schema_version(
    SchemaId={'RegistryName': 'msk-schemas', 'SchemaName': 'user-events'},
    SchemaDefinition=json.dumps({
        "type": "record",
        "name": "UserEvent",
        "namespace": "com.company.events",
        "fields": [
            {"name": "user_id", "type": "string"},
            {"name": "name", "type": "string"},
            {"name": "email", "type": "string"},
            {"name": "phone", "type": ["null", "string"], "default": None}
            #                      ↑ OPTIONAL (union with null + default)
            # This is BACKWARD COMPATIBLE:
            # Old consumers ignore the new field
            # New consumers handle null for old messages
        ]
    })
)

# Step 4: Try to register INCOMPATIBLE change → REJECTED!
try:
    glue.register_schema_version(
        SchemaId={'RegistryName': 'msk-schemas', 'SchemaName': 'user-events'},
        SchemaDefinition=json.dumps({
            "type": "record",
            "name": "UserEvent",
            "fields": [
                {"name": "userId", "type": "string"},  # RENAMED! (breaks old consumers)
                {"name": "name", "type": "string"},
            ]
        })
    )
except glue.exceptions.InvalidInputException:
    print("Schema rejected! Not backward compatible.")
    # Registry PREVENTS breaking changes from being deployed!
```

```
COMPATIBILITY MODES:

BACKWARD (recommended for most cases):
├── New schema can READ old data
├── Allowed: Add optional fields, widen types
├── Blocked: Remove fields, rename fields, narrow types
├── Meaning: Deploy new consumers FIRST, then new producers
└── Example: Adding "phone" as optional field

FORWARD:
├── Old schema can READ new data
├── Allowed: Remove optional fields, narrow types
├── Meaning: Deploy new producers FIRST, then new consumers
└── Example: Removing a deprecated field

FULL:
├── Both backward AND forward compatible
├── Most restrictive (only add/remove optional fields)
└── Safest but least flexible

NONE:
├── No compatibility checking
├── Any change allowed
├── DANGEROUS: Will break consumers silently
└── Use ONLY for development/testing
```

---

### Q10: Your MSK Connect S3 Sink is creating thousands of small files. How do you configure it for optimal data lake performance?

**Answer:**

```
PROBLEM: Default MSK Connect S3 Sink creates files too frequently
├── Default flush.size=1000 → one file per 1000 records
├── High-volume topic (100K msg/sec) → 100 files/second!
├── Result: Millions of tiny files in S3 → Athena/Spark scans are slow

SOLUTION: Configure for larger, less-frequent files
```

```json
{
  "connector.class": "io.confluent.connect.s3.S3SinkConnector",
  "tasks.max": "10",
  "topics": "order-events,payment-events,user-activity",
  
  "s3.bucket.name": "company-datalake-bronze",
  "s3.region": "us-east-1",
  
  "format.class": "io.confluent.connect.s3.format.parquet.ParquetFormat",
  "parquet.codec": "snappy",
  
  "flush.size": "500000",
  "rotate.interval.ms": "900000",
  "rotate.schedule.interval.ms": "900000",
  
  "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
  "path.format": "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH",
  "locale": "en-US",
  "timezone": "UTC",
  "partition.duration.ms": "3600000",
  "timestamp.extractor": "RecordField",
  "timestamp.field": "event_time",

  "storage.class": "io.confluent.connect.s3.storage.S3Storage",
  "schema.compatibility": "BACKWARD",
  
  "value.converter": "io.confluent.connect.avro.AvroConverter",
  "value.converter.schema.registry.url": "https://glue-schema-registry",
  
  "behavior.on.null.values": "ignore",
  "errors.tolerance": "all",
  "errors.deadletterqueue.topic.name": "dlq-s3-sink",
  "errors.deadletterqueue.topic.replication.factor": "3"
}
```

```
KEY CONFIGURATION EXPLAINED:

flush.size = 500000
├── Write file after 500K records (not default 1000!)
├── At 100K msg/sec: file every 5 seconds (still frequent)
└── Produces ~50-200MB files (good for Athena)

rotate.interval.ms = 900000 (15 minutes)
├── Even if flush.size not reached, rotate file every 15 min
├── Ensures data freshness (max 15 min delay)
└── Balance: freshness vs file size

rotate.schedule.interval.ms = 900000
├── Scheduled rotation (wall clock time)
├── Ensures aligned file boundaries (all files end at :00, :15, :30, :45)
└── Easier for downstream partition management

partitioner = TimeBasedPartitioner
├── path.format: Creates S3 paths like year=2024/month=06/day=15/hour=10/
├── Matches Hive-style partitioning (Athena/Glue compatible)
├── partition.duration.ms: New partition every hour
└── Combined with format: Parquet files in time-based directories

RESULT:
├── Before: 100 files/second × 3600s = 360,000 files/hour (tiny, slow scans!)
├── After: ~4 files/hour per partition (large, fast scans!)
└── Athena query speed: 10x improvement
    S3 request cost: 99% reduction
```

```
ADDITIONAL OPTIMIZATION — Post-ingestion compaction:

Even with good S3 Sink config, run periodic compaction:

Glue Job (daily):
├── Read all files from bronze/order-events/hour=XX/
├── Coalesce into 128-256MB files
├── Write to silver/order-events/ (Iceberg table)
├── Delete small files from bronze (after verification)
└── Result: Optimal file sizes for analytics queries

Or use Iceberg auto-compaction:
CALL glue_catalog.system.rewrite_data_files(
    table => 'bronze.order_events',
    strategy => 'binpack',
    options => map('target-file-size-bytes', '134217728')
);
```

---

## 🆚 MSK vs Competitors

| Feature | MSK Provisioned | MSK Serverless | Kinesis DS | Confluent Cloud | Redpanda |
|---------|----------------|----------------|-----------|----------------|----------|
| Protocol | Kafka native | Kafka native | AWS SDK | Kafka native | Kafka compatible |
| Management | Semi-managed | Fully managed | Fully managed | Fully managed | Self/managed |
| Scaling | Manual (brokers) | Automatic | On-demand shards | Automatic | Manual |
| Max retention | Unlimited | 24 hours | 365 days | Unlimited | Unlimited |
| Max throughput | Very high | Auto | Per-shard | Very high | Very high |
| Connect | MSK Connect | MSK Connect | Firehose | Confluent Connect | Custom |
| Schema Registry | Glue SR | Glue SR | None | Confluent SR | Built-in |
| Exactly-once | Yes | Yes | No | Yes | Yes |
| Multi-region | MirrorMaker | No | No | Cluster Linking | No |
| Cost model | Per-broker-hr | Per-GB | Per-shard-hr | Per-CKU | Per-node |
| Best for | Kafka expertise, high-vol | Variable workloads | AWS-native, simple | Multi-cloud | Performance |

---

## 🏆 Production Best Practices

```yaml
cluster_design:
  - Minimum 3 brokers across 3 AZs (HA baseline)
  - replication.factor=3 (data on 3 different AZs)
  - min.insync.replicas=2 (tolerate 1 AZ failure)
  - unclean.leader.election.enable=false (prevent data loss)
  - Enable rack awareness (broker.rack = AZ ID)
  - Size for peak throughput + 30% headroom

topic_design:
  - Partition count: 3× max expected consumer instances
  - Partition key: Natural business key (order_id, customer_id)
  - NEVER reduce partitions (only increase)
  - Retention: 7 days default, longer for replay/audit
  - Compaction: For state/changelog topics
  - Naming: domain.entity.event (e.g., payments.orders.created)

producer_best_practices:
  - Always set message key (ordering guarantee)
  - acks=all for critical data (durability)
  - enable.idempotence=true (prevent duplicates)
  - Batch messages: linger.ms=5, batch.size=32768
  - Compress: compression.type=lz4 (fast) or zstd (smallest)
  - Retries: Integer.MAX_VALUE with idempotent producer

consumer_best_practices:
  - Manual offset commit (after processing)
  - Idempotent processing (handle duplicates)
  - session.timeout.ms=45000 (avoid false rebalances)
  - cooperative sticky assignor (minimize rebalance impact)
  - Dead letter queue for poison pills
  - Monitor consumer lag (alert on growth)

security:
  - IAM authentication (preferred for AWS workloads)
  - TLS encryption in-transit (always enable)
  - KMS encryption at-rest (enabled by default)
  - Private subnets only (no public access)
  - VPC endpoints for cross-VPC access
  - Separate security groups for producers/consumers

monitoring:
  - CRITICAL: UnderReplicatedPartitions = 0
  - CRITICAL: OfflinePartitionsCount = 0
  - WARNING: Consumer SumOffsetLag growing
  - WARNING: BrokerStorageUtilization > 70%
  - INFO: BytesInPerSec / BytesOutPerSec (capacity)
  - ALERT: RequestHandlerAvgIdlePercent < 30%

operations:
  - Enable Tiered Storage for cost optimization (long retention)
  - Use MSK Connect for standard source/sink integrations
  - Schema Registry for all topics (prevent breaking changes)
  - Regular partition reassignment (balance after scaling)
  - Broker rolling upgrades (zero-downtime version upgrades)
  - Backup: MirrorMaker 2 to DR region
```
