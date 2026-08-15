# Data Platform Mandatory Skills - Deep Tricky Interview Questions & Answers

## AWS Glue, Redshift, EMR, MSK, MWAA, Lake Formation

---

### Q1: Your AWS Glue job reads from a partitioned S3 data lake (year/month/day/hour) with 2 million partitions in the Glue Catalog. The job takes 25 minutes just to list partitions before any processing starts. How do you fix this?

**Answer:**

**Root Cause:** Glue Catalog `GetPartitions` API call is paginated (max 1000 partitions per call). With 2M partitions, it needs 2000+ API calls just to enumerate them. This is the **partition explosion problem**.

**Solution 1: Partition Filtering (Push-Down Predicate)**

```python
# BAD: Reads ALL partitions then filters
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="datalake",
    table_name="events"
)
# This fetches metadata for ALL 2M partitions!

# GOOD: Push-down predicate filters partitions at catalog level
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="datalake",
    table_name="events",
    push_down_predicate="year='2024' and month='06'",
    # Only fetches ~720 partitions (30 days × 24 hours)
    additional_options={
        "catalogPartitionPredicate": "year='2024' and month >= '05'"
    }
)
```

**Solution 2: Partition Indexes (Glue Catalog Feature)**

```python
# Create partition index on frequently filtered columns
import boto3
glue = boto3.client('glue')

glue.create_partition_index(
    DatabaseName='datalake',
    TableName='events',
    PartitionIndex={
        'Keys': ['year', 'month', 'day'],
        'IndexName': 'date-index'
    }
)
# NOW: GetPartitions with filter uses INDEX instead of full scan
# Reduces partition lookup from 25 min to < 5 seconds
```

**Solution 3: Migrate to Apache Iceberg (Eliminate Partition Problem)**

```python
# Iceberg stores partition info in metadata files (not catalog API)
# No matter how many partitions — planning takes seconds

spark.sql("""
    CREATE TABLE glue_catalog.datalake.events_v2 (
        event_id STRING,
        user_id STRING,
        event_type STRING,
        payload STRING,
        event_time TIMESTAMP
    )
    USING iceberg
    PARTITIONED BY (days(event_time))
    TBLPROPERTIES (
        'format-version' = '2'
    )
""")

# Migrate data
spark.sql("""
    INSERT INTO glue_catalog.datalake.events_v2
    SELECT * FROM glue_catalog.datalake.events
    WHERE year >= '2024'
""")

# Iceberg reads partition metadata from S3 manifest files
# Planning: O(1) regardless of partition count
# Bonus: Hidden partitioning — queries don't need to know partition structure
```

**Solution 4: S3 Listing Optimization**

```python
# If not using Catalog but reading S3 directly:
datasource = glueContext.create_dynamic_frame.from_options(
    connection_type="s3",
    connection_options={
        "paths": ["s3://datalake/events/year=2024/month=06/"],
        "useS3ListImplementation": True,  # Faster S3 listing
        "recurse": True,
        # Exclude _SUCCESS files and temp directories
        "exclusions": "[\"**/_SUCCESS\", \"**/_temporary/**\"]"
    },
    format="parquet"
)
```

**Solution 5: Reduce Partition Granularity**

```yaml
# Instead of: year/month/day/hour (365 × 24 = 8,760 partitions/year)
# Use: year/month/day (365 partitions/year)
# Within each partition, use file-level organization (sorted by hour)

# For Iceberg: Use bucket partitioning for high-cardinality
spark.sql("""
    ALTER TABLE events_v2 
    SET PARTITION SPEC (days(event_time), bucket(16, user_id))
""")
# 365 days × 16 buckets = 5,840 partitions/year (manageable)
```

---

### Q2: Your Redshift cluster shows "disk full" errors but you're using RA3 nodes with managed storage. You have 5TB of actual data but Redshift reports 12TB storage used. What's happening and how do you fix it?

**Answer:**

**Root Cause Analysis:**

```sql
-- Check actual disk usage per table
SELECT 
    schema as table_schema,
    "table" as table_name,
    size as size_mb,
    tbl_rows,
    unsorted as pct_unsorted,
    stats_off as stats_accuracy
FROM svv_table_info
ORDER BY size DESC
LIMIT 20;

-- Check for tombstoned rows (marked for deletion but not vacuumed)
SELECT 
    "table", 
    size as current_mb,
    pct_used,
    empty as empty_blocks_pct,
    unsorted,
    tbl_rows,
    deleted_rows  -- THIS! Deleted rows still consuming space
FROM svv_table_info
WHERE deleted_rows > 0
ORDER BY deleted_rows DESC;
```

**Why 12TB with 5TB data — The 7TB ghost data:**

1. **Tombstoned rows (DELETE without VACUUM):** 3TB
2. **Unsorted regions (INSERT without SORT):** 2TB
3. **Temporary result sets (complex queries):** 1TB
4. **Snapshots/copies (cluster resize artifacts):** 1TB

**Fix 1: VACUUM (Reclaim space from deletes)**

```sql
-- Full vacuum: reclaims space + re-sorts
-- WARNING: This is resource-intensive, run during maintenance window
VACUUM FULL schema_name.large_table;

-- For all tables (background, less disruptive):
VACUUM DELETE ONLY;  -- Just reclaim deleted rows
VACUUM SORT ONLY;   -- Just re-sort unsorted region
VACUUM REINDEX;     -- Rebuild interleaved sort key indexes

-- Check vacuum progress:
SELECT * FROM svv_vacuum_progress;

-- Auto vacuum settings (should be enabled):
-- Redshift runs auto-vacuum in background, but it's conservative
-- For large tables with heavy deletes, manual vacuum is needed
```

**Fix 2: Deep Copy (Faster than VACUUM for heavily fragmented tables)**

```sql
-- Instead of vacuuming a 3TB table (hours), deep copy (minutes):

-- Method 1: CTAS (Create Table As Select)
CREATE TABLE schema_name.large_table_new
DISTKEY(customer_id)
SORTKEY(event_date)
AS SELECT * FROM schema_name.large_table;

-- Rename swap
ALTER TABLE schema_name.large_table RENAME TO large_table_old;
ALTER TABLE schema_name.large_table_new RENAME TO large_table;

-- Drop old (instantly reclaims all space)
DROP TABLE schema_name.large_table_old;

-- Method 2: Unload + Copy (for very large tables)
UNLOAD ('SELECT * FROM schema_name.large_table')
TO 's3://temp-bucket/deep-copy/large_table/'
FORMAT AS PARQUET;

TRUNCATE schema_name.large_table;

COPY schema_name.large_table
FROM 's3://temp-bucket/deep-copy/large_table/'
FORMAT AS PARQUET;
```

**Fix 3: Table Design Optimization (Prevent future bloat)**

