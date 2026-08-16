# Amazon Redshift — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | Details |
|-----------|---------|
| **Service Name** | Amazon Redshift |
| **Category** | Analytics / Data Warehousing |
| **Type** | Columnar, Massively Parallel Processing (MPP) |
| **Engine** | PostgreSQL-compatible (PartiQL for semi-structured) |
| **Launched** | February 2013 |
| **Pricing Model** | Per-node-hour (Provisioned) or per-RPU-hour (Serverless) |
| **Key Differentiator** | Petabyte-scale analytics with SQL, integrated with S3 via Spectrum |

---

## 🏗️ What Amazon Redshift Does

Redshift is a **cloud data warehouse** built for analytics at scale — it stores structured/semi-structured data in columnar format and executes complex SQL queries across massive datasets using MPP architecture.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AMAZON REDSHIFT                                  │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  LEADER NODE                                                     ││
│  │  ├── Receives SQL queries from clients                          ││
│  │  ├── Parses, optimizes, creates execution plan                  ││
│  │  ├── Distributes work to compute nodes                          ││
│  │  └── Aggregates results and returns to client                   ││
│  └─────────────────────────────────────────────────────────────────┘│
│           │                    │                    │                 │
│           ▼                    ▼                    ▼                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │ COMPUTE      │    │ COMPUTE      │    │ COMPUTE      │          │
│  │ NODE 1       │    │ NODE 2       │    │ NODE N       │          │
│  │              │    │              │    │              │          │
│  │ Slice 1 │ S2│    │ Slice 1 │ S2│    │ Slice 1 │ S2│          │
│  │ (1 CPU   )  │    │ (1 CPU   )  │    │ (1 CPU   )  │          │
│  │              │    │              │    │              │          │
│  │ Local SSD   │    │ Local SSD   │    │ Local SSD   │          │
│  │ or Managed  │    │ or Managed  │    │ or Managed  │          │
│  │ Storage(RA3)│    │ Storage(RA3)│    │ Storage(RA3)│          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  REDSHIFT MANAGED STORAGE (RMS) — for RA3 nodes                 ││
│  │  ├── Hot data cached on local NVMe SSD                          ││
│  │  ├── Warm/cold data on S3 (auto-tiered)                         ││
│  │  ├── Scales independently of compute                            ││
│  │  └── Pay only for data stored ($0.024/GB/month)                 ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

---

## 💰 Pricing

### Redshift Provisioned

| Node Type | vCPU | Memory | Storage | Price/hr |
|-----------|------|--------|---------|----------|
| dc2.large | 2 | 15 GB | 160 GB SSD | $0.25 |
| dc2.8xlarge | 32 | 244 GB | 2.56 TB SSD | $4.80 |
| ra3.xlplus | 4 | 32 GB | Managed (up to 32 TB) | $1.086 |
| ra3.4xlarge | 12 | 96 GB | Managed (up to 128 TB) | $3.26 |
| ra3.16xlarge | 48 | 384 GB | Managed (up to 128 TB) | $13.04 |

**RA3 managed storage:** $0.024/GB/month (separate from compute)

### Redshift Serverless

| Component | Cost |
|-----------|------|
| Compute (RPU) | $0.375/RPU-hour |
| Storage | $0.024/GB/month |
| Base capacity | 8-512 RPU (configurable) |

**Cost Example:**
```
Provisioned: 3 × ra3.4xlarge (24/7):
= 3 × $3.26 × 24 × 30 = $7,041/month + storage

Serverless: Queries run 8 hours/day at 32 RPU average:
= 32 × $0.375 × 8 × 30 = $2,880/month + storage

Savings with Serverless: 59% (if workload is intermittent)
```

---

## 📊 Key Service Limits

| Limit | Value |
|-------|-------|
| Max nodes per cluster | 128 (ra3.16xl) |
| Max databases per cluster | 60 |
| Max schemas per database | 9,900 |
| Max tables per database | 100,000 |
| Max columns per table | 1,600 |
| Max concurrent queries | 50 (WLM configurable) |
| Max query text length | 16 MB |
| Max COPY file size | 5 GB (compressed) |
| Max Spectrum files per query | 1,000,000 |
| Concurrency Scaling clusters | 10 (per main cluster) |
| Max RPU (Serverless) | 512 |

