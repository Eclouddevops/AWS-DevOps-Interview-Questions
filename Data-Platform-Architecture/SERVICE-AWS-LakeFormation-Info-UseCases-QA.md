# AWS Lake Formation — Service Information, Use Cases & Interview Q&A

---

## 📋 Service Overview

| Attribute | Details |
|-----------|---------|
| **Service Name** | AWS Lake Formation |
| **Category** | Analytics / Data Governance |
| **Type** | Centralized Data Lake Security & Governance |
| **Engine** | Built on top of Glue Data Catalog + IAM + S3 |
| **Launched** | August 2019 (GA) |
| **Pricing Model** | Free (no additional charge — uses underlying services) |
| **Key Differentiator** | Fine-grained (column/row/cell-level) access control for data lakes |

---

## 🏗️ What AWS Lake Formation Does

Lake Formation is a **governance layer** that sits on top of your data lake (S3 + Glue Catalog) and provides fine-grained access control, cross-account sharing, and data discovery — all through a centralized permission model.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AWS LAKE FORMATION                               │
│                                                                       │
│  WITHOUT Lake Formation:                                              │
│  ├── Access controlled by: IAM policies + S3 bucket policies         │
│  ├── Granularity: Bucket or prefix level (coarse)                   │
│  ├── Cross-account: Complex resource policies                        │
│  └── Problem: Cannot restrict by TABLE, COLUMN, or ROW!             │
│                                                                       │
│  WITH Lake Formation:                                                 │
│  ├── Access controlled by: Lake Formation permissions                │
│  ├── Granularity: Database → Table → Column → Row → Cell            │
│  ├── Cross-account: Simple GRANT statements                         │
│  ├── Tag-based: ABAC (Attribute-Based Access Control)                │
│  └── Benefit: One place to manage ALL data access                    │
│                                                                       │
│  Architecture:                                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  LAKE FORMATION (Governance Layer)                            │   │
│  │                                                               │   │
│  │  ┌─────────────┐  ┌─────────────────┐  ┌────────────────┐  │   │
│  │  │ Permissions │  │ Data Catalog    │  │ Data Locations │  │   │
│  │  │ (GRANT/     │  │ (Tables,        │  │ (S3 paths      │  │   │
│  │  │  REVOKE)    │  │  Schemas,       │  │  registered    │  │   │
│  │  │             │  │  Partitions)    │  │  with LF)      │  │   │
│  │  └─────────────┘  └─────────────────┘  └────────────────┘  │   │
│  │                                                               │   │
│  │  ┌─────────────┐  ┌─────────────────┐  ┌────────────────┐  │   │
│  │  │ LF-Tags     │  │ Data Filters    │  │ Cross-Account  │  │   │
│  │  │ (Tag-based  │  │ (Row/Cell       │  │ Sharing        │  │   │
│  │  │  access)    │  │  filtering)     │  │ (Data Shares)  │  │   │
│  │  └─────────────┘  └─────────────────┘  └────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                           │                                           │
│                           ▼ (enforced when querying)                  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  QUERY ENGINES (all respect Lake Formation permissions)       │   │
│  │  ├── Amazon Athena                                            │   │
│  │  ├── Amazon Redshift Spectrum                                 │   │
│  │  ├── AWS Glue ETL                                             │   │
│  │  ├── Amazon EMR (with LF integration enabled)                │   │
│  │  └── Third-party engines (via credential vending)             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                           │                                           │
│                           ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  DATA STORAGE (S3)                                            │   │
│  │  ├── s3://datalake-bucket/bronze/                             │   │
│  │  ├── s3://datalake-bucket/silver/                             │   │
│  │  └── s3://datalake-bucket/gold/                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 💰 Pricing

**Lake Formation itself is FREE.** You pay only for underlying services:

| Component | Cost | Notes |
|-----------|------|-------|
| Lake Formation permissions | **$0** | No charge for GRANT/REVOKE/Tags |
| Lake Formation data sharing | **$0** | No charge for cross-account sharing |
| Glue Data Catalog | $1 per 100K objects/month | Tables, partitions, databases |
| S3 Storage | Standard S3 pricing | Your data lake storage |
| Athena queries | $5 per TB scanned | Querying data via LF |
| Glue ETL jobs | $0.44/DPU-hour | Processing data |
| EMR clusters | Instance + EMR pricing | Computing |

**Key insight:** Lake Formation adds ZERO cost to your data lake — it's purely a governance overlay. The value is security and compliance, not compute.

---

## 📊 Key Service Limits

| Limit | Value |
|-------|-------|
| Max databases per Catalog | 10,000 |
| Max tables per database | 200,000 |
| Max columns per table | 1,600 |
| Max LF-Tags per account | 50 |
| Max values per LF-Tag | 1,000 |
| Max data cells filters per table | 100 |
| Max principals per grant | 1 (grant per principal) |
| Max data lake admins | 30 |
| Max cross-account shares | 500 |
| Max registered locations | 10,000 |

---

## 🎯 Real-World Use Cases

### Use Case 1: Enterprise Data Lake Governance (Most Common)

```
PROBLEM: 50,000 tables in S3, 500 users, no access control at column level.
Analyst Bob should NOT see customer SSN, phone, or salary columns.

SOLUTION:
Lake Formation grants:
├── Analyst role → SELECT on customers table → columns: [name, email, region, signup_date]
│   (EXCLUDES: ssn, phone, salary, address — PII columns)
├── Data Engineer → ALL on all tables
├── Finance team → SELECT on revenue tables → all columns
└── ML team → SELECT on customer_behavior → all columns (non-PII aggregate data)

Implementation:
```

```python
import boto3
lf = boto3.client('lakeformation')

# Grant analyst access to specific columns only
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/AnalystRole'},
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'customers_db',
            'Name': 'customer_profiles',
            'ColumnNames': ['customer_id', 'name', 'email', 'region', 'signup_date']
            # Deliberately EXCLUDES: ssn, phone, salary, address
        }
    },
    Permissions=['SELECT']
)

# Result: When analyst queries in Athena:
# SELECT * FROM customers_db.customer_profiles
# → Only sees granted columns! SSN/phone/salary are INVISIBLE.
```

---

### Use Case 2: Cross-Account Data Sharing (Data Mesh)

