# Platform Foundation - Tricky Production-Based Interview Questions & Answers

## Landing Zone & Multi-Account Architecture

### Q1: Your organization has 200+ AWS accounts. A developer accidentally deleted an SCP that enforced encryption. How would you detect and prevent this in production?

**Answer:**

**Detection:**
- Enable AWS CloudTrail Organization Trail logging all management events across all accounts
- Create an EventBridge rule in the management account to detect `organizations:DeletePolicy` and `organizations:DetachPolicy` API calls
- Configure AWS Config rule `organizations-policy-changes` to track SCP modifications
- Set up SNS alerting + PagerDuty integration for immediate notification

**Prevention:**
- Apply an SCP on the management account OU that denies `organizations:Delete*` actions unless the principal is a break-glass role
- Use AWS Control Tower Guardrails (preventive) to restrict SCP modification
- Implement a CI/CD pipeline for SCP changes using Terraform/CloudFormation with mandatory PR reviews
- Enable MFA delete on critical SCPs using session policy conditions

**Production Pattern:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteSCPUnlessBreakGlass",
      "Effect": "Deny",
      "Action": [
        "organizations:DeletePolicy",
        "organizations:DetachPolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/BreakGlassRole"
        }
      }
    }
  ]
}
```

---

### Q2: You need to design a multi-account architecture for a financial services company with strict PCI-DSS compliance. How would you structure the OUs and accounts?

**Answer:**

**OU Structure:**
```
Root
├── Security OU
│   ├── Log Archive Account (centralized logging)
│   ├── Security Tooling Account (GuardDuty, Security Hub delegated admin)
│   └── Forensics Account (isolated incident response)
├── Infrastructure OU
│   ├── Network Hub Account (Transit Gateway, DNS, Firewall)
│   ├── Shared Services Account (AD, CI/CD tools)
│   └── Backup Account (centralized backup vault)
├── Workloads OU
│   ├── PCI OU (Cardholder Data Environment)
│   │   ├── PCI-Prod Account
│   │   ├── PCI-Staging Account
│   │   └── PCI-Dev Account
│   └── Non-PCI OU
│       ├── Prod Account
│       ├── Staging Account
│       └── Dev Account
├── Sandbox OU
│   └── Developer Sandbox Accounts
└── Suspended OU
    └── (Quarantined accounts)
