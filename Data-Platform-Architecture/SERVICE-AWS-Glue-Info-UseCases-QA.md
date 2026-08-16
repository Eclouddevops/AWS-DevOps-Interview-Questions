# AWS Glue — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | Details |
|-----------|---------|
| **Service Name** | AWS Glue |
| **Category** | Analytics / Data Integration |
| **Type** | Serverless ETL (Extract, Transform, Load) |
| **Engine** | Apache Spark (Glue ETL), Python Shell, Ray |
| **Launched** | August 2017 |
| **Pricing Model** | Pay-per-second (DPU hours) |
| **Key Differentiator** | Serverless Spark + Data Catalog in one service |

---

## 🏗️ What AWS Glue Does

AWS Glue is a **serverless data integration service** that combines three capabilities:

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS GLUE                                   │
│                                                                   │
│  ┌───────────────────┐  ┌───────────────────┐  ┌─────────────┐ │
│  │  DATA CATALOG     │  │  ETL ENGINE       │  │  CRAWLERS   │ │
│  │                   │  │                   │  │             │ │
│  │  Metadata store   │  │  Serverless Spark │  │  Auto-      │ │
│  │  for all your     │  │  for data         │  │  discover   │ │
│  │  data assets      │  │  transformation   │  │  schemas    │ │
│  │                   │  │                   │  │             │ │
│  │  Used by:         │  │  Languages:       │  │  Sources:   │ │
│  │  - Athena         │  │  - PySpark        │  │  - S3       │ │
│  │  - Redshift       │  │  - Scala          │  │  - RDS      │ │
│  │  - EMR            │  │  - Python Shell   │  │  - DynamoDB │ │
│  │  - Lake Formation │  │  - Ray            │  │  - JDBC     │ │
│  └───────────────────┘  └───────────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💰 Pricing (as of 2024)

| Component | Cost | Details |
|-----------|------|---------|
| Data Catalog | First 1M objects free, then $1/100K | Tables, partitions, databases |
| Crawlers | $0.44/DPU-hour | Schema discovery |
| ETL Jobs (Spark) | $0.44/DPU-hour | G.1X = 1 DPU, G.2X = 2 DPU |
| ETL Jobs (Flex) | $0.29/DPU-hour (34% cheaper) | Non-urgent jobs |
| Python Shell | $0.44/DPU-hour (0.0625 or 1 DPU) | Small jobs |
| Interactive Sessions | $0.44/DPU-hour | Development/notebooks |
| Schema Registry | Free | Schema versioning |
| Data Quality | Included in job cost | Rules evaluation |

**Cost Example:**
```
Job with 10 G.1X workers running for 30 minutes:
= 10 workers × 1 DPU × 0.5 hours × $0.44
= $2.20 per run

Daily run for a month:
= $2.20 × 30 = $66/month
```

---

## 📊 Service Limits (Important for Interviews)

| Limit | Default | Adjustable? |
|-------|---------|-------------|
| Max concurrent job runs | 200 per account | Yes |
| Max DPUs per job | 100 (Spark) | Yes |
| Max job runtime | 48 hours | No |
| Max Crawlers per account | 1000 | Yes |
| Max databases in Catalog | 10,000 | Yes |
| Max tables per database | 200,000 | Yes |
| Max partitions per table | 10,000,000 | Yes |
| Max triggers per account | 500 | Yes |
| Job bookmark size | 5 MB | No |

---

## 🎯 Real-World Use Cases

### Use Case 1: Data Lake ETL Pipeline (Most Common)

```
Raw Data (S3)          →  Glue ETL Job  →  Processed Data (S3)
JSON/CSV/Parquet            Transform         Parquet (optimized)
(Bronze layer)              Clean             (Silver/Gold layer)
                            Deduplicate
                            Type-cast
```

**When to use:** Daily/hourly batch processing of data lake files

**Example:** E-commerce company processes 500GB of clickstream data daily from JSON to optimized Parquet for analytics.

---

### Use Case 2: Database Migration / CDC

```
Source DB (RDS/Oracle)  →  Glue JDBC  →  S3 Data Lake
                            Connection      (Parquet)
                            Full load +
                            Incremental
```

