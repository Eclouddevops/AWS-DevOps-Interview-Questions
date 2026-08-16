# Amazon MWAA — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | Details |
|-----------|---------|
| **Service Name** | Amazon MWAA (Managed Workflows for Apache Airflow) |
| **Category** | Analytics / Workflow Orchestration |
| **Type** | Managed Apache Airflow |
| **Engine** | Apache Airflow (versions 2.6 – 2.9+) |
| **Launched** | November 2020 (GA) |
| **Pricing Model** | Per environment-hour + per worker-hour |
| **Key Differentiator** | Fully managed Airflow — no infrastructure management for DAG orchestration |

---

## 🏗️ What Amazon MWAA Does

MWAA is a **managed Apache Airflow service** — it runs the Airflow scheduler, web server, and workers without you managing the underlying infrastructure (EC2, database, Redis, networking).

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AMAZON MWAA                                   │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  YOUR RESPONSIBILITY:                                            ││
│  │  ├── Write DAGs (Python files)                                  ││
│  │  ├── Upload to S3 (DAG bucket)                                  ││
│  │  ├── Define dependencies (requirements.txt)                     ││
│  │  ├── Configure Airflow parameters (overrides)                   ││
│  │  └── Monitor DAG execution (Airflow UI)                         ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  AWS MANAGES:                                                    ││
│  │  ├── Scheduler (HA — 2 schedulers)                              ││
│  │  ├── Web Server (Airflow UI, API)                               ││
│  │  ├── Workers (Celery + auto-scaling)                            ││
│  │  ├── Metadata Database (PostgreSQL, managed)                    ││
│  │  ├── Message Broker (SQS/Celery, managed)                      ││
│  │  ├── Networking (VPC, security groups)                          ││
│  │  ├── Patching, upgrades, HA                                     ││
│  │  └── Logging (CloudWatch integration)                           ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  Architecture:                                                        │
│  ┌──────────┐     ┌──────────────────────────────────────────┐      │
│  │  S3      │     │  MWAA Environment                         │      │
│  │  Bucket  │────▶│                                           │      │
│  │          │     │  ┌───────────┐  ┌───────────┐           │      │
│  │ /dags/   │     │  │Scheduler 1│  │Scheduler 2│  (HA)     │      │
│  │ /plugins/│     │  └─────┬─────┘  └─────┬─────┘           │      │
│  │ /require │     │        │               │                  │      │
│  │  ments.txt│    │        ▼               ▼                  │      │
│  └──────────┘     │  ┌─────────────────────────────────┐    │      │
│                    │  │  Workers (2-25, auto-scaling)    │    │      │
│                    │  │  ├── Worker 1 (Celery executor) │    │      │
│                    │  │  ├── Worker 2                    │    │      │
│                    │  │  └── Worker N                    │    │      │
│                    │  └─────────────────────────────────┘    │      │
│                    │                                           │      │
│                    │  ┌──────────┐  ┌──────────┐             │      │
│                    │  │ MetaDB   │  │ Web      │             │      │
│                    │  │(Postgres)│  │ Server   │             │      │
│                    │  └──────────┘  └──────────┘             │      │
│                    └──────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 💰 Pricing

| Component | mw1.small | mw1.medium | mw1.large |
|-----------|-----------|------------|-----------|
| Environment (scheduler+webserver) | $0.49/hr | $0.98/hr | $1.97/hr |
| Workers (per worker) | $0.055/hr | $0.109/hr | $0.218/hr |
| Additional schedulers | Included (2) | Included (2) | Included (2) |
| Storage (DAGs in S3) | Standard S3 | Standard S3 | Standard S3 |

**Environment Classes:**

| Class | Scheduler vCPU | Scheduler Memory | Worker vCPU | Worker Memory |
|-------|---------------|-----------------|-------------|---------------|
| mw1.small | 1 | 2 GB | 1 | 2 GB |
| mw1.medium | 2 | 4 GB | 2 | 4 GB |
| mw1.large | 4 | 8 GB | 4 | 8 GB |

**Cost Examples:**
```
Development environment (mw1.small, 2 workers):
= $0.49/hr (env) + 2 × $0.055/hr (workers)
= $0.60/hr × 730 hours = $438/month

Production environment (mw1.large, 5-15 workers avg 8):
= $1.97/hr (env) + 8 × $0.218/hr (workers)
= $3.71/hr × 730 hours = $2,711/month

Enterprise (mw1.large, 25 workers max):
= $1.97/hr (env) + 25 × $0.218/hr (workers)
= $7.42/hr × 730 hours = $5,417/month
```

---

## 📊 Key Service Limits

| Limit | Value |
|-------|-------|
| Min workers | 1 |
| Max workers | 25 |
| Schedulers | 2 (fixed, HA) |
| Max DAG files | 1,000 |
| DAG S3 bucket size | 1 GB |
| plugins.zip max size | 1 GB |
| requirements.txt packages | No hard limit (but install time matters) |
| Max environment per region | 10 |
| Max Airflow configurations | 50 overrides |
| Web server timeout | 30 seconds (API calls) |
| Task execution timeout | Configurable (no default limit) |
| DAG parsing timeout | 30 seconds (default) |

---

## 🎯 Real-World Use Cases

### Use Case 1: Data Lake ETL Orchestration (Most Common)

```
DAG: daily_data_pipeline
├── 07:00 — Trigger: Schedule (cron)
├── Task 1: check_source_data (sensor — wait for data)
├── Task 2: run_glue_etl (GlueJobOperator — bronze→silver)
├── Task 3: data_quality_check (PythonOperator — validate)
├── Task 4: run_gold_aggregation (GlueJobOperator — silver→gold)
├── Task 5: refresh_redshift (RedshiftSQLOperator — COPY)
├── Task 6: update_catalog (PythonOperator — Glue Catalog)
└── Task 7: notify_consumers (SNSOperator — Slack/email)

Dependencies:
1 → 2 → 3 → [4, 5 parallel] → 6 → 7
         └── if fails → alert + retry
```

**Why MWAA:** Complex multi-step pipeline with dependencies, retries, monitoring, and cross-service orchestration.