```sql
-- Use MERGE instead of DELETE + INSERT (reduces tombstones)
MERGE INTO target_table t
USING staging_table s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT VALUES (*);

-- Use TRUNCATE instead of DELETE (when replacing entire table)
-- TRUNCATE is instant and doesn't create tombstones
TRUNCATE schema_name.daily_snapshot;
INSERT INTO schema_name.daily_snapshot SELECT * FROM staging;

-- Optimize sort keys to reduce unsorted data
-- Compound sort key on most common filter column:
ALTER TABLE large_table ALTER COMPOUND SORTKEY(event_date, customer_id);

-- Monitor and alert on table health:
CREATE VIEW admin.table_health AS
SELECT 
    "table",
    size as mb,
    CASE WHEN unsorted > 20 THEN 'NEEDS_SORT'
         WHEN deleted_rows > tbl_rows * 0.2 THEN 'NEEDS_VACUUM'
         ELSE 'HEALTHY' END as status
FROM svv_table_info;
```

**Fix 4: Prevent Temp Space Exhaustion**

```sql
-- Large queries spill to disk. Limit spill:
SET query_execution_timeout = 1800000;  -- 30 min max

-- Use WLM to limit memory per query
-- Increase WLM memory allocation for ETL queue

-- Identify queries using excessive temp space:
SELECT 
    query, 
    segment,
    step,
    rows,
    bytes/1024/1024 as mb_spilled,
    label
FROM svl_query_summary
WHERE is_diskbased = 't'
ORDER BY bytes DESC
LIMIT 20;
```

---

### Q3: Your MSK cluster has 3 brokers, each with 2TB storage. You notice replication lag increasing and consumer lag growing. One broker's `UnderReplicatedPartitions` metric is consistently high. Diagnose and fix.

**Answer:**

**Systematic Diagnosis:**

```bash
# Step 1: Check broker metrics
# Key metrics to check:
# - UnderReplicatedPartitions (should be 0)
# - ActiveControllerCount (should be 1)
# - OfflinePartitionsCount (should be 0)
# - BytesInPerSec / BytesOutPerSec (throughput)
# - NetworkRxPacketsPerSec (network saturation)
# - ReplicaMaxLag
# - BrokerStorageUtilization

# Step 2: Identify the struggling broker
aws kafka describe-cluster --cluster-arn $CLUSTER_ARN
```

**Root Causes (in order of likelihood):**

**Cause 1: Storage Throttling (Most Common)**
```
- EBS gp2/gp3 volumes have IOPS and throughput limits
- When broker handles heavy replication + consumer reads → IOPS exhausted
- Symptoms: High VolumeQueueLength, high replication lag
```

```bash
# Check CloudWatch: AWS/Kafka BrokerStorageUtilization
# If > 85%, MSK may throttle

# Fix: Increase storage (live, no downtime)
aws kafka update-broker-storage \
  --cluster-arn $CLUSTER_ARN \
  --target-broker-ebs-volume-info '[
    {"KafkaBrokerNodeId":"1","VolumeSizeGB":4000},
    {"KafkaBrokerNodeId":"2","VolumeSizeGB":4000},
    {"KafkaBrokerNodeId":"3","VolumeSizeGB":4000}
  ]'

# Or enable provisioned throughput:
aws kafka update-broker-storage \
  --cluster-arn $CLUSTER_ARN \
  --target-broker-ebs-volume-info '[
    {
      "KafkaBrokerNodeId":"1",
      "VolumeSizeGB":4000,
      "ProvisionedThroughput": {
        "Enabled": true,
        "VolumeThroughput": 250
      }
    }
  ]'
```

**Cause 2: Unbalanced Partition Distribution**
```bash
# Check partition distribution across brokers
kafka-topics.sh --bootstrap-server $BROKER \
  --describe --topic high-volume-topic

# If one broker has significantly more leader partitions:
# Use kafka-reassign-partitions or cruise-control

# Auto-rebalance with MSK (enable cruise control):
# MSK Configuration: auto.leader.rebalance.enable=true
```

**Cause 3: Network Saturation**
```
- Each broker instance type has network limits
- kafka.m5.large: 10 Gbps
- With replication factor 3: each byte written generates 2x network for replication
- 1 GB/s produce → 2 GB/s replication → 3 GB/s total per broker
```

```bash
# Fix: Scale UP broker instance type
aws kafka update-broker-type \
  --cluster-arn $CLUSTER_ARN \
  --target-instance-type kafka.m5.4xlarge
# This is a rolling update (one broker at a time)
```

**Cause 4: Consumer Reading from Follower (Fetch from leader only)**
```properties
# MSK Configuration fix:
# Ensure consumers read from leader (default)
# If using rack-aware consumers with fetch from closest replica:
replica.fetch.max.bytes=10485760
replica.fetch.min.bytes=1
replica.fetch.wait.max.ms=500
num.replica.fetchers=4  # Increase replication threads
```

**Cause 5: Topic with Too Many Partitions on Single Broker**
```bash
# Check partition count per broker:
kafka-topics.sh --bootstrap-server $BROKER --describe | \
  grep "Leader:" | awk '{print $4}' | sort | uniq -c | sort -rn

# If broker 2 has 5000 partitions and others have 1000:
# Rebalance partitions across brokers

# Generate reassignment plan:
kafka-reassign-partitions.sh --bootstrap-server $BROKER \
  --topics-to-move-json-file topics.json \
  --broker-list "1,2,3" \
  --generate

# Execute (throttled to prevent impact):
kafka-reassign-partitions.sh --bootstrap-server $BROKER \
  --reassignment-json-file plan.json \
  --throttle 50000000 \
  --execute
```

**Monitoring Dashboard for Prevention:**
```hcl
resource "aws_cloudwatch_metric_alarm" "under_replicated" {
  alarm_name          = "msk-under-replicated-partitions"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "UnderReplicatedPartitions"
  namespace           = "AWS/Kafka"
  period              = 60
  statistic           = "Maximum"
  threshold           = 0
  alarm_description   = "Partitions are under-replicated - data durability at risk"
  
  dimensions = {
    "Cluster Name" = var.cluster_name
  }
  
  alarm_actions = [aws_sns_topic.critical_alerts.arn]
}
```

---

### Q4: Your MWAA (Airflow) environment shows "scheduler heartbeat timeout" errors and tasks are getting stuck in "queued" state for hours. The environment class is mw1.medium with 5 workers. What's wrong?

**Answer:**

**Diagnosis Framework:**

