# AWS Organizations, Control Tower, IAM & VPC — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Service | Purpose | Pricing | Key Feature |
|---------|---------|---------|-------------|
| **AWS Organizations** | Multi-account management | Free | SCPs (guardrails) |
| **Control Tower** | Managed landing zone | Free | Guardrails + Account Factory |
| **IAM** | Identity & access management | Free | Roles, policies, federation |
| **VPC** | Network isolation | Free (VPC) + resource costs | Subnets, SGs, NACLs |

---

## 🎯 Use Cases

### AWS Organizations
1. **Consolidated billing** — One bill, volume discounts, shared savings plans
2. **Security guardrails** — SCPs prevent dangerous actions across all accounts
3. **Account lifecycle** — Create, move, close accounts programmatically
4. **Resource sharing** — RAM shares VPCs, Transit Gateways across accounts
5. **Delegated admin** — GuardDuty/SecurityHub managed from Security account

### Control Tower
1. **Landing zone setup** — Automated multi-account foundation in hours
2. **Account vending** — Self-service account creation with baseline
3. **Compliance enforcement** — 300+ pre-built guardrails (preventive + detective)
4. **Drift detection** — Know when accounts deviate from baseline
5. **AFT (Account Factory for Terraform)** — IaC-driven account customization

### IAM
1. **Human access** — IAM Identity Center (SSO) for federated access
2. **Service roles** — EC2/ECS/Lambda assume roles for AWS API access
3. **Cross-account** — AssumeRole for multi-account architectures
4. **ABAC** — Tag-based access (team=X can manage team=X resources)
5. **Permission boundaries** — Delegate admin safely (cap permissions)

### VPC
1. **Network isolation** — Each workload in its own VPC
2. **Transit Gateway** — Hub-and-spoke connectivity (100s of VPCs)
3. **PrivateLink** — Expose services without public internet
4. **VPC Endpoints** — Access AWS services without NAT (free for S3!)
5. **Network Firewall** — IDS/IPS inspection for egress traffic

---

## ❓ Interview Questions & Answers

### Q1: How do SCPs interact with IAM policies? Draw the authorization flow.

**Answer:**

```
AUTHORIZATION EVALUATION ORDER:

1. Explicit DENY (anywhere) → DENIED (final, no override)
2. SCP check: Does the SCP chain ALLOW this action?
   └── NO → DENIED (SCP is an outer boundary)
3. Resource-based policy: Does it grant access?
   └── Same-account: Can grant independently
   └── Cross-account: BOTH resource policy + identity policy needed
4. Identity policy: Does IAM policy ALLOW?
   └── NO → DENIED (implicit deny)
5. Permission Boundary: Does it ALLOW?
   └── NO → DENIED (narrows identity policy)
6. Session Policy: Does it ALLOW?
   └── NO → DENIED (narrows assumed role session)

EFFECTIVE PERMISSIONS = INTERSECTION of all layers

KEY RULES:
├── SCPs CANNOT grant access (only restrict what's possible)
├── SCPs don't affect the Management Account
├── SCPs don't affect service-linked roles
├── Permission Boundaries CANNOT grant access (only restrict)
├── Explicit Deny ALWAYS wins (regardless of where it appears)
└── Cross-account: BOTH sides must allow
```

### Q2: Design a multi-account architecture for a 200-person engineering org with PCI compliance requirements.

**Answer:**

```
Root OU
├── Security OU (hardened SCPs)
│   ├── Log Archive Account (immutable logs, S3 Object Lock)
│   ├── Security Tooling (GuardDuty admin, Security Hub admin)
│   └── Forensics Account (isolated, no internet, for investigations)
│
├── Infrastructure OU
│   ├── Network Hub (Transit Gateway, DNS, Network Firewall, NAT GWs)
│   ├── Shared Services (CI/CD, AD, artifact repos)
│   └── Backup Account (centralized, cross-account vault)
│
├── PCI OU (strictest SCPs — encryption mandatory, no public access)
│   ├── PCI-Production (cardholder data environment)
│   ├── PCI-Staging
│   └── PCI-Development
│
├── Workloads OU (standard SCPs)
│   ├── Production Sub-OU
│   └── Non-Production Sub-OU
│
├── Sandbox OU (limited SCPs — budget caps, no expensive services)
│   └── Individual developer accounts
│
└── Suspended OU (deny-all SCP — quarantine)

KEY DESIGN DECISIONS:
├── PCI scope isolated to its own OU (separate SCPs, separate network)
├── Log Archive: S3 Object Lock (WORM — immutable for compliance)
├── Network Hub: Centralized egress (one point for inspection)
├── Shared Services: CI/CD tools, not workloads
└── Sandbox: Restricted but free for experimentation (budget alarms)
```