---

### Use Case 2: Cross-Account Data Processing

```
DAG: cross_account_pipeline
├── Task 1: assume_role_account_a → Read data from Account A
├── Task 2: assume_role_account_b → Process in Account B
├── Task 3: assume_role_account_c → Write results to Account C
└── Task 4: notify_data_team

Each task assumes different IAM roles for different accounts.
MWAA execution role has sts:AssumeRole permissions.
```

**Why MWAA:** Centralized orchestration across multiple AWS accounts.

---

### Use Case 3: ML Pipeline Orchestration

```
DAG: ml_training_pipeline (weekly)
├── Task 1: extract_training_data (Athena → S3)
├── Task 2: feature_engineering (GlueJobOperator)
├── Task 3: train_model (SageMakerTrainingOperator)
├── Task 4: evaluate_model (SageMakerProcessingOperator)
├── Task 5: compare_with_production (PythonOperator)
│   ├── If better → Task 6a: deploy_model (SageMakerEndpointOperator)
│   └── If worse → Task 6b: alert_team (no deployment)
└── Task 7: update_feature_store (GlueJobOperator)
```

**Why MWAA:** ML workflows with conditional logic, approval gates, and multi-service integration.

---

### Use Case 4: Event-Driven + Scheduled Hybrid

```
DAG: order_processing (triggered by S3 event + daily schedule)
├── Trigger A: New file lands in S3 (EventBridge → MWAA API)
├── Trigger B: Daily at 06:00 UTC (scheduled)
│
├── Task 1: validate_order_file
├── Task 2: enrich_with_customer_data (DynamoDB lookup)
├── Task 3: calculate_shipping (external API call)
├── Task 4: update_inventory (RDS)
├── Task 5: send_to_fulfillment (SQS)
└── Task 6: update_dashboard (Redshift)
```

**Why MWAA:** Mix of event-driven and scheduled triggers with complex business logic.

---

### Use Case 5: Infrastructure Automation

```
DAG: monthly_compliance_report
├── Task 1: scan_all_accounts (SecurityHub API)
├── Task 2: aggregate_findings (PythonOperator)
├── Task 3: generate_report (EMR job — large dataset)
├── Task 4: store_report (S3 + encrypt)
├── Task 5: send_to_compliance_team (SES email)
└── Task 6: create_jira_tickets (API call for violations)
```

---

### Use Case 6: Data Quality Monitoring Pipeline

```
DAG: hourly_data_quality (every hour)
├── Task 1: check_freshness (is data < 2 hours old?)
├── Task 2: check_completeness (are required columns populated?)
├── Task 3: check_volume (row count within expected range?)
├── Task 4: check_referential_integrity (foreign keys valid?)
├── Task 5: compute_quality_score
│   ├── If score > 95% → Task 6a: update_green_status
│   └── If score < 95% → Task 6b: alert_data_team + block downstream
└── Task 7: publish_metrics_to_cloudwatch
```

---

## 🔑 Key Features to Know

### 1. Environment Classes & Sizing

```
mw1.small: Development, < 10 DAGs, simple pipelines
├── 1 vCPU, 2 GB per scheduler/worker
├── Handles: ~20 concurrent task instances
└── Cost: ~$440/month

mw1.medium: Staging/small production, 10-50 DAGs
├── 2 vCPU, 4 GB per scheduler/worker
├── Handles: ~50 concurrent task instances
└── Cost: ~$1,400/month

mw1.large: Production, 50+ DAGs, complex pipelines
├── 4 vCPU, 8 GB per scheduler/worker
├── Handles: ~100+ concurrent task instances
└── Cost: ~$2,700-5,400/month
```

### 2. Worker Auto-Scaling

```
How it works:
├── min_workers: Always running (guaranteed capacity)
├── max_workers: Upper limit during peak
├── Scale out: When tasks queue up (Celery queue depth > threshold)
├── Scale in: When workers idle for extended period
└── Note: Scale-out takes 1-3 minutes (not instant!)

Configuration:
min_workers = 2    # Always have 2 ready
max_workers = 15   # Burst to 15 during peak DAG runs

GOTCHA: Scale-out is NOT instant!
├── If 50 tasks arrive at once, initial workers get overloaded
├── New workers take 1-3 min to provision
├── During this lag, tasks queue (not lost, just delayed)
└── Solution: Set min_workers high enough for average load
```

### 3. DAG Deployment via S3

```
S3 Bucket Structure:
s3://mwaa-dags-bucket/
├── dags/                    ← Python DAG files here
│   ├── etl_pipeline.py
│   ├── ml_pipeline.py
│   └── quality_checks.py
├── plugins/                 ← Custom Airflow plugins
│   └── plugins.zip         ← Zipped plugin package
└── requirements.txt         ← Python dependencies

DAG Sync:
├── MWAA polls S3 every 30 seconds for changes
├── New/modified DAG file → parsed by scheduler → available in UI
├── Deleted DAG file → removed from Airflow (not immediately)
└── requirements.txt change → environment update (takes 10-20 min!)
```

### 4. Airflow Configuration Overrides

```python
# Key configuration options for MWAA:
mwaa_config = {
    # Core settings
    "core.parallelism": "32",                    # Max tasks across all DAGs
    "core.max_active_tasks_per_dag": "16",       # Per-DAG concurrency
    "core.max_active_runs_per_dag": "3",         # Concurrent DAG runs
    "core.dagbag_import_timeout": "60",          # DAG parsing timeout
    
    # Scheduler settings
    "scheduler.min_file_process_interval": "60",  # Don't re-parse too often
    "scheduler.dag_dir_list_interval": "120",     # S3 sync interval
    "scheduler.zombie_task_threshold": "600",     # 10 min zombie detection
    
    # Celery settings
    "celery.worker_concurrency": "8",             # Tasks per worker
    "celery.worker_autoscale": "8,2",             # max,min per worker
    
    # Performance
    "webserver.web_server_worker_timeout": "120",  # Long-running API calls
    "scheduler.parsing_processes": "2",            # Parallel DAG parsing
}
```

### 5. Operators for AWS Services

