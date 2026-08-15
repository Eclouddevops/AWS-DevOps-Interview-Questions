# Physical Datacenter to AWS Migration — Deep-Dive Interview Q&A

## Table of Contents
1. [Migration Strategy & Planning](#1-migration-strategy--planning)
2. [AWS Migration Services](#2-aws-migration-services)
3. [Compute Migration (EC2)](#3-compute-migration-ec2)
4. [Database Migration (RDS/DMS)](#4-database-migration-rdsdms)
5. [VPC & Network Migration](#5-vpc--network-migration)
6. [Storage Migration](#6-storage-migration)
7. [Application Migration Patterns](#7-application-migration-patterns)
8. [Security & Compliance During Migration](#8-security--compliance-during-migration)
9. [Cutover & Go-Live Strategy](#9-cutover--go-live-strategy)
10. [Tricky Scenario-Based Questions](#10-tricky-scenario-based-questions)

---

## 1. Migration Strategy & Planning


**Q1: Explain the AWS 7R Migration Strategies. When would you use each?**

**A:**

| Strategy | Description | Effort | Downtime | Use Case |
|----------|-------------|--------|----------|----------|
| **Rehost** (Lift & Shift) | Move as-is to EC2 | Low | Minimal | Legacy apps, quick migration, no time for optimization |
| **Replatform** (Lift, Tinker & Shift) | Minor optimizations during move | Medium | Minimal | Move to RDS instead of self-managed DB, use ELB |
| **Repurchase** (Drop & Shop) | Replace with SaaS | Low | Varies | Move email to SES/O365, CRM to Salesforce |
| **Refactor** (Re-architect) | Redesign for cloud-native | High | Planned | Microservices, serverless, containers |
| **Retire** | Decommission | None | None | Unused/duplicate systems (20-30% of apps typically) |
| **Retain** | Keep on-premises | None | None | Compliance-bound, recently upgraded, complex dependencies |
| **Relocate** (VMware Cloud on AWS) | Move VMware VMs to AWS | Low | Minimal | Large VMware estates, keep same tools |

**Decision framework:**
```
Is the application still needed?
├── NO → RETIRE (saves license + hosting costs immediately)
├── YES → Does it have a SaaS equivalent?
│   ├── YES (and acceptable) → REPURCHASE
│   └── NO → Is business value high enough for redesign?
│       ├── YES + budget + time → REFACTOR (best long-term)
│       ├── YES + limited time → REPLATFORM (quick wins)
│       └── NO / Time pressure → REHOST (fastest, revisit later)
```

**Tricky**: Most enterprises use a MIX of strategies. The "Big Bang" approach (migrate everything at once) almost always fails. Best practice: Wave-based migration — group apps into waves of 10-20, migrate wave by wave over months. Start with low-risk, low-complexity apps to build confidence.

---

**Q2: Explain the AWS Migration phases. What happens in each phase?**

**A:**

```
Phase 1: ASSESS (4-8 weeks)
├── Discovery: Inventory all servers, apps, dependencies
│   ├── AWS Application Discovery Service (agent-based or agentless)
│   ├── AWS Migration Hub
│   ├── Third-party: CloudEndure, RVTools (VMware), ADDM
│   └── Manual: Interviews, runbooks, architecture diagrams
├── Dependency mapping: Which apps talk to which?
├── Business case: TCO comparison (on-prem vs AWS)
├── Risk assessment: Compliance, licensing, skills gaps
└── Output: Migration portfolio, prioritized wave plan

Phase 2: MOBILIZE (4-12 weeks)
├── Landing Zone setup: AWS Control Tower, Organizations, accounts
├── Network connectivity: Direct Connect, VPN, Transit Gateway
├── Security baseline: IAM, GuardDuty, Config Rules, SCPs
├── Operating model: CI/CD, monitoring, runbooks for cloud
├── Proof of concept: Migrate 2-3 apps to validate approach
├── Team training: AWS certifications, hands-on labs
└── Output: Fully operational AWS foundation

Phase 3: MIGRATE & MODERNIZE (ongoing, wave-based)
├── Wave 1: Low-risk apps (dev/test environments)
├── Wave 2-N: Production apps (increasing complexity)
├── Each wave: Plan → Build → Test → Cutover → Optimize
├── Continuous optimization: Right-sizing, Reserved Instances
└── Output: Fully migrated workloads, datacenter decommissioned
```

**Tricky**: The ASSESS phase is where most migrations fail. If you don't accurately map dependencies, you'll discover them during cutover (worst time). Example: App A migrated to AWS, but it depends on App B still on-premises via a hardcoded IP that doesn't work cross-network. Always use agent-based discovery for accurate dependency mapping.

---

**Q3: How do you build a business case for datacenter-to-AWS migration? What costs are often missed?**

**A:**

**Cost comparison framework:**

```
ON-PREMISES COSTS (often underestimated):
├── Hardware: Servers, storage, network equipment (3-5 year refresh)
├── Software: OS licenses, DB licenses (Oracle/SQL Server), virtualization
├── Facilities: Power, cooling, rack space, physical security
├── People: Sysadmins, network engineers, security team, DBAs
├── Networking: WAN links, ISP costs, CDN
├── DR: Secondary site hardware + replication licenses
├── Over-provisioning: Buying for peak (servers idle 70-80% of time)
└── Opportunity cost: 6-12 month procurement cycles

AWS COSTS:
├── Compute: EC2 (on-demand → Reserved → Savings Plans)
├── Storage: EBS, S3, EFS
├── Database: RDS, DynamoDB
├── Network: Data transfer, NAT Gateway, Direct Connect
├── Managed services: ALB, CloudFront, WAF
├── Operations: CloudWatch, Systems Manager
└── Migration: DMS, MGN, professional services (temporary)
```

**Commonly MISSED costs in AWS:**
1. **Data transfer OUT** ($0.09/GB — can be massive for media/CDN workloads)
2. **NAT Gateway** ($0.045/hr + $0.045/GB processed — adds up fast!)
3. **Cross-AZ traffic** ($0.01/GB each direction — multiplies with microservices)
4. **EBS snapshots** (stored in S3, often forgotten)
5. **CloudWatch Logs** storage (no retention = infinite growth)
6. **Idle resources** (dev environments running 24/7)

**Commonly MISSED savings:**
1. **Retire 20-30% of apps** (never needed, duplicates, unused)
2. **Right-sizing** (most on-prem servers are 2-4x oversized)
3. **Elasticity** (scale down nights/weekends = 40-60% savings)
4. **Reserved Instances** (up to 72% savings for committed workloads)
5. **Managed services** (eliminate DBA, storage admin roles)
6. **DR consolidation** (AWS multi-AZ replaces secondary datacenter)

**Tricky**: Never compare on-premises HARDWARE cost to AWS ON-DEMAND pricing. On-premises hardware amortized over 5 years IS cheap per hour. Compare: Total cost of ownership (people, power, space, licenses, opportunity cost) vs AWS total (with Reserved Instances + right-sizing). Apples-to-apples TCO usually favors AWS by 30-60%.

---


## 2. AWS Migration Services

**Q4: Explain ALL AWS migration services. When do you use each?**

**A:**

| Service | Migrates | Method | Use Case |
|---------|----------|--------|----------|
| **AWS MGN** (Application Migration Service) | Servers (OS + apps) | Continuous block-level replication | Lift & shift any server to EC2 |
| **AWS DMS** (Database Migration Service) | Databases | Continuous data replication | DB migration with minimal downtime |
| **AWS SCT** (Schema Conversion Tool) | DB schemas | Schema analysis + conversion | Heterogeneous DB migration (Oracle → PostgreSQL) |
| **AWS DataSync** | File storage | Network transfer (agent-based) | NFS/SMB → S3/EFS/FSx |
| **AWS Transfer Family** | Files via protocols | SFTP/FTPS/FTP → S3 | Replace file transfer servers |
| **AWS Snow Family** | Large data volumes | Physical device | Petabytes of data, limited bandwidth |
| **AWS Migration Hub** | Everything | Dashboard + tracking | Central visibility across all migrations |
| **AWS Application Discovery** | Server inventory | Agent or agentless | Discovery and dependency mapping |
| **CloudEndure Migration** | Servers | Block replication (now MGN) | Legacy — replaced by MGN |
| **VM Import/Export** | VMware/Hyper-V VMs | OVA/VMDK import | One-time VM conversion to AMI |

**Decision tree:**
```
What are you migrating?

SERVERS (OS + applications):
├── Live migration with minimal downtime → AWS MGN
├── VMware environment → MGN or VMware Cloud on AWS
└── One-time VM conversion → VM Import/Export

DATABASES:
├── Same engine (MySQL → RDS MySQL) → DMS (homogeneous)
├── Different engine (Oracle → PostgreSQL) → SCT + DMS (heterogeneous)
├── Very large DB (>10TB) → DMS + Snow + DMS (hybrid)
└── NoSQL (MongoDB → DynamoDB) → DMS with transformation rules

FILE STORAGE:
├── NFS/SMB shares → DataSync → EFS or FSx
├── Large volume (>100TB, limited bandwidth) → Snowball → S3
├── Ongoing sync needed → DataSync (scheduled)
└── SFTP/FTP server replacement → Transfer Family

MASSIVE DATA (Petabytes):
├── < 10TB, good bandwidth → Direct transfer (DataSync, S3 CLI)
├── 10-80TB → Snowball Edge (single device)
├── 80TB-1PB → Multiple Snowball Edge devices
└── > 1PB → Snowmobile (literal truck, 100PB capacity)
```

**Tricky**: AWS MGN (Application Migration Service) has REPLACED CloudEndure Migration. If an interviewer asks about CloudEndure, mention it's legacy and explain MGN instead. MGN provides the same block-level replication but is natively integrated into AWS console.

---

**Q5: Explain AWS Application Migration Service (MGN) in detail. How does it work end-to-end?**

**A:**

```
On-Premises Server                              AWS
┌──────────────────┐                  ┌─────────────────────┐
│ Source Server    │                  │  Staging Area       │
│ (Linux/Windows)  │                  │  (lightweight EC2)  │
│                  │   Continuous     │                     │
│ ┌──────────────┐ │   Block-Level   │  ┌───────────────┐  │
│ │ MGN Agent    │─┼──Replication───→│  │ EBS Volumes   │  │
│ │ (installed)  │ │   (encrypted)   │  │ (replica of   │  │
│ └──────────────┘ │                  │  │  source disks)│  │
└──────────────────┘                  │  └───────────────┘  │
                                      └─────────────────────┘
                                               │
                                         Launch Cutover
                                               │
                                               ▼
                                      ┌─────────────────────┐
                                      │  Target EC2 Instance │
                                      │  (production-ready)  │
                                      │  - Right-sized       │
                                      │  - In target VPC     │
                                      │  - With target SG    │
                                      └─────────────────────┘
```

**Step-by-step process:**

1. **Install Agent** on source server (supports Linux & Windows):
```bash
# Linux
wget -O installer https://aws-application-migration-service-<region>.s3.amazonaws.com/latest/linux/aws-replication-installer-init.py
sudo python3 aws-replication-installer-init.py
```

2. **Initial Sync** (full disk replication):
- Copies all blocks from source disks to EBS in staging area
- Duration: Depends on disk size and bandwidth (hours to days)

3. **Continuous Replication** (ongoing):
- Only changed blocks replicated (like incremental backup)
- Sub-second RPO (Recovery Point Objective)
- Source server continues running normally

4. **Test** (non-disruptive):
- Launch test instance from replicated EBS
- Validate application works in AWS
- Terminate test instance (replication continues)

5. **Cutover** (actual migration):
- Final sync of last changes
- Launch target instance
- Update DNS/load balancers to point to new instance
- Decommission source server

**Key configuration (Launch Template):**
```
Right-sizing: Choose instance type based on utilization data
Network: Target VPC, subnet, security groups
Storage: EBS volume type (gp3 recommended)
Tags: Environment, migration wave, cost center
Post-launch actions: Install CloudWatch agent, join domain, register with LB
```

**Tricky**: MGN replicates at the BLOCK level (not file level). This means it works with ANY operating system and ANY application — it doesn't need to understand the application. But it also means it copies EVERYTHING including temporary files, caches, and unused space. Right-size the target EBS volumes, don't just replicate the same size.

---

**Q6: What is the difference between MGN Test and MGN Cutover? Why is testing critical?**

**A:**

| Aspect | Test Launch | Cutover Launch |
|--------|-------------|----------------|
| Purpose | Validate before migration | Actual production migration |
| Source server | Keeps running (no impact) | Eventually decommissioned |
| Replication | Continues after test | Stops after cutover |
| DNS/Traffic | Not switched | Switched to new instance |
| Rollback | Just terminate test instance | More complex (switch back) |
| Times allowed | Unlimited | Once (per source) |

**Why testing is CRITICAL:**

1. **Driver compatibility**: Windows may need AWS PV drivers (paravirtual)
2. **License activation**: Windows activation may fail (KMS server unreachable)
3. **Network dependencies**: Hardcoded IPs, DNS names that don't resolve in AWS
4. **Application startup**: Services may fail if dependencies aren't available
5. **Performance validation**: Instance type may not match source server specs
6. **Security groups**: Application ports may be blocked by default deny

**Test checklist:**
```
□ Instance boots successfully
□ All services start automatically
□ Application responds on expected ports
□ Database connectivity works (if applicable)
□ DNS resolution works for all dependencies
□ Monitoring agent reports metrics
□ Backup jobs configured
□ SSL certificates valid
□ License servers reachable (if applicable)
□ Performance meets baseline (load test)
```

**Tricky**: During test, the source server is still running and still being replicated. But the test instance is launched from a POINT-IN-TIME snapshot. If you test today and cutover next week, changes made during that week aren't in the test. Always do a final test 24-48 hours before cutover.

---


## 3. Compute Migration (EC2)

**Q7: You have 500 physical servers to migrate. How do you right-size them for EC2? Most are running at 10-20% CPU utilization.**

**A:**

**Discovery data needed (from AWS Application Discovery or third-party):**
```
For each server, collect 2-4 weeks of:
├── Peak CPU utilization (%)
├── Average CPU utilization (%)
├── Peak memory utilization (%)
├── Average memory utilization (%)
├── Disk IOPS (read + write)
├── Disk throughput (MB/s)
├── Network throughput (Mbps)
└── Storage used (GB)
```

**Right-sizing methodology:**
```
Step 1: Size for PEAK, not average
├── If peak CPU = 40%, don't need current 32-core server
├── Size for peak + 20% headroom = target 50% at peak
└── A 4-vCPU instance may suffice for a "32-core" server!

Step 2: Match CPU architecture
├── Physical: Intel Xeon E5-2680 v4 (2.4 GHz, 14 cores)
├── AWS: c5.4xlarge (16 vCPU, 3.0 GHz) — better per-core performance!
└── Often need FEWER vCPUs than physical cores (AWS CPUs are faster)

Step 3: Match memory requirements
├── Physical: 128 GB RAM, usage peaks at 64 GB
├── AWS: r5.2xlarge (64 GB) — matches actual usage
└── Don't replicate oversized physical memory allocation

Step 4: Match storage performance
├── Physical: Local SAS drives (200 IOPS, 100 MB/s)
├── AWS: gp3 (3000 IOPS, 125 MB/s baseline) — likely BETTER
└── Only use io2 if actual IOPS need > 3000
```

**Common right-sizing results:**
```
Physical Servers → Right-sized EC2:
├── 30% can drop 2+ instance sizes (massively overprovisioned)
├── 50% can drop 1 size
├── 15% stay similar
└── 5% may need LARGER (were actually constrained on-prem)

Typical savings: 40-60% cost reduction vs 1:1 instance mapping
```

**Tools for right-sizing:**
- AWS Migration Hub + TSO Logic (free with migration)
- AWS Compute Optimizer (post-migration, ongoing)
- Third-party: Densify, CloudHealth, Turbonomic

**Tricky**: Don't right-size too aggressively on day 1. Start with slightly larger instances (1 size above recommendation), then use AWS Compute Optimizer after 14 days of CloudWatch data to fine-tune. It's easier to scale DOWN than to deal with performance issues during migration.

---

**Q8: How do you handle Windows licensing during datacenter-to-AWS migration?**

**A:**

**Windows licensing options on AWS:**

| Option | Description | When to Use |
|--------|-------------|-------------|
| License Included (LI) | AWS provides Windows license in hourly cost | Default, simplest |
| BYOL (Bring Your Own License) | Use existing SA/EA licenses | Enterprise Agreement with SA |
| Dedicated Hosts | Physical server for license compliance | Core-based licensing (SQL Server) |
| Dedicated Instances | Single-tenant hardware | Some compliance requirements |

**Critical licensing scenarios:**

1. **Windows Server on EC2:**
```
License Included: ~$0.046/hr premium over Linux (included in EC2 price)
BYOL: Must use Dedicated Hosts OR shared tenancy with specific license types
Requirement for BYOL: Software Assurance (SA) active on licenses
```

2. **SQL Server on EC2:**
```
License Included: Standard Edition included in some RDS/EC2 options
BYOL on Dedicated Hosts:
├── SQL Server Standard: Per-core licensing
├── SQL Server Enterprise: Per-core licensing (EXPENSIVE)
├── Dedicated Host gives you specific physical cores for licensing
└── One host can run multiple SQL Server instances (maximize license usage)
```

3. **Oracle on EC2:**
```
Oracle licensing on AWS:
├── Each vCPU = 0.5 Oracle processor license (default hyperthreading)
├── Dedicated Hosts with hyperthreading disabled: 1 vCPU = 1 core
├── Can reduce Oracle license requirements by 50% on Dedicated Hosts!
└── OR migrate to Aurora PostgreSQL (eliminate Oracle license entirely)
```

**Migration licensing checklist:**
```
□ Audit current licenses (what do you own? what's in SA?)
□ Check license mobility rights (can you move to cloud?)
□ Verify Dedicated Host/Instance requirements
□ Calculate: BYOL savings vs License Included simplicity
□ Consider: Migration = opportunity to shed expensive licenses
  (Oracle → PostgreSQL/Aurora, SQL Server → Aurora MySQL, Exchange → SES)
```

**Tricky**: Microsoft changed licensing rules in 2019/2022 — running Windows Server or SQL Server on "Listed Providers" (AWS, Azure, GCP) requires Software Assurance OR you must use License Included pricing. You CANNOT simply move old OEM/retail Windows licenses to EC2 shared tenancy without SA. Dedicated Hosts have different rules.

---

**Q9: A physical server has 4 NICs bonded together for redundancy and throughput. How do you replicate this on AWS?**

**A:**

**You DON'T replicate NIC bonding on AWS — the architecture is fundamentally different.**

```
ON-PREMISES:
├── 4 × 1 Gbps NICs bonded = 4 Gbps + redundancy
├── Bond mode: Active/passive (failover) or 802.3ad (LACP aggregation)
└── Purpose: Network redundancy + bandwidth

AWS EQUIVALENT:
├── Single ENI with instance-level bandwidth (5-100+ Gbps depending on type)
├── Redundancy: AWS handles at infrastructure level (no single NIC failure)
├── Bandwidth: Instance type determines bandwidth (not number of NICs)
└── Multiple ENIs for: Multi-homing (different subnets), not bonding
```

**AWS network bandwidth by instance type:**
```
t3.medium:    Up to 5 Gbps (burst)
m5.large:     Up to 10 Gbps
m5.4xlarge:   Up to 10 Gbps
m5.8xlarge:   10 Gbps
m5.16xlarge:  20 Gbps
m5.24xlarge:  25 Gbps
m5n.24xlarge: 100 Gbps
p4d.24xlarge: 400 Gbps (EFA)
```

**When you DO use multiple ENIs on AWS:**
- Multi-homed instances (management network + data network)
- Network appliances (firewall with inside/outside interfaces)
- Running network functions (NAT, routing between VPCs)
- License servers bound to specific MAC addresses

**For high availability (replacing NIC bonding purpose):**
```
On-prem: NIC bonding for failover
AWS: 
├── ENI moves between instances (failover via Lambda/script)
├── Elastic IP moves between instances (instant failover)
├── Or better: Use ALB/NLB (no single instance dependency)
└── Multi-AZ deployment (AZ-level redundancy, not NIC-level)
```

**Tricky**: Some customers try to install bonding drivers (ifenslave) on EC2 — this doesn't work and isn't needed. AWS ENIs already have redundancy built into the fabric. If your on-prem server needed 4 Gbps bandwidth, just choose an instance type that provides 10 Gbps (most modern instances do).

---


## 4. Database Migration (RDS/DMS)

**Q10: You have a 5TB Oracle database that must be migrated to Aurora PostgreSQL with less than 1 hour of downtime. Design the migration.**

**A:**

**This is a HETEROGENEOUS migration (different engine) — most complex type.**

```
Phase 1: Schema Conversion (weeks before migration)
├── Run AWS SCT (Schema Conversion Tool)
├── Convert: Tables, views, stored procedures, triggers, functions
├── Manual fixes: ~20-40% of code needs manual conversion
│   ├── Oracle-specific PL/SQL → PostgreSQL PL/pgSQL
│   ├── Oracle sequences → PostgreSQL sequences/IDENTITY
│   ├── Oracle synonyms → PostgreSQL schemas/search_path
│   ├── Oracle packages → PostgreSQL schemas + functions
│   └── Data types: NUMBER → NUMERIC, VARCHAR2 → VARCHAR, CLOB → TEXT
└── Test: Run application against new schema (fix SQL compatibility)

Phase 2: Data Migration (DMS - Full Load)
├── Create DMS Replication Instance (r5.4xlarge for 5TB)
├── Create Source Endpoint (Oracle, with supplemental logging enabled)
├── Create Target Endpoint (Aurora PostgreSQL)
├── Task: Full Load (initial bulk copy)
│   ├── Duration: 8-24 hours for 5TB (depends on table count + network)
│   ├── Parallel threads per table (up to 8)
│   ├── LOB handling: Limited LOB mode for speed
│   └── Disable: Foreign keys, triggers, indexes on target (re-enable after)
└── Output: Target has full copy of source data

Phase 3: Continuous Replication (CDC - Change Data Capture)
├── DMS enables CDC from Oracle redo logs
├── Captures all INSERT/UPDATE/DELETE after full load
├── Applies changes to Aurora PostgreSQL continuously
├── Replication lag: < 5 minutes (monitor DMS metrics)
├── Duration: Run for days/weeks while validating
└── Output: Source and target in near-real-time sync

Phase 4: Validation
├── AWS DMS Data Validation (row count + checksum comparison)
├── Application testing against Aurora (read-only queries)
├── Performance testing (query plans different on PostgreSQL!)
├── Fix any conversion issues found during testing
└── Output: Confidence that target is ready

Phase 5: Cutover (< 1 hour downtime)
├── T-60min: Stop application writes to source Oracle
├── T-55min: Wait for DMS CDC lag to reach 0 (all changes applied)
├── T-50min: Run final validation (row counts match exactly)
├── T-40min: Re-enable indexes, foreign keys, triggers on Aurora
├── T-20min: Update application connection strings to Aurora endpoint
├── T-10min: Start application against Aurora
├── T-0: Verify application functioning correctly
└── Rollback plan: Switch back to Oracle if critical issues (Oracle still has all data)
```

**Critical DMS settings for large migrations:**
```json
{
  "ReplicationInstanceClass": "dms.r5.4xlarge",
  "AllocatedStorage": 500,
  "MultiAZ": true,
  "TaskSettings": {
    "TargetMetadata": {
      "ParallelLoadThreads": 8,
      "ParallelLoadBufferSize": 500
    },
    "FullLoadSettings": {
      "TargetTablePrepMode": "TRUNCATE_BEFORE_LOAD",
      "MaxFullLoadSubTasks": 8,
      "CommitRate": 50000
    },
    "Logging": {
      "EnableLogging": true,
      "LogComponents": [{"Id": "SOURCE_UNLOAD", "Severity": "LOGGER_SEVERITY_DETAILED_DEBUG"}]
    }
  }
}
```

**Tricky**: Oracle → PostgreSQL migration is NEVER just a "data copy." Stored procedures, triggers, and application SQL all need modification. Budget 40-60% of migration time for schema/code conversion, not data movement. SCT converts ~60% automatically; the remaining 40% requires manual developer effort.

---

**Q11: Explain DMS CDC (Change Data Capture). How does it achieve near-zero downtime?**

**A:**

```
Without CDC (full load only):
├── Stop application → Copy data (hours) → Start on new DB
└── Downtime = Full copy duration (8-24+ hours for large DBs)

With CDC (continuous replication):
├── Start CDC → Full load runs while app continues → CDC catches changes
├── Application keeps writing to source (no downtime during copy!)
├── Only cutover window needs downtime (minutes, not hours)
└── Downtime = Final sync + switchover (< 1 hour typically)
```

**How CDC works per database engine:**

| Source DB | CDC Mechanism | Requirements |
|-----------|--------------|--------------|
| Oracle | LogMiner (redo logs) | Supplemental logging enabled, ARCHIVELOG mode |
| MySQL | Binary Log (binlog) | binlog_format = ROW, binlog_row_image = full |
| PostgreSQL | Logical replication slots | wal_level = logical, max_replication_slots > 0 |
| SQL Server | MS-CDC or CT | CDC enabled on database, tables |
| MongoDB | Change Streams | Replica set required |

**DMS CDC lag monitoring:**
```
CloudWatch Metrics:
├── CDCLatencySource: Seconds since last event read from source
├── CDCLatencyTarget: Seconds since last event applied to target
├── CDCIncomingChanges: Pending changes to apply
└── CDCThroughputRowsSource/Target: Rows processed per second

Alarm: CDCLatencyTarget > 300 seconds → target falling behind!
```

**Tricky**: CDC requires source database changes that can impact performance:
- Oracle: Supplemental logging adds 5-15% overhead on writes
- MySQL: ROW binlog format generates larger binlogs than STATEMENT
- PostgreSQL: Logical replication slots PREVENT WAL cleanup (disk can fill if DMS disconnects!)

Always enable these in a maintenance window and monitor source DB performance after enabling.

---

**Q12: You're migrating a MySQL 5.7 database to Aurora MySQL. What's the fastest method with minimal downtime?**

**A:**

**This is a HOMOGENEOUS migration (same engine) — simpler than heterogeneous.**

**Option 1: DMS (Most flexible, minimal downtime):**
```
Source: On-prem MySQL 5.7
Target: Aurora MySQL 5.7-compatible
Method: Full Load + CDC
Downtime: < 15 minutes (final sync + switchover)
```

**Option 2: Native MySQL replication (fastest for MySQL):**
```
1. Take mysqldump or xtrabackup from source
2. Restore to Aurora MySQL
3. Configure Aurora as a READ REPLICA of on-prem MySQL
   (using binlog replication: CALL mysql.rds_set_external_master)
4. Monitor replication lag until 0
5. Promote Aurora (CALL mysql.rds_stop_replication)
6. Switch application connection string
Downtime: < 5 minutes
```

**Option 3: Aurora MySQL as binlog replica (recommended):**
```bash
# On source MySQL: Create replication user
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl_user'@'%';

# On Aurora: Configure as external replica
CALL mysql.rds_set_external_master(
  'source-mysql-host.example.com',  -- source host
  3306,                              -- port
  'repl_user',                       -- user
  'password',                        -- password
  'mysql-bin.000154',                -- binlog file
  1234567,                           -- binlog position
  0                                  -- SSL (0=no, 1=yes)
);

CALL mysql.rds_start_replication;

# Monitor lag
SHOW SLAVE STATUS\G
# Seconds_Behind_Master should be 0 before cutover

# Cutover:
CALL mysql.rds_stop_replication;
CALL mysql.rds_reset_external_master;
# Switch app connection string to Aurora endpoint
```

**Option 4: Percona XtraBackup → S3 → Aurora (fastest initial load):**
```bash
# For very large databases (>1TB):
# 1. Take XtraBackup on source
xtrabackup --backup --target-dir=/backup

# 2. Upload to S3
aws s3 sync /backup s3://migration-bucket/mysql-backup/

# 3. Restore directly to Aurora from S3 (AWS native feature!)
aws rds restore-db-cluster-from-s3 \
  --db-cluster-identifier my-aurora \
  --engine aurora-mysql \
  --source-engine mysql \
  --source-engine-version "5.7" \
  --s3-bucket-name migration-bucket \
  --s3-prefix mysql-backup/ \
  --s3-ingestion-role-arn arn:aws:iam::123456:role/rds-s3-import

# 4. Then set up binlog replication for CDC (catch up changes)
```

**Tricky**: Native MySQL replication (Option 2/3) is FASTER and MORE RELIABLE than DMS for MySQL-to-MySQL migrations. DMS adds overhead of reading binlog → converting → writing. Native replication is the engine's built-in mechanism. Use DMS only when you need transformations or are changing engines.

---


## 5. VPC & Network Migration

**Q13: Design the network architecture for migrating a datacenter with 200 servers across 5 VLANs to AWS.**

**A:**

**On-Premises network:**
```
VLAN 10: Web servers (192.168.10.0/24) — 40 servers
VLAN 20: Application servers (192.168.20.0/24) — 80 servers
VLAN 30: Database servers (192.168.30.0/24) — 30 servers
VLAN 40: Management (192.168.40.0/24) — 20 servers
VLAN 50: DMZ (10.0.50.0/24) — 30 servers
```

**AWS VPC Design:**
```
VPC: 10.0.0.0/16 (65,536 IPs — room to grow)

├── Public Subnets (DMZ equivalent):
│   ├── 10.0.1.0/24 (AZ-a) — ALB, NAT Gateway, Bastion
│   ├── 10.0.2.0/24 (AZ-b) — ALB, NAT Gateway
│   └── 10.0.3.0/24 (AZ-c) — ALB, NAT Gateway
│
├── Private Subnets - Web Tier (VLAN 10 equivalent):
│   ├── 10.0.10.0/24 (AZ-a) — Web EC2 instances
│   ├── 10.0.11.0/24 (AZ-b) — Web EC2 instances
│   └── 10.0.12.0/24 (AZ-c) — Web EC2 instances
│
├── Private Subnets - App Tier (VLAN 20 equivalent):
│   ├── 10.0.20.0/24 (AZ-a) — App servers
│   ├── 10.0.21.0/24 (AZ-b) — App servers
│   └── 10.0.22.0/24 (AZ-c) — App servers
│
├── Private Subnets - Data Tier (VLAN 30 equivalent):
│   ├── 10.0.30.0/24 (AZ-a) — RDS, ElastiCache
│   ├── 10.0.31.0/24 (AZ-b) — RDS Multi-AZ standby
│   └── 10.0.32.0/24 (AZ-c) — Read replicas
│
└── Private Subnets - Management (VLAN 40 equivalent):
    ├── 10.0.40.0/24 (AZ-a) — SSM, monitoring, CI/CD
    └── 10.0.41.0/24 (AZ-b) — Backup management
```

**CIDR planning rules:**
```
1. Don't overlap with on-premises CIDR (you'll need connectivity!)
   On-prem: 192.168.0.0/16 → AWS: 10.0.0.0/16 ✓
   
2. Size for growth (at least 2x current needs)
   200 servers today → Plan for 500 IPs minimum → /16 gives 65K

3. Reserve space for future VPCs / peering
   VPC 1: 10.0.0.0/16 (production)
   VPC 2: 10.1.0.0/16 (staging)
   VPC 3: 10.2.0.0/16 (development)
   
4. Each subnet should be /24 minimum (251 usable IPs)
   AWS reserves 5 IPs per subnet: network, VPC router, DNS, future, broadcast
```

**Tricky**: On-premises VLANs provide Layer 2 isolation (same broadcast domain). AWS subnets are Layer 3 (routed, no broadcast). Security Groups replace VLAN-based ACLs. Don't try to replicate exact VLAN structure — redesign for AWS best practices (multi-AZ, tiered subnets).

---

**Q14: How do you establish connectivity between on-premises datacenter and AWS during migration? Explain all options.**

**A:**

| Option | Bandwidth | Latency | Setup Time | Cost | Encryption |
|--------|-----------|---------|-----------|------|------------|
| **Site-to-Site VPN** | 1.25 Gbps per tunnel | Variable (internet) | Hours | ~$36/month | IPsec (built-in) |
| **AWS Direct Connect** | 1/10/100 Gbps | Low, consistent | 2-12 weeks | $0.20-0.30/GB + port fee | Optional (MACsec/VPN over DX) |
| **VPN over Direct Connect** | DX bandwidth | Low + encrypted | 2-12 weeks + hours | DX cost + VPN cost | IPsec over DX |
| **AWS Transit Gateway** | Aggregates VPN/DX | Varies | Hours-weeks | $0.05/hr + data | Inherits underlying |
| **Client VPN** | Per-user | Variable | Hours | $0.05/hr + $0.10/conn | TLS |

**Architecture for migration (typically both VPN + DX):**
```
Phase 1 (Day 1): Site-to-Site VPN
├── Quick to set up (hours)
├── Provides immediate connectivity for testing
├── Encrypted over internet
├── Limitation: 1.25 Gbps, variable latency
└── Use for: Development, testing, non-critical traffic

Phase 2 (Week 4-8): Direct Connect
├── Dedicated fiber connection
├── Consistent latency (<10ms typically)
├── High bandwidth (10-100 Gbps)
├── No internet path (more secure, predictable)
└── Use for: Production traffic, large data replication

Phase 3 (Ongoing): Transit Gateway
├── Hub-and-spoke connectivity
├── Multiple VPCs share one DX/VPN
├── Centralized routing
├── Cross-region peering
└── Use for: Multi-VPC, multi-account architecture
```

**Redundancy design:**
```
Primary: Direct Connect (10 Gbps) through Provider A
├── DX Connection → Virtual Private Gateway → VPC
├── Dedicated connection to AWS Direct Connect location

Secondary: Direct Connect (10 Gbps) through Provider B (different path)
├── Different physical path / DX location
└── Protects against single provider failure

Tertiary: Site-to-Site VPN (over internet)
├── Always-on failover
├── BGP with lower preference than DX
└── Automatic failover if both DX connections fail
```

**Tricky**: Direct Connect is NOT encrypted by default! Traffic travels over dedicated fiber but is NOT encrypted in transit. For compliance (HIPAA, PCI, etc.), you must add encryption: either VPN over DX (IPsec) or MACsec (Layer 2 encryption, DX-specific). Many people assume DX = secure because it's "private" — it's private (not on internet) but not encrypted.

---

**Q15: During migration, some apps are on-premises and some are on AWS. How do you handle DNS resolution between both environments?**

**A:**

**The "hybrid DNS" challenge — apps need to resolve names in BOTH directions:**

```
On-Premises App → needs to resolve: mydb.us-east-1.rds.amazonaws.com (AWS)
AWS App → needs to resolve: fileserver.internal.company.com (on-premises)
Both → need to resolve: api.company.com (could be either location during migration)
```

**Solution: Route 53 Resolver (Inbound + Outbound Endpoints):**

```
┌─────────────────────────┐          ┌──────────────────────────┐
│    ON-PREMISES          │          │         AWS VPC           │
│                         │          │                           │
│  DNS Server             │          │  Route 53 Resolver        │
│  (Active Directory DNS) │          │  ├── Inbound Endpoint     │
│        │                │   DX/VPN │  │   (10.0.40.10)         │
│        ├─── Forward     │◄────────►│  │   On-prem forwards     │
│        │    AWS zones   │          │  │   AWS queries HERE      │
│        │    to Route53  │          │  │                         │
│        │                │          │  └── Outbound Endpoint    │
│        └─── Respond to  │◄─────────│      (10.0.40.20)         │
│             on-prem     │          │      AWS forwards          │
│             queries     │          │      on-prem queries HERE  │
└─────────────────────────┘          └──────────────────────────┘
```

**Configuration:**

1. **On-prem → AWS resolution** (Inbound Endpoint):
```bash
# Create Inbound Endpoint (receives queries from on-prem)
aws route53resolver create-resolver-endpoint \
  --direction INBOUND \
  --ip-addresses SubnetId=subnet-123,Ip=10.0.40.10 SubnetId=subnet-456,Ip=10.0.40.11

# On-premises DNS: Add conditional forwarder
# Forward *.amazonaws.com → 10.0.40.10, 10.0.40.11
# Forward *.aws.internal → 10.0.40.10, 10.0.40.11
```

2. **AWS → On-prem resolution** (Outbound Endpoint + Forwarding Rules):
```bash
# Create Outbound Endpoint (sends queries to on-prem)
aws route53resolver create-resolver-endpoint \
  --direction OUTBOUND \
  --ip-addresses SubnetId=subnet-123 SubnetId=subnet-456

# Create Forwarding Rule
aws route53resolver create-resolver-rule \
  --rule-type FORWARD \
  --domain-name "internal.company.com" \
  --target-ips "Ip=192.168.40.10,Port=53" "Ip=192.168.40.11,Port=53"
```

3. **Shared domain during migration** (Split-horizon DNS):
```
api.company.com → 
  If queried from on-prem: Returns on-prem server IP (192.168.20.5)
  If queried from AWS: Returns ALB endpoint (internal-alb-123.us-east-1.elb.amazonaws.com)
  
Implement with Route 53 Private Hosted Zone (associated with VPC)
+ On-prem DNS serves the on-prem version
```

**Tricky**: Route 53 Resolver endpoints need IPs in your VPC subnets. These IPs are the targets for on-premises conditional forwarders. If these IPs are in a subnet that loses connectivity (AZ failure), DNS resolution breaks for on-premises → AWS. Always deploy endpoints across 2+ AZs.

---


## 6. Storage Migration

**Q16: How do you migrate 50TB of NFS file shares to AWS? Compare options.**

**A:**

| Method | Speed | Best For | Limitation |
|--------|-------|----------|-----------|
| AWS DataSync | 10 Gbps (over DX) | Active NFS/SMB shares | Needs network bandwidth |
| AWS Snowball Edge | 80TB per device | Limited bandwidth, large data | 1-2 week shipping time |
| S3 Transfer Acceleration | Variable | Global uploads | Cost per GB |
| Direct `aws s3 sync` | Limited by bandwidth | Small datasets < 5TB | Single-threaded by default |

**Recommended approach for 50TB NFS → EFS/S3:**

```
Phase 1: Initial bulk transfer (DataSync)
├── Install DataSync Agent on-premises (VM)
├── Configure source: NFS server (192.168.10.5:/shares)
├── Configure destination: EFS or S3
├── Enable: Verify data integrity, preserve permissions/timestamps
├── Schedule: Run during off-hours (maximize bandwidth)
├── Duration: 50TB / 1 Gbps = ~4.6 days
└── Note: Non-disruptive (reads from source without locking)

Phase 2: Incremental sync (ongoing until cutover)
├── DataSync only transfers CHANGED files on subsequent runs
├── Schedule every 6-12 hours
├── Gets progressively faster (less delta each time)
└── Final sync before cutover: minutes (only recent changes)

Phase 3: Cutover
├── Stop writes to NFS
├── Run final DataSync job (catch last changes)
├── Verify file counts/sizes match
├── Update applications to use EFS mount / S3 endpoint
└── Decommission on-prem NFS
```

**Tricky**: DataSync preserves POSIX permissions, timestamps, and file ownership when copying to EFS. But when copying to S3, these are stored as METADATA (S3 doesn't have traditional file permissions). If your application relies on Unix permissions, use EFS (not S3) as the target.

---

## 10. Tricky Scenario-Based Questions

**Q17: SCENARIO — During migration weekend, you performed the cutover but users report the application is 5x slower on AWS than on-premises. What are the most common causes?**

**A:**

**Root causes (in order of likelihood):**

1. **Database latency — app and DB now separated by network:**
```
On-prem: App server → DB server (same rack, <1ms latency)
AWS: EC2 (us-east-1a) → RDS (us-east-1b) = 1-2ms
But: EC2 (us-east-1) → On-prem DB (still migrating) = 20-50ms via DX!

Fix: Migrate DB BEFORE or SAME TIME as app
     Or: Use read replica in AWS, write to on-prem until DB cutover
```

2. **DNS resolution crossing network boundary:**
```
App resolves "dbserver.internal" → On-prem DNS → delays 
Every DB query adds DNS lookup latency

Fix: Update /etc/hosts or use Route 53 Resolver properly
     Cache DNS (nscd/systemd-resolved with positive TTL)
```

3. **EBS storage not warmed (first-access penalty):**
```
New EBS volumes restored from snapshot: First read = 20x slower!
Blocks fetched from S3 on first access (lazy loading)
Subsequent reads: Normal EBS speed

Fix: Initialize (warm) EBS volumes before cutover:
sudo fio --filename=/dev/xvdf --rw=read --bs=1M --direct=1 --iodepth=32
OR use EBS Fast Snapshot Restore (FSR) — costs extra but instant performance
```

4. **Instance under-sized (right-sizing too aggressive):**
```
Physical: 32 cores, 128GB RAM → AWS: m5.xlarge (4 vCPU, 16GB)
Right-sizing said "10% CPU utilization" but missed burst patterns

Fix: Temporarily scale UP, then optimize based on CloudWatch data
```

5. **Network throughput limit (instance-level bandwidth cap):**
```
On-prem: 10 Gbps dedicated NIC
AWS t3.medium: Up to 5 Gbps (burst, shared)
Large data transfers throttled by instance bandwidth

Fix: Use larger instance type OR Enhanced Networking (ENA)
```

6. **Missing connection pooling:**
```
On-prem: App → DB (same network, connection creation = 0.5ms)
AWS: App → RDS (create connection = 5-10ms including TLS handshake)
If app creates new connection per request = massive overhead

Fix: Implement connection pooling (PgBouncer, RDS Proxy, HikariCP)
```

7. **Security group / NACL causing retransmissions:**
```
Some traffic allowed, some blocked → TCP retransmission timeouts
Partial connectivity = worst kind (sometimes works, sometimes 3s timeout)

Fix: Review SG rules, enable VPC Flow Logs, check for rejected packets
```

**Tricky**: The #1 post-migration performance issue is latency between components that were previously co-located. A 1ms network hop multiplied by 100 DB queries per request = 100ms overhead that didn't exist on-premises. Solution: Move dependent components TOGETHER and use connection pooling aggressively.

---

**Q18: SCENARIO — You migrated 10 servers successfully but the 11th (Active Directory Domain Controller) breaks all other servers. Why?**

**A:**

**Active Directory is one of the hardest services to migrate because EVERYTHING depends on it.**

```
What depends on AD:
├── Windows authentication (ALL Windows servers)
├── DNS resolution (AD-integrated DNS)
├── Group Policy (GPOs for server configuration)
├── Service accounts (applications authenticating)
├── Certificate services (internal PKI)
├── LDAP queries (applications using AD as directory)
└── Kerberos (cross-server authentication)
```

**What breaks when AD is migrated wrong:**

1. **DNS breaks (most common):**
```
AD Domain Controller IS the DNS server for all joined machines
If DC is migrated but DNS client config not updated on remaining servers:
├── On-prem servers still point to old DC IP → DNS fails
├── AWS servers can't find DC → authentication fails
└── EVERYTHING stops (logins, app auth, service accounts)
```

2. **Kerberos time sensitivity:**
```
Kerberos requires < 5 minute clock skew between DC and clients
If migrated DC's time drifts (NTP config lost) → all auth fails!
Fix: Ensure NTP configured correctly after migration
```

3. **Sites and Services misconfigured:**
```
AD Site topology controls which DC clients talk to
If AWS DC not configured as new AD Site:
├── Clients may try to reach on-prem DC for auth
├── Over VPN/DX = slow authentication
└── DX goes down = ALL auth fails
```

**Correct AD migration approach:**

```
Option 1: Extend AD to AWS (RECOMMENDED during migration)
├── Deploy new DC in AWS VPC (EC2 with AD role)
├── Replicate from on-prem DC
├── Configure AD Site: "AWS-US-EAST-1"
├── AWS servers use AWS DC, on-prem servers use on-prem DC
├── Both coexist during migration
└── Decommission on-prem DC only after full migration

Option 2: AWS Managed Microsoft AD
├── AWS manages the DCs (patching, monitoring, HA)
├── Establish forest trust with on-prem AD
├── Supports: Group Policy, LDAP, Kerberos
├── Limitation: Can't install custom schema extensions
└── Best for: New deployments or when you want managed service

Option 3: AD Connector (simplest for migration period)
├── Proxy that forwards auth requests to on-prem AD
├── No AD data stored in AWS
├── Limitation: Requires constant connectivity to on-prem
└── Risk: If DX/VPN fails, ALL authentication fails
```

**Tricky**: NEVER migrate a Domain Controller using MGN/lift-and-shift! DC migration requires proper AD promotion/demotion procedures. Lifting a DC image can cause USN rollback, replication issues, or split-brain scenarios. Always deploy a NEW DC in AWS and let AD replication handle the data.

---

**Q19: SCENARIO — Migration is 70% complete. The remaining 30% of servers have dependencies on the already-migrated servers AND on-premises systems (bidirectional). How do you handle this "hybrid hell" period?**

**A:**

**The "messy middle" — the hardest part of any migration:**

```
Current state (hybrid):
├── 70% of apps on AWS
├── 30% of apps on-premises
├── Dependencies go BOTH directions
├── Some apps talk to services in BOTH locations
└── Duration: Can last weeks to months

Challenges:
├── Latency: Cross-network calls (AWS ↔ on-prem) add 20-50ms each
├── Reliability: VPN/DX outage affects cross-environment apps
├── Security: Traffic flowing between environments needs encryption
├── DNS: Some hostnames resolve to on-prem, others to AWS
├── Monitoring: Split across two environments
└── Rollback complexity: Reverting one app may break others
```

**Strategies to survive hybrid period:**

1. **Network: Ensure robust connectivity:**
```
Primary: Direct Connect (10 Gbps, < 5ms latency)
Secondary: Site-to-Site VPN (failover)
Monitor: DX connection state, BGP session, latency
```

2. **DNS: Split-horizon and service discovery:**
```
Route 53 Private Hosted Zone (for AWS resources)
+ On-prem DNS (for remaining on-prem resources)
+ Conditional forwarding in both directions
+ Service discovery: All services registered in one place
```

3. **Connectivity patterns:**
```
App on AWS calling service on-prem:
├── Use Private IP (via DX) — not public internet
├── Add caching layer to reduce cross-network calls
├── Implement circuit breaker (handle DX failure gracefully)
└── Monitor: Cross-environment latency metric

App on-prem calling service on AWS:
├── Use internal ALB (private) + VPN/DX routing
├── NLB with static IP (for whitelist-based firewalls on-prem)
└── Never expose via public internet during migration
```

4. **Minimize hybrid duration:**
```
Migration wave strategy:
├── Group apps by dependency clusters
├── Migrate entire cluster in same wave
├── Goal: No cluster split across environments
└── If split unavoidable: Put caching/queue between halves
```

5. **Async decoupling:**
```
Instead of: App-A (AWS) → sync HTTP → App-B (on-prem) [20ms added]
Use: App-A (AWS) → SQS queue → App-B (on-prem) reads from queue
Benefit: Tolerates latency, survives DX outage (queue buffers)
```

**Tricky**: The hybrid period exposes the weakest link in your architecture. If a monolithic app has 50 microservice dependencies split across environments, each call adds latency. 50 calls × 20ms = 1 second overhead. Solution: Migrate the entire dependency group together OR refactor to async communication during the hybrid period.

---

**Q20: SCENARIO — Your CFO asks: "We've been migrating for 6 months. We're now paying for BOTH the datacenter AND AWS. When will we stop paying double?"**

**A:**

**This is the "double-bubble" cost problem — the #1 executive complaint during migration.**

```
Cost timeline during migration:

Months 1-3:   On-prem: 100% | AWS: 10% | Total: 110% (just started)
Months 4-6:   On-prem: 100% | AWS: 30% | Total: 130% (peak "double-bubble")
Months 7-9:   On-prem:  80% | AWS: 50% | Total: 130% (still paying most on-prem)
Months 10-12: On-prem:  50% | AWS: 60% | Total: 110% (decommissioning starting)
Months 13-15: On-prem:  20% | AWS: 65% | Total:  85% (significant DC reduction)
Month 16+:    On-prem:   0% | AWS: 65% | Total:  65% (fully migrated, optimized)
```

**Why on-prem costs DON'T decrease linearly:**

1. **Datacenter fixed costs don't reduce until you EXIT:**
```
├── Lease: Paid monthly regardless of servers removed (contract term)
├── Power/cooling: Mostly fixed until entire rows powered down
├── Network: WAN/ISP contracts have minimum terms
├── Staff: Can't reduce team until enough migrated
└── You pay FULL datacenter costs until you can fully vacate
```

2. **Hardware refresh commitments:**
```
├── Servers on 3-5 year lease: Can't return early without penalty
├── Storage arrays: Maintenance contracts paid annually upfront
├── Network equipment: Same
└── Exit penalties may exceed remaining lease payments
```

**How to shorten double-bubble:**

1. **Accelerate migration (reduce hybrid period):**
```
├── Larger wave sizes (20-30 servers per wave instead of 10)
├── More parallel workstreams
├── Accept more risk (less testing per wave)
└── Automated migration tooling (MGN templates)
```

2. **Early datacenter exit triggers:**
```
├── Negotiate early lease termination clause upfront
├── Sublease unused space/racks
├── Return hardware to vendor (lease buyout vs keep paying)
└── Consolidate remaining on-prem servers to fewer racks
```

3. **AWS cost optimization during migration:**
```
├── Don't buy Reserved Instances yet (workloads still fluctuating)
├── Use Spot for dev/test (70-90% savings)
├── Right-size aggressively (don't replicate on-prem oversizing)
├── Shut down dev/test outside business hours (40% savings)
└── Use Savings Plans once workload stabilizes (post-migration)
```

4. **Quick wins to show savings:**
```
├── Retire unused apps immediately (20-30% of portfolio)
├── DR consolidation (eliminate secondary datacenter first)
├── License elimination (Oracle → Aurora, Exchange → SES)
└── Development environments to AWS first (immediate flexibility)
```

**Tricky**: The business case often promises "40% savings" but that's AFTER migration completion AND optimization. During migration, costs INCREASE 20-30%. Set executive expectations: "Total cost temporarily increases during migration, reaches breakeven at month 10-12, then decreases below on-prem baseline." If you don't set this expectation, the project gets killed at month 6 when costs are highest.

---

**Q21: SCENARIO — Your company has strict compliance (PCI-DSS). What additional considerations exist for migrating payment processing systems to AWS?**

**A:**

**PCI-DSS compliance framework for AWS migration:**

```
AWS Shared Responsibility for PCI:
├── AWS responsible for: Physical security, hypervisor, managed service infrastructure
│   (AWS is PCI Level 1 certified — highest level)
├── YOU responsible for: Everything IN the cloud
│   ├── Network segmentation
│   ├── Access control
│   ├── Encryption
│   ├── Logging and monitoring
│   ├── Vulnerability management
│   └── Application security
```

**Key PCI requirements mapped to AWS:**

| PCI Requirement | On-Premises | AWS Implementation |
|-----------------|-------------|-------------------|
| Network segmentation | VLANs + firewalls | VPC + Security Groups + NACLs |
| Restrict access to cardholder data | Physical access controls | IAM + encryption + least privilege |
| Encrypt transmission | SSL/TLS | ACM + ALB TLS + VPN/DX encryption |
| Encrypt storage | Disk encryption | EBS encryption (KMS) + S3 SSE |
| Track and monitor access | SIEM + logs | CloudTrail + GuardDuty + Config |
| Vulnerability scanning | Nessus/Qualys | Inspector + third-party in AWS |
| Penetration testing | Annual + on-demand | Allowed on AWS (most services, no pre-approval needed) |
| Key management | HSM | CloudHSM or KMS (FIPS 140-2 Level 3 vs Level 2) |

**Migration-specific PCI considerations:**

1. **Data in transit during migration:**
```
DMS replication of cardholder data MUST be encrypted:
├── DMS endpoint: SSL = required
├── Network path: Over Direct Connect with MACsec OR VPN
├── Never over public internet (even encrypted)
└── Validate certificate chain (no MITM possible)
```

2. **Separate CDE (Cardholder Data Environment):**
```
Dedicated VPC for PCI workloads:
├── No internet gateway (air-gapped from internet)
├── Access only via bastion in separate VPC (with MFA)
├── VPC flow logs enabled (all traffic logged)
├── No VPC peering to non-PCI VPCs
└── Separate AWS account (recommended: PCI account)
```

3. **Audit trail during migration:**
```
Document everything:
├── What data was migrated (data classification)
├── Who had access during migration (personnel)
├── Network paths data traveled (topology)
├── Encryption used at every stage (methods + keys)
└── When old copies were securely deleted (media sanitization)
```

**Tricky**: When you use AWS managed services (RDS, DynamoDB, S3), AWS handles some PCI controls for the infrastructure. But when you use EC2 (self-managed), YOU own ALL PCI controls on that instance. Using managed services REDUCES your PCI scope. This is a strong argument for: on-prem Oracle → RDS/Aurora (reduces PCI compliance burden).

---

**Q22: SCENARIO — You need to migrate a real-time trading application that cannot tolerate more than 100ms of added latency. How do you plan the migration?**

**A:**

**Ultra-low-latency migration is the hardest category.**

```
Requirements:
├── Current latency: 5ms (app to DB, same rack)
├── Maximum acceptable: 5ms + 100ms = 105ms total
├── Availability: 99.99% (52 minutes downtime/year)
├── Data consistency: Zero data loss (RPO = 0)
└── Cutover window: Maximum 30 minutes
```

**Architecture considerations:**

1. **Network latency reduction:**
```
Standard AWS:
├── Same AZ: < 1ms (placement group: < 0.25ms)
├── Cross-AZ: 1-2ms
├── DX to on-prem: 2-10ms (depends on distance)
└── Public internet: 20-100ms (unacceptable for this use case)

Optimizations:
├── Placement Group (cluster): All instances on same rack = <0.25ms
├── Enhanced Networking (ENA): Kernel bypass, 25-100 Gbps
├── EFA (Elastic Fabric Adapter): HPC networking, <10μs
└── AWS Local Zones: AWS infrastructure in metro areas (ultra-low latency)
```

2. **Compute optimization:**
```
Instance types for low-latency:
├── c5n.18xlarge: 100 Gbps networking
├── z1d: Highest single-core performance (up to 4.0 GHz)
├── Bare metal (.metal): No hypervisor overhead
└── c6i.metal: 128 vCPU, 256GB RAM, 50 Gbps, no virtualization
```

3. **Storage for low-latency:**
```
├── io2 Block Express: Up to 256,000 IOPS, sub-millisecond latency
├── Instance store (local NVMe): <0.1ms latency (but ephemeral!)
├── FSx for Lustre: High-performance parallel filesystem
└── ElastiCache Redis on r6gd (local SSD): Sub-millisecond reads
```

4. **Migration approach:**
```
Phase 1: Set up AWS environment with performance validation
├── Deploy same architecture in AWS
├── Run synthetic workload at production volume
├── Measure ACTUAL latency (must be < 100ms added)
└── Tune until meeting SLA

Phase 2: Data replication with zero RPO
├── Synchronous replication: Aurora Multi-AZ (sync commit)
├── Or: EC2 + Oracle Data Guard (synchronous mode)
├── Validate zero data loss under failure scenarios
└── Run parallel for 2+ weeks validating consistency

Phase 3: Cutover (30-minute window)
├── Pre-cutover: Verify replication lag = 0
├── T-0: Stop writes to on-prem (put app in read-only/maintenance)
├── T+5: Verify final sync complete
├── T+10: DNS switch (TTL pre-lowered to 60 seconds)
├── T+15: Validate new environment serving traffic
├── T+20: Monitor latency metrics
└── T+30: Confirm or rollback
```

**Tricky**: For trading applications, even PLACEMENT GROUP latency (0.25ms) between app and DB might not be enough if on-premises they were on the SAME physical server (shared memory access). In such cases, co-locate app + DB on the same EC2 instance (or same bare metal) — accepting the operational complexity for the latency requirement.

---

**Q23: How do you validate that migration was successful? What metrics confirm "go" vs "no-go" for cutover?**

**A:**

**Pre-cutover validation checklist:**

```
INFRASTRUCTURE VALIDATION:
├── □ All servers responding to health checks
├── □ DNS resolution working for all services
├── □ Network connectivity between all tiers verified
├── □ Security groups allowing correct traffic (tested)
├── □ SSL certificates valid and not expiring soon
├── □ Backup jobs running and completing
├── □ Monitoring/alerting configured and tested
└── □ Logs flowing to CloudWatch/centralized logging

DATA VALIDATION:
├── □ Row counts match between source and target (all tables)
├── □ Checksums match for critical tables
├── □ DMS validation task shows 0 mismatches
├── □ Application data integrity checks pass
├── □ File storage: File counts and sizes match
└── □ No replication lag (CDC lag = 0)

APPLICATION VALIDATION:
├── □ All services start correctly
├── □ Health check endpoints returning 200
├── □ Smoke tests pass (core user journeys)
├── □ Load test shows acceptable performance
├── □ Response times within SLA (p50, p95, p99)
├── □ Error rate < baseline threshold
└── □ Integration tests with all external systems pass

PERFORMANCE VALIDATION:
├── □ Response time: p99 < on-prem p99 + acceptable overhead
├── □ Throughput: Handles peak load + 20% headroom
├── □ Database query performance: Critical queries < baseline
├── □ CPU, Memory, Disk: All below 70% at expected peak
└── □ No throttling (EBS IOPS, Lambda concurrency, DynamoDB)
```

**Go/No-Go decision matrix:**

| Category | GO criteria | NO-GO triggers |
|----------|-------------|----------------|
| Data integrity | 100% match | Any mismatch in critical tables |
| Performance | Within 20% of baseline | > 50% degradation |
| Error rate | < 1% | > 5% |
| Availability | All health checks pass | Any critical service down |
| Security | No open findings | Critical vulnerability discovered |
| Rollback | Tested and confirmed working | Rollback plan untested |

**Tricky**: The most dangerous moment is AFTER cutover when you think everything is fine but haven't seen peak traffic yet. Always schedule cutover BEFORE a known peak (e.g., Friday evening if peak is Monday morning) so you have a weekend to address issues before full load hits. NEVER cut over on Friday evening before a long weekend.
