# Amazon EMR — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | Details |
|-----------|---------|
| **Service Name** | Amazon EMR (Elastic MapReduce) |
| **Category** | Analytics / Big Data Processing |
| **Type** | Managed Hadoop/Spark Cluster Platform |
| **Engines** | Spark, Hive, Presto/Trino, HBase, Flink, Pig, Tez |
| **Launched** | April 2009 |
| **Pricing Model** | Per-instance-hour (EC2 + EMR surcharge) |
| **Key Differentiator** | Full control of big data frameworks on managed infrastructure |

---

## 🏗️ What Amazon EMR Does

EMR is a **managed cluster platform** that runs open-source big data frameworks (Spark, Hadoop, Hive, Presto, Flink) on AWS infrastructure. You get full framework access with AWS handling provisioning, configuration, and tuning.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AMAZON EMR                                    │
│                                                                       │
│  Deployment Options:                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐ │
│  │  EMR on EC2     │  │  EMR on EKS     │  │  EMR Serverless     │ │
│  │  (Traditional)  │  │  (Kubernetes)   │  │  (No cluster mgmt)  │ │
│  │                 │  │                 │  │                     │ │
│  │  Full control   │  │  Share K8s      │  │  Submit job →       │ │
│  │  Custom config  │  │  resources      │  │  auto-provisions    │ │
│  │  Long-running   │  │  Pod isolation  │  │  Pay per vCPU-hour  │ │
│  │  or transient   │  │  Multi-tenant   │  │  Scales to zero     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────┘ │
│                                                                       │
│  Frameworks Available:                                                │
│  ┌─────────┐ ┌──────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌──────────┐ │
│  │ Apache  │ │ Apache│ │ Presto│ │ Apache│ │ Apache│ │ Apache   │ │
│  │ Spark   │ │ Hive  │ │ Trino │ │ Flink │ │ HBase │ │ Hadoop   │ │
│  │ (ETL,ML)│ │ (SQL) │ │ (SQL) │ │(Stream│ │(NoSQL)│ │(MapReduce│ │
│  └─────────┘ └──────┘ └───────┘ └───────┘ └───────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### EMR on EC2 — Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  EMR CLUSTER                                                     │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  MASTER NODE (1 or 3 for HA)                                │ │
│  │  ├── YARN Resource Manager (assigns containers to nodes)    │ │
│  │  ├── HDFS NameNode (tracks file locations)                  │ │
│  │  ├── Spark Driver (coordinates your job)                   │ │
│  │  ├── Hive Metastore (table definitions)                    │ │
│  │  ├── Ganglia / Spark History Server (monitoring)           │ │
│  │  └── CANNOT run on Spot (single point of failure)          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  CORE NODES (2+ nodes)                                      │ │
│  │  ├── YARN NodeManager (runs task containers)               │ │
│  │  ├── HDFS DataNode (stores data blocks)                    │ │
│  │  ├── Spark Executors (process data)                        │ │
│  │  ├── Data is STORED here (losing core = data loss!)        │ │
│  │  └── On-Demand recommended (if using HDFS)                 │ │
│  │      Spot OK if using S3 only (no local HDFS data)        │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  TASK NODES (0+ nodes, optional)                            │ │
│  │  ├── YARN NodeManager only (compute, NO storage)           │ │
│  │  ├── Spark Executors (process data)                        │ │
│  │  ├── NO HDFS data stored (safe to lose)                    │ │
│  │  ├── 100% Spot instances (70% cost savings!)              │ │
│  │  └── Auto-scales based on YARN metrics                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  Storage Options:                                                 │
│  ├── HDFS (on Core nodes): Fast, temporary, lost on termination │
│  ├── S3 (via EMRFS): Persistent, scalable, decoupled           │
│  ├── EBS volumes: Attached to nodes, persists across restarts   │
│  └── Instance Store: Fastest but ephemeral (NVMe SSD)           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💰 Pricing

### EMR on EC2 Pricing

| Component | Cost |
|-----------|------|
| EC2 instance cost | Standard EC2 pricing |
| EMR surcharge | ~15-25% on top of EC2 (varies by instance) |
| EBS storage | Standard EBS pricing |
| S3 (if used) | $0.023/GB/month + requests |

**Example cluster cost:**
```
Master: 1 × m5.4xlarge On-Demand = $0.768/hr + $0.192 EMR = $0.96/hr
Core:   4 × r5.4xlarge On-Demand = $1.008/hr + $0.252 EMR = $5.04/hr
Task:   8 × r5.4xlarge Spot (70% off) = $0.302/hr + $0.252 EMR = $4.43/hr

Total hourly: $0.96 + $5.04 + $4.43 = $10.43/hr
8-hour daily job: $83.44/day = $2,503/month
```

### EMR Serverless Pricing

| Component | Cost |
|-----------|------|
| vCPU-hour | $0.052624/vCPU-hour |
| Memory-hour | $0.0057785/GB-hour |
| Storage-hour | $0.000111/GB-hour |

**Example:** Job using 40 vCPU and 160 GB memory for 1 hour:
```
Compute: 40 × $0.052624 = $2.10
Memory: 160 × $0.0057785 = $0.92
Total per run: $3.02
Daily: $3.02 × 1 = $3.02/day = $90.66/month
```

### EMR on EKS Pricing

| Component | Cost |
|-----------|------|
| EMR surcharge | Same rate as EMR on EC2 |
| EKS pods | Your existing EKS cluster cost |
| Benefit | Share infrastructure with other K8s workloads |

---

## 📊 Key Service Limits

| Limit | Default | Adjustable? |
|-------|---------|-------------|
| Max nodes per cluster | No hard limit (EC2 limits apply) | Yes |
| Max clusters per region | 500 | Yes |
| Max instance groups per cluster | 50 | No |
| Max instances per instance fleet | 5 instance types | No |
| Max bootstrap actions | 16 | No |
| Max steps per cluster | 256 (pending) | No |
| Max concurrent steps | 1 (default), configurable to 10 | Yes |
| EMR Serverless max workers | 1500 (default) | Yes |
| EMR Serverless max vCPUs | 6000 (default) | Yes |

---

## 🎯 Real-World Use Cases

### Use Case 1: Large-Scale ETL (Petabyte Processing)

```
Data Lake (S3, 50TB/day)  →  EMR Spark  →  Processed Data (S3)
├── Raw JSON/CSV               Transform     ├── Parquet (Silver)
├── Streaming events           Clean         ├── Iceberg tables
└── Database exports           Aggregate     └── Aggregates (Gold)

Cluster: 1 Master + 10 Core + 40 Task (Spot)
Runtime: 4 hours daily
```

**When to use EMR over Glue:** Data > 50TB, job > 4 hours, need custom Spark config, specific library versions, or non-Spark frameworks.

---

### Use Case 2: Machine Learning at Scale

```
Training Data (S3, 10TB)  →  EMR Spark  →  Model + Features
                              ├── Feature engineering (Spark MLlib)
                              ├── Distributed training (SparkML / Horovod)
                              ├── Hyperparameter tuning (parallel jobs)
                              └── Feature store population

Why EMR: Custom library versions, GPU instances, long-running notebooks
```

---

### Use Case 3: Interactive SQL Analytics (Presto/Trino)

```
Data Lake (S3)  →  EMR Presto/Trino  →  Analyst SQL Queries
                    (always-on cluster)     (sub-second for small)
                                            (seconds for large)

Advantage over Athena:
├── Custom UDFs
├── Cross-source federation (S3 + MySQL + PostgreSQL in one query)
├── Finer caching control
└── No per-query cost (only cluster cost)
```

**When to use over Athena:** Consistent high-volume queries, federation needed, custom functions, dedicated compute.

---

### Use Case 4: Real-Time Stream Processing (Flink on EMR)

```
MSK (Kafka)  →  EMR Flink  →  Real-time output
                  ├── Windowed aggregations (every 5 sec)
                  ├── Pattern detection (fraud)
                  ├── Enrichment (join with reference data)
                  └── Write to DynamoDB/OpenSearch/S3

Why EMR Flink (vs Managed Flink): Custom connectors, complex state management, existing Flink expertise.
```

---

### Use Case 5: Data Lake Table Maintenance (Iceberg/Delta)

```
Iceberg Tables (S3)  →  EMR Spark  →  Optimized Tables
                          ├── Compaction (merge small files)
                          ├── Snapshot expiration (cleanup)
                          ├── Orphan file removal
                          ├── Sort order optimization
                          └── Partition evolution

Daily maintenance job on schedule (transient cluster).
```

---

### Use Case 6: Genomics / Scientific Computing

```
Genome Sequencing Data (TB)  →  EMR Spark + GATK  →  Analysis Results
                                  ├── Variant calling
                                  ├── Quality scoring
                                  ├── Population analysis
                                  └── Statistical modeling

Why EMR: Custom bioinformatics libraries, high-memory instances (r5.24xlarge), temporary burst compute.
```

---

## 🔑 Key Features to Know

### 1. Instance Fleets vs Instance Groups

```
INSTANCE GROUPS (Older, simpler):
├── One instance type per group
├── Master group: 1 × m5.xlarge
├── Core group: 5 × r5.4xlarge
├── Task group: 10 × r5.4xlarge
└── Limited flexibility for Spot availability

INSTANCE FLEETS (Newer, recommended):
├── Multiple instance types per fleet (up to 5)
├── Master fleet: [m5.xl, m5.2xl, m6g.xl] — picks cheapest available
├── Core fleet: [r5.4xl, r5a.4xl, r6g.4xl, m5.4xl] — diversified
├── Task fleet: [r5.4xl, r5a.4xl, r5d.4xl, r6g.4xl, m5.4xl]
├── Capacity: Specify total vCPUs/memory, not instance count
└── Much better Spot availability (more options = less interruption)
```

### 2. Managed Scaling (Auto-Scale)

```
EMR Managed Scaling automatically adjusts cluster size:

Configuration:
├── Minimum units: 10 (never go below)
├── Maximum units: 100 (never exceed)
├── Unit type: InstanceFleetUnits or Instances or vCPU
├── On-Demand limit: 20 (cap on-demand, rest is Spot)
└── Scale-down behavior: Graceful (waits for tasks to complete)

How it works:
├── Monitors YARN metrics (pending containers, memory utilization)
├── Scales OUT: When containers are pending for > 5 seconds
├── Scales IN: When resources are idle for > 5 minutes
└── Respects On-Demand/Spot split
```

### 3. EMR Steps (Job Orchestration)

```
Steps = Sequential jobs submitted to a running cluster

Cluster Launch
  │
  ├── Step 1: spark-submit etl_job.py
  │     └── On success → continue
  ├── Step 2: spark-submit aggregation_job.py
  │     └── On success → continue
  ├── Step 3: spark-submit export_job.py
  │     └── On success → continue
  └── Cluster Termination (if auto-terminate enabled)

Step properties:
├── ActionOnFailure: TERMINATE_CLUSTER | CANCEL_AND_WAIT | CONTINUE
├── Can be added to running cluster (via API)
└── Max 10 concurrent steps (configured)
```

### 4. Security Features

```
Authentication & Access:
├── Kerberos: Cross-realm trust with Active Directory
├── Lake Formation: Fine-grained column/row access
├── IAM Roles: Per-application roles (EMRFS + Instance Profile)
└── Security Configurations: Encrypt at rest + in transit

Encryption:
├── At-rest: EBS encryption (KMS), S3 encryption (SSE-KMS)
├── In-transit: TLS for Spark shuffle, HDFS encryption
├── LUKS encryption for local disks
└── Certificate management via AWS PCA

Network:
├── VPC deployment (always in private subnet)
├── Security groups (separate for master/core/task)
├── No public IPs (access via bastion or SSM)
└── Interface VPC endpoints for S3 and other services
```

### 5. EMR Notebooks / EMR Studio

```
EMR Studio = Managed Jupyter Notebook IDE for EMR

Features:
├── Jupyter notebooks attached to EMR clusters
├── Multiple users, multiple clusters
├── Git integration (save notebooks to repos)
├── Persistent workspace (survives cluster termination)
├── Supports: PySpark, Spark SQL, Python, R
└── Access control via IAM Identity Center

Use cases:
├── Interactive data exploration
├── ML model development
├── Ad-hoc analysis on large datasets
├── Collaborative development
└── Job prototyping before production
```

---

## ❓ Interview Questions & Answers

### Q1: When would you choose EMR over Glue for Spark workloads? Give specific scenarios.

**Answer:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Choose EMR when:                     │  Choose Glue when:            │
├───────────────────────────────────────┼───────────────────────────────┤
│ Job runtime > 4 hours                 │ Job runtime < 4 hours         │
│ Data > 50TB per job                   │ Data < 50TB per job           │
│ Need specific Spark version           │ Latest Glue version is fine   │
│ Need custom Spark config tuning       │ Default config works          │
│ Need non-Spark (Flink, Presto, HBase) │ Only need Spark               │
│ Need interactive notebooks            │ Batch processing only         │
│ Long-running cluster (always-on)      │ Job-based (start→run→stop)   │
│ Need GPU instances (ML training)      │ No GPU needed                 │
│ Multiple frameworks on same cluster   │ Single-purpose ETL            │
│ Team has Spark/Hadoop expertise       │ Team wants simplicity         │
│ Need 100+ worker nodes                │ < 100 DPUs sufficient         │
│ Budget: Spot instances save 70%       │ Flex execution saves 34%      │
├───────────────────────────────────────┼───────────────────────────────┤
│ Cost model: EC2 + EMR surcharge       │ Cost model: DPU-seconds       │
│ Management: You manage cluster        │ Management: AWS manages all   │
│ Cold start: 5-10 min (cluster launch) │ Cold start: 30-60 sec         │
└───────────────────────────────────────┴───────────────────────────────┘

SPECIFIC SCENARIOS for EMR:
1. ML pipeline training 200GB model with custom TensorFlow (GPU + custom libs)
2. Always-on Presto cluster serving 50 analysts with sub-second SQL
3. 80TB daily batch processing with heavily tuned Spark (custom memory/shuffle)
4. Flink streaming job processing 1M events/sec from Kafka
5. Monthly compliance report processing 500TB of archived data (transient Spot cluster)
```

---

### Q2: How do you design an EMR cluster for a job that processes 50TB of data with optimal cost and performance?

**Answer:**

```
SIZING METHODOLOGY:

Step 1: Determine memory needs
├── Data size: 50TB (compressed Parquet ~5:1 → ~250TB raw in memory worst case)
├── Working memory per executor: Data processed in partitions (not all at once)
├── Target partition size: 256MB → 50TB / 256MB = ~200,000 partitions
├── Executor memory needed: 4-8GB per executor core (for shuffles/caching)
└── Total cluster memory: At least 2-3x the largest shuffle stage

Step 2: Calculate nodes
├── Instance type: r6g.4xlarge (16 vCPU, 128GB RAM, Graviton = 20% cheaper)
├── Usable memory per node: ~100GB (128GB minus OS/YARN overhead)
├── Target: Process data in 2 hours
├── Spark parallelism: 4 cores/executor × (128GB/4) = 4 executors/node
└── Nodes needed: ~20-30 (depending on shuffle intensity)

Step 3: Cost optimization
├── Master: 1 × m6g.2xlarge On-Demand ($0.308/hr)
├── Core: 5 × r6g.4xlarge On-Demand ($0.806/hr × 5 = $4.03/hr) — for HDFS
├── Task: 20 × r6g.4xlarge Spot ($0.242/hr × 20 = $4.84/hr) — 70% savings!
└── Total: $0.308 + $4.03 + $4.84 = $9.18/hr

With 2-hour runtime: $18.36 per job run
Monthly (daily): $550/month vs $2,000+ if all On-Demand
```

```hcl
# Terraform configuration
resource "aws_emr_cluster" "optimized" {
  name          = "50tb-processing"
  release_label = "emr-7.1.0"
  applications  = ["Spark"]
  
  ec2_attributes {
    instance_profile = aws_iam_instance_profile.emr.arn
    subnet_ids       = var.private_subnet_ids
  }
  
  master_instance_fleet {
    target_on_demand_capacity = 1
    instance_type_configs {
      instance_type     = "m6g.2xlarge"
      weighted_capacity = 1
    }
  }
  
  core_instance_fleet {
    target_on_demand_capacity = 5
    instance_type_configs {
      instance_type     = "r6g.4xlarge"
      weighted_capacity = 1
    }
    instance_type_configs {
      instance_type     = "r5.4xlarge"
      weighted_capacity = 1
    }
  }
  
  # Task fleet: 100% Spot with multiple instance types
  # (more types = better Spot availability)
  task_instance_fleet {
    target_spot_capacity = 20
    launch_specifications {
      spot_specification {
        allocation_strategy            = "capacity-optimized"
        timeout_action                 = "SWITCH_TO_ON_DEMAND"
        timeout_duration_minutes       = 10
      }
    }
    instance_type_configs {
      instance_type     = "r6g.4xlarge"
      weighted_capacity = 1
    }
    instance_type_configs {
      instance_type     = "r5.4xlarge"
      weighted_capacity = 1
    }
    instance_type_configs {
      instance_type     = "r5a.4xlarge"
      weighted_capacity = 1
    }
    instance_type_configs {
      instance_type     = "r6i.4xlarge"
      weighted_capacity = 1
    }
    instance_type_configs {
      instance_type     = "m6g.8xlarge"
      weighted_capacity = 2
    }
  }
  
  configurations_json = jsonencode([{
    "Classification" = "spark-defaults"
    "Properties" = {
      "spark.executor.memory"              = "28g"
      "spark.executor.cores"               = "4"
      "spark.executor.memoryOverhead"      = "4g"
      "spark.sql.adaptive.enabled"         = "true"
      "spark.sql.shuffle.partitions"       = "2000"
      "spark.dynamicAllocation.enabled"    = "true"
      "spark.dynamicAllocation.maxExecutors" = "200"
    }
  }])
}
```

---

### Q3: Your EMR Spark job fails with "Container killed by YARN for exceeding memory limits." How do you troubleshoot?

**Answer:**

**Root Causes (in order of likelihood):**

```
1. EXECUTOR MEMORY TOO LOW:
   ├── Cause: Data partition larger than executor memory
   ├── Diagnosis: Check YARN logs: "Container killed by YARN for exceeding physical memory"
   ├── Fix: Increase spark.executor.memory or spark.executor.memoryOverhead
   └── Rule: memoryOverhead should be 10-15% of executor.memory (or 4GB minimum)

2. DATA SKEW:
   ├── Cause: One partition has 100x more data than others
   ├── Diagnosis: Spark UI → Stages → Tasks → Duration/Input size uneven
   ├── Fix: Repartition by different key, enable AQE skew join
   └── Example: customer_id = "UNKNOWN" has 80% of rows

3. SHUFFLE EXPLOSION:
   ├── Cause: Wide transformation (groupBy, join) produces more data than input
   ├── Diagnosis: Spark UI → Shuffle Write >> Shuffle Read
   ├── Fix: Filter before join, broadcast small tables, reduce partition count
   └── Increase: spark.sql.shuffle.partitions (from 200 to 2000+)

4. DRIVER OOM:
   ├── Cause: collect(), toPandas(), or broadcast() on too-large data
   ├── Diagnosis: "java.lang.OutOfMemoryError: Java heap space" on DRIVER
   ├── Fix: Never collect() large data; increase spark.driver.memory
   └── Rule: Driver only needs memory for coordination, not data
```

**Systematic fix:**
```python
# Spark configuration for memory-intensive jobs
spark_conf = {
    # Executor memory (per executor)
    "spark.executor.memory": "28g",          # Main heap
    "spark.executor.memoryOverhead": "6g",   # Off-heap (shuffle, native)
    "spark.executor.cores": "4",             # Cores per executor
    
    # Memory management
    "spark.memory.fraction": "0.8",          # 80% for execution+storage
    "spark.memory.storageFraction": "0.3",   # 30% of that for caching
    "spark.memory.offHeap.enabled": "true",  # Use off-heap (avoids GC)
    "spark.memory.offHeap.size": "8g",       # Off-heap size
    
    # Adaptive Query Execution (auto-fixes many issues)
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.adaptive.coalescePartitions.enabled": "true",
    "spark.sql.adaptive.skewJoin.enabled": "true",
    
    # Shuffle tuning
    "spark.sql.shuffle.partitions": "2000",  # More partitions = smaller each
    "spark.shuffle.compress": "true",
    "spark.shuffle.spill.compress": "true",
}
```

---

### Q4: Explain the difference between EMR on EC2, EMR on EKS, and EMR Serverless. When would you use each?

**Answer:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Feature          │ EMR on EC2      │ EMR on EKS     │ EMR Serverless   │
├───────────────────┼─────────────────┼────────────────┼──────────────────┤
│ Infrastructure    │ Dedicated EC2   │ Shared EKS     │ None (AWS)       │
│ Cluster mgmt     │ You manage      │ K8s manages    │ Fully managed    │
│ Scaling          │ Managed Scaling │ Pod auto-scale │ Automatic        │
│ Startup time     │ 5-10 minutes    │ 30-60 seconds  │ 15-30 seconds    │
│ Idle cost        │ YES (cluster)   │ Shared w/ K8s  │ NO (scales to 0) │
│ Multi-framework  │ YES (all)       │ Spark + Hive   │ Spark + Hive     │
│ Custom libraries │ Full control    │ Container img  │ Limited          │
│ GPU support      │ YES             │ YES            │ NO               │
│ HDFS             │ YES             │ NO             │ NO               │
│ Notebooks        │ EMR Studio      │ Via Jupyter    │ NO               │
│ Best for         │ Complex/large,  │ K8s teams,     │ Job-based,       │
│                  │ long-running    │ multi-tenant   │ simple config    │
├───────────────────┼─────────────────┼────────────────┼──────────────────┤
│ Use case example │ 50TB daily ETL  │ Multiple teams │ Hourly Spark     │
│                  │ with custom     │ sharing one    │ job, variable    │
│                  │ Spark + Flink   │ EKS cluster    │ data volume      │
└───────────────────┴─────────────────┴────────────────┴──────────────────┘

DECISION TREE:
├── "Do we already have EKS and want to share infrastructure?"
│   └── YES → EMR on EKS
├── "Do we need GPU, HDFS, Flink, HBase, or heavy customization?"
│   └── YES → EMR on EC2
├── "Is our workload job-based (start→run→finish) with standard config?"
│   └── YES → EMR Serverless
└── "Do we want zero idle cost and fastest startup?"
    └── YES → EMR Serverless
```

---

### Q5: How does EMR handle Spot Instance interruptions? How do you make jobs resilient?

**Answer:**

```
SPOT INTERRUPTION TIMELINE:
T-0: AWS sends 2-minute warning to the instance
T-2min: Instance terminated (task node gone)

WHAT HAPPENS TO YOUR JOB:
├── If task node is interrupted:
│   ├── YARN detects node loss
│   ├── Tasks on that node are re-scheduled to other nodes
│   ├── Spark re-executes lost shuffle data (from lineage)
│   └── Job continues (slightly slower, but doesn't fail)
│
├── If core node is interrupted (using HDFS):
│   ├── HDFS data blocks have 3 replicas → data survives
│   ├── YARN re-schedules tasks
│   ├── But if multiple cores lost → potential data loss!
│   └── RECOMMENDATION: Use S3 (not HDFS) with Spot
│
└── If master is interrupted:
    └── CLUSTER DIES (never use Spot for master!)
```

**Resilience best practices:**
```hcl
# 1. Instance Fleet with diversification (most important!)
task_instance_fleet {
  instance_type_configs {
    instance_type = "r5.4xlarge"      # x86
    weighted_capacity = 1
  }
  instance_type_configs {
    instance_type = "r5a.4xlarge"     # AMD
    weighted_capacity = 1
  }
  instance_type_configs {
    instance_type = "r6g.4xlarge"     # Graviton (ARM)
    weighted_capacity = 1
  }
  instance_type_configs {
    instance_type = "r6i.4xlarge"     # Latest gen
    weighted_capacity = 1
  }
  instance_type_configs {
    instance_type = "m5.8xlarge"      # Different family
    weighted_capacity = 2
  }
  # 5 different instance types = low chance ALL are interrupted!
  
  launch_specifications {
    spot_specification {
      allocation_strategy      = "capacity-optimized"  # Best availability
      timeout_action           = "SWITCH_TO_ON_DEMAND"  # Fallback if no Spot
      timeout_duration_minutes = 10
    }
  }
}
```

```python
# 2. Spark configuration for Spot resilience
spark_conf = {
    # External shuffle service (survives executor loss)
    "spark.shuffle.service.enabled": "true",
    
    # Decommissioning (graceful removal of Spot instances)
    "spark.decommission.enabled": "true",
    "spark.storage.decommission.enabled": "true",
    "spark.storage.decommission.shuffleBlocks.enabled": "true",
    # Migrates shuffle data BEFORE instance is terminated!
    
    # Retry configuration
    "spark.task.maxFailures": "8",          # Retry tasks up to 8 times
    "spark.stage.maxConsecutiveAttempts": "4",
    
    # Speculation (detect slow/stuck tasks)
    "spark.speculation": "true",
    "spark.speculation.interval": "5s",
    "spark.speculation.quantile": "0.9",  # Speculate if task > 90th percentile duration
}
```

```python
# 3. Checkpointing for long-running jobs
# Save intermediate state to S3 (survive total cluster failure)
df_intermediate = df.transform(expensive_operation)
df_intermediate.write.mode("overwrite").parquet("s3://checkpoints/stage1/")

# Read from checkpoint if restarting
df_from_checkpoint = spark.read.parquet("s3://checkpoints/stage1/")
df_final = df_from_checkpoint.transform(next_operation)
```

---

### Q6: How do you monitor and debug a slow EMR Spark job in production?

**Answer:**

**Monitoring Stack:**
```
┌─────────────────────────────────────────────────────────────────┐
│  EMR Monitoring Layers:                                          │
│                                                                   │
│  1. Spark UI (port 18080) — Job/Stage/Task level detail         │
│     ├── DAG visualization                                        │
│     ├── Stage timing + task distribution (find skew)            │
│     ├── Shuffle read/write metrics                               │
│     ├── Executor memory/GC metrics                              │
│     └── SQL plan (physical execution plan)                      │
│                                                                   │
│  2. YARN Resource Manager (port 8088)                            │
│     ├── Application status (RUNNING, ACCEPTED, FAILED)          │
│     ├── Container allocation                                     │
│     ├── Memory/CPU utilization per node                         │
│     └── Queue depth (pending applications)                      │
│                                                                   │
│  3. CloudWatch Metrics (automatic)                               │
│     ├── YARNMemoryAvailablePercentage                           │
│     ├── IsIdle (cluster has no running jobs)                    │
│     ├── ContainerPending (jobs waiting for resources)           │
│     ├── HDFSUtilization                                         │
│     └── Custom metrics via EMR bootstrap                        │
│                                                                   │
│  4. Spark History Server (persists after job completes)          │
│     ├── Review completed job metrics                            │
│     ├── Compare runs over time                                  │
│     └── Requires: spark.eventLog.enabled=true + S3 path        │
│                                                                   │
│  5. Ganglia (port 80) — Cluster hardware metrics                │
│     ├── CPU, memory, network, disk per node                     │
│     └── Historical trends                                       │
└─────────────────────────────────────────────────────────────────┘
```

**Debugging workflow for slow job:**
```
Step 1: Spark UI → Jobs tab → Find slow job → Click into stages
Step 2: In slow stage → Tasks tab → Sort by Duration
Step 3: Check:
├── Is one task 100x slower than others? → DATA SKEW
│   Fix: Repartition, salt key, enable AQE skew join
├── Are all tasks slow but evenly distributed? → RESOURCE ISSUE
│   Fix: More executors, more memory, or smaller partitions
├── Is Shuffle Write very large? → SHUFFLE BOTTLENECK
│   Fix: Broadcast small table, filter before join, reduce shuffles
├── Is GC Time > 10% of task time? → MEMORY PRESSURE
│   Fix: Increase executor memory, reduce partition size
└── Is Input size very different between tasks? → PARTITION IMBALANCE
    Fix: Repartition input data, adjust spark.sql.files.maxPartitionBytes
    
Step 4: SQL tab → Find the query → Physical Plan
├── Look for "Exchange" (shuffle) — minimize these
├── Look for "BroadcastHashJoin" vs "SortMergeJoin" — prefer broadcast for small tables
└── Look for "Filter" pushed down vs applied late — push filters early

Step 5: Executor tab → Check memory usage
├── Storage Memory used → Too much caching?
├── Execution Memory → Shuffle spilling to disk?
└── GC time → Need bigger heap or fewer objects?
```

---

### Q7: Compare EMR Serverless with EMR on EC2 for a job that runs hourly and processes 5TB each time.

**Answer:**

```
SCENARIO: Hourly Spark job, 5TB input, ~30 min runtime, 24/7 operation

EMR ON EC2 (Always-on cluster):
├── Cluster: 1 Master + 5 Core + 15 Task (Spot)
├── Instance: r6g.4xlarge (16 vCPU, 128GB)
├── Cost/hour: $0.31 + $4.03 + $3.63 = $7.97/hr
├── Monthly: $7.97 × 24 × 30 = $5,738/month
├── Pros: Fast job start (cluster already warm), custom config
├── Cons: Paying for idle time between jobs (~50% idle)
└── Effective utilization: 50% (30 min running, 30 min idle per hour)

EMR SERVERLESS:
├── Resources: Auto-provisions ~60 vCPU + 240GB per run
├── Cost/run: 60 × $0.052624 × 0.5hr + 240 × $0.0057785 × 0.5hr
├──         = $1.58 + $0.69 = $2.27/run
├── Monthly: $2.27 × 24 × 30 = $1,634/month
├── Pros: Zero idle cost, no cluster management, auto-scales
├── Cons: Slightly slower start (15-30 sec), less customization
└── With pre-initialized capacity: Near-instant start

EMR ON EC2 (Transient cluster per run):
├── Launch cluster → Run job → Terminate (every hour)
├── Overhead: 5-10 min cluster launch time each hour
├── Cost: Only pay during 30 min job + 10 min startup = 40 min/hr
├── Monthly: $7.97 × (40/60) × 24 × 30 = $3,825/month
├── Pros: No idle cost (cluster terminates)
├── Cons: 10 min wasted on startup each hour, complexity
└── Best for: Longer intervals (daily, not hourly)

RECOMMENDATION for hourly 5TB:
├── EMR Serverless: $1,634/month ← WINNER (71% cheaper)
├── Reason: No idle cost, auto-scales perfectly for this pattern
├── If custom Spark config needed: EMR on EC2 with managed scaling
└── If sub-second latency to start: Pre-initialized Serverless capacity
```

---

### Q8: How do you handle data locality in EMR when using S3 instead of HDFS?

**Answer:**

```
THE PROBLEM:
HDFS: Data blocks stored on the SAME node that processes them (locality!)
S3: Data is REMOTE — every read goes over network

Does S3 eliminate data locality benefit? YES, but with mitigations:

WHY S3 IS STILL PREFERRED:
├── S3 bandwidth: 100+ Gbps aggregate per cluster (sufficient)
├── Cost: S3 = $0.023/GB/month vs HDFS = 3x (replication) × EBS cost
├── Persistence: S3 data survives cluster termination
├── Scalability: No HDFS NameNode bottleneck
└── Sharing: Multiple clusters access same S3 data

PERFORMANCE OPTIMIZATION for S3:
```

```python
# 1. File size optimization (128-256MB per file for S3)
# Small files = many S3 GET requests = slow
# Large files = too few splits = poor parallelism
# Sweet spot: 128-256MB per file (matches Spark split size)

# 2. S3 connection tuning
spark_conf = {
    "spark.hadoop.fs.s3a.connection.maximum": "200",    # More concurrent S3 connections
    "spark.hadoop.fs.s3a.threads.max": "64",            # More I/O threads
    "spark.hadoop.fs.s3a.fast.upload": "true",          # Parallel upload
    "spark.hadoop.fs.s3a.fast.upload.buffer": "bytebuffer",  # In-memory buffer
    "spark.hadoop.fs.s3a.multipart.size": "67108864",   # 64MB multipart chunks
    "spark.sql.files.maxPartitionBytes": "268435456",   # 256MB per split
}

# 3. S3 request optimizations
spark_conf.update({
    "spark.hadoop.fs.s3a.list.version": "2",            # Faster S3 listing
    "spark.hadoop.fs.s3a.committer.name": "magic",      # Efficient S3 writes
    "spark.hadoop.fs.s3a.committer.magic.enabled": "true",
    "spark.sql.sources.commitProtocolClass": 
        "org.apache.spark.internal.io.cloud.PathOutputCommitProtocol",
})

# 4. Use S3 Express One Zone (new!) for lowest latency
# Single-digit ms latency vs 10-100ms for standard S3
# 10x faster metadata operations
# Best for: Shuffle-heavy workloads that read/write temp data

# 5. Use local NVMe for shuffle (instance store)
# r5d, i3, d2 instances have local NVMe SSDs
# Configure Spark to use local disk for shuffle (not S3)
spark_conf.update({
    "spark.local.dir": "/mnt,/mnt1",  # Local NVMe mount points
    # Shuffle data stays local (fast), input/output on S3 (persistent)
})
```

---

### Q9: Explain EMR's integration with Lake Formation for fine-grained access control.

**Answer:**

```
THE CHALLENGE:
EMR traditionally uses IAM roles for S3 access.
IAM operates at BUCKET/PREFIX level, not TABLE/COLUMN level.

Lake Formation adds: Table, column, row, and cell-level access control.
But EMR must be configured specifically to use it.

ARCHITECTURE:
┌──────────────────────────────────────────────────────────────────┐
│  Without Lake Formation:                                          │
│  EMR → IAM Role → S3 (access ALL files in path)                 │
│  Problem: Can't restrict columns, can't share across accounts    │
│                                                                    │
│  With Lake Formation:                                             │
│  EMR → Lake Formation → Temporary Credentials → S3               │
│  Benefit: Column-level security, cross-account sharing           │
└──────────────────────────────────────────────────────────────────┘
```

**How to enable:**
```hcl
# EMR cluster configuration for Lake Formation
resource "aws_emr_cluster" "lf_enabled" {
  release_label = "emr-6.15.0"  # Must be 6.9+ for LF
  
  configurations_json = jsonencode([
    {
      "Classification" = "spark-hive-site"
      "Properties" = {
        # Enable Glue Catalog as Hive metastore
        "hive.metastore.client.factory.class" = 
          "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory"
      }
    },
    {
      "Classification" = "spark-defaults"
      "Properties" = {
        # Enable Lake Formation credential vending
        "spark.sql.catalog.glue_catalog" = "org.apache.iceberg.spark.SparkCatalog"
        "spark.sql.catalog.glue_catalog.catalog-impl" = 
          "org.apache.iceberg.aws.glue.GlueCatalog"
      }
    },
    {
      "Classification" = "emrfs-site"
      "Properties" = {
        # CRITICAL: Use Lake Formation for S3 authorization
        "fs.s3.cse.enabled" = "false"
        "fs.s3.authorization.enabled" = "true"
        "fs.s3.authorization.mode" = "lake-formation"
      }
    }
  ])
}
```

**What this enables:**
```python
# With Lake Formation, THIS happens:
spark.sql("SELECT customer_id, name FROM glue_catalog.customers")
# → Lake Formation checks: "Does this EMR role have SELECT on customer_id, name?"
# → If YES: Returns temporary S3 credentials scoped to those columns only
# → If NO: AccessDeniedException

# Different EMR jobs get different column access:
# ETL job role → All columns (for processing)
# Analyst job role → Only non-PII columns (name, purchase_count — not SSN)
# Auditor job role → All columns (for compliance)
```

---

### Q10: Your EMR cluster is costing $15K/month. Walk me through a cost optimization strategy without reducing performance.

**Answer:**

```
COST BREAKDOWN ANALYSIS:
$15,000/month typical breakdown:
├── Master nodes: $500 (3% — small, not much to optimize)
├── Core nodes (On-Demand): $7,000 (47% — biggest target!)
├── Task nodes (some On-Demand): $5,000 (33% — should be Spot!)
├── EBS storage: $1,500 (10% — often over-provisioned)
└── S3 requests: $1,000 (7% — small file problem?)
```

**Optimization plan:**

| Strategy | Savings | Action |
|----------|---------|--------|
| Task nodes → 100% Spot | 70% of task = -$3,500 | Instance Fleets with 5+ types |
| Core nodes → Graviton (r6g) | 20% cheaper = -$1,400 | Switch to ARM-based instances |
| Right-size cluster | 30% = -$2,100 | Enable Managed Scaling (auto) |
| Transient clusters | 40% = -$2,800 | Terminate between jobs (if batch) |
| S3 Express One Zone | 50% less requests = -$500 | For shuffle/temp data |
| EBS optimization | 30% = -$450 | gp3 instead of gp2, right-size |
| Spot for Core (S3 mode) | 50% = -$3,500 | If using S3 (not HDFS) as primary |

**Total potential savings: $10,750/month → New cost: ~$4,250/month (72% reduction!)**

```python
# Implementation priority (quick wins first):
optimizations = [
    {
        "action": "Switch task nodes to 100% Spot Instance Fleets",
        "effort": "Low (config change)",
        "savings": "$3,500/month",
        "risk": "Low (task nodes are expendable)"
    },
    {
        "action": "Enable Managed Scaling (auto-scale down when idle)",
        "effort": "Low (enable feature)",
        "savings": "$2,100/month",
        "risk": "None (scales back up automatically)"
    },
    {
        "action": "Switch to Graviton instances (r6g, m6g)",
        "effort": "Medium (test job compatibility with ARM)",
        "savings": "$1,400/month",
        "risk": "Low (most Spark jobs work on ARM)"
    },
    {
        "action": "Use transient clusters (terminate between jobs)",
        "effort": "Medium (update orchestration)",
        "savings": "$2,800/month",
        "risk": "Medium (add startup time overhead)"
    },
    {
        "action": "Move Core to Spot (requires S3-only, no HDFS)",
        "effort": "High (architectural change)",
        "savings": "$3,500/month",
        "risk": "Medium (need to verify no HDFS dependency)"
    },
]
```

---

## 🆚 EMR vs Competitors

| Feature | EMR on EC2 | EMR Serverless | Databricks | GCP Dataproc | Glue |
|---------|-----------|---------------|------------|--------------|------|
| Control | Full | Limited | Medium | Medium | None |
| Multi-framework | Yes (all) | Spark+Hive | Spark+SQL | Spark+Hadoop | Spark |
| Cost model | Instance-hr | vCPU-hr | DBU | Instance-hr | DPU-sec |
| Spot/Preemptible | Yes (70% off) | N/A | Yes | Yes (80% off) | Flex (34%) |
| Startup time | 5-10 min | 15-30 sec | 2-5 min | 1-2 min | 30-60 sec |
| Idle cost | Yes | No | Optional | Optional | No |
| GPU support | Yes | No | Yes | Yes | No |
| Notebooks | EMR Studio | No | Built-in | Jupyter | Studio |
| Best for | Large/complex | Simple batch | ML + SQL | GCP native | AWS ETL |

---

## 🏆 Production Best Practices

```yaml
cluster_design:
  - Use Instance Fleets (not Instance Groups) for Spot resilience
  - Master: Always On-Demand, m-series (no heavy compute needed)
  - Core: On-Demand if HDFS, Spot if S3-only
  - Task: Always 100% Spot (no data stored, safe to lose)
  - List 5+ instance types per fleet (Spot availability!)
  - Use Graviton instances (20-30% cost savings, same performance)

storage:
  - Use S3 as primary storage (not HDFS) for most workloads
  - HDFS only for: extreme speed needs, temporary shuffle data
  - Local NVMe (instance store) for shuffle/temp (r5d, i3 instances)
  - S3 VPC Endpoint (Gateway) — free, no NAT charges

spark_tuning:
  - Enable AQE (Adaptive Query Execution) — auto-tunes runtime
  - Dynamic allocation for variable workloads
  - Broadcast small tables (< 100MB) to avoid shuffle
  - Right-size partitions: 128-256MB per partition
  - Enable speculation for Spot tolerance

security:
  - Deploy in private subnet (no public IPs)
  - Use Lake Formation for fine-grained access
  - Encrypt at rest (EBS + S3 with KMS)
  - Encrypt in transit (TLS for shuffle)
  - Kerberos for multi-tenant clusters
  - Separate security groups for Master/Core/Task

operations:
  - Enable Spark event logs to S3 (persist history)
  - Set up CloudWatch alarms (ContainerPending, Memory)
  - Use EMR Managed Scaling (auto-adjust cluster size)
  - Transient clusters for batch (terminate after job)
  - Long-running clusters for interactive (notebooks, Presto)
  - Bootstrap actions for custom setup (install libraries, configure)
```
