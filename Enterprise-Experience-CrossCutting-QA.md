# Enterprise Experience - Cross-Cutting Interview Questions & Answers

## Enterprise Data Lake / Data Platform Architecture, Multi-Account AWS Environments, Platform Governance & Security

---

### Q1: You've been hired to design a data platform for a Fortune 500 company that has 200+ data sources, 5000 data consumers, and zero cloud presence. They want to migrate from on-premises Hadoop to AWS within 18 months. How do you approach this?

**Answer:**

**Phase 0: Assessment & Strategy (Months 1-2)**

```yaml
assessment:
  current_state:
    - Inventory all 200+ data sources (RDBMS, files, APIs, mainframes)
    - Map data flows and dependencies
    - Identify data owners and stewards
    - Classify data (PII, PHI, financial, public)
    - Assess data quality and governance maturity
    - Document SLAs and access patterns
    
  target_state:
    - Cloud-native lakehouse architecture
    - Self-service analytics for 5000 consumers
    - Real-time + batch processing
    - Enterprise-grade governance and security
    - Cost-optimized (vs on-prem Hadoop: typically 40-60% savings)
    
  migration_strategy:
    approach: "Lift-and-Shift → Re-Architecture (phased)"
    # Don't try to modernize everything at once
    # Phase 1: Move data to S3 (lift-and-shift)
    # Phase 2: Modernize processing (EMR → Glue/Spark)
    # Phase 3: Optimize architecture (Iceberg, streaming)
```

**Phase 1: Foundation (Months 2-5)**

```
┌─────────────────────────────────────────────────────────────┐
│  AWS Landing Zone + Data Platform Foundation                  │
│                                                              │
│  ├── Multi-Account Structure                                 │
│  │   ├── Management Account (billing, organizations)        │
│  │   ├── Security Account (GuardDuty, SecurityHub)          │
│  │   ├── Network Account (Transit GW, Direct Connect)       │
│  │   ├── Shared Services (CI/CD, AD, DNS)                   │
│  │   ├── Data Platform - Prod (lake, processing, analytics) │
│  │   ├── Data Platform - Non-Prod                           │
│  │   └── Data Sandbox (exploration, ML experiments)          │
│  │                                                           │
│  ├── Networking                                              │
│  │   ├── Direct Connect (10 Gbps) for data transfer         │
│  │   ├── Transit Gateway for cross-account routing          │
│  │   └── VPC Endpoints for all data services                │
│  │                                                           │
│  ├── Security Baseline                                       │
│  │   ├── Lake Formation (central governance)                │
│  │   ├── KMS (encryption strategy)                          │
│  │   ├── IAM Identity Center (federated access)             │
│  │   └── SCPs (prevent data exfiltration)                   │
│  │                                                           │
│  └── Data Platform Core                                      │
│      ├── S3 Data Lake (Bronze/Silver/Gold)                   │
│      ├── Glue Catalog (metadata management)                  │
│      ├── Lake Formation (access control)                     │
│      └── MWAA (orchestration)                                │
└─────────────────────────────────────────────────────────────┘
```

**Phase 2: Data Migration (Months 5-12)**

```python
# Migration priority matrix:
migration_waves = {
    "wave_1": {  # Months 5-7 (Quick wins, low risk)
        "sources": "Reference data, lookup tables, configs",
        "method": "AWS DMS (full-load)",
        "volume": "~5TB",
        "risk": "Low",
        "validation": "Row count + checksum comparison"
    },
    "wave_2": {  # Months 7-9 (Core transactional data)
        "sources": "OLTP databases (Oracle, SQL Server)",
        "method": "DMS CDC (change data capture)",
        "volume": "~50TB",
        "risk": "Medium",
        "validation": "Data quality framework + reconciliation jobs"
    },
    "wave_3": {  # Months 9-11 (Hadoop HDFS data)
        "sources": "HDFS (Hive tables, raw files)",
        "method": "S3 DistCP + Glue migration",
        "volume": "~500TB",
        "risk": "Medium",
        "validation": "Query result comparison (Hive vs Athena)"
    },
    "wave_4": {  # Months 11-14 (Streaming + real-time)
        "sources": "Kafka clusters, event streams",
        "method": "MSK MirrorMaker + topic migration",
        "volume": "~1TB/day continuous",
        "risk": "High",
        "validation": "Consumer lag monitoring + end-to-end latency"
    }
}
```