---

## 🎯 Real-World Use Cases

### Use Case 1: Enterprise BI Data Warehouse

```
Source Systems          →  ETL/ELT  →  Redshift  →  BI Tools
├── ERP (SAP)                          │              ├── QuickSight
├── CRM (Salesforce)    Glue/Firehose  │              ├── Tableau
├── Marketing (HubSpot)                │              ├── Power BI
└── Custom apps                        │              └── Looker

Star Schema in Redshift:
├── fact_orders (billions of rows, partitioned by date)
├── fact_page_views
├── dim_customers
├── dim_products
└── dim_time
```

**Why Redshift:** Sub-second query response for dashboards on petabyte-scale data.

---

### Use Case 2: Data Lakehouse (Redshift Spectrum + S3)

```
┌──────────────────────────────────────────────────────────────┐
│  HOT DATA (Recent 90 days)          │  COLD DATA (Historical)│
│  Stored IN Redshift                 │  Stored IN S3          │
│  (fast local queries)               │  (queried via Spectrum) │
│                                     │                        │
│  fact_orders (last 90d)             │  fact_orders_archive   │
│  dim_customers                      │  (years of history)    │
│                                     │  Format: Parquet       │
│  Query: < 1 second                  │  Query: 5-30 seconds   │
└──────────────────────────────────────────────────────────────┘

Unified View:
CREATE VIEW all_orders AS
SELECT * FROM local.fact_orders        -- Hot (Redshift local)
UNION ALL
SELECT * FROM spectrum.fact_orders_archive  -- Cold (S3 via Spectrum)
WHERE order_date < DATEADD(day, -90, GETDATE());

Users query ONE view — don't know/care where data physically lives!
```

**Why Redshift:** Best of both worlds — fast local + cheap S3 storage.

---

### Use Case 3: Real-Time Analytics Dashboard

```
Data Sources           →  Streaming Ingestion  →  Redshift  →  Dashboard
├── Payment events          Firehose/Kinesis       (Serverless)   (real-time)
├── User activity           (micro-batch)
└── System metrics          every 60 seconds

Materialized Views (auto-refresh):
CREATE MATERIALIZED VIEW mv_hourly_revenue
AUTO REFRESH YES
AS
SELECT DATE_TRUNC('hour', event_time) as hour,
       SUM(amount) as revenue,
       COUNT(*) as transactions
FROM streaming_events
GROUP BY 1;
```

**Why Redshift:** Streaming ingestion + auto-refreshing materialized views.

---

### Use Case 4: Multi-Tenant SaaS Analytics

```
One Redshift Cluster serves multiple customers:

Option A: Schema per tenant
├── tenant_001.fact_orders
├── tenant_002.fact_orders
└── Row-Level Security (RLS) policies

Option B: Shared schema with tenant column
├── fact_orders (tenant_id column)
└── RLS policy: Users see only their tenant's data

Option C: Data Sharing (cross-account)
Producer Cluster → Data Share → Consumer (Redshift Serverless per tenant)
└── No data copy, real-time access, independent compute
```

**Why Redshift:** Data Sharing + RLS enables multi-tenant isolation efficiently.

---

### Use Case 5: ML Feature Engineering

```
Raw Data (S3)  →  Redshift (transform, aggregate)  →  SageMaker
                      │
                      ├── Feature: avg_order_value_30d
                      ├── Feature: purchase_frequency
                      ├── Feature: days_since_last_login
                      └── Feature: lifetime_value

UNLOAD to S3 for training:
UNLOAD ('SELECT * FROM ml_features WHERE split = ''train''')
TO 's3://ml-bucket/training/'
FORMAT AS PARQUET;
```

**Why Redshift:** SQL-based feature engineering at scale, direct S3 integration.

---

### Use Case 6: Cross-Organization Data Sharing

```
Producer (Data Team Cluster):
├── CREATE DATASHARE analytics_share;
├── ADD TABLE fact_orders, dim_customers;
└── GRANT to Consumer namespace;

Consumer A (Marketing - Serverless):
├── Queries shared data (no copy, no ETL!)
├── Independent compute (doesn't affect producer)
└── Sees only granted tables/columns

Consumer B (Finance - Provisioned):
├── Same shared data
├── Different workload patterns
└── Can join with their own local tables
```