```
┌─────────────────────────────────────────────────────────────────┐
│  PRODUCER ACCOUNT (Data Team — Account A)                        │
│  ├── Owns: s3://company-datalake/ (all data)                    │
│  ├── Catalog: 50,000 tables                                     │
│  └── Shares specific tables/columns to consumers                │
└───────────────────┬─────────────────────────────────────────────┘
                    │ Lake Formation Cross-Account Grant
          ┌─────────┼─────────────────────┐
          ▼         ▼                     ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ MARKETING    │ │ FINANCE      │ │ RISK         │
│ Account B    │ │ Account C    │ │ Account D    │
│              │ │              │ │              │
│ Sees:        │ │ Sees:        │ │ Sees:        │
│ - customer   │ │ - revenue    │ │ - transactions│
│   segments   │ │ - costs      │ │ - risk_scores│
│ - campaigns  │ │ - budgets    │ │ - all columns│
│ (no PII!)   │ │ (all columns)│ │ (including PII)│
└──────────────┘ └──────────────┘ └──────────────┘

Each consumer account:
├── Creates a Resource Link to the shared database
├── Queries via Athena in THEIR account
├── Gets billed for compute in THEIR account
├── Cannot see columns not shared (enforced by LF!)
└── No data copy — reads directly from producer's S3
```

---

### Use Case 3: Tag-Based Access Control (TBAC) — Dynamic Governance

```
PROBLEM: 500 new tables created monthly. Manually granting access to each? Impossible!

SOLUTION: Tag tables with attributes → Grant access to TAG VALUES (not individual tables)

LF-Tags:
├── "Sensitivity": [public, internal, confidential, restricted]
├── "Domain": [marketing, finance, engineering, hr, legal]
├── "PII": [true, false]

Grants based on tags:
├── Marketing team → Access all tables tagged: Domain=marketing AND Sensitivity∈[public, internal]
├── Finance team → Access all tables tagged: Domain=finance (any sensitivity)
├── Data Engineers → Access all tables tagged: Sensitivity∈[public, internal, confidential]
└── Compliance → Access ALL tags (full access for auditing)

MAGIC: When a NEW table is created and tagged Domain=marketing + Sensitivity=internal,
Marketing team AUTOMATICALLY gets access! No manual grant needed!
```

```python
# Create LF-Tags
lf.create_lf_tag(TagKey='Sensitivity', TagValues=['public', 'internal', 'confidential', 'restricted'])
lf.create_lf_tag(TagKey='Domain', TagValues=['marketing', 'finance', 'engineering', 'hr'])
lf.create_lf_tag(TagKey='PII', TagValues=['true', 'false'])

# Assign tags to a table
lf.add_lf_tags_to_resource(
    Resource={'Table': {'DatabaseName': 'analytics', 'Name': 'campaign_results'}},
    LFTags=[
        {'TagKey': 'Domain', 'TagValues': ['marketing']},
        {'TagKey': 'Sensitivity', 'TagValues': ['internal']},
        {'TagKey': 'PII', 'TagValues': ['false']}
    ]
)

# Grant access based on tags (applies to ALL matching tables!)
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/MarketingAnalyst'},
    Resource={
        'LFTagPolicy': {
            'ResourceType': 'TABLE',
            'Expression': [
                {'TagKey': 'Domain', 'TagValues': ['marketing']},
                {'TagKey': 'Sensitivity', 'TagValues': ['public', 'internal']}
            ]
        }
    },
    Permissions=['SELECT', 'DESCRIBE']
)
# ANY table tagged Domain=marketing + Sensitivity=public/internal → accessible!
# New tables with these tags → AUTOMATICALLY accessible!
```

---

### Use Case 4: Row-Level Security (Data Cell Filtering)

```
PROBLEM: Multi-tenant data in one table. Each customer should only see THEIR data.

Table: orders
├── customer_id | order_id | amount | status | date
├── ACME        | 001      | $500   | done   | 2024-01-01
├── ACME        | 002      | $300   | pending| 2024-01-02
├── GLOBEX      | 003      | $800   | done   | 2024-01-01
├── GLOBEX      | 004      | $200   | done   | 2024-01-03

ACME users should ONLY see rows where customer_id = 'ACME'
GLOBEX users should ONLY see rows where customer_id = 'GLOBEX'
```

```python
# Create data cells filter (row-level + column-level combined!)
lf.create_data_cells_filter(
    TableData={
        'DatabaseName': 'sales_db',
        'TableName': 'orders',
        'Name': 'acme_only_filter',
        'RowFilter': {
            'FilterExpression': "customer_id = 'ACME'"  # Row filter!
        },
        'ColumnNames': ['order_id', 'amount', 'status', 'date']
        # Excludes: customer_id (they already know who they are)
    }
)

# Grant ACME user access through the filter
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::ACME_ACCOUNT:role/ACMEAnalyst'},
    Resource={
        'DataCellsFilter': {
            'DatabaseName': 'sales_db',
            'TableName': 'orders',
            'Name': 'acme_only_filter'
        }
    },
    Permissions=['SELECT']
)
# ACME analyst queries: SELECT * FROM orders
# → Only sees ACME rows + only granted columns!
```

---

### Use Case 5: PII Data Masking / Anonymization

```
PROBLEM: Data scientists need customer data but must NOT see raw PII.

Original table (customer_profiles):
├── customer_id | name       | email             | ssn        | purchase_count
├── 001         | John Smith | john@company.com  | 123-45-6789| 42

What data scientists should see:
├── customer_id | name_hash        | email_domain | ssn  | purchase_count
├── 001         | a3b2c1...        | company.com  | ***  | 42

SOLUTION: Create a VIEW with masking + grant access to view only
```

```sql
-- Create masked view in Glue Catalog (via Athena)
CREATE OR REPLACE VIEW analytics.customers_masked AS
SELECT
    customer_id,
    MD5(name) as name_hash,                    -- Hash instead of raw name
    SPLIT_PART(email, '@', 2) as email_domain, -- Only domain, not full email
    '***-**-' || SUBSTR(ssn, 8) as ssn_masked, -- Last 4 only
    purchase_count,
    region,
    signup_year
FROM raw.customer_profiles;
```

```python
# Grant data scientists access to the MASKED view only
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/DataScientist'},
    Resource={
        'Table': {
            'DatabaseName': 'analytics',
            'Name': 'customers_masked'  # Masked view only!
        }
    },
    Permissions=['SELECT']
)
# They CANNOT access raw.customer_profiles (no grant exists)
```

