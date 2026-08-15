# Disaster Recovery Patterns - AWS Enterprise DR Architecture

## DR Strategy Comparison

| Strategy | RTO | RPO | Cost | Complexity |
|----------|-----|-----|------|-----------|
| Backup & Restore | 24h+ | Hours | $ | Low |
| Pilot Light | 1-4h | Minutes | $$ | Medium |
| Warm Standby | 15-60m | Minutes | $$$ | High |
| Multi-Site Active-Active | Seconds | Near-zero | $$$$ | Very High |

## Strategy Selection by Workload Tier

```yaml
tier_1_mission_critical:
  description: "Payment processing, core trading systems"
  strategy: "Multi-Site Active-Active"
  rpo: "< 1 second (synchronous replication)"
  rto: "< 1 minute (automatic failover)"
  services: DynamoDB Global Tables, Aurora Global DB, Route 53 health checks
  cost_multiplier: 2x (full duplicate infrastructure)
  
tier_2_business_critical:
  description: "Customer-facing APIs, order management"
  strategy: "Warm Standby"
  rpo: "< 5 minutes (async replication)"
  rto: "< 15 minutes (automated scale-up)"
  services: Aurora Cross-Region Replica, ECS (min capacity), S3 CRR
  cost_multiplier: 1.3-1.5x
  
tier_3_important:
  description: "Analytics, reporting, internal tools"
  strategy: "Pilot Light"
  rpo: "< 1 hour (periodic backups)"
  rto: "< 4 hours (infrastructure provisioned, data restored)"
  services: RDS snapshots, AMIs, Terraform apply
  cost_multiplier: 1.1x
  
tier_4_non_critical:
  description: "Dev environments, documentation"
  strategy: "Backup & Restore"
  rpo: "24 hours (daily backups)"
  rto: "24-48 hours (rebuild from IaC)"
  services: S3 backup, CloudFormation/Terraform
  cost_multiplier: 1.02x (backup storage only)
```

## Active-Active Multi-Region Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Global Traffic Management                          │
│                                                                       │
│  ┌────────────────┐     ┌────────────────────────────┐              │
│  │  Route 53      │     │  Global Accelerator        │              │
│  │  (Latency)     │     │  (Static IPs, TCP optim.)  │              │
│  └───────┬────────┘     └─────────────┬──────────────┘              │
│          │                            │                              │
└──────────┼────────────────────────────┼──────────────────────────────┘
           │                            │
    ┌──────┴──────────────┐      ┌──────┴──────────────┐
    │  US-EAST-1 (Primary)│      │  EU-WEST-1 (Second) │
    │                     │      │                     │
    │  ┌───────────────┐  │      │  ┌───────────────┐  │
    │  │ ALB + WAF     │  │      │  │ ALB + WAF     │  │
    │  └───────┬───────┘  │      │  └───────┬───────┘  │
    │          │          │      │          │          │
    │  ┌───────▼───────┐  │      │  ┌───────▼───────┐  │
    │  │ ECS Fargate   │  │      │  │ ECS Fargate   │  │
    │  │ (Full scale)  │  │      │  │ (Full scale)  │  │
    │  └───────┬───────┘  │      │  └───────┬───────┘  │
    │          │          │      │          │          │
    │  ┌───────▼───────┐  │      │  ┌───────▼───────┐  │
    │  │ Aurora Global │  │◄────▶│  │ Aurora Global │  │
    │  │ (Writer)      │  │ Sync │  │ (Reader)      │  │
    │  └───────────────┘  │      │  └───────────────┘  │
    │  ┌───────────────┐  │      │  ┌───────────────┐  │
    │  │ DynamoDB      │  │◄────▶│  │ DynamoDB      │  │
    │  │ Global Table  │  │ Async│  │ Global Table  │  │
    │  └───────────────┘  │      │  └───────────────┘  │
    │  ┌───────────────┐  │      │  ┌───────────────┐  │
    │  │ ElastiCache   │  │      │  │ ElastiCache   │  │
    │  │ Global DS     │  │◄────▶│  │ Global DS     │  │
    │  └───────────────┘  │      │  └───────────────┘  │
    │  ┌───────────────┐  │      │  ┌───────────────┐  │
    │  │ S3 (CRR)      │  │◄────▶│  │ S3 (Replica)  │  │
    │  └───────────────┘  │      │  └───────────────┘  │
    └─────────────────────┘      └─────────────────────┘
```

## Aurora Global Database DR

```hcl
# Aurora Global Cluster
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "app-global-db"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  database_name             = "appdb"
  storage_encrypted         = true
}