**Why Redshift:** Zero-copy data sharing across accounts/teams.

---

## 🔑 Key Features to Know

### 1. Distribution Styles
```sql
-- KEY distribution: Rows with same key on same node (good for JOINs)
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT
)
DISTKEY(customer_id);  -- JOINs on customer_id avoid network shuffle

-- EVEN distribution: Round-robin across all nodes (balanced load)
CREATE TABLE events (...)
DISTSTYLE EVEN;  -- Good when no clear join key

-- ALL distribution: Full copy on every node (for small dimension tables)
CREATE TABLE dim_status (...)
DISTSTYLE ALL;  -- < 1M rows, joined frequently — no network shuffle

-- AUTO distribution: Redshift decides based on table size
CREATE TABLE auto_table (...)
DISTSTYLE AUTO;  -- Starts as ALL, switches to EVEN as table grows
```

### 2. Sort Keys
```sql
-- COMPOUND sort key: Filter on first column is fast, subsequent benefit
CREATE TABLE events (
    event_date DATE,
    customer_id BIGINT,
    event_type VARCHAR(50)
)
COMPOUND SORTKEY(event_date, customer_id);
-- WHERE event_date = '2024-06-15' → very fast (zone maps skip blocks)
-- WHERE customer_id = 123 alone → no benefit (must have event_date first)

-- INTERLEAVED sort key: Equal benefit for any column combination
CREATE TABLE events (...)
INTERLEAVED SORTKEY(event_date, customer_id, event_type);
-- Any column in WHERE clause benefits equally
-- BUT: VACUUM REINDEX is expensive, use sparingly
```

### 3. Concurrency Scaling
```
Normal operation: Main cluster handles all queries
Peak hours: Queries queue up → Concurrency Scaling activates
→ Transient clusters spin up automatically (seconds)
→ Handle burst queries → Shut down when queue empty
→ First hour/day free, then $0.25/credit

Effect: Near-unlimited concurrency for BI dashboards
```

### 4. Workload Management (WLM)
```sql
-- Priority queues for different workload types:
-- Queue 1: BI dashboards (high priority, short queries, many concurrent)
-- Queue 2: Analysts (medium priority, longer queries, fewer concurrent)
-- Queue 3: ETL jobs (low priority, very long, few concurrent)

-- Automatic WLM (recommended):
-- Redshift ML decides priority, memory, concurrency automatically
-- Manual WLM for specific requirements (max_execution_time, memory %)
```

### 5. AQUA (Advanced Query Accelerator)
```
What: Hardware-accelerated cache layer between compute and storage
When: RA3 nodes with large scans on managed storage
Effect: 10x faster for scan-heavy queries (filtering, pattern matching)
Cost: Free (included with RA3 nodes)
Benefit: Pushes filtering/aggregation to storage layer
```

---

## ❓ Interview Questions & Answers

### Q1: Explain the difference between Redshift distribution styles. When would you choose KEY vs EVEN vs ALL?

**Answer:**

```
DECISION FRAMEWORK:

Table Size < 3M rows AND frequently joined?
└── YES → DISTSTYLE ALL (copy to every node, eliminate shuffles)
    Example: dim_country (200 rows), dim_product_category (5000 rows)

Table has a column that's always in JOIN conditions?
└── YES → DISTKEY(that_column)
    Example: fact_orders DISTKEY(customer_id) + dim_customers DISTKEY(customer_id)
    → JOIN between them requires ZERO network shuffle (co-located!)

No clear join pattern OR table is write-heavy?
└── → DISTSTYLE EVEN (balanced across all nodes)
    Example: staging tables, log tables

Not sure?
└── → DISTSTYLE AUTO (Redshift decides and can change over time)
```