**Phase 3: Modernization (Months 12-18)**

```yaml
modernization:
  processing:
    before: "MapReduce + Hive on Hadoop (batch only)"
    after: "Glue (ETL) + EMR (complex) + Flink (streaming)"
    
  storage:
    before: "HDFS (no schema evolution, no ACID)"
    after: "S3 + Iceberg (ACID, schema evolution, time travel)"
    
  analytics:
    before: "Hive queries (slow, batch only)"
    after: "Athena (ad-hoc) + Redshift (BI) + SageMaker (ML)"
    
  governance:
    before: "Apache Ranger (partial coverage)"
    after: "Lake Formation (column-level, cross-account, tag-based)"
    
  orchestration:
    before: "Oozie (complex XML, fragile)"
    after: "MWAA/Airflow (Python DAGs, modular, testable)"
```

**Key Success Metrics:**

```yaml
success_metrics:
  migration:
    - 100% data sources migrated by month 14
    - Zero data loss during migration (validated via checksums)
    - All Hadoop jobs equivalent running on AWS by month 16
    
  performance:
    - Query performance: Same or better than on-prem
    - ETL job duration: 30% faster (Spark on S3 vs HDFS)
    - Data freshness: From daily batch → near-real-time for critical tables
    
  cost:
    - Total cost of ownership: 40% reduction vs on-prem Hadoop
    - Compute cost: 60% reduction (spot instances + serverless)
    - Storage cost: 70% reduction (S3 vs HDFS 3x replication)
    
  governance:
    - 100% of data classified and tagged
    - Column-level access control on all PII
    - Full audit trail for all data access
    
  adoption:
    - 5000 data consumers onboarded
    - Self-service: 80% of queries without platform team intervention
    - Time-to-insight: From weeks → hours
```

---

### Q2: Your multi-account AWS environment has 300 accounts. The CISO discovered that 15 accounts have S3 buckets with public access, 8 accounts have unencrypted RDS instances, and 3 accounts have IAM users with console access without MFA. You have 48 hours to prove compliance. What's your approach?

**Answer:**

**Hour 0-4: Immediate Assessment (Discovery)**

```python
import boto3
import json
from concurrent.futures import ThreadPoolExecutor

class ComplianceScanner:
    """Scan all 300 accounts for compliance violations in parallel."""
    
    def __init__(self):
        self.org_client = boto3.client('organizations')
        self.violations = []
    
    def scan_all_accounts(self):
        accounts = self.get_all_accounts()
        
        with ThreadPoolExecutor(max_workers=20) as executor:
            futures = {
                executor.submit(self.scan_account, acct): acct
                for acct in accounts
            }
            for future in futures:
                result = future.result()
                self.violations.extend(result)
        
        return self.generate_report()
    
    def scan_account(self, account):
        """Scan single account for all compliance issues."""
        creds = self.assume_role(account['Id'])
        violations = []
        
        # Check 1: Public S3 buckets
        violations.extend(self.check_public_s3(account['Id'], creds))
        
        # Check 2: Unencrypted RDS
        violations.extend(self.check_unencrypted_rds(account['Id'], creds))
        
        # Check 3: IAM users without MFA
        violations.extend(self.check_mfa_compliance(account['Id'], creds))
        
        # Check 4: Security groups with 0.0.0.0/0
        violations.extend(self.check_open_security_groups(account['Id'], creds))
        
        # Check 5: Unencrypted EBS volumes
        violations.extend(self.check_unencrypted_ebs(account['Id'], creds))
        
        return violations
    
    def check_public_s3(self, account_id, creds):
        s3 = boto3.client('s3', **creds)
        violations = []
        
        buckets = s3.list_buckets()['Buckets']
        for bucket in buckets:
            try:
                # Check bucket-level public access block
                pab = s3.get_public_access_block(Bucket=bucket['Name'])
                config = pab['PublicAccessBlockConfiguration']
                if not all([
                    config.get('BlockPublicAcls', False),
                    config.get('BlockPublicPolicy', False),
                    config.get('IgnorePublicAcls', False),
                    config.get('RestrictPublicBuckets', False)
                ]):
                    violations.append({
                        'account_id': account_id,
                        'resource_type': 'S3',
                        'resource_id': bucket['Name'],
                        'violation': 'Public access not fully blocked',
                        'severity': 'CRITICAL',
                        'auto_remediate': True
                    })
            except s3.exceptions.NoSuchPublicAccessBlockConfiguration:
                violations.append({
                    'account_id': account_id,
                    'resource_type': 'S3',
                    'resource_id': bucket['Name'],
                    'violation': 'No public access block configured',
                    'severity': 'CRITICAL',
                    'auto_remediate': True
                })
        
        return violations
    
    def check_mfa_compliance(self, account_id, creds):
        iam = boto3.client('iam', **creds)
        violations = []
        
        users = iam.list_users()['Users']
        for user in users:
            # Check if user has console access
            try:
                login_profile = iam.get_login_profile(UserName=user['UserName'])
                # User has console access - check MFA
                mfa_devices = iam.list_mfa_devices(UserName=user['UserName'])
                if not mfa_devices['MFADevices']:
                    violations.append({
                        'account_id': account_id,
                        'resource_type': 'IAM',
                        'resource_id': user['UserName'],
                        'violation': 'Console access without MFA',
                        'severity': 'CRITICAL',
                        'auto_remediate': False  # Need user action
                    })
            except iam.exceptions.NoSuchEntityException:
                pass  # No console access - OK
        
        return violations
```

