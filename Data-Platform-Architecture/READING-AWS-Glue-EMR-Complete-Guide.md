# AWS Glue & EMR — Complete Knowledge Guide

> **Purpose**: Understand how AWS Glue and EMR work from the ground up — when to use each, how they process data, performance tuning, and production patterns.

---

## 1. AWS Glue: The Big Picture

AWS Glue is a **serverless** data integration service. Think of it as three things in one:

```
┌─────────────────────────────────────────────────────────────────┐
│  AWS Glue = Three Services Combined                              │
│                                                                   │
│  1. DATA CATALOG (Metadata Store)                                │
│     ├── Central repository of table definitions                  │
│     ├── Like a "phone book" for your data lake                  │
│     ├── Used by: Athena, Redshift Spectrum, EMR, Lake Formation │
│     └── Contains: Databases, Tables, Partitions, Schemas        │
│                                                                   │
│  2. ETL ENGINE (Apache Spark, serverless)                        │
│     ├── Runs PySpark/Scala jobs without managing clusters        │
│     ├── Auto-scales workers (DPUs)                               │
│     ├── Supports: Spark, Python Shell, Ray                       │
│     └── You write code → Glue runs it → you pay per second     │
│                                                                   │
│  3. CRAWLERS (Auto-discovery)                                    │
│     ├── Scans S3/databases and infers schema                    │
│     ├── Creates/updates tables in the Data Catalog              │
│     └── Runs on schedule or on-demand                           │
└─────────────────────────────────────────────────────────────────┘
```

### When to Use Glue vs EMR

```
┌────────────────────────────────────────────────────────────────┐
│  Decision Matrix: Glue vs EMR                                   │
│                                                                  │
│  Use GLUE when:                                                  │
│  ├── Job runs < 4 hours                                         │
│  ├── Standard ETL (read → transform → write)                   │
│  ├── You don't want to manage infrastructure                    │
│  ├── Team knows Python/PySpark                                  │
│  ├── Data volume < 50TB per job                                 │
│  └── Need Data Catalog integration                              │
│                                                                  │
│  Use EMR when:                                                   │
│  ├── Job runs > 4 hours (cost-effective for long jobs)          │
│  ├── Need custom Spark/Hive/Presto configuration                │
│  ├── Data volume > 50TB per job                                 │
│  ├── Need specific library versions                             │
│  ├── Need persistent cluster (interactive notebooks)            │
│  ├── Need non-Spark frameworks (Flink, HBase, Trino)           │
│  └── Need fine-grained cluster tuning (memory, cores)           │
│                                                                  │
│  Use GLUE + EMR together when:                                   │
│  ├── Glue Catalog as metadata store for EMR queries             │
│  ├── Glue Crawlers discover data, EMR processes it              │
│  └── Glue for light ETL, EMR for heavy computation             │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. Glue Data Catalog Deep Dive

### What's Inside the Catalog

```
Glue Data Catalog Structure:

Catalog (one per account per region)
├── Database: "raw_events"
│   ├── Table: "clickstream"
│   │   ├── Schema: {user_id: string, event: string, ts: timestamp}
│   │   ├── Location: s3://datalake/raw/clickstream/
│   │   ├── Format: JSON
│   │   ├── Partition Keys: [year, month, day]
│   │   └── Partitions:
│   │       ├── year=2024/month=01/day=01 → s3://datalake/raw/.../
│   │       ├── year=2024/month=01/day=02 → s3://datalake/raw/.../
│   │       └── ... (thousands of partitions)
│   └── Table: "user_profiles"
│       ├── Schema: {user_id: string, name: string, email: string}
│       ├── Location: s3://datalake/raw/users/
│       └── Format: Parquet
├── Database: "processed"
│   └── Table: "daily_metrics"
└── Database: "curated"
    └── Table: "revenue_report"