```
Symptoms:
- Scheduler heartbeat timeout → Scheduler is overloaded/dying
- Tasks stuck in "queued" → Workers not picking up tasks OR queue is full

Root cause tree:
├── Scheduler overload
│   ├── Too many DAGs to parse (DAG parsing takes > heartbeat interval)
│   ├── Complex DAGs with many tasks (scheduling overhead)
│   └── Scheduler memory exhaustion
├── Worker saturation
│   ├── All worker slots occupied (tasks running > capacity)
│   ├── Tasks taking too long (blocking slots)
│   └── Worker pods dying (OOM)
└── Celery queue issues
    ├── Queue depth growing (production rate > consumption rate)
    └── Message broker connection issues
```

**Step 1: Identify the Bottleneck**

```python
# Check scheduler health via Airflow CLI (MWAA exec)
# aws mwaa create-cli-token → use Airflow REST API

import requests

# Check scheduler info
response = requests.get(
    f"{MWAA_WEB_URL}/api/v1/health",
    headers={"Authorization": f"Bearer {token}"}
)
# Look for: scheduler.latest_heartbeat, scheduler.status

# Check DAG parsing times
response = requests.get(
    f"{MWAA_WEB_URL}/api/v1/dagSources",
    headers={"Authorization": f"Bearer {token}"}
)
```

**Step 2: Fix Scheduler Overload**

```python
# MWAA Configuration overrides:
mwaa_configuration = {
    # Reduce scheduler pressure
    "scheduler.min_file_process_interval": "60",    # Parse DAGs every 60s (not 30s default)
    "scheduler.dag_dir_list_interval": "120",       # Scan DAG folder every 2 min
    "scheduler.parsing_processes": "2",             # Limit parallel parsing
    "core.dagbag_import_timeout": "30",             # Kill slow DAG parsing after 30s
    "core.dag_file_processor_timeout": "120",       # Max time to process a DAG file
    
    # Reduce scheduler memory usage
    "scheduler.max_dagruns_to_create_per_loop": "10",
    "scheduler.max_dagruns_per_loop_to_schedule": "20",
    "scheduler.max_tis_per_query": "512",           # Reduce from default 512 if needed
    
    # Critical: Increase heartbeat tolerance
    "scheduler.scheduler_heartbeat_sec": "10",      # More frequent heartbeat
    "scheduler.scheduler_health_check_threshold": "60",  # More tolerance before marking unhealthy
}
```

**Step 3: Fix Worker Saturation**

```python
mwaa_configuration_workers = {
    # Celery worker tuning
    "celery.worker_concurrency": "8",               # Tasks per worker (reduce if OOM)
    "celery.worker_autoscale": "8,2",               # max,min workers per node
    
    # Task execution limits
    "core.parallelism": "32",                       # Max tasks running across ALL workers
    "core.max_active_tasks_per_dag": "16",          # Per-DAG concurrency
    "core.max_active_runs_per_dag": "3",            # Concurrent DAG runs
    
    # Pool management (limit specific task types)
    # Create pools in Airflow UI: "heavy_compute" = 4 slots
    # "api_calls" = 10 slots (prevent API throttling)
}
```

**Step 4: Optimize DAG Design (Root Cause Fix)**

```python
# BAD: Heavy computation in DAG file (runs during PARSING!)
import pandas as pd
df = pd.read_csv("s3://bucket/config.csv")  # THIS RUNS ON SCHEDULER!

# GOOD: Only lightweight code in DAG file
from airflow import DAG
from airflow.decorators import task

with DAG("optimized_dag", schedule_interval="@daily") as dag:
    
    @task(pool="heavy_compute", pool_slots=1)
    def heavy_processing():
        # Heavy code runs in WORKER, not scheduler
        import pandas as pd
        df = pd.read_csv("s3://bucket/config.csv")
        # Process...

# BAD: 1000 tasks generated dynamically (scheduler memory explosion)
for i in range(1000):
    PythonOperator(task_id=f"task_{i}", ...)

# GOOD: Use Dynamic Task Mapping (Airflow 2.3+)
@task
def generate_work_items():
    return list(range(1000))

@task
def process_item(item):
    # Each item gets its own task instance
    pass

items = generate_work_items()
process_item.expand(item=items)  # Dynamic, memory-efficient
```

**Step 5: Scale the Environment**

```hcl
resource "aws_mwaa_environment" "production" {
  name               = "data-platform-airflow"
  airflow_version    = "2.9.2"
  
  # Upgrade environment class for more scheduler resources
  environment_class  = "mw1.large"   # 2x CPU/memory vs medium
  
  # Scale workers
  min_workers = 5   # Always have 5 ready
  max_workers = 25  # Burst to 25 for peak
  
  # HA Scheduler (2 schedulers)
  schedulers = 2    # Active-active schedulers
  
  # Network (ensure subnets have enough ENIs)
  network_configuration {
    security_group_ids = [aws_security_group.mwaa.id]
    subnet_ids         = var.private_subnet_ids  # Must be in 2 AZs
  }
}
```

**Step 6: Task-Level Patterns to Prevent Queue Buildup**

```python
# Pattern: Offload to Glue/EMR (don't use Airflow workers for compute)
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.providers.amazon.aws.sensors.glue import GlueJobSensor

# Submit to Glue (Airflow worker is FREE immediately)
submit = GlueJobOperator(
    task_id='submit_etl',
    job_name='heavy_transform',
    wait_for_completion=False,  # Don't block worker!
)

# Sensor uses 'reschedule' mode (frees worker slot while waiting)
wait = GlueJobSensor(
    task_id='wait_etl',
    job_name='heavy_transform',
    run_id="{{ task_instance.xcom_pull(task_ids='submit_etl') }}",
    mode='reschedule',      # FREE the worker slot between checks
    poke_interval=60,
    timeout=7200
)

submit >> wait
```

---

### Q5: You have a Redshift data warehouse and an S3 data lake. Business users want to query both seamlessly. Some tables are in Redshift (hot data, last 90 days) and some in S3 (cold data, historical). How do you architect a unified query layer?

**Answer:**

**Architecture: Lakehouse Pattern with Federated Query**

```
┌─────────────────────────────────────────────────────────────────┐
│  Unified Query Layer (User's Perspective: Single SQL Interface)  │
│                                                                   │
│  SELECT * FROM sales                                             │
│  WHERE sale_date BETWEEN '2020-01-01' AND '2024-06-15'          │
│  -- Transparently queries BOTH Redshift AND S3                   │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
         ┌──────────▼──────────┐   ┌─────────▼──────────────┐
         │   Redshift Local    │   │   Redshift Spectrum     │
         │   (Hot: Last 90d)   │   │   (Cold: S3 Historical)│
         │                     │   │                         │
         │   - Fast queries    │   │   - Pushdown filters    │
         │   - Materialized    │   │   - Columnar scan       │
         │   - Aggregated      │   │   - Parquet/ORC         │
         └─────────────────────┘   └─────────────────────────┘
```

