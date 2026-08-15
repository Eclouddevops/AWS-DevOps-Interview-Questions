# Data Platform Architecture - Tricky Production-Based Interview Questions & Answers

## AWS Glue, EMR, Redshift, MSK, MWAA, Lake Formation

### Q1: Your Glue ETL job processes 500GB daily but has been failing with OOM errors after data volume grew 3x. The job uses DynamicFrames. How do you diagnose and fix this in production without rewriting the entire pipeline?

**Answer:**

**Diagnosis Steps:**

1. **Check CloudWatch Metrics:**
   ```
   - Glue Job Run: Memory utilization per executor
   - Look for "executor lost" or "FetchFailed" in logs
   - Check shuffle spill metrics
   ```

2. **Enable Glue Job Bookmarks Monitoring:**
   ```python
   # Check if bookmarks are causing full reprocessing
   glueContext.read_bookmark()
   ```

3. **Root Cause Analysis:**
   - DynamicFrames load entire dataset into memory for schema inference
   - Data skew: One partition has 80% of the data
   - No pushdown predicates: Reading entire table when only subset needed

**Production Fix (Incremental):**

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'date_partition'])

# Fix 1: Use worker type G.2X for memory-intensive jobs
# Job parameters: --worker-type G.2X --number-of-workers 20

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Fix 2: Push down predicates to reduce data read
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="events",
    push_down_predicate=f"year=2024 and month=06 and day={args['date_partition']}",
    additional_options={
        "useS3ListImplementation": True,  # Fix 3: Faster S3 listing
        "recurse": True
    }
)

# Fix 4: Repartition to handle data skew
df = datasource.toDF()
df = df.repartition(200, "customer_id")  # Even distribution

# Fix 5: Enable adaptive query execution
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")

# Fix 6: Use Parquet with snappy for intermediate writes
df.write.mode("overwrite") \
    .option("compression", "snappy") \
    .partitionBy("year", "month", "day") \
    .parquet("s3://datalake-bucket/silver/events/")

job.commit()
```

**Additional Optimizations:**
- Enable Auto Scaling: `--enable-auto-scaling true`
- Use Glue 4.0 (Spark 3.3.0 with optimized memory management)
- Enable Spark UI for visual debugging: `--enable-spark-ui true --spark-event-logs-path s3://logs/`

---

### Q2: You're designing a real-time data pipeline with MSK (Kafka) that processes 1 million events/second. The consumer lag is growing unbounded. How do you troubleshoot and scale?

**Answer:**

**Troubleshooting Consumer Lag:**

1. **Identify the bottleneck:**
   ```bash
   # Check consumer group lag
   kafka-consumer-groups.sh --bootstrap-server $BROKER \
     --describe --group my-consumer-group
   
   # Output shows: TOPIC, PARTITION, CURRENT-OFFSET, LOG-END-OFFSET, LAG
   ```

2. **Check producer throughput:**
   ```bash
   # Verify actual ingestion rate
   kafka-run-class.sh kafka.tools.GetOffsetShell \
     --broker-list $BROKER --topic events --time -1
   ```

3. **Root Cause Categories:**

   **a) Under-provisioned partitions:**
   - Rule of thumb: partitions ≥ consumer instances × 3
   - MSK max throughput per partition: ~1 MB/s (consumer), ~1 MB/s (producer)
   
   **b) Consumer processing too slow:**
   - Database writes blocking the consumer loop
   - Synchronous external API calls
   - Large message deserialization overhead
   
   **c) Broker bottleneck:**
   - Disk I/O saturation (check `BrokerStorageUtilization`)
   - Network throughput limit (`NetworkRxThroughput`)

**Scaling Solution:**

```python
# Architecture: MSK → Flink/Lambda Fan-out → Destinations

# Option 1: Increase partitions (cannot decrease later!)
# aws kafka update-broker-count (for broker scaling)
# Topic partition increase:
kafka-topics.sh --bootstrap-server $BROKER \
  --alter --topic events --partitions 128

# Option 2: MSK Serverless (auto-scales partitions)
# Migrate to MSK Serverless for elastic scaling

# Option 3: Consumer optimization with batch processing
from confluent_kafka import Consumer, KafkaError
import concurrent.futures

consumer_config = {
    'bootstrap.servers': BROKERS,
    'group.id': 'my-consumer-group',
    'auto.offset.reset': 'earliest',
    'enable.auto.commit': False,
    'max.poll.interval.ms': 300000,
    'fetch.min.bytes': 50000,          # Batch fetching
    'fetch.max.wait.ms': 500,          # Max wait for batch
    'max.partition.fetch.bytes': 1048576,  # 1MB per partition
    'session.timeout.ms': 45000
}

consumer = Consumer(consumer_config)
consumer.subscribe(['events'])

# Micro-batch processing with thread pool
with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
    while True:
        messages = consumer.consume(num_messages=500, timeout=1.0)
        if messages:
            # Process in parallel batches
            futures = []
            for batch in chunk(messages, 50):
                futures.append(executor.submit(process_batch, batch))
            
            concurrent.futures.wait(futures)
            consumer.commit()
```

**MSK Cluster Sizing for 1M events/sec:**
```
Assumptions:
- Average message size: 1KB
- Throughput: 1M × 1KB = 1 GB/s ingestion
- Replication factor: 3
- Retention: 7 days

Broker Configuration:
- Instance type: kafka.m5.8xlarge (16 vCPU, 128GB RAM)
- Number of brokers: 12 (across 3 AZs)
- Storage: 10TB per broker (EBS gp3, 16000 IOPS)
- Partitions: 128 per topic

Network: 1GB/s × 3 (replication) = 3 GB/s cluster throughput
Storage: 1GB/s × 86400s × 7 days = ~600TB total (distributed)
```