```

### How Crawlers Work

```
┌──────────────┐         ┌──────────────────┐        ┌──────────────┐
│   S3 Bucket  │ ──────▶ │   Glue Crawler   │ ─────▶ │  Data Catalog│
│              │  scans   │                  │ creates │              │
│  /raw/       │         │  1. Lists files   │        │  Tables +    │
│  ├── *.json  │         │  2. Reads samples │        │  Schemas +   │
│  ├── *.csv   │         │  3. Infers schema │        │  Partitions  │
│  └── *.parq  │         │  4. Detects format│        │              │
└──────────────┘         │  5. Finds partns  │        └──────────────┘
                         └──────────────────┘

Important Crawler Settings:
├── SchemaChangePolicy:
│   ├── UPDATE_IN_DATABASE (auto-update schema — risky in prod!)
│   ├── LOG (log changes but don't update — safer)
│   └── ADD_NEW_COLUMNS (only add, never remove — safest)
├── Grouping: Combine compatible schemas into one table
└── Exclusions: Ignore _temporary, _SUCCESS, .crc files
```

### Partitions: Why They Matter

```
WITHOUT Partitions:
Query: SELECT * FROM events WHERE date = '2024-06-15'
→ Athena/Spark scans ALL files in the table (5 years of data!)
→ Cost: $$$$, Time: minutes/hours

WITH Partitions (partitioned by date):
Query: SELECT * FROM events WHERE year='2024' AND month='06' AND day='15'
→ Only scans files in s3://lake/events/year=2024/month=06/day=15/
→ Cost: $, Time: seconds

Rule of Thumb:
├── Good partition: Reduces data scanned by 10x-1000x
├── Bad partition: Too many partitions (> 1M) → catalog overhead
├── Partition column: Should appear in WHERE clauses frequently
└── Partition size: Each partition should have 100MB-1GB of data
```

---

## 3. Glue ETL Jobs

### Job Types

```
┌─────────────────────────────────────────────────────────────┐
│  Glue Job Types:                                             │
│                                                              │
│  1. Spark ETL (Most Common)                                  │
│     ├── Language: PySpark or Scala                           │
│     ├── Engine: Apache Spark 3.3+ (Glue 4.0)               │
│     ├── Workers: G.1X (4 vCPU, 16GB) or G.2X (8 vCPU, 32GB)│
│     ├── Scaling: Fixed or Auto-scaling                       │
│     ├── Use for: Any data transformation at scale            │
│     └── Min billing: 1 minute, then per-second              │
│                                                              │
│  2. Python Shell                                             │
│     ├── Language: Python (no Spark)                          │
│     ├── Workers: 1 DPU (4 vCPU, 16GB) or 0.0625 (1 vCPU)  │
│     ├── Use for: Small jobs, API calls, orchestration logic  │
│     ├── Libraries: pandas, boto3, requests                   │
│     └── Much cheaper than Spark for small tasks             │
│                                                              │
│  3. Ray (New)                                                │
│     ├── Language: Python                                     │
│     ├── Engine: Ray distributed computing                    │
│     ├── Use for: ML inference, Python-native parallelism    │
│     └── When Spark is overkill but you need distribution    │
│                                                              │
│  4. Streaming ETL                                            │
│     ├── Engine: Spark Structured Streaming                   │
│     ├── Sources: Kinesis, Kafka (MSK)                       │
│     ├── Continuous: Runs indefinitely                        │
│     └── Use for: Real-time data processing                  │
└─────────────────────────────────────────────────────────────┘
```

### DynamicFrames vs DataFrames

```python
# Glue has TWO APIs for data processing:

# 1. DynamicFrame (Glue-native, forgiving with schema)
from awsglue.dynamicframe import DynamicFrame

dyf = glueContext.create_dynamic_frame.from_catalog(
    database="raw", table_name="events"
)
# DynamicFrame advantages:
# - Handles schema inconsistencies (different types in same column)
# - Built-in transforms (ApplyMapping, ResolveChoice)
# - Integrates with Glue bookmarks
# - Better for messy/evolving data

# 2. DataFrame (Standard Spark, strict schema)
df = spark.read.parquet("s3://lake/events/")
# DataFrame advantages:
# - Full Spark SQL support
# - More operations available
# - Better performance for complex transformations
# - Industry standard (same API everywhere)

# CONVERT between them:
df = dyf.toDF()           # DynamicFrame → DataFrame
dyf = DynamicFrame.fromDF(df, glueContext, "name")  # DataFrame → DynamicFrame

# BEST PRACTICE: 
# Use DynamicFrame for reading from Catalog + writing
# Convert to DataFrame for complex transformations
# Convert back to DynamicFrame for writing with bookmarks
```

### Job Bookmarks (Incremental Processing)

```
WITHOUT Bookmarks:
Run 1: Process ALL data (1TB) — 2 hours
Run 2: Process ALL data AGAIN (1.01TB) — 2 hours (wasted!)
Run 3: Process ALL data AGAIN... (cycle continues)

WITH Bookmarks:
Run 1: Process ALL data (1TB) — 2 hours, bookmark saved
Run 2: Process ONLY NEW data since last run (10GB) — 5 minutes!
Run 3: Process ONLY NEW data since run 2... (efficient!)

How it works:
├── Bookmark tracks: last processed file timestamp or key
├── On next run: only reads files/records AFTER the bookmark
├── Supports: S3 (file timestamp), JDBC (column-based), Kafka (offset)
└── Reset: Can reset bookmark to reprocess all data
```

```python
# Enable bookmarks in your job:
args = getResolvedOptions(sys.argv, ['JOB_NAME'])
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read with bookmark tracking
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="events",
    transformation_ctx="datasource"  # THIS enables bookmarks
)