**Implementation:**

```sql
-- Step 1: Create external schema pointing to S3 (via Glue Catalog)
CREATE EXTERNAL SCHEMA datalake
FROM DATA CATALOG
DATABASE 'analytics_db'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole'
REGION 'us-east-1';

-- Step 2: Create local Redshift table (hot data)
CREATE TABLE local_schema.sales (
    sale_id BIGINT,
    customer_id BIGINT,
    product_id INT,
    amount DECIMAL(10,2),
    sale_date DATE,
    region VARCHAR(50)
)
DISTKEY(customer_id)
SORTKEY(sale_date)
DISTSTYLE KEY;

-- Step 3: Create UNIFIED view (transparent to users)
CREATE OR REPLACE VIEW analytics.v_sales AS
-- Hot data (last 90 days) from Redshift local storage
SELECT sale_id, customer_id, product_id, amount, sale_date, region
FROM local_schema.sales
WHERE sale_date >= DATEADD(day, -90, CURRENT_DATE)

UNION ALL

-- Cold data (historical) from S3 via Spectrum
SELECT sale_id, customer_id, product_id, amount, sale_date, region
FROM datalake.sales_historical
WHERE sale_date < DATEADD(day, -90, CURRENT_DATE);

-- Step 4: Users query the view (don't know or care about storage location)
SELECT 
    region,
    DATE_TRUNC('month', sale_date) as month,
    SUM(amount) as revenue,
    COUNT(DISTINCT customer_id) as unique_customers
FROM analytics.v_sales
WHERE sale_date BETWEEN '2022-01-01' AND '2024-06-15'
GROUP BY 1, 2
ORDER BY 1, 2;
-- Redshift optimizer automatically routes:
-- 2024-03-15 to 2024-06-15 → local Redshift (fast)
-- 2022-01-01 to 2024-03-14 → Spectrum/S3 (scan historical parquet)
```

**Step 5: Automated Data Lifecycle (Hot → Cold)**

```python
# Airflow DAG: Move aged data from Redshift to S3
from airflow import DAG
from airflow.providers.amazon.aws.operators.redshift_sql import RedshiftSQLOperator

with DAG("data_lifecycle_hot_to_cold", schedule_interval="@daily") as dag:
    
    # Unload old data to S3 (Parquet, partitioned)
    unload_cold = RedshiftSQLOperator(
        task_id='unload_old_data',
        sql="""
            UNLOAD ('
                SELECT * FROM local_schema.sales
                WHERE sale_date < DATEADD(day, -90, CURRENT_DATE)
                  AND sale_date >= DATEADD(day, -91, CURRENT_DATE)
            ')
            TO 's3://datalake/sales_historical/year={{ ds[:4] }}/month={{ ds[5:7] }}/'
            IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftUnloadRole'
            FORMAT AS PARQUET
            PARTITION BY (region)
            ALLOWOVERWRITE
        """
    )
    
    # Add partition to Glue Catalog
    add_partition = GlueOperator(
        task_id='add_partition',
        script='add_partition.py'
    )
    
    # Delete from Redshift local (free up space)
    delete_local = RedshiftSQLOperator(
        task_id='delete_from_redshift',
        sql="""
            DELETE FROM local_schema.sales
            WHERE sale_date < DATEADD(day, -91, CURRENT_DATE)
        """
    )
    
    # Vacuum to reclaim space
    vacuum = RedshiftSQLOperator(
        task_id='vacuum',
        sql="VACUUM DELETE ONLY local_schema.sales;"
    )
    
    unload_cold >> add_partition >> delete_local >> vacuum
```

**Alternative: Redshift Serverless + Data Sharing**

```sql
-- For teams that need ad-hoc historical queries:
-- Use Redshift Serverless (pay per query, auto-scales)

-- Producer (provisioned cluster): Share hot data
CREATE DATASHARE live_data_share;
ALTER DATASHARE live_data_share ADD SCHEMA local_schema;
ALTER DATASHARE live_data_share ADD ALL TABLES IN SCHEMA local_schema;
GRANT USAGE ON DATASHARE live_data_share TO NAMESPACE 'serverless-ns-id';

-- Consumer (serverless): Query both shared + Spectrum data
CREATE DATABASE live_data FROM DATASHARE live_data_share
  OF NAMESPACE 'provisioned-ns-id';

-- Now serverless can query:
-- live_data.local_schema.sales (shared from provisioned, no copy)
-- datalake.sales_historical (Spectrum, direct S3 read)
```

---

### Q6: Your EMR Spark job processes 50TB of data with a JOIN between a 45TB fact table and a 500MB dimension table. The job fails with "Container killed by YARN for exceeding memory limits" during the shuffle phase. How do you fix without just adding more hardware?

**Answer:**

**Diagnosis:**

```
The 500MB dim table is small enough to BROADCAST, but Spark may not
auto-broadcast if it doesn't know the table size (e.g., reading from S3).

During a regular shuffle join of 45TB:
- Data shuffles across ALL executors (massive network I/O)
- Shuffle partitions default = 200 → each partition = 45TB/200 = 225GB!
- Single partition > executor memory → OOM
```

**Fix 1: Broadcast the Small Table (Eliminate Shuffle)**

```python
from pyspark.sql.functions import broadcast

# Force broadcast of dimension table (500MB fits in memory)
fact_df = spark.read.parquet("s3://datalake/fact_table/")
dim_df = spark.read.parquet("s3://datalake/dim_table/")  # 500MB

# This eliminates shuffle entirely for this join
result = fact_df.join(broadcast(dim_df), "product_id", "left")

# If Spark doesn't auto-broadcast, increase threshold:
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "600m")  # 600MB
```

**Fix 2: Fix Shuffle Partition Count (If Broadcast Not Applicable)**

```python
# 200 default partitions for 45TB → 225GB per partition (TOO BIG!)
# Target: 128-256MB per partition for optimal performance

# Calculate: 45TB / 256MB = ~175,000 partitions
spark.conf.set("spark.sql.shuffle.partitions", "50000")
# Start with 50K and increase if still OOM

# Better: Use Adaptive Query Execution (auto-tunes partitions)
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.initialPartitionNum", "50000")
spark.conf.set("spark.sql.adaptive.coalescePartitions.minPartitionSize", "64m")
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "256m")
```

**Fix 3: Handle Data Skew (Common OOM Cause)**

