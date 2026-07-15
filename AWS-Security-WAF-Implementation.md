# AWS Security & End-to-End WAF Implementation — Deep-Dive Interview Q&A

## Table of Contents
1. [IAM Deep Dive](#iam-deep-dive)
2. [VPC Security](#vpc-security)
3. [WAF Implementation](#waf-implementation)
4. [Encryption & KMS](#encryption--kms)
5. [Security Services](#security-services)
6. [Tricky Scenarios](#tricky-scenarios)

---

## IAM Deep Dive


**Q1: Explain IAM policy evaluation logic. How does AWS decide Allow vs Deny?**

**A:**

```
Decision flow:
1. Default: DENY (everything denied unless explicitly allowed)
2. Evaluate all applicable policies
3. Explicit DENY found? → DENY (overrides everything)
4. Explicit ALLOW found? → ALLOW
5. No Allow found? → DENY (implicit deny)

Policy types evaluated (in order of authority):
├── Service Control Policies (SCP) - Organization-level guardrails
├── Resource-based policies - On the resource itself
├── IAM Permission Boundaries - Maximum permissions cap
├── Session policies - Temporary session restrictions
├── Identity-based policies - User/Role/Group policies
└── Effective permission = Intersection of all applicable policies
```

**Tricky**: An Allow in identity policy + Deny in SCP = DENY. SCPs are restrictive boundaries, NOT grants. An SCP allowing `s3:*` doesn't GIVE S3 access — it just doesn't BLOCK it.

**Cross-account access:**
- Same account: Identity OR resource policy can allow
- Cross-account: BOTH identity AND resource policy must allow

---

**Q2: Explain IAM Permission Boundaries. How do they differ from SCPs?**

**A:**

| Feature | Permission Boundary | SCP |
|---------|-------------------|-----|
| Scope | Individual IAM user/role | OU/Account in Organization |
| Purpose | Delegate IAM administration safely | Guardrails for entire account |
| Effect | Caps maximum permissions | Caps maximum permissions |
| Grants permissions? | No | No |
| Required for access? | Yes (intersection with identity policy) | Yes (intersection) |

**Use case**: Allow developers to create IAM roles but only with permissions they themselves have:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "iam:CreateRole",
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "iam:PermissionsBoundary": "arn:aws:iam::123456:policy/DevBoundary"
      }
    }
  }]
}
```

**Effective permissions = Identity Policy ∩ Permission Boundary ∩ SCP**

---

**Q3: How do you implement least-privilege IAM in a large organization?**

**A:**

1. **Start with deny-all**: No permissions by default
2. **Use IAM Access Analyzer**: Generates policies based on actual API usage (last 90 days)
3. **Access Advisor**: Shows which services were accessed and when
4. **CloudTrail-based policy generation**: Analyze actual API calls → generate minimal policy
5. **Service Control Policies**: Organization-wide guardrails
6. **Permission Boundaries**: Cap what developers can self-assign
7. **Condition keys**: Restrict by IP, MFA, time, resource tags

```json
{
  "Effect": "Allow",
  "Action": "ec2:*",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:RequestedRegion": ["us-east-1", "us-west-2"],
      "ec2:ResourceTag/Environment": "dev"
    },
    "Bool": {"aws:MultiFactorAuthPresent": "true"},
    "IpAddress": {"aws:SourceIp": "10.0.0.0/8"}
  }
}
```

---

## VPC Security

**Q4: Explain Security Groups vs NACLs. When does each apply?**

**A:**

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance/ENI | Subnet |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Rules | Allow only | Allow AND Deny |
| Evaluation | All rules evaluated together | Rules evaluated in order (first match) |
| Default | Deny all inbound, allow all outbound | Allow all (default NACL) |
| Association | Multiple SGs per ENI | One NACL per subnet |

**Tricky scenario**: Traffic from internet to EC2 in private subnet via NAT:
```
Inbound path:  Internet → NAT GW (public subnet NACL) → Private subnet NACL → EC2 SG
Return path:   EC2 SG (stateful, auto-allows return) → Private NACL (stateLESS, needs explicit allow!) → NAT GW
```

If private subnet NACL doesn't allow outbound to ephemeral ports (1024-65535), return traffic is DROPPED even though Security Group is stateful!

---

**Q5: Design a multi-layer VPC security architecture.**

**A:**

```
Internet
    ↓