```

**Key Design Decisions:**
- **PCI Scope Isolation:** Separate OU with stricter SCPs (deny internet-facing resources, enforce encryption, restrict regions)
- **Log Immutability:** Log Archive uses S3 Object Lock with Governance mode, CloudTrail logs with resource policy preventing deletion
- **Network Segmentation:** PCI VPCs are isolated with dedicated Transit Gateway route tables, no peering to non-PCI
- **Break-Glass:** Dedicated IAM role in management account with hardware MFA, audited via CloudTrail

**SCPs for PCI OU:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedStorage",
      "Effect": "Deny",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyNonApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

---

### Q3: Your Transit Gateway has 5000+ routes and you're hitting the route table limit. How do you redesign the network to scale beyond this?

**Answer:**

**Problem Analysis:**
- AWS Transit Gateway route table limit: 10,000 static routes, but performance degrades at scale
- With 200+ VPCs, route propagation creates operational complexity

**Solution - Hub-and-Spoke with Regional TGWs:**

1. **Regional Transit Gateways:** Deploy TGW per region, connect via TGW Peering (inter-region)
2. **Route Domain Segmentation:**
   - Create separate TGW Route Tables per environment (Prod, Non-Prod, Shared)
   - Use route table associations and propagations to limit blast radius
3. **Summarized Routes:**
   - Instead of propagating /28 per VPC, use CIDR summarization
   - Allocate contiguous CIDR blocks per OU: `10.0.0.0/16` for Prod, `10.1.0.0/16` for Non-Prod
   - Advertise summary routes only

4. **AWS Cloud WAN (for large scale):**
   - Migrate to Cloud WAN with network policies
   - Define segments (Production, Development, Shared Services)
   - Use attachment policies for automatic segment association

```hcl
# Cloud WAN Core Network Policy
resource "aws_networkmanager_core_network" "main" {
  global_network_id = aws_networkmanager_global_network.main.id
  
  policy_document = jsonencode({
    "core-network-configuration" = {
      "vpn-ecmp-support" = true
      "asn-ranges"       = ["64512-65534"]
      "edge-locations" = [
        { "location" = "us-east-1" },
        { "location" = "us-west-2" }
      ]
    }
    "segments" = [
      { "name" = "production", "require-attachment-acceptance" = true },
      { "name" = "development", "isolate-attachments" = true },
      { "name" = "shared-services" }
    ]
    "segment-actions" = [
      {
        "action" = "share"
        "mode"   = "attachment-route"
        "segment" = "shared-services"
        "share-with" = ["production", "development"]
      }
    ]
  })
}
```

---

### Q4: An application team says they cannot access a service in a Shared Services account via AWS PrivateLink. How do you troubleshoot this in production?

**Answer:**

**Systematic Troubleshooting Steps:**

1. **Verify Endpoint Service Configuration (Provider Side - Shared Services Account):**
   ```bash
   aws ec2 describe-vpc-endpoint-services \
     --service-names com.amazonaws.vpce.us-east-1.vpce-svc-XXXXXX
   ```
   - Check if the consumer account is in the allowed principals list
   - Verify the NLB behind the endpoint service is healthy
   - Check NLB target group health checks

2. **Verify VPC Endpoint (Consumer Side - Application Account):**
   ```bash
   aws ec2 describe-vpc-endpoints \
     --filters "Name=service-name,Values=com.amazonaws.vpce.us-east-1.vpce-svc-XXXXXX"
   ```
   - Confirm state is `available` (not `pendingAcceptance` or `rejected`)
   - Verify the endpoint is in the correct subnets/AZs

3. **Security Group Check:**
   - Consumer endpoint SG must allow inbound from application
   - Provider NLB doesn't have SGs (passthrough), but targets must allow NLB CIDR

4. **DNS Resolution:**
   ```bash
   # Check if private DNS is enabled
   nslookup service.internal.company.com
   
   # If using private DNS name, verify:
   # - Private DNS enabled on endpoint
   # - VPC DNS resolution enabled
   # - Route 53 Private Hosted Zone associated with consumer VPC
   ```

5. **Cross-AZ Issues:**
   - PrivateLink is zonal - endpoint must exist in the same AZ as the consumer
   - AZ IDs (use1-az1) differ from AZ names across accounts

6. **Network ACLs:**
   - NACLs on endpoint subnet must allow ephemeral ports (1024-65535) for return traffic

**Common Production Gotcha:** AZ mapping differs between accounts. `us-east-1a` in Account A might be `use1-az2`, but in Account B it might be `use1-az4`. Always use AZ IDs for PrivateLink endpoint placement.

---

### Q5: How do you implement a centralized DNS architecture across 100+ AWS accounts while supporting both AWS resources and on-premises resolution?

**Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────┐
│          Network Hub Account                 │
│  ┌─────────────────────────────────┐        │
│  │ Route 53 Resolver Inbound       │◄── On-Prem DNS
│  │ Endpoint (10.0.1.10, 10.0.1.11)│        │
│  └─────────────────────────────────┘        │
│  ┌─────────────────────────────────┐        │
│  │ Route 53 Resolver Outbound      │──► On-Prem DNS
│  │ Endpoint                        │        │
│  └─────────────────────────────────┘        │
│  ┌─────────────────────────────────┐        │
│  │ Route 53 Resolver Rules (RAM)   │        │
│  │ - corp.internal → On-Prem       │        │
│  │ - aws.internal → PHZ            │        │
│  └─────────────────────────────────┘        │
│  ┌─────────────────────────────────┐        │
│  │ Private Hosted Zones (shared)   │        │
│  │ - platform.aws.internal         │        │
│  │ - shared.aws.internal           │        │
│  └─────────────────────────────────┘        │
└─────────────────────────────────────────────┘
         │ RAM Share
         ▼
┌──────────────────┐  ┌──────────────────┐
│ Workload Acct A  │  │ Workload Acct B  │
│ PHZ: app-a.internal│  │ PHZ: app-b.internal│
│ (associated with │  │ (associated with │
│  local VPC)      │  │  local VPC)      │
└──────────────────┘  └──────────────────┘
```

**Implementation Steps:**

1. **Centralized Resolver Endpoints** in Network Hub Account VPC
2. **RAM-shared Resolver Rules** to all accounts via AWS Organizations
3. **Private Hosted Zones:**
   - Central PHZs in Network Hub (shared services domains)
   - Application-specific PHZs in workload accounts
   - Cross-account VPC association using authorization

4. **Terraform Module:**
```hcl
module "dns_hub" {
  source = "./modules/dns-hub"
  
  vpc_id              = module.network_hub.vpc_id
  inbound_subnet_ids  = module.network_hub.resolver_subnet_ids
  outbound_subnet_ids = module.network_hub.resolver_subnet_ids
  
  forwarding_rules = {
    "corp.internal" = {
      target_ips = ["10.100.0.53", "10.100.1.53"]  # On-prem DNS
    }
    "legacy.datacenter" = {
      target_ips = ["10.200.0.53"]
    }
  }
  
  ram_share_principals = [data.aws_organizations_organization.current.arn]
}
```

---

### Q6: During a production incident, you discover that AWS Config rules are not evaluating in 3 out of 15 accounts. What's your troubleshooting and remediation approach?

**Answer:**

