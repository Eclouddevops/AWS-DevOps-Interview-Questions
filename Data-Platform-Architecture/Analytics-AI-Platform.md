# Analytics & AI Platform - Redshift, Athena, MWAA, SageMaker

## Amazon Redshift Architecture

### Redshift Serverless vs Provisioned Decision:

| Factor | Serverless | Provisioned |
|--------|-----------|-------------|
| Workload | Variable/unpredictable | Steady/predictable |
| Cost Model | Pay per RPU-hour used | Pay per node-hour (always on) |
| Scaling | Automatic (seconds) | Manual/scheduled (minutes) |
| Concurrency | Auto-scales | WLM queues + concurrency scaling |
| Best For | Dev/test, ad-hoc analytics | Production BI, predictable loads |
| Data Sharing | ✅ Consumer | ✅ Producer & Consumer |
| Max Storage | Managed | RA3: up to 16PB |

### Redshift Spectrum (Query S3 directly):

```sql
-- Create external schema pointing to Glue Catalog
CREATE EXTERNAL SCHEMA datalake
FROM DATA CATALOG
DATABASE 'silver_db'
IAM_ROLE 'arn:aws:iam::123456789:role/RedshiftSpectrumRole'
REGION 'us-east-1';

-- Query S3 data directly (no loading required)
-- Pushes filters to S3 layer for performance
SELECT 
    customer_id,
    SUM(amount) as total_spend,
    COUNT(*) as order_count
FROM datalake.orders  -- This is in S3 (Parquet)
WHERE year = 2024 AND month = 6
GROUP BY customer_id
HAVING SUM(amount) > 10000
ORDER BY total_spend DESC
LIMIT 100;

-- JOIN local Redshift tables with S3 data
SELECT 
    c.customer_name,
    c.tier,
    s.total_spend,
    s.order_count
FROM local_schema.customers c  -- In Redshift storage
JOIN (
    SELECT customer_id, SUM(amount) as total_spend, COUNT(*) as order_count
    FROM datalake.orders  -- In S3
    WHERE year = 2024
    GROUP BY customer_id
) s ON c.customer_id = s.customer_id
WHERE c.tier = 'premium';
```

### Redshift Materialized Views (Auto-Refresh):

```sql
-- Auto-refreshing materialized view for dashboards
CREATE MATERIALIZED VIEW mv_daily_revenue
AUTO REFRESH YES  -- Incrementally refreshes when base tables change
AS
SELECT 
    DATE_TRUNC('day', order_time) as order_date,
    product_category,
    region,
    COUNT(*) as order_count,
    SUM(amount) as revenue,
    AVG(amount) as avg_order_value,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY amount) as p95_order_value
FROM orders
WHERE order_time >= DATEADD(month, -6, GETDATE())
GROUP BY 1, 2, 3;

-- Create late-binding view for Spectrum tables (schema can change)
CREATE VIEW v_combined_metrics
AS
SELECT * FROM mv_daily_revenue
UNION ALL
SELECT * FROM datalake.historical_revenue  -- S3 data
WITH NO SCHEMA BINDING;
```

## Amazon Athena Advanced Patterns

### Athena Workgroups for Cost Control:

```hcl
resource "aws_athena_workgroup" "analysts" {
  name = "analysts"
  
  configuration {
    enforce_workgroup_configuration = true
    publish_cloudwatch_metrics_enabled = true
    bytes_scanned_cutoff_per_query = 10737418240  # 10GB limit per query
    
    result_configuration {
      output_location = "s3://athena-results/analysts/"
      
      encryption_configuration {
        encryption_option = "SSE_KMS"
        kms_key_arn       = aws_kms_key.athena.arn
      }
    }
    
    engine_version {
      selected_engine_version = "Athena engine version 3"
    }
  }
}

resource "aws_athena_workgroup" "bi_dashboards" {
  name = "bi-dashboards"
  
  configuration {
    enforce_workgroup_configuration = true
    publish_cloudwatch_metrics_enabled = true
    # No byte limit for BI (but monitor via CloudWatch)
    
    result_configuration {
      output_location = "s3://athena-results/bi/"
    }
  }
}
```

### Athena Federated Queries:

```sql
-- Query across multiple data sources using Athena connectors
-- DynamoDB connector
SELECT 
    d.user_id,
    d.session_data,
    s.total_purchases
FROM 
    "lambda:dynamodb-connector".default.user_sessions d
JOIN 
    silver_db.purchase_summary s ON d.user_id = s.user_id
WHERE 
    d.last_active > CURRENT_TIMESTAMP - INTERVAL '24' HOUR;

-- RDS (MySQL) connector - join operational DB with data lake
SELECT 
    r.product_name,
    r.current_stock,
    l.weekly_sales_velocity,
    r.current_stock / NULLIF(l.weekly_sales_velocity, 0) as weeks_of_stock
FROM 
    "lambda:rds-connector".inventory.products r
JOIN 
    gold_db.sales_velocity l ON r.product_id = l.product_id
WHERE 
    r.current_stock / NULLIF(l.weekly_sales_velocity, 0) < 2
ORDER BY weeks_of_stock ASC;
```

