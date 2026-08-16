# IAM & Security — Complete Knowledge Guide

> **Purpose**: Master AWS Identity and Access Management from fundamentals through enterprise patterns. This guide covers how authentication, authorization, and security services work together.

---

## 1. IAM Fundamentals: The Mental Model

### The Three Questions Every AWS Request Must Answer

```
Every API call to AWS goes through this evaluation:

1. AUTHENTICATION: "WHO are you?"
   → IAM User credentials, IAM Role session, Federated identity
   → Verified by: Access Key + Secret, Session Token, SAML assertion

2. AUTHORIZATION: "Are you ALLOWED to do this?"
   → Evaluated by: IAM policies, SCPs, Permission boundaries, Resource policies
   → Decision: Allow or Deny

3. AUDIT: "WHAT did you do?"
   → Recorded by: CloudTrail (every API call logged)
   → Stored in: S3, CloudWatch Logs, EventBridge
```

### IAM Principals (The "Who")

```
┌──────────────────────────────────────────────────────────────┐
│  IAM Principals (entities that can make requests)            │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  IAM Users      │  │  IAM Roles      │                  │
│  │  (long-lived)   │  │  (temporary)    │                  │
│  │                 │  │                 │                  │
│  │  - Has password │  │  - No password  │                  │
│  │  - Has access   │  │  - Assumed by   │                  │
│  │    keys         │  │    principals   │                  │
│  │  - Bad for      │  │  - Best practice│                  │
│  │    production   │  │    for everything│                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  AWS Services   │  │  Federated      │                  │
│  │  (service roles)│  │  Users          │                  │
│  │                 │  │  (external IdP) │                  │
│  │  - Lambda       │  │  - SSO/SAML     │                  │
│  │  - EC2          │  │  - OIDC (GitHub)│                  │
│  │  - ECS Task     │  │  - Cognito      │                  │
│  └─────────────────┘  └─────────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

### IAM Roles: How They Actually Work

```
Step 1: Trust Policy (WHO can assume this role)
┌─────────────────────────────────────────────┐
│  "Who is allowed to become this role?"       │
│                                              │
│  {                                           │
│    "Principal": {                            │
│      "Service": "ec2.amazonaws.com",         │
│      "AWS": "arn:aws:iam::OTHER:root",      │
│      "Federated": "cognito-identity..."     │
│    },                                        │
│    "Action": "sts:AssumeRole",              │
│    "Condition": { ... }                      │
│  }                                           │
└─────────────────────────────────────────────┘

Step 2: Permission Policy (WHAT can this role do)
┌─────────────────────────────────────────────┐
│  "Once assumed, what actions are allowed?"   │
│                                              │
│  {                                           │
│    "Effect": "Allow",                        │
│    "Action": "s3:GetObject",                │
│    "Resource": "arn:aws:s3:::my-bucket/*"   │
│  }                                           │
└─────────────────────────────────────────────┘

Step 3: AssumeRole (the actual process)
┌─────────────────────────────────────────────┐
│  sts:AssumeRole → Returns:                   │
│  - AccessKeyId (temporary)                   │
│  - SecretAccessKey (temporary)               │
│  - SessionToken (temporary)                  │
│  - Expiration (1 hour default)               │
│                                              │
│  These credentials ARE the role session      │
└─────────────────────────────────────────────┘
```

---

## 2. Policy Evaluation Logic (Deep Understanding)

### The Complete Authorization Flow

```
Request arrives → Policy Evaluation Engine:

Step 1: Explicit DENY check (across ALL policy types)
├── ANY explicit Deny in any policy → DENIED (final)
└── No explicit Deny found → continue

Step 2: Organization SCPs (if in an Organization)
├── Does the full SCP chain ALLOW this action?
├── SCPs on Root → OU → Sub-OU → Account (intersection)
└── If NOT allowed by SCPs → DENIED

Step 3: Resource-based policies (S3 bucket policy, KMS key policy, etc.)
├── Does the resource policy ALLOW this specific principal?
├── Same account: Resource policy OR identity policy can grant access
└── Cross account: BOTH resource policy AND identity policy must allow