```python
# If one join key (e.g., product_id = "UNKNOWN") has 80% of rows:
# One partition gets 80% of data → OOM

# Solution A: AQE Skew Join (Spark 3.x)
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256m")
# AQE detects skewed partitions and splits them automatically

# Solution B: Manual Salt Key (for Spark 2.x or extreme skew)
from pyspark.sql.functions import col, lit, rand, concat, floor

# Add salt to fact table (split skewed key into N sub-keys)
SALT_BUCKETS = 100
fact_salted = fact_df.withColumn("salt", floor(rand() * SALT_BUCKETS))
fact_salted = fact_salted.withColumn(
    "join_key_salted", 
    concat(col("product_id"), lit("_"), col("salt"))
)

# Explode dimension table to match all salt values
from pyspark.sql.functions import explode, array
dim_exploded = dim_df.withColumn(
    "salt", explode(array([lit(i) for i in range(SALT_BUCKETS)]))
)
dim_exploded = dim_exploded.withColumn(
    "join_key_salted",
    concat(col("product_id"), lit("_"), col("salt"))
)

# Join on salted key (evenly distributed!)
result = fact_salted.join(dim_exploded, "join_key_salted", "left")
result = result.drop("salt", "join_key_salted")
```

**Fix 4: Memory Configuration**

```python
# EMR Spark memory settings for 50TB job:
spark_config = {
    # Executor memory (80% of instance RAM)
    "spark.executor.memory": "56g",          # r5.4xlarge has 128GB
    "spark.executor.memoryOverhead": "8g",   # Off-heap for shuffle
    "spark.executor.cores": "5",             # 5 cores per executor
    
    # Memory management
    "spark.memory.fraction": "0.8",          # 80% for execution+storage
    "spark.memory.storageFraction": "0.2",   # Less caching, more shuffle
    
    # Shuffle optimization
    "spark.shuffle.compress": "true",
    "spark.shuffle.spill.compress": "true",
    "spark.reducer.maxSizeInFlight": "96m",
    "spark.shuffle.file.buffer": "1m",
    "spark.unsafe.sorter.spill.reader.buffer.size": "1m",
    
    # Off-heap (prevents GC pauses on large heaps)
    "spark.memory.offHeap.enabled": "true",
    "spark.memory.offHeap.size": "16g",
}
```

**Fix 5: Pre-partition / Bucket the Fact Table**

```python
# If this join runs daily, pre-bucket the fact table:
fact_df.write \
    .bucketBy(1024, "product_id") \
    .sortBy("product_id") \
    .saveAsTable("fact_table_bucketed")

dim_df.write \
    .bucketBy(1024, "product_id") \
    .sortBy("product_id") \
    .saveAsTable("dim_table_bucketed")

# Bucket join = NO SHUFFLE (data pre-colocated)
spark.conf.set("spark.sql.sources.bucketing.enabled", "true")
spark.conf.set("spark.sql.bucketing.coalesceBucketsInJoin.enabled", "true")

result = spark.table("fact_table_bucketed").join(
    spark.table("dim_table_bucketed"), "product_id"
)
# Zero shuffle! Both tables bucketed by same key and same # of buckets
```

---

### Q7: Your Lake Formation setup works for Athena and Glue but EMR jobs are bypassing Lake Formation permissions and accessing S3 directly. How do you enforce Lake Formation access control for EMR?

**Answer:**

**Why EMR Bypasses Lake Formation:**

```
Default EMR behavior:
EMR → IAM Role → S3 (direct access via IAM policy)
       ↑ This bypasses Lake Formation entirely!

Lake Formation controls:
Athena/Glue → Lake Formation → Vended credentials → S3
               ↑ These services are "integrated" with LF
```

**The Problem:**
EMR uses its EC2 instance profile (IAM role) to access S3 directly. Lake Formation permissions are NOT evaluated because EMR is not using Lake Formation's credential vending mechanism.

**Solution: Enable Lake Formation with EMR (Runtime Role)**

```hcl
# Step 1: Enable Lake Formation for EMR
# EMR must use "Runtime Roles" (not instance profile for data access)

resource "aws_emr_cluster" "lf_enabled" {
  name          = "lf-governed-emr"
  release_label = "emr-6.15.0"  # Must be 6.15+ for LF integration
  
  # Enable Lake Formation
  configurations_json = jsonencode([
    {
      "Classification": "spark-hive-site",
      "Properties": {
        "hive.metastore.client.factory.class": "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory",
        # Lake Formation integration
        "aws.glue.catalog.separator": "/",
      }
    },
    {
      "Classification": "spark-defaults",
      "Properties": {
        "spark.sql.catalog.glue_catalog": "org.apache.iceberg.spark.SparkCatalog",
        "spark.sql.catalog.glue_catalog.catalog-impl": "org.apache.iceberg.aws.glue.GlueCatalog",
        # Enable LF credential vending
        "spark.hadoop.fs.s3.authorization.enabled": "true",
        "spark.hadoop.fs.s3.authorization.roleArn": "arn:aws:iam::123456789:role/EMRLakeFormationRole"
      }
    },
    {
      "Classification": "emrfs-site",
      "Properties": {
        # CRITICAL: Use Lake Formation for S3 access
        "fs.s3.authorization.enabled": "true",
        "fs.s3.authorization.mode": "lake-formation"
      }
    }
  ])
  
  # EC2 instance profile (minimal permissions - no direct S3 data access!)
  ec2_attributes {
    instance_profile = aws_iam_instance_profile.emr_minimal.arn
  }
}

# Step 2: Minimal EC2 role (NO S3 data access)
resource "aws_iam_role" "emr_ec2_minimal" {
  name = "EMR-EC2-Minimal"
  
  inline_policy {
    name = "minimal"
    policy = jsonencode({
      Version = "2012-10-17"
      Statement = [
        {
          Effect = "Allow"
          Action = [
            # Only EMR operational access (NOT s3:GetObject on data buckets!)
            "s3:GetObject", "s3:ListBucket"
          ]
          Resource = [
            "arn:aws:s3:::emr-logs-*",
            "arn:aws:s3:::emr-logs-*/*",
            "arn:aws:s3:::elasticmapreduce/*"
          ]
        },
        {
          # Lake Formation credential vending
          Effect = "Allow"
          Action = [
            "lakeformation:GetDataAccess",
            "lakeformation:GetTemporaryGlueTableCredentials",
            "lakeformation:GetTemporaryGluePartitionCredentials"
          ]
          Resource = "*"
        },
        {
          Effect = "Allow"
          Action = ["glue:GetTable", "glue:GetPartitions", "glue:GetDatabase"]
          Resource = "*"
        }
      ]
    })
  }
}
```

**Step 3: Grant Lake Formation Permissions to EMR Role:**

```python
import boto3

lf = boto3.client('lakeformation')

# Grant EMR role access to specific tables/columns via Lake Formation
lf.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/EMRLakeFormationRole'
    },
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'analytics_db',
            'Name': 'customer_data',
            'ColumnNames': ['customer_id', 'name', 'email', 'purchase_count']
            # EMR CANNOT access: ssn, phone, address (column-level control!)
        }
    },
    Permissions=['SELECT']
)
```