---

### Q3: Your data lake has 50TB in S3 with thousands of small files (< 1MB each). Athena queries are extremely slow and expensive. How do you fix this without disrupting ongoing pipelines?

**Answer:**

**Problem:** Small files problem - each file requires a separate S3 GET request, and Athena creates a separate split per file. With millions of small files:
- S3 request costs are high (GET at $0.0004/1000 requests)
- Athena scans inefficiently (column pruning less effective)
- Glue Crawlers timeout trying to catalog millions of objects

**Solution Architecture:**

```
Current State:              Target State:
s3://lake/raw/              s3://lake/optimized/
  ├── file1.json (500B)      ├── year=2024/
  ├── file2.json (800B)      │   ├── month=06/
  ├── file3.json (200B)      │   │   ├── part-0001.parquet (128MB)
  └── ... (millions)         │   │   ├── part-0002.parquet (128MB)
                              │   │   └── part-0003.parquet (128MB)
```

**Implementation - Compaction Pipeline:**

```python
# Glue Job: Small File Compaction (runs hourly)
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from pyspark.context import SparkContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_path', 'target_path', 'target_size_mb'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

TARGET_SIZE_BYTES = int(args['target_size_mb']) * 1024 * 1024  # 128MB default

# Read all small files
df = spark.read.json(args['source_path'])

# Calculate optimal partition count
total_size = df.rdd.map(lambda row: len(str(row))).reduce(lambda a, b: a + b)
num_partitions = max(1, int(total_size / TARGET_SIZE_BYTES))

# Repartition and write as Parquet with optimal file sizes
df.repartition(num_partitions) \
  .write \
  .mode("overwrite") \
  .option("compression", "snappy") \
  .option("maxRecordsPerFile", 1000000) \
  .partitionBy("year", "month", "day") \
  .parquet(args['target_path'])

job.commit()
```

**Better Solution - Apache Iceberg with Compaction:**

```python
# Use Iceberg table format for automatic compaction
spark.sql("""
    CREATE TABLE glue_catalog.analytics.events (
        event_id STRING,
        user_id STRING,
        event_type STRING,
        payload STRING,
        event_time TIMESTAMP
    )
    USING iceberg
    PARTITIONED BY (days(event_time))
    TBLPROPERTIES (
        'write.target-file-size-bytes' = '134217728',
        'write.distribution-mode' = 'hash',
        'write.metadata.compression-codec' = 'gzip'
    )
""")

# Iceberg compaction (merge small files)
spark.sql("""
    CALL glue_catalog.system.rewrite_data_files(
        table => 'analytics.events',
        strategy => 'binpack',
        options => map(
            'target-file-size-bytes', '134217728',
            'min-file-size-bytes', '104857600',
            'max-file-size-bytes', '180355072'
        )
    )
""")

# Expire old snapshots (cleanup)
spark.sql("""
    CALL glue_catalog.system.expire_snapshots(
        table => 'analytics.events',
        older_than => TIMESTAMP '2024-06-01 00:00:00',
        retain_last => 5
    )
""")
```

**Non-Disruptive Migration Strategy:**
1. Keep existing pipeline writing to raw path
2. Run compaction job on a schedule (hourly/daily)
3. Create new Athena table pointing to compacted path
4. Update downstream consumers to use new table
5. Once validated, update pipeline to write directly in optimized format
6. Decommission raw path after retention period

---

### Q4: You need to implement cross-account data sharing using Lake Formation where Team A (Account A) produces data and Teams B, C, D (different accounts) need different column-level access. How do you architect this?

**Answer:**

**Architecture:**

```
┌─────────────────────────────────┐
│  Producer Account (Team A)      │
│                                 │
│  ┌───────────────────────────┐  │
│  │  Lake Formation           │  │
│  │  - Data Catalog (source)  │  │
│  │  - S3 Data Location       │  │
│  │  - Tag-Based Access Ctrl  │  │
│  └───────────────────────────┘  │
│                                 │
│  ┌───────────────────────────┐  │
│  │  S3: s3://producer-lake/  │  │
│  │  ├── customers/           │  │
│  │  ├── transactions/        │  │
│  │  └── analytics/           │  │
│  └───────────────────────────┘  │
└────────────────┬────────────────┘
                 │ LF Grant (cross-account)
    ┌────────────┼────────────────────┐
    ▼            ▼                    ▼
┌─────────┐  ┌─────────┐    ┌─────────────┐
│ Acct B  │  │ Acct C  │    │ Acct D      │
│ (Mktg)  │  │ (Risk)  │    │ (Analytics) │
│         │  │         │    │             │
│ Columns:│  │ Columns:│    │ All columns │
│ name,   │  │ name,   │    │ (masked PII)│
│ email,  │  │ txn_amt,│    │             │
│ segment │  │ risk_sc │    │             │
└─────────┘  └─────────┘    └─────────────┘
```

**Implementation:**

**Step 1: Enable Lake Formation Cross-Account in Organization:**
```hcl
# In Producer Account
resource "aws_lakeformation_data_lake_settings" "main" {
  admins = [
    "arn:aws:iam::PRODUCER_ACCT:role/LakeFormationAdmin"
  ]
  
  create_database_default_permissions {
    permissions = ["ALL"]
    principal   = "IAM_ALLOWED_PRINCIPALS"
  }
}

# Register S3 location
resource "aws_lakeformation_resource" "data_lake" {
  arn      = "arn:aws:s3:::producer-lake"
  role_arn = aws_iam_role.lf_service_role.arn
}
```