**When to use:** Migrating on-prem databases to cloud data lake

**Example:** Bank migrates 200 Oracle tables to S3 data lake for analytics.

---

### Use Case 3: Data Catalog for Analytics

```
S3 Data Lake        →  Glue Crawler  →  Data Catalog  →  Athena/Redshift
(many formats)          (discovers)       (metadata)       (query via SQL)
```

**When to use:** Making data lake queryable without loading into database

**Example:** Data team catalogs 50,000+ files across 200 S3 prefixes so analysts can query with Athena SQL.

---

### Use Case 4: Streaming ETL

```
Kinesis/MSK (real-time)  →  Glue Streaming  →  S3/Redshift
                              ETL Job            (near real-time)
                              (micro-batch)
```

**When to use:** Near real-time processing (30s-5min latency acceptable)

**Example:** IoT company processes sensor data from Kinesis every minute, writing to S3 in Parquet.

---

### Use Case 5: Data Quality Validation

```
S3 Data  →  Glue DQ Rules  →  Pass → Silver Layer
                              Fail → Quarantine + Alert
```

**When to use:** Ensuring data quality before downstream consumption

**Example:** Healthcare company validates patient records meet 15 quality rules before loading to analytics.

---

### Use Case 6: Cross-Account Data Sharing (with Lake Formation)

```
Producer Account        →  Glue Catalog  →  Consumer Accounts
(owns data in S3)           (metadata)        (query via Athena)
                            + Lake Formation
                            (column-level access)
```

**When to use:** Enterprise data mesh / cross-team data sharing

---

## 🔑 Key Features to Know

### 1. Job Bookmarks (Incremental Processing)
```
Without bookmarks: Process ALL data every run (expensive, slow)
With bookmarks: Process ONLY new data since last run (cheap, fast)

Supported for: S3, JDBC sources, Kafka
Tracks: File timestamps (S3), column values (JDBC), offsets (Kafka)
```

### 2. Glue Crawlers (Schema Discovery)
```
Input: S3 path or JDBC connection
Output: Table definition in Data Catalog

What it does:
├── Lists files/tables
├── Samples data (reads portion)
├── Infers schema (column names, types)
├── Detects partitions
└── Creates/updates Catalog table
```

### 3. Glue Data Quality
```
Built-in rule types:
├── Completeness: "column X must be > 99% non-null"
├── Uniqueness: "column X must be unique"
├── Freshness: "data must be < 24 hours old"
├── Validity: "values must be in [A, B, C]"
├── Accuracy: "referential integrity to other table"
└── Custom SQL: Any SQL condition
```

### 4. Glue Schema Registry
```
Purpose: Manage and enforce schemas for streaming data
Supports: Avro, JSON Schema, Protobuf
Compatibility: BACKWARD, FORWARD, FULL, NONE
Integrates with: MSK, Kinesis, Kafka producers/consumers
```

### 5. Auto Scaling (Glue 4.0+)
```
Start with: N workers
Auto-scales: Based on workload needs
Reduces cost: Don't pay for idle workers
Enable: --enable-auto-scaling true
```

---

## ❓ Interview Questions & Answers

### Q1: What is the difference between Glue DynamicFrame and Spark DataFrame?

**Answer:**

| Feature | DynamicFrame | DataFrame |
|---------|-------------|-----------|
| Schema enforcement | Flexible (handles mixed types) | Strict (fails on mismatch) |
| Null handling | Tracks null reasons | Standard null handling |
| Catalog integration | Native (from_catalog) | Requires extra code |
| Bookmark support | Built-in (transformation_ctx) | Manual implementation |
| API availability | Glue-specific transforms | Full Spark SQL API |
| Performance | Slightly slower (tracking) | Faster for complex ops |
| Best for | Reading messy/evolving data | Complex transformations |

**Production pattern:** Read with DynamicFrame → convert to DataFrame for transforms → convert back for writing with bookmarks.

```python
# Read as DynamicFrame (handles schema issues)
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="raw", table_name="events",
    transformation_ctx="source"  # Enables bookmarks
)

# Convert to DataFrame for complex transformations
df = dyf.toDF()
df = df.groupBy("customer_id").agg(sum("amount"))

# Convert back and write with bookmarks
output_dyf = DynamicFrame.fromDF(df, glueContext, "output")
glueContext.write_dynamic_frame.from_options(
    frame=output_dyf,
    connection_type="s3",
    connection_options={"path": "s3://output/"},
    format="parquet",
    transformation_ctx="sink"  # Enables bookmarks
)
```

