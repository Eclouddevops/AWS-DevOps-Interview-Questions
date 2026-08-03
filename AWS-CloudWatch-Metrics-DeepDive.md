# AWS CloudWatch Metrics — Deep-Dive Interview Q&A

## Table of Contents
1. [CloudWatch Fundamentals](#1-cloudwatch-fundamentals)
2. [EC2 Metrics & CPU Burst Credits](#2-ec2-metrics--cpu-burst-credits)
3. [RDS / Aurora Metrics](#3-rds--aurora-metrics)
4. [ElastiCache Metrics (Redis/Memcached)](#4-elasticache-metrics)
5. [ALB / NLB Metrics](#5-alb--nlb-metrics)
6. [Lambda Metrics](#6-lambda-metrics)
7. [ECS / Fargate Metrics](#7-ecs--fargate-metrics)
8. [S3 Metrics](#8-s3-metrics)
9. [DynamoDB Metrics](#9-dynamodb-metrics)
10. [Custom Metrics & Alarms](#10-custom-metrics--alarms)
11. [Tricky Scenario-Based Questions](#11-tricky-scenario-based-questions)

---

## 1. CloudWatch Fundamentals


**Q1: Explain CloudWatch architecture — Namespaces, Metrics, Dimensions, Statistics, and Periods.**

**A:**

```
CloudWatch Hierarchy:

Namespace (e.g., AWS/EC2, AWS/RDS, Custom/MyApp)
└── Metric (e.g., CPUUtilization, FreeableMemory)
    └── Dimensions (e.g., InstanceId=i-123, DBInstanceIdentifier=mydb)
        └── Datapoints (timestamp + value)
            └── Statistics (Average, Sum, Min, Max, SampleCount, pNN)
```

| Concept | Description | Example |
|---------|-------------|---------|
| **Namespace** | Container for metrics (groups by service) | `AWS/EC2`, `AWS/RDS`, `AWS/ELB` |
| **Metric** | Time-ordered set of data points | `CPUUtilization`, `FreeableMemory` |
| **Dimension** | Name/value pair that identifies a metric uniquely | `InstanceId=i-abc123` |
| **Period** | Length of time for one data point (seconds) | 60, 300, 3600 |
| **Statistic** | Aggregation over a period | Average, Sum, Min, Max, p99 |
| **Resolution** | Standard (60s) or High-resolution (1s) | Custom metrics can be 1s |

**Data retention:**
```
< 60 seconds (high-res): Retained for 3 hours
60 seconds:              Retained for 15 days
300 seconds (5 min):     Retained for 63 days
3600 seconds (1 hour):   Retained for 455 days (15 months)
```

**Tricky**: EC2 basic monitoring sends metrics every 5 minutes (free). Detailed monitoring sends every 1 minute (paid). RDS sends every 60 seconds by default. Enhanced Monitoring for RDS sends every 1-60 seconds (OS-level metrics via agent).

---

**Q2: What is the difference between CloudWatch Metrics, CloudWatch Logs, CloudWatch Alarms, and CloudWatch Insights?**

**A:**

| Service | What It Does | Data Type | Use Case |
|---------|-------------|-----------|----------|
| **Metrics** | Numeric time-series data | Numbers (CPU%, bytes, counts) | Performance monitoring, dashboards |
| **Logs** | Text log storage and analysis | Log events (text/JSON) | Application debugging, audit trails |
| **Alarms** | Threshold-based notifications | Watches a metric | Alert when CPU > 80%, auto-scaling trigger |
| **Logs Insights** | SQL-like log querying | Queries log groups | Find errors, analyze patterns in logs |
| **Container Insights** | ECS/EKS monitoring | Metrics + Logs | Container-level visibility |
| **Application Insights** | .NET/Java app monitoring | Metrics + anomaly | Detect app problems automatically |
| **Contributor Insights** | Top-N analysis | Time-series rules | Find top talkers, hottest keys |
| **Metric Streams** | Real-time metric export | Firehose → S3/partners | Send to Datadog, Splunk, etc. |

**Tricky**: CloudWatch Alarms evaluate ONLY metrics, not logs directly. To alert on log patterns, you must create a **Metric Filter** on a Log Group (e.g., count "ERROR" occurrences → custom metric → alarm on that metric).

---

**Q3: How do CloudWatch Alarms work? Explain states, evaluation, and actions.**

**A:**

**Alarm States:**
```
OK → Metric is within threshold
ALARM → Metric breached threshold
INSUFFICIENT_DATA → Not enough data to evaluate (startup or missing data)
```

**Evaluation:**
```
Alarm triggers when: M out of N consecutive evaluation periods breach threshold

Example: "CPU > 80% for 3 out of 5 periods (5-min periods)"
├── Period: 300 seconds
├── Evaluation periods: 5
├── Datapoints to alarm: 3
└── Total evaluation window: 25 minutes
```

**Missing Data Treatment:**
| Setting | Behavior | Use When |
|---------|----------|----------|
| `missing` | Alarm stays in current state | Default — safe |
| `notBreaching` | Treat as within threshold (OK) | Metric only exists during problems |
| `breaching` | Treat as threshold breached | Conservative — assume worst |
| `ignore` | Skip missing periods | Intermittent metric |

**Actions:**
```
ALARM state → SNS notification, Auto Scaling action, EC2 action (stop/terminate/reboot)
OK state → SNS notification (recovery alert)
INSUFFICIENT_DATA → SNS notification
```

**Composite Alarms:**
```
Alarm: "Service Down" = Alarm_HighCPU AND Alarm_High5xx AND Alarm_HighLatency
└── Only triggers when ALL sub-alarms are in ALARM state
└── Reduces alert noise (single alarm might be false positive)
```

**Tricky**: An alarm on `Average CPUUtilization > 80%` with a 5-minute period might MISS short spikes. A 2-second spike to 100% averaged over 5 minutes = 0.7% average. Use `Maximum` statistic or shorter periods to catch spikes.

---

## 2. EC2 Metrics & CPU Burst Credits

**Q4: Explain ALL important EC2 CloudWatch metrics. Which require the CloudWatch Agent?**

**A:**

**Default EC2 Metrics (no agent needed):**

| Metric | What It Measures | Key Insight |
|--------|-----------------|-------------|
| `CPUUtilization` | % CPU used | High = compute bottleneck |
| `DiskReadOps` / `DiskWriteOps` | IOPS count | Disk I/O pressure |
| `DiskReadBytes` / `DiskWriteBytes` | Throughput | Data volume |
| `NetworkIn` / `NetworkOut` | Network bytes | Bandwidth usage |
| `NetworkPacketsIn` / `NetworkPacketsOut` | Packet count | Connection density |
| `StatusCheckFailed` | Hardware/software health | 1 = UNHEALTHY |
| `StatusCheckFailed_Instance` | OS-level check | Software/config issue |
| `StatusCheckFailed_System` | AWS hardware check | Hardware failure (migrate!) |
| `MetadataNoToken` | IMDSv1 usage (no token) | Security: migrate to IMDSv2 |

**CloudWatch Agent Required (OS-level metrics):**

| Metric | What It Measures | Why Default Misses It |
|--------|-----------------|----------------------|
| `mem_used_percent` | Memory utilization | EC2 hypervisor can't see inside OS |
| `disk_used_percent` | Disk space usage | Hypervisor doesn't track filesystem |
| `swap_used_percent` | Swap usage | OS-level only |
| `processes_total` | Process count | OS-level only |
| `cpu_usage_iowait` | I/O wait percentage | Granular CPU breakdown |
| `netstat_tcp_established` | TCP connections | OS networking stack |

**Tricky**: CloudWatch does NOT provide memory or disk utilization by default! This is the #1 interview trap. You MUST install the CloudWatch Agent. Without it, your EC2 can run out of memory with no alarm.

---

**Q5: Explain T2/T3 CPU burst credits in detail. What are `CPUCreditUsage`, `CPUCreditBalance`, `CPUSurplusCreditBalance`, and `CPUSurplusCreditsCharged`?**

**A:**

**How CPU Burst Credits Work:**

```
T3.medium baseline: 20% CPU

When CPU < 20% → EARNING credits (max accumulation: 576 credits for t3.medium)
When CPU > 20% → SPENDING credits
When credits = 0 → Performance LIMITED to baseline (20%) unless Unlimited mode

1 CPU credit = 1 vCPU at 100% for 1 minute
            = 2 vCPUs at 50% for 1 minute  
            = 1 vCPU at 50% for 2 minutes
```

**All T-instance burst metrics explained:**

| Metric | Description | What to Watch For |
|--------|-------------|-------------------|
| `CPUCreditUsage` | Credits spent in the period | High value = heavy bursting |
| `CPUCreditBalance` | Credits available (bank) | 0 = will be throttled |
| `CPUSurplusCreditBalance` | Credits spent BEYOND earned (Unlimited mode) | Growing = overspending |
| `CPUSurplusCreditsCharged` | Surplus credits charged as $ (Unlimited) | Surprise bill! |

**T3 Instance Credit Details:**

| Instance | Baseline | Credits/hour | Max Accumulation | Max Burst Duration at 100% |
|----------|----------|-------------|------------------|-----------------------------|
| t3.nano | 5% | 6 | 144 | 24 min |
| t3.micro | 10% | 12 | 288 | 24 min |
| t3.small | 20% | 24 | 576 | 24 min |
| t3.medium | 20% | 24 | 576 | 24 min |
| t3.large | 30% | 36 | 864 | 24 min |
| t3.xlarge | 40% | 96 | 2304 | 24 min |

**Standard vs Unlimited Mode:**

```
STANDARD Mode:
├── Credits run out → CPU capped at baseline
├── No extra cost
└── Predictable performance and billing

UNLIMITED Mode (default for T3):
├── Credits run out → Keeps bursting using SURPLUS credits
├── Surplus credits charged at $0.05/vCPU-hour
├── Can result in unexpected bills!
└── Instance never throttled
```

**Real-world scenario:**
```
T3.medium (20% baseline, Unlimited mode):
├── Normal load: 15% CPU → Earning credits ✓
├── Spike: 80% CPU for 2 hours → Spending 72 credits/hour
├── Credits exhausted after ~8 hours of accumulation burned
├── Now using surplus credits → $0.05/vCPU-hour charge
└── 2 vCPU × $0.05 × remaining hours = surprise cost

ALARM SETUP:
├── CPUCreditBalance < 50 → Warning (credits running low)
├── CPUCreditBalance = 0 → Critical (will throttle or charge)
├── CPUSurplusCreditBalance > 0 → Alert (spending money on burst)
└── CPUUtilization > 20% sustained → Consider larger instance
```

**Tricky**: A `t3.medium` running at 60% CPU 24/7 in Unlimited mode costs MORE than a `m5.large` because of surplus credit charges! Always set alarms on `CPUCreditBalance` and size instances based on sustained workload, not peak.

---

**Q6: Your t3.large has CPUCreditBalance at 0 and CPUUtilization shows only 30%. But users report the application is slow. What's happening?**

**A:**

**The 30% you see IS the throttled baseline!** The instance is being capped at its baseline performance (30% for t3.large).

**Investigation:**
```bash
# Check credit metrics over time
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUCreditBalance \
  --dimensions Name=InstanceId,Value=i-123 \
  --start-time 2024-01-01T00:00:00 \
  --end-time 2024-01-01T12:00:00 \
  --period 300 \
  --statistics Average
```

**What happened:**
1. Application had a traffic spike → CPU went to 80%+
2. Credits depleted over several hours
3. Instance throttled to 30% baseline (t3.large baseline)
4. Application needs >30% to function properly → SLOW
5. CPUUtilization metric shows 30% because that's ALL it's allowed

**Solutions:**
1. **Immediate**: Stop/Start instance (credits reset on new hardware for some types) — NOT guaranteed
2. **Short-term**: Enable Unlimited mode (costs money but removes throttle)
3. **Long-term**: Upgrade to fixed-performance instance (m5.large, c5.large)
4. **Best practice**: Set alarm on `CPUCreditBalance < 100` → alert before throttling

**Tricky**: New T3 instances start with 576 credits (t3.medium). But STOPPED instances lose credits over time! A t3 instance stopped for days starts with fewer credits — it may throttle immediately on restart under load.

---


## 3. RDS / Aurora Metrics

**Q7: Explain `FreeableMemory` in detail. What does it actually measure? When should you be alarmed?**

**A:**

**`FreeableMemory`** = Amount of available RAM (in bytes) on the RDS instance that is NOT being used by the database engine or OS.

```
Total Instance Memory
├── OS Reserved (~5-10%)
├── Database Engine (buffer pool, connections, temp tables, sort buffers)
│   ├── InnoDB Buffer Pool (largest consumer)
│   ├── Connection memory (per-connection overhead)
│   ├── Query execution memory (sorts, joins, temp tables)
│   └── Internal caches
├── File system cache (OS page cache)
└── FreeableMemory ← WHAT'S LEFT (this is the metric)
```

**Key thresholds:**

| FreeableMemory | Status | Action |
|----------------|--------|--------|
| > 25% of total | Healthy | No action |
| 10-25% of total | Warning | Monitor trend |
| 5-10% of total | Critical | Investigate, plan upgrade |
| < 5% of total | Emergency | Risk of OOM, swap usage, crashes |
| Decreasing trend | Memory leak | Check connections, queries |

**What causes low FreeableMemory:**
1. **Too many connections**: Each MySQL connection uses 1-10MB+ memory
2. **Large InnoDB buffer pool**: Configured to use too much RAM (default: 75% of instance RAM)
3. **Memory-intensive queries**: Large sorts, GROUP BY, JOINs with temp tables
4. **Connection leaks**: App not closing connections → accumulating
5. **Instance too small**: Workload outgrew the instance class

**Alarm setup:**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "RDS-LowMemory-prod-db" \
  --namespace AWS/RDS \
  --metric-name FreeableMemory \
  --dimensions Name=DBInstanceIdentifier,Value=prod-db \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 500000000 \  # 500MB in bytes
  --comparison-operator LessThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456:db-alerts
```

**Tricky**: FreeableMemory INCLUDES the OS file system cache. Linux uses free RAM for caching (good for performance). If FreeableMemory drops but `SwapUsage` is 0, the OS may just be aggressively caching disk blocks — this is often FINE. Only worry when `SwapUsage > 0` (means actual memory pressure is forcing data to disk).

---

**Q8: Explain `SwapUsage` for RDS. Why is ANY swap usage a serious problem?**

**A:**

**`SwapUsage`** = Bytes of swap space used on the RDS instance.

```
Normal: SwapUsage = 0 (all data in RAM)
Problem: SwapUsage > 0 (OS forced to page memory to disk)
```

**Why swap is critical for databases:**
- Database performance depends on data being in RAM (buffer pool)
- Swap = reading/writing pages to EBS disk (100x slower than RAM)
- Even 1MB of swap can cause 100x latency spikes for affected queries
- Indicates system is OUT of usable RAM

**What causes swap usage:**
1. `FreeableMemory` approaching 0 → OS starts swapping
2. InnoDB buffer pool too large for instance
3. Sudden connection spike (each connection allocates memory)
4. Large temp tables for complex queries
5. Memory leak in stored procedures/plugins

**Alarm:** 
```
SwapUsage > 0 for 2 consecutive periods → CRITICAL ALERT
```

**Resolution:**
```
1. Immediate: Kill expensive queries (SHOW PROCESSLIST → KILL <id>)
2. Short-term: Reduce max_connections, reduce buffer_pool_size
3. Long-term: Scale UP instance class (more RAM)
4. Investigate: Check queries doing filesort/tmp tables
```

**Tricky**: RDS Enhanced Monitoring shows `swap` in OS metrics. The standard `SwapUsage` metric in `AWS/RDS` namespace may have 60-second delay. For faster detection, use Enhanced Monitoring (1-second resolution).

---

**Q9: Explain ALL critical RDS metrics — what to monitor and why.**

**A:**

| Metric | What It Measures | Alarm Threshold | Why It Matters |
|--------|-----------------|-----------------|----------------|
| `CPUUtilization` | Database engine CPU usage | > 80% sustained | Query performance degradation |
| `FreeableMemory` | Available RAM (bytes) | < 500MB or < 10% total | Risk of swap, OOM |
| `SwapUsage` | Swap space used (bytes) | > 0 | CRITICAL: DB reading from disk |
| `DatabaseConnections` | Active connections count | > 80% of max_connections | Connection pool exhaustion |
| `ReadIOPS` / `WriteIOPS` | I/O operations per second | Near provisioned IOPS limit | Disk bottleneck |
| `ReadLatency` / `WriteLatency` | Avg time per I/O op (seconds) | Read > 5ms, Write > 10ms | Storage performance issue |
| `ReadThroughput` / `WriteThroughput` | Bytes/second to storage | Near EBS throughput limit | Bandwidth saturation |
| `FreeStorageSpace` | Remaining disk space (bytes) | < 20% or < 10GB | Risk of DB crash if full |
| `DiskQueueDepth` | Pending I/O requests | > 10 sustained | Storage overloaded |
| `BurstBalance` | % EBS burst credits remaining (gp2) | < 20% | Risk of IOPS throttling |
| `ReplicaLag` | Seconds behind primary (replicas) | > 30 seconds | Read replica stale data |
| `NetworkReceiveThroughput` | Incoming network bytes/s | Near instance limit | Network bottleneck |
| `NetworkTransmitThroughput` | Outgoing network bytes/s | Near instance limit | Network bottleneck |

**RDS Enhanced Monitoring (OS-level, 1-60s resolution):**
```json
{
  "cpuUtilization": {"user": 45.2, "system": 5.1, "wait": 2.3, "idle": 47.4},
  "memory": {"total": 16384, "free": 2048, "cached": 8192, "buffers": 512},
  "swap": {"total": 2048, "free": 2048, "cached": 0},
  "diskIO": [{"readIOsPS": 500, "writeIOsPS": 200, "avgQueueLen": 2.5}],
  "network": [{"rx": 50000000, "tx": 30000000}],
  "processList": [{"name": "mysqld", "vss": 12000000, "rss": 8000000}]
}
```

**Tricky**: Standard CloudWatch `CPUUtilization` for RDS includes BOTH your queries AND RDS maintenance tasks (backups, replication). If CPU spikes during backup windows, that's normal. Use `Enhanced Monitoring` → `cpuUtilization.user` for query-only CPU.

---

**Q10: Explain `BurstBalance` for RDS (gp2 volumes). What happens when it hits 0?**

**A:**

**`BurstBalance`** = Percentage of I/O burst credits remaining for gp2 EBS volumes.

```
gp2 Performance Model:
├── Baseline IOPS: 3 IOPS per GB (min 100, max 16,000)
├── Burst IOPS: Up to 3,000 IOPS (for volumes < 1000 GB)
├── Burst credits: Earn when below baseline, spend when above
└── BurstBalance: 0% = NO burst available = throttled to baseline

Example: 100 GB gp2 volume
├── Baseline: 300 IOPS (100 GB × 3 IOPS/GB)
├── Can burst to: 3,000 IOPS
├── Burst duration at 3,000 IOPS: ~33 minutes (from full)
└── BurstBalance at 0% → Limited to 300 IOPS = VERY SLOW DATABASE
```

**When BurstBalance hits 0:**
- IOPS limited to baseline (e.g., 300 IOPS for 100GB volume)
- Read/Write latency increases dramatically (10x-100x)
- `DiskQueueDepth` spikes (I/O requests queuing)
- Database queries time out
- Application appears "frozen"

**Monitoring alarm:**
```
BurstBalance < 20% → Warning (will run out in ~7 minutes at max burst)
BurstBalance = 0% → Critical (already throttled)
```

**Solutions:**
1. **Increase volume size** (larger gp2 = higher baseline IOPS)
   - 334 GB gp2 = 1,002 IOPS baseline (no burst needed for many workloads)
   - 1,000+ GB gp2 = 3,000+ IOPS (never needs burst at all)
2. **Switch to gp3**: Consistent 3,000 IOPS baseline regardless of size (no burst concept)
3. **Switch to io1/io2**: Provisioned IOPS (guaranteed, no burst model)
4. **Optimize queries**: Reduce I/O by adding indexes, fixing full table scans

**Tricky**: gp3 volumes DON'T have BurstBalance! They have consistent 3,000 IOPS baseline (free) + up to 16,000 provisioned. The BurstBalance metric ONLY applies to gp2. If you see this metric, you're on gp2 and should consider migrating to gp3 (usually cheaper AND faster).

---

## 4. ElastiCache Metrics

**Q11: Explain `CacheHits` and `CacheMisses` in detail. What is cache hit ratio and how do you optimize it?**

**A:**

**`CacheHits`** = Number of successful read requests that found the key in cache.
**`CacheMisses`** = Number of read requests where the key was NOT in cache (must fetch from origin/DB).

```
Cache Hit Ratio = CacheHits / (CacheHits + CacheMisses) × 100%

Example:
├── CacheHits: 9,500/minute
├── CacheMisses: 500/minute  
├── Hit Ratio: 9,500 / 10,000 = 95% ← GOOD
└── Every miss = a database query (slow, expensive)
```

**Hit ratio benchmarks:**
| Hit Ratio | Assessment | Action |
|-----------|------------|--------|
| > 95% | Excellent | Maintain |
| 85-95% | Good | Minor tuning |
| 70-85% | Needs work | Review TTLs, key patterns |
| 50-70% | Poor | Major redesign needed |
| < 50% | Cache is barely helping | Architecture issue |

**Causes of LOW cache hit ratio:**

1. **TTL too short**: Keys expire before being reused
   ```
   Fix: Increase TTL based on data change frequency
   Tradeoff: Higher TTL = staler data
   ```

2. **Cache too small**: Evictions happening (LRU removing useful keys)
   ```
   Check: Evictions metric > 0 → cache is full, evicting old keys
   Fix: Increase node type or add nodes
   ```

3. **Cold cache**: After restart, cache is empty (all misses)
   ```
   Fix: Cache warming strategy on startup
   ```

4. **Random key patterns**: Each request uses unique keys (no reuse)
   ```
   Example: Caching by timestamp → every request is different
   Fix: Normalize cache keys (round timestamps, use user_id not session_id)
   ```

5. **Cache stampede**: Popular key expires → hundreds of simultaneous misses
   ```
   Fix: Staggered TTL, cache-aside with locking, background refresh
   ```

**CloudWatch alarm:**
```bash
# Alert when hit ratio drops below 80%
# Use Math Expression:
# CacheHitRatio = (CacheHits / (CacheHits + CacheMisses)) * 100
aws cloudwatch put-metric-alarm \
  --alarm-name "ElastiCache-LowHitRatio" \
  --metrics '[
    {"Id":"hits","MetricStat":{"Metric":{"Namespace":"AWS/ElastiCache","MetricName":"CacheHits","Dimensions":[{"Name":"CacheClusterId","Value":"prod-redis"}]},"Period":300,"Stat":"Sum"}},
    {"Id":"misses","MetricStat":{"Metric":{"Namespace":"AWS/ElastiCache","MetricName":"CacheMisses","Dimensions":[{"Name":"CacheClusterId","Value":"prod-redis"}]},"Period":300,"Stat":"Sum"}},
    {"Id":"ratio","Expression":"(hits/(hits+misses))*100","Label":"HitRatio"}
  ]' \
  --threshold 80 \
  --comparison-operator LessThanThreshold
```

**Tricky**: A hit ratio of 100% might actually indicate a PROBLEM — it could mean nothing new is being cached (no writes), or cache is serving stale data because TTL is infinite. Healthy caches have some misses (new data being fetched and cached).

---

**Q12: Explain ALL critical ElastiCache (Redis) metrics.**

**A:**

| Metric | What It Measures | Alarm Threshold | Impact |
|--------|-----------------|-----------------|--------|
| `CacheHits` | Successful key lookups | Use ratio with CacheMisses | Cache effectiveness |
| `CacheMisses` | Failed key lookups (key not found) | Rising trend → investigation | DB load increases |
| `Evictions` | Keys removed to make space (LRU) | > 0 sustained | Cache too small! |
| `CurrConnections` | Current client connections | > 80% of maxclients | Connection exhaustion |
| `NewConnections` | New connections per period | High churn → connection pool issue | Missing connection pooling |
| `FreeableMemory` | Available RAM on cache node | < 10% of total | OOM risk, evictions |
| `BytesUsedForCache` | Memory used by cached data | > 80% of node memory | Near capacity |
| `CPUUtilization` | Node CPU usage | > 65% (Redis is single-threaded!) | Performance degradation |
| `EngineCPUUtilization` | Redis engine thread CPU | > 90% → bottleneck | Need to scale |
| `ReplicationLag` | Replica behind primary (seconds) | > 1 second | Stale reads from replica |
| `SaveInProgress` | RDB snapshot in progress (0/1) | N/A | Causes latency spike |
| `CacheHitRate` | Calculated hit percentage | < 80% | Cache ineffective |
| `NetworkBytesIn/Out` | Network throughput | Near node limit | Network bottleneck |
| `CommandsProcessed` | Total commands executed/sec | Baseline comparison | Workload changes |
| `CurrItems` | Total keys stored | Track growth | Capacity planning |

**Critical insight — Redis CPU:**
```
Redis is SINGLE-THREADED for command processing!

CPUUtilization metric = ALL CPU cores averaged
EngineCPUUtilization = Redis engine thread ONLY

Example: 4-core node
├── CPUUtilization: 25% (one core at 100%, three at 0%)
├── EngineCPUUtilization: 100% ← REDIS IS MAXED OUT!
└── You'd think 25% CPU is fine, but Redis is actually at capacity!

ALWAYS alert on EngineCPUUtilization for Redis, not CPUUtilization!
```

**Tricky**: `Evictions > 0` means cache is FULL and removing old keys to make room for new ones. If evictions are happening AND cache misses are high, your cache is too small. But evictions with HIGH hit ratio might be OK — LRU is removing rarely-used keys successfully.

---

**Q13: Your ElastiCache Redis cluster shows `Evictions` increasing rapidly. What do you do?**

**A:**

**Immediate diagnosis:**
```bash
# Check memory usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name BytesUsedForCache \
  --dimensions Name=CacheClusterId,Value=prod-redis-001

# Check key count
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CurrItems

# Check if specific key patterns are growing
redis-cli --bigkeys  # Find largest keys
redis-cli info memory  # Memory breakdown
redis-cli info keyspace  # Keys per database
```

**Root causes and solutions:**

1. **Cache growing beyond capacity (most common):**
   ```
   Symptom: CurrItems keeps growing, BytesUsedForCache near max
   Fix: 
   ├── Set TTL on ALL keys (never cache forever unless necessary)
   ├── Review maxmemory-policy (allkeys-lru recommended)
   ├── Scale UP to larger node type
   └── Scale OUT with cluster mode (sharding)
   ```

2. **Large keys consuming disproportionate memory:**
   ```
   Symptom: Few keys but high memory usage
   Fix:
   ├── redis-cli --bigkeys (find them)
   ├── Compress large values before caching
   ├── Split large hashes/lists into smaller keys
   └── Use Redis Streams instead of large lists
   ```

3. **Keys without TTL accumulating:**
   ```
   Symptom: CurrItems only grows, never shrinks
   Fix:
   ├── Audit keys: redis-cli --scan | xargs redis-cli TTL
   ├── Set default TTL in application code
   └── Use EXPIRE on existing keys programmatically
   ```

4. **Sudden workload change:**
   ```
   Symptom: Evictions spike after deployment or traffic change
   Fix:
   ├── Review new code caching patterns
   ├── Check for cache stampede (popular key expired → many new keys written)
   └── Increase node size temporarily
   ```

**Eviction policies (maxmemory-policy):**
| Policy | Behavior | Best For |
|--------|----------|----------|
| `allkeys-lru` | Evict least recently used key (any key) | General caching |
| `volatile-lru` | Evict LRU among keys WITH expire set | Mixed persistent + cache |
| `allkeys-lfu` | Evict least frequently used (Redis 4.0+) | Frequency-based caching |
| `noeviction` | Return error on write when full | Critical data (don't lose!) |
| `volatile-ttl` | Evict keys with shortest TTL first | TTL-based priority |

---


## 5. ALB / NLB Metrics

**Q14: Explain critical ALB metrics. How do you differentiate ALB issues from backend issues?**

**A:**

| Metric | Description | ALB Issue or Backend? |
|--------|-------------|----------------------|
| `RequestCount` | Total requests processed | Baseline (neither) |
| `HTTPCode_ELB_4XX_Count` | 4xx FROM ALB itself | ALB issue (client or config) |
| `HTTPCode_ELB_5XX_Count` | 5xx FROM ALB itself | ALB issue (no healthy targets, timeout) |
| `HTTPCode_Target_2XX_Count` | 2xx from backend | Backend healthy |
| `HTTPCode_Target_4XX_Count` | 4xx from backend | Backend returning errors |
| `HTTPCode_Target_5XX_Count` | 5xx from backend | Backend crashing |
| `TargetResponseTime` | Time from ALB→target→response | Backend performance |
| `ActiveConnectionCount` | Current active connections | Load indicator |
| `HealthyHostCount` | Healthy targets | < desired = problem |
| `UnHealthyHostCount` | Unhealthy targets | > 0 = investigate |
| `RejectedConnectionCount` | Connections rejected | ALB at capacity (rare) |
| `TargetConnectionErrorCount` | ALB can't connect to target | Target down/unreachable |
| `RuleEvaluations` | Listener rule evaluations | Routing complexity |
| `ConsumedLCUs` | Load Balancer Capacity Units used | Cost + capacity |

**Critical distinction:**
```
ELB_5XX = ALB generated the error (backend didn't respond properly)
├── 502: Target sent invalid response or closed connection
├── 503: No registered/healthy targets
├── 504: Target didn't respond within timeout

Target_5XX = Backend application returned 5xx (app error)
├── 500: Application internal error
├── 503: Application overloaded
└── These are YOUR code problems, not infrastructure
```

**Tricky**: `HTTPCode_ELB_502` + `TargetConnectionErrorCount > 0` = target is refusing connections (app not running on port or crashed). But `HTTPCode_ELB_502` + `TargetResponseTime near idle timeout` = target is too slow (increase timeout or fix backend).

---

## 6. Lambda Metrics

**Q15: Explain ALL Lambda CloudWatch metrics. Which indicate problems vs normal behavior?**

**A:**

| Metric | Description | Problem Indicator |
|--------|-------------|-------------------|
| `Invocations` | Total function executions | Baseline |
| `Errors` | Invocations that returned error | > 1% of invocations |
| `Throttles` | Rejected due to concurrency limit | > 0 = scale issue |
| `Duration` | Execution time (ms) | Near timeout = risk |
| `ConcurrentExecutions` | Simultaneous executions | Near account limit |
| `UnreservedConcurrentExecutions` | Executions not using reserved capacity | Monitor for starvation |
| `IteratorAge` | Age of last record processed (streams) | Growing = falling behind |
| `DeadLetterErrors` | Failures delivering to DLQ | > 0 = DLQ problem |
| `DestinationDeliveryFailures` | Failures delivering to destination | > 0 = destination issue |
| `ProvisionedConcurrencyInvocations` | Invocations using provisioned capacity | Efficiency tracking |
| `ProvisionedConcurrencySpilloverInvocations` | Invocations BEYOND provisioned | Cold starts happening |
| `ProvisionedConcurrencyUtilization` | % of provisioned capacity in use | < 50% = overpaying |
| `PostRuntimeExtensionsDuration` | Time for extensions after function | Performance overhead |
| `InitDuration` | Cold start initialization time | Only on cold starts |

**Critical alarms to set:**
```
1. Errors / Invocations > 5% → Application errors
2. Throttles > 0 → Concurrency issue
3. Duration p99 > 80% of timeout → Risk of timeouts
4. IteratorAge > 60000ms (1 min) → Stream processor falling behind
5. ConcurrentExecutions > 80% of limit → Scale issue imminent
6. ProvisionedConcurrencySpilloverInvocations > 0 → Cold starts happening
```

**Tricky**: `Duration` metric does NOT include cold start time! Cold start is reported in `InitDuration` (only for cold starts). Total response time = InitDuration + Duration. So a function with 200ms Duration could actually take 2.2s including a 2s cold start.

---

## 7. ECS / Fargate Metrics

**Q16: How do ECS CloudWatch metrics work? Explain cluster, service, and task-level metrics.**

**A:**

```
Metrics hierarchy:
AWS/ECS namespace:
├── Cluster level: CPUUtilization, MemoryUtilization (average across all tasks)
├── Service level: CPUUtilization, MemoryUtilization (per service)
└── Container Insights (must enable): Per-task, per-container metrics

Dimensions:
├── ClusterName = "production"
├── ServiceName = "web-api"  
└── (Container Insights adds: TaskId, ContainerName)
```

| Metric | Level | Description | Alarm |
|--------|-------|-------------|-------|
| `CPUUtilization` | Service | % CPU used vs reserved | > 80% → scale out |
| `MemoryUtilization` | Service | % memory used vs reserved | > 80% → scale out |
| `RunningTaskCount` | Service | Active healthy tasks | < desired count |
| `PendingTaskCount` | Service | Tasks waiting to launch | > 0 for > 5min |
| `DesiredTaskCount` | Service | Target number of tasks | Track scaling |
| `CPUReservation` | Cluster | % CPU reserved vs available | > 80% → add capacity |
| `MemoryReservation` | Cluster | % memory reserved vs available | > 80% → add capacity |

**Container Insights metrics (detailed):**
| Metric | Description |
|--------|-------------|
| `CpuUtilized` | Actual CPU units used by container |
| `MemoryUtilized` | Actual memory (MB) used |
| `NetworkRxBytes` | Bytes received per container |
| `NetworkTxBytes` | Bytes sent per container |
| `StorageReadBytes` | Container disk reads |
| `StorageWriteBytes` | Container disk writes |
| `RunningTaskCount` | Per-service task count |

**Tricky**: ECS `CPUUtilization` is relative to CPU RESERVED, not total CPU available. If a task reserves 256 CPU units and uses 256, utilization = 100%. But the host may have 4096 CPU units available. For auto-scaling, use service-level `CPUUtilization` with Target Tracking.

---

## 8. S3 Metrics

**Q17: Explain S3 CloudWatch metrics. Which require request metrics (paid)?**

**A:**

**Free storage metrics (daily, bucket-level):**
| Metric | Description |
|--------|-------------|
| `BucketSizeBytes` | Total size of bucket (by storage class) |
| `NumberOfObjects` | Total object count |

**Request metrics (must enable per bucket/prefix, paid):**
| Metric | Description | Use Case |
|--------|-------------|----------|
| `AllRequests` | Total requests | Traffic baseline |
| `GetRequests` | GET request count | Read traffic |
| `PutRequests` | PUT request count | Write traffic |
| `DeleteRequests` | DELETE request count | Deletion patterns |
| `HeadRequests` | HEAD request count | Metadata lookups |
| `4xxErrors` | Client error responses | Permission issues |
| `5xxErrors` | Server error responses | S3 issues (rare) |
| `TotalRequestLatency` | Time from first byte to last | Performance |
| `FirstByteLatency` | Time to first byte | S3 response speed |
| `BytesDownloaded` | GET response bytes | Bandwidth cost |
| `BytesUploaded` | PUT request bytes | Ingestion rate |

**Tricky**: S3 storage metrics (`BucketSizeBytes`) are reported only ONCE PER DAY (delayed by 24-48 hours). You cannot use them for real-time monitoring. Request metrics are near-real-time (1-minute periods) but must be explicitly enabled and cost money.

---

## 9. DynamoDB Metrics

**Q18: Explain critical DynamoDB metrics, especially throttling.**

**A:**

| Metric | Description | Critical When |
|--------|-------------|---------------|
| `ConsumedReadCapacityUnits` | RCUs consumed | Near provisioned limit |
| `ConsumedWriteCapacityUnits` | WCUs consumed | Near provisioned limit |
| `ProvisionedReadCapacityUnits` | RCUs provisioned | N/A (reference) |
| `ProvisionedWriteCapacityUnits` | WCUs provisioned | N/A (reference) |
| `ReadThrottleEvents` | Reads rejected (exceeded capacity) | > 0 = DATA LOSS RISK |
| `WriteThrottleEvents` | Writes rejected | > 0 = DATA LOSS RISK |
| `ThrottledRequests` | Total throttled requests | > 0 |
| `SystemErrors` | DynamoDB internal errors | > 0 (rare, AWS issue) |
| `UserErrors` | Client-side errors (validation, conditional) | Check application logic |
| `SuccessfulRequestLatency` | Avg latency for successful ops | > 10ms investigate |
| `ConditionalCheckFailedRequests` | Failed condition expressions | Expected or not? |
| `ReturnedItemCount` | Items returned by Query/Scan | Too high = inefficient |
| `AccountMaxReads` / `AccountMaxWrites` | Account-level limits | Near limit = all tables affected |

**Throttling deep dive:**
```
Provisioned Mode:
├── 1 RCU = 1 strongly consistent read of item ≤ 4KB/second
├── 1 RCU = 2 eventually consistent reads of item ≤ 4KB/second
├── 1 WCU = 1 write of item ≤ 1KB/second
└── Exceed = ThrottleException (HTTP 400)

On-Demand Mode:
├── Adapts automatically
├── Can still throttle if traffic doubles within 30 minutes
└── Previous peak = base, 2× previous peak = instant capacity
```

**Hot partition problem:**
```
Table: 10,000 WCU provisioned
Partitions: 10 (each gets 1,000 WCU)

If ONE partition gets 3,000 WCU of writes:
├── That partition throttled at 1,000 WCU
├── Other 9 partitions idle
├── Overall table utilization: 30%
└── But you still get throttling!

Fix: Better partition key design (high cardinality)
Metric: Check per-partition metrics via Contributor Insights
```

**Tricky**: DynamoDB `ConsumedCapacityUnits` shows AVERAGE over 1 minute. A 1-second spike of 60,000 RCU averages to 1,000 RCU/min — looks fine in CloudWatch but causes throttling! Use `ReadThrottleEvents` metric (count, not average) to detect actual throttling events.

---

## 10. Custom Metrics & Alarms

**Q19: How do you create custom CloudWatch metrics? Explain PutMetricData, Embedded Metric Format, and Metric Filters.**

**A:**

**Method 1: PutMetricData API (from application):**
```python
import boto3
cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_metric_data(
    Namespace='MyApp/Production',
    MetricData=[{
        'MetricName': 'OrderProcessingTime',
        'Value': 1.5,
        'Unit': 'Seconds',
        'Dimensions': [
            {'Name': 'Service', 'Value': 'order-service'},
            {'Name': 'Environment', 'Value': 'production'}
        ],
        'Timestamp': datetime.utcnow(),
        'StorageResolution': 1  # High-resolution (1 second)
    }]
)
```

**Method 2: Embedded Metric Format (EMF) — print JSON to stdout:**
```json
{
  "_aws": {
    "Timestamp": 1234567890,
    "CloudWatchMetrics": [{
      "Namespace": "MyApp",
      "Dimensions": [["Service", "Environment"]],
      "Metrics": [
        {"Name": "RequestLatency", "Unit": "Milliseconds"},
        {"Name": "RequestCount", "Unit": "Count"}
      ]
    }]
  },
  "Service": "order-api",
  "Environment": "production",
  "RequestLatency": 125,
  "RequestCount": 1
}
```
Benefits: No SDK needed, works in Lambda (just print JSON), zero API calls, supports high-cardinality dimensions.

**Method 3: Metric Filters (from CloudWatch Logs):**
```
Log Group: /app/production
Filter Pattern: [timestamp, level="ERROR", ...]
Metric: ErrorCount (namespace: MyApp, value: 1 per match)

# More complex: Extract numeric value from log
Filter Pattern: { $.latency > 0 }
Metric Value: $.latency
```

**Tricky**: PutMetricData costs $0.01 per 1,000 metrics (standard) or $0.01 per 150 (high-resolution). If you publish 100 metrics every second from 100 instances = 864 million datapoints/day = $5,760/day! Use EMF (included in Logs pricing) or batch PutMetricData calls (up to 1,000 values per call).

---

## 11. Tricky Scenario-Based Questions

**Q20: Your RDS instance shows FreeableMemory decreasing daily but SwapUsage is still 0 and performance is fine. Should you be concerned?**

**A:**

**Not necessarily!** This is often normal behavior:

```
What's happening:
├── Database is warming up its buffer pool (InnoDB/PostgreSQL shared buffers)
├── OS is using free memory for filesystem caching
├── Both are GOOD — data in memory = faster queries
└── Memory is being USED effectively, not wasted

When to worry:
├── FreeableMemory < 5% of total AND SwapUsage > 0
├── FreeableMemory dropping AND response times increasing
├── FreeableMemory dropping AND DatabaseConnections growing
└── Trend shows it will hit 0 within days (extrapolate)

When NOT to worry:
├── FreeableMemory stabilizes (buffer pool fully warm)
├── SwapUsage = 0 (no memory pressure)
├── Performance metrics (latency, throughput) stable
└── FreeableMemory = OS cache (reclaimable if needed)
```

**Best practice alarm:**
```
DON'T alarm on: FreeableMemory < X alone
DO alarm on:    FreeableMemory < X AND SwapUsage > 0
OR:             FreeableMemory rate of change < -100MB/hour sustained
```

---

**Q21: Production shows intermittent Lambda throttling (Throttles > 0) but ConcurrentExecutions is well below the account limit. What's happening?**

**A:**

**Possible causes:**

1. **Reserved concurrency on OTHER functions is stealing capacity:**
```
Account limit: 1,000
Function A reserved: 400
Function B reserved: 300
Unreserved pool: 300 (for ALL other functions!)
Your function uses unreserved pool → only 300 available, not 1,000
```

2. **Burst limit reached:**
```
Initial burst: 3,000 (or 500-1000 depending on region)
After burst: +500 concurrent per minute scaling
If traffic spikes instantly to 2,000 → first 3,000 allowed, then rate-limited
```

3. **Per-function reserved concurrency:**
```
Your function has reserved concurrency = 50
Even if account has 950 free, your function is capped at 50
```

4. **Provisioned concurrency spillover:**
```
Provisioned: 100 instances
Traffic: 150 concurrent → 50 must use on-demand
On-demand pool (unreserved) is exhausted → throttle
```

**Debugging:**
```bash
# Check account-level
aws lambda get-account-settings
# Shows: ConcurrentExecutions limit, UnreservedConcurrentExecutions

# Check function-level
aws lambda get-function-concurrency --function-name my-func
# Shows reserved concurrency (if set)

# CloudWatch: Check WHICH function is actually using the capacity
# Metric: ConcurrentExecutions with FunctionName dimension
```

---

**Q22: Your ElastiCache CacheHits suddenly dropped to 0 and CacheMisses spiked. What happened?**

**A:**

**Immediate possibilities:**

1. **Cache node restarted/failover occurred:**
```
Check: ReplicationLag spike, then reset
Check: ElastiCache Events (console or API)
Result: Cold cache (all keys gone)
Fix: Wait for cache to warm, or implement cache warming
```

2. **Application deployment changed key format:**
```
Before: cache.get("user:123")
After:  cache.get("user_123")  ← Different key! All misses!
Fix: Rollback or migrate keys
```

3. **MaxMemory reached + noeviction policy:**
```
Cache full → new writes fail → application falls back to DB for everything
Check: Evictions metric, BytesUsedForCache near max
Fix: Clear old keys, increase cache size
```

4. **Network issue between app and cache:**
```
App can't reach cache → all operations timeout → counts as miss
Check: CurrConnections (dropped to 0?), New connection errors
Fix: Check security groups, VPC networking
```

5. **TTL mass-expiration:**
```
All keys set with same TTL → all expire simultaneously
Check: CurrItems dropped sharply at same time
Fix: Add random jitter to TTL (TTL = base + random(0, 60))
```

**Investigation priority:**
```bash
# 1. Check cache health
aws elasticache describe-cache-clusters --cache-cluster-id prod-redis-001

# 2. Check events (failover, maintenance)
aws elasticache describe-events --duration 60

# 3. Check connection metrics
# CurrConnections, NewConnections - did they drop?

# 4. Check memory metrics
# BytesUsedForCache, Evictions, CurrItems
```

---

**Q23: ALB shows increasing `TargetResponseTime` but backend EC2 `CPUUtilization` is only 30%. What's the bottleneck?**

**A:**

**CPU is NOT the bottleneck. Check these in order:**

1. **Memory pressure (swap thrashing):**
```
Check: CloudWatch Agent → mem_used_percent > 90%
Check: swap_used_percent > 0
Result: Application paging to disk = slow responses
Fix: Increase instance size or fix memory leak
```

2. **Database bottleneck (most common!):**
```
Check: RDS ReadLatency / WriteLatency increasing
Check: RDS DiskQueueDepth > 5
Check: RDS DatabaseConnections near max
Result: App waiting on DB queries
Fix: Optimize queries, add read replicas, use cache
```

3. **Disk I/O bottleneck:**
```
Check: EC2 EBSWriteLatency or CloudWatch Agent diskio
Check: EBS BurstBalance = 0 (gp2 throttled!)
Result: App doing disk I/O, waiting on slow EBS
Fix: Upgrade to gp3/io2, or increase volume size
```

4. **Network bandwidth saturation:**
```
Check: NetworkOut approaching instance limit
Example: t3.medium = 5 Gbps max, but up to 10 Gbps burst
Result: Network packets queuing
Fix: Larger instance type (more bandwidth)
```

5. **Thread/connection pool exhaustion:**
```
Check: App metrics - thread pool utilization, connection pool wait time
Result: All workers busy, new requests queue
Fix: Increase thread pool, add instances (scale out)
```

6. **External dependency slow:**
```
Check: X-Ray traces, app logs for external API call times
Result: App waiting on 3rd party API
Fix: Circuit breaker, caching, async processing
```

**Key insight**: Low CPU + High latency = I/O bound workload (waiting on disk, network, DB, or external service). The application spends time WAITING, not computing.

---

**Q24: Design a complete CloudWatch monitoring strategy for a production 3-tier application (ALB → EC2 → RDS).**

**A:**

```
┌─────────────────────────────────────────────────────────────┐
│                 MONITORING DASHBOARD                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  LAYER 1: ALB (User-Facing)                                 │
│  ├── RequestCount (traffic volume)                           │
│  ├── HTTPCode_ELB_5XX (ALB errors) → Alarm > 10/min        │
│  ├── HTTPCode_Target_5XX (app errors) → Alarm > 50/min     │
│  ├── TargetResponseTime p99 → Alarm > 2 seconds            │
│  ├── HealthyHostCount → Alarm < desired count              │
│  └── UnHealthyHostCount → Alarm > 0                        │
│                                                              │
│  LAYER 2: EC2 / Application                                 │
│  ├── CPUUtilization → Alarm > 80% for 5 min                │
│  ├── CPUCreditBalance (T instances) → Alarm < 50           │
│  ├── mem_used_percent (Agent) → Alarm > 85%                │
│  ├── disk_used_percent (Agent) → Alarm > 80%               │
│  ├── StatusCheckFailed → Alarm = 1                          │
│  ├── NetworkOut → Alarm > 80% instance limit               │
│  └── Custom: ActiveConnections, QueueDepth, ErrorRate       │
│                                                              │
│  LAYER 3: RDS Database                                       │
│  ├── FreeableMemory → Alarm < 500MB                         │
│  ├── SwapUsage → Alarm > 0                                  │
│  ├── CPUUtilization → Alarm > 80%                           │
│  ├── DatabaseConnections → Alarm > 80% of max              │
│  ├── ReadLatency → Alarm > 10ms                             │
│  ├── WriteLatency → Alarm > 20ms                            │
│  ├── FreeStorageSpace → Alarm < 10GB                        │
│  ├── DiskQueueDepth → Alarm > 10                            │
│  ├── BurstBalance (gp2) → Alarm < 20%                      │
│  └── ReplicaLag → Alarm > 30 seconds                        │
│                                                              │
│  LAYER 4: ElastiCache (if applicable)                        │
│  ├── CacheHitRate → Alarm < 80%                             │
│  ├── Evictions → Alarm > 100/min                            │
│  ├── EngineCPUUtilization → Alarm > 80%                     │
│  ├── FreeableMemory → Alarm < 500MB                         │
│  ├── CurrConnections → Alarm > 80% maxclients              │
│  └── ReplicationLag → Alarm > 1 second                      │
│                                                              │
│  COMPOSITE ALARMS:                                           │
│  ├── "Service Degraded" = ALB_5xx OR TargetResponseTime_High │
│  ├── "Database Critical" = LowMemory AND SwapUsage          │
│  └── "Full Outage" = UnhealthyHosts AND ELB_5xx > 100      │
│                                                              │
│  ACTIONS:                                                     │
│  ├── Warning → Slack notification                            │
│  ├── Critical → PagerDuty (page on-call)                    │
│  ├── Auto Scaling → Scale out when CPU > 70%                │
│  └── Runbook links in alarm descriptions                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

**Q25: What is the difference between CloudWatch Agent metrics and default EC2 metrics? Name 5 metrics you can ONLY get with the agent.**

**A:**

**Default EC2 metrics** (hypervisor-level, no agent needed):
- CPUUtilization, NetworkIn/Out, DiskReadOps, StatusChecks

**Agent-only metrics** (OS-level, requires installation):

| # | Metric | Why Agent Needed |
|---|--------|-----------------|
| 1 | `mem_used_percent` | Hypervisor can't see inside guest OS memory |
| 2 | `disk_used_percent` | Hypervisor doesn't know filesystem layout |
| 3 | `swap_used_percent` | OS-level swap management |
| 4 | `cpu_usage_iowait` | Granular CPU state breakdown |
| 5 | `netstat_tcp_established` | OS network stack connections |
| 6 | `processes_running` | OS process table |
| 7 | `cpu_usage_steal` | VM CPU stolen by hypervisor |
| 8 | Custom application metrics (StatsD/collectd) | Application-specific |

**Agent installation:**
```bash
# Install on Amazon Linux 2
sudo yum install amazon-cloudwatch-agent

# Configure
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Or use JSON config
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s -c ssm:AmazonCloudWatch-linux-config
```

**Agent config example:**
```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {"measurement": ["mem_used_percent"]},
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/", "/data"]
      },
      "swap": {"measurement": ["swap_used_percent"]},
      "cpu": {
        "measurement": ["cpu_usage_iowait", "cpu_usage_steal"],
        "totalcpu": true
      },
      "netstat": {
        "measurement": ["tcp_established", "tcp_time_wait"]
      }
    }
  }
}
```

**Tricky**: `cpu_usage_steal` is critical for T-instances. High steal % means the hypervisor is taking CPU away from your instance (burst credits exhausted, noisy neighbor on bare metal). This is INVISIBLE without the agent!



---

## Additional Scenario-Based Tricky Questions

---

**Q26: Your CloudWatch alarm for high CPU triggers at 2 AM every night (alarm state for 5 minutes, then resolves). Investigation shows no actual performance issues during that time. Users don't notice anything. How do you fix this false alarm without missing real issues?**

**A:**

**Why the alarm triggers but nothing is wrong:**
```
1. SCHEDULED BATCH JOB:
   - Cron job at 2 AM: backup, log rotation, metrics aggregation
   - Spikes CPU to 90% for 3 minutes
   - Alarm threshold: CPU > 80% for 3/5 datapoints (1-min periods)
   - 3 consecutive points above 80% → ALARM!
   - Job finishes → CPU drops → OK state
   - This is EXPECTED behavior, not a problem

2. METRIC MATH AVERAGING CONFUSION:
   - CPU metric reported every 5 minutes (basic monitoring)
   - Alarm evaluates 1-minute periods (but data is 5-min resolution)
   - CloudWatch backfills: one 5-min datapoint treated as 5 identical 1-min points
   - A single 85% reading becomes 5 datapoints of 85% → exceeds threshold

3. STEAL TIME on shared instances (t-series burstable):
   - Other tenants using physical CPU → your instance gets "steal" time
   - Shows as CPU utilization even though YOUR app isn't busy
   - Fix: Switch to dedicated/non-burstable instance types
```

**Fix strategies:**
```bash
# Strategy 1: Exclude known batch windows using Metric Math
# Create alarm that ignores 2-3 AM window:
aws cloudwatch put-metric-alarm --alarm-name "CPU-High-Production" \
  --metrics '[
    {"Id":"cpu","MetricStat":{"Metric":{"Namespace":"AWS/EC2","MetricName":"CPUUtilization","Dimensions":[{"Name":"InstanceId","Value":"i-xxx"}]},"Period":300,"Stat":"Average"}},
    {"Id":"hour","Expression":"HOUR(cpu)"},
    {"Id":"filtered","Expression":"IF(hour >= 2 AND hour < 3, 0, cpu)"}
  ]' \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 --datapoints-to-alarm 3 \
  --treat-missing-data notBreaching

# Strategy 2: Use composite alarms (require multiple signals)
aws cloudwatch put-composite-alarm --alarm-name "Real-CPU-Issue" \
  --alarm-rule 'ALARM("CPU-High") AND ALARM("Response-Latency-High")'
# Only fires if BOTH CPU is high AND latency is affected
# Batch job = high CPU but normal latency → doesn't fire

# Strategy 3: Increase evaluation period
# Instead of 3/5 datapoints at 1-min:
# Use 3/5 datapoints at 5-min periods = must be sustained 15+ minutes
--period 300 --evaluation-periods 5 --datapoints-to-alarm 3
# 5-min batch job won't sustain alarm across 3 five-minute periods
```

**Tricky**: CloudWatch alarms with `treat-missing-data: missing` (default) will stay in ALARM state if data stops flowing (instance dies). Use `treat-missing-data: notBreaching` for most alarms so that missing data = assume OK. But for availability alarms (instance health), use `treat-missing-data: breaching` so missing data = assume problem. The wrong choice here either causes false alarms (instance reboot = alarm) or missed outages (instance dead = no alarm).

---

**Q27: Your custom CloudWatch metrics are delayed by 5-10 minutes. By the time an alarm fires, the issue has been ongoing for 10+ minutes. How do you get near-real-time alerting?**

**A:**

**Understanding metric delays:**
```
Where delays happen:
1. Application emits metric → CloudWatch Agent buffer (10-60s)
2. Agent sends to CloudWatch API → API processing (0-2s)
3. Metric visible in CloudWatch (0-60s)
4. Alarm evaluation (evaluates every "Period" seconds)
5. Alarm state change → SNS notification (1-5s)

Total: 1-5 minutes minimum with standard settings
```

**Reducing to near-real-time (<60s):**
```bash
# Fix 1: CloudWatch Agent flush interval
# /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json:
{
  "metrics": {
    "metrics_collected": {
      "statsd": {
        "metrics_aggregation_interval": 10  # Aggregate every 10s (not 60s)
      }
    },
    "force_flush_interval": 5  # Flush to CloudWatch every 5 seconds!
  }
}

# Fix 2: Use high-resolution metrics (1-second resolution)
aws cloudwatch put-metric-data --namespace "MyApp" \
  --metric-data '[{
    "MetricName": "RequestLatency",
    "Value": 250,
    "Unit": "Milliseconds",
    "StorageResolution": 1
  }]'
# StorageResolution=1 → data stored at 1-second granularity
# Default is 60 seconds

# Fix 3: Set alarm period to minimum
aws cloudwatch put-metric-alarm --alarm-name "LatencyHigh" \
  --period 10 \                       # Evaluate every 10 seconds!
  --evaluation-periods 3 \            # 3 consecutive breaches
  --datapoints-to-alarm 3 \           # All 3 must breach
  --treat-missing-data notBreaching
# Total detection time: 30 seconds (3 × 10s)

# Fix 4: Use Embedded Metric Format (EMF) for instant metrics
# Application logs in EMF → CloudWatch Logs → automatic metric extraction
# No agent buffering delay!
console.log(JSON.stringify({
  "_aws": {
    "Timestamp": Date.now(),
    "CloudWatchMetrics": [{
      "Namespace": "MyApp",
      "Dimensions": [["Service", "Endpoint"]],
      "Metrics": [{"Name": "Latency", "Unit": "Milliseconds"}]
    }]
  },
  "Service": "payment-api",
  "Endpoint": "/charge",
  "Latency": 250
}));
```

**Alternative: Skip CloudWatch for ultra-fast alerting:**
```bash
# For sub-10-second alerting, use streaming:
# App → CloudWatch Logs → Subscription Filter → Lambda → SNS/PagerDuty

# Or: App → Kinesis Data Stream → Lambda (processes in real-time)
# Detection in <5 seconds

# Prometheus + Alertmanager (Kubernetes):
# scrape_interval: 10s + evaluation_interval: 10s = 20s to detect
# Much faster than CloudWatch for K8s workloads
```

**Tricky**: High-resolution metrics (1-second) cost MORE: $0.30 per metric per month vs $0.30 per metric per month for standard. But the real cost driver is NUMBER of metrics, not resolution. Also, high-resolution metrics are only stored at 1-second granularity for 3 hours, then aggregated to 1-minute for 15 days, then 5-minute for 63 days. You can't query 1-second data from last week — it's already aggregated.

---

**Q28: Your CloudWatch dashboard shows CPU at 45% average but your application team reports "the server is at 100% CPU." Both are correct. How is this possible and who should you believe?**

**A:**

**How both can be true:**
```
Scenario 1: MULTI-CORE AVERAGING
- Instance has 8 CPUs
- 4 CPUs at 100%, 4 CPUs at 0%
- CloudWatch reports AVERAGE: (4×100 + 4×0) / 8 = 50%
- Application (single-threaded) sees 100% on its core
- Top shows: %Cpu0: 100%, %Cpu1: 100%, %Cpu2: 100%, %Cpu3: 100%
            %Cpu4: 0%, %Cpu5: 0%, %Cpu6: 0%, %Cpu7: 0%
- Fix: Use per-core metrics or look at application-specific CPU

Scenario 2: TIME AVERAGING
- CPU spikes to 100% for 30 seconds, then 0% for 30 seconds
- CloudWatch 1-minute average: (100+0)/2 = 50%
- User experiences: 50% of requests hit during 100% spike → timeout
- Fix: Use Maximum statistic instead of Average

Scenario 3: CONTAINER vs HOST METRICS
- CloudWatch reports HOST CPU: 45% (of m5.2xlarge = 8 vCPU)
- Container has 1 vCPU limit → container is at 100% of its limit
- Host has plenty of capacity, but container is throttled
- Fix: Monitor container metrics (CAdvisor, CloudWatch Container Insights)

Scenario 4: STEAL TIME hidden in CloudWatch
- CloudWatch CPU includes ALL usage (user+system+nice+steal)
- Server at 45% CloudWatch = 35% app + 10% steal
- From app perspective: it's getting throttled 10% of the time
- Top shows: %st: 10% (burstable instance credits depleted)
```

**What to actually monitor:**
```bash
# CloudWatch: good for host-level, bad for application-level
# For accurate application CPU:

# Option 1: CloudWatch Agent custom metrics (per-process)
{
  "metrics": {
    "metrics_collected": {
      "procstat": [{
        "pattern": "java.*myapp",
        "measurement": ["cpu_usage", "memory_rss", "num_threads"]
      }]
    }
  }
}

# Option 2: Application-level metrics (most accurate)
# Expose /metrics endpoint with:
# - process_cpu_seconds_total (Prometheus format)
# - Per-endpoint request processing time
# - Thread pool utilization

# Option 3: Container Insights (for ECS/EKS)
# Shows per-container CPU, memory, network
# This is what the application actually experiences
```

**Tricky**: CloudWatch EC2 CPUUtilization metric reports the AVERAGE across all vCPUs at 5-minute (basic) or 1-minute (detailed) intervals. A single-threaded application on an 8-core instance can be completely CPU-bound (100% on one core) while CloudWatch shows 12.5%. For accurate monitoring: use per-CPU metrics from CloudWatch Agent, use the `Maximum` statistic (catches spikes), and always correlate with application-level latency metrics.

---