**Step 2: Define LF-Tags for Column-Level Access:**
```hcl
# Create LF-Tags
resource "aws_lakeformation_lf_tag" "sensitivity" {
  key    = "Sensitivity"
  values = ["Public", "Internal", "Confidential", "Restricted"]
}

resource "aws_lakeformation_lf_tag" "domain" {
  key    = "Domain"
  values = ["Marketing", "Risk", "Analytics", "Finance"]
}

# Assign tags to columns
resource "aws_lakeformation_resource_lf_tags" "customers_table" {
  database {
    name = "customers_db"
  }
  table {
    database_name = "customers_db"
    name          = "customers"
  }
  
  lf_tag {
    key   = "Sensitivity"
    value = "Confidential"
  }
}
```

**Step 3: Grant Cross-Account Access:**
```python
import boto3

lf_client = boto3.client('lakeformation')

# Grant Team B (Marketing) - only specific columns
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::ACCOUNT_B:root'
    },
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'customers_db',
            'Name': 'customers',
            'ColumnNames': ['customer_id', 'name', 'email', 'segment', 'region']
            # Excludes: ssn, phone, address, salary
        }
    },
    Permissions=['SELECT'],
    PermissionsWithGrantOption=[]
)

# Grant Team C (Risk) - different columns
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::ACCOUNT_C:root'
    },
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'customers_db',
            'Name': 'transactions',
            'ColumnNames': ['txn_id', 'customer_id', 'amount', 'risk_score', 'timestamp']
        }
    },
    Permissions=['SELECT'],
    PermissionsWithGrantOption=[]
)

# Grant Team D (Analytics) - all columns with cell-level filter
lf_client.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::ACCOUNT_D:root'
    },
    Resource={
        'DataCellsFilter': {
            'DatabaseName': 'customers_db',
            'TableName': 'customers',
            'Name': 'mask_pii_filter'
        }
    },
    Permissions=['SELECT'],
    PermissionsWithGrantOption=[]
)

# Create data cell filter to mask PII
lf_client.create_data_cells_filter(
    TableData={
        'DatabaseName': 'customers_db',
        'TableName': 'customers',
        'Name': 'mask_pii_filter',
        'RowFilter': {
            'AllRowsWildcard': {}  # All rows
        },
        'ColumnNames': ['customer_id', 'name', 'segment', 'region', 'signup_date'],
        'ColumnWildcard': {
            'ExcludedColumnNames': ['ssn', 'phone', 'address']  # Exclude PII
        }
    }
)
```

**Step 4: Consumer Account Setup:**
```hcl
# In Consumer Account (Account B)
# Create resource link to shared database
resource "aws_glue_catalog_database" "shared_customers" {
  name = "shared_customers_db"
  
  target_database {
    catalog_id    = var.producer_account_id
    database_name = "customers_db"
  }
}

# Now Team B can query:
# SELECT name, email, segment FROM shared_customers_db.customers
# (Only sees columns they were granted)
```

---

### Q5: Your MWAA (Airflow) DAGs are failing intermittently with "Task killed by OOM" and "Zombie task detected". The environment has 30 DAGs with complex dependencies. How do you stabilize this?

**Answer:**

**Root Cause Analysis:**

1. **OOM Issues:**
   - Workers running too many tasks concurrently
   - Individual tasks loading large DataFrames in memory
   - DAG parsing consuming excessive memory

2. **Zombie Tasks:**
   - Workers dying during task execution
   - Heartbeat timeout exceeded
   - Environment auto-scaling causing worker replacement mid-task

**Production Fixes:**

```python
# airflow.cfg overrides via MWAA Environment
mwaa_configuration_options = {
    # Fix 1: Reduce concurrency to prevent OOM
    "core.parallelism": "32",                    # Max tasks across all DAGs
    "core.max_active_tasks_per_dag": "16",       # Max tasks per DAG
    "core.max_active_runs_per_dag": "3",         # Limit concurrent DAG runs
    
    # Fix 2: Increase heartbeat for long-running tasks
    "scheduler.scheduler_heartbeat_sec": "5",
    "scheduler.zombie_task_threshold": "600",    # 10 min before marking zombie
    
    # Fix 3: Celery worker tuning
    "celery.worker_concurrency": "4",            # Tasks per worker (reduce from default)
    "celery.worker_autoscale": "4,2",            # max,min concurrency
    
    # Fix 4: Task timeout and retry
    "core.killed_task_cleanup_time": "120",
    "core.dagbag_import_timeout": "60",          # Fail fast on bad DAG parsing
    
    # Fix 5: Reduce scheduler overhead
    "scheduler.min_file_process_interval": "60", # Don't re-parse DAGs too often
    "scheduler.dag_dir_list_interval": "120"
}
```

**DAG Optimization Patterns:**

