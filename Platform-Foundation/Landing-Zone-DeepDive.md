# AWS Landing Zone - Technical Deep Dive

## What is a Landing Zone?

A Landing Zone is a well-architected, multi-account AWS environment that serves as the foundation for your organization's cloud workloads. It provides a baseline of security, governance, networking, and identity management.

## AWS Control Tower vs. Custom Landing Zone

| Feature | AWS Control Tower | Custom Landing Zone |
|---------|------------------|-------------------|
| Setup Time | Hours | Weeks/Months |
| Customization | Limited (AFT extends) | Unlimited |
| Guardrails | Pre-built | Custom SCPs/Config Rules |
| Account Factory | Built-in (AFT) | Custom (Lambda/Step Functions) |
| Drift Detection | Automatic | Custom monitoring |
| Best For | Greenfield, <100 accounts | Complex enterprises, existing orgs |

## Control Tower Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Management Account                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │Control Tower │  │ AWS Orgs     │  │ CloudFormation    │  │
│  │ Dashboard    │  │ (SCPs)       │  │ StackSets        │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Log Archive    │  │  Audit Account  │  │  AFT Account    │
│  Account        │  │  (Security)     │  │  (Account       │
│                 │  │                 │  │   Factory)      │
│  - CloudTrail   │  │  - Config Aggr  │  │  - CodePipeline │
│  - Config logs  │  │  - GuardDuty    │  │  - Terraform    │
│  - VPC Flow     │  │  - Security Hub │  │  - Customizations│
│  - S3 access    │  │  - Access Anlzr │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

## Account Factory for Terraform (AFT)

AFT automates account provisioning with customizations:

### Directory Structure:
```
aft-account-request/
├── terraform/
│   └── main.tf           # Account request definitions
aft-global-customizations/
├── terraform/
│   └── main.tf           # Applied to ALL accounts
aft-account-customizations/
├── terraform/
│   ├── TEAM-A-DEV/
│   │   └── main.tf      # Team A specific customizations
│   └── PCI-PROD/
│       └── main.tf      # PCI environment specific
aft-account-provisioning-customizations/
└── terraform/
    └── main.tf           # Run during account creation
```

### Account Request Example:
```hcl
module "team_a_dev" {
  source = "./modules/aft-account-request"
  
  control_tower_parameters = {
    AccountEmail              = "team-a-dev@company.com"
    AccountName              = "team-a-dev"
    ManagedOrganizationalUnit = "Workloads/Development"
    SSOUserEmail             = "platform-team@company.com"
    SSOUserFirstName         = "Platform"
    SSOUserLastName          = "Admin"
  }
  
  account_tags = {
    Environment = "Development"
    Team        = "Team-A"
    CostCenter  = "CC-1234"
    Compliance  = "SOC2"
  }
  
  account_customizations_name = "TEAM-A-DEV"
  
  change_management_parameters = {
    change_requested_by = "platform-team"
    change_reason       = "New development environment for Team A"
  }
}
```

## Guardrails Deep Dive

### Types of Guardrails:

1. **Preventive (SCPs):** Block non-compliant actions before they happen
2. **Detective (AWS Config Rules):** Detect non-compliant resources after creation
3. **Proactive (CloudFormation Hooks):** Block non-compliant resources during deployment

### Critical Production Guardrails:

```json
// Preventive: Deny root user access
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyRootUser",
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringLike": {
        "aws:PrincipalArn": "arn:aws:iam::*:root"
      }
    }
  }]
}
```

```json
// Preventive: Deny disabling CloudTrail
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyCloudTrailModification",
    "Effect": "Deny",
    "Action": [
      "cloudtrail:StopLogging",
      "cloudtrail:DeleteTrail",
      "cloudtrail:UpdateTrail"
    ],
    "Resource": "*",
    "Condition": {
      "StringNotLike": {
        "aws:PrincipalARN": "arn:aws:iam::*:role/AWSControlTowerExecution"
      }
    }
  }]
}
```

```json
// Preventive: Deny leaving the organization
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyLeaveOrg",
    "Effect": "Deny",
    "Action": "organizations:LeaveOrganization",
    "Resource": "*"
  }]
}
```

## Landing Zone Networking Patterns

### Pattern 1: Centralized Egress

```
Internet
    │
    ▼
┌──────────────────┐
│  Network Hub     │
│  ┌────────────┐  │
│  │ NAT GW    │  │
│  │ (shared)  │  │
│  └────────────┘  │
│  ┌────────────┐  │
│  │ NW Firewall│  │
│  └────────────┘  │
└──────────────────┘
    │ TGW
    ├────────────────┐
    ▼                ▼
┌──────────┐  ┌──────────┐
│ Acct A   │  │ Acct B   │
│ (no NAT) │  │ (no NAT) │
│ (no IGW) │  │ (no IGW) │
└──────────┘  └──────────┘
```

