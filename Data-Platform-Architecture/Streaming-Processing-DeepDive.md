# Streaming & Processing Deep Dive - MSK, Kinesis, EMR, Glue

## Amazon MSK (Managed Kafka) Architecture

### Production Cluster Design:

```
┌─────────────────────────────────────────────────────────────────┐
│                    MSK Cluster (Production)                       │
│                                                                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │ Broker 1 │    │ Broker 2 │    │ Broker 3 │   AZ-a            │
│  │ m5.4xl   │    │ m5.4xl   │    │ m5.4xl   │                   │
│  └──────────┘    └──────────┘    └──────────┘                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │ Broker 4 │    │ Broker 5 │    │ Broker 6 │   AZ-b            │
│  │ m5.4xl   │    │ m5.4xl   │    │ m5.4xl   │                   │
│  └──────────┘    └──────────┘    └──────────┘                   │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                   │
│  │ Broker 7 │    │ Broker 8 │    │ Broker 9 │   AZ-c            │
│  │ m5.4xl   │    │ m5.4xl   │    │ m5.4xl   │                   │
│  └──────────┘    └──────────┘    └──────────┘                   │
│                                                                   │
│  ZooKeeper: Managed (3 nodes, multi-AZ)                          │
│  Storage: 2TB EBS gp3 per broker (auto-expand)                   │
│  Authentication: IAM + SASL/SCRAM                                │
│  Encryption: TLS in-transit + KMS at-rest                        │
└─────────────────────────────────────────────────────────────────┘
```

### MSK Configuration:
```hcl
resource "aws_msk_cluster" "production" {
  cluster_name           = "data-platform-production"
  kafka_version          = "3.6.0"
  number_of_broker_nodes = 9  # 3 per AZ

  broker_node_group_info {
    instance_type   = "kafka.m5.4xlarge"
    client_subnets  = var.private_subnet_ids  # 3 AZs
    security_groups = [aws_security_group.msk.id]
    
    storage_info {
      ebs_storage_info {
        volume_size = 2000  # 2TB per broker
        provisioned_throughput {
          enabled           = true
          volume_throughput  = 250  # MB/s
        }
      }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"
      in_cluster    = true
    }
    encryption_at_rest_kms_key_arn = aws_kms_key.msk.arn
  }

  client_authentication {
    sasl {
      iam   = true
      scram = true
    }
  }

  configuration_info {
    arn      = aws_msk_configuration.production.arn
    revision = aws_msk_configuration.production.latest_revision
  }

  open_monitoring {
    prometheus {
      jmx_exporter {
        enabled_in_broker = true
      }
      node_exporter {
        enabled_in_broker = true
      }
    }
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk.name
      }
      s3 {
        enabled = true
        bucket  = aws_s3_bucket.msk_logs.id
        prefix  = "msk/broker-logs"
      }
    }
  }
}

resource "aws_msk_configuration" "production" {
  name              = "production-config"
  kafka_versions    = ["3.6.0"]
  
  server_properties = <<PROPERTIES
# Replication
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Performance
num.io.threads=16
num.network.threads=8
num.replica.fetchers=4
socket.request.max.bytes=104857600
socket.receive.buffer.bytes=102400
socket.send.buffer.bytes=102400

# Log retention
log.retention.hours=168
log.retention.bytes=-1
log.segment.bytes=1073741824
log.cleanup.policy=delete

# Consumer groups
group.max.session.timeout.ms=300000
offsets.retention.minutes=10080

# Compression
compression.type=producer
PROPERTIES
}
```