```python
from airflow import DAG
from airflow.decorators import task
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.providers.amazon.aws.sensors.glue import GlueJobSensor
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-platform',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(minutes=30),
    'execution_timeout': timedelta(hours=2),  # Kill if stuck
    'on_failure_callback': alert_on_failure,
    'sla': timedelta(hours=4)  # SLA monitoring
}

dag = DAG(
    'data_pipeline_v2',
    default_args=default_args,
    schedule_interval='@hourly',
    catchup=False,
    max_active_runs=2,
    tags=['production', 'data-lake']
)

# Pattern: Offload heavy processing to Glue/EMR (not Airflow workers)
run_glue_job = GlueJobOperator(
    task_id='transform_data',
    job_name='heavy_etl_job',
    script_location='s3://scripts/transform.py',
    num_of_dpus=10,
    wait_for_completion=False,  # Don't block worker
    dag=dag
)

# Use sensor with poke_interval to check completion
wait_for_glue = GlueJobSensor(
    task_id='wait_for_transform',
    job_name='heavy_etl_job',
    run_id="{{ task_instance.xcom_pull(task_ids='transform_data') }}",
    poke_interval=60,
    timeout=7200,
    mode='reschedule',  # Free up worker slot while waiting
    dag=dag
)

# Pattern: Use @task decorator for lightweight tasks only
@task(pool='lightweight_pool', pool_slots=1)
def validate_output():
    """Lightweight validation - OK for Airflow worker."""
    import boto3
    s3 = boto3.client('s3')
    response = s3.list_objects_v2(
        Bucket='datalake', Prefix='silver/output/', MaxKeys=1
    )
    if response['KeyCount'] == 0:
        raise ValueError("No output files generated!")
    return True

run_glue_job >> wait_for_glue >> validate_output()
```

**MWAA Environment Sizing:**
```hcl
resource "aws_mwaa_environment" "production" {
  name               = "data-platform-airflow"
  airflow_version    = "2.8.1"
  environment_class  = "mw1.large"  # Upgrade from medium
  
  min_workers = 2
  max_workers = 10  # Auto-scale workers
  
  # Scheduler - dedicated, not sharing resources
  schedulers = 2  # HA schedulers
  
  network_configuration {
    security_group_ids = [aws_security_group.mwaa.id]
    subnet_ids         = var.private_subnet_ids
  }
  
  logging_configuration {
    dag_processing_logs {
      enabled   = true
      log_level = "WARNING"
    }
    scheduler_logs {
      enabled   = true
      log_level = "INFO"
    }
    task_logs {
      enabled   = true
      log_level = "INFO"
    }
    worker_logs {
      enabled   = true
      log_level = "WARNING"
    }
  }
}
```

---

### Q6: You're running a Redshift cluster (ra3.4xlarge, 6 nodes) that serves both BI dashboards and ad-hoc analyst queries. During business hours, BI dashboards timeout because analysts run expensive full-table scans. How do you solve this without separate clusters?

**Answer:**

**Solution: Workload Management (WLM) + Concurrency Scaling + Data Sharing**

**Step 1: Configure WLM Queues:**
```sql
-- Create WLM configuration with queue priorities
-- Dashboard queries (short, frequent, high priority)
-- Analyst queries (long, fewer, lower priority)

-- Via Parameter Group (Terraform):
```

```hcl
resource "aws_redshift_parameter_group" "production" {
  name   = "production-wlm"
  family = "redshift-1.0"
  
  parameter {
    name = "wlm_json_configuration"
    value = jsonencode([
      {
        "name": "dashboard_queue",
        "query_group": ["dashboard", "bi_tool"],
        "user_group": ["bi_service_account"],
        "memory_percent_to_use": 50,
        "max_concurrency_scaling_clusters": 5,
        "concurrency_scaling": "auto",
        "priority": "highest",
        "auto_wlm": false,
        "slots": 15,
        "max_execution_time": 60000  # 60 sec timeout
      },
      {
        "name": "analyst_queue",
        "query_group": ["analyst"],
        "user_group": ["analysts"],
        "memory_percent_to_use": 30,
        "concurrency_scaling": "off",
        "priority": "normal",
        "auto_wlm": false,
        "slots": 5,
        "max_execution_time": 1800000  # 30 min timeout
      },
      {
        "name": "etl_queue",
        "query_group": ["etl"],
        "user_group": ["etl_service"],
        "memory_percent_to_use": 20,
        "priority": "low",
        "auto_wlm": false,
        "slots": 3,
        "max_execution_time": 7200000  # 2 hour timeout
      }
    ])
  }
}
```

**Step 2: Query Monitoring Rules (QMR):**
```sql
-- Abort queries that scan too much data
CREATE QUERY MONITORING RULE analyst_scan_limit
  FOR analyst_queue
  WHEN scan_row_count > 1000000000  -- 1B rows
  THEN ABORT;

-- Log expensive queries for review
CREATE QUERY MONITORING RULE expensive_query_log
  FOR ALL
  WHEN query_cpu_time > 300000  -- 5 minutes CPU
  THEN LOG;

-- Hop short queries from analyst queue if they match dashboard pattern
CREATE QUERY MONITORING RULE hop_quick_queries
  FOR analyst_queue
  WHEN query_execution_time < 5000  -- Less than 5 sec
  THEN HOP TO dashboard_queue;
```

**Step 3: Concurrency Scaling for Dashboards:**
```sql
-- Enable concurrency scaling for burst capacity
-- Dashboard queue auto-scales to handle peak BI load
-- Charges only for actual burst usage ($0.25/credit/hour)

-- Monitor concurrency scaling usage:
SELECT * FROM svl_concurrency_scaling_usage
WHERE start_time > DATEADD(day, -7, GETDATE())
ORDER BY start_time DESC;
```