**The JOIN shuffle problem:**
```
Without co-location (EVEN + EVEN):
Node 1 has: orders for customer 1,5,9,13...
Node 2 has: orders for customer 2,6,10,14...
Node 3 has: customer dim for customer 1,2,3,...

JOIN requires: Shuffle ALL data across network → SLOW!

With co-location (both DISTKEY customer_id):
Node 1 has: orders for customer 1,2,3 AND dim for customer 1,2,3
→ JOIN is LOCAL → NO network shuffle → 10-100x faster for large joins
```

---

### Q2: Your Redshift queries are slow. Walk me through systematic performance tuning.

**Answer:**

**Step 1: Identify the problem query**
```sql
-- Find slow queries
SELECT query, elapsed/1000000 as seconds, substring(querytxt, 1, 100)
FROM stl_query
WHERE elapsed > 60000000  -- > 60 seconds
ORDER BY elapsed DESC
LIMIT 20;
```

**Step 2: Check execution plan**
```sql
EXPLAIN SELECT ...;
-- Look for:
-- "DS_DIST_ALL_NONE" → Good (no redistribution)
-- "DS_DIST_INNER" → Bad (redistributing inner table, wrong dist key)
-- "DS_BCAST_INNER" → Broadcasting small table (OK if table is small)
```

**Step 3: Common fixes**

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| Network shuffle | `DS_DIST_INNER` in EXPLAIN | Change DISTKEY to JOIN column |
| Full table scan | Missing sort key filter | Add SORTKEY on WHERE column |
| Disk-based query | `is_diskbased = t` in svl_query_summary | Increase WLM memory or reduce data |
| Stale statistics | Wrong row estimates | Run `ANALYZE table_name` |
| Unsorted data | `pct_unsorted > 20%` in svv_table_info | Run `VACUUM SORT ONLY` |
| Tombstoned rows | `deleted_rows > 0` in svv_table_info | Run `VACUUM DELETE ONLY` |
| Skewed data | One node does 90% of work | Change distribution key |

**Step 4: Check for data skew**
```sql
-- Are slices balanced?
SELECT slice, COUNT(*) as rows
FROM stv_blocklist
WHERE tbl = (SELECT id FROM stv_tbl_perm WHERE name = 'your_table')
GROUP BY slice
ORDER BY rows DESC;
-- If one slice has 10x more → bad distribution key!
```

---

### Q3: What is Redshift Spectrum and when would you use it instead of loading data into Redshift?

**Answer:**

```
Redshift Spectrum = Query S3 data directly using Redshift SQL engine

┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Redshift Cluster          Spectrum Layer            S3 Data Lake │
│  ┌──────────────┐         ┌──────────────┐         ┌──────────┐│
│  │ SQL Engine   │────────▶│ Spectrum     │────────▶│ Parquet  ││
│  │ (parse,      │         │ Workers      │         │ ORC      ││
│  │  optimize)   │◀────────│ (filter,     │◀────────│ CSV      ││
│  │              │ results │  aggregate)  │ data    │ JSON     ││
│  └──────────────┘         └──────────────┘         └──────────┘│
│                                                                    │
│  Spectrum workers are SEPARATE from your cluster nodes!           │
│  They scale independently (up to 1000s of workers)               │
└──────────────────────────────────────────────────────────────────┘

LOAD into Redshift:                    QUERY via Spectrum:
├── Data: < 5TB, frequently queried   ├── Data: > 5TB, infrequently queried
├── Latency: Sub-second needed        ├── Latency: 5-30 seconds acceptable
├── Access: Many concurrent users     ├── Access: Ad-hoc, exploratory
├── Pattern: Same query repeated       ├── Pattern: Varied queries
├── Cost: Storage on Redshift ($)     ├── Cost: $5 per TB scanned
└── Best for: BI dashboards           └── Best for: Historical analysis
```

**Hybrid pattern (production standard):**
```sql
-- Hot data: Last 90 days in Redshift (fast)
-- Cold data: Historical in S3 via Spectrum (cheap)

CREATE VIEW unified_orders AS
SELECT * FROM redshift_schema.orders  -- Hot: local, sub-second
WHERE order_date >= DATEADD(day, -90, GETDATE())
UNION ALL
SELECT * FROM spectrum_schema.orders_historical  -- Cold: S3, seconds
WHERE order_date < DATEADD(day, -90, GETDATE());
```

---