┌─────────────────────────────────────────┐
│ Public Subnet (NACL: allow 80,443 in)   │
│ ├── ALB (SG: allow 80,443 from 0.0.0.0)│
│ └── NAT Gateway                         │
├─────────────────────────────────────────┤
│ Private Subnet (NACL: deny internet in) │
│ ├── App servers (SG: allow from ALB SG) │
│ └── ECS tasks                           │
├─────────────────────────────────────────┤
│ Data Subnet (NACL: allow only app subnet)│
│ ├── RDS (SG: allow 3306 from App SG)   │
│ └── ElastiCache (SG: allow 6379 from App SG) │
└─────────────────────────────────────────┘

VPC Endpoints (no internet needed):
├── S3 Gateway Endpoint
├── DynamoDB Gateway Endpoint
├── SSM Interface Endpoint
└── Secrets Manager Interface Endpoint
```

**Security layers:**
1. NACLs: Subnet-level coarse filtering
2. Security Groups: Instance-level fine-grained access
3. VPC Flow Logs: Traffic monitoring and forensics
4. VPC Endpoints: Keep AWS API traffic private
5. PrivateLink: Access third-party services privately

---

## WAF Implementation

**Q6: Walk through a complete WAF implementation from scratch for a production web application.**

**A:**

**Step 1: Architecture decision — Where to attach WAF:**
```
CloudFront → WAF (BEST: blocks at edge)
  OR
ALB → WAF (Good: regional protection)
  OR
API Gateway → WAF (Good: API-specific rules)
```

**Step 2: Create Web ACL with layered rules:**
```json
{
  "Rules": [
    {"Priority": 0, "Name": "AllowTrustedIPs", "Action": "Allow"},
    {"Priority": 1, "Name": "BlockBadIPs", "Action": "Block"},
    {"Priority": 2, "Name": "RateLimit-2000per5min", "Action": "Block"},
    {"Priority": 3, "Name": "AWS-BotControl", "Action": "Block/Challenge"},
    {"Priority": 4, "Name": "AWS-IPReputation", "Action": "Block"},
    {"Priority": 5, "Name": "AWS-CommonRuleSet", "Action": "Block"},
    {"Priority": 6, "Name": "AWS-SQLiRuleSet", "Action": "Block"},
    {"Priority": 7, "Name": "AWS-KnownBadInputs", "Action": "Block"},
    {"Priority": 8, "Name": "GeoBlock", "Action": "Block"},
    {"Priority": 9, "Name": "CustomAppRules", "Action": "Block"}
  ],
  "DefaultAction": "Allow"
}
```

**Step 3: Enable logging:**
- WAF logs → Kinesis Firehose → S3 → Athena (analysis)
- CloudWatch metrics for dashboards and alarms

**Step 4: Test with COUNT mode → Analyze logs → Switch to BLOCK**

**Step 5: Ongoing operations:**
- Review false positives weekly
- Update IP sets (threat intelligence feeds)
- Tune rate limits based on traffic patterns
- Review WAF Sampled Requests

---

**Q7: How do you implement IP whitelisting/blacklisting that updates automatically?**

**A:**

```python
# Lambda function to update WAF IP Set from threat feeds
import boto3
import requests

wafv2 = boto3.client('wafv2')

def lambda_handler(event, context):
    # Fetch threat intelligence feed
    response = requests.get('https://threat-feed.example.com/bad-ips.txt')
    bad_ips = [f"{ip}/32" for ip in response.text.strip().split('\n')]

    # Get current IP set
    ip_set = wafv2.get_ip_set(
        Name='BlockedIPs',
        Scope='REGIONAL',
        Id='ip-set-id'
    )

    # Update IP set
    wafv2.update_ip_set(
        Name='BlockedIPs',
        Scope='REGIONAL',
        Id='ip-set-id',
        Addresses=bad_ips,
        LockToken=ip_set['LockToken']
    )
```

**Automated architecture:**
```
EventBridge (hourly) → Lambda → Updates WAF IP Set
                                 ├── Threat feeds
                                 ├── AWS GuardDuty findings
                                 └── Custom honeypot detections