**Step 4: Redshift Data Sharing (for true isolation):**
```sql
-- Producer cluster (main production)
CREATE DATASHARE analytics_share;

ALTER DATASHARE analytics_share 
  ADD SCHEMA public;
ALTER DATASHARE analytics_share 
  ADD TABLE public.fact_orders;
ALTER DATASHARE analytics_share 
  ADD TABLE public.dim_customers;

-- Grant to consumer (analyst Redshift Serverless)
GRANT USAGE ON DATASHARE analytics_share 
  TO NAMESPACE 'analyst-serverless-namespace-id';

-- In consumer (Redshift Serverless for analysts):
CREATE DATABASE analyst_data FROM DATASHARE analytics_share
  OF NAMESPACE 'producer-namespace-id';

-- Analysts query against serverless (no impact on production)
-- BI dashboards hit the provisioned cluster directly
```

**Architecture:**
```
┌──────────────────────────────────┐
│  Redshift Provisioned Cluster    │
│  (ra3.4xlarge × 6)              │
│                                  │
│  WLM Queue: Dashboard (priority) │◄── QuickSight / Tableau
│  WLM Queue: ETL (low priority)  │◄── Glue / Airflow
│  Concurrency Scaling: ON        │
│                                  │
│  Data Share ──────────────────┐  │
└──────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────┐
│  Redshift Serverless             │
│  (for analysts - auto scales)   │
│                                  │
│  Reads shared data (no copy)    │◄── Analyst SQL Tools
│  Independent compute            │
│  No impact on production        │
└──────────────────────────────────┘
```

---

### Q7: Your EMR cluster running Spark jobs on a 10TB dataset takes 4 hours to complete. The business needs it under 1 hour. How do you optimize without just adding more instances?

**Answer:**

**Optimization Strategy (4h → <1h):**

**Step 1: Analyze the bottleneck:**
```bash
# Enable Spark History Server
# Check: Stage timeline, task skew, shuffle read/write, GC time

# Key metrics to examine:
# - Shuffle Read/Write (data movement between stages)
# - Task duration distribution (skew?)
# - GC Time percentage (>10% = memory issue)
# - Spill to disk (memory insufficient)
```

**Step 2: Data Format Optimization (Save 50-70% time):**
```python
# BEFORE: Reading CSV/JSON (no predicate pushdown, no column pruning)
df = spark.read.csv("s3://lake/raw/sales/")

# AFTER: Convert to Parquet with Z-ordering on query columns
df.write \
    .mode("overwrite") \
    .option("compression", "zstd") \
    .sortBy("customer_id", "date") \
    .bucketBy(256, "customer_id") \
    .parquet("s3://lake/optimized/sales/")
```

**Step 3: Spark Configuration Tuning:**
```python
spark_config = {
    # Memory management
    "spark.executor.memory": "28g",         # 80% of instance memory
    "spark.executor.memoryOverhead": "4g",   # Off-heap for shuffles
    "spark.memory.fraction": "0.8",          # More memory for execution
    "spark.memory.storageFraction": "0.3",   # Less for caching
    
    # Shuffle optimization (biggest bottleneck)
    "spark.sql.shuffle.partitions": "2000",  # Match data size
    "spark.shuffle.compress": "true",
    "spark.shuffle.spill.compress": "true",
    "spark.reducer.maxSizeInFlight": "96m",
    
    # Adaptive Query Execution (AQE)
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.adaptive.coalescePartitions.enabled": "true",
    "spark.sql.adaptive.skewJoin.enabled": "true",
    "spark.sql.adaptive.skewJoin.skewedPartitionFactor": "5",
    
    # S3 optimizations
    "spark.hadoop.fs.s3a.connection.maximum": "200",
    "spark.hadoop.fs.s3a.threads.max": "64",
    "spark.hadoop.fs.s3a.fast.upload": "true",
    "spark.hadoop.fs.s3a.fast.upload.buffer": "bytebuffer",
    "spark.sql.files.maxPartitionBytes": "256m",  # Larger input splits
    
    # Catalyst optimizer
    "spark.sql.autoBroadcastJoinThreshold": "50m",  # Broadcast small tables
    "spark.sql.broadcastTimeout": "600",
    
    # Serialization
    "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
    "spark.kryoserializer.buffer.max": "1024m"
}
```

**Step 4: Code-Level Optimizations:**
```python
from pyspark.sql import functions as F

# BEFORE (causes full shuffle):
result = large_df.join(small_df, "customer_id")

# AFTER (broadcast small table):
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "customer_id")

# BEFORE (collect + loop = disaster):
for row in df.collect():
    process(row)

# AFTER (use transformations):
result = df.withColumn("processed", process_udf(F.col("data")))

# BEFORE (repartition causes full shuffle):
df.repartition(100).write.parquet("s3://output")

# AFTER (coalesce for reducing partitions):
df.coalesce(100).write.parquet("s3://output")

# Partition pruning - filter BEFORE join
df_filtered = large_df.filter(F.col("date") >= "2024-01-01")
result = df_filtered.join(broadcast(dim_table), "product_id")
```

**Step 5: EMR Instance Strategy:**
```hcl
resource "aws_emr_cluster" "optimized" {
  name          = "optimized-spark-cluster"
  release_label = "emr-7.0.0"  # Latest for Spark 3.5
  
  # Use Graviton for 20-30% better price/performance
  master_instance_group {
    instance_type  = "m6g.2xlarge"
    instance_count = 1
  }
  
  core_instance_group {
    instance_type  = "r6g.4xlarge"  # Memory-optimized for shuffles
    instance_count = 10
    ebs_config {
      size = 500
      type = "gp3"
      iops = 6000
      throughput = 250
      volumes_per_instance = 2  # More shuffle space
    }
  }
  
  # Spot instances for task nodes (fault-tolerant)
  # Saves 60-70% on compute cost
  configurations_json = jsonencode([{
    "Classification": "spark-defaults",
    "Properties": spark_config
  }])
}

# Managed Scaling - auto-scale based on YARN metrics
resource "aws_emr_managed_scaling_policy" "auto_scale" {
  cluster_id = aws_emr_cluster.optimized.id
  
  compute_limits {
    unit_type                       = "InstanceFleetUnits"
    minimum_capacity_units          = 10
    maximum_capacity_units          = 50
    maximum_on_demand_capacity_units = 10
    maximum_core_capacity_units     = 10
  }
}
```