**Hour 4-12: Auto-Remediation (Fix Critical Issues)**

```python
def auto_remediate_violations(violations):
    """Fix all auto-remediable violations immediately."""
    
    for v in violations:
        if not v['auto_remediate']:
            continue
        
        creds = assume_role(v['account_id'])
        
        if v['resource_type'] == 'S3' and 'public access' in v['violation']:
            # Block public access immediately
            s3 = boto3.client('s3', **creds)
            s3.put_public_access_block(
                Bucket=v['resource_id'],
                PublicAccessBlockConfiguration={
                    'BlockPublicAcls': True,
                    'IgnorePublicAcls': True,
                    'BlockPublicPolicy': True,
                    'RestrictPublicBuckets': True
                }
            )
            log_remediation(v, 'Public access blocked')
        
        elif v['resource_type'] == 'RDS' and 'unencrypted' in v['violation']:
            # Cannot encrypt in-place - create encrypted copy
            rds = boto3.client('rds', **creds)
            # Take snapshot → Copy with encryption → Restore
            create_encrypted_copy_plan(rds, v['resource_id'])
            log_remediation(v, 'Encrypted copy plan created (requires maintenance window)')
        
        elif v['resource_type'] == 'S3' and 'account-level' in v['violation']:
            # Apply account-level S3 block
            s3control = boto3.client('s3control', **creds)
            s3control.put_public_access_block(
                AccountId=v['account_id'],
                PublicAccessBlockConfiguration={
                    'BlockPublicAcls': True,
                    'IgnorePublicAcls': True,
                    'BlockPublicPolicy': True,
                    'RestrictPublicBuckets': True
                }
            )
            log_remediation(v, 'Account-level public access block applied')
```