---

### Use Case 6: Data Lineage & Discovery (Governed Tables)

```
Lake Formation Governed Tables (Iceberg-based):
├── ACID transactions on S3 data
├── Automatic compaction
├── Time travel (query historical versions)
├── Lineage tracking (who wrote what, when)
└── Governed by LF permissions

Use: Audit-critical data that requires:
├── "Who accessed this data?" → CloudTrail + LF audit logs
├── "What was the data at this point in time?" → Time travel
├── "Who is allowed to access this?" → LF permissions dashboard
└── "Has this data been modified since compliance check?" → Version tracking
```

---

## 🔑 Key Features to Know

### 1. Permission Model (CRITICAL to understand)

```
LAKE FORMATION PERMISSION MODEL:

Two paths to access data:
├── Path 1: IAM-only (traditional — Lake Formation not enforcing)
│   └── IAM policy grants s3:GetObject on data path → direct access
│
└── Path 2: Lake Formation (recommended — centralized governance)
    └── LF permission grants SELECT on table → credential vending → S3 access

HOW CREDENTIAL VENDING WORKS:
1. User queries via Athena: SELECT * FROM db.table
2. Athena asks Lake Formation: "Can this user SELECT from this table?"
3. Lake Formation checks permissions → YES (specific columns only)
4. Lake Formation vends TEMPORARY S3 credentials (scoped to those columns!)
5. Athena uses temp credentials to read ONLY allowed column files from S3
6. User sees only granted columns

KEY CONCEPT: "IAM_ALLOWED_PRINCIPALS"
├── When a database has IAM_ALLOWED_PRINCIPALS as default permissions,
│   it means BOTH IAM and LF permissions are checked
├── To enable LF-only mode: Remove IAM_ALLOWED_PRINCIPALS
│   → Now ONLY Lake Formation permissions matter
└── GOTCHA: Must migrate gradually (don't break existing access!)
```

### 2. Data Lake Administrator

```
LF Admin = Super-user for data governance (NOT an IAM admin!)

Powers:
├── Grant/revoke permissions on ANY data lake resource
├── Create databases, register locations
├── Manage LF-Tags
├── Share data cross-account
└── Override all LF permissions

Who should be LF Admin:
├── Data governance team lead
├── Platform team (for automation)
└── NOT individual data engineers (too much power!)

Best practice: 2-3 LF admins + service role for automation
```

### 3. Registered Locations

```
WHY register S3 locations with Lake Formation?

Before registration: IAM controls S3 access (coarse-grained)
After registration: Lake Formation controls S3 access (fine-grained)

Registration process:
1. Tell LF: "This S3 path is part of the data lake"
2. LF takes over access control (credential vending)
3. Services (Athena, EMR) get temp credentials FROM LF (not IAM directly)

lf.register_resource(
    ResourceArn='arn:aws:s3:::company-datalake',
    UseServiceLinkedRole=True  # LF manages access via SLR
)
```

### 4. LF-Tags (Tag-Based Access Control)

```
CONCEPT: Instead of granting access to INDIVIDUAL resources,
grant access to TAG VALUES. New resources auto-inherit permissions.

Traditional (doesn't scale):
├── GRANT SELECT ON table_1 TO analyst_role
├── GRANT SELECT ON table_2 TO analyst_role
├── GRANT SELECT ON table_3 TO analyst_role
├── ... (repeat for 500 tables — nightmare!)
└── NEW table created → MANUALLY grant again

Tag-Based (scales infinitely):
├── Tag all marketing tables: Domain=marketing
├── GRANT SELECT ON (Domain=marketing) TO analyst_role
├── NEW marketing table created + tagged → ACCESS AUTOMATIC!
└── One grant covers ALL current AND future matching tables
```

### 5. Cross-Account Sharing

```
SHARING MECHANISMS:

Method 1: Named Resource Sharing (specific tables)
├── Producer grants to specific consumer account/role
├── Consumer creates Resource Link
├── Good for: Few specific tables shared to few accounts

Method 2: Tag-Based Sharing (dynamic, scalable)
├── Producer shares tag expression to consumer
├── Any table matching tags is accessible
├── Good for: Many tables, many consumers, data mesh

Method 3: AWS RAM Integration
├── Share LF resources via Resource Access Manager
├── Can share to entire Organization
├── Good for: Organization-wide data catalog sharing

COMPARISON:
├── Named: Explicit (1 table → 1 consumer)
├── Tag-Based: Dynamic (matching tables → all matching consumers)
└── RAM: Broad (entire databases → all org accounts)
```

### 6. Hybrid Mode (IAM + Lake Formation)

```
MIGRATION CHALLENGE:
├── Existing data lake uses IAM policies for access
├── Can't switch to LF-only immediately (breaks existing access!)
├── Need gradual migration

HYBRID MODE:
├── Both IAM AND Lake Formation permissions apply
├── Access = IAM allows AND LF allows (intersection)
├── Start: Register locations with LF, keep IAM working
├── Gradually: Add LF permissions, remove IAM permissions
├── End: LF-only mode (IAM_ALLOWED_PRINCIPALS removed)

Migration steps:
1. Enable Lake Formation
2. Register S3 locations
3. Grant LF permissions (mirrors existing IAM access)
4. Verify: Users still have access via LF
5. Remove IAM_ALLOWED_PRINCIPALS (LF takes over)
6. Remove old S3 IAM policies (cleanup)
```

---

## ❓ Interview Questions & Answers

### Q1: Explain how Lake Formation permissions differ from IAM policies. Why do you need both?

**Answer:**

```
IAM POLICIES vs LAKE FORMATION PERMISSIONS:

IAM:
├── Operates at: AWS API level (actions on services)
├── Granularity: S3 bucket / prefix level
├── Example: "Allow s3:GetObject on s3://bucket/data/*"
├── Problem: Cannot restrict by TABLE or COLUMN
├── Scale: One policy per role, complex at 100s of tables
└── Cross-account: Complex resource policies required

LAKE FORMATION:
├── Operates at: Data Catalog level (logical tables/columns)
├── Granularity: Database → Table → Column → Row → Cell
├── Example: "Allow SELECT on customers table, columns [name, email]"
├── Benefit: Users don't need to know S3 paths
├── Scale: Tag-based grants apply to 1000s of tables automatically
└── Cross-account: Simple GRANT statement

WHY NEED BOTH:
├── IAM: Still needed for SERVICE-LEVEL access
│   └── "Can this role call athena:StartQueryExecution?"
│   └── "Can this role call glue:GetTable?"
├── Lake Formation: For DATA-LEVEL access
│   └── "Can this role SELECT from this table/column?"
└── Both work together:
    1. IAM verifies: "Are you allowed to use Athena?" → YES
    2. Lake Formation verifies: "Can you see this table?" → YES (columns X, Y, Z)
    3. Query succeeds with only granted columns
```