---

### Q8: Design a real-time fraud detection pipeline that processes credit card transactions with sub-second latency. What AWS services and architecture would you use?

**Answer:**

**Architecture:**

```
┌──────────┐    ┌─────────┐    ┌──────────────────┐    ┌──────────────┐
│ Payment  │───▶│  MSK    │───▶│ Flink on KDA     │───▶│ Action Layer │
│ Gateway  │    │ (Kafka) │    │ (Real-time ML)   │    │              │
└──────────┘    └─────────┘    └──────────────────┘    └──────────────┘
                     │                   │                      │
                     │                   ▼                      ▼
                     │         ┌──────────────────┐   ┌──────────────┐
                     │         │ SageMaker        │   │ EventBridge  │
                     │         │ Real-time        │   │ → SNS/SQS    │
                     │         │ Inference        │   │ → Step Func  │
                     │         └──────────────────┘   └──────────────┘
                     │
                     ▼
              ┌─────────────┐    ┌──────────────┐
              │ S3 (raw)    │───▶│ Glue/EMR     │──▶ Redshift
              │ Firehose    │    │ (batch retrain)   (analytics)
              └─────────────┘    └──────────────┘
```

**Implementation:**

```python
# Flink Application (Kinesis Data Analytics / Amazon Managed Flink)
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment, EnvironmentSettings

env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(16)
t_env = StreamTableEnvironment.create(env)

# Source: MSK topic
t_env.execute_sql("""
    CREATE TABLE transactions (
        txn_id STRING,
        card_number STRING,
        merchant_id STRING,
        amount DECIMAL(10,2),
        currency STRING,
        location_lat DOUBLE,
        location_lon DOUBLE,
        txn_time TIMESTAMP(3),
        WATERMARK FOR txn_time AS txn_time - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'transactions',
        'properties.bootstrap.servers' = 'b-1.msk-cluster:9092',
        'properties.group.id' = 'fraud-detection',
        'format' = 'json',
        'scan.startup.mode' = 'latest-offset'
    )
""")

# Feature engineering with windowed aggregations
t_env.execute_sql("""
    CREATE VIEW enriched_transactions AS
    SELECT 
        t.*,
        -- Velocity features (sliding window)
        COUNT(*) OVER w AS txn_count_1h,
        SUM(amount) OVER w AS total_amount_1h,
        COUNT(DISTINCT merchant_id) OVER w AS unique_merchants_1h,
        
        -- Distance from last transaction
        LAST_VALUE(location_lat) OVER w AS prev_lat,
        LAST_VALUE(location_lon) OVER w AS prev_lon
    FROM transactions t
    WINDOW w AS (
        PARTITION BY card_number 
        ORDER BY txn_time 
        RANGE BETWEEN INTERVAL '1' HOUR PRECEDING AND CURRENT ROW
    )
""")

# Real-time scoring with SageMaker endpoint
t_env.execute_sql("""
    CREATE TABLE fraud_scores (
        txn_id STRING,
        card_number STRING,
        fraud_score DOUBLE,
        is_fraud BOOLEAN,
        processing_time TIMESTAMP(3)
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'fraud-decisions',
        'properties.bootstrap.servers' = 'b-1.msk-cluster:9092',
        'format' = 'json'
    )
""")

# UDF for SageMaker inference
t_env.execute_sql("""
    INSERT INTO fraud_scores
    SELECT
        txn_id,
        card_number,
        invoke_sagemaker_endpoint(
            txn_count_1h, total_amount_1h, unique_merchants_1h,
            amount, calculate_distance(location_lat, location_lon, prev_lat, prev_lon)
        ) AS fraud_score,
        CASE WHEN fraud_score > 0.85 THEN TRUE ELSE FALSE END AS is_fraud,
        CURRENT_TIMESTAMP AS processing_time
    FROM enriched_transactions
""")
```