# ... transform ...

# Write with bookmark tracking
glueContext.write_dynamic_frame.from_options(
    frame=output,
    connection_type="s3",
    connection_options={"path": "s3://lake/processed/"},
    format="parquet",
    transformation_ctx="datasink"  # THIS enables bookmarks
)

# CRITICAL: Must call job.commit() to save the bookmark
job.commit()
```

---

## 4. Glue Performance Tuning

### Common Performance Issues & Fixes

```
Problem 1: Job takes forever to START (10+ minutes before processing)
├── Cause: Cold start (downloading Spark libraries)
├── Fix: Use Glue 4.0 (faster cold start)
├── Fix: Enable job warm pools (pre-warmed resources)
└── Fix: For critical jobs, use EMR (always-on cluster)

Problem 2: Job runs slowly despite many workers
├── Cause: Data skew (one partition has 90% of data)
├── Diagnosis: Check Spark UI → stages with uneven task duration
├── Fix: Repartition by a different key before processing
├── Fix: Enable AQE (Adaptive Query Execution)
└── Fix: Salt the skewed key (add random prefix)

Problem 3: Out of Memory errors
├── Cause: Dataset too large for worker memory
├── Fix: Use G.2X workers (32GB instead of 16GB)
├── Fix: Increase number of workers
├── Fix: Reduce partition size (more partitions = less per worker)
├── Fix: Use pushdown predicate (read less data)
└── Fix: Avoid collect() or broadcast() on large datasets

Problem 4: Too many small output files
├── Cause: Many Spark partitions → many output files
├── Fix: coalesce(N) before writing (reduce partitions)
├── Fix: Set maxRecordsPerFile option
└── Fix: Run post-job compaction
```

### Key Configuration Settings

```python
# Essential Spark settings for Glue jobs
spark.conf.set("spark.sql.adaptive.enabled", "true")  # AQE (auto-tune)
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.shuffle.partitions", "200")  # Adjust based on data size

# S3 performance
spark.conf.set("spark.hadoop.fs.s3a.connection.maximum", "200")
spark.conf.set("spark.sql.files.maxPartitionBytes", "256mb")