### Q4: Explain Redshift Data Sharing. How does it differ from traditional data replication?

**Answer:**

```
TRADITIONAL REPLICATION:                DATA SHARING:
┌─────────────┐   ETL/Copy   ┌──────┐  ┌─────────────┐  Live Link  ┌──────┐
│ Producer DB │──────────────▶│Copy  │  │ Producer DB │─────────────▶│ View │
│             │               │of DB │  │             │              │(live)│
└─────────────┘               └──────┘  └─────────────┘              └──────┘
                                         
Problems:                              Benefits:
├── Data is stale (ETL lag)            ├── Real-time (no lag)
├── Storage duplicated ($$)            ├── Zero storage (no copy!)
├── ETL maintenance burden             ├── Zero ETL needed
├── Inconsistency risk                 ├── Always consistent
└── Scales poorly (N copies for N)     └── One source, many consumers
```

**Data Sharing implementation:**
```sql
-- PRODUCER (creates and shares data)
CREATE DATASHARE sales_share;
ALTER DATASHARE sales_share ADD SCHEMA public;
ALTER DATASHARE sales_share ADD TABLE public.fact_orders;
ALTER DATASHARE sales_share ADD TABLE public.dim_customers;

-- Grant to specific consumer (by AWS account or namespace)
GRANT USAGE ON DATASHARE sales_share TO NAMESPACE 'consumer-namespace-id';

-- CONSUMER (uses shared data without copying)
CREATE DATABASE sales_db FROM DATASHARE sales_share
  OF NAMESPACE 'producer-namespace-id';

-- Query shared data (live, real-time)
SELECT * FROM sales_db.public.fact_orders WHERE order_date = CURRENT_DATE;

-- Can JOIN shared data with local tables!
SELECT s.*, l.internal_notes
FROM sales_db.public.fact_orders s
JOIN local_schema.annotations l ON s.order_id = l.order_id;
```

