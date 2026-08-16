# AWS Organizations & Control Tower — Complete Knowledge Guide

> **Purpose**: This document is a comprehensive reading guide to deeply understand AWS Organizations, Control Tower, and multi-account governance. Read it from top to bottom to build foundational knowledge.

---

## 1. What is AWS Organizations?

AWS Organizations is a **free** service that lets you consolidate multiple AWS accounts into a single organization for centralized management, billing, and governance.

### Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│  AWS Organization                                            │
│                                                              │
│  Management Account (Root)                                   │
│  ├── This account OWNS the organization                     │
│  ├── Cannot be changed (choose wisely!)                     │
│  ├── Pays all bills (consolidated billing)                  │
│  ├── NEVER run workloads here                               │
│  └── Only 1-2 people should have access                    │
│                                                              │
│  Organizational Units (OUs)                                  │
│  ├── Containers for grouping accounts                       │
│  ├── Can be nested (OU inside OU, max 5 levels)            │
│  ├── SCPs attach here and inherit downward                  │
│  └── Think of them like folders in a file system            │
│                                                              │
│  Member Accounts                                             │
│  ├── Individual AWS accounts in the organization            │
│  ├── Isolated by default (separate IAM, resources)          │
│  ├── Can be moved between OUs                               │
│  └── Maximum 10 accounts by default (increase via support)  │
│                                                              │
│  Policies                                                    │
│  ├── SCPs (Service Control Policies) — permission guardrails│
│  ├── Tag Policies — enforce tagging standards               │
│  ├── Backup Policies — centralized backup rules             │
│  └── AI Services Opt-out Policies                           │
└─────────────────────────────────────────────────────────────┘
```

### How SCPs Work (This is Critical to Understand)

**Key Rule**: SCPs do NOT grant permissions. They only RESTRICT what's possible.

```
Think of it like a Venn diagram:

┌────────────────────────────────────────┐
│  What the SCP ALLOWS (outer boundary)  │
│                                        │
│    ┌─────────────────────────────┐     │
│    │  What IAM policy GRANTS     │     │
│    │  (effective permissions)    │     │
│    └─────────────────────────────┘     │
│                                        │
└────────────────────────────────────────┘

Effective Permissions = INTERSECTION of (SCP allows) AND (IAM grants)

Example:
- SCP allows: ec2:*, s3:*, rds:* (but NOT iam:CreateUser)
- IAM policy grants: ec2:*, iam:*
- Effective: ec2:* only (iam:CreateUser blocked by SCP, rds:* not in IAM)
```

**What SCPs DON'T affect:**
- The management account (never)
- Service-linked roles
- Resource-based policies (for cross-account access from external)
- Actions performed by the organization itself

### SCP Inheritance Model

```
Root (FullAWSAccess — default SCP)
├── Security OU (SCP: DenyLeaveOrg + DenyDisableCloudTrail)
│   └── Security Account → Has both SCPs applied
├── Production OU (SCP: DenyUnapprovedRegions + RequireEncryption)
│   ├── Team-A Prod → Has Root + Production OU SCPs
│   └── Team-B Prod → Has Root + Production OU SCPs
└── Sandbox OU (SCP: LimitExpensiveServices + LimitRegions)
    └── Dev Account → Has Root + Sandbox SCPs

RULE: Account gets the INTERSECTION of all SCPs in its path
If any SCP in the path DENIES an action → it's denied (no override)
```

---

## 2. AWS Control Tower

Control Tower is an **opinionated**, AWS-managed landing zone that automates account provisioning, guardrails, and compliance. It's built ON TOP of Organizations.

### What Control Tower Gives You (That You'd Otherwise Build Manually)

| Feature | Without Control Tower | With Control Tower |
|---------|----------------------|-------------------|
| Account creation | Manual (Organizations API) | Account Factory (automated) |
| Guardrails | Write your own SCPs/Config rules | 300+ pre-built guardrails |
| Compliance | Build from scratch | Dashboard + drift detection |
| Log centralization | CloudTrail + S3 bucket setup | Auto-configured |
| SSO | Manual IAM Identity Center setup | Pre-integrated |
| Networking | Design from scratch | Optional VPC baseline |

### Control Tower Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Control Tower Landing Zone                                      │
│                                                                   │
│  Managed by Control Tower:                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Management Account                                         │  │
│  │  - Control Tower dashboard                                  │  │
│  │  - Organizations management                                 │  │
│  │  - CloudFormation StackSets                                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Auto-Created Accounts:                                           │
│  ┌──────────────────────┐  ┌──────────────────────┐             │
│  │  Log Archive Account │  │  Audit Account       │             │
│  │                      │  │                      │             │
│  │  Stores:             │  │  Contains:           │             │
│  │  - CloudTrail logs   │  │  - Config aggregator │             │
│  │  - Config logs       │  │  - SNS topics        │             │
│  │  - Access logs       │  │  - Cross-account     │             │
│  │                      │  │    audit roles       │             │
│  │  S3 Object Lock:     │  │                      │             │
│  │  ENABLED (immutable) │  │  Security team       │             │
│  └──────────────────────┘  │  operates from here  │             │
│                             └──────────────────────┘             │
│                                                                   │
│  Your Accounts:                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Created via Account Factory (Service Catalog)            │   │
│  │  - Pre-configured with baseline                           │   │
│  │  - VPC (optional)                                         │   │
│  │  - IAM roles for Control Tower                            │   │
│  │  - Config rules, CloudTrail, GuardDuty                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Guardrails: The Three Types

**1. Preventive Guardrails (SCPs)**
- Block actions BEFORE they happen
- Example: "Disallow changes to CloudTrail configuration"
- Implementation: SCP that denies `cloudtrail:StopLogging`
- **Cannot be overridden** (even by admin in member account)

**2. Detective Guardrails (AWS Config Rules)**
- Detect non-compliant resources AFTER creation
- Example: "Detect whether S3 buckets have public read access"
- Implementation: Config rule that evaluates S3 bucket policies
- **Reports violations** but doesn't prevent them

**3. Proactive Guardrails (CloudFormation Hooks)**
- Block non-compliant resources DURING deployment
- Example: "Prevent deploying unencrypted RDS via CloudFormation"
- Implementation: CloudFormation hook that validates templates
- **Blocks the deployment** if non-compliant

### Account Factory for Terraform (AFT)

AFT is the modern way to provision accounts with custom configurations:

```
Developer/Team Request
        │
        ▼