---

### Q2: Your Glue job processes 100GB daily but sometimes processes 500GB (data spikes). How do you handle this cost-effectively?

**Answer:**

```python
# Solution 1: Enable Auto Scaling
# Job parameter: --enable-auto-scaling true
# Starts with fewer workers, scales UP only when needed
# Scales DOWN when data is smaller → saves money on small days

# Solution 2: Use Flex Execution (34% cheaper)
# For non-time-critical jobs that can wait 1-5 min for capacity
# Job parameter: --execution-class FLEX

# Solution 3: Dynamic worker count based on data size
import boto3

def calculate_workers(s3_path, target_partition_size_mb=256):
    """Calculate workers based on actual data size."""
    s3 = boto3.client('s3')
    bucket, prefix = parse_s3_path(s3_path)
    
    total_size_mb = 0
    paginator = s3.get_paginator('list_objects_v2')
    for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
        for obj in page.get('Contents', []):
            total_size_mb += obj['Size'] / (1024 * 1024)
    
    # Each DPU processes ~256MB efficiently
    recommended_workers = max(2, int(total_size_mb / target_partition_size_mb))
    return min(recommended_workers, 100)  # Cap at 100 DPUs

# Use in Glue job:
workers = calculate_workers("s3://bucket/input/today/")
# Then adjust parallelism: df.repartition(workers * 4)
```

**Best practice:** Enable auto-scaling + Flex execution for non-critical daily jobs. Use fixed workers for SLA-bound jobs.

---

### Q3: How do Glue Job Bookmarks work internally? What happens if a job fails mid-way?

**Answer:**

**How bookmarks work:**
```
Run 1: Process files A, B, C → Success → Bookmark saved: "processed up to file C"
Run 2: Process files D, E (only new since C) → Success → Bookmark: "up to file E"
Run 3: Process files F → Fails mid-way → Bookmark NOT updated (stays at E)
Run 4: Process files F again (retry from E) → Success → Bookmark: "up to F"
```

**Internal mechanism:**
- S3 source: Tracks file modification timestamp + file path
- JDBC source: Tracks a column value (e.g., `last_modified > saved_value`)
- Kafka: Tracks offsets per partition

**Key behavior on failure:**
```
Job fails → job.commit() is NEVER called → Bookmark stays at previous position
Next run → Re-processes from last successful bookmark → No data loss

BUT: If job succeeds on some files then fails:
├── Bookmark is ALL-OR-NOTHING (commit at end)
├── Entire batch is reprocessed
└── Make your target IDEMPOTENT (overwrite mode or deduplicate)
```

**Edge cases:**
```python
# PROBLEM: Job writes partial output then fails
# SOLUTION: Write to temp location, then move atomically
temp_path = f"s3://bucket/temp/{job_run_id}/"
final_path = "s3://bucket/output/"

# Write to temp
df.write.parquet(temp_path)

# Move to final (atomic operation)
# Only after successful write → then commit bookmark
move_s3_objects(temp_path, final_path)
job.commit()  # Bookmark updated only after everything succeeded
```

---

### Q4: You have 10,000 small JSON files (< 1MB each) arriving every hour. How do you design an efficient Glue pipeline?

**Answer:**

**Problem:** Small files = poor performance (each file = separate S3 GET = slow)

**Solution Architecture:**
```
Hourly:
S3 (10K small JSON files)
    │
    ▼
Glue Job (compaction + transform):
    ├── Read all small files
    ├── Transform to Parquet
    ├── Coalesce to ~128MB files
    └── Write to output path

Result: 10,000 files → 5-10 large Parquet files
```