# Primary Cluster (us-east-1)
resource "aws_rds_cluster" "primary" {
  provider = aws.us_east_1
  
  cluster_identifier        = "app-db-primary"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  master_username           = "admin"
  master_password           = var.db_password
  
  db_subnet_group_name    = aws_db_subnet_group.primary.name
  vpc_security_group_ids  = [aws_security_group.db_primary.id]
  
  backup_retention_period = 7
  preferred_backup_window = "03:00-04:00"
  deletion_protection     = true
  
  enabled_cloudwatch_logs_exports = ["postgresql"]
}

# Secondary Cluster (eu-west-1) - Read replica
resource "aws_rds_cluster" "secondary" {
  provider = aws.eu_west_1
  
  cluster_identifier        = "app-db-secondary"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
  
  db_subnet_group_name   = aws_db_subnet_group.secondary.name
  vpc_security_group_ids = [aws_security_group.db_secondary.id]
  
  # In DR: This gets promoted to standalone writer
  # Promotion takes < 1 minute for Aurora Global DB
}

# Failover Lambda (triggered by health check failure)
# Aurora Global DB managed failover:
# aws rds failover-global-cluster \
#   --global-cluster-identifier app-global-db \
#   --target-db-cluster-identifier app-db-secondary
```

## Backup Strategy

### AWS Backup Organization Policy:
```hcl
resource "aws_backup_plan" "organization" {
  name = "org-backup-plan-tier1"
  
  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.primary.name
    schedule          = "cron(0 3 * * ? *)"  # 3 AM daily
    
    lifecycle {
      cold_storage_after = 30   # Move to cold after 30 days
      delete_after       = 365  # Delete after 1 year
    }
    
    # Cross-region copy for DR
    copy_action {
      destination_vault_arn = aws_backup_vault.dr_region.arn
      
      lifecycle {
        cold_storage_after = 30
        delete_after       = 365
      }
    }
    
    # Cross-account copy (immutable backup)
    copy_action {
      destination_vault_arn = "arn:aws:backup:us-west-2:BACKUP_ACCOUNT:backup-vault:immutable-vault"
      
      lifecycle {
        delete_after = 2555  # 7 years (compliance)
      }
    }
  }
  
  rule {
    rule_name                = "hourly-backup-critical"
    target_vault_name        = aws_backup_vault.primary.name
    schedule                 = "cron(0 * * * ? *)"  # Every hour
    start_window             = 60
    completion_window        = 120
    enable_continuous_backup = true  # Point-in-time recovery
    
    lifecycle {
      delete_after = 35  # 35 days of PITR
    }
  }
}

# Backup Vault with lock (immutable - cannot be deleted)
resource "aws_backup_vault" "immutable" {
  name = "immutable-compliance-vault"
}

resource "aws_backup_vault_lock_configuration" "immutable" {
  backup_vault_name   = aws_backup_vault.immutable.name
  changeable_for_days = 3       # 3-day grace period
  max_retention_days  = 2555    # 7 years max
  min_retention_days  = 365     # 1 year minimum (cannot delete sooner)
}

# Selection: What to backup
resource "aws_backup_selection" "tier1" {
  name         = "tier1-resources"
  iam_role_arn = aws_iam_role.backup.arn
  plan_id      = aws_backup_plan.organization.id
  
  selection_tag {
    type  = "STRINGEQUALS"
    key   = "BackupTier"
    value = "tier-1"
  }
}
```

## Failover Automation

### Complete Failover Orchestration:
```python
# Step Functions state machine for automated regional failover
import boto3
import json
import time