```

---

## Encryption & KMS

**Q8: Explain AWS KMS key hierarchy and envelope encryption.**

**A:**

```
AWS KMS
├── Customer Master Key (CMK) - Never leaves KMS
│   ├── AWS Managed Keys (aws/s3, aws/ebs) - AWS rotates
│   ├── Customer Managed Keys - You control rotation/policies
│   └── Custom Key Stores (CloudHSM-backed)
│
└── Envelope Encryption:
    CMK encrypts → Data Key (plaintext + encrypted)
    Data Key encrypts → Your actual data
    Store: Encrypted data + Encrypted data key
    Discard: Plaintext data key (never stored)
```

**Why envelope encryption:**
- CMK has 4KB encryption limit
- Sending all data to KMS = slow + expensive
- Instead: Get data key from KMS → encrypt locally → store encrypted key + data

**Key policies vs IAM policies:**
- Key policy is REQUIRED for any access (unlike S3 where IAM alone works)
- IAM policy alone CANNOT grant KMS access unless key policy allows it
- Cross-account: Key policy must explicitly allow the other account

---

**Q9: How do you implement end-to-end encryption for data at rest and in transit?**

**A:**

**At Rest:**
| Service | Encryption | Key Management |
|---------|-----------|----------------|
| S3 | SSE-S3, SSE-KMS, SSE-C | KMS CMK or S3-managed |
| EBS | AES-256 (KMS-backed) | Default or custom CMK |
| RDS | TDE or KMS | CMK per instance |
| DynamoDB | AWS-managed or CMK | Table-level encryption |
| EFS | KMS | CMK per filesystem |
| Secrets Manager | KMS | Automatic encryption |

**In Transit:**
| Layer | Implementation |
|-------|---------------|
| Client → CloudFront | TLS 1.2+ (ACM certificate) |
| CloudFront → ALB | TLS (origin protocol policy) |
| ALB → EC2/ECS | TLS or plain (internal) |
| App → RDS | SSL/TLS (rds-ca-2019 cert) |
| App → ElastiCache | TLS (in-transit encryption) |
| VPC → VPC | VPC Peering (encrypted by default) |

**Force encryption everywhere:**
```json
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": {"s3:x-amz-server-side-encryption": "aws:kms"}
  }
}
```

---

## Security Services

**Q10: Explain GuardDuty, Security Hub, Inspector, and Macie. How do they work together?**

**A:**

| Service | What It Does | Data Sources |
|---------|-------------|-------------|
| GuardDuty | Threat detection (AI/ML) | VPC Flow Logs, CloudTrail, DNS logs |
| Security Hub | Central security dashboard | Aggregates from all security services |
| Inspector | Vulnerability scanning | EC2 instances, ECR images, Lambda |
| Macie | Sensitive data discovery | S3 buckets (PII, credentials) |
| Config | Compliance rules | Resource configuration changes |
| Detective | Investigation/forensics | GuardDuty findings deep-dive |

**Integration flow:**
```
GuardDuty finding (e.g., "EC2 communicating with known C&C server")
    ↓
Security Hub (aggregates, prioritizes)
    ↓
EventBridge rule (matches finding type)
    ↓
Lambda (automated remediation)
    ├── Isolate EC2 (change SG to deny-all)
    ├── Create forensic snapshot (EBS)
    ├── Notify security team (SNS/PagerDuty)
    └── Update WAF IP blocklist
```

---

**Q11: How do you automate security incident response in AWS?**

**A:**

```yaml
# EventBridge rule for GuardDuty finding
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{"numeric": [">=", 7]}],
    "type": ["UnauthorizedAccess:EC2/MaliciousIPCaller"]
  }
}
```

**Auto-remediation Lambda:**
```python
def lambda_handler(event, context):
    instance_id = event['detail']['resource']['instanceDetails']['instanceId']

    # 1. Isolate instance
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=['sg-isolation-deny-all']
    )

    # 2. Snapshot for forensics
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'attachment.instance-id', 'Values': [instance_id]}]
    )
    for vol in volumes['Volumes']:
        ec2.create_snapshot(VolumeId=vol['VolumeId'], Description='Forensic snapshot')

    # 3. Tag as compromised
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[{'Key': 'SecurityStatus', 'Value': 'COMPROMISED'}]
    )

    # 4. Revoke IAM role sessions
    iam.put_role_policy(
        RoleName=instance_role,
        PolicyName='DenyAll',
        PolicyDocument='{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*"}]}'
    )