```python
# Efficient Glue job for small files
from awsglue.context import GlueContext
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

# Optimization 1: Use S3 list implementation (faster for many files)
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="small_json_files",
    additional_options={
        "useS3ListImplementation": True,    # Faster S3 listing
        "recurse": True,
        "groupFiles": "inPartition",        # Group small files together
        "groupSize": "134217728"            # Target 128MB groups
    },
    transformation_ctx="source"
)

df = datasource.toDF()

# Optimization 2: Coalesce into optimal file sizes
# Calculate target partitions: total_data / 128MB
total_bytes = df.rdd.map(lambda row: len(str(row))).reduce(lambda a, b: a + b)
target_files = max(1, total_bytes // (128 * 1024 * 1024))

df_compacted = df.coalesce(int(target_files))

# Optimization 3: Write as Parquet with snappy (fast + small)
df_compacted.write \
    .mode("append") \
    .option("compression", "snappy") \
    .partitionBy("year", "month", "day", "hour") \
    .parquet("s3://datalake/silver/events/")
```

**Additional optimizations:**
- Use `groupFiles` option (Glue groups small files automatically)
- Set `groupSize` to target partition size (128MB)
- Use `useS3ListImplementation` for faster file discovery
- Coalesce output to prevent creating more small files

---

### Q5: How does Glue handle schema evolution? What if a source adds a new column?

**Answer:**

**Crawler behavior with schema changes:**
```yaml
schema_change_policies:
  UPDATE_IN_DATABASE:
    behavior: "Automatically adds new columns to table"
    risk: "May rename/remove columns unexpectedly"
    use_when: "Dev/test environments"
    
  LOG:
    behavior: "Logs changes but doesn't update table"
    risk: "Table becomes stale"
    use_when: "When you want manual review"
    
  ADD_NEW_COLUMNS:
    behavior: "Only adds columns, never removes"
    risk: "Old columns stay even if removed from source"
    use_when: "Production (safest option)"
```

**Handling in ETL code:**
```python
# Pattern 1: Schema-on-read with DynamicFrame (handles new columns automatically)
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db", table_name="events"
)
# DynamicFrame automatically includes new columns!

# Pattern 2: Explicit schema mapping (controlled evolution)
mapped = ApplyMapping.apply(
    frame=dyf,
    mappings=[
        ("user_id", "string", "user_id", "string"),
        ("event_type", "string", "event_type", "string"),
        ("amount", "string", "amount", "decimal(10,2)"),
        # New columns are IGNORED unless explicitly mapped
    ]
)

# Pattern 3: ResolveChoice (handle type conflicts)
resolved = dyf.resolveChoice(
    specs=[
        ("amount", "cast:double"),  # If amount is sometimes string/int
        ("timestamp", "cast:timestamp")
    ]
)

# Pattern 4: Detect and alert on new columns
expected_columns = {"user_id", "event_type", "amount", "timestamp"}
actual_columns = set(dyf.toDF().columns)
new_columns = actual_columns - expected_columns
if new_columns:
    alert(f"New columns detected: {new_columns}")
```

---

### Q6: When would you choose Glue over a Lambda function for data processing?

**Answer:**

```
┌──────────────────────────────────────────────────────────────────┐
│  Choose GLUE when:                │  Choose LAMBDA when:          │
├───────────────────────────────────┼───────────────────────────────┤
│ Data > 100MB                      │ Data < 100MB                  │
│ Need Spark operations (joins,     │ Simple transformations        │
│   aggregations, window functions) │ (rename, filter, format)     │
│ Processing time > 15 minutes      │ Processing time < 15 minutes  │
│ Need Data Catalog integration     │ Event-driven (S3 trigger)    │
│ Complex ETL with multiple steps   │ Single-step processing       │
│ Need job bookmarks                │ Stateless processing         │
│ Need to process many file formats │ Single format (e.g., JSON)   │
│ Data lake workloads               │ Real-time stream processing  │
│ Schema discovery needed           │ Schema is known/fixed        │
│ Team knows PySpark/Spark SQL      │ Team knows Python/Node.js    │
├───────────────────────────────────┼───────────────────────────────┤
│ Cost: $0.44/DPU/hour              │ Cost: $0.20/1M invocations   │
│ Min billing: 1 minute             │ Min billing: 1ms              │
│ Cold start: 30-60 seconds         │ Cold start: 100ms-3s         │
│ Max runtime: 48 hours             │ Max runtime: 15 minutes      │
│ Memory: Up to 32GB per worker     │ Memory: Up to 10GB           │
└───────────────────────────────────┴───────────────────────────────┘
```