```python
# MWAA comes with pre-installed AWS providers:

from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.providers.amazon.aws.operators.emr import (
    EmrCreateJobFlowOperator, EmrTerminateJobFlowOperator
)
from airflow.providers.amazon.aws.operators.redshift_sql import RedshiftSQLOperator
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.providers.amazon.aws.operators.athena import AthenaOperator
from airflow.providers.amazon.aws.operators.sagemaker import (
    SageMakerTrainingOperator, SageMakerModelOperator
)
from airflow.providers.amazon.aws.operators.step_function import (
    StepFunctionStartExecutionOperator
)
from airflow.providers.amazon.aws.operators.ecs import EcsRunTaskOperator
from airflow.providers.amazon.aws.operators.lambda_function import (
    LambdaInvokeFunctionOperator
)

# Sensors (wait for conditions):
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.amazon.aws.sensors.glue import GlueJobSensor
from airflow.providers.amazon.aws.sensors.emr import EmrJobFlowSensor
from airflow.providers.amazon.aws.sensors.redshift_cluster import (
    RedshiftClusterSensor
)
```

### 6. Security & Networking

```
MWAA Networking:
├── Runs in YOUR VPC (private subnets)
├── Needs: 2 private subnets in different AZs
├── NAT Gateway: Required for outbound internet (pip installs, APIs)
├── VPC Endpoints: Recommended for AWS services (S3, SQS, CloudWatch)
└── Web server: Public (internet-facing) or Private (VPC-only)

IAM:
├── Execution Role: What MWAA workers can do (assume roles, call APIs)
├── Must have: S3 access (DAG bucket), CloudWatch (logs), SQS (Celery)
├── Add: Permissions for any service your DAGs use (Glue, EMR, etc.)
└── Cross-account: Add sts:AssumeRole for target accounts

Web Server Access:
├── Public mode: Accessible via internet (with IAM auth)
├── Private mode: Accessible only from VPC (more secure)
├── Authentication: AWS IAM (web login token via CLI/API)
└── No native LDAP/SSO (use IAM Identity Center → IAM)
```

---

## ❓ Interview Questions & Answers

### Q1: When would you choose MWAA over Step Functions for workflow orchestration?

**Answer:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Choose MWAA (Airflow) when:        │ Choose Step Functions when:     │
├─────────────────────────────────────┼─────────────────────────────────┤
│ Complex DAG dependencies            │ Simple sequential/parallel flows│
│ Hundreds of tasks in a pipeline     │ < 25 steps per workflow         │
│ Need backfill (reprocess history)   │ Event-driven (trigger → run)   │
│ Python-heavy logic in tasks         │ Service integrations (native)  │
│ Team knows Airflow                  │ Serverless preferred            │
│ Cross-system orchestration          │ AWS-native only                 │
│ Schedule-based (cron) workflows     │ Event-based triggers            │
│ Need Airflow UI (visual DAG, logs)  │ Visual Workflow Studio          │
│ Dynamic DAGs (generate at runtime)  │ Static workflow definition      │
│ Long-running tasks (hours)          │ Tasks < 1 year (max timeout)   │
│ Existing Airflow investment         │ Greenfield, simple workflows   │
├─────────────────────────────────────┼─────────────────────────────────┤
│ Cost: ~$500-5,000/month (always-on) │ Cost: $0.025/1000 transitions  │
│ Idle cost: YES                      │ Idle cost: NO (pay per run)    │
│ Cold start: None (always running)   │ Cold start: ~100ms             │
│ Max DAG complexity: Unlimited       │ Max states: 25,000             │
└─────────────────────────────────────┴─────────────────────────────────┘

SWEET SPOTS:
├── MWAA: "Run 50 Glue jobs in dependency order every day with retry, 
│          backfill, monitoring, and cross-account orchestration"
└── Step Functions: "When S3 file arrives, validate → transform → notify
                    (3 steps, event-driven, pay-per-use)"
```

---

### Q2: Your MWAA environment has 30 DAGs but the scheduler keeps timing out and tasks get stuck in "queued" state. How do you diagnose and fix?

**Answer:**

```
DIAGNOSIS FRAMEWORK:

Symptom: Scheduler timeout + tasks stuck queued
├── Cause A: DAG parsing takes too long (heavy imports in DAG files)
├── Cause B: Too many tasks scheduled simultaneously (overwhelmed)
├── Cause C: Workers at capacity (all slots full)
├── Cause D: Environment too small (mw1.small for production load)
└── Cause E: Worker auto-scaling too slow (tasks queue during scale-out)
```

**Step 1: Check scheduler health**
```python
# Via Airflow REST API:
import requests
response = requests.get(
    f"{MWAA_WEBSERVER_URL}/api/v1/health",
    headers={"Authorization": f"Bearer {token}"}
)
# Look for: scheduler.latest_heartbeat (is it recent?)
# Look for: scheduler.status (healthy/unhealthy)
```

**Step 2: Check DAG parsing time**
```python
# DAG parsing issues (THE #1 cause of scheduler problems!)

# BAD: Heavy imports at module level (runs during PARSING, not execution!)
import pandas as pd                    # 2 seconds to import!
import tensorflow as tf                # 5 seconds to import!
from heavy_library import something    # Every DAG parse = import time!

# GOOD: Import inside the task function (runs only during EXECUTION)
def my_task():
    import pandas as pd  # Only imported when task actually runs
    df = pd.read_csv("s3://data/file.csv")
```

**Step 3: Apply fixes**
```python
# Fix 1: Reduce parsing overhead
mwaa_config = {
    "scheduler.min_file_process_interval": "60",    # Parse every 60s (not 30s)
    "scheduler.dag_dir_list_interval": "120",       # Scan S3 every 2 min
    "core.dagbag_import_timeout": "60",             # Kill slow DAG parsing
    "scheduler.parsing_processes": "2",             # Limit parser parallelism
}

# Fix 2: Reduce concurrency (prevent overload)
mwaa_config.update({
    "core.parallelism": "32",                       # Max 32 tasks total
    "core.max_active_tasks_per_dag": "16",          # Max 16 per DAG
    "core.max_active_runs_per_dag": "2",            # Max 2 concurrent runs
    "celery.worker_concurrency": "4",               # 4 tasks per worker
})