# Memory management
spark.conf.set("spark.memory.fraction", "0.8")
spark.conf.set("spark.memory.storageFraction", "0.3")
```

---

## 5. Amazon EMR: Complete Understanding

### What EMR Actually Is

```
EMR = Managed Hadoop Ecosystem on EC2/EKS/Serverless

┌─────────────────────────────────────────────────────────────────┐
│  EMR Cluster Architecture:                                       │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Master Node (1 or 3 for HA)                                │ │
│  │  ├── YARN Resource Manager                                  │ │
│  │  ├── HDFS NameNode                                          │ │
│  │  ├── Spark Driver / Hive Metastore                         │ │
│  │  ├── Ganglia / Spark History Server                        │ │
│  │  └── Instance: m5.xlarge or larger                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Core Nodes (2+, stores HDFS data)                          │ │
│  │  ├── YARN NodeManager (runs containers)                    │ │
│  │  ├── HDFS DataNode (stores data blocks)                    │ │
│  │  ├── Spark Executor processes                               │ │
│  │  ├── Instance: Storage-optimized (d2, i3) or compute (m5)  │ │
│  │  └── CANNOT be terminated (data loss!)                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Task Nodes (0+, compute only, no HDFS)                     │ │
│  │  ├── YARN NodeManager only                                  │ │
│  │  ├── Spark Executor processes                               │ │
│  │  ├── CAN be terminated safely (no data stored)             │ │
│  │  ├── Perfect for SPOT instances (70% savings!)             │ │
│  │  └── Auto-scales based on YARN metrics                     │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### EMR Deployment Options

```
┌─────────────────────────────────────────────────────────────────┐
│  EMR on EC2 (Traditional)                                        │
│  ├── Full control over cluster configuration                    │
│  ├── Choose instance types, count, storage                      │
│  ├── Long-running clusters OR transient (per-job)              │
│  ├── Best for: Heavy, long-running, custom requirements        │
│  └── You manage: Cluster lifecycle, scaling                     │
│                                                                   │
│  EMR on EKS (Containerized)                                      │
│  ├── Run Spark on existing EKS cluster                          │
│  ├── Share compute with other K8s workloads                     │
│  ├── Per-job isolation via pods                                  │
│  ├── Best for: Teams already using Kubernetes                   │
│  └── Benefits: Better resource utilization, pod scheduling      │
│                                                                   │
│  EMR Serverless (Newest, Simplest)                               │
│  ├── No cluster management AT ALL                               │
│  ├── Submit jobs → EMR handles everything                       │
│  ├── Auto-scales to zero (pay nothing when idle)                │
│  ├── Pre-initialized capacity (for fast startup)                │
│  ├── Best for: Variable workloads, cost optimization            │
│  └── Limitations: Less customization than EC2                   │
└─────────────────────────────────────────────────────────────────┘
```

### Spot Instances Strategy for EMR

```
Production EMR Cost Optimization:

Master: ALWAYS On-Demand (losing master = lose cluster)
Core:   On-Demand (HDFS data, losing core = data loss)
        OR: On-Demand if using S3 as primary storage
Task:   100% Spot (no data, easily replaceable)

Cost Example:
├── 10 × m5.4xlarge On-Demand: $7.68/hr × 10 = $76.80/hr
├── 10 × m5.4xlarge Spot:      $2.30/hr × 10 = $23.00/hr
└── Savings: 70%!

Spot Best Practices:
├── Use Instance Fleets (not Instance Groups)
├── List 10+ instance types (more types = better availability)
├── Use "capacity-optimized" allocation strategy
├── Mix instance families: r5, r5a, r5d, r6g, m5, m5a
└── Set "timeout and switch to on-demand" for critical jobs
```

---

## 6. Spark Fundamentals (Essential for Both Glue & EMR)

### How Spark Processes Data

