# Networking & Security Framework - Enterprise Architecture

## Network Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        INTERNET                                   │
└─────────────────┬──────────────────────┬────────────────────────┘
                  │                      │
         ┌────────▼────────┐    ┌────────▼────────┐
         │  CloudFront     │    │  Global          │
         │  Distributions  │    │  Accelerator     │
         └────────┬────────┘    └────────┬────────┘
                  │                      │
         ┌────────▼──────────────────────▼────────┐
         │          WAF + Shield Advanced          │
         └────────────────────┬───────────────────┘
                              │
         ┌────────────────────▼───────────────────┐
         │        Network Hub Account              │
         │  ┌──────────────────────────────────┐  │
         │  │     Ingress VPC                   │  │
         │  │  ┌──────┐  ┌──────┐  ┌────────┐ │  │
         │  │  │ ALB  │  │ NLB  │  │ NW FW  │ │  │
         │  │  └──────┘  └──────┘  └────────┘ │  │
         │  └──────────────────────────────────┘  │
         │  ┌──────────────────────────────────┐  │
         │  │     Inspection VPC                │  │
         │  │  ┌─────────────────────────────┐ │  │
         │  │  │   Network Firewall          │ │  │
         │  │  │   (Stateful + Stateless)    │ │  │
         │  │  └─────────────────────────────┘ │  │
         │  └──────────────────────────────────┘  │
         │  ┌──────────────────────────────────┐  │
         │  │     Egress VPC                    │  │
         │  │  ┌──────────┐  ┌──────────────┐ │  │
         │  │  │ NAT GW   │  │ Proxy (Squid)│ │  │
         │  │  └──────────┘  └──────────────┘ │  │
         │  └──────────────────────────────────┘  │
         │  ┌──────────────────────────────────┐  │
         │  │     Transit Gateway               │  │
         │  │  ┌────────────────────────────┐  │  │
         │  │  │ Route Tables:              │  │  │
         │  │  │ - Production RT            │  │  │
         │  │  │ - Non-Production RT        │  │  │
         │  │  │ - Shared Services RT       │  │  │
         │  │  │ - On-Premises RT           │  │  │
         │  │  └────────────────────────────┘  │  │
         │  └──────────────────────────────────┘  │
         └─────────────────────┬──────────────────┘
                               │
         ┌─────────────────────┼──────────────────────────┐
         │                     │                          │
┌────────▼────────┐  ┌────────▼────────┐  ┌─────────────▼──────┐
│ Production VPCs │  │ Non-Prod VPCs   │  │ On-Premises        │
│ (Workload Accts)│  │ (Workload Accts)│  │ (Direct Connect)   │
└─────────────────┘  └─────────────────┘  └────────────────────┘
```

## VPC Design Patterns

### Standard Workload VPC Architecture:

```hcl
# Terraform module for standardized VPC
module "workload_vpc" {
  source = "./modules/standard-vpc"
  
  vpc_cidr = "10.1.0.0/22"  # /22 = 1024 IPs
  
  # 3 AZs, 4 subnet tiers
  subnet_config = {
    public = {
      # ALB/NLB only (no EC2 with public IPs)
      cidrs = ["10.1.0.0/26", "10.1.0.64/26", "10.1.0.128/26"]
      # 62 usable IPs per subnet
    }
    private_app = {
      # Application tier (ECS, EKS, EC2)
      cidrs = ["10.1.1.0/24", "10.1.2.0/25", "10.1.2.128/25"]
      # Largest allocation for compute
    }
    private_data = {
      # Database tier (RDS, ElastiCache, OpenSearch)
      cidrs = ["10.1.3.0/26", "10.1.3.64/26", "10.1.3.128/26"]
    }
    private_mgmt = {
      # Management (endpoints, bastion if needed)
      cidrs = ["10.1.3.192/27", "10.1.3.224/27", "10.1.0.192/27"]
    }
  }
  
  # VPC Endpoints for private access
  gateway_endpoints = ["s3", "dynamodb"]
  interface_endpoints = [
    "ec2", "ecr.api", "ecr.dkr", "ecs", 
    "logs", "monitoring", "ssm", "ssmmessages",
    "kms", "secretsmanager", "sts"
  ]
  
  # Transit Gateway attachment
  tgw_id          = var.transit_gateway_id
  tgw_route_table = var.tgw_route_table_id
  