**Step 4: Verify Enforcement**

```python
# In EMR Spark job:
spark.sql("SELECT ssn FROM glue_catalog.analytics_db.customer_data")
# Result: AccessDeniedException!
# Lake Formation blocks access to non-granted columns

spark.sql("SELECT customer_id, name, purchase_count FROM glue_catalog.analytics_db.customer_data")
# Result: Success! Only granted columns accessible
```

**For EMR on EKS (Modern Approach):**

```yaml
# Pod-level IAM role with Lake Formation
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: lf-governed-spark-job
spec:
  sparkConf:
    "spark.hadoop.fs.s3.authorization.enabled": "true"
    "spark.hadoop.fs.s3.authorization.mode": "lake-formation"
  driver:
    serviceAccount: "emr-lf-sa"  # IRSA with LF permissions
  executor:
    serviceAccount: "emr-lf-sa"
```

---

### Q8: Your MSK cluster processes 500K messages/second. You need exactly-once processing semantics for financial transactions. How do you implement this end-to-end?

**Answer:**

**The Exactly-Once Challenge:**

```
At-most-once:  Fire and forget (may lose messages)
At-least-once: Retry on failure (may process duplicates)
Exactly-once:  Process every message once and only once (hardest)

Kafka provides exactly-once within Kafka (transactions),
but end-to-end exactly-once requires careful design.
```

**End-to-End Architecture:**

```
┌──────────┐    ┌─────────────────┐    ┌──────────────────────┐
│ Producer │───▶│ MSK (Kafka)     │───▶│ Consumer + Sink       │
│ (idempotent)  │ (transactions)   │    │ (idempotent writes)  │
└──────────┘    └─────────────────┘    └──────────────────────┘

Exactly-once = Idempotent Producer + Kafka Transactions + Idempotent Consumer
```

**Layer 1: Idempotent Producer**

```python
from confluent_kafka import Producer

producer_config = {
    'bootstrap.servers': BROKERS,
    
    # CRITICAL: Enable idempotent producer
    'enable.idempotence': True,      # Prevents duplicate produces
    'acks': 'all',                   # All replicas must acknowledge
    'retries': 2147483647,           # Infinite retries (idempotent handles dedup)
    'max.in.flight.requests.per.connection': 5,  # Max 5 with idempotence
    
    # Transactional producer (for atomic multi-partition writes)
    'transactional.id': 'payment-processor-001',  # Unique per producer instance
}

producer = Producer(producer_config)
producer.init_transactions()  # Required for transactional producer

def produce_transaction(payment_event):
    """Produce with exactly-once guarantee."""
    try:
        producer.begin_transaction()
        
        # Write to multiple topics atomically
        producer.produce(
            topic='payment-events',
            key=payment_event['payment_id'],
            value=json.dumps(payment_event)
        )
        producer.produce(
            topic='payment-audit',
            key=payment_event['payment_id'],
            value=json.dumps({'action': 'processed', **payment_event})
        )
        
        producer.commit_transaction()
    except Exception as e:
        producer.abort_transaction()
        raise
```

**Layer 2: MSK Configuration for Exactly-Once**

```properties
# MSK Cluster Configuration
min.insync.replicas=2
unclean.leader.election.enable=false
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
transactional.id.expiration.ms=604800000

# Topic-level settings for payment-events
retention.ms=604800000
cleanup.policy=delete
min.insync.replicas=2
```

**Layer 3: Exactly-Once Consumer (Consume-Transform-Produce)**

```python
from confluent_kafka import Consumer, Producer, KafkaError

consumer_config = {
    'bootstrap.servers': BROKERS,
    'group.id': 'payment-processor',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,         # CRITICAL: Manual offset commit
    'isolation.level': 'read_committed', # Only read committed messages
}

producer_config = {
    'bootstrap.servers': BROKERS,
    'enable.idempotence': True,
    'transactional.id': 'payment-processor-consumer-001',
}

consumer = Consumer(consumer_config)
producer = Producer(producer_config)
producer.init_transactions()

consumer.subscribe(['payment-events'])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    if msg.error():
        continue
    
    payment = json.loads(msg.value())
    
    try:
        # Begin transaction (atomic: produce + offset commit)
        producer.begin_transaction()
        
        # Process the payment
        result = process_payment(payment)
        
        # Produce result to output topic
        producer.produce(
            topic='payment-results',
            key=payment['payment_id'],
            value=json.dumps(result)
        )
        
        # Commit consumer offset AS PART OF the transaction
        # If transaction fails → offset NOT committed → message reprocessed
        producer.send_offsets_to_transaction(
            consumer.position(consumer.assignment()),
            consumer.consumer_group_metadata()
        )
        
        producer.commit_transaction()
        
    except Exception as e:
        producer.abort_transaction()
        # Message will be reprocessed (offset not committed)
        logger.error(f"Transaction aborted: {e}")
```

**Layer 4: Idempotent Database Sink**

```python
# Even with Kafka exactly-once, the FINAL sink must be idempotent
# Because: Consumer crash AFTER DB write but BEFORE offset commit
# → Message reprocessed → Duplicate write to DB

def write_to_database_idempotent(payment_result):
    """Idempotent write using payment_id as deduplication key."""
    
    # Option A: INSERT ON CONFLICT (PostgreSQL/Aurora)
    cursor.execute("""
        INSERT INTO payment_results (payment_id, status, amount, processed_at)
        VALUES (%s, %s, %s, %s)
        ON CONFLICT (payment_id) DO NOTHING
    """, (
        payment_result['payment_id'],
        payment_result['status'],
        payment_result['amount'],
        datetime.utcnow()
    ))
    
    # Option B: DynamoDB conditional write
    dynamodb.put_item(
        TableName='payment_results',
        Item={
            'payment_id': {'S': payment_result['payment_id']},
            'status': {'S': payment_result['status']},
            'amount': {'N': str(payment_result['amount'])}
        },
        ConditionExpression='attribute_not_exists(payment_id)'
        # Fails silently if already exists (idempotent!)
    )
```

**Layer 5: Deduplication Table Pattern (Belt & Suspenders)**

```python
# For maximum safety: Outbox pattern + deduplication table

def process_with_outbox(payment):
    """Full exactly-once with outbox pattern."""
    
    with db.transaction():
        # Check if already processed
        existing = db.query(
            "SELECT 1 FROM processed_events WHERE event_id = %s",
            payment['event_id']
        )
        if existing:
            return  # Already processed — skip (idempotent)
        
        # Process the payment
        result = charge_customer(payment)
        
        # Write result AND mark as processed (SAME transaction!)
        db.execute(
            "INSERT INTO payment_results VALUES (...)", result
        )
        db.execute(
            "INSERT INTO processed_events (event_id, processed_at) VALUES (%s, NOW())",
            payment['event_id']
        )
        
        # Outbox: Write to outbox table (same transaction)
        db.execute(
            "INSERT INTO outbox (topic, key, value) VALUES (%s, %s, %s)",
            ('payment-results', payment['payment_id'], json.dumps(result))
        )
    
    # Separate process polls outbox → produces to Kafka → deletes from outbox
```