# Fix 3: Scale up environment
# mw1.small → mw1.large (4x more resources for scheduler)
# Increase min_workers from 2 → 5 (always have capacity ready)

# Fix 4: Use pools to limit resource-intensive tasks
# In DAG:
heavy_task = GlueJobOperator(
    task_id='heavy_etl',
    pool='glue_jobs',      # Only 5 Glue jobs can run simultaneously
    pool_slots=1,
)
# Create pool in Airflow UI: "glue_jobs" with 5 slots
```

---

### Q3: How do you implement a production-grade DAG with error handling, retries, SLA monitoring, and alerting?

**Answer:**

```python
from airflow import DAG
from airflow.decorators import task
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.amazon.aws.operators.sns import SnsPublishOperator
from airflow.utils.trigger_rule import TriggerRule
from datetime import datetime, timedelta
import boto3

# Production default_args
default_args = {
    'owner': 'data-platform-team',
    'depends_on_past': False,
    'email_on_failure': True,
    'email_on_retry': False,
    'email': ['data-alerts@company.com'],
    
    # Retry configuration
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'retry_exponential_backoff': True,
    'max_retry_delay': timedelta(minutes=60),
    
    # Timeouts
    'execution_timeout': timedelta(hours=2),
    'dagrun_timeout': timedelta(hours=4),
    
    # SLA
    'sla': timedelta(hours=3),  # Alert if DAG takes > 3 hours
    
    # Callbacks
    'on_failure_callback': notify_failure,
    'on_retry_callback': notify_retry,
    'on_success_callback': notify_success,
}

def notify_failure(context):
    """Send alert on task failure."""
    task_instance = context['task_instance']
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789:data-alerts',
        Subject=f"🚨 DAG Failed: {context['dag'].dag_id}",
        Message=f"""
Task: {task_instance.task_id}
Execution Date: {context['execution_date']}
Error: {context.get('exception', 'Unknown')}
Log URL: {task_instance.log_url}
        """
    )

def notify_retry(context):
    """Log retry attempt."""
    task_instance = context['task_instance']
    print(f"⚠️ Retrying: {task_instance.task_id}, attempt {task_instance.try_number}")

def notify_success(context):
    """Record success metrics."""
    pass  # Emit CloudWatch metric

with DAG(
    dag_id='production_data_pipeline',
    default_args=default_args,
    description='Daily data lake processing pipeline',
    schedule_interval='0 7 * * *',  # 7 AM UTC daily
    start_date=datetime(2024, 1, 1),
    catchup=False,                   # Don't backfill missed runs
    max_active_runs=1,               # Only one run at a time
    tags=['production', 'data-lake', 'tier-1'],
    doc_md="""
    ## Production Data Pipeline
    
    Processes daily data from bronze to gold layer.
    SLA: Complete within 3 hours of trigger.
    Owner: data-platform-team
    Runbook: https://wiki.company.com/runbooks/data-pipeline
    """,
) as dag:

    # Task 1: Wait for source data
    wait_for_data = S3KeySensor(
        task_id='wait_for_source_data',
        bucket_name='source-data-bucket',
        bucket_key='incoming/{{ ds }}/data_*.parquet',
        wildcard_match=True,
        timeout=3600,           # Wait max 1 hour
        poke_interval=60,       # Check every minute
        mode='reschedule',      # FREE worker slot while waiting!
        soft_fail=False,        # Fail DAG if data never arrives
    )

    # Task 2: Run Glue ETL (offload heavy work to Glue, not Airflow worker!)
    run_etl = GlueJobOperator(
        task_id='run_bronze_to_silver_etl',
        job_name='bronze-to-silver-daily',
        script_args={
            '--date': '{{ ds }}',
            '--source_path': 's3://source-data/incoming/{{ ds }}/',
            '--target_path': 's3://datalake/silver/events/',
        },
        num_of_dpus=20,
        wait_for_completion=True,  # Wait until Glue job finishes
        check_interval=60,         # Poll every 60 seconds
        pool='glue_jobs',          # Limit concurrent Glue jobs
    )

    # Task 3: Data quality check (lightweight — OK for Airflow worker)
    @task(pool='lightweight', retries=1)
    def check_data_quality(**context):
        """Validate output data meets quality standards."""
        import boto3
        
        athena = boto3.client('athena')
        
        # Check row count
        result = run_athena_query(
            f"SELECT COUNT(*) as cnt FROM silver_db.events WHERE date = '{context['ds']}'"
        )
        row_count = int(result[0]['cnt'])
        
        if row_count < 1000:
            raise ValueError(f"Row count too low: {row_count} (expected > 1000)")
        
        if row_count > 100000000:
            raise ValueError(f"Row count suspiciously high: {row_count}")
        
        # Check null rate
        null_result = run_athena_query(
            f"""SELECT 
                COUNT(*) - COUNT(user_id) as null_user_id,
                COUNT(*) as total
            FROM silver_db.events WHERE date = '{context['ds']}'"""
        )
        null_rate = int(null_result[0]['null_user_id']) / int(null_result[0]['total'])
        
        if null_rate > 0.05:
            raise ValueError(f"Null rate too high: {null_rate:.2%} (threshold: 5%)")
        
        return {"row_count": row_count, "null_rate": null_rate, "status": "PASSED"}

    quality_result = check_data_quality()

    # Task 4: Run gold aggregation (parallel with Redshift load)
    run_gold = GlueJobOperator(
        task_id='run_gold_aggregation',
        job_name='silver-to-gold-aggregation',
        script_args={'--date': '{{ ds }}'},
        num_of_dpus=10,
        wait_for_completion=True,
        pool='glue_jobs',
    )

    # Task 5: Notify success
    notify_complete = SnsPublishOperator(
        task_id='notify_pipeline_complete',
        target_arn='arn:aws:sns:us-east-1:123456789:data-pipeline-status',
        subject='✅ Daily Pipeline Complete - {{ ds }}',
        message='Pipeline completed successfully. Data available for queries.',
        trigger_rule=TriggerRule.ALL_SUCCESS,
    )

    # Task 6: Alert on any failure (always runs)
    alert_failure = SnsPublishOperator(
        task_id='alert_on_failure',
        target_arn='arn:aws:sns:us-east-1:123456789:data-alerts-critical',
        subject='🚨 Daily Pipeline FAILED - {{ ds }}',
        message='Pipeline failed. Check Airflow logs for details.',
        trigger_rule=TriggerRule.ONE_FAILED,  # Runs if ANY upstream failed
    )

    # Dependencies
    wait_for_data >> run_etl >> quality_result >> run_gold >> notify_complete
    [run_etl, quality_result, run_gold] >> alert_failure