  # No IGW, No NAT GW (centralized in Network Hub)
  enable_igw = false
  enable_nat = false
}
```

### Network ACLs (Defense in Depth):

```hcl
# Data tier NACL - extremely restrictive
resource "aws_network_acl" "data_tier" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = var.data_subnet_ids

  # Allow inbound from App tier only on specific ports
  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = var.app_tier_cidr
    from_port  = 5432  # PostgreSQL
    to_port    = 5432
  }
  
  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = var.app_tier_cidr
    from_port  = 6379  # Redis
    to_port    = 6379
  }

  # Allow ephemeral return traffic
  ingress {
    protocol   = "tcp"
    rule_no    = 900
    action     = "allow"
    cidr_block = var.vpc_cidr
    from_port  = 1024
    to_port    = 65535
  }

  # Deny all other inbound
  ingress {
    protocol   = -1
    rule_no    = 999
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  # Allow outbound to App tier (responses)
  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = var.app_tier_cidr
    from_port  = 1024
    to_port    = 65535
  }
  
  # Deny all other outbound (data tier should never initiate)
  egress {
    protocol   = -1
    rule_no    = 999
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
}
```

## AWS Network Firewall

### Stateful Rule Groups:

```hcl
resource "aws_networkfirewall_rule_group" "block_domains" {
  capacity = 100
  name     = "block-malicious-domains"
  type     = "STATEFUL"
  
  rule_group {
    rule_variables {
      ip_sets {
        key = "HOME_NET"
        ip_set { definition = ["10.0.0.0/8"] }
      }
    }
    
    rules_source {
      rules_source_list {
        generated_rules_type = "DENYLIST"
        target_types         = ["HTTP_HOST", "TLS_SNI"]
        targets = [
          ".malware-domain.com",
          ".crypto-mining.net",
          ".bad-actor.org"
        ]
      }
    }
    
    stateful_rule_options {
      rule_order = "STRICT_ORDER"
    }
  }
}

resource "aws_networkfirewall_rule_group" "allow_egress" {
  capacity = 200
  name     = "allowed-egress-destinations"
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
          ".ubuntu.com"
        ]
      }
    }
  }
}
```

### Suricata Rules for Advanced Inspection:

```
# Drop crypto mining traffic
drop tls any any -> any any (tls.sni; content:"mining"; nocase; msg:"Crypto mining detected"; sid:1000001; rev:1;)

# Alert on data exfiltration patterns (large outbound)
alert tcp $HOME_NET any -> $EXTERNAL_NET any (flow:established,to_server; dsize:>10000; threshold:type threshold, track by_src, count 100, seconds 60; msg:"Potential data exfiltration"; sid:1000002; rev:1;)

