# Multi-Account Strategy - Enterprise Patterns & Guardrails

## Why Multi-Account?

| Concern | Single Account | Multi-Account |
|---------|---------------|---------------|
| Blast Radius | Entire org affected | Isolated per account |
| Security | Flat network | Account-level isolation |
| Billing | Complex tagging | Natural cost separation |
| Service Limits | Shared across teams | Per-account limits |
| Compliance | Difficult to audit | Clear scope boundaries |
| Autonomy | Conflicts between teams | Teams own their accounts |

## Account Types & Purposes

### 1. Management Account
- **Purpose:** AWS Organizations root, billing consolidation
- **Workloads:** NONE (only org management)
- **Access:** Restricted to <5 platform administrators
- **Key Services:** Organizations, Control Tower, Billing

### 2. Security Accounts

#### Log Archive Account
```
Purpose: Immutable log storage
Services: S3 (Object Lock), CloudWatch Logs (cross-account)
Access: Read-only for security team, no delete access

S3 Bucket Configuration:
- Object Lock: Governance Mode (365 days)
- Versioning: Enabled
- Lifecycle: Transition to Glacier after 90 days
- Replication: Cross-region for DR
```

#### Security Tooling Account
```
Purpose: Centralized security operations
Services: 
  - GuardDuty (delegated admin)
  - Security Hub (delegated admin)
  - Detective
  - Inspector
  - Access Analyzer (organization-level)
  - Macie (delegated admin)
Access: Security team only
```

#### Forensics Account
```
Purpose: Incident response isolation
Services: EC2 (for analysis), EBS snapshots, VPC (isolated)
Access: Incident responders only, MFA required
Network: No internet, no peering to production
```

### 3. Infrastructure Accounts

#### Network Hub Account
```
Purpose: Centralized networking
Services:
  - Transit Gateway (owner)
  - Route 53 (Resolver endpoints, Private Hosted Zones)
  - Network Firewall
  - Direct Connect / VPN
  - NAT Gateways (shared)
  - IPAM
```

#### Shared Services Account
```
Purpose: Common tools and services
Services:
  - Active Directory
  - CI/CD Pipelines (CodePipeline, Jenkins)
  - Artifact Repositories (ECR, CodeArtifact)
  - Secrets Manager (shared secrets)
  - Systems Manager
```

#### Backup Account
```
Purpose: Centralized backup vault
Services:
  - AWS Backup (organization policy)
  - Cross-account vault
  - Immutable backups
```

### 4. Workload Accounts (per team/application)
```
Purpose: Application hosting
Pattern: Separate accounts per environment (Dev/Staging/Prod)
Naming: {team}-{app}-{env} (e.g., payments-api-prod)
```

## Organizational Unit (OU) Hierarchy

```
Root (Management Account)
│
├── Security OU
│   ├── Log Archive
│   ├── Security Tooling
│   └── Forensics
│
├── Infrastructure OU
│   ├── Network Hub
│   ├── Shared Services
│   └── Backup
│
├── Workloads OU
│   ├── Production OU
│   │   ├── Team-A-Prod
│   │   ├── Team-B-Prod
│   │   └── ...
│   ├── Non-Production OU
│   │   ├── Development Sub-OU
│   │   │   ├── Team-A-Dev
│   │   │   └── Team-B-Dev
│   │   └── Staging Sub-OU
│   │       ├── Team-A-Staging
│   │       └── Team-B-Staging
│   └── PCI OU (if applicable)
│       ├── PCI-Prod
│       └── PCI-NonProd
│
├── Sandbox OU
│   └── Individual developer accounts
│
├── Deployments OU
│   └── CI/CD deployment accounts
│
├── Transitional OU
│   └── Accounts being migrated
│
└── Suspended OU
    └── Decommissioned accounts
```

## SCP Strategy by OU