```

---

### Q4: How does MWAA handle DAG versioning and deployment? What's the best practice for CI/CD of DAGs?

**Answer:**

```
DAG DEPLOYMENT WORKFLOW:

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Developer   │────▶│  CI Pipeline │────▶│  S3 DAG      │
│  writes DAG  │     │  (test/lint) │     │  Bucket      │
│  in Git repo │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
                                                  │
                                                  ▼ (auto-sync)
                                           ┌──────────────┐
                                           │  MWAA        │
                                           │  Environment │
                                           │  (parses DAG)│
                                           └──────────────┘
```

```yaml
# .github/workflows/deploy-dags.yml
name: Deploy Airflow DAGs
on:
  push:
    branches: [main]
    paths: ['dags/**', 'plugins/**', 'requirements.txt']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install apache-airflow==2.9.0
          pip install -r requirements.txt
          pip install pytest pylint
      
      - name: Lint DAGs
        run: pylint dags/ --disable=all --enable=E  # Only errors
      
      - name: Validate DAG integrity
        run: |
          python -c "
          import sys
          from airflow.models import DagBag
          dag_bag = DagBag(dag_folder='dags/', include_examples=False)
          if dag_bag.import_errors:
              for dag_id, error in dag_bag.import_errors.items():
                  print(f'ERROR in {dag_id}: {error}')
              sys.exit(1)
          print(f'✅ All {len(dag_bag.dags)} DAGs valid')
          "
      
      - name: Run unit tests
        run: pytest tests/dags/ -v
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Sync DAGs to S3
        run: |
          aws s3 sync dags/ s3://$MWAA_BUCKET/dags/ \
            --delete \
            --exclude "*.pyc" \
            --exclude "__pycache__/*"
      
      - name: Sync plugins
        run: |
          cd plugins && zip -r ../plugins.zip .
          aws s3 cp ../plugins.zip s3://$MWAA_BUCKET/plugins/plugins.zip
      
      - name: Sync requirements
        run: |
          aws s3 cp requirements.txt s3://$MWAA_BUCKET/requirements.txt
      
      - name: Trigger environment update (if requirements changed)
        if: contains(github.event.head_commit.modified, 'requirements.txt')
        run: |
          aws mwaa update-environment \
            --name production-airflow \
            --requirements-s3-path s3://$MWAA_BUCKET/requirements.txt
          # WARNING: This takes 10-20 min and briefly interrupts workers!
```

**Best Practices:**
```yaml
dag_deployment_best_practices:
  version_control:
    - All DAGs in Git (single source of truth)
    - requirements.txt version-pinned (no floating versions!)
    - Plugins bundled as zip, versioned in Git
    - Branch strategy: feature → staging → production
    
  testing:
    - Validate DAG parsing (no import errors)
    - Unit test task logic (mock AWS services)
    - Integration test in staging MWAA environment
    - Test backfill behavior (catchup=True scenarios)
    
  deployment:
    - S3 sync from CI/CD (never manual upload!)
    - Blue-green: Deploy to staging MWAA first, validate, then production
    - Requirements changes: Schedule during low-activity (takes 15 min!)
    - DAG changes: Instant (S3 sync → scheduler picks up in 30 sec)
    
  safety:
    - max_active_runs=1 for most production DAGs
    - catchup=False (prevent accidental massive backfills)
    - Tags on DAGs (for filtering, team ownership)
    - Pools for resource-intensive tasks (prevent stampede)
```

---

### Q5: How do you handle cross-account orchestration in MWAA? A DAG needs to trigger Glue jobs in 5 different accounts.

**Answer:**

```python
from airflow import DAG
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.providers.amazon.aws.hooks.base_aws import AwsBaseHook
from airflow.decorators import task
from datetime import datetime
import boto3

# Method 1: Using Airflow AWS Connection with role assumption
# Create connections in Airflow UI or via environment variables:
# Connection ID: "aws_account_a" → Extra: {"role_arn": "arn:aws:iam::ACCOUNT_A:role/AirflowCrossAccount"}
# Connection ID: "aws_account_b" → Extra: {"role_arn": "arn:aws:iam::ACCOUNT_B:role/AirflowCrossAccount"}

with DAG('cross_account_pipeline', schedule_interval='@daily', 
         start_date=datetime(2024, 1, 1), catchup=False) as dag:

    # GlueJobOperator with explicit AWS connection (assumes role automatically)
    glue_account_a = GlueJobOperator(
        task_id='etl_account_a',
        job_name='daily-etl',
        aws_conn_id='aws_account_a',  # Uses role from this connection
        region_name='us-east-1',
        script_args={'--date': '{{ ds }}'},
        wait_for_completion=True,
    )

    glue_account_b = GlueJobOperator(
        task_id='etl_account_b',
        job_name='daily-etl',
        aws_conn_id='aws_account_b',  # Different account
        region_name='us-east-1',
        script_args={'--date': '{{ ds }}'},
        wait_for_completion=True,
    )