**SageMaker Real-Time Endpoint Configuration:**
```hcl
resource "aws_sagemaker_endpoint_configuration" "fraud_model" {
  name = "fraud-detection-endpoint"
  
  production_variants {
    variant_name           = "primary"
    model_name             = aws_sagemaker_model.fraud_v2.name
    initial_instance_count = 3
    instance_type          = "ml.c5.2xlarge"
    
    # Shadow testing new model version
    container_startup_health_check_timeout_in_seconds = 120
  }
  
  # Auto-scaling
  data_capture_config {
    enable_capture              = true
    initial_sampling_percentage = 100
    destination_s3_uri          = "s3://model-monitoring/capture/"
    
    capture_options {
      capture_mode = "InputAndOutput"
    }
  }
}

# Auto-scaling for inference endpoint
resource "aws_appautoscaling_target" "sagemaker" {
  max_capacity       = 10
  min_capacity       = 3
  resource_id        = "endpoint/${aws_sagemaker_endpoint.fraud.name}/variant/primary"
  scalable_dimension = "sagemaker:variant:DesiredInstanceCount"
  service_namespace  = "sagemaker"
}

resource "aws_appautoscaling_policy" "sagemaker" {
  name               = "fraud-endpoint-scaling"
  resource_id        = aws_appautoscaling_target.sagemaker.resource_id
  scalable_dimension = aws_appautoscaling_target.sagemaker.scalable_dimension
  service_namespace  = aws_appautoscaling_target.sagemaker.service_namespace
  policy_type        = "TargetTrackingScaling"
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "SageMakerVariantInvocationsPerInstance"
    }
    target_value       = 1000  # invocations per instance
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

**Latency Budget:**
```
Total: < 500ms (P99)
├── MSK produce: 5ms
├── Flink processing: 50-100ms
├── SageMaker inference: 20-50ms
├── MSK consume (action): 5ms
└── Decision routing: 10ms
Buffer: ~300ms for retries/jitter
```

---

### Q9: Your Glue Data Catalog has 50,000+ tables across 20 accounts. Crawlers are conflicting, schemas are drifting, and consumers are breaking. How do you implement governance?

**Answer:**

**Data Governance Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                Data Governance Layer                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Schema       │  │ Data Quality │  │ Data Lineage     │  │
│  │ Registry     │  │ (Glue DQ)    │  │ (Catalog + Tags) │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Lake         │  │ Access       │  │ Lifecycle        │  │
│  │ Formation    │  │ Control      │  │ Management       │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**Solution 1: Schema Registry + Evolution Control:**

```python
# Use Glue Schema Registry for Kafka/streaming data
import boto3
from aws_schema_registry import DataAndSchema, SchemaRegistryClient
from aws_schema_registry.avro import AvroSchema

# Define schema with compatibility mode
schema_registry = boto3.client('glue')

schema_registry.create_registry(
    RegistryName='data-platform-registry',
    Description='Central schema registry for all data products'
)

# Create schema with BACKWARD compatibility (consumers safe)
schema_registry.create_schema(
    RegistryName='data-platform-registry',
    SchemaName='transactions',
    DataFormat='AVRO',
    Compatibility='BACKWARD',  # New schema can read old data
    SchemaDefinition=json.dumps({
        "type": "record",
        "name": "Transaction",
        "namespace": "com.company.data",
        "fields": [
            {"name": "txn_id", "type": "string"},
            {"name": "amount", "type": "double"},
            {"name": "timestamp", "type": "long"},
            {"name": "customer_id", "type": "string"},
            # New field with default (backward compatible)
            {"name": "channel", "type": ["null", "string"], "default": None}
        ]
    })
)
```

**Solution 2: Controlled Crawler Strategy:**

```python
# DON'T: Run crawlers with broad S3 paths
# DO: Use targeted crawlers with strict configuration

import boto3
glue = boto3.client('glue')

# Pattern: One crawler per data product (not per table)
glue.create_crawler(
    Name='product-orders-crawler',
    Role='GlueCrawlerRole',
    DatabaseName='orders_db',
    Targets={
        'S3Targets': [{
            'Path': 's3://datalake/gold/orders/',
            'Exclusions': ['_temporary/**', '_spark_metadata/**']
        }]
    },
    SchemaChangePolicy={
        'UpdateBehavior': 'LOG',        # Don't auto-update! Log changes for review
        'DeleteBehavior': 'DEPRECATE_IN_DATABASE'  # Don't delete tables
    },
    Configuration=json.dumps({
        "Version": 1.0,
        "CrawlerOutput": {
            "Partitions": {"AddOrUpdateBehavior": "InheritFromTable"},
            "Tables": {"AddOrUpdateBehavior": "MergeNewColumns"}  # Only add, never remove
        },
        "Grouping": {
            "TableGroupingPolicy": "CombineCompatibleSchemas"
        }
    }),
    Schedule='cron(0 */6 * * ? *)',  # Every 6 hours, not continuous
    Tags={
        'DataProduct': 'orders',
        'Owner': 'team-commerce',
        'SLA': 'tier-1'
    }
)
```

**Solution 3: Glue Data Quality Rules:**

```python
# Define quality rules per table
glue.create_data_quality_ruleset(
    Name='orders_quality_rules',
    TargetTable={
        'TableName': 'orders',
        'DatabaseName': 'gold_db'
    },
    Ruleset="""
        Rules = [
            # Completeness
            Completeness "order_id" >= 1.0,
            Completeness "customer_id" >= 0.99,
            Completeness "amount" >= 1.0,
            
            # Validity
            ColumnValues "amount" > 0,
            ColumnValues "order_status" in ["pending", "completed", "cancelled", "refunded"],
            
            # Uniqueness
            IsUnique "order_id",
            
            # Freshness
            Freshness "order_date" <= 24 hours,
            
            # Referential integrity
            ReferentialIntegrity "customer_id" "customers_db.customers.customer_id" >= 0.98,
            
            # Statistical
            StandardDeviation "amount" between 10 and 500,
            ColumnValues "amount" <= 50000,
            
            # Row count (detect data loss)
            RowCount between 10000 and 1000000
        ]
    """
)
```

**Solution 4: Data Contracts (Infrastructure as Code):**

```yaml
# data-contract.yaml - version controlled per data product
apiVersion: datacontract/v1
kind: DataContract
metadata:
  name: orders-gold
  owner: team-commerce
  tier: tier-1
  sla: 99.9%
  freshness: 1h
  
schema:
  type: avro
  registryName: data-platform-registry
  schemaName: orders-gold
  compatibility: BACKWARD
  
quality:
  completeness:
    order_id: 100%
    customer_id: 99%
  freshness: 24h
  