**Key insight for interviews:**
```
IAM = "Can you use the service?"
Lake Formation = "What data can you see through that service?"

Example:
├── IAM: Allow athena:* (can use Athena)
├── LF: SELECT on customers (name, email only)
└── Result: User can run Athena queries, but only sees name+email columns

Without LF: IAM allows s3:GetObject on entire bucket → sees EVERYTHING!
```

---

### Q2: How do you implement column-level security for a table where different teams need different columns?

**Answer:**

```python
import boto3
lf = boto3.client('lakeformation')

# Table: customer_360
# Columns: customer_id, name, email, phone, ssn, purchase_history, 
#           credit_score, address, lifetime_value, churn_risk

# Marketing team: See behavior data, NO PII
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/MarketingTeam'},
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'analytics',
            'Name': 'customer_360',
            'ColumnNames': ['customer_id', 'purchase_history', 'lifetime_value', 'churn_risk']
            # EXCLUDED: name, email, phone, ssn, credit_score, address (PII!)
        }
    },
    Permissions=['SELECT']
)

# Risk team: See financial data including PII
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/RiskTeam'},
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'analytics',
            'Name': 'customer_360',
            'ColumnNames': ['customer_id', 'name', 'credit_score', 'purchase_history', 'lifetime_value']
            # Has name (for identification) + credit_score (for risk)
            # EXCLUDED: ssn, phone, address, email (not needed for risk analysis)
        }
    },
    Permissions=['SELECT']
)

# Data Engineering: Full access (for ETL processing)
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/DataEngineer'},
    Resource={
        'Table': {
            'DatabaseName': 'analytics',
            'Name': 'customer_360'
            # No ColumnNames = ALL columns
        }
    },
    Permissions=['SELECT', 'INSERT', 'DELETE', 'DESCRIBE', 'ALTER']
)

# Compliance/Audit: Full access + grant ability
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/ComplianceTeam'},
    Resource={
        'Table': {
            'DatabaseName': 'analytics',
            'Name': 'customer_360'
        }
    },
    Permissions=['ALL'],
    PermissionsWithGrantOption=['ALL']  # Can delegate to others!
)
```

**What happens when each team queries:**
```sql
-- Marketing team runs:
SELECT * FROM analytics.customer_360 WHERE churn_risk > 0.8;
-- Result: Only sees customer_id, purchase_history, lifetime_value, churn_risk
-- Other columns are INVISIBLE (not even in schema!)

-- Risk team runs:
SELECT * FROM analytics.customer_360;
-- Result: Sees customer_id, name, credit_score, purchase_history, lifetime_value
-- SSN, phone, address are INVISIBLE

-- Data Engineer:
SELECT * FROM analytics.customer_360;
-- Result: Sees ALL columns (full access)
```

---

### Q3: Explain LF-Tags (Tag-Based Access Control). How is it better than named-resource grants?

**Answer:**

```
PROBLEM WITH NAMED RESOURCE GRANTS:
├── 500 tables today → 500 individual GRANT statements
├── 50 new tables/month → 50 new grants EVERY month
├── 20 teams → 500 × 20 = 10,000 grant combinations!
├── Forgot to grant? Team can't access their data.
├── Person leaves? Must revoke from each resource individually.
└── DOES NOT SCALE.

SOLUTION: LF-TAGS (grant access to TAG VALUES, not resources)

Step 1: Define tag taxonomy
├── Domain: [marketing, finance, engineering, hr, operations]
├── Sensitivity: [public, internal, confidential, restricted]
├── DataLayer: [bronze, silver, gold]
├── Environment: [production, staging, development]
└── PII: [true, false]

Step 2: Tag resources (one-time, or automated via pipeline)
├── Table "campaign_results" → {Domain: marketing, Sensitivity: internal, PII: false}
├── Table "customer_pii" → {Domain: marketing, Sensitivity: restricted, PII: true}
├── Table "revenue_report" → {Domain: finance, Sensitivity: confidential, PII: false}
└── (New tables are tagged during creation by ETL pipeline)

Step 3: Grant based on tag expressions (ONCE — covers all current + future!)
├── Marketing Analyst:
│   └── WHERE Domain=marketing AND Sensitivity∈[public, internal] AND PII=false
│       → Sees campaign_results ✓ (matches)
│       → Does NOT see customer_pii ✗ (PII=true blocks it!)
│
├── Finance Team:
│   └── WHERE Domain=finance AND Sensitivity∈[public, internal, confidential]
│       → Sees revenue_report ✓
│
├── Data Engineers:
│   └── WHERE DataLayer∈[bronze, silver, gold]
│       → Sees ALL tables (all have a DataLayer tag)
│
└── WHEN NEW TABLE CREATED with Domain=marketing, Sensitivity=internal:
    → Marketing Analyst AUTOMATICALLY gets access! (tags match expression)
    → No manual grant needed! SCALES INFINITELY!
```

**Implementation:**
```python
# ONE-TIME tag-based grant (replaces 500 individual grants):
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/MarketingAnalyst'},
    Resource={
        'LFTagPolicy': {
            'ResourceType': 'TABLE',
            'Expression': [
                {'TagKey': 'Domain', 'TagValues': ['marketing']},
                {'TagKey': 'Sensitivity', 'TagValues': ['public', 'internal']},
                {'TagKey': 'PII', 'TagValues': ['false']}
            ]
        }
    },
    Permissions=['SELECT', 'DESCRIBE']
)
# This SINGLE grant covers ALL tables matching the expression!
# New tables auto-match! Old tables auto-match! No manual work!
```

---

### Q4: How do you implement cross-account data sharing using Lake Formation? Walk through the complete setup.

**Answer:**