### MSK Connect (Managed Kafka Connect):
```hcl
resource "aws_mskconnect_connector" "s3_sink" {
  name = "s3-sink-connector"
  
  kafkaconnect_version = "2.7.1"
  
  capacity {
    autoscaling {
      mcu_count        = 2
      min_worker_count = 2
      max_worker_count = 8
      
      scale_in_policy {
        cpu_utilization_percentage = 20
      }
      scale_out_policy {
        cpu_utilization_percentage = 80
      }
    }
  }
  
  connector_configuration = {
    "connector.class"          = "io.confluent.connect.s3.S3SinkConnector"
    "tasks.max"                = "8"
    "topics"                   = "events,transactions,user-activity"
    "s3.bucket.name"           = "company-datalake-bronze-prod"
    "s3.region"                = "us-east-1"
    "flush.size"               = "100000"
    "rotate.interval.ms"       = "300000"
    "storage.class"            = "io.confluent.connect.s3.storage.S3Storage"
    "format.class"             = "io.confluent.connect.s3.format.parquet.ParquetFormat"
    "parquet.codec"            = "snappy"
    "partitioner.class"        = "io.confluent.connect.storage.partitioner.TimeBasedPartitioner"
    "path.format"              = "'year'=YYYY/'month'=MM/'day'=dd/'hour'=HH"
    "locale"                   = "en-US"
    "timezone"                 = "UTC"
    "partition.duration.ms"    = "3600000"
    "schema.compatibility"     = "BACKWARD"
  }
  
  kafka_cluster {
    apache_kafka_cluster {
      bootstrap_servers = aws_msk_cluster.production.bootstrap_brokers_sasl_iam
      vpc {
        security_groups = [aws_security_group.msk_connect.id]
        subnets         = var.private_subnet_ids
      }
    }
  }
  
  plugin {
    custom_plugin {
      arn      = aws_mskconnect_custom_plugin.s3_sink.arn
      revision = aws_mskconnect_custom_plugin.s3_sink.latest_revision
    }
  }
}
```

## Amazon EMR Architecture

### EMR on EKS (Modern Pattern):
```hcl
# EMR on EKS - containerized Spark
resource "aws_emrcontainers_virtual_cluster" "data_platform" {
  name = "data-platform-spark"
  
  container_provider {
    id   = aws_eks_cluster.data.id
    type = "EKS"
    
    info {
      eks_info {
        namespace = "spark-jobs"
      }
    }
  }
}

# Submit Spark job to EMR on EKS
resource "null_resource" "submit_spark_job" {
  provisioner "local-exec" {
    command = <<-EOF
      aws emr-containers start-job-run \
        --virtual-cluster-id ${aws_emrcontainers_virtual_cluster.data_platform.id} \
        --name "daily-etl-${formatdate("YYYY-MM-DD", timestamp())}" \
        --execution-role-arn ${aws_iam_role.emr_execution.arn} \
        --release-label emr-7.0.0-latest \
        --job-driver '{
          "sparkSubmitJobDriver": {
            "entryPoint": "s3://scripts/daily_etl.py",
            "entryPointArguments": ["--date", "${var.processing_date}"],
            "sparkSubmitParameters": "--conf spark.executor.instances=20 --conf spark.executor.memory=16g --conf spark.executor.cores=4 --conf spark.driver.memory=8g"
          }
        }' \
        --configuration-overrides '{
          "applicationConfiguration": [
            {
              "classification": "spark-defaults",
              "properties": {
                "spark.sql.adaptive.enabled": "true",
                "spark.sql.catalog.glue_catalog": "org.apache.iceberg.spark.SparkCatalog",
                "spark.sql.catalog.glue_catalog.catalog-impl": "org.apache.iceberg.aws.glue.GlueCatalog",
                "spark.kubernetes.executor.podTemplateFile": "s3://config/pod-template.yaml"
              }
            }
          ],
          "monitoringConfiguration": {
            "cloudWatchMonitoringConfiguration": {
              "logGroupName": "/emr-on-eks/spark-jobs",
              "logStreamNamePrefix": "daily-etl"
            },
            "s3MonitoringConfiguration": {
              "logUri": "s3://logs/emr-on-eks/spark/"
            }
          }
        }'
    EOF
  }
}
```

