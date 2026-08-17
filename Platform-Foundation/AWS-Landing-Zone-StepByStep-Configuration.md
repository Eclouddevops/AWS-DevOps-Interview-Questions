# AWS Landing Zone — Step-by-Step Configuration Guide

> **Purpose**: Complete step-by-step instructions to set up an enterprise AWS Landing Zone from scratch using AWS Control Tower, including multi-account structure, networking, security, and governance.

---

## Table of Contents

1. [Prerequisites & Planning](#1-prerequisites--planning)
2. [Step 1: Set Up AWS Organizations](#step-1-set-up-aws-organizations)
3. [Step 2: Deploy AWS Control Tower](#step-2-deploy-aws-control-tower)
4. [Step 3: Configure Organizational Units (OUs)](#step-3-configure-organizational-units)
5. [Step 4: Set Up IAM Identity Center (SSO)](#step-4-set-up-iam-identity-center)
6. [Step 5: Configure Service Control Policies (SCPs)](#step-5-configure-service-control-policies)
7. [Step 6: Set Up Centralized Networking](#step-6-set-up-centralized-networking)
8. [Step 7: Configure Security Services](#step-7-configure-security-services)
9. [Step 8: Set Up Centralized Logging](#step-8-set-up-centralized-logging)
10. [Step 9: Account Factory & Vending](#step-9-account-factory--vending)
11. [Step 10: Implement Guardrails & Compliance](#step-10-implement-guardrails--compliance)
12. [Post-Setup: Day 2 Operations](#post-setup-day-2-operations)

---

## 1. Prerequisites & Planning

### Before You Start

```
CHECKLIST:
├── [ ] Dedicated email for Management Account (aws-root@company.com — distribution list!)
├── [ ] Dedicated email for Log Archive Account (aws-logarchive@company.com)
├── [ ] Dedicated email for Audit Account (aws-audit@company.com)
├── [ ] Credit card for billing (consolidated billing enabled)
├── [ ] MFA device (hardware YubiKey recommended for root)
├── [ ] IP address plan (CIDR allocation for all VPCs)
├── [ ] Compliance requirements identified (PCI, HIPAA, SOC2?)
├── [ ] Team structure documented (who needs access to what?)
├── [ ] Cost budget approved
└── [ ] Corporate directory info (Azure AD, Okta, etc.) for SSO
```

### Planning Decisions

```yaml
decisions_to_make_upfront:
  regions:
    primary: "us-east-1"
    secondary: "us-west-2"  # For DR
    additional: "eu-west-1"  # If needed for compliance
    blocked: "All other regions (via SCP)"
    
  cidr_allocation:
    total_range: "10.0.0.0/8"
    production: "10.0.0.0/12"    # 10.0.0.0 - 10.15.255.255
    non_production: "10.16.0.0/12" # 10.16.0.0 - 10.31.255.255
    shared_services: "10.32.0.0/16"
    network_hub: "10.33.0.0/16"
    sandbox: "10.100.0.0/16"
    
  account_email_pattern: "aws-{account-name}@company.com"
  
  naming_convention:
    accounts: "{team}-{environment}"  # e.g., payments-prod, data-dev
    resources: "{env}-{service}-{purpose}"  # e.g., prod-vpc-main
    
  governance:
    scp_strategy: "deny-list"  # Allow all, deny specific dangerous actions
    guardrail_level: "mandatory + strongly-recommended"
    compliance_frameworks: ["CIS", "SOC2"]
```

---

## Step 1: Set Up AWS Organizations

### 1.1 Create the Organization

```
CONSOLE STEPS:
1. Log in to AWS Console with the account that will be Management Account
2. Go to: AWS Organizations → Create organization
3. Choose: "Create organization" (enables all features including SCPs)
4. Verify email (check root email for verification link)

AWS CLI (if account already exists):
```

```bash
# Create organization (run from Management Account)
aws organizations create-organization --feature-set ALL

# Verify organization created
aws organizations describe-organization
# Output shows: Id, Arn, MasterAccountId, AvailableFeatures
```

### 1.2 Enable Required Organization Services

```bash
# Enable trusted access for services we'll use
aws organizations enable-aws-service-access --service-principal cloudtrail.amazonaws.com
aws organizations enable-aws-service-access --service-principal config.amazonaws.com
aws organizations enable-aws-service-access --service-principal guardduty.amazonaws.com
aws organizations enable-aws-service-access --service-principal securityhub.amazonaws.com
aws organizations enable-aws-service-access --service-principal ram.amazonaws.com
aws organizations enable-aws-service-access --service-principal sso.amazonaws.com
aws organizations enable-aws-service-access --service-principal backup.amazonaws.com
aws organizations enable-aws-service-access --service-principal macie.amazonaws.com
aws organizations enable-aws-service-access --service-principal access-analyzer.amazonaws.com
aws organizations enable-aws-service-access --service-principal stacksets.cloudformation.amazonaws.com
aws organizations enable-aws-service-access --service-principal ipam.amazonaws.com

# Verify
aws organizations list-aws-service-access-for-organization
```

### 1.3 Enable Policy Types

```bash
# Enable Service Control Policies (SCPs)
aws organizations enable-policy-type \
  --root-id r-xxxx \
  --policy-type SERVICE_CONTROL_POLICY

# Enable Tag Policies
aws organizations enable-policy-type \
  --root-id r-xxxx \
  --policy-type TAG_POLICY

# Enable Backup Policies
aws organizations enable-policy-type \
  --root-id r-xxxx \
  --policy-type BACKUP_POLICY

# Verify all enabled
aws organizations list-roots
# Look for: "PolicyTypes": [{"Type": "SERVICE_CONTROL_POLICY", "Status": "ENABLED"}, ...]
```

---

## Step 2: Deploy AWS Control Tower

### 2.1 Prerequisites for Control Tower

```
BEFORE launching Control Tower:
├── [ ] Management Account must NOT already have Config enabled
├── [ ] Management Account must NOT have CloudTrail org trail (CT creates one)
├── [ ] No existing StackSets with same names CT will create
├── [ ] us-east-1 region available (CT uses this for some resources)
├── [ ] Organization already created (Step 1)
└── [ ] Account has AdministratorAccess or equivalent
```

### 2.2 Launch Control Tower

```
CONSOLE STEPS:
1. Go to: AWS Control Tower → "Set up landing zone"
2. Review pricing (Control Tower itself is FREE)
3. Configure home region: us-east-1 (or your primary)
4. Additional regions: Add secondary regions (us-west-2, eu-west-1)
5. OU configuration:
   └── Foundation OU name: "Security" (auto-created)
   └── Additional OU: "Sandbox" (auto-created)
6. Log Archive account:
   └── Email: aws-logarchive@company.com
   └── Name: "Log Archive"
7. Audit account:
   └── Email: aws-audit@company.com
   └── Name: "Audit"
8. Additional configuration:
   └── AWS CloudTrail: Enable (organization trail)
   └── S3 log bucket retention: 365 days
   └── S3 access log bucket retention: 365 days
   └── KMS encryption: Enable (custom key recommended)
9. Review and "Set up landing zone"
   └── Takes approximately 60 minutes to complete

WHAT GETS CREATED:
├── Organization structure with Security OU + Sandbox OU
├── Log Archive account (CloudTrail logs, Config logs, S3 access logs)
├── Audit account (cross-account audit roles, SNS topics)
├── Organization CloudTrail (all accounts, all regions)
├── AWS Config enabled in all accounts
├── IAM Identity Center (SSO) enabled
├── Default guardrails applied (20+ mandatory)
├── CloudFormation StackSets for baseline deployment
└── Landing Zone dashboard (compliance view)
```

### 2.3 Verify Control Tower Deployment

```bash
# Check Control Tower status
aws controltower list-landing-zones

# Check enabled guardrails
aws controltower list-enabled-controls \
  --target-identifier "arn:aws:organizations::MGMT_ACCOUNT:ou/o-xxx/ou-xxx"

# Check account status
aws controltower list-enrolled-accounts
```

```
POST-DEPLOYMENT VERIFICATION:
├── [ ] Log Archive account created and accessible
├── [ ] Audit account created and accessible
├── [ ] CloudTrail logging to Log Archive S3 bucket
├── [ ] Config enabled in Management, Log Archive, and Audit accounts
├── [ ] SSO portal accessible (https://yourdomain.awsapps.com/start)
├── [ ] Guardrails showing "Enabled" in dashboard
├── [ ] No drift detected in dashboard
└── [ ] Can create new account via Account Factory
```

---

## Step 3: Configure Organizational Units

### 3.1 Create OU Structure

```bash
# Get Root ID
ROOT_ID=$(aws organizations list-roots --query 'Roots[0].Id' --output text)

# Create Infrastructure OU
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Infrastructure"

# Create Workloads OU
WORKLOADS_OU=$(aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Workloads" \
  --query 'OrganizationalUnit.Id' --output text)

# Create sub-OUs under Workloads
aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU \
  --name "Production"

aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU \
  --name "Non-Production"

# Create Suspended OU (for decommissioned accounts)
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Suspended"

# Verify structure
aws organizations list-organizational-units-for-parent --parent-id $ROOT_ID
```

### 3.2 Final OU Structure

```
Root
├── Security OU (created by Control Tower)
│   ├── Log Archive Account
│   └── Audit Account
├── Infrastructure OU (created in 3.1)
│   ├── Network Hub Account (create later)
│   ├── Shared Services Account (create later)
│   └── Backup Account (create later)
├── Workloads OU (created in 3.1)
│   ├── Production Sub-OU
│   │   └── (team accounts go here)
│   └── Non-Production Sub-OU
│       └── (dev/staging accounts go here)
├── Sandbox OU (created by Control Tower)
│   └── (developer sandbox accounts)
└── Suspended OU (created in 3.1)
    └── (decommissioned accounts moved here)
```

### 3.3 Register OUs with Control Tower

```
CONSOLE:
1. AWS Control Tower → Organization → Register OU
2. Select each new OU → "Register"
3. Control Tower deploys baseline (Config, CloudTrail) to new OUs
4. Guardrails are applied based on OU registration

NOTE: This step is important! Unregistered OUs don't get CT guardrails.
```

---

## Step 4: Set Up IAM Identity Center

### 4.1 Configure Identity Source

```
CONSOLE STEPS:
1. Go to: IAM Identity Center → Settings → Identity source
2. Choose source:
   ├── Option A: Identity Center directory (simple, small teams)
   ├── Option B: Active Directory (AWS Managed Microsoft AD)
   └── Option C: External Identity Provider (Azure AD, Okta) — RECOMMENDED
3. For External IdP (Azure AD):
   a. Download SAML metadata from IAM Identity Center
   b. In Azure AD → Enterprise Applications → New Application → AWS SSO
   c. Upload SAML metadata
   d. Configure SCIM provisioning (auto-sync users/groups)
   e. Map attributes (email, firstName, lastName, groups)
   f. Enable provisioning
   g. Back in IAM Identity Center: Enable automatic provisioning
```

### 4.2 Create Permission Sets

```bash
# Create Read-Only permission set
aws sso-admin create-permission-set \
  --instance-arn $SSO_INSTANCE_ARN \
  --name "ReadOnlyAccess" \
  --description "Read-only access for viewing resources" \
  --session-duration "PT8H" \
  --managed-policies '[{"Arn":"arn:aws:iam::aws:policy/ReadOnlyAccess"}]'

# Create Developer permission set (PowerUser without IAM changes)
aws sso-admin create-permission-set \
  --instance-arn $SSO_INSTANCE_ARN \
  --name "DeveloperAccess" \
  --description "Developer access - PowerUser without IAM" \
  --session-duration "PT8H" \
  --managed-policies '[{"Arn":"arn:aws:iam::aws:policy/PowerUserAccess"}]'

# Create Admin permission set (full access — restricted to break-glass)
aws sso-admin create-permission-set \
  --instance-arn $SSO_INSTANCE_ARN \
  --name "AdministratorAccess" \
  --description "Full admin - break-glass only" \
  --session-duration "PT1H" \
  --managed-policies '[{"Arn":"arn:aws:iam::aws:policy/AdministratorAccess"}]'

# Create Platform Engineer permission set
aws sso-admin create-permission-set \
  --instance-arn $SSO_INSTANCE_ARN \
  --name "PlatformEngineerAccess" \
  --description "Platform team - infra management" \
  --session-duration "PT8H"

# Attach inline policy to Platform Engineer
aws sso-admin put-inline-policy-to-permission-set \
  --instance-arn $SSO_INSTANCE_ARN \
  --permission-set-arn $PLATFORM_PS_ARN \
  --inline-policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ec2:*", "ecs:*", "eks:*", "rds:*", "s3:*",
          "lambda:*", "cloudformation:*", "iam:Get*", "iam:List*",
          "iam:PassRole", "logs:*", "cloudwatch:*", "ssm:*"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Deny",
        "Action": [
          "iam:CreateUser", "iam:DeleteUser",
          "iam:CreateAccessKey", "organizations:*"
        ],
        "Resource": "*"
      }
    ]
  }'
```

### 4.3 Assign Permission Sets to Accounts

```bash
# Assign "DeveloperAccess" to "Developers" group for Dev account
aws sso-admin create-account-assignment \
  --instance-arn $SSO_INSTANCE_ARN \
  --target-id "DEV_ACCOUNT_ID" \
  --target-type AWS_ACCOUNT \
  --permission-set-arn $DEVELOPER_PS_ARN \
  --principal-type GROUP \
  --principal-id $DEVELOPERS_GROUP_ID

# Assign "ReadOnlyAccess" to "Developers" group for Production account
aws sso-admin create-account-assignment \
  --instance-arn $SSO_INSTANCE_ARN \
  --target-id "PROD_ACCOUNT_ID" \
  --target-type AWS_ACCOUNT \
  --permission-set-arn $READONLY_PS_ARN \
  --principal-type GROUP \
  --principal-id $DEVELOPERS_GROUP_ID

# Assign "PlatformEngineerAccess" to "Platform" group for ALL accounts
for ACCOUNT_ID in $(aws organizations list-accounts --query 'Accounts[].Id' --output text); do
  aws sso-admin create-account-assignment \
    --instance-arn $SSO_INSTANCE_ARN \
    --target-id "$ACCOUNT_ID" \
    --target-type AWS_ACCOUNT \
    --permission-set-arn $PLATFORM_PS_ARN \
    --principal-type GROUP \
    --principal-id $PLATFORM_GROUP_ID
done
```

### 4.4 Verify SSO Access

```
VERIFICATION:
1. Go to SSO portal: https://yourdomain.awsapps.com/start
2. Login with corporate credentials
3. Verify: Correct accounts visible with correct permission sets
4. Test: Click "Management Console" → lands in correct account with correct role
5. Test: CLI access via `aws sso login --profile dev`
```

---

## Step 5: Configure Service Control Policies

### 5.1 Deny Unapproved Regions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*", "budgets:*", "ce:*", "chime:*", "cloudfront:*",
        "cur:*", "globalaccelerator:*", "health:*", "iam:*",
        "importexport:*", "kms:*", "mobileanalytics:*",
        "organizations:*", "pricing:*", "route53:*",
        "route53domains:*", "s3:GetBucketLocation",
        "s3:ListAllMyBuckets", "shield:*", "sts:*",
        "support:*", "trustedadvisor:*", "waf:*",
        "waf-regional:*", "wafv2:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
        }
      }
    }
  ]
}
```

```bash
# Create and attach SCP
aws organizations create-policy \
  --name "deny-unapproved-regions" \
  --description "Restrict resources to approved regions only" \
  --type SERVICE_CONTROL_POLICY \
  --content file://scps/deny-unapproved-regions.json

# Attach to Root (applies to ALL accounts except Management)
aws organizations attach-policy \
  --policy-id p-xxxxxxxx \
  --target-id $ROOT_ID
```

### 5.2 Protect Security Baseline

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectCloudTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalARN": [
            "arn:aws:iam::*:role/AWSControlTowerExecution",
            "arn:aws:iam::*:role/stacksets-exec-*"
          ]
        }
      }
    },
    {
      "Sid": "ProtectConfigRecorder",
      "Effect": "Deny",
      "Action": [
        "config:StopConfigurationRecorder",
        "config:DeleteConfigurationRecorder",
        "config:DeleteDeliveryChannel"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyLeavingOrganization",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    },
    {
      "Sid": "DenyRootUserActions",
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

### 5.3 Enforce Encryption

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedS3",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": ["aws:kms", "AES256"]
        },
        "Null": {
          "s3:x-amz-server-side-encryption": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedEBS",
      "Effect": "Deny",
      "Action": "ec2:CreateVolume",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "ec2:Encrypted": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedRDS",
      "Effect": "Deny",
      "Action": ["rds:CreateDBInstance", "rds:CreateDBCluster"],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "rds:StorageEncrypted": "false"
        }
      }
    },
    {
      "Sid": "EnforceIMDSv2",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:MetadataHttpTokens": "required"
        }
      }
    }
  ]
}
```

### 5.4 Sandbox OU Restrictions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyExpensiveServices",
      "Effect": "Deny",
      "Action": [
        "redshift:*", "es:*", "emr:*",
        "sagemaker:CreateNotebookInstance",
        "ec2:RunInstances"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "ec2:InstanceType": ["t3.*", "t3a.*", "t4g.*", "m5.large"]
        }
      }
    },
    {
      "Sid": "DenyNetworkChanges",
      "Effect": "Deny",
      "Action": [
        "ec2:CreateVpc", "ec2:CreateVpnConnection",
        "ec2:CreateTransitGateway*", "directconnect:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### 5.5 Apply SCPs to Correct OUs

```bash
# Production OU: Region + Encryption + Security baseline
aws organizations attach-policy --policy-id $REGION_POLICY --target-id $PROD_OU_ID
aws organizations attach-policy --policy-id $ENCRYPT_POLICY --target-id $PROD_OU_ID
aws organizations attach-policy --policy-id $SECURITY_POLICY --target-id $PROD_OU_ID

# Sandbox OU: All of above + Sandbox restrictions
aws organizations attach-policy --policy-id $SANDBOX_POLICY --target-id $SANDBOX_OU_ID

# Root level: Security baseline (applies everywhere)
aws organizations attach-policy --policy-id $SECURITY_POLICY --target-id $ROOT_ID
```

---

## Step 6: Set Up Centralized Networking

### 6.1 Create Network Hub Account

```bash
# Create account via Control Tower Account Factory
# CONSOLE: Control Tower → Account Factory → Create account
# Name: "Network-Hub"
# Email: aws-network-hub@company.com
# OU: Infrastructure

# OR via CLI (Organizations direct):
aws organizations create-account \
  --account-name "Network-Hub" \
  --email "aws-network-hub@company.com"

# Move to Infrastructure OU
aws organizations move-account \
  --account-id $NETWORK_ACCOUNT_ID \
  --source-parent-id $ROOT_ID \
  --destination-parent-id $INFRA_OU_ID
```

### 6.2 Deploy Transit Gateway

```hcl
# In Network Hub Account — Terraform
# transit-gateway.tf

resource "aws_ec2_transit_gateway" "main" {
  description                     = "Central Transit Gateway"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  auto_accept_shared_attachments  = "enable"
  dns_support                     = "enable"
  vpn_ecmp_support               = "enable"
  
  tags = {
    Name = "central-tgw"
  }
}

# Share TGW with Organization via RAM
resource "aws_ram_resource_share" "tgw_share" {
  name                      = "transit-gateway-share"
  allow_external_principals = false
}

resource "aws_ram_resource_association" "tgw" {
  resource_arn       = aws_ec2_transit_gateway.main.arn
  resource_share_arn = aws_ram_resource_share.tgw_share.arn
}

resource "aws_ram_principal_association" "org" {
  principal          = data.aws_organizations_organization.current.arn
  resource_share_arn = aws_ram_resource_share.tgw_share.arn
}

# Route Tables (segmented)
resource "aws_ec2_transit_gateway_route_table" "production" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "tgw-rt-production" }
}

resource "aws_ec2_transit_gateway_route_table" "non_production" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "tgw-rt-non-production" }
}

resource "aws_ec2_transit_gateway_route_table" "shared_services" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  tags               = { Name = "tgw-rt-shared-services" }
}

# Blackhole route: Prod CANNOT reach Non-Prod
resource "aws_ec2_transit_gateway_route" "prod_blackhole_dev" {
  destination_cidr_block         = "10.16.0.0/12"  # Non-prod CIDR
  blackhole                      = true
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.production.id
}
```

### 6.3 Deploy Centralized Egress (NAT + Firewall)

```hcl
# Egress VPC in Network Hub Account
resource "aws_vpc" "egress" {
  cidr_block           = "10.33.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags                 = { Name = "egress-vpc" }
}

# NAT Gateways (one per AZ)
resource "aws_nat_gateway" "egress" {
  for_each      = toset(["us-east-1a", "us-east-1b", "us-east-1c"])
  allocation_id = aws_eip.nat[each.key].id
  subnet_id     = aws_subnet.public[each.key].id
  tags          = { Name = "nat-${each.key}" }
}

# Network Firewall (for egress inspection)
resource "aws_networkfirewall_firewall" "egress" {
  name                = "egress-firewall"
  firewall_policy_arn = aws_networkfirewall_firewall_policy.egress.arn
  vpc_id              = aws_vpc.egress.id
  
  dynamic "subnet_mapping" {
    for_each = aws_subnet.firewall
    content {
      subnet_id = subnet_mapping.value.id
    }
  }
}

# Firewall Policy (domain-based allowlist)
resource "aws_networkfirewall_firewall_policy" "egress" {
  name = "egress-policy"
  
  firewall_policy {
    stateless_default_actions          = ["aws:forward_to_sfe"]
    stateless_fragment_default_actions = ["aws:forward_to_sfe"]
    
    stateful_rule_group_reference {
      resource_arn = aws_networkfirewall_rule_group.allow_domains.arn
    }
  }
}

resource "aws_networkfirewall_rule_group" "allow_domains" {
  capacity = 100
  name     = "allow-egress-domains"
  type     = "STATEFUL"
  
  rule_group {
    rules_source {
      rules_source_list {
        generated_rules_type = "ALLOWLIST"
        target_types         = ["HTTP_HOST", "TLS_SNI"]
        targets = [
          ".amazonaws.com",
          ".docker.io",
          ".github.com",
          ".npmjs.org",
          "pypi.org",
          ".ubuntu.com",
          ".debian.org"
        ]
      }
    }
  }
}
```

### 6.4 Set Up DNS (Route 53 Resolver)

```hcl
# Centralized DNS in Network Hub Account
resource "aws_route53_resolver_endpoint" "inbound" {
  name      = "inbound-resolver"
  direction = "INBOUND"
  
  security_group_ids = [aws_security_group.dns.id]
  
  ip_address { subnet_id = aws_subnet.resolver["us-east-1a"].id }
  ip_address { subnet_id = aws_subnet.resolver["us-east-1b"].id }
}

resource "aws_route53_resolver_endpoint" "outbound" {
  name      = "outbound-resolver"
  direction = "OUTBOUND"
  
  security_group_ids = [aws_security_group.dns.id]
  
  ip_address { subnet_id = aws_subnet.resolver["us-east-1a"].id }
  ip_address { subnet_id = aws_subnet.resolver["us-east-1b"].id }
}

# Forwarding rule for on-premises domains
resource "aws_route53_resolver_rule" "on_prem" {
  domain_name          = "corp.internal"
  name                 = "forward-to-onprem"
  rule_type            = "FORWARD"
  resolver_endpoint_id = aws_route53_resolver_endpoint.outbound.id
  
  target_ip {
    ip = "10.100.0.53"  # On-prem DNS server
  }
  target_ip {
    ip = "10.100.1.53"  # On-prem DNS server (backup)
  }
}

# Share resolver rules with Organization (RAM)
resource "aws_ram_resource_share" "dns_rules" {
  name                      = "dns-resolver-rules"
  allow_external_principals = false
}

resource "aws_ram_resource_association" "dns_rule" {
  resource_arn       = aws_route53_resolver_rule.on_prem.arn
  resource_share_arn = aws_ram_resource_share.dns_rules.arn
}

resource "aws_ram_principal_association" "dns_org" {
  principal          = data.aws_organizations_organization.current.arn
  resource_share_arn = aws_ram_resource_share.dns_rules.arn
}
```

### 6.5 Deploy IPAM (IP Address Management)

```hcl
# Centralized IP address management
resource "aws_vpc_ipam" "main" {
  operating_regions { region_name = "us-east-1" }
  operating_regions { region_name = "us-west-2" }
  tags = { Name = "central-ipam" }
}

# Top-level pool (entire address space)
resource "aws_vpc_ipam_pool" "root" {
  address_family = "ipv4"
  ipam_scope_id  = aws_vpc_ipam.main.private_default_scope_id
}

resource "aws_vpc_ipam_pool_cidr" "root" {
  ipam_pool_id = aws_vpc_ipam_pool.root.id
  cidr         = "10.0.0.0/8"
}

# Production pool (allocates /22 VPCs)
resource "aws_vpc_ipam_pool" "production" {
  address_family                    = "ipv4"
  ipam_scope_id                     = aws_vpc_ipam.main.private_default_scope_id
  source_ipam_pool_id               = aws_vpc_ipam_pool.root.id
  locale                            = "us-east-1"
  allocation_default_netmask_length = 22
  allocation_min_netmask_length     = 20
  allocation_max_netmask_length     = 24
  tags                              = { Environment = "production" }
}

resource "aws_vpc_ipam_pool_cidr" "production" {
  ipam_pool_id = aws_vpc_ipam_pool.production.id
  cidr         = "10.0.0.0/12"
}

# Share IPAM pools with Organization
resource "aws_ram_resource_share" "ipam" {
  name                      = "ipam-pools"
  allow_external_principals = false
}
```

---

## Step 7: Configure Security Services

### 7.1 Enable GuardDuty (Organization-Wide)

```bash
# In Audit Account (delegated admin):
# First, delegate admin from Management Account:
aws guardduty create-detector --enable

# Delegate to Audit Account
aws organizations register-delegated-administrator \
  --account-id $AUDIT_ACCOUNT_ID \
  --service-principal guardduty.amazonaws.com

# In Audit Account — enable for all accounts:
aws guardduty create-members \
  --detector-id $DETECTOR_ID \
  --account-details '[{"AccountId":"111111111111","Email":"a@x.com"},...]'

# Enable auto-enrollment for new accounts:
aws guardduty update-organization-configuration \
  --detector-id $DETECTOR_ID \
  --auto-enable
```

```hcl
# Terraform: GuardDuty Organization Configuration
resource "aws_guardduty_organization_admin_account" "audit" {
  admin_account_id = var.audit_account_id
}

resource "aws_guardduty_organization_configuration" "main" {
  auto_enable_organization_members = "ALL"
  detector_id                      = aws_guardduty_detector.main.id
  
  datasources {
    s3_logs { auto_enable = true }
    kubernetes { audit_logs { enable = true } }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes { auto_enable = true }
      }
    }
  }
}
```

### 7.2 Enable Security Hub

```hcl
# Delegate Security Hub to Audit Account
resource "aws_securityhub_organization_admin_account" "audit" {
  admin_account_id = var.audit_account_id
}

# Enable for all accounts with auto-enrollment
resource "aws_securityhub_organization_configuration" "main" {
  auto_enable           = true
  auto_enable_standards = "DEFAULT"
  
  organization_configuration {
    configuration_type = "CENTRAL"
  }
}

# Enable standards
resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0"
}

resource "aws_securityhub_standards_subscription" "foundational" {
  standards_arn = "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
}

# Aggregate findings from all regions
resource "aws_securityhub_finding_aggregator" "all_regions" {
  linking_mode = "ALL_REGIONS"
}
```

### 7.3 Enable IAM Access Analyzer

```bash
# Organization-level Access Analyzer (from Management or Delegated Admin)
aws accessanalyzer create-analyzer \
  --analyzer-name "org-external-access" \
  --type ORGANIZATION

# This analyzer finds:
# ├── S3 buckets shared externally
# ├── IAM roles assumable from outside the org
# ├── KMS keys accessible from outside
# ├── Lambda functions invokable from outside
# └── SQS queues accessible from outside
```

### 7.4 Enable Macie (PII Detection)

```hcl
resource "aws_macie2_organization_admin_account" "audit" {
  admin_account_id = var.audit_account_id
}

# Auto-enable for all accounts
resource "aws_macie2_member" "all_accounts" {
  for_each = toset(var.all_account_ids)
  
  account_id = each.value
  invite      = true
  status      = "ENABLED"
}
```

---

## Step 8: Set Up Centralized Logging

### 8.1 Organization CloudTrail (Automatic with Control Tower)

```
Control Tower automatically creates:
├── Organization Trail → logs to Log Archive S3 bucket
├── All accounts, all regions
├── Management events + S3 data events (configurable)
├── Log file validation enabled
├── KMS encryption
└── CloudWatch Logs integration (optional — add manually)

VERIFY:
```

```bash
aws cloudtrail describe-trails --query 'trailList[?IsOrganizationTrail==`true`]'
```

### 8.2 Configure VPC Flow Logs (All Accounts via StackSets)

```yaml
# CloudFormation template deployed via StackSets
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC Flow Logs for all VPCs

Parameters:
  LogArchiveBucket:
    Type: String
    Default: "org-vpc-flow-logs-ACCOUNT_ID"

Resources:
  FlowLogRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: vpc-flow-logs.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: FlowLogPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: '*'

  # Enable for default VPC
  DefaultVPCFlowLog:
    Type: AWS::EC2::FlowLog
    Properties:
      ResourceId: !Ref DefaultVPC
      ResourceType: VPC
      TrafficType: ALL
      LogDestinationType: s3
      LogDestination: !Sub 'arn:aws:s3:::${LogArchiveBucket}'
      MaxAggregationInterval: 60
```

```bash
# Deploy via StackSets to all accounts
aws cloudformation create-stack-set \
  --stack-set-name "vpc-flow-logs" \
  --template-body file://vpc-flow-logs.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false

aws cloudformation create-stack-instances \
  --stack-set-name "vpc-flow-logs" \
  --deployment-targets OrganizationalUnitIds=$ROOT_OU_ID \
  --regions us-east-1 us-west-2
```

### 8.3 Set Up Log Archive Bucket Protection

```hcl
# In Log Archive Account
resource "aws_s3_bucket" "org_logs" {
  bucket = "org-cloudtrail-logs-${data.aws_caller_identity.current.account_id}"
}

# Enable versioning (recovery from accidental deletion)
resource "aws_s3_bucket_versioning" "org_logs" {
  bucket = aws_s3_bucket.org_logs.id
  versioning_configuration { status = "Enabled" }
}

# Object Lock (WORM — Write Once Read Many) for compliance
resource "aws_s3_bucket_object_lock_configuration" "org_logs" {
  bucket = aws_s3_bucket.org_logs.id
  
  rule {
    default_retention {
      mode = "GOVERNANCE"  # Can be overridden with special permission
      days = 365           # Retain for 1 year minimum
    }
  }
}

# Lifecycle (move to Glacier after 90 days)
resource "aws_s3_bucket_lifecycle_configuration" "org_logs" {
  bucket = aws_s3_bucket.org_logs.id
  
  rule {
    id     = "archive-old-logs"
    status = "Enabled"
    
    transition {
      days          = 90
      storage_class = "GLACIER_IR"
    }
    
    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "org_logs" {
  bucket                  = aws_s3_bucket.org_logs.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Deny delete (except break-glass role)
resource "aws_s3_bucket_policy" "org_logs" {
  bucket = aws_s3_bucket.org_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyDeletion"
        Effect    = "Deny"
        Principal = "*"
        Action    = ["s3:DeleteObject", "s3:DeleteObjectVersion"]
        Resource  = "${aws_s3_bucket.org_logs.arn}/*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" = "arn:aws:iam::*:role/BreakGlassRole"
          }
        }
      },
      {
        Sid       = "AllowOrgTrail"
        Effect    = "Allow"
        Principal = { Service = "cloudtrail.amazonaws.com" }
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.org_logs.arn}/*"
        Condition = {
          StringEquals = {
            "aws:SourceOrgID" = data.aws_organizations_organization.current.id
          }
        }
      }
    ]
  })
}
```

---

## Step 9: Account Factory & Vending

### 9.1 Create Accounts via Control Tower Account Factory

```
CONSOLE (Simple):
1. AWS Control Tower → Account Factory → "Create account"
2. Fill in:
   ├── Account email: aws-teamname-env@company.com
   ├── Display name: "TeamName-Environment"
   ├── SSO user email: team-lead@company.com
   ├── SSO user name: Team Lead
   └── OU: Workloads/Production (or Non-Production)
3. Click "Create account"
4. Wait ~30 minutes (account creation + baseline deployment)
```

### 9.2 Account Factory for Terraform (AFT)

```
AFT enables IaC-driven account provisioning with customizations:

Directory structure:
aft-account-request/
├── terraform/
│   └── main.tf           # Define accounts to create

aft-global-customizations/
├── terraform/
│   └── main.tf           # Applied to ALL accounts

aft-account-customizations/
├── terraform/
│   ├── TEAM-A-PROD/
│   │   └── main.tf      # Specific to Team A Production
│   └── DATA-PLATFORM/
│       └── main.tf      # Specific to Data Platform account
```

```hcl
# aft-account-request/terraform/main.tf
module "payments_prod" {
  source = "./modules/aft-account-request"
  
  control_tower_parameters = {
    AccountEmail              = "aws-payments-prod@company.com"
    AccountName               = "Payments-Production"
    ManagedOrganizationalUnit = "Workloads/Production"
    SSOUserEmail              = "payments-lead@company.com"
    SSOUserFirstName          = "Payments"
    SSOUserLastName           = "Lead"
  }
  
  account_tags = {
    Environment = "Production"
    Team        = "Payments"
    CostCenter  = "CC-PAYMENTS"
    Tier        = "Tier-1"
  }
  
  account_customizations_name = "TEAM-A-PROD"
}

# aft-global-customizations/terraform/main.tf
# Applied to EVERY account automatically:

# VPC with standard layout
module "vpc" {
  source = "./modules/standard-vpc"
  
  cidr_block = var.vpc_cidr  # From IPAM
  environment = var.environment
}

# Security baseline
module "security" {
  source = "./modules/security-baseline"
  
  enable_guardduty   = true
  enable_config      = true
  log_archive_bucket = var.log_archive_bucket
}

# IAM roles for CI/CD
module "cicd_roles" {
  source = "./modules/cicd-roles"
  
  shared_services_account = var.shared_services_account_id
}
```

### 9.3 Account Baseline Template

```yaml
# What every new account gets automatically (via StackSets + AFT):
account_baseline:
  networking:
    - VPC with /22 CIDR (from IPAM)
    - 3 AZs: Public, Private-App, Private-Data subnets
    - Transit Gateway attachment (connects to Network Hub)
    - Route tables: 0.0.0.0/0 → TGW (centralized egress)
    - VPC Endpoints: S3 (Gateway), ECR, Logs, SSM, KMS
    - Flow Logs enabled → Log Archive bucket
    
  security:
    - GuardDuty detector enrolled in organization
    - Security Hub enrolled with CIS + AWS Best Practices
    - Config recorder enabled → centralized bucket
    - EBS default encryption enabled (KMS)
    - S3 account-level public access block enabled
    - IAM password policy enforced
    
  iam:
    - Break-glass role (emergency access, heavily audited)
    - CI/CD deployment role (cross-account from Shared Services)
    - ReadOnly audit role (for Security team)
    - SSO permission sets assigned based on OU
    
  monitoring:
    - CloudWatch Log Groups with retention policies
    - Basic CloudWatch alarms (billing, unauthorized API calls)
    - Cross-account observability (OAM link to Monitoring account)
    
  tagging:
    - Tag policy enforced (Environment, Team, CostCenter required)
    - Default tags applied to all resources via provider
    
  cost:
    - Budget alarm at 80% and 100% of allocated budget
    - Cost anomaly detection enabled
```

---

## Step 10: Implement Guardrails & Compliance

### 10.1 Enable Control Tower Guardrails

```bash
# Enable strongly-recommended guardrails
# CONSOLE: Control Tower → Guardrails → Enable

# Key guardrails to enable:

# PREVENTIVE (SCPs):
# ├── Disallow changes to encryption configuration for S3 buckets
# ├── Disallow changes to replication configuration for S3 buckets
# ├── Disallow delete actions on S3 buckets without MFA
# ├── Disallow changes to bucket policy for S3 buckets
# ├── Disallow changes to lifecycle configuration for S3 buckets
# ├── Disallow changes to CloudWatch Log Groups
# ├── Disallow deletion of VPC Flow Logs
# └── Disallow creation of access keys for root user

# DETECTIVE (Config Rules):
# ├── Detect whether public access is granted to S3 buckets
# ├── Detect whether MFA is enabled for root user
# ├── Detect whether encryption is enabled for EBS volumes
# ├── Detect whether encryption is enabled for RDS instances
# ├── Detect whether VPC subnets are associated with NACLs
# └── Detect whether IAM policies are attached only to groups
```

### 10.2 Deploy AWS Config Rules (Organization-Wide)

```hcl
# Organization Config Rules (deployed to all accounts automatically)
resource "aws_config_organization_managed_rule" "s3_encryption" {
  name            = "s3-bucket-server-side-encryption-enabled"
  rule_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
}

resource "aws_config_organization_managed_rule" "rds_encryption" {
  name            = "rds-storage-encrypted"
  rule_identifier = "RDS_STORAGE_ENCRYPTED"
}

resource "aws_config_organization_managed_rule" "ebs_encryption" {
  name            = "encrypted-volumes"
  rule_identifier = "ENCRYPTED_VOLUMES"
}

resource "aws_config_organization_managed_rule" "mfa_root" {
  name            = "root-account-mfa-enabled"
  rule_identifier = "ROOT_ACCOUNT_MFA_ENABLED"
}

resource "aws_config_organization_managed_rule" "vpc_flow_logs" {
  name            = "vpc-flow-logs-enabled"
  rule_identifier = "VPC_FLOW_LOGS_ENABLED"
}

resource "aws_config_organization_managed_rule" "no_public_sg" {
  name            = "restricted-ssh"
  rule_identifier = "INCOMING_SSH_DISABLED"
}

resource "aws_config_organization_managed_rule" "imdsv2" {
  name            = "ec2-imdsv2-check"
  rule_identifier = "EC2_IMDSV2_CHECK"
}
```

### 10.3 Auto-Remediation for Config Rules

```hcl
# Auto-remediate: Public S3 bucket → Block public access
resource "aws_config_remediation_configuration" "s3_public" {
  config_rule_name = "s3-bucket-public-read-prohibited"
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-DisableS3BucketPublicReadWrite"
  
  parameter {
    name           = "S3BucketName"
    resource_value = "RESOURCE_ID"
  }
  
  automatic                  = true
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60
}

# Auto-remediate: Unencrypted EBS → Enable encryption (for new volumes)
resource "aws_ebs_encryption_by_default" "enabled" {
  enabled = true
}
```

---

## Post-Setup: Day 2 Operations

### Daily Operations Checklist

```yaml
daily:
  - Check Security Hub dashboard for new findings
  - Review GuardDuty alerts (should be auto-routed to Security team)
  - Verify Control Tower dashboard shows no drift
  - Monitor cost anomalies

weekly:
  - Review IAM Access Analyzer findings
  - Check for unused IAM roles/permissions
  - Review CloudTrail for unusual activity
  - Verify backup completion across accounts

monthly:
  - SCP review (any needed changes?)
  - Permission set audit (who has access to what?)
  - Cost optimization review (right-sizing, savings plans)
  - Security Hub compliance score trend

quarterly:
  - Break-glass access test (verify it works!)
  - DR drill (failover test)
  - Guardrail effectiveness review
  - Account lifecycle review (any accounts to decommission?)
```

### Monitoring Landing Zone Health

```hcl
# CloudWatch Alarms for Landing Zone Health
resource "aws_cloudwatch_metric_alarm" "org_policy_changes" {
  alarm_name          = "organization-policy-changes"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "OrganizationPolicyChanges"
  namespace           = "CloudTrailMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "SCP or Tag Policy was modified!"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "root_login" {
  alarm_name          = "root-account-login"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "RootAccountUsage"
  namespace           = "CloudTrailMetrics"
  period              = 60
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Root user login detected!"
  alarm_actions       = [aws_sns_topic.security_critical.arn]
}

resource "aws_cloudwatch_metric_alarm" "unauthorized_api" {
  alarm_name          = "unauthorized-api-calls"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "UnauthorizedAPICalls"
  namespace           = "CloudTrailMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Multiple unauthorized API calls detected"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

### Break-Glass Procedure Documentation

```yaml
break_glass_procedure:
  purpose: "Emergency access when normal access methods are unavailable"
  
  when_to_use:
    - SSO/Identity Center is down
    - Need to modify SCPs during incident
    - Management Account access required
    - Root user action needed (rare!)
  
  procedure:
    step_1: "Notify security team and incident commander"
    step_2: "Retrieve break-glass credentials"
    location: "Hardware MFA in fire-safe (Building A, Floor 3, Safe #2)"
    backup: "Secondary MFA in sealed envelope (CFO's office safe)"
    step_3: "Login to Management Account as root user"
    step_4: "Perform required action (document everything!)"
    step_5: "Logout immediately when done"
    step_6: "Rotate any credentials that were accessed"
    step_7: "File incident report documenting: who, what, when, why"
  
  testing:
    frequency: "Quarterly"
    last_tested: "2024-06-01"
    test_procedure: "Login as root → verify MFA → take screenshot → logout"
    test_participants: ["CISO", "Platform Lead"]
    next_test: "2024-09-01"
```

---

## Summary: What You've Built

```
┌─────────────────────────────────────────────────────────────────────┐
│  YOUR AWS LANDING ZONE                                               │
│                                                                       │
│  ✅ AWS Organizations (multi-account structure)                      │
│  ✅ Control Tower (managed landing zone + guardrails)                │
│  ✅ 5+ OUs with proper SCP inheritance                              │
│  ✅ IAM Identity Center (federated SSO, no IAM users)               │
│  ✅ 4+ SCPs (regions, encryption, security baseline, sandbox)       │
│  ✅ Transit Gateway (centralized networking)                        │
│  ✅ Network Firewall (egress inspection)                            │
│  ✅ Route 53 Resolver (centralized DNS)                             │
│  ✅ IPAM (IP address management)                                    │
│  ✅ GuardDuty (threat detection, all accounts)                      │
│  ✅ Security Hub (compliance, all accounts)                         │
│  ✅ IAM Access Analyzer (external access detection)                 │
│  ✅ Organization CloudTrail (audit logging)                         │
│  ✅ VPC Flow Logs (network logging)                                 │
│  ✅ Log Archive (immutable, encrypted, lifecycle)                   │
│  ✅ Account Factory (automated account provisioning)                │
│  ✅ Config Rules (compliance detection)                             │
│  ✅ Auto-remediation (fix violations automatically)                 │
│  ✅ Break-glass procedures (documented + tested)                    │
│  ✅ Monitoring & alerting (unauthorized access, policy changes)     │
└─────────────────────────────────────────────────────────────────────┘

Total setup time: 2-4 weeks (with team of 2-3 platform engineers)
Ongoing maintenance: 2-4 hours/week (mostly automated)
```