# Method 2: Manual role assumption in PythonOperator (more control)
@task
def trigger_glue_in_account(account_id: str, role_name: str, job_name: str, **context):
    """Assume role in target account and trigger Glue job."""
    sts = boto3.client('sts')
    
    # Assume role in target account
    assumed = sts.assume_role(
        RoleArn=f'arn:aws:iam::{account_id}:role/{role_name}',
        RoleSessionName=f'airflow-{context["dag"].dag_id}',
        DurationSeconds=3600
    )
    
    # Create Glue client with assumed credentials
    glue = boto3.client('glue',
        aws_access_key_id=assumed['Credentials']['AccessKeyId'],
        aws_secret_access_key=assumed['Credentials']['SecretAccessKey'],
        aws_session_token=assumed['Credentials']['SessionToken'],
        region_name='us-east-1'
    )
    
    # Start Glue job
    response = glue.start_job_run(
        JobName=job_name,
        Arguments={'--date': context['ds']}
    )
    
    # Wait for completion
    run_id = response['JobRunId']
    while True:
        status = glue.get_job_run(JobName=job_name, RunId=run_id)
        state = status['JobRun']['JobRunState']
        if state == 'SUCCEEDED':
            return {'run_id': run_id, 'state': state}
        elif state in ('FAILED', 'TIMEOUT', 'STOPPED'):
            raise Exception(f"Glue job failed: {state}")
        time.sleep(60)

# Dynamic: Run across all 5 accounts
accounts = [
    {"account_id": "111111111111", "role": "AirflowRole", "job": "etl-team-a"},
    {"account_id": "222222222222", "role": "AirflowRole", "job": "etl-team-b"},
    {"account_id": "333333333333", "role": "AirflowRole", "job": "etl-team-c"},
    {"account_id": "444444444444", "role": "AirflowRole", "job": "etl-team-d"},
    {"account_id": "555555555555", "role": "AirflowRole", "job": "etl-team-e"},
]

# Parallel execution across all accounts
tasks = [
    trigger_glue_in_account.override(task_id=f"glue_{acct['account_id']}")(
        account_id=acct['account_id'],
        role_name=acct['role'],
        job_name=acct['job']
    )
    for acct in accounts
]
```

**IAM Setup (in each target account):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::MWAA_ACCOUNT:role/MWAAExecutionRole"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "airflow-cross-account"
      }
    }
  }]
}
```

---

### Q6: Your DAG has 200 tasks that all need to run. Only 10 can run simultaneously due to API rate limits on the target service. How do you implement rate limiting in Airflow?

**Answer:**

```python
# Solution 1: POOLS (Recommended — built-in Airflow feature)

# Create pool via CLI or API:
# airflow pools set api_rate_limited 10 "Limit to 10 concurrent API calls"

# In your DAG:
from airflow import DAG
from airflow.decorators import task
from datetime import datetime

with DAG('rate_limited_dag', schedule_interval='@hourly', 
         start_date=datetime(2024, 1, 1), catchup=False) as dag:

    @task(pool='api_rate_limited', pool_slots=1)  # Each task takes 1 slot
    def call_external_api(item_id):
        """Each task instance takes 1 pool slot."""
        import requests
        response = requests.post(f"https://api.service.com/process/{item_id}")
        return response.json()

    # Dynamic Task Mapping: 200 tasks, but only 10 run at a time!
    items = list(range(200))
    call_external_api.expand(item_id=items)
    # Airflow queues all 200, but pool limits to 10 concurrent

# Solution 2: Semaphore with task concurrency
# (Per-DAG limit, not global)
with DAG('rate_limited_v2', 
         max_active_tasks=10,  # Only 10 tasks active at once in this DAG
         ...) as dag:
    pass

# Solution 3: Custom throttling in code
@task(pool='api_rate_limited', pool_slots=1)
def call_api_with_backoff(item_id):
    """Rate limiting within the task itself."""
    import time
    import random
    from tenacity import retry, stop_after_attempt, wait_exponential
    
    @retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, max=30))
    def api_call():
        response = requests.post(f"https://api.service.com/process/{item_id}")
        if response.status_code == 429:  # Rate limited
            raise RateLimitException("Rate limited, will retry")
        return response.json()
    
    # Add jitter to prevent thundering herd
    time.sleep(random.uniform(0.1, 1.0))
    return api_call()
```

---

### Q7: How do you implement backfill and data reprocessing in MWAA?

**Answer:**

```python
# BACKFILL = Reprocess historical data for past dates

# Method 1: Airflow CLI (via MWAA CLI endpoint)
# Backfill last 30 days:
# airflow dags backfill daily_pipeline -s 2024-05-15 -e 2024-06-15

# Method 2: Via MWAA API (programmatic)
import boto3
import base64
import json

mwaa = boto3.client('mwaa')

# Get CLI token
token_response = mwaa.create_cli_token(Name='production-airflow')
web_server = token_response['WebServerHostname']
cli_token = token_response['CliToken']

# Execute backfill command
import requests
response = requests.post(
    f"https://{web_server}/aws_mwaa/cli",
    headers={"Authorization": f"Bearer {cli_token}", "Content-Type": "text/plain"},
    data="dags backfill daily_pipeline -s 2024-05-15 -e 2024-06-15 --reset-dagruns"
)
result = base64.b64decode(response.json()['stdout']).decode()
print(result)

# Method 3: DAG designed for safe backfill
with DAG(
    'backfill_safe_pipeline',
    schedule_interval='@daily',
    start_date=datetime(2024, 1, 1),
    catchup=True,    # ENABLE for backfill-capable DAGs
    max_active_runs=3,  # Limit parallel backfill runs
) as dag:
    
    @task
    def process_date(**context):
        """Each run processes exactly ONE date — idempotent!"""
        processing_date = context['ds']  # The logical date
        
        # Use processing_date (not current date!) for data selection
        input_path = f"s3://datalake/bronze/events/date={processing_date}/"
        output_path = f"s3://datalake/silver/events/date={processing_date}/"
        
        # OVERWRITE mode makes it safe to rerun (idempotent)
        process_data(input_path, output_path, mode='overwrite')
        
        return {"date": processing_date, "status": "completed"}
```