Step 4: Identity-based policies (policies attached to IAM user/role)
├── Does any attached policy ALLOW this action?
└── If nothing allows → DENIED (implicit deny)

Step 5: Permission Boundaries (if attached to the principal)
├── Does the boundary ALLOW this action?
└── If not in boundary → DENIED

Step 6: Session Policies (if role was assumed with inline policy)
├── Does the session policy ALLOW this action?
└── If not in session policy → DENIED

RESULT: ALLOWED only if ALL applicable levels permit it
```

### Visual Decision Tree

```
                         ┌──────────────┐
                         │  Explicit     │
                    ┌────│  Deny?        │
                    │YES └──────┬───────┘
                    ▼           │NO
               ┌────────┐      ▼
               │ DENY   │  ┌──────────────┐
               └────────┘  │  SCP allows? │
                           └──────┬───────┘
                              │NO │YES
                              ▼   ▼
                         ┌────┐ ┌──────────────┐
                         │DENY│ │ Resource     │
                         └────┘ │ policy       │
                                │ allows?      │
                                └──────┬───────┘
                                   │   │
                              ┌────┘   └────┐
                              │SAME ACC     │CROSS ACC
                              ▼             ▼
                         (Continue)    (Need BOTH)
                              │             │
                              ▼             ▼
                         ┌──────────────┐ ┌──────────────┐
                         │ Identity     │ │ Identity     │
                         │ policy       │ │ policy       │
                         │ allows?      │ │ allows?      │
                         └──────┬───────┘ └──────┬───────┘
                            YES │                  │
                              ▼                  ▼
                         ┌──────────────┐  ┌──────────┐
                         │ Permission   │  │ Both     │
                         │ Boundary     │  │ allow?   │
                         │ allows?      │  └──────────┘
                         └──────┬───────┘
                            YES │
                              ▼
                         ┌──────────┐
                         │ ALLOW    │
                         └──────────┘
```

---

## 3. IAM Best Practices (Production Grade)

### The Principle of Least Privilege

```yaml
least_privilege_implementation:
  
  step_1_start_with_nothing:
    description: "Begin with zero permissions"
    policy: "Deny everything by default (IAM default)"
    
  step_2_add_what_is_needed:
    description: "Grant only specific actions on specific resources"
    example:
      # BAD (too broad):
      Action: "s3:*"
      Resource: "*"
      
      # GOOD (specific):
      Action: ["s3:GetObject", "s3:PutObject"]
      Resource: "arn:aws:s3:::app-data-bucket/uploads/*"
  
  step_3_add_conditions:
    description: "Further restrict with conditions"
    example:
      Condition:
        IpAddress:
          aws:SourceIp: "10.0.0.0/8"  # Only from VPC
        StringEquals:
          s3:x-amz-server-side-encryption: "aws:kms"  # Only encrypted
  
  step_4_review_and_tighten:
    description: "Use IAM Access Analyzer to find unused permissions"
    tools:
      - IAM Access Analyzer (policy generation from CloudTrail)
      - IAM Last Accessed (find unused permissions)
      - AWS Config (detect over-privileged roles)
```

### IAM Roles for Common Services

```
EC2 Instance Role:
├── Attached via Instance Profile
├── Credentials available at http://169.254.169.254/latest/meta-data/iam/
├── Auto-rotated by AWS (every ~6 hours)
├── ALWAYS use IMDSv2 (require token for metadata access)
└── Never put credentials on EC2 instances manually

ECS Task Role:
├── Per-task IAM role (different tasks = different permissions)
├── Available via ECS credential provider (169.254.170.2)
├── Separate from ECS Execution Role (which is for ECS agent)
└── Task Role = what YOUR code can do
    Execution Role = what ECS needs to pull images, write logs

Lambda Execution Role:
├── Attached when creating the function
├── Credentials injected as environment variables
├── Session duration = function timeout + 15 min
└── Keep permissions MINIMAL (Lambda is already scoped to single function)