access:
  lake_formation:
    grants:
      - principal: "arn:aws:iam::ANALYTICS_ACCT:role/BITeam"
        permissions: [SELECT]
        columns: [order_id, amount, status, date]  # No PII
      - principal: "arn:aws:iam::RISK_ACCT:role/RiskTeam"
        permissions: [SELECT]
        columns: ["*"]  # Full access

lifecycle:
  retention: 7 years
  archival: glacier after 1 year
  deletion: approved by data-governance-board
```

---

### Q10: You need to build a cost-effective data lakehouse on AWS that supports both batch analytics (Redshift/Athena) and real-time queries. What's the architecture?

**Answer:**

**Lakehouse Architecture with Apache Iceberg:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA SOURCES                                  │
│  ┌────────┐  ┌──────────┐  ┌─────────┐  ┌────────────────┐    │
│  │ RDS    │  │ Kafka/MSK│  │ APIs    │  │ SaaS (S3 imp) │    │
│  └───┬────┘  └────┬─────┘  └────┬────┘  └───────┬────────┘    │
└──────┼─────────────┼────────────┼────────────────┼──────────────┘
       │             │            │                │
       ▼             ▼            ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                  INGESTION LAYER                                  │
│  ┌──────────┐  ┌───────────┐  ┌─────────────┐  ┌───────────┐  │
│  │ DMS      │  │ Firehose  │  │ AppFlow     │  │ Event     │  │
│  │ (CDC)    │  │ (Stream)  │  │ (SaaS)      │  │ Bridge    │  │
│  └──────────┘  └───────────┘  └─────────────┘  └───────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  STORAGE LAYER (S3 + Iceberg)                     │
│                                                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   BRONZE        │  │   SILVER        │  │   GOLD          │ │
│  │   (Raw)         │  │   (Cleaned)     │  │   (Curated)     │ │
│  │                 │  │                 │  │                 │ │
│  │  - Original fmt │  │  - Deduplicated │  │  - Business     │ │
│  │  - Append only  │  │  - Typed        │  │    aggregates   │ │
│  │  - Full history │  │  - Validated    │  │  - Star schema  │ │
│  │  - Iceberg tbl  │  │  - Iceberg tbl  │  │  - Iceberg tbl  │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                   │
│  Table Format: Apache Iceberg                                     │
│  Catalog: AWS Glue Data Catalog                                   │
│  Storage: S3 (Intelligent Tiering)                               │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PROCESSING LAYER                                 │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Glue (ETL)   │  │ EMR          │  │ Flink (streaming)    │  │
│  │ - Bronze→Slvr│  │ - Heavy xfrm │  │ - Real-time upserts  │  │
│  │ - Scheduled  │  │ - ML training│  │ - CDC processing     │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                   │
│  Orchestration: MWAA (Airflow)                                   │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  CONSUMPTION LAYER                                │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Athena       │  │ Redshift     │  │ SageMaker            │  │
│  │ (Ad-hoc SQL) │  │ (BI/Dashbrd) │  │ (ML Training)        │  │
│  │ - Serverless │  │ - Spectrum   │  │ - Feature Store      │  │
│  │ - Pay/query  │  │ - Data Share │  │ - Notebooks          │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Iceberg Table Implementation:**

```python
# Create Iceberg tables in Glue Catalog
spark.sql("""
    CREATE TABLE glue_catalog.silver.orders (
        order_id STRING,
        customer_id STRING,
        product_id STRING,
        quantity INT,
        amount DECIMAL(10,2),
        status STRING,
        created_at TIMESTAMP,
        updated_at TIMESTAMP
    )
    USING iceberg
    PARTITIONED BY (days(created_at))
    LOCATION 's3://datalake/silver/orders/'
    TBLPROPERTIES (
        'table_type' = 'ICEBERG',
        'format' = 'parquet',
        'write.format.default' = 'parquet',
        'write.target-file-size-bytes' = '134217728',
        'write.parquet.compression-codec' = 'zstd',
        'write.metadata.compression-codec' = 'gzip',
        'write.upsert.enabled' = 'true',
        'format-version' = '2'
    )
""")

# MERGE for upserts (real-time CDC from DMS)
spark.sql("""
    MERGE INTO glue_catalog.silver.orders AS target
    USING staging_orders AS source
    ON target.order_id = source.order_id
    WHEN MATCHED AND source.op = 'U' THEN
        UPDATE SET *
    WHEN MATCHED AND source.op = 'D' THEN
        DELETE
    WHEN NOT MATCHED AND source.op IN ('I', 'R') THEN
        INSERT *
""")

# Time travel query (rollback/audit)
spark.sql("""
    SELECT * FROM glue_catalog.silver.orders
    FOR SYSTEM_TIME AS OF TIMESTAMP '2024-06-01 00:00:00'
""")

# Incremental reads (for downstream processing)
spark.sql("""
    SELECT * FROM glue_catalog.silver.orders
    AFTER TIMESTAMP '2024-06-14 12:00:00'
""")
```

**Cost Optimization:**
```
Tier 1 (Hot - Gold tables): S3 Standard
Tier 2 (Warm - Silver): S3 Intelligent-Tiering
Tier 3 (Cold - Bronze archive): S3 Glacier Instant Retrieval
Tier 4 (Archive): S3 Glacier Deep Archive

Compute:
- Glue: Flex execution (50% cheaper, non-urgent jobs)
- EMR: Spot instances for task nodes (70% savings)
- Athena: Workgroups with query cost limits
- Redshift: Serverless for variable workloads
```