---

### Q9: You need to implement a data quality framework that prevents bad data from reaching your Gold layer tables. The framework should handle schema drift, null checks, referential integrity, and statistical anomalies. Design this on AWS.

**Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│              Data Quality Framework                               │
│                                                                   │
│  ┌───────────┐   ┌──────────────┐   ┌─────────────────────┐    │
│  │ Bronze    │──▶│ DQ Gateway   │──▶│ Silver (if passed)   │    │
│  │ (raw)     │   │ (Glue DQ +   │   │                     │    │
│  │           │   │  Custom)      │   │                     │    │
│  └───────────┘   └──────┬───────┘   └─────────────────────┘    │
│                          │                                       │
│                          │ FAILED                                │
│                          ▼                                       │
│                   ┌──────────────┐                               │
│                   │ Quarantine   │──▶ Alert + Ticket             │
│                   │ Zone         │                               │
│                   └──────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation - Multi-Layer DQ Engine:**

```python
# Glue Job with comprehensive data quality checks
import sys
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext
from pyspark.sql import functions as F
from pyspark.sql.types import *
from datetime import datetime, timedelta
import json

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

class DataQualityEngine:
    """Enterprise data quality validation framework."""
    
    def __init__(self, df, table_name, config):
        self.df = df
        self.table_name = table_name
        self.config = config
        self.results = []
        self.passed = True
        self.severity_threshold = config.get('severity_threshold', 'HIGH')
    
    def run_all_checks(self):
        """Execute all quality checks in sequence."""
        self.check_schema_conformance()
        self.check_completeness()
        self.check_uniqueness()
        self.check_validity()
        self.check_freshness()
        self.check_referential_integrity()
        self.check_statistical_anomalies()
        self.check_volume()
        return self.get_results()
    
    def check_schema_conformance(self):
        """Detect schema drift from expected structure."""
        expected_schema = self.config['expected_schema']
        actual_columns = set(self.df.columns)
        expected_columns = set(expected_schema.keys())
        
        # Missing columns (CRITICAL - breaks downstream)
        missing = expected_columns - actual_columns
        if missing:
            self._add_result('schema_conformance', 'CRITICAL', 
                           f"Missing columns: {missing}", failed=True)
        
        # Extra columns (WARNING - may indicate source change)
        extra = actual_columns - expected_columns
        if extra:
            self._add_result('schema_conformance', 'WARNING',
                           f"Unexpected columns: {extra}")
        
        # Type mismatches
        for col_name, expected_type in expected_schema.items():
            if col_name in actual_columns:
                actual_type = str(self.df.schema[col_name].dataType)
                if actual_type != expected_type:
                    self._add_result('schema_conformance', 'HIGH',
                                   f"Column {col_name}: expected {expected_type}, got {actual_type}")
    
    def check_completeness(self):
        """Check for null/empty values in required columns."""
        total_rows = self.df.count()
        
        for col_name, threshold in self.config.get('completeness_rules', {}).items():
            null_count = self.df.filter(
                F.col(col_name).isNull() | (F.col(col_name) == '')
            ).count()
            
            completeness_pct = (total_rows - null_count) / total_rows * 100
            
            if completeness_pct < threshold:
                self._add_result('completeness', 'HIGH',
                    f"Column {col_name}: {completeness_pct:.2f}% complete (threshold: {threshold}%)",
                    failed=True,
                    metrics={'column': col_name, 'completeness': completeness_pct}
                )
    
    def check_statistical_anomalies(self):
        """Detect statistical anomalies using historical baselines."""
        for rule in self.config.get('statistical_rules', []):
            col_name = rule['column']
            
            # Calculate current statistics
            stats = self.df.agg(
                F.avg(col_name).alias('mean'),
                F.stddev(col_name).alias('stddev'),
                F.min(col_name).alias('min_val'),
                F.max(col_name).alias('max_val'),
                F.percentile_approx(col_name, 0.99).alias('p99')
            ).collect()[0]
            
            # Compare with historical baseline (stored in DynamoDB/S3)
            baseline = self._get_baseline(col_name)
            
            if baseline:
                # Z-score test: Is current mean significantly different?
                z_score = abs(stats['mean'] - baseline['mean']) / baseline['stddev']
                if z_score > rule.get('z_threshold', 3):
                    self._add_result('statistical_anomaly', 'HIGH',
                        f"Column {col_name}: Mean shifted by {z_score:.1f} std deviations "
                        f"(current: {stats['mean']:.2f}, baseline: {baseline['mean']:.2f})",
                        failed=rule.get('block_on_anomaly', False)
                    )
                
                # Volume anomaly: Row count significantly different?
                current_count = self.df.count()
                expected_count = baseline['row_count']
                deviation_pct = abs(current_count - expected_count) / expected_count * 100
                
                if deviation_pct > rule.get('volume_threshold_pct', 50):
                    self._add_result('volume_anomaly', 'HIGH',
                        f"Row count deviation: {deviation_pct:.1f}% "
                        f"(current: {current_count}, expected: ~{expected_count})",
                        failed=True
                    )
            
            # Update baseline for next run
            self._update_baseline(col_name, stats, self.df.count())
    
    def check_referential_integrity(self):
        """Verify foreign key relationships across tables."""
        for rule in self.config.get('referential_rules', []):
            # Read the reference table
            ref_df = spark.read.parquet(rule['reference_path'])
            ref_keys = ref_df.select(rule['reference_column']).distinct()
            
            # Find orphan records
            orphans = self.df.join(
                ref_keys,
                self.df[rule['source_column']] == ref_keys[rule['reference_column']],
                'left_anti'
            )
            
            orphan_count = orphans.count()
            total_count = self.df.count()
            integrity_pct = (total_count - orphan_count) / total_count * 100
            
            if integrity_pct < rule['threshold']:
                self._add_result('referential_integrity', 'HIGH',
                    f"Column {rule['source_column']}: {orphan_count} orphan records "
                    f"({integrity_pct:.2f}% integrity, threshold: {rule['threshold']}%)",
                    failed=True
                )
    
    def get_results(self):
        """Return quality assessment."""
        overall_score = sum(1 for r in self.results if not r.get('failed', False)) / max(len(self.results), 1)
        
        return {
            'table': self.table_name,
            'timestamp': datetime.utcnow().isoformat(),
            'overall_score': overall_score,
            'passed': self.passed,
            'total_checks': len(self.results),
            'failed_checks': sum(1 for r in self.results if r.get('failed')),
            'results': self.results
        }
    
    def _add_result(self, check_type, severity, message, failed=False, metrics=None):
        self.results.append({
            'check_type': check_type,
            'severity': severity,
            'message': message,
            'failed': failed,
            'metrics': metrics or {}
        })
        if failed and severity in ['CRITICAL', 'HIGH']:
            self.passed = False


# Configuration (stored in S3 or DynamoDB):
dq_config = {
    'expected_schema': {
        'order_id': 'StringType()',
        'customer_id': 'StringType()',
        'amount': 'DecimalType(10,2)',
        'order_date': 'TimestampType()',
        'status': 'StringType()'
    },
    'completeness_rules': {
        'order_id': 100,      # 100% required
        'customer_id': 99.5,  # 99.5% required
        'amount': 100,
        'order_date': 100,
        'status': 99
    },
    'statistical_rules': [
        {
            'column': 'amount',
            'z_threshold': 3,
            'volume_threshold_pct': 30,
            'block_on_anomaly': True
        }
    ],
    'referential_rules': [
        {
            'source_column': 'customer_id',
            'reference_path': 's3://datalake/silver/customers/',
            'reference_column': 'customer_id',
            'threshold': 98
        }
    ]
}

# Execute
df = spark.read.parquet("s3://datalake/bronze/orders/date=2024-06-15/")
engine = DataQualityEngine(df, "orders", dq_config)
results = engine.run_all_checks()

if results['passed']:
    # Write to Silver
    df.write.mode("append").parquet("s3://datalake/silver/orders/")
else:
    # Quarantine
    df.write.mode("append").parquet("s3://datalake/quarantine/orders/")
    # Alert
    publish_quality_alert(results)
```