EKS Pod Role (IRSA):
├── IAM Roles for Service Accounts
├── Each pod gets its own IAM role via K8s Service Account
├── Uses OIDC federation (EKS as the IdP)
└── Replaces: kube2iam, kiam (legacy approaches)
```

---

## 4. AWS Security Services Ecosystem

### The Security Stack (How They Work Together)

```
┌─────────────────────────────────────────────────────────────────┐
│  PREVENTIVE (Stop bad things from happening)                     │
│                                                                   │
│  ├── SCPs: Organization-wide permission boundaries               │
│  ├── Permission Boundaries: Per-user/role limits                 │
│  ├── VPC Security Groups: Network-level access control           │
│  ├── NACLs: Subnet-level stateless firewall                     │
│  ├── WAF: Web application firewall (Layer 7)                    │
│  ├── Shield: DDoS protection                                    │
│  └── Network Firewall: VPC-level stateful inspection            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  DETECTIVE (Find bad things that happened)                       │
│                                                                   │
│  ├── GuardDuty: ML-based threat detection                       │
│  │   └── Analyzes: CloudTrail, VPC Flow Logs, DNS logs          │
│  ├── Security Hub: Aggregates findings from all services        │
│  ├── Inspector: Vulnerability scanning (EC2, ECR, Lambda)       │
│  ├── Macie: PII/sensitive data detection in S3                  │
│  ├── Detective: Investigate and visualize security events        │
│  ├── IAM Access Analyzer: Find resources shared externally      │
│  └── AWS Config: Track configuration compliance                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  RESPONSIVE (React to incidents)                                 │
│                                                                   │
│  ├── EventBridge: Route security events to automation            │
│  ├── Lambda: Auto-remediate violations                           │
│  ├── Step Functions: Orchestrate incident response               │
│  ├── Systems Manager: Run commands on compromised instances     │
│  └── Forensics Account: Isolated investigation environment      │
└─────────────────────────────────────────────────────────────────┘
```

### GuardDuty: How It Detects Threats

```
Data Sources GuardDuty Analyzes:
├── CloudTrail Management Events (API calls)
│   └── Detects: Unusual API calls, impossible travel, recon activity
├── CloudTrail S3 Data Events
│   └── Detects: Anomalous S3 access patterns, data exfiltration
├── VPC Flow Logs
│   └── Detects: Port scanning, C2 communication, crypto mining
├── DNS Logs
│   └── Detects: Communication with known malicious domains
├── EKS Audit Logs
│   └── Detects: Privilege escalation, suspicious pod behavior
└── EBS Malware Scanning
    └── Detects: Malware on EBS volumes

Finding Types (examples):
├── Recon:EC2/PortScan - Instance scanning other IPs
├── UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B - Unusual login
├── CryptoCurrency:EC2/BitcoinTool.B - Mining software detected
├── Trojan:EC2/BlackholeTraffic - Traffic to known bad IPs
└── Exfiltration:S3/AnomalousBehavior - Unusual S3 download patterns
```

---

## 5. Encryption Strategy

### KMS Key Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│  KMS Key Types:                                              │
│                                                              │
│  AWS Managed Keys (aws/service)                              │
│  ├── Created automatically by AWS services                   │
│  ├── Rotation: Automatic (every year)                       │
│  ├── Management: AWS handles everything                      │
│  ├── Cost: Free (included in service usage)                 │
│  └── Use when: Default encryption, no special requirements  │
│                                                              │
│  Customer Managed Keys (CMK)                                 │
│  ├── You create and manage in your account                  │
│  ├── Rotation: Optional (configurable, 1 year)              │
│  ├── Key policy: YOU define who can use it                  │
│  ├── Cost: $1/month + $0.03 per 10,000 API calls           │
│  ├── Cross-account: Can share via key policy                │
│  └── Use when: Compliance requirements, cross-account,     │
│       custom rotation, deletion control                      │
│                                                              │
│  AWS Owned Keys                                              │
│  ├── Used internally by AWS services                        │
│  ├── You can't view, manage, or audit                       │
│  └── Used for: DynamoDB default encryption, etc.            │
└─────────────────────────────────────────────────────────────┘
```

### Envelope Encryption (How S3/EBS Encryption Works)