**Immediate Troubleshooting:**

1. **Check Config Recorder Status:**
   ```bash
   aws configservice describe-configuration-recorder-status
   # Look for: recording=false or lastStatus=FAILURE
   ```

2. **Check Delivery Channel:**
   ```bash
   aws configservice describe-delivery-channel-status
   # Verify S3 bucket exists and IAM role has permissions
   ```

3. **Common Root Causes:**
   - **S3 Bucket Deleted:** Centralized config bucket was modified/deleted
   - **IAM Role Issues:** Config service-linked role missing permissions after SCP change
   - **Regional Mismatch:** Config aggregator expects all regions, but some are disabled
   - **Throttling:** High number of resources causing API throttling

4. **Remediation:**
   ```bash
   # Re-start the recorder
   aws configservice start-configuration-recorder \
     --configuration-recorder-name default
   
   # Verify delivery channel
   aws configservice put-delivery-channel \
     --delivery-channel file://delivery-channel.json
   ```

5. **Prevention - Organization Config:**
   ```hcl
   resource "aws_config_organization_managed_rule" "encryption_check" {
     name            = "encrypted-volumes"
     rule_identifier = "ENCRYPTED_VOLUMES"
     
     # Automatically deploys to all member accounts
     excluded_accounts = []
   }
   ```

6. **Monitoring Config Health:**
   - CloudWatch metric: `ConfigRulesEvaluationStatus`
   - Custom Lambda that checks recorder status across all accounts daily
   - Alert on `ConfigurationRecorderStopped` CloudTrail event

---

### Q7: How would you implement least-privilege access at scale using IAM Identity Center (SSO) for 500+ developers across multiple teams?

**Answer:**

**Architecture:**

```
┌─────────────────────────────────────────┐
│         IAM Identity Center             │
│                                         │
│  Identity Source: Azure AD (SCIM)       │
│                                         │
│  Permission Sets:                       │
│  ├── PlatformAdmin (PowerUser + extras) │
│  ├── DeveloperReadOnly (ViewOnly)       │
│  ├── DeveloperDeploy (CI/CD invoke)     │
│  ├── DataEngineer (Glue, EMR, S3)      │
│  ├── SecurityAuditor (ReadOnly+Sec)    │
│  └── IncidentResponder (time-bound)    │
│                                         │
│  Account Assignments (via Terraform):   │
│  ├── Team-A-Devs → Dev-A Acct (Deploy) │
│  ├── Team-A-Devs → Prod-A Acct (Read)  │
│  └── Platform → All Accts (Admin)      │
└─────────────────────────────────────────┘
```

**Implementation Strategy:**

1. **ABAC (Attribute-Based Access Control):**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["ec2:*"],
         "Resource": "*",
         "Condition": {
           "StringEquals": {
             "aws:PrincipalTag/team": "${aws:ResourceTag/team}"
           }
         }
       }
     ]
   }
   ```

2. **Permission Set Boundaries:**
   - Attach Permission Boundaries to all permission sets
   - Boundary denies: creating IAM users, modifying SCPs, disabling CloudTrail

3. **Temporary Elevated Access:**
   ```python
   # Custom solution using Step Functions
   # Developer requests access → Manager approves → 
   # Lambda assigns permission set → TTL expires → Lambda removes
   
   def grant_temporary_access(event):
       sso_admin = boto3.client('sso-admin')
       sso_admin.create_account_assignment(
           InstanceArn=SSO_INSTANCE_ARN,
           TargetId=event['account_id'],
           TargetType='AWS_ACCOUNT',
           PermissionSetArn=event['permission_set_arn'],
           PrincipalType='USER',
           PrincipalId=event['user_id']
       )
       # Schedule removal via EventBridge
       schedule_removal(event, ttl_hours=4)
   ```

4. **Terraform-managed assignments:**
   ```hcl
   resource "aws_ssoadmin_account_assignment" "team_assignments" {
     for_each = local.team_account_assignments
     
     instance_arn       = tolist(data.aws_ssoadmin_instances.main.arns)[0]
     permission_set_arn = each.value.permission_set_arn
     principal_id       = each.value.group_id
     principal_type     = "GROUP"
     target_id          = each.value.account_id
     target_type        = "AWS_ACCOUNT"
   }
   ```

---

### Q8: You're migrating from a flat single-account setup to a multi-account architecture. How do you migrate 500+ EC2 instances without downtime?

**Answer:**

**Migration Strategy - Phased Approach:**

**Phase 1: Foundation (Weeks 1-4)**
- Deploy Landing Zone with Control Tower
- Set up networking (Transit Gateway, shared VPCs)
- Establish DNS architecture
- Deploy centralized logging and security tooling

**Phase 2: Network Connectivity (Weeks 5-6)**
- Peer/TGW connect legacy account to new accounts
- Ensure DNS resolution works cross-account
- Test connectivity end-to-end

**Phase 3: Migration Waves (Weeks 7-20)**

**Option A - AMI Copy + Re-launch (Recommended for stateless):**
```bash
# Create AMI in source account
aws ec2 create-image --instance-id i-xxx --name "migration-app-v1"