---

### Q10: Design a cost-optimized architecture for a data platform that processes 10TB/day ingestion from 50+ sources, requires both real-time (< 5 sec) and batch analytics, and must keep 5 years of historical data accessible. Budget: $50K/month.

**Answer:**

**Cost-Optimized Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│  INGESTION: 10TB/day from 50+ sources ($5K/month)               │
│                                                                   │
│  Real-time (30%):  MSK Serverless → Flink → Iceberg             │
│  Batch (50%):      S3 direct upload / DMS CDC → Glue            │
│  API/SaaS (20%):   AppFlow / EventBridge → Firehose → S3        │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  STORAGE: 5 years × 10TB/day = ~18PB ($15K/month)              │
│                                                                   │
│  Hot (0-30 days):     S3 Standard         = 300TB × $0.023 = $7K│
│  Warm (30-365 days):  S3 Standard-IA      = 3PB × $0.0125 = $4K│
│  Cold (1-5 years):    S3 Glacier IR       = 15PB × $0.004 = $5K│
│                                                                   │
│  Format: Apache Iceberg (ZSTD compression = 5:1 ratio)          │
│  Actual stored: ~3.6PB (after compression)                       │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  PROCESSING ($15K/month)                                         │
│                                                                   │
│  Real-time:  Managed Flink (2 KPU × 24h)          = $2K         │
│  Daily ETL:  Glue (Flex execution, spot)           = $5K         │
│  Heavy jobs: EMR (Spot instances, 70% savings)     = $5K         │
│  Orchestration: MWAA (mw1.small)                   = $1K         │
│  Compaction: Glue (weekly Iceberg maintenance)     = $2K         │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  QUERY/ANALYTICS ($12K/month)                                    │
│                                                                   │
│  Real-time dashboards: Redshift Serverless (128 RPU) = $6K      │
│  Ad-hoc SQL:           Athena (scan-based pricing)   = $3K      │
│  ML/Data Science:      SageMaker (spot notebooks)    = $2K      │
│  API access:           Lambda + API Gateway          = $1K      │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│  GOVERNANCE & MONITORING ($3K/month)                             │
│                                                                   │
│  Lake Formation:    Free (included)                              │
│  Glue Catalog:     $1/100K objects                    = $0.5K   │
│  CloudWatch:       Metrics + Logs                     = $1K     │
│  Data Quality:     Glue DQ (included in job cost)               │
│  VPC Endpoints:    S3 Gateway (free) + Interface      = $1.5K   │
└─────────────────────────────────────────────────────────────────┘

TOTAL ESTIMATED: ~$50K/month ✓
```

**Key Cost Optimization Techniques:**

```yaml
cost_optimizations:
  storage:
    - Iceberg with ZSTD compression (5:1 ratio): saves 80% storage
    - S3 Intelligent-Tiering for unpredictable access patterns
    - Iceberg expire_snapshots + remove_orphan_files (cleanup)
    - S3 Lifecycle policies (auto-transition to cheaper tiers)
    
  compute:
    - Glue Flex execution: 34% cheaper (for non-time-sensitive ETL)
    - EMR Spot instances: 60-70% cheaper for task nodes
    - Redshift Serverless: Pay only when queries run (vs 24/7 provisioned)
    - Athena: $5/TB scanned → use Iceberg partition pruning to minimize scan
    - MSK Serverless: Pay per partition-hour (no idle broker costs)
    
  data_transfer:
    - S3 Gateway Endpoint: FREE data transfer (vs $0.09/GB through NAT)
    - Same-region processing: All services in us-east-1
    - Compress data in transit (ZSTD/Snappy)
    
  right_sizing:
    - MWAA mw1.small (not medium): $0.49/hr vs $0.98/hr
    - Redshift Serverless with usage limits (128 RPU cap)
    - Athena workgroups with byte-scan limits per query
```

**MSK Serverless (Cost Saver for Variable Workloads):**

```hcl
# MSK Serverless: No brokers to manage, pay per use
resource "aws_msk_serverless_cluster" "streaming" {
  cluster_name = "data-platform-streaming"
  
  client_authentication {
    sasl {
      iam { enabled = true }
    }
  }
  
  vpc_config {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.msk.id]
  }
}
# Cost: $0.10/GB ingested + $0.05/GB replicated
# vs Provisioned: $0.21/hr/broker × 6 brokers = $900/month minimum
```

**Glue Flex (34% Savings):**

```python
# Use Flex execution for non-urgent ETL jobs
# Glue Flex: Jobs may wait up to 5 min for resources (but 34% cheaper)

glue.create_job(
    Name='daily-etl-bronze-to-silver',
    ExecutionClass='FLEX',  # 34% cheaper than STANDARD
    Command={
        'Name': 'glueetl',
        'ScriptLocation': 's3://scripts/daily_etl.py'
    },
    WorkerType='G.1X',
    NumberOfWorkers=20,
    GlueVersion='4.0'
)
```