```
SETUP OVERVIEW (Producer → Consumer):

Producer Account (111111111111):
├── Owns S3 data
├── Has Glue Catalog tables
├── Grants access to Consumer

Consumer Account (222222222222):
├── Creates Resource Link (pointer to Producer's table)
├── Queries data (Athena in Consumer account)
├── Pays for compute (own account)
├── Sees ONLY granted columns

STEP-BY-STEP:
```

**Step 1: Producer — Enable cross-account sharing**
```python
# In Producer Account:

# 1a. Register data location with Lake Formation
lf.register_resource(
    ResourceArn='arn:aws:s3:::producer-datalake',
    UseServiceLinkedRole=True
)

# 1b. Grant permissions to Consumer account
lf.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': '222222222222'  # Consumer account ID
    },
    Resource={
        'TableWithColumns': {
            'DatabaseName': 'analytics',
            'Name': 'orders',
            'ColumnNames': ['order_id', 'customer_id', 'amount', 'status', 'order_date']
            # Consumer can see these 5 columns (out of 12 total)
        }
    },
    Permissions=['SELECT', 'DESCRIBE'],
    PermissionsWithGrantOption=['SELECT']  # Consumer can re-grant to their users
)
```

**Step 2: Consumer — Accept and create Resource Link**
```python
# In Consumer Account:

# 2a. Accept the RAM share (if using RAM-based sharing)
# OR: Automatically available if using Organization-based sharing

# 2b. Create Resource Link (pointer to Producer's database)
glue = boto3.client('glue')
glue.create_database(
    DatabaseInput={
        'Name': 'shared_analytics',  # Local name in Consumer's catalog
        'TargetDatabase': {
            'CatalogId': '111111111111',  # Producer account
            'DatabaseName': 'analytics'    # Producer's database
        }
    }
)

# 2c. Grant local users access (Consumer's own LF permissions)
lf.grant_permissions(
    Principal={
        'DataLakePrincipalIdentifier': 'arn:aws:iam::222222222222:role/ConsumerAnalyst'
    },
    Resource={
        'Table': {
            'DatabaseName': 'shared_analytics',
            'Name': 'orders'
        }
    },
    Permissions=['SELECT']
)
```

**Step 3: Consumer queries data**
```sql
-- Consumer analyst runs in Athena (Consumer account):
SELECT customer_id, SUM(amount) as total
FROM shared_analytics.orders
WHERE order_date >= '2024-01-01'
GROUP BY customer_id;

-- They see: order_id, customer_id, amount, status, order_date (5 columns)
-- They DON'T see: internal columns like cost_of_goods, margin, warehouse_id, etc.
-- Data is read directly from Producer's S3 (no copy!)
-- Compute cost: Consumer pays (Athena scan charges)
```

---

### Q5: Your company has 50,000 tables in the data lake. A new compliance requirement says all PII columns must be restricted to authorized users only. How do you implement this at scale with Lake Formation?

**Answer:**

```
CHALLENGE: 50,000 tables, unknown number have PII columns
REQUIREMENT: Only authorized roles can access PII data

APPROACH: Tag-Based Access Control (TBAC) + Automated Classification

Phase 1: DISCOVER PII (automated)
├── Run AWS Macie on S3 data lake → identifies PII locations
├── Map Macie findings to Glue Catalog columns
├── Auto-tag tables/columns that contain PII
└── Output: List of tables/columns with PII

Phase 2: TAG with LF-Tags
```

```python
# Automated PII tagging script (runs after Macie scan)
import boto3

lf = boto3.client('lakeformation')
glue = boto3.client('glue')
macie = boto3.client('macie2')

# Create PII tag
lf.create_lf_tag(TagKey='PII', TagValues=['true', 'false'])
lf.create_lf_tag(TagKey='PIIType', TagValues=['ssn', 'email', 'phone', 'address', 'financial', 'none'])

def tag_pii_tables():
    """Auto-tag tables based on Macie PII findings."""
    
    # Get all tables
    databases = glue.get_databases()['DatabaseList']
    
    for db in databases:
        tables = glue.get_tables(DatabaseName=db['Name'])['TableList']
        
        for table in tables:
            # Check if any column is PII (from Macie findings or naming convention)
            pii_columns = identify_pii_columns(table)
            
            if pii_columns:
                # Tag the TABLE as containing PII
                lf.add_lf_tags_to_resource(
                    Resource={'Table': {'DatabaseName': db['Name'], 'Name': table['Name']}},
                    LFTags=[{'TagKey': 'PII', 'TagValues': ['true']}]
                )
                
                # Tag specific COLUMNS with PII type
                for col_name, pii_type in pii_columns.items():
                    lf.add_lf_tags_to_resource(
                        Resource={
                            'TableWithColumns': {
                                'DatabaseName': db['Name'],
                                'Name': table['Name'],
                                'ColumnNames': [col_name]
                            }
                        },
                        LFTags=[{'TagKey': 'PIIType', 'TagValues': [pii_type]}]
                    )
            else:
                # Tag as non-PII
                lf.add_lf_tags_to_resource(
                    Resource={'Table': {'DatabaseName': db['Name'], 'Name': table['Name']}},
                    LFTags=[{'TagKey': 'PII', 'TagValues': ['false']}]
                )

def identify_pii_columns(table):
    """Identify PII columns by name pattern + Macie results."""
    pii_patterns = {
        'ssn': ['ssn', 'social_security', 'tax_id'],
        'email': ['email', 'email_address', 'e_mail'],
        'phone': ['phone', 'mobile', 'telephone', 'cell'],
        'address': ['address', 'street', 'zip_code', 'postal'],
        'financial': ['credit_card', 'card_number', 'account_number', 'salary'],
    }
    
    pii_columns = {}
    for col in table.get('StorageDescriptor', {}).get('Columns', []):
        col_lower = col['Name'].lower()
        for pii_type, patterns in pii_patterns.items():
            if any(p in col_lower for p in patterns):
                pii_columns[col['Name']] = pii_type
    
    return pii_columns
```