```
Your Code (Transformations)
        │
        ▼
┌────────────────────┐
│  Spark Driver      │  ← Plans the execution
│  (your program)    │  ← Creates DAG of operations
│  (master node)     │  ← Splits work into stages/tasks
└────────┬───────────┘
         │ distributes tasks
         ▼
┌────────────────────────────────────────────────────────────────┐
│  Executors (worker nodes)                                       │
│                                                                  │
│  Executor 1        Executor 2        Executor 3                 │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐              │
│  │ Task 1   │     │ Task 4   │     │ Task 7   │              │
│  │ Task 2   │     │ Task 5   │     │ Task 8   │              │
│  │ Task 3   │     │ Task 6   │     │ Task 9   │              │
│  └──────────┘     └──────────┘     └──────────┘              │
│                                                                  │
│  Each task processes ONE PARTITION of data                       │
│  Partition = chunk of data (typically 128MB-256MB)              │
└────────────────────────────────────────────────────────────────┘
```

### Transformations vs Actions (Lazy Evaluation)

```python
# TRANSFORMATIONS (lazy - nothing happens yet):
df = spark.read.parquet("s3://data/")         # Lazy
df2 = df.filter(df.amount > 100)              # Lazy
df3 = df2.groupBy("category").sum("amount")   # Lazy
# NO COMPUTATION HAS HAPPENED YET!

# ACTIONS (trigger execution):
df3.show()                    # Triggers computation → displays results
df3.write.parquet("s3://out") # Triggers computation → writes files
count = df3.count()           # Triggers computation → returns number

# WHY this matters:
# Spark optimizes the ENTIRE chain of transformations before executing
# Called "Catalyst Optimizer" — reorders, pushes filters, prunes columns
# This is why Spark can be fast even with many transformation steps
```

### The Shuffle: Understanding Data Movement

```
NARROW Transformations (no data movement between nodes):
├── filter(), map(), flatMap()
├── select(), withColumn()
└── Each partition processed independently (FAST)

WIDE Transformations (data moves between nodes = SHUFFLE):
├── groupBy(), reduceByKey(), join()
├── repartition(), distinct(), sort()
└── Data must be redistributed across cluster (SLOW, expensive)

Example of a Shuffle:
┌────────────┐          ┌────────────┐
│ Partition 1│          │ Partition A│
│ (A,1)(B,2) │──┐  ┌──▶│ (A,1)(A,3) │ ← All A's together
│ (A,3)      │  │  │   └────────────┘
└────────────┘  │  │   ┌────────────┐
                ├──┤   │ Partition B│
┌────────────┐  │  └──▶│ (B,2)(B,4) │ ← All B's together
│ Partition 2│  │      └────────────┘
│ (B,4)(C,5) │──┤      ┌────────────┐
│ (C,6)      │  └────▶ │ Partition C│
└────────────┘         │ (C,5)(C,6) │ ← All C's together
                       └────────────┘

Shuffle = Network I/O + Disk I/O + Serialization
It's the #1 performance bottleneck in Spark!

How to MINIMIZE shuffles:
├── Broadcast small tables (avoid shuffle join)
├── Use partition-compatible operations
├── Pre-partition data by join key
├── Reduce before shuffle (filter early)
└── Use AQE (Adaptive Query Execution) for auto-tuning
```

---

## 7. Common Data Processing Patterns

### Pattern 1: Bronze → Silver → Gold (Medallion)

```python
# BRONZE (Raw ingestion — append only, as-is format)
bronze_df = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db", table_name="events",
    push_down_predicate=f"date='{today}'"
).toDF()

# SILVER (Cleaned, deduplicated, typed)
silver_df = bronze_df \
    .dropDuplicates(["event_id"]) \
    .withColumn("amount", col("amount").cast("decimal(10,2)")) \
    .filter(col("event_id").isNotNull()) \
    .withColumn("processed_at", current_timestamp())

silver_df.write.mode("append") \
    .partitionBy("year", "month", "day") \
    .parquet("s3://lake/silver/events/")

# GOLD (Business aggregations, ready for consumption)
gold_df = silver_df \
    .groupBy("customer_id", "product_category") \
    .agg(
        sum("amount").alias("total_spend"),
        count("event_id").alias("purchase_count"),
        avg("amount").alias("avg_order_value")
    )

gold_df.write.mode("overwrite") \
    .parquet("s3://lake/gold/customer_metrics/")
```