**Benefits:**
- Single point of egress inspection (Network Firewall)
- Cost optimization (shared NAT Gateways)
- Consistent logging and monitoring
- Easy to implement allowlist/denylist

**Route Table Configuration (Workload VPC):**
```
Destination     | Target
0.0.0.0/0       | Transit Gateway
10.0.0.0/8      | Transit Gateway
VPC CIDR        | Local
```

**Route Table Configuration (Network Hub - Firewall Subnet):**
```
Destination     | Target
0.0.0.0/0       | NAT Gateway
10.0.0.0/8      | Transit Gateway
VPC CIDR        | Local
```

### Pattern 2: Centralized Ingress with ALB

```
Internet
    │
    ▼
┌──────────────────────────┐
│  Shared Ingress Account  │
│  ┌────────────────────┐  │
│  │  AWS WAF + Shield  │  │
│  └────────────────────┘  │
│  ┌────────────────────┐  │
│  │  ALB (shared)      │  │
│  │  - Host routing    │  │
│  │  - Path routing    │  │
│  └────────────────────┘  │
└──────────────────────────┘
    │ PrivateLink / TGW
    ├─────────────────────┐
    ▼                     ▼
┌──────────────┐  ┌──────────────┐
│ App A Acct   │  │ App B Acct   │
│ NLB (target) │  │ NLB (target) │
└──────────────┘  └──────────────┘
```

### Pattern 3: Decentralized with Guardrails

Each account has its own VPC with internet access, controlled by:
- AWS Firewall Manager policies (WAF rules pushed to all accounts)
- Network Firewall rules (inspected via TGW)
- SCPs preventing public S3 buckets, public RDS, etc.

## IPAM (IP Address Management)

```hcl
resource "aws_vpc_ipam" "main" {
  operating_regions {
    region_name = "us-east-1"
  }
  operating_regions {
    region_name = "us-west-2"
  }
}

resource "aws_vpc_ipam_pool" "top_level" {
  address_family = "ipv4"
  ipam_scope_id  = aws_vpc_ipam.main.private_default_scope_id
  locale         = "us-east-1"
}

resource "aws_vpc_ipam_pool_cidr" "top_level" {
  ipam_pool_id = aws_vpc_ipam_pool.top_level.id
  cidr         = "10.0.0.0/8"
}

# Production pool - /12 allocation
resource "aws_vpc_ipam_pool" "production" {
  address_family      = "ipv4"
  ipam_scope_id       = aws_vpc_ipam.main.private_default_scope_id
  source_ipam_pool_id = aws_vpc_ipam_pool.top_level.id
  locale              = "us-east-1"
  
  allocation_default_netmask_length = 22
  allocation_min_netmask_length     = 20
  allocation_max_netmask_length     = 24
  
  tags = {
    Environment = "Production"
  }
}

resource "aws_vpc_ipam_pool_cidr" "production" {
  ipam_pool_id = aws_vpc_ipam_pool.production.id
  cidr         = "10.0.0.0/12"
}
```

## Landing Zone Monitoring & Compliance

### Organization-wide CloudTrail:
```hcl
resource "aws_cloudtrail" "org_trail" {
  name                          = "org-management-trail"
  s3_bucket_name                = aws_s3_bucket.trail_bucket.id
  is_organization_trail         = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  include_global_service_events = true
  kms_key_id                    = aws_kms_key.trail_key.arn
  
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.trail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.trail_role.arn
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3"]
    }
  }
  
  insight_selector {
    insight_type = "ApiCallRateInsight"
  }
  
  insight_selector {
    insight_type = "ApiErrorRateInsight"
  }
}
```

### Security Hub Aggregation:
```hcl
# In Delegated Admin Account
resource "aws_securityhub_organization_admin_account" "main" {
  admin_account_id = var.security_account_id
}

resource "aws_securityhub_organization_configuration" "main" {
  auto_enable           = true
  auto_enable_standards = "DEFAULT"
  
  organization_configuration {
    configuration_type = "CENTRAL"
  }
}

resource "aws_securityhub_finding_aggregator" "main" {
  linking_mode = "ALL_REGIONS"
}
```

## Best Practices Checklist

- [ ] Enable CloudTrail in all regions with organization trail
- [ ] Configure AWS Config with organization-wide rules
- [ ] Implement SCPs before provisioning workload accounts
- [ ] Use IPAM for CIDR management across accounts
- [ ] Centralize VPC Flow Logs in Log Archive account
- [ ] Enable GuardDuty with delegated admin in Security account
- [ ] Configure Security Hub with CIS/PCI-DSS standards
- [ ] Implement centralized DNS with Route 53 Resolver
- [ ] Deploy Network Firewall for egress inspection
- [ ] Use Service Catalog / AFT for account provisioning
- [ ] Establish tagging strategy and enforce with Tag Policies
- [ ] Configure backup policies at Organization level