## MWAA (Managed Airflow) Production Patterns

### DAG Design Patterns:

```python
# Pattern 1: Data Pipeline with Quality Gates
from airflow import DAG
from airflow.decorators import task, task_group
from airflow.providers.amazon.aws.operators.glue import GlueJobOperator
from airflow.providers.amazon.aws.operators.athena import AthenaOperator
from airflow.providers.amazon.aws.sensors.glue import GlueJobSensor
from airflow.operators.python import BranchPythonOperator
from airflow.utils.trigger_rule import TriggerRule
from datetime import datetime, timedelta
import boto3

default_args = {
    'owner': 'data-platform',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=3),
}

with DAG(
    dag_id='silver_layer_pipeline',
    default_args=default_args,
    schedule_interval='0 */4 * * *',  # Every 4 hours
    start_date=datetime(2024, 1, 1),
    catchup=False,
    max_active_runs=1,
    tags=['production', 'silver-layer'],
    doc_md="""
    ## Silver Layer Pipeline
    Processes raw bronze data into cleaned silver tables.
    Includes data quality checks with automatic alerting.
    """
) as dag:

    @task(pool='lightweight')
    def check_source_data_available(**context):
        """Verify new data exists before processing."""
        import boto3
        s3 = boto3.client('s3')
        execution_date = context['execution_date']
        prefix = f"bronze/events/year={execution_date.year}/month={execution_date.month:02d}/day={execution_date.day:02d}/"
        response = s3.list_objects_v2(Bucket='datalake-bronze', Prefix=prefix, MaxKeys=1)
        if response.get('KeyCount', 0) == 0:
            raise ValueError(f"No source data for {prefix}")
        return True

    transform_job = GlueJobOperator(
        task_id='transform_to_silver',
        job_name='bronze-to-silver-etl',
        script_args={
            '--date': '{{ ds }}',
            '--source_database': 'bronze_db',
            '--target_path': 's3://datalake-silver/events/'
        },
        num_of_dpus=20,
        wait_for_completion=True,
        pool='glue_jobs'
    )

    @task(pool='lightweight')
    def run_quality_checks(**context):
        """Run DQ checks on transformed data."""
        glue = boto3.client('glue')
        response = glue.start_data_quality_ruleset_evaluation_run(
            DataSource={
                'GlueTable': {
                    'DatabaseName': 'silver_db',
                    'TableName': 'events'
                }
            },
            Role='GlueDataQualityRole',
            RulesetNames=['silver_events_quality_rules'],
            AdditionalRunOptions={
                'CloudWatchMetricsEnabled': True
            }
        )
        return response['RunId']

    @task.branch(pool='lightweight')
    def evaluate_quality_results(run_id: str):
        """Branch based on quality results."""
        glue = boto3.client('glue')
        # Wait for completion and check results
        import time
        for _ in range(60):
            response = glue.get_data_quality_ruleset_evaluation_run(RunId=run_id)
            if response['Status'] in ['SUCCEEDED', 'FAILED']:
                break
            time.sleep(10)
        
        if response['Status'] == 'SUCCEEDED':
            score = response.get('Score', 0)
            if score >= 0.95:
                return 'publish_to_catalog'
            else:
                return 'alert_quality_degradation'
        return 'alert_quality_failure'

    @task(pool='lightweight')
    def publish_to_catalog():
        """Update Glue catalog and notify consumers."""
        glue = boto3.client('glue')
        glue.update_table(
            DatabaseName='silver_db',
            TableInput={
                'Name': 'events',
                'Parameters': {
                    'last_successful_update': datetime.now().isoformat(),
                    'quality_score': '0.98'
                }
            }
        )
        return "Published successfully"

    @task(trigger_rule=TriggerRule.ONE_SUCCESS, pool='lightweight')
    def alert_quality_degradation():
        """Alert but don't block - quality degraded but acceptable."""
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789:data-quality-alerts',
            Subject='⚠️ Data Quality Degradation - Silver Events',
            Message='Quality score below threshold. Review required.'
        )

    @task(trigger_rule=TriggerRule.ONE_SUCCESS, pool='lightweight')
    def alert_quality_failure():
        """Alert and block - quality check failed."""
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789:data-quality-critical',
            Subject='🚨 Data Quality FAILURE - Silver Events',
            Message='Quality check failed. Pipeline blocked. Immediate review required.'
        )

    # DAG Flow
    source_check = check_source_data_available()
    quality_run_id = run_quality_checks()
    quality_branch = evaluate_quality_results(quality_run_id)
    
    source_check >> transform_job >> quality_run_id >> quality_branch
    quality_branch >> [publish_to_catalog(), alert_quality_degradation(), alert_quality_failure()]
```

### MWAA Cross-Account Pattern:

```python
# Pattern: DAG triggers pipelines in multiple accounts
from airflow.providers.amazon.aws.hooks.base_aws import AwsBaseHook

@task
def trigger_cross_account_glue_job(account_id: str, job_name: str, **context):
    """Assume role in target account and trigger Glue job."""
    import boto3
    
    sts = boto3.client('sts')
    assumed = sts.assume_role(
        RoleArn=f'arn:aws:iam::{account_id}:role/CrossAccountGlueRole',
        RoleSessionName='airflow-cross-account',
        DurationSeconds=3600
    )
    
    glue = boto3.client('glue',
        aws_access_key_id=assumed['Credentials']['AccessKeyId'],
        aws_secret_access_key=assumed['Credentials']['SecretAccessKey'],
        aws_session_token=assumed['Credentials']['SessionToken']
    )
    
    response = glue.start_job_run(
        JobName=job_name,
        Arguments={
            '--date': context['ds'],
            '--source_account': context['var']['value']['current_account_id']
        }
    )
    return response['JobRunId']
```

## SageMaker Integration with Data Platform

### Feature Store Pattern:

```python
import sagemaker
from sagemaker.feature_store.feature_group import FeatureGroup

session = sagemaker.Session()

# Create feature group (online + offline store)
customer_feature_group = FeatureGroup(
    name='customer-features',
    sagemaker_session=session
)

customer_feature_group.load_feature_definitions(
    data_frame=customer_features_df  # Pandas DataFrame with features
)

customer_feature_group.create(
    s3_uri='s3://feature-store/customer-features/',
    record_identifier_name='customer_id',
    event_time_feature_name='event_time',
    role_arn=role,
    enable_online_store=True,  # DynamoDB for real-time inference
    offline_store_config=sagemaker.feature_store.OfflineStoreConfig(
        s3_storage_config=sagemaker.feature_store.S3StorageConfig(
            s3_uri='s3://feature-store/offline/customer-features/'
        ),
        data_catalog_config=sagemaker.feature_store.DataCatalogConfig(
            table_name='customer_features',
            catalog='AwsDataCatalog',
            database='feature_store_db'
        )
    )
)

# Ingest features from Glue/EMR pipeline
customer_feature_group.ingest(data_frame=new_features_df, max_workers=5, wait=True)

# Real-time feature retrieval (for inference)
featurestore_runtime = boto3.client('sagemaker-featurestore-runtime')
record = featurestore_runtime.get_record(
    FeatureGroupName='customer-features',
    RecordIdentifierValueAsString='customer-123'
)
```

### ML Pipeline Integration:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│ Data Lake    │────▶│ Feature      │────▶│ SageMaker        │
│ (Glue ETL)  │     │ Engineering  │     │ Training         │
│              │     │ (Glue/EMR)   │     │                  │
└──────────────┘     └──────────────┘     └──────────────────┘
                            │                       │
                            ▼                       ▼
                     ┌──────────────┐     ┌──────────────────┐
                     │ Feature      │     │ Model Registry   │
                     │ Store        │     │ (Versioned)      │
                     │ (Online/     │     │                  │
                     │  Offline)    │     └────────┬─────────┘
                     └──────┬───────┘              │
                            │                      ▼
                            │              ┌──────────────────┐
                            └─────────────▶│ Real-time        │
                                           │ Inference        │
                                           │ (SageMaker       │
                                           │  Endpoint)       │
                                           └──────────────────┘
```

## Data Observability & Monitoring

### CloudWatch Metrics for Data Platform:

```hcl
# Custom CloudWatch dashboard for data platform
resource "aws_cloudwatch_dashboard" "data_platform" {
  dashboard_name = "DataPlatform-Operations"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "Glue Job Success Rate"
          metrics = [
            ["Glue", "glue.driver.aggregate.numCompletedTasks", "JobName", "bronze-to-silver"],
            ["Glue", "glue.driver.aggregate.numFailedTasks", "JobName", "bronze-to-silver"]
          ]
          period = 300
          stat   = "Sum"
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title   = "MSK Consumer Lag"
          metrics = [
            ["AWS/Kafka", "SumOffsetLag", "Cluster Name", "production", "Consumer Group", "flink-processor"],
            ["AWS/Kafka", "SumOffsetLag", "Cluster Name", "production", "Consumer Group", "s3-sink"]
          ]
          period = 60
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          title   = "Data Quality Score"
          metrics = [
            ["Glue", "dq.rule.pass_percentage", "Database", "silver_db", "Table", "events"]
          ]
          period = 3600
          stat   = "Average"
        }
      }
    ]
  })
}
```

### Data SLAs:
```yaml
# Data Platform SLAs
data_products:
  - name: customer_events_silver
    freshness_sla: 1 hour
    completeness_sla: 99%
    availability_sla: 99.9%
    quality_score_minimum: 0.95
    
  - name: daily_revenue_gold
    freshness_sla: 4 hours
    completeness_sla: 99.9%
    availability_sla: 99.95%
    quality_score_minimum: 0.99
    
  - name: real_time_fraud_scores
    latency_sla_p99: 500ms
    availability_sla: 99.99%
    throughput_sla: 1M events/sec
```