### Pattern 2: CDC (Change Data Capture) Processing

```python
# DMS sends CDC records to S3 in format:
# {Op: "I/U/D", before: {...}, after: {...}}

def process_cdc(cdc_df, target_path):
    """Apply CDC records to target table using MERGE."""
    
    # Read current target table
    target_df = spark.read.parquet(target_path)
    
    # Get latest change per record (in case of multiple updates)
    latest_changes = cdc_df \
        .withColumn("rn", row_number().over(
            Window.partitionBy("id").orderBy(desc("timestamp"))
        )) \
        .filter(col("rn") == 1)
    
    # Apply changes
    inserts = latest_changes.filter(col("Op") == "I").select("after.*")
    updates = latest_changes.filter(col("Op") == "U").select("after.*")
    deletes = latest_changes.filter(col("Op") == "D").select("before.id")
    
    # Remove deleted and updated records from target
    ids_to_remove = updates.select("id").union(deletes)
    remaining = target_df.join(ids_to_remove, "id", "left_anti")
    
    # Add new and updated records
    result = remaining.union(inserts).union(updates)
    
    result.write.mode("overwrite").parquet(target_path)
```

### Pattern 3: Data Quality Checks

```python
from pyspark.sql.functions import col, count, when, isnan, isnull

def quality_check(df, table_name):
    """Run data quality assertions."""
    total_rows = df.count()
    
    checks = {
        "row_count": total_rows > 0,
        "no_null_ids": df.filter(col("id").isNull()).count() == 0,
        "amount_positive": df.filter(col("amount") <= 0).count() == 0,
        "valid_dates": df.filter(col("date") > current_date()).count() == 0,
        "completeness_95pct": (
            df.filter(col("email").isNotNull()).count() / total_rows > 0.95
        ),
    }
    
    failed = {k: v for k, v in checks.items() if not v}
    
    if failed:
        raise DataQualityException(
            f"Quality checks failed for {table_name}: {failed}"
        )
    
    return True
```

---

## 8. File Format Selection Guide

```
┌─────────────────────────────────────────────────────────────────┐
│  Format Comparison for Data Lakes:                               │
│                                                                   │
│  Format   │ Type      │ Best For              │ Compression      │
│  ─────────┼───────────┼───────────────────────┼─────────────────│
│  Parquet  │ Columnar  │ Analytics (Athena,    │ Snappy (fast)    │
│           │           │ Redshift Spectrum)    │ ZSTD (smaller)   │
│           │           │                       │ GZIP (smallest)  │
│  ─────────┼───────────┼───────────────────────┼─────────────────│
│  ORC      │ Columnar  │ Hive/Presto (legacy)  │ ZLIB, Snappy    │
│  ─────────┼───────────┼───────────────────────┼─────────────────│
│  Avro     │ Row-based │ Streaming, CDC events │ Snappy, Deflate  │
│           │           │ Schema evolution      │                  │
│  ─────────┼───────────┼───────────────────────┼─────────────────│
│  JSON     │ Row-based │ Raw ingestion (Bronze)│ GZIP             │
│           │           │ Human-readable        │                  │
│  ─────────┼───────────┼───────────────────────┼─────────────────│
│  CSV      │ Row-based │ Legacy import/export  │ GZIP             │
│           │           │ Small datasets        │                  │
└─────────────────────────────────────────────────────────────────┘

Recommendation:
├── Bronze layer: JSON or Avro (preserve original format)
├── Silver layer: Parquet with Snappy (fast reads, good compression)
├── Gold layer: Parquet with ZSTD (best compression for storage)
└── Streaming: Avro (schema evolution support)

WHY Parquet is King for Analytics:
├── Columnar: Only reads columns your query needs (skip unused)
├── Predicate pushdown: Filters applied during read (skip rows)
├── Statistics: Min/max per column chunk (skip entire files)
├── Compression: Same-type data compresses 5-10x better
└── Ecosystem: Supported by every AWS analytics service
```