### Q3: Your VPC has 500 microservices that need to communicate. VPC peering is unmanageable. What's the architecture?

**Answer:**

```
SOLUTION TIERS (from simple → complex):

Tier 1: VPC Lattice (NEW — best for microservices)
├── Service-to-service communication with IAM auth
├── No NLBs, no PrivateLink endpoints needed
├── Cross-VPC, cross-account natively
├── Service-level auth policies (who can call who)
└── Best for: Most microservice communication patterns

Tier 2: Transit Gateway (for broad VPC connectivity)
├── Hub-and-spoke (one TGW, all VPCs connect)
├── Route table segmentation (Prod can't reach Dev)
├── 5000 routes per route table
└── Best for: VPC-to-VPC routing at scale

Tier 3: PrivateLink (for specific service exposure)
├── Producer exposes via NLB → Endpoint Service
├── Consumer creates VPC Endpoint → accesses service
├── No IP overlap issues! Works across accounts.
└── Best for: Specific services shared to specific consumers

RECOMMENDATION:
├── VPC Lattice: For application-layer (service mesh replacement)
├── Transit Gateway: For network-layer (IP connectivity)
├── PrivateLink: For specific cross-account service access
└── Use ALL THREE together (different layers, different purposes)
```

### Q4: Explain IAM Identity Center (SSO). How does it differ from IAM users?

**Answer:**

```
IAM USERS (Legacy):                    IAM IDENTITY CENTER (Modern):
├── Long-lived credentials             ├── Temporary session credentials
├── One per person per account         ├── One identity, many accounts
├── Console password + access keys     ├── Portal + federated login
├── No MFA by default                  ├── MFA enforced centrally
├── Can't revoke instantly             ├── Sessions expire automatically
├── Manage in EACH account             ├── Manage ONCE, centrally
├── Scale: Doesn't (100 accounts =     ├── Scale: Perfect (one place
│   100 sets of credentials!)          │   for all accounts + roles)
└── Use: Service accounts (legacy)     └── Use: All human access

IDENTITY CENTER ARCHITECTURE:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Corporate    │     │ IAM Identity │     │ AWS Accounts │
│ Directory    │────▶│ Center       │────▶│              │
│ (Azure AD)   │SCIM │              │ STS │ ├── Dev     │
│              │     │ Permission   │     │ ├── Staging │
│ Groups:      │     │ Sets:        │     │ ├── Prod    │
│ - Developers │     │ - ReadOnly   │     │ └── Sandbox │
│ - Admins     │     │ - Deploy     │     │              │
│ - Finance    │     │ - Admin      │     │ Each account │
│              │     │              │     │ gets IAM role│
└──────────────┘     └──────────────┘     └──────────────┘

Permission Set = Template for an IAM role
├── Created once in Identity Center
├── Deployed as IAM role to each assigned account
├── Examples: "ReadOnly", "PowerUser", "DBAdmin", "SecurityAudit"
└── Can include managed policies + inline + permission boundary
```

### Q5: How do you implement "zero standing privileges" with no permanent production access?

**Answer:**

```
JUST-IN-TIME (JIT) ACCESS:

Normal state: No one has production access (zero standing privileges)
Exception: Request → Approve → Grant (time-limited) → Auto-revoke

Implementation:
1. Developer requests access via Slack/portal
2. Manager/security approves (Step Functions workflow)
3. Lambda creates SSO account assignment (temporary)
4. EventBridge Scheduler triggers revocation after 4 hours
5. All actions logged for audit

# Grant access (Lambda):
sso_admin.create_account_assignment(
    InstanceArn=SSO_INSTANCE_ARN,
    TargetId='PROD_ACCOUNT_ID',
    TargetType='AWS_ACCOUNT',
    PermissionSetArn=INCIDENT_RESPONDER_ARN,  # Limited permissions
    PrincipalType='USER',
    PrincipalId=user_id
)

# Schedule revocation (4 hours later):
scheduler.create_schedule(
    Name=f'revoke-{request_id}',
    ScheduleExpression=f'at(2024-06-15T14:00:00)',
    Target={'Arn': REVOKE_LAMBDA_ARN}
)

# Result:
# ├── No permanent access to production
# ├── Access only when needed, auto-expires
# ├── Full audit trail (who, when, why, how long)
# └── Compliant with SOC2, PCI (principle of least privilege)
```

### Q6: You're running out of IP addresses in your VPC (/16 = 65K IPs). Production is live. What do you do?

