# Data Lake Architecture - Enterprise Design Patterns

## Medallion Architecture (Bronze / Silver / Gold)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA LAKE ZONES                                │
│                                                                       │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐   │
│  │    BRONZE        │   │    SILVER        │   │    GOLD          │   │
│  │    (Landing)     │   │    (Conformed)   │   │    (Curated)     │   │
│  │                  │   │                  │   │                  │   │
│  │  • Raw data      │──▶│  • Cleaned       │──▶│  • Business      │   │
│  │  • As-is format  │   │  • Deduplicated  │   │    logic applied │   │
│  │  • Immutable     │   │  • Schema        │   │  • Aggregated    │   │
│  │  • Append-only   │   │    enforced      │   │  • Star/snowflake│   │
│  │  • Full history  │   │  • Type-safe     │   │  • Performance   │   │
│  │  • JSON/CSV/Avro │   │  • Parquet/ORC   │   │    optimized     │   │
│  │                  │   │  • Iceberg tables│   │  • Iceberg tables│   │
│  │  Retention: ∞    │   │  Retention: 2yr  │   │  Retention: 5yr  │   │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘   │
│                                                                       │
│  ┌─────────────────┐   ┌─────────────────┐                          │
│  │   QUARANTINE     │   │   SANDBOX        │                          │
│  │   (Failed data)  │   │   (Exploration)  │                          │
│  │                  │   │                  │                          │
│  │  • DQ failures  │   │  • Ad-hoc work   │                          │
│  │  • Schema issues│   │  • Temp datasets │                          │
│  │  • For review   │   │  • 30-day TTL    │                          │
│  └─────────────────┘   └─────────────────┘                          │
└─────────────────────────────────────────────────────────────────────┘
```

## S3 Bucket Strategy

### Bucket Organization:
```
# One bucket per zone per environment (recommended)
s3://company-datalake-bronze-prod/
s3://company-datalake-silver-prod/
s3://company-datalake-gold-prod/
s3://company-datalake-sandbox-prod/

# Path convention within buckets:
s3://company-datalake-silver-prod/
├── database_name/
│   ├── table_name/
│   │   ├── year=2024/
│   │   │   ├── month=06/
│   │   │   │   ├── day=15/
│   │   │   │   │   ├── part-00000-uuid.snappy.parquet
│   │   │   │   │   └── part-00001-uuid.snappy.parquet
```

### S3 Configuration:
```hcl
resource "aws_s3_bucket" "bronze" {
  bucket = "company-datalake-bronze-prod"
  
  tags = {
    Zone        = "bronze"
    Environment = "production"
    DataClass   = "internal"
  }
}