```python
# Phase 3: GRANT based on PII tags

# Standard users: Can access NON-PII data only
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/StandardAnalyst'},
    Resource={
        'LFTagPolicy': {
            'ResourceType': 'TABLE',
            'Expression': [
                {'TagKey': 'PII', 'TagValues': ['false']}  # Only non-PII tables!
            ]
        }
    },
    Permissions=['SELECT', 'DESCRIBE']
)

# Authorized PII users: Full access (after compliance training + approval)
lf.grant_permissions(
    Principal={'DataLakePrincipalIdentifier': 'arn:aws:iam::123456789:role/PIIAuthorizedUser'},
    Resource={
        'LFTagPolicy': {
            'ResourceType': 'TABLE',
            'Expression': [
                {'TagKey': 'PII', 'TagValues': ['true', 'false']}  # ALL tables
            ]
        }
    },
    Permissions=['SELECT', 'DESCRIBE']
)

# RESULT:
# ├── 50,000 tables tagged automatically (one-time script)
# ├── 2 grants cover ALL current + future tables
# ├── New PII table created + tagged → restricted automatically
# ├── Audit: CloudTrail logs every LF permission check
# └── Compliance: Can prove PII access is restricted (tag-based report)
```

---

### Q6: How does Lake Formation work with EMR? What's different about EMR vs Athena integration?

**Answer:**

```
ATHENA + LAKE FORMATION (simple — built-in):
├── Athena natively calls Lake Formation for every query
├── No extra configuration needed
├── Automatic credential vending (LF gives temp S3 creds)
└── Just works™ when LF permissions are set

EMR + LAKE FORMATION (requires explicit configuration):
├── EMR traditionally bypasses LF (uses IAM role for S3 directly!)
├── Must be explicitly configured to use LF credential vending
├── Requires: EMR 6.9+ with specific configurations
├── Only works with: Spark + Hive (Glue Catalog metastore)
└── Not all EMR workloads support it (Presto/Trino has limitations)

WHY THE DIFFERENCE?
├── Athena is AWS-native → built specifically for LF integration
├── EMR is open-source (Spark/Hive) → needs adapter layer
└── EMR's IAM role traditionally gives FULL S3 access → LF is bypassed
```

**Enabling EMR with Lake Formation:**
```hcl
resource "aws_emr_cluster" "lf_enabled" {
  configurations_json = jsonencode([
    {
      "Classification" = "emrfs-site"
      "Properties" = {
        # CRITICAL: Enable Lake Formation authorization mode
        "fs.s3.authorization.enabled" = "true"
        "fs.s3.authorization.mode" = "lake-formation"
      }
    },
    {
      "Classification" = "spark-hive-site"
      "Properties" = {
        "hive.metastore.client.factory.class" = 
          "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory"
      }
    }
  ])
  
  # EMR EC2 role must have MINIMAL S3 access (LF provides the rest)
  ec2_attributes {
    instance_profile = aws_iam_instance_profile.emr_minimal.arn
  }
}

# EMR EC2 role (minimal — no direct S3 data access!):
resource "aws_iam_role_policy" "emr_minimal" {
  policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Action = [
        "lakeformation:GetDataAccess",
        "lakeformation:GetTemporaryGlueTableCredentials",
        "glue:GetTable", "glue:GetPartitions", "glue:GetDatabase"
      ]
      Resource = "*"
    }]
  })
}
```

**What changes in EMR behavior:**
```python
# WITHOUT Lake Formation:
spark.sql("SELECT * FROM db.customers")
# → EMR uses IAM role → reads ALL S3 data (ALL columns visible!)

# WITH Lake Formation:
spark.sql("SELECT * FROM db.customers")
# → EMR asks LF: "Can this role access db.customers?"
# → LF checks permissions → "Yes, but only columns: name, email, region"
# → LF gives TEMPORARY S3 credentials scoped to those column files
# → EMR reads ONLY those columns from S3
# → User sees restricted view (same as Athena!)
```

---

### Q7: What's the difference between Lake Formation and S3 Access Points for data access control?

**Answer:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Feature           │ Lake Formation       │ S3 Access Points        │
├────────────────────┼──────────────────────┼─────────────────────────┤
│ Level              │ Logical (tables/cols)│ Physical (S3 paths)     │
│ Granularity        │ Column/Row/Cell      │ Prefix/Bucket           │
│ Knows about schema │ YES (Glue Catalog)   │ NO (just paths)         │
│ Column-level       │ YES                  │ NO (would need per-col  │
│                    │                      │  file paths — impractical)│
│ Row-level          │ YES (cell filters)   │ NO                      │
│ Cross-account      │ Simple GRANT         │ Access Point policy     │
│ Tag-based          │ YES (LF-Tags)        │ NO                      │
│ Works with         │ Athena, EMR, Glue,   │ Any S3 client           │
│                    │ Redshift Spectrum     │                         │
│ Cost               │ Free                 │ Free                    │
│ Use case           │ Data lake governance │ Application-level S3    │
│                    │                      │ access (API endpoints)  │
└────────────────────┴──────────────────────┴─────────────────────────┘

WHEN TO USE EACH:
├── Lake Formation: Analytics data governance (tables, columns, analysts)
├── S3 Access Points: Application access (services writing/reading raw files)
└── Together: LF for analytics + S3 AP for application data upload
```

---

### Q8: You need to audit who accessed what data in the last 90 days for a compliance report. How does Lake Formation help?

**Answer:**

```
LAKE FORMATION AUDIT TRAIL:

Every permission check is logged to CloudTrail:
├── Who: Principal (IAM role/user) that made the request
├── What: Table, database, columns accessed
├── When: Timestamp of access
├── How: Service used (Athena, EMR, Glue)
├── Result: Allowed or Denied (and why)
└── Where: Account, region

CloudTrail Events for LF:
├── GetTemporaryGlueTableCredentials → "User X accessed table Y"
├── GrantPermissions → "Admin granted access to Z"
├── RevokePermissions → "Admin revoked access from Z"
├── BatchGetPartitions → "Query scanned partitions of table Y"
└── GetDataAccess → "Service requested data access credentials"
```

```python
# Generate compliance audit report from CloudTrail
import boto3
from datetime import datetime, timedelta