---

## 9. Production Checklist

```yaml
glue_production_checklist:
  job_configuration:
    - [ ] Use Glue 4.0 (latest Spark, faster cold start)
    - [ ] Enable auto-scaling (or right-size workers)
    - [ ] Set job timeout (prevent runaway costs)
    - [ ] Enable job bookmarks for incremental processing
    - [ ] Use pushdown predicates for partition pruning
    - [ ] Enable Spark UI for debugging (spark-event-logs to S3)
    
  data_quality:
    - [ ] Validate schema before processing
    - [ ] Check row counts (alert on anomalies)
    - [ ] Null checks on critical columns
    - [ ] Write bad records to quarantine (don't fail entire job)
    
  monitoring:
    - [ ] CloudWatch metrics: job duration, DPU hours, errors
    - [ ] Alert on: job failure, duration > threshold, data quality fail
    - [ ] Log all transformations for lineage tracking
    
  cost_optimization:
    - [ ] Use Flex execution for non-urgent jobs (34% cheaper)
    - [ ] Right-size workers (G.1X vs G.2X based on memory needs)
    - [ ] Enable auto-scaling (scale down when not needed)
    - [ ] Use partition pruning (don't read entire table)
    - [ ] Compact small files (reduce S3 API costs)
    
  security:
    - [ ] Encrypt data at rest (KMS)
    - [ ] Use Lake Formation for access control
    - [ ] VPC configuration for database connections (JDBC)
    - [ ] Minimal IAM role (only needed S3 paths + Glue actions)

emr_production_checklist:
  cluster_design:
    - [ ] Use EMR 7.x (latest, best performance)
    - [ ] Master: On-Demand, HA (3 masters for production)
    - [ ] Core: On-Demand (if using HDFS) or Spot (if S3 only)
    - [ ] Task: 100% Spot with multiple instance types
    - [ ] Enable managed scaling
    - [ ] Use Graviton instances (20-30% better price/performance)
    
  storage:
    - [ ] Use S3 as primary storage (not HDFS) for most workloads
    - [ ] HDFS only for temporary/shuffle data
    - [ ] Enable S3 server-side encryption
    - [ ] Use EMRFS consistent view (if needed)
    
  security:
    - [ ] Deploy in private subnet
    - [ ] Kerberos for authentication (if multi-tenant)
    - [ ] Lake Formation integration for fine-grained access
    - [ ] Enable encryption in-transit (TLS)
    - [ ] Security configurations for at-rest encryption
```

---

## 10. Learning Path

```
Beginner → Intermediate → Advanced → Expert

Beginner (Week 1-2):
├── Create a Glue Crawler to catalog S3 data
├── Write a simple Glue ETL job (read CSV → write Parquet)
├── Query the result with Athena
├── Understand DynamicFrame vs DataFrame
└── Learn partition strategies

Intermediate (Week 3-4):
├── Implement incremental processing with bookmarks
├── Handle schema evolution in ETL jobs
├── Tune Spark configuration for performance
├── Use Glue Workflows for pipeline orchestration
├── Set up EMR cluster with Spot instances
└── Write Spark job on EMR reading from Glue Catalog

Advanced (Week 5-8):
├── Implement CDC processing pipeline
├── Handle data skew and optimize shuffle operations
├── Implement Iceberg tables with Glue/EMR
├── Cross-account data sharing with Lake Formation
├── Build data quality framework
├── Implement streaming ETL (Glue Streaming / EMR Flink)
└── Cost optimization (Flex, Spot, right-sizing)

Expert (Month 3+):
├── Design enterprise data platform architecture
├── Multi-region data replication strategies
├── Implement data mesh on AWS
├── Optimize for 100TB+ daily processing
├── Build self-service analytics platform
└── Implement ML feature engineering pipelines
```