┌─────────────────────┐
│  AFT Pipeline       │
│                     │
│  1. Account Request │  ← Terraform code in Git
│     (defines account│
│      parameters)    │
│                     │
│  2. Account Created │  ← Organizations API
│     (Control Tower) │
│                     │
│  3. Global Custom.  │  ← Terraform applied to ALL accounts
│     (baseline code) │
│                     │
│  4. Account Custom. │  ← Terraform for THIS specific account
│     (team-specific) │
│                     │
│  5. Provisioning    │  ← Runs during account creation only
│     Customizations  │
└─────────────────────┘
        │
        ▼
  Account Ready (10-15 minutes)
```

---

## 3. Multi-Account Strategy Patterns

### The AWS Recommended Structure

```
Root
├── Security OU
│   ├── Log Archive (immutable logs)
│   ├── Security Tooling (GuardDuty, SecurityHub admin)
│   └── Forensics (isolated investigation)
│
├── Infrastructure OU
│   ├── Network Hub (Transit Gateway, DNS, Firewall)
│   ├── Shared Services (CI/CD, Active Directory)
│   └── Backup (centralized backup vault)
│
├── Workloads OU
│   ├── Production Sub-OU
│   │   ├── Team-A-Prod
│   │   ├── Team-B-Prod
│   │   └── Team-C-Prod
│   └── Non-Production Sub-OU
│       ├── Team-A-Dev
│       ├── Team-B-Staging
│       └── Team-C-Dev
│
├── Sandbox OU (developer experimentation)
│   └── Individual sandbox accounts
│
└── Suspended OU (decommissioned accounts)
```

### Why Separate Accounts? (The Key Insight)

```
Account = Hard Security Boundary

Within an account: IAM controls access (software boundary, can be misconfigured)
Between accounts: Account boundary (hardware-level isolation by AWS)

Benefits:
1. BLAST RADIUS: A compromised account can't reach others
2. BILLING: Natural cost separation per team/project
3. SERVICE LIMITS: Each account has its own limits (no contention)
4. COMPLIANCE: Clear scope for audits (PCI account vs non-PCI)
5. AUTONOMY: Teams own their accounts, move fast independently
```

---

## 4. Consolidated Billing & Cost Management

### How Billing Works in Organizations

```
┌──────────────────────────────────────────────┐
│  Management Account (Payer Account)          │
│                                              │
│  Receives ONE BILL for all member accounts   │
│                                              │
│  Benefits:                                   │
│  ├── Volume discounts (combined usage)       │
│  ├── Savings Plans shared across accounts    │
│  ├── Reserved Instance sharing               │
│  ├── Free Tier shared across org             │
│  └── Single payment method                   │
│                                              │
│  Cost Allocation:                            │
│  ├── Per-account breakdown available         │
│  ├── Tags for team/project attribution       │
│  └── AWS Cost Explorer for analysis          │
└──────────────────────────────────────────────┘
```

### Savings Plans Strategy for Organizations

```yaml
savings_plans_best_practices:
  compute_savings_plan:
    scope: Organization-wide (shared across all accounts)
    applies_to: EC2, Fargate, Lambda
    discount: 20-60% depending on commitment
    recommendation: Cover 70-80% of stable baseline
    
  ec2_instance_savings_plan:
    scope: Specific instance family + region
    discount: Higher than Compute SP (but less flexible)
    recommendation: Only for very predictable workloads
    
  strategy:
    - Start with Compute Savings Plans (most flexible)
    - Cover ORGANIZATION-wide stable compute
    - Let the plan float to wherever it saves most
    - Review monthly, adjust quarterly