**Backfill best practices:**
```yaml
backfill_best_practices:
  design:
    - DAG tasks must be IDEMPOTENT (safe to rerun)
    - Use execution_date ({{ ds }}) not current date
    - Write in OVERWRITE mode (not append — prevents duplicates)
    - Set max_active_runs to limit parallel backfills
    
  execution:
    - Backfill during off-peak hours (workers available)
    - Start with small range (test), then expand
    - Monitor: Don't overwhelm downstream systems
    - Use --reset-dagruns to clear previous failed attempts
    
  gotchas:
    - catchup=True required for backfill to work
    - depends_on_past=True blocks parallel backfill (be careful!)
    - Large backfills (years) may overwhelm scheduler
    - Solution: Backfill in weekly chunks, not all at once
```

---

### Q8: What's the best pattern for "sensors" in MWAA? How do you wait for data without wasting worker slots?

**Answer:**

```python
# PROBLEM: Sensors occupy a worker slot while waiting
# If you have 20 sensors waiting for data + only 10 workers:
# → No workers left for actual processing tasks!

# SOLUTION: Use mode='reschedule' (releases worker slot while waiting)

from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

# BAD: mode='poke' (DEFAULT) — holds worker slot for entire wait time!
bad_sensor = S3KeySensor(
    task_id='wait_for_file_BAD',
    bucket_name='data-bucket',
    bucket_key='incoming/{{ ds }}/data.parquet',
    mode='poke',              # Worker slot BLOCKED entire time!
    poke_interval=60,         # Checks every 60 sec
    timeout=7200,             # Waits up to 2 hours
    # This task occupies 1 worker slot for UP TO 2 HOURS doing nothing!
)

# GOOD: mode='reschedule' — releases worker between checks
good_sensor = S3KeySensor(
    task_id='wait_for_file_GOOD',
    bucket_name='data-bucket',
    bucket_key='incoming/{{ ds }}/data.parquet',
    mode='reschedule',        # Releases worker between pokes!
    poke_interval=300,        # Check every 5 min (less frequent for reschedule)
    timeout=7200,             # Still waits up to 2 hours
    # Worker only used for ~1 second per poke (5 min intervals)
    # Slot available for other tasks between pokes!
)

# BEST: Use Deferrable Operators (Airflow 2.6+) — zero worker usage!
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

best_sensor = S3KeySensor(
    task_id='wait_for_file_BEST',
    bucket_name='data-bucket',
    bucket_key='incoming/{{ ds }}/data.parquet',
    deferrable=True,          # Uses triggerer (not worker!)
    poke_interval=60,
    timeout=7200,
    # ZERO worker slots used! Runs on separate lightweight triggerer process.
)
```

```
COMPARISON:

mode='poke':        1 worker slot used for ENTIRE wait time
mode='reschedule':  1 worker slot used for ~1 sec every poke_interval
deferrable=True:    0 worker slots used (uses triggerer, separate process)

Resource Impact:
├── 20 sensors × poke mode × 2 hours = 20 worker slots wasted!
├── 20 sensors × reschedule mode = ~0.1 worker slots (negligible)
└── 20 sensors × deferrable = 0 worker slots (free!)

RULE: ALWAYS use mode='reschedule' or deferrable=True for production sensors
      NEVER use mode='poke' in production (wastes workers)
```

---

### Q9: How do you monitor MWAA health and set up alerting for DAG failures?

**Answer:**

```python
# MWAA Monitoring Stack:

# Layer 1: CloudWatch Metrics (automatic)
# MWAA publishes these automatically:
# ├── QueuedTasks: Tasks waiting for workers
# ├── RunningTasks: Currently executing tasks
# ├── SchedulerHeartbeat: Is scheduler alive?
# └── DagProcessingImportErrors: DAGs failing to parse

# Layer 2: CloudWatch Alarms
```

```hcl
# Terraform: Critical MWAA alarms
resource "aws_cloudwatch_metric_alarm" "scheduler_heartbeat" {
  alarm_name          = "mwaa-scheduler-heartbeat-missing"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 3
  metric_name         = "SchedulerHeartbeat"
  namespace           = "AWS/MWAA"
  period              = 60
  statistic           = "Sum"
  threshold           = 1
  alarm_description   = "MWAA Scheduler not sending heartbeats!"
  alarm_actions       = [aws_sns_topic.critical_alerts.arn]
  
  dimensions = {
    Environment = "production-airflow"
  }
}

resource "aws_cloudwatch_metric_alarm" "tasks_queued_high" {
  alarm_name          = "mwaa-tasks-queued-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5
  metric_name         = "QueuedTasks"
  namespace           = "AWS/MWAA"
  period              = 60
  statistic           = "Average"
  threshold           = 50  # More than 50 tasks queued = workers overwhelmed
  alarm_description   = "Too many tasks queued — workers may be saturated"
  alarm_actions       = [aws_sns_topic.warning_alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "dag_import_errors" {
  alarm_name          = "mwaa-dag-import-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "DagProcessingImportErrors"
  namespace           = "AWS/MWAA"
  period              = 300
  statistic           = "Maximum"
  threshold           = 0  # Any import error is a problem!
  alarm_description   = "DAG parsing error — broken DAG deployed!"
  alarm_actions       = [aws_sns_topic.critical_alerts.arn]
}
```

```python
# Layer 3: In-DAG monitoring (SLA and failure callbacks)

# SLA Miss callback (alert if DAG is too slow)
def sla_miss_callback(dag, task_list, blocking_task_list, slas, blocking_tis):
    """Called when a task exceeds its SLA."""
    sns = boto3.client('sns')
    sns.publish(
        TopicArn=ALERT_TOPIC,
        Subject=f"⏰ SLA Miss: DAG {dag.dag_id}",
        Message=f"Tasks exceeded SLA: {[t.task_id for t in task_list]}"
    )

with DAG('monitored_dag', sla_miss_callback=sla_miss_callback, ...):
    pass

# Layer 4: External monitoring DAG (monitors other DAGs!)
with DAG('meta_monitoring', schedule_interval='*/5 * * * *', ...):
    
    @task
    def check_critical_dags():
        """Verify critical DAGs are running on schedule."""
        from airflow.models import DagRun
        from airflow.utils.state import State
        
        critical_dags = ['daily_etl', 'hourly_quality', 'ml_pipeline']
        
        for dag_id in critical_dags:
            latest_run = DagRun.find(dag_id=dag_id, state=State.FAILED)
            if latest_run:
                alert(f"DAG {dag_id} has recent failures!")
            
            # Check if DAG hasn't run when expected
            last_success = get_last_success(dag_id)
            if last_success and (datetime.now() - last_success) > timedelta(hours=2):
                alert(f"DAG {dag_id} hasn't run successfully in 2+ hours!")
```