**Key characteristics:**
- Producer storage only (consumer pays $0 for shared data storage)
- Consumer has independent compute (doesn't impact producer)
- Cross-account and cross-region supported
- Column-level access control possible
- Works with both Provisioned and Serverless

---

### Q5: Your Redshift cluster shows high WLM queue wait times during business hours. BI queries wait 30+ seconds before executing. How do you solve this?

**Answer:**

**Diagnosis:**
```sql
-- Check WLM queue waits
SELECT service_class, num_queued_queries, avg_queue_time/1000000 as avg_wait_sec
FROM stl_wlm_query
WHERE starttime > DATEADD(hour, -1, GETDATE())
GROUP BY service_class
ORDER BY avg_wait_sec DESC;

-- Check concurrent query slots
SELECT * FROM stv_wlm_service_class_config;
```

**Solution architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│  Multi-Layered Concurrency Solution:                             │
│                                                                   │
│  Layer 1: WLM Queue Prioritization                               │
│  ├── Queue "dashboard": priority=HIGHEST, concurrency=15         │
│  ├── Queue "analyst": priority=NORMAL, concurrency=5             │
│  └── Queue "etl": priority=LOW, concurrency=3                    │
│                                                                   │
│  Layer 2: Concurrency Scaling (auto burst)                       │
│  ├── Dashboard queue: concurrency_scaling = AUTO                 │
│  ├── Up to 10 transient clusters for peak traffic               │
│  └── First 1 hour/day free, then $0.25/credit                  │
│                                                                   │
│  Layer 3: Redshift Serverless (for heavy ad-hoc users)          │
│  ├── Analysts use Serverless endpoint (independent)             │
│  ├── Data Sharing from main cluster (zero copy)                 │
│  └── Auto-scales, no impact on main cluster                     │
│                                                                   │
│  Layer 4: Materialized Views (reduce query load)                │
│  ├── Pre-compute expensive aggregations                          │
│  ├── AUTO REFRESH keeps them current                            │
│  └── Dashboard queries hit MV (sub-second) not base tables     │
└─────────────────────────────────────────────────────────────────┘
```

```sql
-- Create materialized view for dashboards
CREATE MATERIALIZED VIEW mv_daily_sales
AUTO REFRESH YES
AS
SELECT 
    DATE_TRUNC('day', order_time) as day,
    region,
    product_category,
    SUM(amount) as revenue,
    COUNT(*) as orders,
    COUNT(DISTINCT customer_id) as customers
FROM fact_orders
GROUP BY 1, 2, 3;

-- Dashboard queries hit MV (instant) instead of scanning fact table
SELECT * FROM mv_daily_sales WHERE day >= CURRENT_DATE - 30;
```

---

### Q6: How does COPY command differ from INSERT for loading data into Redshift? When would you use each?

**Answer:**

```
COPY Command:                          INSERT Statement:
├── Parallel load from S3/DynamoDB     ├── Single-row or multi-row
├── All nodes load simultaneously      ├── Only leader node processes
├── Compressed formats supported       ├── No compression benefit
├── Automatic compression encoding     ├── No encoding detection
├── 10-100x faster for bulk loads      ├── Slow for large datasets
├── Handles manifest files             ├── Simple, familiar SQL
├── Best for: > 1000 rows              ├── Best for: < 1000 rows
└── Required format: S3/DynamoDB/EMR   └── Any SQL client

PERFORMANCE COMPARISON:
Loading 1 billion rows:
├── COPY from S3 (Parquet): ~10 minutes
├── INSERT (batch of 1000): ~50 hours (!!)
└── Difference: 300x
```

**COPY best practices:**
```sql
-- Optimal COPY command:
COPY fact_orders
FROM 's3://bucket/orders/2024/06/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftCopyRole'
FORMAT AS PARQUET;
-- Parquet is fastest (columnar, compressed, type-safe)

-- Multiple files in parallel (CRITICAL for speed):
-- Split your data into multiple files (ideally: files = slices × multiple)
-- A 6-node cluster with 2 slices each = 12 slices
-- Optimal: 12, 24, 48, or 96 input files
-- Each slice loads one file simultaneously!

-- With manifest (explicit file list):
COPY fact_orders
FROM 's3://bucket/manifests/orders_manifest.json'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftCopyRole'
MANIFEST
GZIP;

-- AVOID: Single large file (only one slice works!)
-- AVOID: Too many tiny files (overhead per file)
-- IDEAL: Files between 1MB and 1GB, compressed
```

---

### Q7: Explain Redshift's zone maps and how they relate to sort keys for query performance.

**Answer:**

```
ZONE MAPS = Redshift's "secret weapon" for fast filtering

What are zone maps?
├── Metadata stored for each 1MB block of data
├── Contains: MIN and MAX value of each column in that block
├── Stored in memory (fast to check)
└── Used to SKIP blocks that can't contain matching rows

HOW IT WORKS:
Block 1: [min=2024-01-01, max=2024-01-31]  ← month of January
Block 2: [min=2024-02-01, max=2024-02-28]  ← month of February
Block 3: [min=2024-03-01, max=2024-03-31]  ← month of March

Query: WHERE order_date = '2024-02-15'
├── Check Block 1: min=Jan, max=Jan → 2024-02-15 NOT in range → SKIP!
├── Check Block 2: min=Feb, max=Feb → 2024-02-15 IS in range → READ
├── Check Block 3: min=Mar, max=Mar → 2024-02-15 NOT in range → SKIP!
└── Result: Only reads 1 out of 3 blocks (67% I/O savings!)

WHY SORT KEY MATTERS:
SORTED data → values in order → zone maps have tight, non-overlapping ranges
UNSORTED data → values scattered → zone maps have wide, overlapping ranges

SORTED (sort key = order_date):
Block 1: [min=Jan-01, max=Jan-31]  ← Tight range, easy to skip
Block 2: [min=Feb-01, max=Feb-28]  ← Tight range, easy to skip

UNSORTED (no sort key):
Block 1: [min=Jan-05, max=Dec-20]  ← Wide range, CANNOT skip!
Block 2: [min=Jan-02, max=Nov-15]  ← Wide range, CANNOT skip!
→ Must read ALL blocks → zone maps useless → SLOW!

LESSON: Sort key = column you filter on most → zone maps work → fast queries
```

---

### Q8: When would you choose Redshift Serverless over Redshift Provisioned?

**Answer:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Choose SERVERLESS when:            │  Choose PROVISIONED when:       │
├─────────────────────────────────────┼─────────────────────────────────┤
│ Workload is unpredictable           │ Workload is predictable/steady  │
│ Usage is < 12 hours/day             │ Cluster runs 24/7               │
│ Don't want to manage cluster        │ Need fine-grained control       │
│ Need auto-scaling (0 to peak)       │ Need reserved pricing (savings) │
│ Ad-hoc analytics/exploration        │ Production BI (SLA-bound)       │
│ Development/testing environments    │ Complex WLM configuration       │
│ Data sharing consumer               │ Concurrency Scaling needed      │
│ New projects (quick start)          │ Very large datasets (>100TB)    │
│ Multiple small teams (isolated)     │ Require node-type selection     │
├─────────────────────────────────────┼─────────────────────────────────┤
│ Cost: $0.375/RPU-hour (pay per use) │ Cost: Reserved = 25-75% savings │
│ Scale: 0 to 512 RPU (auto)         │ Scale: Manual (resize cluster)  │
│ Idle: $0 when no queries            │ Idle: Full cost (always on)     │
└─────────────────────────────────────┴─────────────────────────────────┘

HYBRID PATTERN (common in enterprise):
├── Provisioned cluster: Production BI dashboards (24/7, predictable)
├── Serverless workspace: Analyst ad-hoc queries (variable, bursty)
├── Data Sharing connects them (zero copy, live data)
└── Result: BI gets guaranteed performance, analysts get auto-scaling
```

---

### Q9: How do you implement incremental loading into Redshift from a data lake?

**Answer:**

```sql
-- Pattern 1: MERGE (Upsert) — Redshift supports native MERGE (2023+)
MERGE INTO target_schema.dim_customers AS target
USING staging_schema.new_customers AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
    UPDATE SET 
        name = source.name,
        email = source.email,
        updated_at = GETDATE()
WHEN NOT MATCHED THEN
    INSERT (customer_id, name, email, created_at, updated_at)
    VALUES (source.customer_id, source.name, source.email, GETDATE(), GETDATE());

-- Pattern 2: Staging table + DELETE-INSERT (traditional, pre-MERGE)
-- Step 1: Load new data into staging
COPY staging.orders FROM 's3://bucket/incremental/2024-06-15/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftRole'
FORMAT AS PARQUET;

-- Step 2: Delete existing records that will be updated
DELETE FROM production.orders
USING staging.orders
WHERE production.orders.order_id = staging.orders.order_id;

-- Step 3: Insert all records from staging
INSERT INTO production.orders
SELECT * FROM staging.orders;

-- Step 4: Cleanup
TRUNCATE staging.orders;
-- Note: This runs in a single transaction (atomic!)

-- Pattern 3: Append-only with deduplication view
-- (Best for event/fact tables where data never updates)
COPY fact_events FROM 's3://bucket/events/dt=2024-06-15/'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftRole'
FORMAT AS PARQUET;
-- Dedup view:
CREATE VIEW deduped_events AS
SELECT *, ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY loaded_at DESC) as rn
FROM fact_events
WHERE rn = 1;  -- Only latest version of each event
```

---

### Q10: Your Redshift VACUUM is taking 12+ hours and blocking other operations. How do you handle table maintenance at scale?

**Answer:**

**Understanding VACUUM:**
```
WHY vacuum is needed:
├── DELETE doesn't physically remove rows (marks as "tombstone")
├── UPDATE = DELETE + INSERT (creates tombstones + unsorted rows)
├── Over time: table has dead space + unsorted regions
├── Queries scan dead space (slower)
└── VACUUM reclaims space + re-sorts

VACUUM Types:
├── VACUUM FULL: Reclaim space + re-sort (most expensive)
├── VACUUM DELETE ONLY: Just reclaim space (faster)
├── VACUUM SORT ONLY: Just re-sort (no space reclaim)
└── VACUUM REINDEX: Rebuild interleaved sort key indexes
```

**Production strategies:**

```sql
-- Strategy 1: Use automatic VACUUM (let Redshift handle it)
-- Redshift runs background vacuum during low-activity periods
-- Usually sufficient for tables with < 20% unsorted data

-- Strategy 2: Deep Copy instead of VACUUM (faster for heavily fragmented)
-- Create new table → Copy data → Swap names → Drop old
CREATE TABLE orders_new (LIKE orders);
INSERT INTO orders_new SELECT * FROM orders;
ALTER TABLE orders RENAME TO orders_old;
ALTER TABLE orders_new RENAME TO orders;
DROP TABLE orders_old;
-- This is often 5-10x faster than VACUUM for large tables!

-- Strategy 3: Avoid the need for VACUUM
-- Use TRUNCATE + INSERT instead of DELETE + INSERT
-- Use APPEND-only fact tables (never update)
-- Partition large tables by date, DROP old partitions instead of DELETE

-- Strategy 4: Incremental VACUUM (non-blocking)
VACUUM DELETE ONLY orders TO 95 PERCENT;
-- Only reclaims until 95% clean (doesn't try for 100%)
-- Much faster, doesn't block queries
-- Run nightly, never needs full vacuum

-- Strategy 5: Monitor and alert
SELECT "table", size, pct_used, unsorted, tbl_rows, 
       CASE 
         WHEN unsorted > 20 THEN 'NEEDS_SORT'
         WHEN (size - pct_used * size / 100) > 1000 THEN 'NEEDS_DELETE'
         ELSE 'OK'
       END as status
FROM svv_table_info
WHERE status != 'OK'
ORDER BY size DESC;
```

---

## 🆚 Redshift vs Competitors

| Feature | Redshift | BigQuery | Snowflake | Databricks SQL |
|---------|----------|----------|-----------|----------------|
| Architecture | MPP, Columnar | Serverless, Columnar | Multi-cluster, Columnar | Lakehouse |
| Pricing | Per-node or Per-RPU | Per-TB scanned | Per-credit (compute) | Per-DBU |
| Storage | Managed (RA3) or local | Fully managed | Fully managed | Delta Lake (S3) |
| Scaling | Manual/Serverless | Automatic | Auto-suspend/resume | Auto |
| Semi-structured | SUPER type, PartiQL | Native JSON | VARIANT type | Native |
| Streaming | Native ingestion | Streaming inserts | Snowpipe | Structured Streaming |
| Data sharing | Native (free) | BigQuery Omni | Secure Data Sharing | Delta Sharing |
| ML | Redshift ML (SageMaker) | BigQuery ML | Snowpark | MLflow + Spark |
| Best for | AWS-native, SQL analytics | GCP/multi-cloud, ad-hoc | Multi-cloud, ease | Lakehouse, ML |

---

## 🏆 Production Best Practices

```yaml
table_design:
  - Choose distribution key based on JOIN patterns (most frequent JOIN column)
  - Use SORTKEY on most common WHERE/filter column (usually date)
  - Small dimension tables (< 3M rows): DISTSTYLE ALL
  - Large fact tables: DISTKEY on the most joined column
  - Use ENCODE AUTO (let Redshift choose compression)

loading:
  - Always use COPY from S3 (never INSERT for bulk loads)
  - Split input into multiple files (files = 2× number of slices)
  - Use Parquet format (fastest, type-safe, compressed)
  - Use manifest files for explicit file lists
  - Load into staging → validate → merge into production

querying:
  - Create materialized views for repeated dashboard queries
  - Use WLM to prioritize interactive queries over ETL
  - Enable Concurrency Scaling for burst BI workloads
  - Use EXPLAIN to check for redistribution (shuffle) issues
  - Run ANALYZE after large loads (update statistics)

maintenance:
  - Monitor svv_table_info for unsorted % and deleted rows
  - Use automatic VACUUM (enabled by default)
  - Deep copy for heavily fragmented tables (faster than VACUUM)
  - ANALYZE tables after significant data loads
  - Set appropriate WLM timeout for runaway queries

cost:
  - Use Reserved Instances for 24/7 provisioned clusters (60% savings)
  - Use Serverless for variable/ad-hoc workloads
  - Use Spectrum for cold/historical data (don't load into Redshift)
  - Pause dev/test clusters during off-hours
  - Monitor with Cost Explorer: track cost per query/team
```