### Root-Level SCPs (Applied to ALL):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeavingOrg",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    },
    {
      "Sid": "ProtectCloudTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "arn:aws:cloudtrail:*:*:trail/org-*"
    },
    {
      "Sid": "DenyRootActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

### Production OU SCPs:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyPublicS3",
      "Effect": "Deny",
      "Action": "s3:PutBucketPublicAccessBlock",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:PublicAccessBlockConfiguration/BlockPublicAcls": "true"
        }
      }
    },
    {
      "Sid": "RequireIMDSv2",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:MetadataHttpTokens": "required"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedEBS",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:volume/*",
      "Condition": {
        "Bool": {
          "ec2:Encrypted": "false"
        }
      }
    },
    {
      "Sid": "DenyPublicRDS",
      "Effect": "Deny",
      "Action": [
        "rds:CreateDBInstance",
        "rds:ModifyDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "rds:PubliclyAccessible": "true"
        }
      }
    }
  ]
}
```

### Sandbox OU SCPs:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LimitInstanceTypes",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "ec2:InstanceType": [
            "t3.*",
            "t3a.*",
            "t4g.*"
          ]
        }
      }
    },
    {
      "Sid": "DenyExpensiveServices",
      "Effect": "Deny",
      "Action": [
        "redshift:*",
        "emr:*",
        "es:*",
        "sagemaker:CreateNotebookInstance"
      ],
      "Resource": "*"
    },
    {
      "Sid": "LimitRegions",
      "Effect": "Deny",
      "NotAction": ["iam:*", "sts:*", "organizations:*"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1"]
        }
      }
    }
  ]
}
```

## Tagging Strategy

### Mandatory Tags (enforced via Tag Policies):
```json
{
  "tags": {
    "Environment": {
      "tag_key": { "@@assign": "Environment" },
      "tag_value": {
        "@@assign": ["Production", "Staging", "Development", "Sandbox"]
      },
      "enforced_for": {
        "@@assign": [
          "ec2:instance",
          "rds:db",
          "s3:bucket",
          "lambda:function"
        ]
      }
    },
    "CostCenter": {
      "tag_key": { "@@assign": "CostCenter" },
      "tag_value": {
        "@@assign": ["CC-*"]
      }
    },
    "Team": {
      "tag_key": { "@@assign": "Team" }
    },
    "DataClassification": {
      "tag_key": { "@@assign": "DataClassification" },
      "tag_value": {
        "@@assign": ["Public", "Internal", "Confidential", "Restricted"]
      }
    }
  }
}
```

## Account Vending Machine

### Automated Account Creation Flow:
```
Developer Request (ServiceNow/Jira)
    │
    ▼
Approval Workflow (Manager + Security)
    │
    ▼
AFT/Custom Pipeline Triggered
    │
    ├── Create Account (Organizations API)
    ├── Move to correct OU
    ├── Apply baseline (CloudFormation StackSets)
    │   ├── VPC with standard CIDR (from IPAM)
    │   ├── IAM roles (break-glass, deploy, read-only)
    │   ├── Config rules
    │   ├── GuardDuty member
    │   ├── Security Hub member
    │   ├── CloudWatch cross-account sharing
    │   └── Tagging enforcement
    ├── Configure networking (TGW attachment)
    ├── Assign SSO permission sets
    ├── Create cost allocation tags
    └── Notify team (Slack/Email)
```

## Cost Management at Scale

### Organization-Level Controls:
```hcl
# Budget per account (deployed via StackSets)
resource "aws_budgets_budget" "account" {
  name         = "account-monthly-budget"
  budget_type  = "COST"
  limit_amount = var.budget_limit
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type           = "PERCENTAGE"
    notification_type        = "ACTUAL"
    subscriber_email_addresses = [var.team_email]
  }
  
  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type           = "PERCENTAGE"
    notification_type        = "ACTUAL"
    subscriber_email_addresses = [var.finance_email, var.team_email]
  }
}
```

### Savings Plans Strategy:
- Compute Savings Plans (organization-level) for EC2, Fargate, Lambda
- Assign to specific accounts using Cost Allocation Tags
- Review utilization monthly via Cost Explorer

## Account Lifecycle Management

### Decommissioning Process:
1. Move account to Suspended OU (restricts all actions)
2. Export/backup all data
3. Delete all resources (automated script)
4. Wait 90 days (compliance retention)
5. Close account via Organizations API

### Quarantine Process (compromised account):
1. Move to Suspended OU immediately
2. Revoke all active sessions (IAM)
3. Disable all access keys
4. Isolate network (remove TGW attachment)
5. Snapshot all EBS volumes for forensics
6. Engage security team for investigation