# Versioning for data protection
resource "aws_s3_bucket_versioning" "bronze" {
  bucket = aws_s3_bucket.bronze.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Lifecycle rules for cost optimization
resource "aws_s3_bucket_lifecycle_configuration" "bronze" {
  bucket = aws_s3_bucket.bronze.id
  
  rule {
    id     = "transition-to-ia"
    status = "Enabled"
    
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
    
    transition {
      days          = 90
      storage_class = "GLACIER_IR"  # Instant retrieval
    }
    
    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }
  }
  
  rule {
    id     = "cleanup-incomplete-multipart"
    status = "Enabled"
    
    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
  
  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER_IR"
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# Server-side encryption (KMS)
resource "aws_s3_bucket_server_side_encryption_configuration" "bronze" {
  bucket = aws_s3_bucket.bronze.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.datalake.arn
    }
    bucket_key_enabled = true  # Reduce KMS API calls by 99%
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "bronze" {
  bucket = aws_s3_bucket.bronze.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## Lake Formation Architecture

### Centralized Governance:
```
┌──────────────────────────────────────────────────────────┐
│              Lake Formation (Governance Account)           │
│                                                           │
│  ┌─────────────────┐  ┌──────────────────────────────┐  │
│  │  Data Catalog   │  │  Access Control               │  │
│  │  (Glue Catalog) │  │  - Database-level             │  │
│  │                 │  │  - Table-level                 │  │
│  │  50,000+ tables │  │  - Column-level               │  │
│  │  20 databases   │  │  - Row-level (cell filters)   │  │
│  │                 │  │  - Tag-based (TBAC)           │  │
│  └─────────────────┘  └──────────────────────────────┘  │
│                                                           │
│  ┌─────────────────┐  ┌──────────────────────────────┐  │
│  │  LF-Tags        │  │  Cross-Account Sharing       │  │
│  │  - Sensitivity  │  │  - Data Shares               │  │
│  │  - Domain       │  │  - Resource Links            │  │
│  │  - PII          │  │  - Named Resources           │  │
│  │  - Team         │  │                              │  │
│  └─────────────────┘  └──────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Tag-Based Access Control (TBAC):
```python
import boto3

lf = boto3.client('lakeformation')

# Create LF-Tags
lf.create_lf_tag(TagKey='Sensitivity', TagValues=['public', 'internal', 'confidential', 'restricted'])
lf.create_lf_tag(TagKey='Domain', TagValues=['finance', 'marketing', 'engineering', 'hr'])
lf.create_lf_tag(TagKey='PII', TagValues=['true', 'false'])

# Assign tags to tables
lf.add_lf_tags_to_resource(
    Resource={
        'Table': {
            'DatabaseName': 'gold_db',
            'Name': 'customer_profiles'
        }
    },
    LFTags=[
        {'TagKey': 'Sensitivity', 'TagValues': ['confidential']},
        {'TagKey': 'Domain', 'TagValues': ['marketing']},
        {'TagKey': 'PII', 'TagValues': ['true']}
    ]
)

# Assign tags to specific columns
lf.add_lf_tags_to_resource(
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'gold_db',
            'Name': 'customer_profiles',
            'ColumnNames': ['ssn', 'phone_number', 'email']
        }
    },
    LFTags=[
        {'TagKey': 'PII', 'TagValues': ['true']},
        {'TagKey': 'Sensitivity', 'TagValues': ['restricted']}
    ]
)

# Grant access based on tags (not individual resources!)
# Marketing team can access all 'marketing' domain tables that are 'internal' or 'public'
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::MARKETING_ACCT:role/MarketingAnalyst'},
    Resource={
        'LFTagPolicy': {
            'ResourceType': 'TABLE',
            'Expression': [
                {'TagKey': 'Domain', 'TagValues': ['marketing']},
                {'TagKey': 'Sensitivity', 'TagValues': ['public', 'internal']}
            ]
        }
    },
    Permissions=['SELECT', 'DESCRIBE']
)

# When new tables are tagged with Domain=marketing + Sensitivity=internal,
# Marketing team automatically gets access. No manual grants needed!
```

## Apache Iceberg on AWS

### Why Iceberg?
| Feature | Hive/Traditional | Iceberg |
|---------|-----------------|---------|
| ACID Transactions | ❌ | ✅ |
| Schema Evolution | Break consumers | Safe evolution |
| Time Travel | ❌ | ✅ |
| Partition Evolution | Rewrite data | Metadata only |
| Hidden Partitioning | ❌ | ✅ |
| Row-level Updates | Full rewrite | MERGE support |
| Concurrent Writers | Conflicts | Optimistic concurrency |

### Iceberg Table Management:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.sql.catalog.glue_catalog", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.glue_catalog.catalog-impl", "org.apache.iceberg.aws.glue.GlueCatalog") \
    .config("spark.sql.catalog.glue_catalog.warehouse", "s3://datalake-warehouse/") \
    .config("spark.sql.catalog.glue_catalog.io-impl", "org.apache.iceberg.aws.s3.S3FileIO") \
    .getOrCreate()

# Schema Evolution (safe - won't break consumers)
spark.sql("""
    ALTER TABLE glue_catalog.silver.orders 
    ADD COLUMNS (
        discount_code STRING AFTER amount,
        shipping_method STRING
    )
""")

# Partition Evolution (no data rewrite!)
spark.sql("""
    ALTER TABLE glue_catalog.silver.orders
    ADD PARTITION FIELD hours(created_at)
""")

# Expire old snapshots (storage optimization)
spark.sql("""
    CALL glue_catalog.system.expire_snapshots(
        table => 'silver.orders',
        older_than => TIMESTAMP '2024-05-01 00:00:00',
        retain_last => 10
    )
""")

# Remove orphan files (cleanup)
spark.sql("""
    CALL glue_catalog.system.remove_orphan_files(
        table => 'silver.orders',
        older_than => TIMESTAMP '2024-05-01 00:00:00'
    )
""")

# Rollback to previous snapshot (incident recovery!)
spark.sql("""
    CALL glue_catalog.system.rollback_to_snapshot(
        table => 'silver.orders',
        snapshot_id => 1234567890123456789
    )
""")
```