```

---

## Tricky Scenarios

**Q12: You find an IAM access key exposed on GitHub. Walk through the complete incident response.**

**A:**

**Immediate (within minutes):**
```bash
# 1. Disable the key (don't delete yet - need for investigation)
aws iam update-access-key --access-key-id AKIA... --status Inactive --user-name compromised-user

# 2. Attach deny-all policy to the user
aws iam put-user-policy --user-name compromised-user --policy-name DenyAll \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*"}]}'

# 3. Check for other keys on same user
aws iam list-access-keys --user-name compromised-user

# 4. Revoke all sessions (if role was assumed)
aws iam put-role-policy --role-name associated-role --policy-name RevokeOldSessions \
  --policy-document '...' # Deny with condition aws:TokenIssueTime < now
```

**Investigation:**
```bash
# 5. Check CloudTrail for key usage
aws cloudtrail lookup-events --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIA...

# What to look for:
# - New IAM users/roles created
# - New EC2 instances launched (crypto mining)
# - S3 data downloaded
# - Lambda functions created (persistence)
# - CloudTrail disabled (covering tracks)
```

**Remediation:**
- Delete exposed key, create new key
- Rotate ALL secrets the key had access to
- Check for backdoors (new users, roles, policies, Lambda functions)
- Enable MFA enforcement
- Set up GitHub secret scanning alerts
- Add SCP to prevent disabling CloudTrail

---

**Q13: How do you implement cross-account security in AWS Organizations?**

**A:**

```
Organization Root
├── Security OU
│   ├── Security Account (GuardDuty admin, Security Hub admin)
│   └── Log Archive Account (centralized CloudTrail, Config, VPC Flow Logs)
├── Production OU
│   ├── Prod Account A
│   └── Prod Account B
├── Development OU
│   └── Dev Accounts
└── Sandbox OU
    └── Sandbox Accounts

SCPs applied:
├── Root: Deny disabling CloudTrail, GuardDuty
├── Production OU: Deny all regions except us-east-1, eu-west-1
├── Development OU: Deny expensive instance types
└── Sandbox OU: Deny creating IAM users (SSO only)
```

**Cross-account access patterns:**
1. **IAM roles with trust policy** (recommended)
2. **Resource-based policies** (S3, KMS, SNS)
3. **AWS RAM** (Resource Access Manager - VPCs, subnets, Transit GW)
4. **AWS SSO** (centralized login, temporary credentials)

---

**Q14: What is the difference between AWS Config Rules, GuardDuty, and Security Hub findings?**

**A:**

| Feature | Config Rules | GuardDuty | Security Hub |
|---------|-------------|-----------|--------------|
| Purpose | Compliance checking | Threat detection | Aggregation & scoring |
| Detection | Configuration drift | Behavioral anomalies | Collects from all services |
| Example | "S3 bucket is public" | "EC2 talking to Bitcoin mining pool" | "45 critical findings across 10 accounts" |
| Remediation | Auto via SSM Automation | Manual or Lambda-triggered | Manual or automated |
| Cost model | Per rule evaluation | Per GB of data analyzed | Per finding ingested |
| Real-time | Near real-time (config changes) | Near real-time | Aggregated |

**Tricky**: Config Rules check CONFIGURATION (is this S3 bucket encrypted?). GuardDuty detects BEHAVIOR (is this IAM user downloading unusual amounts of data?). Security Hub PRIORITIZES across all findings with a security score.

---

**Q15: How do you ensure all S3 buckets are encrypted, private, and logged — across 50 accounts?**

**A:**

**Layer 1: Prevention (SCPs)**
```json
{
  "Effect": "Deny",
  "Action": "s3:PutBucketPublicAccessBlock",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "s3:PublicAccessBlockConfiguration/BlockPublicAcls": "true"
    }
  }
}
```

**Layer 2: Detection (AWS Config)**
```
Config Rules (deployed via StackSets):
├── s3-bucket-server-side-encryption-enabled
├── s3-bucket-public-read-prohibited
├── s3-bucket-public-write-prohibited
├── s3-bucket-logging-enabled
└── s3-bucket-ssl-requests-only
```

**Layer 3: Remediation (SSM Automation)**
```
Config Rule violation detected
    → EventBridge
    → Lambda/SSM Automation
    → Auto-enable encryption, block public access, enable logging
```

**Layer 4: Monitoring (Security Hub)**
- Aggregates compliance status across all accounts
- Security score per account
- Dashboard for leadership reporting