# Share AMI with target account
aws ec2 modify-image-attribute --image-id ami-xxx \
  --launch-permission "Add=[{UserId=TARGET_ACCOUNT_ID}]"

# In target account - launch with same config
aws ec2 run-instances --image-id ami-xxx \
  --instance-type m5.xlarge \
  --subnet-id subnet-NEW
```

**Option B - AWS Application Migration Service (MGN) for live migration:**
```bash
# Install replication agent on source
wget -O ./installer https://aws-application-migration-service-us-east-1.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py
python3 aws-replication-installer-init.py \
  --region us-east-1 \
  --aws-access-key-id KEY \
  --aws-secret-access-key SECRET
```
- Continuous block-level replication to target account
- Test instances before cutover
- Sub-second cutover with DNS switch

**Option C - For databases (stateful):**
- RDS: Cross-account snapshot share → Restore in target
- DynamoDB: Cross-account backup/restore or DMS
- EBS: Cross-account snapshot sharing

**Cutover Pattern:**
```
1. Final sync (MGN) or snapshot
2. Stop writes to source
3. Launch in target account
4. Update Route 53 weighted records (0% → 100%)
5. Monitor for 30 minutes
6. Decommission source (keep for 7 days as rollback)
```

---

### Q9: How do you implement a secure cross-account CI/CD pipeline where the pipeline runs in a Shared Services account but deploys to Production?

**Answer:**

**Architecture:**
```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Shared Services │     │   Staging Acct   │     │  Production Acct │
│                  │     │                  │     │                  │
│  CodePipeline    │────▶│  Deploy Role     │────▶│  Deploy Role     │
│  CodeBuild       │     │  (assumed by     │     │  (assumed by     │
│  Artifact Bucket │     │   pipeline)      │     │   pipeline)      │
│  KMS Key         │     │                  │     │                  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

**Cross-Account IAM Configuration:**

**In Production Account (Trust Policy for Deploy Role):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::SHARED_SERVICES_ACCT:role/CodePipelineRole"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "deploy-pipeline-prod",
          "aws:PrincipalOrgID": "o-xxxxxxxxxx"
        }
      }
    }
  ]
}
```

**KMS Key Policy (for cross-account artifact decryption):**
```json
{
  "Sid": "AllowCrossAccountDecrypt",
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:iam::STAGING_ACCT:role/DeployRole",
      "arn:aws:iam::PROD_ACCT:role/DeployRole"
    ]
  },
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

**S3 Artifact Bucket Policy:**
```json
{
  "Sid": "CrossAccountAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:iam::STAGING_ACCT:role/DeployRole",
      "arn:aws:iam::PROD_ACCT:role/DeployRole"
    ]
  },
  "Action": [
    "s3:GetObject",
    "s3:GetObjectVersion"
  ],
  "Resource": "arn:aws:s3:::pipeline-artifacts-bucket/*"
}
```

**Security Controls:**
- Manual approval stage before production deployment
- Deployment window enforcement (no deploys Friday 5PM - Monday 8AM)
- Automated rollback on CloudWatch alarm trigger
- Deployment audit trail in separate Security account

---

### Q10: Your AWS Organization has 150 accounts and you need to enforce that NO account can launch resources in unapproved regions. However, some global services (IAM, CloudFront, Route53) need to work. How do you implement this?

**Answer:**

**SCP Implementation:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*",
        "budgets:*",
        "ce:*",
        "chime:*",
        "cloudfront:*",
        "cur:*",
        "globalaccelerator:*",
        "health:*",
        "iam:*",
        "importexport:*",
        "kms:*",
        "mobileanalytics:*",
        "organizations:*",
        "pricing:*",
        "route53:*",
        "route53domains:*",
        "s3:GetBucketLocation",
        "s3:ListAllMyBuckets",
        "shield:*",
        "sts:*",
        "support:*",
        "trustedadvisor:*",
        "waf:*",
        "waf-regional:*",
        "wafv2:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2",
            "eu-west-1"
          ]
        }
      }
    }
  ]
}
```

**Production Considerations:**
- Use `NotAction` (not `Action`) to allow global services
- Test with a single OU first before applying to root
- Maintain an exception OU for accounts that legitimately need other regions
- Document the approved regions in a governance wiki
- Use AWS Config rule `approved-regions` as detective control (belt and suspenders)

**Tricky Gotcha:** Some services like S3 replication, Lambda@Edge, and CloudFront require `us-east-1`. If you block it, these break silently. Always include `us-east-1` in approved regions even if your primary region is different.