## Data Ingestion Patterns

### Pattern 1: CDC with DMS + Iceberg
```
┌──────────┐     ┌─────────┐     ┌──────────┐     ┌──────────────┐
│ RDS/     │────▶│  DMS    │────▶│ Kinesis  │────▶│ Flink/Glue   │
│ Aurora   │ CDC │ (Source) │     │ Data     │     │ (MERGE into  │
│          │     │         │     │ Stream   │     │  Iceberg)    │
└──────────┘     └─────────┘     └──────────┘     └──────────────┘
```

```python
# DMS task configuration for CDC
dms_task_settings = {
    "TargetMetadata": {
        "TargetSchema": "",
        "SupportLobs": True,
        "LimitedSizeLobMode": True,
        "LobMaxSize": 32
    },
    "FullLoadSettings": {
        "TargetTablePrepMode": "DO_NOTHING"
    },
    "StreamBufferSettings": {
        "StreamBufferCount": 3,
        "StreamBufferSizeInMB": 8
    },
    "ChangeProcessingTuning": {
        "BatchApplyEnabled": True,
        "BatchApplyPreserveTransaction": True,
        "BatchSplitSize": 0,
        "MinTransactionSize": 1000,
        "CommitTimeout": 1,
        "MemoryLimitTotal": 1024,
        "MemoryKeepTime": 60,
        "StatementCacheSize": 50
    }
}
```

### Pattern 2: Streaming Ingestion with Firehose
```hcl
resource "aws_kinesis_firehose_delivery_stream" "to_datalake" {
  name        = "events-to-datalake"
  destination = "extended_s3"
  
  extended_s3_configuration {
    role_arn   = aws_iam_role.firehose.arn
    bucket_arn = aws_s3_bucket.bronze.arn
    prefix     = "events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/"
    
    # Convert to Parquet on ingestion
    data_format_conversion_configuration {
      input_format_configuration {
        deserializer {
          open_x_json_ser_de {}
        }
      }
      output_format_configuration {
        serializer {
          parquet_ser_de {
            compression = "SNAPPY"
          }
        }
      }
      schema_configuration {
        database_name = aws_glue_catalog_database.bronze.name
        table_name    = aws_glue_catalog_table.events.name
        role_arn      = aws_iam_role.firehose.arn
      }
    }
    
    # Buffer configuration
    buffering_size     = 128  # MB - larger = fewer files
    buffering_interval = 300  # seconds
    
    # Error handling
    error_output_prefix = "errors/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/"
    
    # Dynamic partitioning
    dynamic_partitioning_configuration {
      enabled = true
    }
    
    processing_configuration {
      enabled = true
      processors {
        type = "MetadataExtraction"
        parameters {
          parameter_name  = "JsonParsingEngine"
          parameter_value = "JQ-1.6"
        }
        parameters {
          parameter_name  = "MetadataExtractionQuery"
          parameter_value = "{customer_segment:.customer_segment}"
        }
      }
    }
  }
}
```

### Pattern 3: Batch Ingestion with Glue
```python
# Incremental batch ingestion using bookmarks
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read with job bookmarks (only new data since last run)
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="raw_db",
    table_name="source_table",
    transformation_ctx="datasource",  # Required for bookmarks
    additional_options={
        "jobBookmarkKeys": ["last_modified_date"],
        "jobBookmarkKeysSortOrder": "asc"
    }
)

# Transform
mapped = ApplyMapping.apply(
    frame=datasource,
    mappings=[
        ("id", "string", "id", "string"),
        ("name", "string", "name", "string"),
        ("amount", "string", "amount", "decimal(10,2)"),
        ("created_at", "string", "created_at", "timestamp")
    ]
)

# Write with partition
glueContext.write_dynamic_frame.from_options(
    frame=mapped,
    connection_type="s3",
    format="glueparquet",
    connection_options={
        "path": "s3://datalake-silver/processed_table/",
        "partitionKeys": ["year", "month"]
    },
    format_options={"compression": "snappy"},
    transformation_ctx="datasink"
)

job.commit()  # Commit bookmark
```