**Hour 12-24: Prevention (Ensure It Can't Happen Again)**

```hcl
# SCP: Prevent disabling S3 public access block
resource "aws_organizations_policy" "prevent_public_s3" {
  name    = "prevent-public-s3"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyS3PublicAccess"
        Effect = "Deny"
        Action = [
          "s3:PutBucketPublicAccessBlock",
          "s3:PutAccountPublicAccessBlock"
        ]
        Resource = "*"
        Condition = {
          "StringNotEquals" = {
            "s3:PublicAccessBlockConfiguration/BlockPublicAcls" = "true"
          }
        }
      },
      {
        Sid    = "DenyUnencryptedRDS"
        Effect = "Deny"
        Action = ["rds:CreateDBInstance", "rds:CreateDBCluster"]
        Resource = "*"
        Condition = {
          "Bool" = { "rds:StorageEncrypted" = "false" }
        }
      }
    ]
  })
}

# AWS Config Rules (detective + auto-remediation)
resource "aws_config_organization_managed_rule" "s3_public" {
  name            = "s3-bucket-public-read-prohibited"
  rule_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
}

resource "aws_config_organization_managed_rule" "rds_encrypted" {
  name            = "rds-storage-encrypted"
  rule_identifier = "RDS_STORAGE_ENCRYPTED"
}

resource "aws_config_organization_managed_rule" "mfa_enabled" {
  name            = "iam-user-mfa-enabled"
  rule_identifier = "IAM_USER_MFA_ENABLED"
}
```

**Hour 24-48: Compliance Report for CISO**

```yaml
compliance_report:
  executive_summary:
    total_accounts_scanned: 300
    violations_found: 26
    violations_remediated: 23 (auto-remediated within 12 hours)
    violations_pending: 3 (require maintenance window for RDS encryption)
    
  findings:
    critical:
      - "15 S3 buckets had public access → ALL BLOCKED within 4 hours"
      - "8 RDS instances unencrypted → Encryption plan created, scheduled"
      - "3 IAM users without MFA → Users notified, 48-hour deadline"
    
  preventive_controls_deployed:
    - "SCP: Blocks creating unencrypted RDS in all accounts"
    - "SCP: Blocks disabling S3 public access block"
    - "SCP: Enforces IMDSv2 on all EC2 instances"
    - "Config Rules: 15 organization-wide rules deployed"
    - "Auto-Remediation: 5 Lambda functions for automatic fix"
    
  ongoing_monitoring:
    - "Security Hub: CIS Benchmark + PCI-DSS standards enabled"
    - "GuardDuty: Organization-wide threat detection"
    - "Weekly compliance scan: Automated report to CISO"
    - "Real-time alerts: Any new violation pages security team"
```

---

### Q3: Your company has acquired another company. They have their own AWS Organization with 50 accounts, different networking (overlapping CIDRs), different IAM strategy, and no governance. You need to integrate them into your Organization within 90 days. What's your plan?

**Answer:**

**Integration Architecture:**

```
┌──────────────────────────────────────────────────────────────────┐
│  BEFORE (Day 0):                                                  │
│                                                                    │
│  Organization A (Yours)        Organization B (Acquired)          │
│  ├── 300 accounts              ├── 50 accounts                    │
│  ├── CIDR: 10.0.0.0/8         ├── CIDR: 10.0.0.0/8 (OVERLAP!)  │
│  ├── IAM Identity Center       ├── IAM Users (local)             │
│  ├── Control Tower             ├── No governance                  │
│  ├── Transit Gateway           ├── VPC Peering (mesh)             │
│  └── Lake Formation            └── No data governance             │
│                                                                    │
│  AFTER (Day 90):                                                  │
│                                                                    │
│  Single Organization (Merged)                                      │
│  ├── 350 accounts (phased migration)                              │
│  ├── Unified IAM Identity Center                                  │
│  ├── Non-overlapping CIDRs (re-addressed or NATed)               │
│  ├── Single Transit Gateway (interconnected)                      │
│  └── Unified governance (SCPs, Config, Lake Formation)            │
└──────────────────────────────────────────────────────────────────┘
```

**Phase 1: Connectivity Without Merger (Days 1-30)**

```hcl
# Step 1: Don't merge orgs yet! First establish connectivity.
# Use Transit Gateway PEERING between organizations (no org merge needed)

# Your org's TGW
resource "aws_ec2_transit_gateway" "yours" {
  description = "Primary org TGW"
}

# Peering with acquired org's TGW (cross-account)
resource "aws_ec2_transit_gateway_peering_attachment" "acquisition" {
  peer_account_id         = var.acquired_org_account_id
  peer_region             = "us-east-1"
  peer_transit_gateway_id = var.acquired_tgw_id
  transit_gateway_id      = aws_ec2_transit_gateway.yours.id
}

# For OVERLAPPING CIDRs: Use PrivateLink (no routing conflict)
# Service in Acquired Org → NLB → PrivateLink → Your Org (Endpoint)
# This works even with identical CIDRs!
```

**Phase 2: Identity Integration (Days 15-45)**

```yaml
identity_integration:
  approach: "Federate acquired company into YOUR Identity Center"
  
  steps:
    1_azure_ad_integration:
      - Add acquired company's Azure AD as secondary IdP
      - Or: Sync their AD users into your Azure AD tenant
      - SCIM provisioning to IAM Identity Center
    
    2_permission_sets:
      - Create permission sets matching their current access levels
      - Map their roles → your permission sets
      - "Their Admin" → "AcquiredCompany-Admin" permission set (initially broad)
      
    3_phased_access:
      - Week 1: Both old IAM users AND new SSO work simultaneously
      - Week 3: Notify users of migration, provide SSO training
      - Week 5: Disable old IAM users (with 1-week grace period)
      - Week 6: Delete old IAM users
    
    4_principle:
      # NEVER cut access first. Always add new access, validate, THEN remove old.
      # "Make the new way work before breaking the old way"
```

**Phase 3: Account Migration into Your Organization (Days 30-60)**

```python
# Move accounts from Org B to Org A (one at a time)
# WARNING: This removes the account from Org B's SCPs, billing, etc.

def migrate_account(account_id, source_org_id, target_org_id):
    """Migrate account between organizations."""
    
    # Step 1: Remove from source organization
    # (Account owner must accept invitation to new org)
    orgs_source = boto3.client('organizations', **source_org_creds)
    orgs_source.remove_account_from_organization(AccountId=account_id)
    
    # Step 2: Invite to target organization
    orgs_target = boto3.client('organizations', **target_org_creds)
    invitation = orgs_target.invite_account_to_organization(
        Target={'Id': account_id, 'Type': 'ACCOUNT'}
    )
    
    # Step 3: Accept invitation (from the account itself)
    orgs_account = boto3.client('organizations', **account_creds)
    orgs_account.accept_handshake(HandshakeId=invitation['Handshake']['Id'])
    
    # Step 4: Move to correct OU
    orgs_target.move_account(
        AccountId=account_id,
        SourceParentId=ROOT_ID,
        DestinationParentId=ACQUIRED_OU_ID
    )
    
    # Step 5: Apply baseline (StackSets deploy automatically to new OU)
    # Config, GuardDuty, CloudTrail, IAM roles are all applied via StackSets
    
    # Step 6: Verify compliance
    verify_account_compliance(account_id)
```

**Phase 4: Network Re-Addressing (Days 45-75)**

```yaml
network_strategy:
  option_a_nat_translation:
    description: "NAT acquired company's CIDRs to non-overlapping range"
    implementation: "Network Firewall with NAT rules"
    pros: "No application changes in acquired company"
    cons: "Complexity, troubleshooting difficulty"
    
  option_b_gradual_reip:
    description: "Re-address acquired VPCs to new CIDR ranges"
    implementation: "Add secondary CIDR → migrate workloads → remove old"
    pros: "Clean long-term architecture"
    cons: "Application config changes, DNS updates needed"
    timeline: "3-6 months (can extend beyond 90 days)"
    
  option_c_privatelink:
    description: "Cross-org communication via PrivateLink only"
    implementation: "Each shared service exposed via NLB + endpoint"
    pros: "Zero CIDR conflicts, works immediately"
    cons: "Not suitable for all traffic patterns, per-service setup"
    
  recommended: "Phase 1: PrivateLink for immediate connectivity
                Phase 2: Gradual re-addressing over 6 months"
```

**Phase 5: Governance Unification (Days 60-90)**

```hcl
# Create "Acquired" OU with relaxed SCPs (don't break their stuff!)
resource "aws_organizations_organizational_unit" "acquired" {
  name      = "Acquired-Company"
  parent_id = aws_organizations_organizational_unit.workloads.id
}

# Initially permissive SCP (just enforce critical security)
resource "aws_organizations_policy" "acquired_initial" {
  name    = "acquired-initial-guardrails"
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "EnforceMinimumSecurity"
        Effect = "Deny"
        Action = [
          "cloudtrail:StopLogging",
          "cloudtrail:DeleteTrail",
          "config:StopConfigurationRecorder",
          "guardduty:DeleteDetector"
        ]
        Resource = "*"
      },
      {
        Sid    = "DenyPublicAccess"
        Effect = "Deny"
        Action = ["s3:PutBucketPublicAccessBlock"]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "s3:PublicAccessBlockConfiguration/BlockPublicAcls" = "true"
          }
        }
      }
    ]
  })
}

# Over next 6 months: gradually tighten SCPs to match your org's standards
# Month 1: Security basics (above)
# Month 3: Add region restrictions, encryption requirements
# Month 6: Full parity with your org's SCPs
```

---

### Q4: Design a platform governance framework for an enterprise with 500+ engineers across 30 teams that balances developer freedom with security/compliance requirements. How do you avoid becoming a bottleneck while maintaining control?

**Answer:**

**Governance Philosophy: "Paved Roads, Not Walls"**

```
Traditional IT Governance:     Platform Engineering Governance:
┌─────────────────────┐       ┌─────────────────────────────┐
│  WALL               │       │  PAVED ROAD                  │
│                     │       │                              │
│  Ticket → Wait 2wk │       │  Self-service (2 min)        │
│  → Manual approval  │       │  ├── Guardrails (automated) │
│  → Manual provision │       │  ├── Templates (best practice│
│  → Manual config    │       │  │   baked in)               │
│  → Developer angry  │       │  ├── Visibility (not control)│
│                     │       │  └── Freedom within bounds   │
│  Result: Shadow IT  │       │                              │
│  (teams bypass you) │       │  Result: Compliance + Speed  │
└─────────────────────┘       └─────────────────────────────┘
```

**Framework Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│  Platform Governance Layers                                      │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Layer 1: PREVENT (Guardrails - Automated)                │   │
│  │  SCPs, Permission Boundaries, Network Policies            │   │
│  │  "You CAN'T do this even if you try"                     │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Layer 2: DETECT (Compliance Monitoring - Automated)       │   │
│  │  AWS Config, Security Hub, custom Lambda scanners         │   │
│  │  "We KNOW when you do something non-compliant"           │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Layer 3: REMEDIATE (Auto-fix - Automated)                │   │
│  │  Config Remediation, EventBridge → Lambda                 │   │
│  │  "Non-compliance is automatically corrected"              │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Layer 4: ENABLE (Self-Service - Platform Team builds)    │   │
│  │  Service Catalog, Terraform modules, Backstage portal     │   │
│  │  "Here's the easy way to do things right"                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Layer 5: EDUCATE (Documentation + Support)               │   │
│  │  Runbooks, office hours, Slack channels, training         │   │
│  │  "Here's WHY and HOW"                                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation: Self-Service with Guardrails**

```yaml
# What teams CAN do without approval (self-service):
self_service:
  - Deploy applications (to their own accounts)
  - Create databases (from approved templates)
  - Scale resources (within budget limits)
  - Modify application configs
  - Create feature branches and deploy to dev
  - Access their own logs and metrics
  - Create S3 buckets (encryption auto-enforced)
  - Deploy Lambda functions
  - Manage their CI/CD pipelines

# What requires light approval (automated checks + team lead):
light_approval:
  - Production deployments (auto-approve if tests pass)
  - Budget increases > 20% (team lead approval)
  - Cross-team data sharing (data owner approval via Lake Formation)
  - New AWS service usage (security review, 1-day SLA)

# What requires heavy approval (security team + architecture):
heavy_approval:
  - Internet-facing resources (WAF, Shield required)
  - Cross-account IAM roles (security review)
  - New VPC or network changes (network team review)
  - Exceptions to SCPs (CISO approval + time-limited)
  - PII data handling changes (legal + DPO approval)
```

**Guardrail Implementation:**

```hcl
# Preventive: Teams literally cannot violate these
preventive_controls = {
  "no-public-s3"           = "SCP blocks public bucket creation"
  "enforce-encryption"     = "SCP requires KMS on all storage"
  "region-restriction"     = "SCP limits to approved regions"
  "no-iam-users"          = "SCP blocks IAM user creation (SSO only)"
  "imdsv2-required"       = "SCP enforces IMDSv2"
  "no-root-usage"         = "SCP denies root user actions"
  "vpc-required"          = "SCP blocks resources outside VPC"
}

# Detective: Know when violations occur
detective_controls = {
  "unencrypted-storage"   = "Config rule + SNS alert"
  "unused-iam-keys"       = "Config rule + 90-day rotation"
  "public-security-group" = "Config rule + auto-remediation"
  "cost-anomaly"          = "Cost Anomaly Detection + budget alerts"
  "compliance-drift"      = "Security Hub + weekly report"
}

# Corrective: Auto-fix violations
corrective_controls = {
  "public-sg-ingress"     = "Lambda removes 0.0.0.0/0 rules"
  "missing-tags"          = "Lambda applies default tags"
  "unencrypted-ebs"       = "Prevent attachment to instances"
  "exposed-credentials"   = "Lambda rotates + alerts team"
}
```

**Developer Experience (The "Golden Path"):**

```bash
# What a developer experiences:
$ platform new-service --name order-api --team payments --tier standard

# Platform CLI automatically:
# ✅ Creates repo from template (Dockerfile, CI/CD, IaC included)
# ✅ Provisions infrastructure (ECS, RDS, Redis - all compliant)
# ✅ Sets up monitoring (dashboards, alerts based on tier)
# ✅ Configures CI/CD (build, test, deploy pipeline)
# ✅ Registers in service catalog (Backstage)
# ✅ Applies governance (tags, encryption, logging, access control)
# ✅ Grants team access (SSO permissions)

# Time: 5 minutes (vs 2 weeks with traditional ticketing)
# Compliance: 100% (baked into templates)
# Developer friction: Near zero
```

---

### Q5: You're responsible for cost governance across a $10M/month AWS bill with 300 accounts. Finance wants 15% cost reduction without impacting performance. How do you systematically find and implement savings?

**Answer:**

**Cost Optimization Framework:**

```
$10M/month breakdown (typical enterprise):
├── Compute (EC2/ECS/EKS):     40% = $4.0M
├── Storage (S3/EBS/EFS):      15% = $1.5M
├── Data Transfer:             12% = $1.2M
├── Database (RDS/DynamoDB):   15% = $1.5M
├── Analytics (Redshift/EMR):   8% = $0.8M
├── Observability (CW/logs):    5% = $0.5M
└── Other (Lambda, NAT, etc):   5% = $0.5M

Target: 15% = $1.5M/month savings
```

**Tier 1: Quick Wins (Week 1-2, ~$500K savings)**

```python
# 1. Right-sizing (typically 30-40% of instances are oversized)
def find_oversized_instances():
    """Find instances with < 10% avg CPU utilization."""
    ce = boto3.client('ce')
    
    # Get rightsizing recommendations
    response = ce.get_rightsizing_recommendation(
        Service='AmazonEC2',
        Configuration={
            'RecommendationTarget': 'SAME_INSTANCE_FAMILY',
            'BenefitsConsidered': True
        }
    )
    
    total_savings = sum(
        float(r['ModifyRecommendationDetail']['TargetInstances'][0]
              ['EstimatedMonthlySavings']['Value'])
        for r in response['RightsizingRecommendations']
        if r['RightsizingType'] == 'Modify'
    )
    print(f"Right-sizing potential: ${total_savings:,.0f}/month")

# 2. Unused resources (EBS volumes, EIPs, old snapshots)
def find_unused_resources():
    """Find resources costing money but providing no value."""
    # Unattached EBS volumes
    # Elastic IPs not associated with running instances
    # Old EBS snapshots (> 90 days, no AMI reference)
    # Idle load balancers (0 requests/day)
    # Stopped EC2 instances (still paying for EBS)
    pass

# 3. S3 lifecycle policies (move old data to cheaper tiers)
def optimize_s3_storage():
    """Apply lifecycle policies to reduce S3 costs by 50%."""
    # 60% of S3 data is > 30 days old and rarely accessed
    # Move to S3-IA: 43% cheaper
    # Move to Glacier IR after 90 days: 68% cheaper
    pass
```

**Tier 2: Commitment Discounts (Week 2-4, ~$600K savings)**

```yaml
commitment_strategy:
  compute_savings_plans:
    # 1-year No Upfront: 20% discount
    # 3-year All Upfront: 60% discount
    # Target: Cover 70% of baseline compute with Savings Plans
    
    analysis:
      - Current on-demand compute: $4.0M/month
      - Baseline (always-running): $2.8M/month (70%)
      - Savings Plan coverage: $2.8M × 30% discount = $840K savings
      - Actual commitment: $1.96M/month (covers $2.8M of usage)
    
    recommendation:
      - Compute Savings Plan (1-year, partial upfront): $1.5M/month
      - Remaining on-demand for burst: $2.5M/month
      - Savings: ~$600K/month
  
  reserved_instances:
    rds: "Reserve 80% of production RDS instances (1-year)"
    elasticache: "Reserve all production Redis/Memcached nodes"
    redshift: "Reserve provisioned nodes"
    
  spot_instances:
    emr_task_nodes: "90% on Spot (70% savings)"
    ecs_non_critical: "Fargate Spot for staging/dev"
    batch_processing: "100% Spot for batch workloads"
```

**Tier 3: Architectural Changes (Month 1-3, ~$400K savings)**

```yaml
architectural_optimizations:
  # 1. Consolidate NAT Gateways
  nat_gateway:
    current: "3 NAT GWs per VPC × 50 VPCs = 150 NAT GWs"
    optimized: "Centralized egress via Network Hub (3 NAT GWs total)"
    savings: "$50K/month"
  
  # 2. VPC Endpoints (eliminate data transfer through NAT)
  vpc_endpoints:
    current: "All AWS API traffic through NAT Gateway ($0.045/GB)"
    optimized: "S3 Gateway (free) + Interface endpoints for top services"
    savings: "$80K/month"
  
  # 3. Data transfer optimization
  data_transfer:
    current: "Cross-AZ transfer for logging, metrics, inter-service"
    optimized:
      - "VPC Lattice (same-AZ preference)"
      - "Compress log shipping (5:1)"
      - "CloudWatch Logs → S3 (instead of cross-region replication)"
    savings: "$100K/month"
  
  # 4. Serverless migration for appropriate workloads
  serverless:
    candidates: "Event-driven, low-traffic, batch APIs"
    from: "EC2/ECS (always-on, $$$)"
    to: "Lambda + API Gateway (pay per request)"
    savings: "$70K/month"
  
  # 5. Storage optimization
  storage:
    - "S3 Intelligent-Tiering (auto-optimizes access patterns)"
    - "EBS gp3 migration from gp2 (20% cheaper, better performance)"
    - "Delete unused EBS snapshots older than 90 days"
    savings: "$100K/month"
```

**Governance: Prevent Cost Regression**

```hcl
# Budget alarms per account
resource "aws_budgets_budget" "account" {
  for_each = toset(var.all_account_ids)
  
  name         = "monthly-budget-${each.value}"
  budget_type  = "COST"
  limit_amount = var.account_budgets[each.value]
  limit_unit   = "USD"
  time_unit    = "MONTHLY"
  
  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type           = "PERCENTAGE"
    notification_type        = "ACTUAL"
    subscriber_email_addresses = [var.team_emails[each.value]]
  }
  
  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type           = "PERCENTAGE"
    notification_type        = "ACTUAL"
    subscriber_email_addresses = [var.finance_email, var.team_emails[each.value]]
  }
}

# Tag enforcement (mandatory cost allocation tags)
resource "aws_organizations_policy" "tagging" {
  name    = "cost-allocation-tags"
  type    = "TAG_POLICY"
  content = jsonencode({
    tags = {
      CostCenter = {
        tag_key = { "@@assign" = "CostCenter" }
        enforced_for = { "@@assign" = ["ec2:instance", "rds:db", "s3:bucket"] }
      }
      Team = {
        tag_key = { "@@assign" = "Team" }
      }
    }
  })
}
```

**Monthly Cost Review Process:**

```yaml
monthly_cost_governance:
  week_1:
    - Generate per-team cost reports (Cost Explorer + custom dashboard)
    - Identify top 10 cost increases vs last month
    - Flag accounts exceeding budget
    
  week_2:
    - Meet with top 3 cost-increasing teams
    - Review recommendations (Trusted Advisor, Compute Optimizer)
    - Approve/schedule right-sizing changes
    
  week_3:
    - Implement approved changes
    - Update Savings Plans coverage (quarterly)
    - Clean up unused resources (automated script)
    
  week_4:
    - Executive report to CFO
    - Track savings vs target (cumulative)
    - Plan next month's optimization focus

  kpis:
    - Cost per transaction (efficiency metric)
    - Savings Plan utilization > 90%
    - Resource waste ratio < 5%
    - Cost variance to budget < 10%
```

**Results Summary:**

```
Tier 1 (Quick wins):          $500K/month  (5% of bill)
Tier 2 (Commitments):         $600K/month  (6% of bill)
Tier 3 (Architecture):        $400K/month  (4% of bill)
───────────────────────────────────────────────────────
Total savings:                $1.5M/month  (15% of bill) ✓
Implementation timeline:      90 days
Ongoing governance:           Automated + monthly review
```