class RegionalFailover:
    def __init__(self, primary_region, dr_region):
        self.primary_region = primary_region
        self.dr_region = dr_region
        self.failover_log = []
    
    def execute_failover(self):
        """Complete regional failover orchestration."""
        steps = [
            ("Validate DR region health", self.validate_dr_health),
            ("Promote Aurora Global DB", self.promote_aurora),
            ("Scale up ECS services", self.scale_ecs),
            ("Scale up ElastiCache", self.scale_elasticache),
            ("Update application config", self.update_app_config),
            ("Run smoke tests", self.run_smoke_tests),
            ("Update DNS (Route 53)", self.update_dns),
            ("Verify end-to-end", self.verify_e2e),
            ("Notify stakeholders", self.notify),
        ]
        
        for step_name, step_func in steps:
            self.failover_log.append({"step": step_name, "start": time.time()})
            try:
                result = step_func()
                self.failover_log[-1]["status"] = "success"
                self.failover_log[-1]["duration"] = time.time() - self.failover_log[-1]["start"]
            except Exception as e:
                self.failover_log[-1]["status"] = "failed"
                self.failover_log[-1]["error"] = str(e)
                raise FailoverException(f"Failover failed at step: {step_name}", self.failover_log)
        
        return self.failover_log
    
    def promote_aurora(self):
        """Promote Aurora Global DB secondary to primary."""
        rds = boto3.client('rds', region_name=self.dr_region)
        
        # Managed planned failover (fast, minimal data loss)
        rds.failover_global_cluster(
            GlobalClusterIdentifier='app-global-db',
            TargetDbClusterIdentifier=f'arn:aws:rds:{self.dr_region}:ACCOUNT:cluster:app-db-secondary'
        )
        
        # Wait for promotion (typically < 60 seconds)
        waiter = rds.get_waiter('db_cluster_available')
        waiter.wait(
            DBClusterIdentifier='app-db-secondary',
            WaiterConfig={'Delay': 5, 'MaxAttempts': 60}
        )
    
    def scale_ecs(self):
        """Scale ECS services to full production capacity."""
        ecs = boto3.client('ecs', region_name=self.dr_region)
        
        services = ['payment-api', 'order-service', 'user-service']
        target_counts = {'payment-api': 20, 'order-service': 15, 'user-service': 10}
        
        for service in services:
            ecs.update_service(
                cluster='production',
                service=service,
                desiredCount=target_counts[service]
            )
        
        # Wait for services to stabilize
        for service in services:
            waiter = ecs.get_waiter('services_stable')
            waiter.wait(
                cluster='production',
                services=[service],
                WaiterConfig={'Delay': 15, 'MaxAttempts': 40}
            )
    
    def update_dns(self):
        """Switch Route 53 to point to DR region."""
        route53 = boto3.client('route53')
        
        route53.change_resource_record_sets(
            HostedZoneId=HOSTED_ZONE_ID,
            ChangeBatch={
                'Changes': [{
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'api.company.com',
                        'Type': 'A',
                        'AliasTarget': {
                            'HostedZoneId': DR_ALB_ZONE_ID,
                            'DNSName': DR_ALB_DNS_NAME,
                            'EvaluateTargetHealth': True
                        },
                        'SetIdentifier': 'dr-active',
                        'Weight': 100
                    }
                }]
            }
        )
    
    def run_smoke_tests(self):
        """Run critical path tests against DR."""
        import requests
        
        dr_endpoint = f"https://api-dr.company.com"
        tests = [
            ("Health check", "GET", "/health"),
            ("List products", "GET", "/v1/products"),
            ("Auth flow", "POST", "/v1/auth/token"),
        ]
        
        for test_name, method, path in tests:
            response = requests.request(method, f"{dr_endpoint}{path}", timeout=10)
            if response.status_code >= 500:
                raise Exception(f"Smoke test failed: {test_name} - {response.status_code}")
```

## DR Testing Framework

### Automated DR Drill:
```hcl
# EventBridge scheduled rule for DR testing
resource "aws_cloudwatch_event_rule" "dr_drill" {
  name                = "quarterly-dr-drill"
  schedule_expression = "cron(0 10 1 1,4,7,10 ? *)"  # Quarterly
  description         = "Automated DR drill - first day of quarter at 10 AM"
}

resource "aws_cloudwatch_event_target" "dr_drill" {
  rule      = aws_cloudwatch_event_rule.dr_drill.name
  target_id = "trigger-dr-drill"
  arn       = aws_sfn_state_machine.dr_drill.arn
  role_arn  = aws_iam_role.eventbridge.arn
}
```

### DR Metrics & Reporting:
```yaml
dr_metrics:
  rpo_actual:
    measurement: "Time difference between last replicated data and failure point"
    target: "< 5 minutes"
    collection: "Compare Aurora replication lag at time of failover"
  
  rto_actual:
    measurement: "Time from failure detection to full service restoration"
    target: "< 15 minutes"
    breakdown:
      detection: "< 2 minutes (health check + evaluation)"
      failover_decision: "< 1 minute (automated)"
      infrastructure_promotion: "< 5 minutes (Aurora + ECS)"
      dns_propagation: "< 3 minutes (60s TTL)"
      validation: "< 4 minutes (smoke tests)"
  
  data_integrity:
    measurement: "% of transactions successfully processed post-failover"
    target: "100%"
    validation: "Compare transaction logs pre/post failover"
  
  drill_success_rate:
    measurement: "% of DR drills completed within RTO/RPO targets"
    target: "> 95%"
    reporting: quarterly
```