**Hybrid pattern (common in production):**
```
S3 Event → Lambda (detect new file, validate format)
              │
              ▼ (if valid, trigger Glue)
        Glue ETL Job (heavy transformation)
              │
              ▼ (completion event)
        Lambda (update metadata, notify consumers)
```

---

### Q7: How do you optimize Glue job costs for a pipeline that runs 50 times a day?

**Answer:**

**Cost reduction strategies:**

```
BEFORE optimization:
50 runs × 10 workers × G.2X (2 DPU) × 20 min average
= 50 × 10 × 2 × (20/60) × $0.44 = $146.67/day = $4,400/month

AFTER optimization:
```

| Strategy | Savings | How |
|----------|---------|-----|
| Flex execution | 34% | Non-urgent jobs use FLEX class |
| Auto-scaling | 20-40% | Start small, scale only when needed |
| Right-size workers | 15-25% | G.1X instead of G.2X if < 16GB needed |
| Pushdown predicates | 30-50% | Read less data = process faster |
| Job bookmarks | 50-80% | Process only NEW data each run |
| Combine runs | 40-60% | Run every 4h instead of every 30min |
| Parquet input | 60-70% faster | Column pruning reduces I/O |

```
AFTER optimization:
50 runs × 5 workers (auto-scaled avg) × G.1X (1 DPU) × 8 min (faster)
= 50 × 5 × 1 × (8/60) × $0.29 (Flex) = $9.67/day = $290/month

Savings: $4,400 → $290 = 93% reduction!
```

---

### Q8: Your Glue job reads from a partitioned table with year/month/day/hour partitions. The job scans ALL partitions even though you only need today's data. Why?

**Answer:**

**Root cause:** Missing push-down predicate.

```python
# BAD: Reads ALL partitions (scans entire table)
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="events"
)
# Then filters AFTER reading everything:
df = datasource.toDF().filter(col("year") == "2024")
# Data already read! Filter only removes from memory, not from S3 read

# GOOD: Push-down predicate (filters at CATALOG level)
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="events",
    push_down_predicate="year='2024' and month='06' and day='15'"
)
# Only reads partitions matching the predicate!
# 99% less data read = 99% less cost and time
```

**Why it happens:**
- `push_down_predicate` filters partitions in the Glue Catalog BEFORE reading S3
- Without it, Glue reads ALL partition metadata → lists ALL files → reads ALL data
- The filter in Spark code (`df.filter()`) only removes rows AFTER they're read

**Additional option:** `catalogPartitionPredicate` (uses Catalog expression syntax)
```python
additional_options={
    "catalogPartitionPredicate": "year = '2024' and month >= '05'"
}
```

---

### Q9: How do you handle failed records in a Glue job without failing the entire job?

**Answer:**

```python
from awsglue.transforms import *
from pyspark.sql.functions import col, when, lit
from datetime import datetime

# Pattern 1: Try-Except with DynamicFrame error handling
dyf = glueContext.create_dynamic_frame.from_catalog(
    database="raw", table_name="orders"
)

# ResolveChoice handles type mismatches (instead of failing)
resolved = dyf.resolveChoice(
    specs=[("amount", "cast:double")]  # Cast failures become null
)

# Separate good and bad records
df = resolved.toDF()
good_records = df.filter(col("order_id").isNotNull() & col("amount").isNotNull())
bad_records = df.filter(col("order_id").isNull() | col("amount").isNull())

# Write good records to silver layer
good_records.write.mode("append").parquet("s3://lake/silver/orders/")

# Write bad records to quarantine (for manual review)
if bad_records.count() > 0:
    bad_records.withColumn("quarantine_reason", lit("null_required_field")) \
        .withColumn("quarantine_time", lit(datetime.utcnow().isoformat())) \
        .write.mode("append").parquet("s3://lake/quarantine/orders/")
    
    # Alert if too many failures
    failure_rate = bad_records.count() / (good_records.count() + bad_records.count())
    if failure_rate > 0.05:  # > 5% failure rate
        publish_alert(f"High failure rate: {failure_rate:.2%}")

# Pattern 2: Glue Data Quality (native feature)
from awsglueml.transforms import EvaluateDataQuality

dq_results = EvaluateDataQuality.apply(
    frame=dyf,
    ruleset="""Rules = [
        Completeness "order_id" >= 1.0,
        Completeness "amount" >= 0.95,
        ColumnValues "amount" > 0
    ]""",
    publishing_options={
        "dataQualityEvaluationContext": "orders_check",
        "enableDataQualityCloudWatchMetrics": True
    }
)
# Automatically splits into passed/failed frames
```