def generate_access_report(days=90):
    """Generate data access audit report from CloudTrail."""
    ct = boto3.client('cloudtrail')
    
    access_events = []
    
    # Look up Lake Formation data access events
    paginator = ct.get_paginator('lookup_events')
    for page in paginator.paginate(
        LookupAttributes=[
            {'AttributeKey': 'EventName', 'AttributeValue': 'GetTemporaryGlueTableCredentials'}
        ],
        StartTime=datetime.utcnow() - timedelta(days=days),
        EndTime=datetime.utcnow()
    ):
        for event in page['Events']:
            detail = json.loads(event['CloudTrailEvent'])
            access_events.append({
                'timestamp': event['EventTime'],
                'user': detail['userIdentity']['arn'],
                'database': detail['requestParameters'].get('databaseName'),
                'table': detail['requestParameters'].get('tableName'),
                'columns': detail['requestParameters'].get('supportedPermissions', []),
                'source_ip': detail.get('sourceIPAddress'),
                'service': detail.get('userAgent', '').split('/')[0]
            })
    
    return access_events

# Generate report
report = generate_access_report(90)
print(f"Total data access events in last 90 days: {len(report)}")
print(f"Unique users: {len(set(e['user'] for e in report))}")
print(f"Tables accessed: {len(set(e['table'] for e in report))}")

# For compliance: Export to S3 as CSV
# Show: Who accessed PII tables? How often? From where?
```

```sql
-- If using Athena to query CloudTrail logs:
SELECT 
    useridentity.arn as user_principal,
    json_extract_scalar(requestparameters, '$.databaseName') as database_name,
    json_extract_scalar(requestparameters, '$.tableName') as table_name,
    COUNT(*) as access_count,
    MIN(eventtime) as first_access,
    MAX(eventtime) as last_access
FROM cloudtrail_logs
WHERE eventname = 'GetTemporaryGlueTableCredentials'
  AND eventtime >= date_add('day', -90, current_date)