```
Step 1: KMS generates a Data Key
┌────────────────┐         ┌─────────────────────┐
│  KMS Master    │ ──────▶ │  Data Key (plain)   │  ← Used to encrypt data
│  Key (CMK)     │         │  Data Key (encrypted)│  ← Stored with data
└────────────────┘         └─────────────────────┘

Step 2: Data encrypted with Data Key
┌──────────────────────────────────────────────────────────┐
│  S3 Object = [Encrypted Data] + [Encrypted Data Key]     │
│                    ↑                      ↑               │
│             Encrypted with           Encrypted with       │
│             plain data key           KMS master key       │
└──────────────────────────────────────────────────────────┘

Step 3: To decrypt, need access to KMS key
├── Call KMS to decrypt the Data Key → Get plain Data Key
├── Use plain Data Key to decrypt the actual data
└── If you can't call KMS → Can't decrypt anything

WHY Envelope Encryption?
├── Master key NEVER leaves KMS (hardware security module)
├── Only small data key is encrypted/decrypted via network
├── Large data encrypted locally (fast, no network latency)
└── Rotating master key only requires re-encrypting data keys (not all data)
```

---

## 6. Network Security Architecture

### Defense in Depth (Layered Network Security)

```
                    INTERNET
                       │
          ┌────────────▼────────────────┐
Layer 1:  │  CloudFront + WAF + Shield  │  ← DDoS, OWASP Top 10
          └────────────┬────────────────┘
                       │
          ┌────────────▼────────────────┐
Layer 2:  │  Network Firewall           │  ← IDS/IPS, domain filtering
          └────────────┬────────────────┘
                       │
          ┌────────────▼────────────────┐
Layer 3:  │  ALB + Security Groups      │  ← Port/protocol filtering
          └────────────┬────────────────┘
                       │
          ┌────────────▼────────────────┐
Layer 4:  │  NACLs (Subnet level)       │  ← Stateless IP filtering
          └────────────┬────────────────┘
                       │
          ┌────────────▼────────────────┐
Layer 5:  │  Application (IAM + code)   │  ← Authentication + AuthZ
          └─────────────────────────────┘

Each layer catches what the previous missed.
Even if one layer is misconfigured, others provide protection.
```

### Security Groups vs NACLs

```
┌──────────────────────────────────────────────────────────────┐
│  Feature          │  Security Groups    │  NACLs             │
├───────────────────┼────────────────────┼────────────────────┤
│  Level            │  Instance/ENI      │  Subnet            │
│  Stateful         │  YES (return       │  NO (must allow    │
│                   │  traffic auto)     │  return traffic)   │
│  Rules            │  ALLOW only        │  ALLOW + DENY      │
│  Default          │  Deny all inbound  │  Allow all         │
│  Evaluation       │  All rules eval'd  │  Rules by number   │
│  Changes          │  Immediate         │  Immediate         │
│  Best for         │  Application-level │  Subnet-level      │
│                   │  micro-segmentation│  broad blocking    │
└──────────────────────────────────────────────────────────────┘

Production Pattern:
├── NACLs: Block known-bad CIDRs, enforce subnet tiers
├── Security Groups: Application-specific port/protocol rules
└── Both: Used together as defense-in-depth
```

---

## 7. IAM Identity Center (SSO)

### How SSO Works End-to-End

```
1. User goes to: https://mycompany.awsapps.com/start
2. Authenticates via corporate IdP (Azure AD, Okta, etc.)
3. Sees list of accounts + permission sets they can access
4. Clicks "Management Console" for desired account/role
5. Gets temporary credentials (1-12 hour session)
6. Works in AWS Console or CLI with those credentials

Behind the scenes:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Corporate    │     │ IAM Identity │     │ AWS Account  │
│ Directory    │────▶│ Center       │────▶│ (target)     │
│ (Azure AD)   │SCIM │              │ STS │              │
│              │Sync │ Permission   │     │ Role assumed │
│ Groups:      │     │ Sets:        │     │ with perms   │
│ - DevTeam    │     │ - ReadOnly   │     │ of the       │
│ - OpsTeam    │     │ - Deploy     │     │ permission   │
│ - AdminTeam  │     │ - Admin      │     │ set          │
└──────────────┘     └──────────────┘     └──────────────┘
```