**Answer:**

```
OPTIONS (no downtime):

Option 1: Add Secondary CIDR (RECOMMENDED — no disruption!)
├── aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxx --cidr-block 100.64.0.0/16
├── Create new subnets in secondary CIDR
├── Deploy new resources to new subnets
├── TGW/peering route tables updated automatically
└── Constraint: Max 5 CIDRs per VPC, can't overlap with peers

Option 2: IPv6 Dual-Stack
├── Assign IPv6 CIDR to VPC (AWS provides /56)
├── Create dual-stack subnets
├── IPv6 egress is FREE (no NAT Gateway needed!)
└── Reduces IPv4 pressure for new workloads

Option 3: Reclaim Unused IPs
├── Find unattached ENIs (Lambda VPC, deleted ELBs, stopped instances)
├── Release unused Elastic IPs
├── Delete old subnets with no active resources
└── Often recovers 10-20% of address space

Option 4: Network Load Balancer (for specific services)
├── Replace per-pod/per-instance IPs with shared NLB
├── Many backends share one IP endpoint
└── Reduces IP consumption for high-density services

BEST PRACTICE: Use IPAM (IP Address Management) from day 1
├── Allocate CIDRs from centralized pool
├── Prevent overlap across 100+ VPCs/accounts
├── Plan for growth (don't give /24 when /22 needed)
```

### Q7: Control Tower vs manual Organizations setup. What does CT give you that you'd otherwise build yourself?

**Answer:**

```
WHAT CONTROL TOWER AUTO-DEPLOYS (would take MONTHS manually):

1. Account Structure:
   ├── Log Archive account (CloudTrail + Config logs, S3 Object Lock)
   ├── Audit account (security tooling, cross-account roles)
   └── Auto-created during CT setup

2. Guardrails (300+):
   ├── Preventive: SCPs deployed to OUs (deny dangerous actions)
   ├── Detective: Config rules deployed to all accounts
   └── Proactive: CloudFormation Hooks

3. Account Factory:
   ├── Self-service account creation (via Service Catalog)
   ├── Pre-configured baseline (VPC, IAM roles, Config, CloudTrail)
   └── AFT for Terraform-based customization

4. SSO Integration:
   ├── IAM Identity Center pre-configured
   ├── AWSAdministratorAccess + AWSReadOnlyAccess permission sets
   └── Directory connector ready

5. Compliance Dashboard:
   ├── Single view of all accounts' compliance status
   ├── Drift detection (know when accounts deviate)
   └── Auto-remediation for drift (re-apply baseline)

6. Centralized Logging:
   ├── Organization CloudTrail (all accounts, all regions)
   ├── Config logs aggregated to Log Archive
   └── Retention and encryption pre-configured

TIME TO BUILD MANUALLY: 3-6 months
TIME WITH CONTROL TOWER: 1-2 hours
```

### Q8: How do VPC Security Groups differ from NACLs? Design defense-in-depth.

**Answer:**

```
SECURITY GROUPS:                        NACLs:
├── Instance/ENI level                  ├── Subnet level
├── Stateful (return traffic auto)      ├── Stateless (must allow return!)
├── ALLOW rules only                    ├── ALLOW + DENY rules
├── All rules evaluated                 ├── Rules evaluated by number
├── Default: Deny all inbound           ├── Default: Allow all
├── Changes: Immediate                  ├── Changes: Immediate
├── Best for: Application-level         ├── Best for: Subnet-level broad
│   micro-segmentation                  │   blocking

DEFENSE IN DEPTH (4 layers):

Layer 1: VPC-level → Network Firewall (IDS/IPS, domain filtering)
Layer 2: Subnet-level → NACLs (deny known-bad CIDRs, enforce tiers)
Layer 3: Instance-level → Security Groups (port/protocol per service)
Layer 4: Application-level → WAF + IAM + app authentication

EXAMPLE (3-tier app):
Web Subnet NACL:   Allow 443 from internet, deny all other inbound
App Subnet NACL:   Allow from Web subnet only, deny internet
Data Subnet NACL:  Allow from App subnet only on DB port, deny all other

Web SG:   Allow 443 from ALB SG
App SG:   Allow 8080 from Web SG only
DB SG:    Allow 5432 from App SG only (PostgreSQL)
Cache SG:  Allow 6379 from App SG only (Redis)
```

### Q9: Your Transit Gateway has 200 VPC attachments. How do you isolate Production from Development network traffic?

**Answer:**