```

---

## 5. Organization-Wide Services

### Services That Work at Organization Level

| Service | Organization Feature | Purpose |
|---------|---------------------|---------|
| CloudTrail | Organization Trail | Log ALL account activity centrally |
| AWS Config | Organization Rules | Compliance rules across all accounts |
| GuardDuty | Delegated Admin | Threat detection organization-wide |
| Security Hub | Delegated Admin | Security findings aggregation |
| Macie | Delegated Admin | PII detection across S3 buckets |
| IAM Access Analyzer | Organization | Find external access to resources |
| AWS Backup | Organization Policy | Backup rules applied to all accounts |
| RAM | Organization Sharing | Share resources without invitations |
| Service Catalog | Delegated Admin | Portfolio sharing to accounts |
| Systems Manager | Organization | Patch management across accounts |

### Delegated Administrator Pattern

```
BEFORE (everything in Management Account):
Management Account does EVERYTHING → Security risk!

AFTER (Delegated Admin):
Management Account (minimal - only org management)
    └── Delegates to Security Account:
        ├── GuardDuty admin
        ├── Security Hub admin
        ├── Macie admin
        └── Config aggregator admin

WHY: Reduce blast radius of Management Account
     Security team operates from Security Account
     Management Account has minimal permissions/access
```

---

## 6. Real-World Patterns & Anti-Patterns

### ✅ Best Practices

```yaml
do:
  - Keep Management Account EMPTY (no workloads ever)
  - Use IAM Identity Center (SSO) for all human access
  - Apply SCPs at OU level (not individual accounts)
  - Use Tag Policies to enforce naming conventions
  - Enable CloudTrail Organization Trail on day 1
  - Configure AWS Config in all accounts on day 1
  - Use IPAM for IP address management across accounts
  - Implement break-glass procedures and test quarterly
  - Use Account Factory for automated provisioning
  - Keep the OU hierarchy flat (max 3-4 levels)
```

### ❌ Anti-Patterns

```yaml
dont:
  - Run workloads in the Management Account
  - Give developers access to the Management Account
  - Create IAM users (use SSO exclusively)
  - Apply SCPs to individual accounts (use OUs)
  - Skip the Log Archive account setup
  - Use VPC peering at scale (use Transit Gateway)
  - Manually create accounts (use Account Factory)
  - Forget to document break-glass procedures
  - Allow root user access without hardware MFA
  - Create a "flat" organization with all accounts in Root
```

---

## 7. Hands-On Learning Path

### Progression for Mastery

```
Level 1 (Beginner):
├── Create an Organization with 2-3 accounts
├── Create an OU structure (Security, Workloads)
├── Apply a simple SCP (deny specific region)
├── Set up consolidated billing
└── Enable CloudTrail organization trail

Level 2 (Intermediate):
├── Deploy Control Tower in a new organization
├── Create accounts via Account Factory
├── Implement 10+ guardrails (preventive + detective)
├── Set up IAM Identity Center with permission sets
├── Configure AWS Config organization rules
└── Deploy StackSets for baseline to all accounts

Level 3 (Advanced):
├── Implement AFT with Terraform customizations
├── Design and deploy SCP strategy (50+ accounts)
├── Implement cross-account CI/CD pipeline
├── Set up centralized networking (Transit Gateway)
├── Build auto-remediation for compliance violations
├── Implement break-glass with automated testing
└── Design for regulatory compliance (PCI/HIPAA/SOC2)

Level 4 (Expert):
├── Migrate existing organization to Control Tower
├── Handle organization mergers (M&A)
├── Design for 500+ accounts at enterprise scale
├── Implement zero-trust with JIT access
├── Build custom governance automation
└── Optimize $1M+ monthly cloud spend
```

---

## 8. Key AWS Documentation to Read

1. **AWS Organizations User Guide** — Official deep-dive
2. **AWS Control Tower User Guide** — Landing zone automation
3. **AWS Well-Architected Framework: Security Pillar** — Multi-account security
4. **AWS Prescriptive Guidance: Organizing Your AWS Environment** — Enterprise patterns
5. **AWS Security Reference Architecture (SRA)** — Complete security design

---

## 9. Exam/Interview Tips

```
Questions to expect:
├── "How do SCPs interact with IAM policies?" (intersection model)
├── "What happens to the Management Account if an SCP denies all?" (nothing - exempt)
├── "How do you handle a compromised account?" (move to Suspended OU)
├── "What's the difference between preventive and detective guardrails?"
├── "How do you implement break-glass access?"
├── "Design a multi-account strategy for [scenario]"
└── "How do you migrate from single to multi-account?"

Key numbers to remember:
├── Max accounts per organization: 10 (default), increase via support
├── Max OUs: 1000
├── Max OU nesting: 5 levels
├── Max SCPs per entity: 5
├── Max SCP size: 5,120 characters
├── Max policies per organization: 1000
└── CloudTrail org trail: Logs ALL accounts automatically
```