### Permission Sets Explained

```
Permission Set = WHAT you can do once you access an account

It's essentially an IAM Role that gets created in EVERY account
where you assign it.

Example: "DeveloperAccess" permission set assigned to "Team-A" group
for accounts [Dev-A, Staging-A]:

Account: Dev-A
└── IAM Role: AWSReservedSSO_DeveloperAccess_abc123
    ├── Trust Policy: IAM Identity Center can assume
    ├── Permissions: PowerUserAccess managed policy
    └── Permission Boundary: DenyIAMChanges (optional)

Account: Staging-A
└── IAM Role: AWSReservedSSO_DeveloperAccess_def456
    ├── Same structure
    └── Same permissions

Team-A members → Pick an account → Get temporary credentials → Work
```

---

## 8. Security Incident Response

### Compromised EC2 Instance Playbook

```yaml
incident_response_ec2:
  step_1_isolate:
    - Create new Security Group with NO inbound/outbound rules
    - Apply to compromised instance (replaces all existing SGs)
    - DO NOT terminate (preserve evidence)
    - DO NOT stop (memory evidence lost)
    
  step_2_preserve_evidence:
    - Create EBS snapshot of all volumes
    - Enable VPC Flow Logs (if not already)
    - Capture instance metadata
    - Export CloudTrail logs for the instance's role
    - Copy instance memory (if possible) via SSM
    
  step_3_investigate:
    - Move snapshots to Forensics account
    - Launch investigation instance from snapshot
    - Analyze: processes, network connections, file changes
    - Check CloudTrail: what API calls did the role make?
    - Check GuardDuty: what was detected?
    
  step_4_contain:
    - Revoke all active sessions for the instance's role
    - Rotate any credentials the instance had access to
    - Check if lateral movement occurred (other accounts/instances)
    
  step_5_eradicate:
    - Terminate compromised instance
    - Deploy new instance from known-good AMI
    - Patch vulnerability that was exploited
    
  step_6_recover:
    - Restore from clean backup if needed
    - Verify service is functioning correctly
    - Monitor closely for 48 hours
    
  step_7_lessons_learned:
    - Document timeline, root cause, impact
    - Update detection rules
    - Improve prevention (patch management, etc.)
```

### Compromised IAM Credentials Playbook

```yaml
incident_response_iam:
  step_1_immediate:
    - Deactivate the access key: aws iam update-access-key --status Inactive
    - If IAM user: Attach DenyAll inline policy
    - If role: Update trust policy to deny all principals
    
  step_2_assess_blast_radius:
    - CloudTrail: What did the credentials do?
    - Time range: When were credentials leaked to when deactivated
    - Resources: What was created/modified/deleted?
    - Data: Was any data accessed/exfiltrated?
    
  step_3_remediate:
    - Delete the compromised access key
    - Rotate ALL credentials the user/role had access to
    - Revert unauthorized changes (CloudTrail as source of truth)
    - Scan for backdoors (new IAM users, roles, Lambda functions)
    
  step_4_prevent:
    - Implement MFA for all IAM users
    - Use IAM Roles instead of access keys
    - Implement credential rotation policy
    - Enable GuardDuty finding: UnauthorizedAccess/IAMUser
```

---

## 9. Key Takeaways for Interviews

```
The 10 IAM/Security concepts interviewers care about most:

1. Policy evaluation logic (explicit deny wins, then allow chain)
2. Cross-account access (trust policy + identity policy both needed)
3. Least privilege (start with nothing, add only what's needed)
4. Service-linked roles vs service roles (SLR = AWS managed, can't modify)
5. Temporary credentials (always prefer over long-lived keys)
6. Condition keys (powerful way to add context-based restrictions)
7. Resource-based policies (when they bypass identity policies)
8. Permission boundaries (delegate safely without full admin)
9. ABAC (Attribute-Based Access Control with tags)
10. Zero Trust (verify every request, assume breach)
```