---

### Q10: Compare MWAA with self-managed Airflow on EKS. When would you choose each?

**Answer:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Feature              │ MWAA (Managed)        │ Airflow on EKS (Self)   │
├───────────────────────┼───────────────────────┼─────────────────────────┤
│ Infrastructure mgmt   │ AWS handles all       │ You manage K8s + Airflow│
│ Scaling               │ 1-25 workers (auto)   │ Unlimited (K8s pods)    │
│ Executor              │ Celery only           │ Any (K8s, Celery, Local)│
│ Max workers           │ 25                    │ Unlimited (K8s)         │
│ Customization         │ Limited (overrides)   │ Full control            │
│ Airflow version       │ AWS-supported (lag)   │ Any version (immediate) │
│ Plugins               │ Limited (zip upload)  │ Full Docker image       │
│ KubernetesExecutor    │ NOT available         │ YES (pod per task!)     │
│ Startup time          │ 10-25 min (new env)   │ Depends on Helm deploy  │
│ Cost (small)          │ ~$500/month minimum   │ ~$200/month (shared EKS)│
│ Cost (large)          │ ~$5,000/month         │ Varies (pods + infra)   │
│ Operational burden    │ LOW                   │ HIGH                    │
│ Multi-tenant          │ Separate environments │ Namespaces + RBAC       │
│ DAG isolation         │ Shared workers        │ Pod per task (isolated) │
│ Internet access       │ NAT Gateway required  │ Standard K8s networking │
│ Requirements install  │ 10-20 min (env update)│ Seconds (Docker image)  │
│ Git-sync              │ Not native (S3-based) │ Native (git-sync sidecar│
│ Secrets               │ Env vars or SM        │ K8s secrets, Vault, etc │
├───────────────────────┼───────────────────────┼─────────────────────────┤
│ CHOOSE MWAA when:     │                       │ CHOOSE Self-hosted when:│
│ - Small/medium scale  │                       │ - Need > 25 workers     │
│ - Don't want K8s ops  │                       │ - Need KubernetesExec   │
│ - < 100 DAGs          │                       │ - Need latest Airflow   │
│ - Standard AWS integr │                       │ - Need custom Docker    │
│ - Quick to start      │                       │ - Multi-tenant (teams)  │
│ - Team < 5 engineers  │                       │ - Already have EKS      │
│                       │                       │ - Need full flexibility │
└───────────────────────┴───────────────────────┴─────────────────────────┘

DECISION SIMPLIFIED:
├── "We just need Airflow to work, small-medium scale" → MWAA
├── "We need 100+ concurrent tasks, custom executors, latest features" → Self-hosted
└── "We have EKS already and want pod-per-task isolation" → Self-hosted on EKS
```

---

## 🆚 MWAA vs Competitors

| Feature | MWAA | Astronomer (Astro) | GCP Composer | Self-hosted (EKS) | Step Functions |
|---------|------|-------------------|--------------|-------------------|----------------|
| Engine | Airflow | Airflow | Airflow | Airflow | AWS-native |
| Managed | Partially | Fully | Fully | No | Fully |
| Executor | Celery | K8s/Celery | K8s | Any | N/A (states) |
| Max scale | 25 workers | Unlimited | Auto | Unlimited | Unlimited |
| Cost model | Per-hour | Per-deployment | Per-hour | Infrastructure | Per-transition |
| Version lag | 1-3 months | Latest | 1-3 months | Latest | N/A |
| Git-sync | No (S3) | Yes | Yes | Yes | N/A |
| K8s Executor | No | Yes | Yes | Yes | N/A |
| Serverless | No | Hybrid | No | No | Yes |
| Best for | AWS-native, simple | Enterprise Airflow | GCP ecosystem | Full control | Event-driven |

---

## 🏆 Production Best Practices

```yaml
environment_design:
  - Production: mw1.large (don't under-size scheduler!)
  - Development: mw1.small (cost-effective for testing)
  - min_workers: Set to average load (not minimum!)
  - max_workers: Set to peak + 20% headroom
  - Always 2 schedulers (HA — MWAA default)
  - Private web server access for production

dag_development:
  - NEVER import heavy libraries at module level (kills parser!)
  - Use @task decorator (TaskFlow API) for Python tasks
  - Offload heavy work to Glue/EMR/ECS (not Airflow workers!)
  - Use mode='reschedule' or deferrable=True for all sensors
  - Set execution_timeout on EVERY task (prevent runaway)
  - Use pools to limit concurrent access to shared resources
  - catchup=False for most DAGs (prevent accidental backfills)
  - max_active_runs=1 for most production DAGs

deployment:
  - CI/CD pipeline for DAG deployment (lint → test → S3 sync)
  - requirements.txt changes are EXPENSIVE (10-20 min env update!)
  - Bundle infrequent dependencies in plugins.zip instead
  - Test DAGs locally before deploying (airflow db init + parse)
  - Version pin ALL dependencies (no floating versions!)

monitoring:
  - Alert on: SchedulerHeartbeat missing, QueuedTasks high, import errors
  - SLA monitoring on all tier-1 DAGs
  - Failure callbacks for immediate notification
  - Weekly review: task duration trends, failure rates, queue wait times

security:
  - Private web server (VPC-only access)
  - Minimal execution role (only what DAGs actually need)
  - VPC Endpoints for S3, SQS, CloudWatch, Glue, etc.
  - No internet access for workers (use VPC endpoints)
  - Secrets in AWS Secrets Manager (not Airflow variables!)

cost_optimization:
  - Right-size environment class (don't default to large)
  - min_workers based on ACTUAL load (monitor before setting)
  - Use reschedule/deferrable sensors (free up workers)
  - Offload compute to Glue/EMR (cheaper than fat MWAA workers)
  - Consolidate small DAGs (reduce scheduler overhead)
```