## Data Lake Security

### Encryption Strategy:
```
┌─────────────────────────────────────────────────────┐
│  Encryption at Rest                                  │
│                                                      │
│  Bronze: SSE-KMS (aws/s3 managed key)               │
│  Silver: SSE-KMS (customer-managed key, per-domain)  │
│  Gold:   SSE-KMS (customer-managed key, restricted)  │
│                                                      │
│  Key rotation: Automatic (annual)                    │
│  Key policy: Grants per account/role                 │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Encryption in Transit                               │
│                                                      │
│  S3: Enforce ssl (bucket policy)                     │
│  EMR: In-transit encryption for HDFS/Spark shuffle   │
│  Redshift: SSL required for all connections          │
│  Glue: JDBC connections use SSL                      │
└─────────────────────────────────────────────────────┘
```

### S3 Bucket Policy (Enforce Encryption):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedPut",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::company-datalake-gold-prod/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyWrongKMSKey",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::company-datalake-gold-prod/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption-aws-kms-key-id": "arn:aws:kms:us-east-1:123456789:key/gold-key-id"
        }
      }
    },
    {
      "Sid": "DenyNonSSL",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::company-datalake-gold-prod",
        "arn:aws:s3:::company-datalake-gold-prod/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

## Performance Optimization

### Partition Strategy:
```python
# Good: Date-based partitioning for time-series data
# Result: ~100MB-1GB per partition (optimal for Athena)
df.write.partitionBy("year", "month", "day").parquet(path)

# Bad: Over-partitioning (too many small files)
# DON'T: partitionBy("year", "month", "day", "hour", "minute", "customer_id")

# Better: Use Iceberg hidden partitioning
spark.sql("""
    CREATE TABLE events (...)
    USING iceberg
    PARTITIONED BY (days(event_time), bucket(16, customer_id))
""")
# Iceberg handles partition pruning transparently
# Queries don't need to know partition structure
```

### File Size Optimization:
```
Target file sizes:
- Athena: 128MB - 256MB (optimal split size)
- Redshift Spectrum: 128MB - 512MB
- EMR/Spark: 128MB - 1GB

Rules:
- Never < 1MB (small files problem)
- Never > 4GB (memory pressure)
- Sweet spot: 128-256MB compressed Parquet
```

### Athena Query Optimization:
```sql
-- Use CTAS for materialized views
CREATE TABLE gold_db.daily_revenue
WITH (
    format = 'PARQUET',
    parquet_compression = 'ZSTD',
    partitioned_by = ARRAY['year', 'month'],
    bucketed_by = ARRAY['customer_id'],
    bucket_count = 32,
    external_location = 's3://datalake-gold/daily_revenue/'
) AS
SELECT 
    customer_id,
    DATE_TRUNC('day', order_time) as order_date,
    SUM(amount) as total_revenue,
    COUNT(*) as order_count,
    year(order_time) as year,
    month(order_time) as month
FROM silver_db.orders
GROUP BY 1, 2, year(order_time), month(order_time);

-- Use Iceberg for incremental updates instead of full CTAS
-- Athena v3 supports Iceberg MERGE
MERGE INTO gold_db.daily_revenue AS target
USING (
    SELECT customer_id, order_date, total_revenue, order_count
    FROM new_daily_data
) AS source
ON target.customer_id = source.customer_id 
   AND target.order_date = source.order_date
WHEN MATCHED THEN UPDATE SET 
    total_revenue = source.total_revenue,
    order_count = source.order_count
WHEN NOT MATCHED THEN INSERT VALUES (
    source.customer_id, source.order_date, 
    source.total_revenue, source.order_count,
    year(source.order_date), month(source.order_date)
);
```