### EMR Spot Instance Strategy:
```python
# Instance fleet configuration for cost optimization
instance_fleets = [
    {
        "Name": "Master",
        "InstanceFleetType": "MASTER",
        "TargetOnDemandCapacity": 1,
        "InstanceTypeConfigs": [
            {"InstanceType": "m6g.2xlarge", "WeightedCapacity": 1}
        ]
    },
    {
        "Name": "Core",
        "InstanceFleetType": "CORE",
        "TargetOnDemandCapacity": 5,  # On-demand for HDFS stability
        "InstanceTypeConfigs": [
            {"InstanceType": "r6g.2xlarge", "WeightedCapacity": 1, "BidPriceAsPercentageOfOnDemandPrice": 100},
            {"InstanceType": "r6g.4xlarge", "WeightedCapacity": 2, "BidPriceAsPercentageOfOnDemandPrice": 100},
            {"InstanceType": "r5.2xlarge", "WeightedCapacity": 1, "BidPriceAsPercentageOfOnDemandPrice": 100}
        ]
    },
    {
        "Name": "Task",
        "InstanceFleetType": "TASK",
        "TargetSpotCapacity": 20,  # 100% Spot for task nodes
        "LaunchSpecifications": {
            "SpotSpecification": {
                "TimeoutDurationMinutes": 15,
                "TimeoutAction": "SWITCH_TO_ON_DEMAND",
                "AllocationStrategy": "capacity-optimized"  # Best availability
            }
        },
        "InstanceTypeConfigs": [
            # Multiple instance types for spot availability
            {"InstanceType": "r6g.4xlarge", "WeightedCapacity": 2},
            {"InstanceType": "r5.4xlarge", "WeightedCapacity": 2},
            {"InstanceType": "r5a.4xlarge", "WeightedCapacity": 2},
            {"InstanceType": "m6g.4xlarge", "WeightedCapacity": 2},
            {"InstanceType": "r6i.4xlarge", "WeightedCapacity": 2}
        ]
    }
]
```

## AWS Glue Advanced Patterns

### Glue Workflow with Quality Checks:
```python
# Glue Workflow: Ingest → Validate → Transform → Publish
import boto3

glue = boto3.client('glue')

# Create workflow
glue.create_workflow(Name='daily-data-pipeline')

# Trigger: Schedule-based start
glue.create_trigger(
    Name='start-daily-pipeline',
    Type='SCHEDULED',
    Schedule='cron(0 2 * * ? *)',  # 2 AM UTC
    WorkflowName='daily-data-pipeline',
    Actions=[{'JobName': 'ingest-raw-data'}]
)

# Trigger: After ingest completes, run quality check
glue.create_trigger(
    Name='after-ingest-quality-check',
    Type='CONDITIONAL',
    WorkflowName='daily-data-pipeline',
    Predicate={
        'Conditions': [
            {'JobName': 'ingest-raw-data', 'State': 'SUCCEEDED', 'LogicalOperator': 'EQUALS'}
        ]
    },
    Actions=[{'JobName': 'data-quality-validation'}]
)

# Trigger: After quality passes, transform
glue.create_trigger(
    Name='after-quality-transform',
    Type='CONDITIONAL',
    WorkflowName='daily-data-pipeline',
    Predicate={
        'Conditions': [
            {'JobName': 'data-quality-validation', 'State': 'SUCCEEDED'}
        ]
    },
    Actions=[
        {'JobName': 'transform-silver'},
        {'JobName': 'transform-gold'}  # Parallel execution
    ]
)
```

### Glue Interactive Sessions (Development):
```python
# %magic commands for Glue Interactive Sessions
# %idle_timeout 60
# %glue_version 4.0
# %worker_type G.1X
# %number_of_workers 5
# %additional_python_modules great_expectations,delta-spark

from awsglue.context import GlueContext
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session

# Development workflow: test transformations interactively
# Then promote to production Glue Job
```

