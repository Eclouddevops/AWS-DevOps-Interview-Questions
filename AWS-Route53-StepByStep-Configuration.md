# AWS Route 53 - Step-by-Step Configuration Guide

## Table of Contents
1. [Hosted Zone Setup](#1-hosted-zone-setup)
2. [DNS Record Types & Creation](#2-dns-record-types--creation)
3. [Routing Policies Configuration](#3-routing-policies-configuration)
4. [Health Checks Setup](#4-health-checks-setup)
5. [Failover Configuration](#5-failover-configuration)
6. [Domain Registration & Transfer](#6-domain-registration--transfer)
7. [Private Hosted Zones (VPC)](#7-private-hosted-zones-vpc)
8. [DNSSEC Configuration](#8-dnssec-configuration)
9. [Route 53 Resolver](#9-route-53-resolver)
10. [Production Best Practices](#10-production-best-practices)

---

## 1. Hosted Zone Setup

### Step 1.1: Create Public Hosted Zone


**AWS Console Path:**
```
Route 53 → Hosted Zones → Create Hosted Zone
├── Domain Name: myapp.com
├── Type: Public Hosted Zone
└── Click "Create Hosted Zone"
```

**AWS CLI:**
```bash
# Create public hosted zone
aws route53 create-hosted-zone \
  --name myapp.com \
  --caller-reference "$(date +%s)-myapp" \
  --hosted-zone-config Comment="Production hosted zone for myapp.com"

# Output includes:
# - HostedZone.Id: /hostedzone/Z1234567890ABC
# - DelegationSet.NameServers: ns-xxx.awsdns-xx.com (4 nameservers)
```

### Step 1.2: Get Nameservers

```bash
# Get nameservers for your hosted zone
aws route53 get-hosted-zone --id Z1234567890ABC \
  --query 'DelegationSet.NameServers'

# Output:
# [
#   "ns-123.awsdns-15.com",
#   "ns-456.awsdns-57.net",
#   "ns-789.awsdns-34.org",
#   "ns-1011.awsdns-62.co.uk"
# ]

# UPDATE THESE AT YOUR DOMAIN REGISTRAR
```

### Step 1.3: Verify DNS Propagation

```bash
# Check if nameservers are propagated
dig NS myapp.com +short

# Check from specific nameserver
dig @ns-123.awsdns-15.com myapp.com ANY

# Use multiple global DNS checkers
# - whatsmydns.net
# - dnschecker.org
```

---

## 2. DNS Record Types & Creation

### Step 2.1: A Record (ALB Alias)

```bash
# Create A record pointing to ALB (Alias - no charge for queries)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Comment": "Point api.myapp.com to production ALB",
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "production-api-alb-123456.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

**Note:** ALB Hosted Zone IDs by Region:
| Region | ALB Hosted Zone ID |
|---|---|
| us-east-1 | Z35SXDOTRQ7X7K |
| us-west-2 | Z1H1FL5HABSF5 |
| eu-west-1 | Z32O12XQLNTSW2 |
| ap-south-1 | ZP97RAFLXTNZK |
| ap-southeast-1 | Z1LMS91P8CMLE5 |

### Step 2.2: CNAME Record

```bash
# CNAME for subdomain
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "www.myapp.com",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{"Value": "myapp.com"}]
      }
    }]
  }'
```

### Step 2.3: MX Records (Email)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "myapp.com",
        "Type": "MX",
        "TTL": 3600,
        "ResourceRecords": [
          {"Value": "1 aspmx.l.google.com"},
          {"Value": "5 alt1.aspmx.l.google.com"},
          {"Value": "5 alt2.aspmx.l.google.com"},
          {"Value": "10 alt3.aspmx.l.google.com"},
          {"Value": "10 alt4.aspmx.l.google.com"}
        ]
      }
    }]
  }'
```

### Step 2.4: TXT Records (SPF, DKIM, Verification)

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "myapp.com",
          "Type": "TXT",
          "TTL": 300,
          "ResourceRecords": [
            {"Value": "\"v=spf1 include:_spf.google.com include:amazonses.com ~all\""},
            {"Value": "\"google-site-verification=xxxxxxxxxxxxxxxxxxxx\""}
          ]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "google._domainkey.myapp.com",
          "Type": "TXT",
          "TTL": 300,
          "ResourceRecords": [
            {"Value": "\"v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w...\""}
          ]
        }
      }
    ]
  }'
```

### Step 2.5: Multiple Records in One Batch

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Comment": "Initial DNS setup for myapp.com",
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "myapp.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z2FDTNDATAQYW2",
            "DNSName": "d123456.cloudfront.net",
            "EvaluateTargetHealth": false
          }
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "api.myapp.com",
          "Type": "A",
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "production-api-alb-123456.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "staging.myapp.com",
          "Type": "CNAME",
          "TTL": 300,
          "ResourceRecords": [{"Value": "staging-alb-789.us-east-1.elb.amazonaws.com"}]
        }
      }
    ]
  }'
```


---

## 3. Routing Policies Configuration

### Step 3.1: Simple Routing

```bash
# Single resource
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "tools.myapp.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "54.23.45.67"}]
      }
    }]
  }'
```

### Step 3.2: Weighted Routing (Canary/Traffic Splitting)

```bash
# 90% traffic to stable version
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "production-stable",
        "Weight": 90,
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "stable-alb-123.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# 10% traffic to canary version
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "production-canary",
        "Weight": 10,
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "canary-alb-456.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Shift all traffic to canary (after validation)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.myapp.com",
          "Type": "A",
          "SetIdentifier": "production-stable",
          "Weight": 0,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "stable-alb-123.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "api.myapp.com",
          "Type": "A",
          "SetIdentifier": "production-canary",
          "Weight": 100,
          "AliasTarget": {
            "HostedZoneId": "Z35SXDOTRQ7X7K",
            "DNSName": "canary-alb-456.us-east-1.elb.amazonaws.com",
            "EvaluateTargetHealth": true
          }
        }
      }
    ]
  }'
```

### Step 3.3: Latency-Based Routing (Multi-Region)

```bash
# US East region
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "us-east-1",
        "Region": "us-east-1",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "us-east-alb.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# EU West region
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "eu-west-1",
        "Region": "eu-west-1",
        "AliasTarget": {
          "HostedZoneId": "Z32O12XQLNTSW2",
          "DNSName": "eu-west-alb.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Asia Pacific region
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "ap-southeast-1",
        "Region": "ap-southeast-1",
        "AliasTarget": {
          "HostedZoneId": "Z1LMS91P8CMLE5",
          "DNSName": "ap-southeast-alb.ap-southeast-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Step 3.4: Geolocation Routing (Compliance/Localization)

```bash
# Indian users → India servers
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "india",
        "GeoLocation": {"CountryCode": "IN"},
        "AliasTarget": {
          "HostedZoneId": "ZP97RAFLXTNZK",
          "DNSName": "india-alb.ap-south-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# European users → EU servers (GDPR compliance)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "europe",
        "GeoLocation": {"ContinentCode": "EU"},
        "AliasTarget": {
          "HostedZoneId": "Z32O12XQLNTSW2",
          "DNSName": "eu-alb.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Default (all other users) → US servers
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "default",
        "GeoLocation": {"CountryCode": "*"},
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "us-alb.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Step 3.5: Geoproximity Routing (Traffic Flow)

```bash
# Create traffic policy with bias
aws route53 create-traffic-policy \
  --name "geo-proximity-policy" \
  --document '{
    "AWSPolicyFormatVersion": "2015-10-01",
    "RecordType": "A",
    "Endpoints": {
      "us-east-endpoint": {
        "Type": "elastic-load-balancer",
        "Value": "production-api-alb-123456.us-east-1.elb.amazonaws.com"
      },
      "eu-west-endpoint": {
        "Type": "elastic-load-balancer",
        "Value": "production-api-alb-789012.eu-west-1.elb.amazonaws.com"
      }
    },
    "Rules": {
      "geo-proximity-rule": {
        "RuleType": "geoproximity",
        "GeoproximityLocations": [
          {
            "EndpointReference": "us-east-endpoint",
            "Region": "aws:route53:us-east-1",
            "Bias": 10
          },
          {
            "EndpointReference": "eu-west-endpoint",
            "Region": "aws:route53:eu-west-1",
            "Bias": -10
          }
        ]
      }
    },
    "StartRule": "geo-proximity-rule"
  }'
```


---

## 4. Health Checks Setup

### Step 4.1: Create HTTP/HTTPS Health Check

```bash
# HTTPS health check for API endpoint
aws route53 create-health-check \
  --caller-reference "api-health-$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "api.myapp.com",
    "Port": 443,
    "ResourcePath": "/health",
    "RequestInterval": 10,
    "FailureThreshold": 2,
    "MeasureLatency": true,
    "EnableSNI": true,
    "Regions": ["us-east-1", "eu-west-1", "ap-southeast-1"]
  }'

# Get the health check ID from output
# HealthCheck.Id: "hc-12345678-abcd-efgh-ijkl"
```

### Step 4.2: Create Health Check with String Matching

```bash
# Health check that validates response body contains specific text
aws route53 create-health-check \
  --caller-reference "api-health-string-$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS_STR_MATCH",
    "FullyQualifiedDomainName": "api.myapp.com",
    "Port": 443,
    "ResourcePath": "/health",
    "SearchString": "\"status\":\"healthy\"",
    "RequestInterval": 10,
    "FailureThreshold": 3,
    "MeasureLatency": true
  }'
```

### Step 4.3: Create Calculated Health Check

```bash
# Create individual health checks first
# Then create a calculated health check (2 out of 3 must be healthy)
aws route53 create-health-check \
  --caller-reference "calculated-health-$(date +%s)" \
  --health-check-config '{
    "Type": "CALCULATED",
    "ChildHealthChecks": [
      "hc-11111111-aaaa-bbbb-cccc",
      "hc-22222222-dddd-eeee-ffff",
      "hc-33333333-gggg-hhhh-iiii"
    ],
    "HealthThreshold": 2
  }'
```

### Step 4.4: Health Check with CloudWatch Alarm

```bash
# Health check based on CloudWatch alarm state
aws route53 create-health-check \
  --caller-reference "cw-alarm-health-$(date +%s)" \
  --health-check-config '{
    "Type": "CLOUDWATCH_METRIC",
    "AlarmIdentifier": {
      "Region": "us-east-1",
      "Name": "Production-ALB-5xxErrors"
    },
    "InsufficientDataHealthStatus": "LastKnownStatus"
  }'
```

### Step 4.5: Add Health Check Notifications

```bash
# Tag health check for identification
aws route53 change-tags-for-resource \
  --resource-type healthcheck \
  --resource-id hc-12345678-abcd-efgh-ijkl \
  --add-tags Key=Name,Value="API Production Health Check" \
             Key=Environment,Value=production

# Create CloudWatch alarm for health check status
aws cloudwatch put-metric-alarm \
  --alarm-name "Route53-API-HealthCheck-Failed" \
  --metric-name HealthCheckStatus \
  --namespace AWS/Route53 \
  --statistic Minimum \
  --period 60 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=HealthCheckId,Value=hc-12345678-abcd-efgh-ijkl \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:production-critical \
  --alarm-description "Route 53 health check failed for api.myapp.com"
```

---

## 5. Failover Configuration

### Step 5.1: Active-Passive Failover

```bash
# Primary record (active)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "HealthCheckId": "hc-primary-12345678",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "primary-alb.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Secondary record (passive - DR)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "HealthCheckId": "hc-secondary-87654321",
        "AliasTarget": {
          "HostedZoneId": "Z1H1FL5HABSF5",
          "DNSName": "dr-alb.us-west-2.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Step 5.2: Active-Active Multi-Region with Failover

```bash
# Combine latency routing with health checks for active-active
# If one region fails health check, traffic auto-routes to healthy region

# Region 1: us-east-1 (with health check)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "us-east-1-active",
        "Region": "us-east-1",
        "HealthCheckId": "hc-us-east-12345",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "us-east-alb.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Region 2: eu-west-1 (with health check)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "eu-west-1-active",
        "Region": "eu-west-1",
        "HealthCheckId": "hc-eu-west-67890",
        "AliasTarget": {
          "HostedZoneId": "Z32O12XQLNTSW2",
          "DNSName": "eu-west-alb.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Step 5.3: Failover to Static S3 Website (Maintenance Page)

```bash
# Create S3 bucket for maintenance page
aws s3 mb s3://myapp-maintenance-page
aws s3 website s3://myapp-maintenance-page \
  --index-document index.html \
  --error-document error.html

# Upload maintenance page
cat > /tmp/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Maintenance</title></head>
<body>
<h1>We'll be back shortly</h1>
<p>Our service is under maintenance. Please check back in a few minutes.</p>
</body>
</html>
EOF
aws s3 cp /tmp/index.html s3://myapp-maintenance-page/

# Create failover record to S3 maintenance page
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "myapp.com",
        "Type": "A",
        "SetIdentifier": "maintenance-page",
        "Failover": "SECONDARY",
        "AliasTarget": {
          "HostedZoneId": "Z3AQBSTGFYJSTF",
          "DNSName": "s3-website-us-east-1.amazonaws.com",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```


---

## 6. Domain Registration & Transfer

### Step 6.1: Register a Domain

```bash
# Check domain availability
aws route53domains check-domain-availability \
  --domain-name myapp.com \
  --region us-east-1

# Register domain
aws route53domains register-domain \
  --domain-name myapp.com \
  --duration-in-years 1 \
  --auto-renew \
  --admin-contact '{
    "FirstName": "John",
    "LastName": "Doe",
    "ContactType": "COMPANY",
    "OrganizationName": "MyCompany Inc",
    "Email": "admin@mycompany.com",
    "PhoneNumber": "+1.2125551234",
    "AddressLine1": "123 Main St",
    "City": "New York",
    "State": "NY",
    "CountryCode": "US",
    "ZipCode": "10001"
  }' \
  --registrant-contact '{ ... same as above ... }' \
  --tech-contact '{ ... same as above ... }' \
  --privacy-protect-admin-contact \
  --privacy-protect-registrant-contact \
  --privacy-protect-tech-contact \
  --region us-east-1
```

### Step 6.2: Transfer Domain to Route 53

```bash
# Step 1: At current registrar - Unlock domain & get auth code

# Step 2: Initiate transfer
aws route53domains transfer-domain \
  --domain-name myapp.com \
  --duration-in-years 1 \
  --auth-code "EPP_AUTH_CODE_FROM_REGISTRAR" \
  --auto-renew \
  --admin-contact '{ ... }' \
  --registrant-contact '{ ... }' \
  --tech-contact '{ ... }' \
  --region us-east-1

# Step 3: Check transfer status
aws route53domains get-domain-detail \
  --domain-name myapp.com \
  --region us-east-1

# Step 4: List operations
aws route53domains list-operations \
  --region us-east-1 \
  --query 'Operations[?DomainName==`myapp.com`]'
```

### Step 6.3: Domain Lock & Protection

```bash
# Enable transfer lock
aws route53domains enable-domain-transfer-lock \
  --domain-name myapp.com \
  --region us-east-1

# Enable auto-renew
aws route53domains enable-domain-auto-renew \
  --domain-name myapp.com \
  --region us-east-1

# Check domain status
aws route53domains get-domain-detail \
  --domain-name myapp.com \
  --region us-east-1 \
  --query '{Status:StatusList,AutoRenew:AutoRenew,Expiry:ExpirationDate}'
```

---

## 7. Private Hosted Zones (VPC)

### Step 7.1: Create Private Hosted Zone

```bash
# Create private hosted zone for internal services
aws route53 create-hosted-zone \
  --name internal.myapp.com \
  --caller-reference "internal-$(date +%s)" \
  --vpc VPCRegion=us-east-1,VPCId=vpc-0123456789abcdef0 \
  --hosted-zone-config Comment="Internal DNS for production VPC",PrivateZone=true
```

### Step 7.2: Associate Additional VPCs

```bash
# Associate another VPC (e.g., staging VPC needs to resolve internal names)
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0987654321XYZ \
  --vpc VPCRegion=us-east-1,VPCId=vpc-staging-0123456789

# Cross-account VPC association (requires authorization)
# Account A (zone owner): Create authorization
aws route53 create-vpc-association-authorization \
  --hosted-zone-id Z0987654321XYZ \
  --vpc VPCRegion=us-east-1,VPCId=vpc-accountB-0123456789

# Account B: Associate VPC with the zone
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z0987654321XYZ \
  --vpc VPCRegion=us-east-1,VPCId=vpc-accountB-0123456789
```

### Step 7.3: Internal Service Discovery Records

```bash
# Database endpoint (internal)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z0987654321XYZ \
  --change-batch '{
    "Changes": [
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "db-primary.internal.myapp.com",
          "Type": "CNAME",
          "TTL": 60,
          "ResourceRecords": [{"Value": "production-aurora-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com"}]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "db-reader.internal.myapp.com",
          "Type": "CNAME",
          "TTL": 60,
          "ResourceRecords": [{"Value": "production-aurora-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com"}]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "redis.internal.myapp.com",
          "Type": "CNAME",
          "TTL": 60,
          "ResourceRecords": [{"Value": "production-redis.xxxxx.ng.0001.use1.cache.amazonaws.com"}]
        }
      },
      {
        "Action": "CREATE",
        "ResourceRecordSet": {
          "Name": "elasticsearch.internal.myapp.com",
          "Type": "CNAME",
          "TTL": 60,
          "ResourceRecords": [{"Value": "vpc-production-es-xxxxx.us-east-1.es.amazonaws.com"}]
        }
      }
    ]
  }'
```

---

## 8. DNSSEC Configuration

### Step 8.1: Enable DNSSEC Signing

```bash
# Step 1: Create KMS key for DNSSEC (must be in us-east-1)
aws kms create-key \
  --region us-east-1 \
  --description "DNSSEC signing key for myapp.com" \
  --key-usage SIGN_VERIFY \
  --key-spec ECC_NIST_P256 \
  --policy '{
    "Statement": [{
      "Sid": "Allow Route 53 DNSSEC",
      "Effect": "Allow",
      "Principal": {"Service": "dnssec-route53.amazonaws.com"},
      "Action": ["kms:DescribeKey", "kms:GetPublicKey", "kms:Sign"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {"aws:SourceAccount": "123456789012"},
        "ArnLike": {"aws:SourceArn": "arn:aws:route53:::hostedzone/*"}
      }
    }]
  }'

# Step 2: Create Key Signing Key (KSK)
aws route53 create-key-signing-key \
  --hosted-zone-id Z1234567890ABC \
  --name myapp-ksk-2024 \
  --key-management-service-arn arn:aws:kms:us-east-1:123456789012:key/xxx-yyy-zzz \
  --status ACTIVE

# Step 3: Enable DNSSEC signing
aws route53 enable-hosted-zone-dnssec \
  --hosted-zone-id Z1234567890ABC

# Step 4: Add DS record at registrar (output from above command)
```

---

## 9. Route 53 Resolver

### Step 9.1: Inbound Resolver Endpoint (On-Premises to AWS)

```bash
# Create inbound endpoint (on-prem can resolve AWS private DNS)
aws route53resolver create-resolver-endpoint \
  --name "inbound-from-onprem" \
  --direction INBOUND \
  --security-group-ids sg-resolver123456 \
  --ip-addresses SubnetId=subnet-private-az1,Ip=10.0.1.100 \
                 SubnetId=subnet-private-az2,Ip=10.0.2.100 \
  --tags Key=Environment,Value=production

# On-premises DNS forwarder should forward *.internal.myapp.com
# to these IP addresses (10.0.1.100, 10.0.2.100)
```

### Step 9.2: Outbound Resolver Endpoint (AWS to On-Premises)

```bash
# Create outbound endpoint (AWS can resolve on-prem DNS)
aws route53resolver create-resolver-endpoint \
  --name "outbound-to-onprem" \
  --direction OUTBOUND \
  --security-group-ids sg-resolver123456 \
  --ip-addresses SubnetId=subnet-private-az1 \
                 SubnetId=subnet-private-az2 \
  --tags Key=Environment,Value=production

# Create forwarding rule
aws route53resolver create-resolver-rule \
  --name "forward-to-onprem" \
  --rule-type FORWARD \
  --domain-name "corp.internal.com" \
  --resolver-endpoint-id rslvr-out-xxxxxxxxxxxxx \
  --target-ips Ip=192.168.1.10,Port=53 Ip=192.168.1.11,Port=53

# Associate rule with VPC
aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-xxxxxxxxxxxxx \
  --vpc-id vpc-0123456789abcdef0
```

---

## 10. Production Best Practices

### Step 10.1: TTL Strategy

```
Record Type          Recommended TTL    Use Case
─────────────────────────────────────────────────────
ALB Alias            N/A (auto 60s)     Production endpoints
Failover Records     60 seconds         Fast failover required
Weighted/Canary      60 seconds         Deployment traffic shifting
Internal Services    60-300 seconds     Service discovery
MX Records           3600 seconds       Email (rarely changes)
TXT Records          300-3600 seconds   Verification records
Static IPs           3600+ seconds      Rarely changing resources
```

### Step 10.2: DNS Migration Checklist

```bash
# Pre-migration (lower TTLs)
# 1. Lower ALL record TTLs to 60 seconds at OLD provider
# 2. Wait 48 hours for old TTL to expire from caches

# Migration day
# 3. Export all records from old provider
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC \
  --output json > dns-records-backup.json

# 4. Create all records in Route 53 (verify they match)
# 5. Update nameservers at registrar
# 6. Monitor with dig from multiple locations:
dig @ns-123.awsdns-15.com api.myapp.com A +short
dig @8.8.8.8 api.myapp.com A +short
dig @1.1.1.1 api.myapp.com A +short

# Post-migration
# 7. Wait 72 hours for full propagation
# 8. Restore TTLs to production values
# 9. Remove records from old provider
```

### Step 10.3: Route 53 Query Logging

```bash
# Enable query logging
aws route53 create-query-logging-config \
  --hosted-zone-id Z1234567890ABC \
  --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:/aws/route53/myapp.com

# Create the log group first
aws logs create-log-group \
  --log-group-name /aws/route53/myapp.com \
  --region us-east-1

aws logs put-retention-policy \
  --log-group-name /aws/route53/myapp.com \
  --retention-in-days 14

# Resource policy for Route 53 to write logs
aws logs put-resource-policy \
  --policy-name Route53QueryLogging \
  --policy-document '{
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "route53.amazonaws.com"},
      "Action": ["logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/route53/*"
    }]
  }'
```

### Step 10.4: Quick Reference Commands

```bash
# List all hosted zones
aws route53 list-hosted-zones --query 'HostedZones[].{Name:Name,Id:Id,Private:Config.PrivateZone}'

# List all records in a zone
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC

# Get specific record
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --query "ResourceRecordSets[?Name == 'api.myapp.com.']"

# Check health check status
aws route53 get-health-check-status --health-check-id hc-12345678-abcd-efgh-ijkl

# List all health checks
aws route53 list-health-checks \
  --query 'HealthChecks[].{Id:Id,Config:HealthCheckConfig.FullyQualifiedDomainName,Status:HealthCheckConfig.Type}'

# Test DNS resolution
dig api.myapp.com A +short
dig api.myapp.com CNAME +short
dig -x 54.23.45.67  # Reverse lookup
nslookup api.myapp.com ns-123.awsdns-15.com

# Export all records (backup)
aws route53 list-resource-record-sets --hosted-zone-id Z1234567890ABC > dns-backup-$(date +%Y%m%d).json
```

---