---

### Q10: Explain Glue's connection types and when to use each.

**Answer:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Connection Type │ Use Case                    │ Configuration        │
├──────────────────┼─────────────────────────────┼──────────────────────┤
│ S3               │ Read/write data lake files  │ Path, format, options│
│ JDBC             │ Connect to RDS, Redshift,   │ URL, username, pass, │
│                  │ on-prem databases           │ VPC, security group  │
│ Kafka            │ Read from MSK/Kafka topics  │ Bootstrap servers,   │
│                  │                             │ topic, auth          │
│ Kinesis          │ Read from Kinesis streams   │ Stream ARN, region   │
│ MongoDB          │ NoSQL database access       │ Connection string    │
│ DynamoDB         │ Read/write DynamoDB tables  │ Table name, region   │
│ Redshift         │ Direct Redshift COPY/UNLOAD │ Cluster, temp S3     │
│ Custom           │ Any JDBC-compatible source  │ Custom JDBC driver   │
│ Marketplace      │ 3rd-party connectors        │ Varies by connector  │
│ Network          │ VPC access (ENI-based)      │ VPC, subnet, SG      │
└──────────────────┴─────────────────────────────┴──────────────────────┘

VPC Connection (critical for production):
├── Required when: Accessing RDS, Redshift, on-prem via Direct Connect
├── Creates: Elastic Network Interface (ENI) in your VPC
├── Needs: Subnet with available IPs, Security Group allowing traffic
├── Gotcha: Glue needs NAT Gateway to reach S3 (or S3 VPC Endpoint)
└── Best practice: Always use VPC Endpoint for S3 (free, faster)
```

---

## 🆚 Glue vs Competitors

| Feature | AWS Glue | Databricks | Azure Data Factory | GCP Dataflow |
|---------|----------|------------|-------------------|--------------|
| Engine | Spark | Spark + Delta | SSIS + Spark | Apache Beam |
| Serverless | ✅ | ✅ (Serverless) | ✅ | ✅ |
| Data Catalog | Built-in | Unity Catalog | Purview | Data Catalog |
| Streaming | ✅ (Spark Streaming) | ✅ (Structured Streaming) | ✅ | ✅ (native) |
| Cost model | Per-DPU-second | Per-DBU-second | Per-activity | Per-worker-hour |
| Best for | AWS-native data lakes | Multi-cloud, ML-heavy | Azure ecosystem | GCP ecosystem |
| Unique feature | Catalog + Lake Formation | Notebooks + MLflow | Visual ETL designer | Unified batch/stream |

---

## 🏆 Production Best Practices Summary

```yaml
development:
  - Use Interactive Sessions for development (faster than running full jobs)
  - Enable Spark UI for debugging (--enable-spark-ui true)
  - Test with small dataset first, then scale
  - Use Glue Studio visual editor for simple jobs

performance:
  - Always use push-down predicates (partition pruning)
  - Enable auto-scaling for variable workloads
  - Use Parquet/ORC format (not JSON/CSV) for reading
  - Repartition data before writing (avoid small files)
  - Use G.2X workers for memory-intensive operations only

cost:
  - Flex execution for non-urgent jobs (34% savings)
  - Job bookmarks for incremental processing (50-80% savings)
  - Right-size workers (don't default to G.2X)
  - Set job timeouts (prevent runaway costs)
  - Monitor DPU utilization via CloudWatch

reliability:
  - Implement retry logic (Glue job auto-retry setting)
  - Use job bookmarks for exactly-once semantics
  - Write to quarantine for bad records (don't fail entire job)
  - Enable data quality checks before writing to gold layer
  - Monitor: job duration, DPU hours, data quality metrics
```