### Glue Data Quality (Native):
```python
from awsglue.transforms import *
from awsglueml.transforms import FindMatches

# Evaluate Data Quality rules inline
dq_results = EvaluateDataQuality.apply(
    frame=datasource,
    ruleset="""
        Rules = [
            RowCount between 100000 and 10000000,
            Completeness "customer_id" >= 0.99,
            Completeness "email" >= 0.95,
            IsUnique "order_id",
            ColumnValues "amount" between 0.01 and 999999.99,
            ColumnValues "status" in ["active", "pending", "closed"],
            CustomSql "SELECT COUNT(*) FROM primary WHERE amount < 0" = 0
        ]
    """,
    publishing_options={
        "dataQualityEvaluationContext": "dq_check",
        "enableDataQualityCloudWatchMetrics": True,
        "enableDataQualityResultsPublishing": True,
        "resultsS3Prefix": "s3://dq-results/orders/"
    }
)

# Route based on quality results
good_records = dq_results.filter(lambda row: row["DataQualityEvaluationResult"] == "Pass")
bad_records = dq_results.filter(lambda row: row["DataQualityEvaluationResult"] == "Fail")

# Good records → Silver zone
glueContext.write_dynamic_frame.from_options(
    frame=good_records,
    connection_type="s3",
    format="glueparquet",
    connection_options={"path": "s3://datalake-silver/orders/"}
)

# Bad records → Quarantine zone
glueContext.write_dynamic_frame.from_options(
    frame=bad_records,
    connection_type="s3",
    format="json",
    connection_options={"path": "s3://datalake-quarantine/orders/"}
)
```

## Kinesis vs MSK Decision Matrix

| Criteria | Kinesis Data Streams | MSK (Kafka) |
|----------|---------------------|-------------|
| Throughput | 1MB/sec/shard (scale shards) | 1GB+/sec/broker |
| Ordering | Per shard | Per partition |
| Retention | 1-365 days | Unlimited (disk) |
| Consumer Model | Pull (GetRecords) + Enhanced Fan-out | Pull (Consumer Groups) |
| Replay | By timestamp/sequence | By offset/timestamp |
| Ecosystem | AWS native (Lambda, Firehose) | Kafka ecosystem (Connect, Streams) |
| Multi-consumer | Enhanced Fan-out (2MB/sec/shard/consumer) | Consumer Groups (native) |
| Cost Model | Per shard hour + data | Per broker hour + storage |
| Best For | <100K msg/sec, serverless | >100K msg/sec, Kafka expertise |
| Operations | Fully managed | Semi-managed (config tuning needed) |

## Real-Time Processing Patterns

### Pattern: Lambda Architecture on AWS
```
┌────────────────────────────────────────────────────────┐
│  Speed Layer (Real-time)                                │
│  MSK → Flink → DynamoDB/ElastiCache (low-latency)     │
└────────────────────────────────┬───────────────────────┘
                                 │
                                 ▼ (merge at query time)
┌────────────────────────────────────────────────────────┐
│  Serving Layer                                          │
│  Athena / Redshift (query both real-time + batch)      │
└────────────────────────────────┬───────────────────────┘
                                 │
┌────────────────────────────────┴───────────────────────┐
│  Batch Layer (Accurate)                                 │
│  MSK → S3 → Glue/EMR → Iceberg Tables (truth)         │
└────────────────────────────────────────────────────────┘
```

### Pattern: Kappa Architecture (Simplified)
```
# Single pipeline for both real-time and batch
# Using Iceberg + Flink for unified processing

MSK → Flink → Iceberg Tables (S3)
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
  Athena       Redshift         Trino
(ad-hoc SQL)  (BI dashboards)  (federated)
```

### Flink on AWS (Managed Apache Flink):
```java
// Flink application for real-time CDC processing
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(60000); // 1 minute checkpoint interval
env.setStateBackend(new EmbeddedRocksDBStateBackend());

// Source: MSK (Kafka)
KafkaSource<String> source = KafkaSource.<String>builder()
    .setBootstrapServers(brokers)
    .setTopics("cdc-events")
    .setGroupId("flink-processor")
    .setStartingOffsets(OffsetsInitializer.committedOffsets())
    .setDeserializer(new SimpleStringSchema())
    .build();

DataStream<String> stream = env.fromSource(source, WatermarkStrategy.noWatermarks(), "MSK Source");

// Process: Windowed aggregations
DataStream<OrderMetrics> metrics = stream
    .map(new JsonToOrderMapper())
    .keyBy(order -> order.getCustomerId())
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(new OrderMetricsAggregator());

// Sink: Iceberg table
FlinkSink.forRowData(metrics)
    .tableLoader(TableLoader.fromCatalog(catalogLoader, TableIdentifier.of("db", "order_metrics")))
    .overwrite(false)
    .append();

env.execute("Real-time Order Metrics");
```