```
TRANSIT GATEWAY ROUTE TABLE SEGMENTATION:

┌─────────────────────────────────────────────────────────────┐
│  Transit Gateway                                             │
│                                                              │
│  Route Table: "production"                                   │
│  ├── Routes to: All prod VPCs + Shared Services            │
│  ├── Blackhole: All dev/staging CIDRs (CANNOT reach!)      │
│  └── Associated with: All production VPC attachments        │
│                                                              │
│  Route Table: "non-production"                               │
│  ├── Routes to: All dev/staging VPCs + Shared Services     │
│  ├── Blackhole: All production CIDRs (CANNOT reach!)       │
│  └── Associated with: All non-prod VPC attachments         │
│                                                              │
│  Route Table: "shared-services"                              │
│  ├── Routes to: ALL VPCs (prod + non-prod)                 │
│  ├── Contains: CI/CD tools, DNS, monitoring                │
│  └── Associated with: Shared Services VPC attachment       │
│                                                              │
│  Result:                                                     │
│  ├── Prod VPC A ↔ Prod VPC B: ✅ (same route table)       │
│  ├── Dev VPC ↔ Prod VPC: ❌ (blackhole route!)             │
│  ├── Any VPC ↔ Shared Services: ✅ (shared route table)   │
│  └── Inspection: Add Network Firewall for prod egress      │
└─────────────────────────────────────────────────────────────┘
```

### Q10: An IAM role's cross-account AssumeRole is failing with AccessDenied. How do you systematically debug?

**Answer:**

```
DEBUGGING CHECKLIST (check in this order!):

1. TRUST POLICY (on target role):
   ├── Does it allow the calling principal?
   ├── Check: Principal ARN matches exactly?
   ├── Check: Condition keys match? (ExternalId, OrgID, SourceIp)
   └── Command: aws iam get-role --role-name TargetRole

2. CALLING PRINCIPAL's POLICY:
   ├── Does calling role have sts:AssumeRole permission?
   ├── Check: Resource ARN of target role is in the policy?
   └── Command: aws iam simulate-principal-policy

3. SCPs (on BOTH accounts):
   ├── Is there an SCP denying sts:AssumeRole?
   ├── Is there an SCP restricting cross-account actions?
   ├── Check: SCPs on the OU of the TARGET account
   └── Check: SCPs on the OU of the SOURCE account

4. PERMISSION BOUNDARY (on calling principal):
   ├── Does the boundary include sts:AssumeRole?
   └── If not → AssumeRole is blocked regardless of identity policy

5. VPC ENDPOINT POLICY (if calling from within VPC):
   ├── STS VPC endpoint may have restrictive policy
   └── Policy might only allow specific role ARNs

6. CONDITION KEYS (most missed!):
   ├── sts:ExternalId (caller must pass matching value)
   ├── aws:PrincipalOrgID (caller must be in same Org)
   ├── aws:SourceIp (caller must be from allowed IP)
   ├── sts:RoleSessionName (must match pattern)
   └── aws:PrincipalTag (ABAC — principal must have specific tags)

CLOUDTRAIL TELLS YOU WHY:
aws cloudtrail lookup-events --lookup-attributes \
  AttributeKey=EventName,AttributeValue=AssumeRole
# The errorMessage field says EXACTLY which check failed:
# "...with an explicit deny in a service control policy"  ← SCP!
# "...is not authorized to perform sts:AssumeRole"       ← Identity policy!
# "...Conditions were not met"                            ← Condition key!
```

---

## 🏆 Key Takeaways

```
Organizations:
├── Management Account = EMPTY (never run workloads!)
├── SCPs = outer boundary (restrict, never grant)
├── OU hierarchy = simple (3-4 levels max)
├── Delegated Admin = security services in Security Account
└── Suspended OU = quarantine for compromised accounts

Control Tower:
├── Use for: Greenfield, < 500 accounts, standard patterns
├── AFT for: Terraform-based account customization
├── Guardrails: Start with mandatory, add recommended gradually
├── Drift: Monitor and alert (don't ignore!)
└── Custom: Use StackSets for what CT doesn't cover

IAM:
├── NEVER create IAM users for humans (use Identity Center!)
├── Roles EVERYWHERE (EC2, ECS, Lambda, cross-account)
├── Permission Boundaries for safe delegation
├── ABAC (tags) for scalable access control
├── Regular access review (IAM Access Analyzer)
└── Break-glass procedure documented + tested quarterly

VPC:
├── Private subnets for everything (no public EC2!)
├── VPC Endpoints for AWS services (save NAT costs!)
├── Transit Gateway for multi-VPC connectivity
├── Network Firewall for egress inspection
├── IPAM for centralized IP address management
└── VPC Lattice for microservice communication
```