GROUP BY 1, 2, 3
ORDER BY access_count DESC;
```

---

### Q9: How do you migrate from IAM-based S3 access control to Lake Formation without breaking existing access?

**Answer:**

```
THE CHALLENGE:
├── Current state: 500 users access S3 via IAM policies
├── Target state: Lake Formation governs all data access
├── Constraint: ZERO downtime during migration (users can't lose access!)
└── Risk: Wrong migration order → users locked out → production impact!

MIGRATION STRATEGY: Additive First, Then Restrictive

Phase 1: ADD Lake Formation (keep IAM working)
├── Enable Lake Formation
├── Set IAM_ALLOWED_PRINCIPALS as default (BOTH IAM and LF work)
├── Register S3 data locations with LF
├── Create LF permissions that MIRROR existing IAM access
└── Test: Users can still access data (IAM still works)
    Duration: 1-2 weeks

Phase 2: VERIFY Lake Formation permissions work
├── Test with a pilot group (switch them to LF-only)
├── Create test database without IAM_ALLOWED_PRINCIPALS
├── Grant LF permissions to pilot users
├── Verify they can access all needed data via LF
└── Fix any gaps (missing grants)
    Duration: 2-4 weeks

Phase 3: REMOVE IAM_ALLOWED_PRINCIPALS (table by table)
├── Start with low-risk tables (dev/test databases)
├── Remove IAM_ALLOWED_PRINCIPALS from each database
│   → Now ONLY LF permissions control access
├── Monitor: Any access failures? (CloudTrail alerts)
├── If failure: Re-add IAM_ALLOWED_PRINCIPALS (instant rollback)
└── Gradually expand to all databases
    Duration: 4-8 weeks

Phase 4: CLEANUP
├── Remove old IAM policies that granted S3 data access
├── Verify: All access is via Lake Formation only
├── Document new permission model
└── Automate: New tables auto-governed by LF-Tags
    Duration: 1-2 weeks
```

```python
# Phase 1: Mirror IAM permissions in Lake Formation
def mirror_iam_to_lf():
    """Create LF permissions that match existing IAM access."""
    
    # For each IAM role that has S3 data access:
    roles_with_access = analyze_iam_policies_for_s3()
    
    for role in roles_with_access:
        # Map S3 paths to Glue Catalog tables
        tables = map_s3_paths_to_tables(role['s3_paths'])
        
        for table in tables:
            lf.grant_permissions(
                Principal={'DataLakePrincipalIdentifier': role['arn']},
                Resource={
                    'Table': {
                        'DatabaseName': table['database'],
                        'Name': table['table']
                    }
                },
                Permissions=['SELECT', 'DESCRIBE']
            )
            print(f"Granted: {role['name']} → {table['database']}.{table['table']}")

# Phase 3: Remove IAM_ALLOWED_PRINCIPALS (per database)
def enable_lf_only(database_name):
    """Remove IAM_ALLOWED_PRINCIPALS to enforce LF-only access."""
    
    # Get current database permissions
    response = lf.get_database(Name=database_name)
    
    # Remove the "IAM_ALLOWED_PRINCIPALS" grant
    # This makes the database LF-governed only
    lf.revoke_permissions(
        Principal={'DataLakePrincipalIdentifier': 'IAM_ALLOWED_PRINCIPALS'},
        Resource={
            'Database': {'Name': database_name}
        },
        Permissions=['ALL']
    )
    
    # Also revoke table-level IAM_ALLOWED_PRINCIPALS
    tables = glue.get_tables(DatabaseName=database_name)['TableList']
    for table in tables:
        lf.revoke_permissions(
            Principal={'DataLakePrincipalIdentifier': 'IAM_ALLOWED_PRINCIPALS'},
            Resource={
                'Table': {'DatabaseName': database_name, 'Name': table['Name']}
            },
            Permissions=['ALL']
        )
    
    print(f"✅ Database '{database_name}' is now LF-governed only")
```

---

### Q10: Design a complete data governance architecture using Lake Formation for a company with 50 teams, 100 accounts, and 5 data domains.

**Answer:**

```
ARCHITECTURE:

┌─────────────────────────────────────────────────────────────────────┐
│  GOVERNANCE ARCHITECTURE                                             │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  CENTRAL GOVERNANCE ACCOUNT (Hub)                              │  │
│  │  ├── Lake Formation Admin (3 admins)                          │  │
│  │  ├── LF-Tag taxonomy (centrally managed)                      │  │
│  │  ├── Cross-account sharing policies                           │  │
│  │  ├── Audit dashboard (who accessed what)                      │  │
│  │  └── Compliance reporting automation                          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                           │                                           │
│              ┌────────────┼────────────────────────────┐             │
│              ▼            ▼            ▼               ▼              │
│  ┌─────────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────┐  │
│  │ DATA DOMAIN 1   │ │ DOMAIN 2    │ │ DOMAIN 3    │ │ ...     │  │
│  │ (Marketing)     │ │ (Finance)   │ │ (Engineering│ │         │  │
│  │                 │ │             │ │             │ │         │  │
│  │ Producer Acct   │ │ Producer    │ │ Producer    │ │         │  │
│  │ ├── S3 data     │ │ ├── S3 data │ │ ├── S3 data │ │         │  │
│  │ ├── Glue Catalog│ │ ├── Catalog │ │ ├── Catalog │ │         │  │
│  │ ├── LF grants   │ │ ├── LF      │ │ ├── LF      │ │         │  │
│  │ └── Tags: Domain│ │ └── Tags    │ │ └── Tags    │ │         │  │
│  │    =marketing   │ │  =finance   │ │  =engineering│ │         │  │
│  └─────────────────┘ └─────────────┘ └─────────────┘ └─────────┘  │
│              │                                                        │
│              ▼ (shared via LF cross-account)                         │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  CONSUMER ACCOUNTS (50 teams)                                  │  │
│  │  ├── Team A: Sees marketing + public finance data             │  │
│  │  ├── Team B: Sees all finance data                            │  │
│  │  ├── Team C: Sees engineering + marketing data                │  │
│  │  └── ... (access determined by LF-Tags)                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**Implementation plan:**

```yaml
governance_framework:
  tag_taxonomy:
    Domain:
      values: [marketing, finance, engineering, operations, hr]
      assigned_by: Data Domain Owners
      
    Sensitivity:
      values: [public, internal, confidential, restricted]
      assigned_by: Data Classification team (automated + manual)
      
    PII:
      values: [true, false]
      assigned_by: Automated (Macie scan + naming patterns)
      
    DataProduct:
      values: [customer-360, revenue-report, product-catalog, ...]
      assigned_by: Data Product Owners
      
    Environment:
      values: [production, staging, development]
      assigned_by: Platform team (automated via pipeline)

  access_policies:
    # Self-service (no approval needed):
    tier_1_public:
      expression: "Sensitivity=public"
      who: "All authenticated employees"
      approval: None
      
    # Team-level (auto-approved for domain team):
    tier_2_internal:
      expression: "Sensitivity=internal AND Domain={team_domain}"
      who: "Members of the domain team"
      approval: Auto (team membership)
      
    # Restricted (manual approval required):
    tier_3_confidential:
      expression: "Sensitivity=confidential"
      who: "Approved individuals only"
      approval: Data Owner + Security team
      
    # Highly restricted (executive approval):
    tier_4_restricted:
      expression: "Sensitivity=restricted OR PII=true"
      who: "Named individuals with business justification"
      approval: Data Owner + CISO + time-limited (90 days max)

  automation:
    - New table created → Auto-tagged by ETL pipeline (Domain, Sensitivity)
    - Macie scan weekly → Auto-tag PII columns
    - Tag changes → Alert governance team
    - Access request → Jira workflow → Lambda grants LF permission
    - Quarterly review → Report on who has access to what
    
  monitoring:
    - CloudTrail → All LF events → Security Lake (centralized)
    - Alert: Unusual data access patterns (volume, time, source)
    - Report: Weekly access summary per team
    - Audit: Quarterly PII access review (who, why, still needed?)
```

---

## 🆚 Lake Formation vs Competitors

| Feature | Lake Formation | Unity Catalog (Databricks) | Purview (Azure) | Data Catalog (GCP) | Apache Ranger |
|---------|---------------|---------------------------|-----------------|--------------------|----|
| Column-level | ✅ | ✅ | ✅ | Limited | ✅ |
| Row-level | ✅ (Cell filters) | ✅ (Row filters) | ✅ | ❌ | ✅ |
| Tag-based | ✅ (LF-Tags) | ✅ (Unity tags) | ✅ (Classifications) | ✅ (Tags) | ✅ |
| Cross-account | ✅ (Native) | ✅ (Delta Sharing) | ✅ (Purview policies) | ❌ | ❌ |
| Cost | FREE | Included in DBR | Included | Included | Free (OSS) |
| Supported engines | Athena, EMR, Glue, Redshift | Databricks only | Synapse, Fabric | BigQuery | Hadoop/Spark |
| Schema discovery | Glue Crawlers | Unity auto-lineage | Auto-scan | Data Catalog | Manual |
| Lineage | Limited | ✅ (Full) | ✅ (Full) | ✅ | ❌ |
| Best for | AWS data lakes | Databricks customers | Azure | GCP | On-prem Hadoop |

---

## 🏆 Production Best Practices

```yaml
setup:
  - Designate 2-3 Lake Formation admins (not IAM admins!)
  - Register ALL data lake S3 paths with LF
  - Plan IAM → LF migration carefully (don't break access!)
  - Use LF-Tags from day 1 (don't start with named grants)
  - Define tag taxonomy before granting access

access_control:
  - ALWAYS use LF-Tags for scalable governance (not per-table grants)
  - Column-level restrictions for PII (default: exclude PII columns)
  - Row-level filters for multi-tenant data
  - Cross-account sharing via LF (not S3 bucket policies!)
  - Principle of least privilege (start with nothing, add as needed)

operations:
  - Automate tag assignment in ETL pipelines
  - Run Macie scans weekly (auto-detect new PII)
  - Monthly permission review (remove unused access)
  - Quarterly compliance audit (CloudTrail report)
  - Alert on: new LF admin, cross-account grants, PII access spikes

migration:
  - Phase 1: Enable LF alongside IAM (both work)
  - Phase 2: Mirror IAM permissions in LF
  - Phase 3: Remove IAM_ALLOWED_PRINCIPALS (one DB at a time)
  - Phase 4: Remove old IAM data-access policies
  - ALWAYS: Keep rollback plan ready (re-add IAM if LF breaks)

common_mistakes:
  - ❌ Forgetting to register S3 locations (LF can't govern unregistered paths)
  - ❌ Leaving IAM_ALLOWED_PRINCIPALS enabled (LF permissions bypass!)
  - ❌ Using per-table grants at scale (doesn't scale — use tags!)
  - ❌ Not testing EMR + LF integration (requires explicit config)
  - ❌ Granting PermissionsWithGrantOption broadly (delegate governance risks)
  - ❌ Not automating tag assignment (manual tagging falls behind)
```