# Block SSH to external
drop tcp $HOME_NET any -> !$HOME_NET 22 (msg:"External SSH blocked"; sid:1000003; rev:1;)
```

## Security Architecture

### Defense in Depth Layers:

```
Layer 1: Perimeter (CloudFront + WAF + Shield)
Layer 2: Network (Security Groups + NACLs + Network Firewall)
Layer 3: Identity (IAM + SCP + Permission Boundaries)
Layer 4: Application (WAF rules + Input validation)
Layer 5: Data (Encryption at rest + in transit + tokenization)
Layer 6: Detection (GuardDuty + Security Hub + Detective)
Layer 7: Response (Automated remediation + Forensics)
```

### GuardDuty Organization Configuration:

```hcl
resource "aws_guardduty_organization_admin_account" "main" {
  admin_account_id = var.security_account_id
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

### Automated Remediation (Security Hub + EventBridge + Lambda):

```python
import boto3

def remediate_public_s3(event):
    """Auto-remediate public S3 buckets detected by Security Hub."""
    s3 = boto3.client('s3')
    
    finding = event['detail']['findings'][0]
    bucket_name = finding['Resources'][0]['Id'].split(':')[-1]
    account_id = finding['AwsAccountId']
    
    # Assume role in target account
    sts = boto3.client('sts')
    credentials = sts.assume_role(
        RoleArn=f'arn:aws:iam::{account_id}:role/SecurityRemediationRole',
        RoleSessionName='auto-remediation'
    )['Credentials']
    
    s3_target = boto3.client('s3',
        aws_access_key_id=credentials['AccessKeyId'],
        aws_secret_access_key=credentials['SecretAccessKey'],
        aws_session_token=credentials['SessionToken']
    )
    
    # Block public access
    s3_target.put_public_access_block(
        Bucket=bucket_name,
        PublicAccessBlockConfiguration={
            'BlockPublicAcls': True,
            'IgnorePublicAcls': True,
            'BlockPublicPolicy': True,
            'RestrictPublicBuckets': True
        }
    )
    
    # Notify via SNS
    sns = boto3.client('sns')
    sns.publish(
        TopicArn=os.environ['ALERT_TOPIC'],
        Subject=f'Auto-Remediation: Public S3 bucket blocked - {bucket_name}',
        Message=f'Account: {account_id}\nBucket: {bucket_name}\nAction: Public access blocked'
    )
```

## Transit Gateway Deep Dive

### Multi-Region TGW Architecture:

```hcl
# Region 1 - Primary
resource "aws_ec2_transit_gateway" "primary" {
  provider = aws.us_east_1
  
  description                     = "Primary TGW - us-east-1"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  auto_accept_shared_attachments  = "enable"
  
  tags = { Name = "tgw-primary-use1" }
}

# Region 2 - Secondary
resource "aws_ec2_transit_gateway" "secondary" {
  provider = aws.us_west_2
  
  description                     = "Secondary TGW - us-west-2"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  auto_accept_shared_attachments  = "enable"
  
  tags = { Name = "tgw-secondary-usw2" }
}

# Inter-Region Peering
resource "aws_ec2_transit_gateway_peering_attachment" "cross_region" {
  provider = aws.us_east_1
  
  peer_account_id         = data.aws_caller_identity.current.account_id
  peer_region             = "us-west-2"
  peer_transit_gateway_id = aws_ec2_transit_gateway.secondary.id
  transit_gateway_id      = aws_ec2_transit_gateway.primary.id
}
```

### Route Table Segmentation:

```hcl
# Production Route Table - isolated from Dev
resource "aws_ec2_transit_gateway_route_table" "production" {
  transit_gateway_id = aws_ec2_transit_gateway.primary.id
  tags               = { Name = "tgw-rt-production" }
}

# Non-Production Route Table
resource "aws_ec2_transit_gateway_route_table" "non_production" {
  transit_gateway_id = aws_ec2_transit_gateway.primary.id
  tags               = { Name = "tgw-rt-non-production" }
}

# Shared Services Route Table (accessible from all)
resource "aws_ec2_transit_gateway_route_table" "shared" {
  transit_gateway_id = aws_ec2_transit_gateway.primary.id
  tags               = { Name = "tgw-rt-shared-services" }
}

# Production can reach Shared Services but NOT Non-Production
resource "aws_ec2_transit_gateway_route" "prod_to_shared" {
  destination_cidr_block         = var.shared_services_cidr
  transit_gateway_attachment_id  = var.shared_services_attachment_id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.production.id
}

# Blackhole route - Production cannot reach Non-Production
resource "aws_ec2_transit_gateway_route" "prod_blackhole_dev" {
  destination_cidr_block         = var.non_production_cidr
  blackhole                      = true
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.production.id
}
```

## Direct Connect Architecture

```
┌─────────────────┐         ┌──────────────────────┐
│  On-Premises    │         │  AWS                  │
│                 │         │                       │
│  ┌───────────┐  │   DX    │  ┌────────────────┐  │
│  │ Core      │──┼─────────┼──│ DX Gateway     │  │
│  │ Router    │  │  10Gbps │  │                │  │
│  └───────────┘  │  (LAG)  │  │ ┌────────────┐ │  │
│                 │         │  │ │ Private VIF │─┼──┤
│  ┌───────────┐  │   DX    │  │ │ (VPC/TGW)  │ │  │ Transit VIF
│  │ Backup    │──┼─────────┼──│ └────────────┘ │  │ connects to TGW
│  │ Router    │  │  1Gbps  │  │ ┌────────────┐ │  │
│  └───────────┘  │ (backup)│  │ │ Transit VIF│─┼──┤
│                 │         │  │ │ (TGW)      │ │  │
│                 │         │  │ └────────────┘ │  │
│                 │         │  └────────────────┘  │
└─────────────────┘         └──────────────────────┘

Redundancy: 2 connections at different DX locations
Failover: BGP with AS-PATH prepending for active/passive
Backup: Site-to-Site VPN over internet (automatic failover)
```

### Direct Connect Resilience:

```hcl
# Primary DX Connection (Location 1)
resource "aws_dx_connection" "primary" {
  name      = "dx-primary-equinix-dc"
  bandwidth = "10Gbps"
  location  = "EqDC2"
}

# Secondary DX Connection (Location 2 - different facility)
resource "aws_dx_connection" "secondary" {
  name      = "dx-secondary-coresite-va"
  bandwidth = "10Gbps"
  location  = "CoreSite-VA"
}

# DX Gateway (global resource)
resource "aws_dx_gateway" "main" {
  name            = "dx-gateway-main"
  amazon_side_asn = 64512
}

# Transit VIF for multi-VPC access
resource "aws_dx_transit_virtual_interface" "primary" {
  connection_id  = aws_dx_connection.primary.id
  dx_gateway_id  = aws_dx_gateway.main.id
  name           = "transit-vif-primary"
  vlan           = 100
  address_family = "ipv4"
  bgp_asn        = 65000  # Customer ASN
  mtu            = 8500   # Jumbo frames
}

# Associate DX Gateway with Transit Gateway
resource "aws_dx_gateway_association" "main" {
  dx_gateway_id         = aws_dx_gateway.main.id
  associated_gateway_id = aws_ec2_transit_gateway.primary.id
  
  allowed_prefixes = [
    "10.0.0.0/8",    # All AWS VPCs
    "172.16.0.0/12"  # Additional ranges
  ]
}
```

## VPC Endpoints Strategy

### Cost-Optimized Endpoint Deployment:

```hcl
# Centralized VPC Endpoints in Shared Services VPC
# Other VPCs access via Route 53 Resolver + PrivateLink

locals {
  # High-traffic endpoints (deploy per-VPC to avoid cross-AZ charges)
  per_vpc_endpoints = ["s3", "dynamodb"]  # Gateway endpoints (free)
  
  # Low-traffic endpoints (centralize in shared VPC)
  centralized_endpoints = [
    "ec2", "ecr.api", "ecr.dkr", "ecs",
    "logs", "monitoring", "kms", "secretsmanager",
    "ssm", "ssmmessages", "ec2messages",
    "sqs", "sns", "events", "sts"
  ]
}

# Interface endpoints in Shared Services VPC
resource "aws_vpc_endpoint" "centralized" {
  for_each = toset(local.centralized_endpoints)
  
  vpc_id              = var.shared_services_vpc_id
  service_name        = "com.amazonaws.${var.region}.${each.value}"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.endpoint_subnet_ids
  security_group_ids  = [aws_security_group.endpoints.id]
  private_dns_enabled = true
}

# Route 53 PHZ for cross-VPC endpoint resolution
resource "aws_route53_zone" "endpoint_phz" {
  for_each = toset(local.centralized_endpoints)
  
  name = "${each.value}.${var.region}.amazonaws.com"
  
  vpc {
    vpc_id = var.shared_services_vpc_id
  }
}

# Associate workload VPCs with endpoint PHZs
resource "aws_route53_zone_association" "workload" {
  for_each = {
    for pair in setproduct(keys(var.workload_vpcs), local.centralized_endpoints) :
    "${pair[0]}-${pair[1]}" => {
      vpc_id  = var.workload_vpcs[pair[0]]
      zone_id = aws_route53_zone.endpoint_phz[pair[1]].zone_id
    }
  }
  
  zone_id = each.value.zone_id
  vpc_id  = each.value.vpc_id
}
```

## Zero Trust Network Architecture

### Principles:
1. **Never trust, always verify** - No implicit trust based on network location
2. **Least privilege access** - Minimum necessary permissions
3. **Assume breach** - Design for compromise containment
4. **Verify explicitly** - Authenticate and authorize every request

### Implementation:
```
┌─────────────────────────────────────────────────┐
│  Zero Trust Components                           │
│                                                  │
│  Identity: IAM Identity Center + MFA             │
│  Device:   AWS Verified Access (device posture)  │
│  Network:  VPC Lattice (service-to-service)      │
│  Data:     KMS + Macie + DLP                     │
│  App:      WAF + Cognito + API Gateway           │
│  Monitoring: CloudTrail + GuardDuty + Detective  │
└─────────────────────────────────────────────────┘
```

### VPC Lattice (Service Mesh):
```hcl
resource "aws_vpclattice_service_network" "main" {
  name      = "platform-service-network"
  auth_type = "AWS_IAM"
}

resource "aws_vpclattice_service" "api" {
  name      = "payment-api"
  auth_type = "AWS_IAM"
}

resource "aws_vpclattice_auth_policy" "api" {
  resource_identifier = aws_vpclattice_service.api.arn
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.consumer_account}:role/OrderService" }
      Action    = "vpc-lattice-svcs:Invoke"
      Resource  = "*"
      Condition = {
        StringEquals = {
          "vpc-lattice-svcs:RequestMethod" = ["GET", "POST"]
          "vpc-lattice-svcs:RequestHeader/x-api-version" = "v2"
        }
      }
    }]
  })
}
```
