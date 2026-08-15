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

## 12. Advanced Scenario-Based & Tricky Questions

**Q26: SCENARIO — Your Auto Scaling Group keeps launching and terminating instances every 5 minutes. CloudWatch shows CPUUtilization oscillating between 20% and 75%. What's happening and how do you fix it?**

**A:**

**This is called "thrashing" or "flapping" — a classic scaling oscillation problem.**

```
Timeline:
00:00 - CPU at 75% → Scale-out alarm fires → Launch 2 instances
00:03 - New instances start receiving traffic
00:05 - CPU drops to 20% (more instances = less load per instance)
00:07 - Scale-in alarm fires (CPU < 30%) → Terminate 2 instances
00:09 - CPU rises back to 75% (fewer instances = more load per instance)
00:10 - Scale-out fires again → REPEAT FOREVER
```

**Root causes:**

1. **Cooldown period too short:**
```
Default cooldown: 300 seconds
Your cooldown: 60 seconds → too short to stabilize
Fix: Increase cooldown to 300-600 seconds
```

2. **Scale-out and scale-in thresholds too close:**
```
Scale-out: CPU > 70%
Scale-in:  CPU < 30%
Gap is only 40% → natural oscillation crosses both

Fix: Widen the gap
Scale-out: CPU > 70%  (alarm: 2/3 periods breaching)
Scale-in:  CPU < 40%  (alarm: 10/10 periods breaching)
Make scale-in MUCH harder to trigger than scale-out
```

3. **Target Tracking policy with wrong target:**
```
Target: CPUUtilization = 50%
Instance startup time: 3 minutes
ASG overcompensates → launches too many → CPU drops → terminates

Fix: Use Target Tracking with longer warmup:
{
  "TargetValue": 60.0,
  "EstimatedInstanceWarmup": 300  ← Key! Wait for instances to warm
}
```

4. **Step scaling without proper step design:**
```
Bad: Add 5 instances when CPU > 70%, Remove 5 when CPU < 30%
Good: Add 2 when 70-80%, Add 4 when 80-90%, Add 8 when > 90%
      Remove 1 when 40-30%, Remove 2 when < 30%
Scale IN slowly, scale OUT aggressively
```

**Best practice configuration:**
```json
{
  "ScaleOutPolicy": {
    "Threshold": 70,
    "EvaluationPeriods": 2,
    "Period": 60,
    "Cooldown": 120
  },
  "ScaleInPolicy": {
    "Threshold": 35,
    "EvaluationPeriods": 15,
    "Period": 60,
    "Cooldown": 600
  }
}
```

**Tricky**: The `EstimatedInstanceWarmup` in Target Tracking is CRITICAL. Without it, ASG sees new instances reporting 0% CPU (still booting) → average CPU drops → ASG thinks it over-scaled → terminates instances before they even start serving traffic!

---


**Q27: SCENARIO — RDS `ReadLatency` suddenly spiked from 2ms to 200ms. `CPUUtilization` is only 40%, `FreeableMemory` is stable at 4GB. What's wrong?**

**A:**

**Key insight: CPU and memory are fine, but reads are 100x slower. This points to STORAGE.**

**Investigation order:**

1. **Check `DiskQueueDepth`:**
```
DiskQueueDepth > 10 → Storage is overloaded
I/O requests are queuing up → each read waits longer
```

2. **Check `BurstBalance` (gp2 only):**
```
BurstBalance = 0% → Your EBS volume has NO burst credits left!
Volume is throttled to baseline IOPS
100GB gp2 baseline = 300 IOPS → Trying to do 3000 → queuing → 200ms latency

FIX: Increase volume to 1TB+ gp2 (baseline = 3000 IOPS)
     OR migrate to gp3 (3000 IOPS baseline always)
     OR migrate to io2 (provision exact IOPS needed)
```

3. **Check `ReadIOPS` vs provisioned IOPS:**
```
If ReadIOPS hitting ceiling → storage maxed out
For io1/io2: Check if actual IOPS = provisioned IOPS (hitting limit)
```

4. **Check for vacuum/analyze running (PostgreSQL):**
```
PostgreSQL auto-vacuum can cause I/O storms
Check: pg_stat_progress_vacuum
Symptom: Random I/O spike during low-traffic hours
Fix: Tune vacuum settings, schedule during maintenance window
```

5. **Check for backup/snapshot in progress:**
```
RDS automated backups cause additional I/O
Check: RDS Events → "Backing up DB instance"
Snapshot reads entire volume → competes with production I/O
Symptom: Daily spike at backup time
Fix: Adjust backup window to lowest traffic period
```

6. **Check for buffer pool miss storm:**
```
If a large table scan happened (SELECT without index):
├── Reads bypass buffer pool → goes to disk
├── Flushes useful pages from buffer pool
├── Subsequent normal queries also miss cache → disk reads
└── "Cascading cache miss"
Fix: Add missing indexes, check slow query log
```

**Complete diagnosis command:**
```bash
# Get multiple metrics at once
aws cloudwatch get-metric-data --metric-data-queries '[
  {"Id":"latency","MetricStat":{"Metric":{"Namespace":"AWS/RDS","MetricName":"ReadLatency","Dimensions":[{"Name":"DBInstanceIdentifier","Value":"prod-db"}]},"Period":60,"Stat":"Average"}},
  {"Id":"iops","MetricStat":{"Metric":{"Namespace":"AWS/RDS","MetricName":"ReadIOPS","Dimensions":[{"Name":"DBInstanceIdentifier","Value":"prod-db"}]},"Period":60,"Stat":"Average"}},
  {"Id":"queue","MetricStat":{"Metric":{"Namespace":"AWS/RDS","MetricName":"DiskQueueDepth","Dimensions":[{"Name":"DBInstanceIdentifier","Value":"prod-db"}]},"Period":60,"Stat":"Average"}},
  {"Id":"burst","MetricStat":{"Metric":{"Namespace":"AWS/RDS","MetricName":"BurstBalance","Dimensions":[{"Name":"DBInstanceIdentifier","Value":"prod-db"}]},"Period":60,"Stat":"Average"}}
]' --start-time 2024-01-01T00:00:00 --end-time 2024-01-01T01:00:00
```

**Tricky**: `ReadLatency` in RDS is per-OPERATION average. Even a few extremely slow reads (full table scan) average UP the metric significantly. Check `ReadIOPS` too — if IOPS is normal but latency is high, individual I/O operations are slow (storage layer problem, not volume of reads).

---


**Q28: SCENARIO — Lambda function shows 0 `Errors` but users report receiving wrong data. How do you detect "silent failures" with CloudWatch?**

**A:**

**The most dangerous type of bug — function succeeds (no error) but returns incorrect data.**

**Why standard metrics miss this:**
```
Lambda Metrics:
├── Invocations: 10,000 ✓
├── Errors: 0 ✓ (no exceptions thrown)
├── Duration: 200ms ✓ (normal speed)
├── Throttles: 0 ✓
└── Everything looks PERFECT... but data is wrong!
```

**Detection strategies:**

1. **Custom business metrics (Embedded Metric Format):**
```python
import json

def handler(event, context):
    result = process_order(event)
    
    # Emit business metric alongside response
    print(json.dumps({
        "_aws": {
            "Timestamp": int(time.time() * 1000),
            "CloudWatchMetrics": [{
                "Namespace": "MyApp/Orders",
                "Dimensions": [["Status"]],
                "Metrics": [{"Name": "OrderResult", "Unit": "Count"}]
            }]
        },
        "Status": result.status,  # "success", "empty_response", "stale_data"
        "OrderResult": 1
    }))
    
    # Validate response before returning
    if not result.items or len(result.items) == 0:
        # Emit "empty response" metric
        print(json.dumps({...EmptyResponseMetric...}))
    
    return {"statusCode": 200, "body": json.dumps(result)}
```

2. **Response validation with custom metric:**
```python
# After processing, validate the response makes sense
def validate_response(response, request):
    if response.user_id != request.user_id:
        cloudwatch.put_metric_data(
            Namespace='MyApp/Integrity',
            MetricData=[{'MetricName': 'DataMismatch', 'Value': 1}]
        )
    if response.timestamp < (time.time() - 3600):
        cloudwatch.put_metric_data(
            Namespace='MyApp/Integrity',
            MetricData=[{'MetricName': 'StaleData', 'Value': 1}]
        )
```

3. **Canary/Synthetic monitoring:**
```
CloudWatch Synthetics Canary:
├── Runs every 5 minutes
├── Calls API with KNOWN input
├── Validates response matches EXPECTED output
├── Emits SuccessPercent metric
└── Alarm when SuccessPercent < 100%
```

4. **Cross-service comparison metrics:**
```
Compare: Orders Created (API) vs Orders in Database vs Orders in SQS
If API says 100 orders created but DB only has 95 → 5 lost!
Metric: "OrderDiscrepancy" = API_count - DB_count
Alarm: OrderDiscrepancy > 0
```

5. **Log-based anomaly detection:**
```
CloudWatch Logs Insights:
fields @timestamp, response_body
| filter response_body like /null/ or response_body like /\[\]/
| stats count() as empty_responses by bin(5m)
→ Create metric filter: count of empty/null responses
→ Alarm when empty_responses > baseline × 2
```

**Tricky**: The deadliest production bugs have NO error signals. You MUST instrument business-level metrics: "orders processed correctly", "cache responses validated", "data freshness confirmed". Infrastructure metrics alone cannot detect logic bugs.

---


**Q29: SCENARIO — ElastiCache `EngineCPUUtilization` is at 95% but `CPUUtilization` shows only 25%. The team says "CPU is fine, only 25%." Why are they wrong?**

**A:**

**This is one of the most common ElastiCache misunderstandings!**

```
Redis Architecture:
├── SINGLE-THREADED command processing (1 core handles ALL commands)
├── Background threads: Lazy freeing, I/O threads (Redis 6+), RDB save
└── Multiple CPU cores available but main thread uses ONLY ONE

Metrics explained:
├── CPUUtilization = Average across ALL CPU cores
│   Example: 4-core node, 1 core at 100%, 3 at 0% → shows 25%
│   
└── EngineCPUUtilization = Redis main thread ONLY
    Example: Main thread at 100% → shows 100% (actual bottleneck!)
```

**Visual:**
```
4-Core Redis Node:
Core 0: ████████████ 100% ← Redis engine (all commands)
Core 1: ░░░░░░░░░░░░   0% ← Idle
Core 2: ██░░░░░░░░░░  15% ← Background (RDB save, lazy free)
Core 3: █░░░░░░░░░░░  10% ← I/O threads

CPUUtilization = (100+0+15+10)/4 = 31.25%  ← "Looks fine!"
EngineCPUUtilization = 100%  ← REDIS IS MAXED OUT!
```

**Impact when EngineCPUUtilization = 95%+:**
- Command latency increases exponentially
- Timeouts for client operations
- Pub/Sub messages delayed
- Cluster operations slow (if cluster mode)
- Looks like network issue to application (but it's CPU!)

**Solutions:**

1. **Read replicas** (for read-heavy workloads):
```
Primary: Handles writes + some reads
Replicas: Handle read traffic (offload primary CPU)
Configure app: Read from replicas, write to primary
```

2. **Scale OUT with cluster mode (sharding):**
```
Shard 1: Keys A-M → own CPU
Shard 2: Keys N-Z → own CPU
Each shard handles less traffic → lower CPU per node
```

3. **Optimize expensive commands:**
```
# Find slow commands
redis-cli slowlog get 10

# Common CPU killers:
KEYS * → O(N) scans entire keyspace! Use SCAN instead
SORT with large lists → CPU intensive
Lua scripts with loops → blocks everything
Large HGETALL on huge hashes → serialize + send

# Fix: Replace KEYS with SCAN, paginate large operations
```

4. **Larger node type** (more powerful single core):
```
cache.r6g.large → cache.r6g.xlarge (faster CPU clock)
More GHz per core = more commands per second
```

**Alarm:**
```
ALWAYS alarm on EngineCPUUtilization, NOT CPUUtilization for Redis!
Threshold: EngineCPUUtilization > 65% → Warning
           EngineCPUUtilization > 80% → Critical
```

**Tricky**: Redis 7.0+ has multi-threaded I/O (io-threads config) which can help with network-bound workloads. But the COMMAND PROCESSING is still single-threaded. Multi-threaded I/O helps with serialization/deserialization, not with actual command execution.

---


**Q30: SCENARIO — Your CloudWatch alarm triggered (CPU > 80% for 3 periods) but when you checked, CPU was at 20%. Was it a false alarm? How do you investigate?**

**A:**

**This is NOT necessarily a false alarm. Common reasons:**

1. **Alarm already resolved by the time you looked:**
```
Timeline:
T+0min: CPU spikes to 85% (alarm evaluating...)
T+5min: CPU still 85% (2nd period breaching)
T+10min: CPU hits 90% (3rd period → ALARM triggers → SNS notification)
T+12min: Auto-scaling adds instances → CPU drops to 40%
T+15min: You check CloudWatch dashboard → shows 20% (already resolved!)

Investigation: Look at the GRAPH, not just current value
Fix: Always include time range in alarm notification
```

2. **Statistics mismatch — you're looking at the wrong statistic:**
```
Alarm configured on: Maximum CPUUtilization > 80%
Dashboard showing: Average CPUUtilization = 20%

Both are correct! Maximum of 85% triggered alarm,
but Average over the period is only 20% (spike was brief)

Fix: Align dashboard widgets with alarm statistics
```

3. **Dimension mismatch — alarm on one instance, dashboard on all:**
```
Alarm: CPUUtilization for instance i-abc123 > 80% → FIRED
Dashboard: CPUUtilization AVERAGE across ALL instances = 20%

One instance was at 85%, others at 15% → average is low
Fix: Dashboard should show per-instance breakdown
```

4. **Metric math alarm vs raw metric:**
```
Alarm on: (CPUUtilization + IOWaitPercent) > 80%
You check: CPUUtilization alone = 20%
But: IOWaitPercent was 65% → Combined = 85% → alarm valid!
```

**How to investigate retroactively:**
```bash
# Check alarm state history
aws cloudwatch describe-alarm-history \
  --alarm-name "HighCPU-prod" \
  --history-item-type StateUpdate \
  --max-records 10

# Get the actual data points that triggered the alarm
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-abc123 \
  --start-time "2024-01-01T10:00:00Z" \
  --end-time "2024-01-01T11:00:00Z" \
  --period 60 \
  --statistics Maximum Average

# Check if auto-scaling acted
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name prod-asg \
  --max-records 5
```

**Best practices for alarm investigation:**
```
1. ALWAYS include metric graph snapshot in alarm notification
2. Use CloudWatch Alarm annotation (shows when alarm fired on graph)
3. Set up dashboard with auto-refresh during incidents
4. Use Composite Alarms to reduce false positives
5. Log alarm state changes to CloudWatch Logs for audit trail
```

**Tricky**: CloudWatch evaluates alarms using data points AT THE TIME of evaluation. If a metric is reported with a delay (some custom metrics), the alarm may fire "late" — when you check, the real issue was 5-10 minutes ago. Always look at the historical graph, not the current value.

---


**Q31: SCENARIO — DynamoDB shows `ReadThrottleEvents` > 0 but `ConsumedReadCapacityUnits` is well below `ProvisionedReadCapacityUnits`. How is throttling happening below capacity?**

**A:**

**This is the famous "hot partition" problem — one of the trickiest DynamoDB interview questions!**

```
Table configuration:
├── Provisioned: 10,000 RCU
├── Consumed (average): 3,000 RCU (only 30% utilized!)
├── Partitions: 10 (AWS manages internally)
├── Per-partition capacity: ~1,000 RCU each
└── BUT: One partition receiving 3,000 RCU → THROTTLED!

Visual:
Partition 1: ████████████████████ 3,000 RCU (THROTTLED at 1,000!)
Partition 2: ██                   200 RCU
Partition 3: ███                  300 RCU
Partition 4: █                    100 RCU
Partition 5-10: █                 100 RCU each
Total: 3,000 RCU consumed / 10,000 provisioned = 30% utilization
BUT: Partition 1 is throttled!
```

**Why this happens:**
1. **Poor partition key design:**
```
Bad keys: "status" (only 3 values: pending/active/completed)
         "date" (today's date → all traffic hits ONE partition)
         "country" (90% of users in one country)

Good keys: user_id, order_id, UUID (high cardinality, uniform distribution)
```

2. **Popular item ("celebrity problem"):**
```
One item accessed 10,000 times/second (viral product, trending user)
That item's partition key → single partition → throttled
Fix: Write-sharding (user#1, user#2, user#3) + scatter-gather reads
     OR use DAX cache in front (absorbs hot key reads)
```

3. **Adaptive capacity (helps but doesn't eliminate):**
```
DynamoDB adaptive capacity:
├── Automatically reallocates unused capacity to hot partitions
├── Can BOOST a partition up to total table provisioned capacity
├── Takes 5-30 minutes to detect and adapt
└── Doesn't help for instant spikes (first few minutes still throttle)
```

**Detection:**
```bash
# Enable Contributor Insights for DynamoDB
aws dynamodb update-contributor-insights \
  --table-name MyTable \
  --contributor-insights-action ENABLE

# This shows TOP partition keys by consumption
# Identify which keys are "hot"

# CloudWatch metric with partition-level detail:
# Use Contributor Insights → shows most accessed keys
```

**Solutions by severity:**

| Approach | Effort | Effectiveness |
|----------|--------|---------------|
| Switch to On-Demand mode | Low | Good (auto-adapts, still has per-partition limits) |
| Add DAX cache for hot reads | Medium | Excellent (absorbs hot key traffic) |
| Write-sharding (add random suffix to PK) | Medium | Excellent (distributes load) |
| Redesign partition key | High | Best (fundamental fix) |
| GSI with different partition key | Medium | Good (alternative access pattern) |

**Tricky**: Even On-Demand mode has per-partition limits (~3,000 RCU / 1,000 WCU per partition). On-Demand just removes the TABLE-level limit. If your partition key is bad, you STILL get throttled on On-Demand!

---


**Q32: SCENARIO — Your application's P99 latency doubled overnight but P50 is unchanged. What does this mean and how do you investigate?**

**A:**

**P50 unchanged + P99 doubled = A SUBSET of requests is very slow while most are fine.**

```
Before:
├── P50 (median): 50ms (half of requests < 50ms)
├── P95: 100ms
└── P99: 200ms (1% of requests > 200ms)

After:
├── P50: 50ms (unchanged — most requests still fast!)
├── P95: 150ms (slightly worse)
└── P99: 400ms (1% of requests now taking 400ms+)
```

**What causes P99 to spike without affecting P50:**

1. **Cold starts (Lambda/containers):**
```
99% of requests hit warm instances → 50ms (P50 unchanged)
1% hit cold starts → 400ms+ (P99 doubles)
Cause: Scaling event, deployment, or provisioned concurrency exhausted
Check: Lambda InitDuration metric, ProvisionedConcurrencySpilloverInvocations
```

2. **Database connection pool exhaustion:**
```
Normal: Get connection from pool → 0ms overhead
When pool full: Wait for connection → 200ms+ wait time
Only affects requests that arrive when pool is empty (tail latency)
Check: DatabaseConnections metric, app connection pool wait metrics
```

3. **Garbage collection pauses (JVM):**
```
Most requests: No GC → fast
Occasional: Full GC pause → 200ms+ stop-the-world
Affects random 1-2% of requests
Check: JVM GC metrics, cpu_usage_system spikes, Lambda duration outliers
```

4. **Network retries (microservices):**
```
First attempt: Succeeds in 50ms (99% of requests)
Retry needed: Timeout (200ms) + retry (50ms) = 250ms extra
Only requests hitting a momentarily unhealthy backend retry
Check: Retry metrics, downstream service health
```

5. **Cache miss vs cache hit:**
```
Cache hit: 5ms (direct from Redis)
Cache miss: 200ms (database query + cache write)
If 1% of requests are cache misses (new/expired keys) → P99 spikes
Check: ElastiCache CacheHitRate, per-endpoint latency breakdown
```

6. **DNS resolution timeout:**
```
Normally: DNS cached → 0ms
Occasionally: Cache expired → DNS lookup → 50-100ms
If DNS server slow → timeout + retry → 200ms+
Check: Custom metric on DNS resolution time, or VPC DNS throttling
```

**Investigation approach:**
```bash
# CloudWatch Logs Insights - find slow requests
fields @timestamp, @message, latency
| filter latency > 400
| sort latency desc
| limit 50

# Look for patterns:
# - Same user/IP? (specific client issue)
# - Same endpoint? (specific code path slow)
# - Same time pattern? (correlates with cron job, GC, backup?)
# - Same target? (specific instance unhealthy)
```

**Tricky**: Most dashboards show AVERAGE latency which hides P99 problems completely! Average of 1000 requests at 50ms + 10 requests at 5000ms = 99ms average — looks fine! But those 10 users waited 5 SECONDS. ALWAYS monitor P99, not just average.

---


**Q33: SCENARIO — CloudWatch shows `StatusCheckFailed_System = 1` on an EC2 instance. What exactly happened and what are your options?**

**A:**

**`StatusCheckFailed_System = 1` means AWS HARDWARE has a problem — NOT your software.**

```
Two types of status checks:
├── StatusCheckFailed_Instance (software/OS level)
│   ├── OS crashed/hung
│   ├── Networking misconfigured
│   ├── Disk full / corrupt filesystem
│   ├── Incompatible kernel
│   └── YOUR responsibility to fix
│
└── StatusCheckFailed_System (hardware level) ← THIS ONE
    ├── Loss of network connectivity (host-level)
    ├── Loss of system power (physical host)
    ├── Hardware issues on physical host
    ├── Software issues on host hypervisor
    └── AWS responsibility — you can't fix the hardware!
```

**What to do:**

1. **Immediate — Stop and Start (not reboot!):**
```bash
# STOP then START moves instance to different physical hardware
aws ec2 stop-instances --instance-ids i-abc123
aws ec2 start-instances --instance-ids i-abc123

# WARNING: Public IP changes (unless Elastic IP)!
# WARNING: Instance store data is LOST!
# WARNING: Does NOT work for instance-store-backed instances!

# REBOOT does NOT help — stays on same hardware!
```

2. **Automated recovery (recommended):**
```bash
# CloudWatch Alarm with EC2 Recovery Action
aws cloudwatch put-metric-alarm \
  --alarm-name "AutoRecover-i-abc123" \
  --namespace AWS/EC2 \
  --metric-name StatusCheckFailed_System \
  --dimensions Name=InstanceId,Value=i-abc123 \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:automate:us-east-1:ec2:recover
```

3. **Auto Scaling Group (best for production):**
```
ASG detects unhealthy instance → Terminates → Launches new one
├── Works for both instance and system failures
├── New instance on healthy hardware
└── Combined with ELB health checks for full coverage
```

**Recovery action limitations:**
- Only works for instances with EBS root volume (not instance store)
- Instance retains: private IP, Elastic IP, EBS volumes, metadata
- Instance loses: instance store data, public IP (if no EIP)
- Does NOT work across AZs (recovers in same AZ)

**Tricky**: If an entire AZ has issues (rare but happens), system status checks fail for MANY instances simultaneously. Recovery alarm tries to recover in the SAME AZ (may fail again). Solution: Multi-AZ architecture with ASG spanning multiple AZs.

---


**Q34: SCENARIO — ECS Service shows `CPUUtilization` at 10% but `MemoryUtilization` at 95%. Auto-scaling on CPU isn't triggering. What's the fix?**

**A:**

**Classic mistake: Scaling on the WRONG metric for a memory-bound application.**

```
Your ECS Service:
├── Task Definition: 1024 CPU units, 2048 MB memory
├── Running Tasks: 4
├── CPUUtilization: 10% (barely using CPU)
├── MemoryUtilization: 95% (about to OOMKill!)
├── Auto-scaling: Target Tracking on CPUUtilization = 60%
└── Result: ASG thinks everything is fine! Won't scale out.

Meanwhile: Tasks are hitting memory limits → OOMKilled → restarted → bad UX
```

**Why this happens:**
- Java/Node.js applications that are memory-heavy but CPU-light
- Applications with large in-memory caches
- Applications processing large payloads/files
- Services with many idle connections (each holds memory)

**Solutions:**

1. **Scale on MemoryUtilization instead:**
```json
{
  "TargetTrackingScalingPolicyConfiguration": {
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
    },
    "TargetValue": 70.0,
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }
}
```

2. **Scale on BOTH metrics (most robust):**
```
Policy 1: Target CPU at 60% → scales if CPU-bound
Policy 2: Target Memory at 70% → scales if memory-bound
ECS picks whichever results in MORE capacity (most aggressive wins)
```

3. **Scale on custom application metric:**
```
Custom metric: "RequestQueueDepth" or "ActiveConnections"
More accurate than infrastructure metrics for application load
```

4. **Fix the memory issue:**
```
├── Memory leak? → Profile and fix application
├── JVM heap too large? → Reduce -Xmx, enable GC tuning
├── Task memory too small? → Increase task memory definition
└── Need larger node? → Use bigger Fargate task size
```

**Critical ECS memory concepts:**
```
Task Definition Memory = Hard limit (OOMKill if exceeded)
Task Definition MemoryReservation = Soft limit (for scheduling/metrics)
MemoryUtilization metric = Used / Hard Limit × 100%

Example:
├── Hard limit: 2048 MB
├── Soft reservation: 1024 MB
├── Actual usage: 1900 MB
├── MemoryUtilization: 1900/2048 = 92.8%
└── If it hits 2048 → Container killed with exit code 137 (OOMKilled)
```

**Tricky**: ECS `MemoryUtilization` is relative to the HARD limit in task definition. If you set memory=4096 but your app only needs 2048 at peak, utilization shows 50% even when the app is actually at its maximum. Rightsize the task definition to match actual peak usage for meaningful metrics.

---


**Q35: SCENARIO — You set up a CloudWatch alarm for `NetworkIn` > 1GB/period but it never fires even during traffic spikes. What's wrong?**

**A:**

**Most likely: UNIT MISMATCH. One of the most common CloudWatch mistakes!**

```
The problem:
├── NetworkIn metric reports in BYTES
├── You set threshold to 1,000,000,000 (1GB in bytes) ✓
├── BUT you're using wrong STATISTIC!

Scenario 1: Using "Average" statistic
├── Period: 300 seconds (5 min)
├── NetworkIn Average = Total bytes / Number of data points
├── If 10 data points reported: Average = TotalBytes / 10
├── This is NOT total traffic! It's average per report cycle!
└── FIX: Use "Sum" statistic for cumulative metrics!

Scenario 2: Using wrong period
├── You want: alert if > 1GB in 5 minutes
├── But period = 3600 (1 hour)
├── Over 1 hour, 1GB is easy to reach normally
└── FIX: Match period to your alert intent
```

**Correct alarm for "alert if network traffic > 1GB in 5 minutes":**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "HighNetworkTraffic" \
  --namespace AWS/EC2 \
  --metric-name NetworkIn \
  --dimensions Name=InstanceId,Value=i-abc123 \
  --statistic Sum \         # SUM not Average!
  --period 300 \            # 5 minutes
  --evaluation-periods 1 \
  --threshold 1073741824 \  # 1GB in bytes
  --comparison-operator GreaterThanThreshold
```

**When to use which statistic:**

| Metric Type | Use Statistic | Why |
|-------------|--------------|-----|
| NetworkIn/Out (bytes) | Sum | Total traffic in period |
| CPUUtilization (%) | Average | Mean utilization |
| Latency | Average, p99 | Mean or tail latency |
| Error count | Sum | Total errors in period |
| Queue depth | Average or Maximum | Current depth or worst-case |
| StatusCheckFailed | Maximum | Any failure in period = 1 |
| Connections | Average | Typical concurrency |

**Other common unit mistakes:**
```
FreeableMemory: Reported in BYTES → alarm threshold must be in bytes
                500 MB = 524,288,000 bytes (not 500!)
                
Duration (Lambda): Reported in MILLISECONDS
                   5 seconds = 5000 (not 5!)

ReadLatency (RDS): Reported in SECONDS (with decimals)
                   5ms = 0.005 (not 5!)
```

**Tricky**: Some metrics like `CPUCreditBalance` are unitless (count of credits). `FreeableMemory` is in bytes. `ReadLatency` is in seconds. Always check the UNIT in metric documentation before setting thresholds. A threshold of "500" means very different things depending on the unit!

---


**Q36: SCENARIO — After a deployment, Lambda `Duration` increased from 200ms to 800ms. No errors, no throttles. How do you find the root cause using CloudWatch?**

**A:**

**Systematic investigation using CloudWatch + X-Ray:**

**Step 1: Isolate WHEN it started:**
```bash
# Get Duration metric with 1-minute resolution around deployment time
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-function \
  --start-time "2024-01-01T14:00:00Z" \
  --end-time "2024-01-01T16:00:00Z" \
  --period 60 \
  --statistics Average p99 Maximum
  
# Confirm: Duration jumped exactly at deployment time
```

**Step 2: Check if it's cold starts vs all invocations:**
```bash
# Check InitDuration (only reported for cold starts)
# If InitDuration also increased → dependency loading slower
# If InitDuration unchanged but Duration increased → runtime issue

# Also check: ProvisionedConcurrencySpilloverInvocations
# If > 0 after deployment → cold starts happening (maybe new version not warmed)
```

**Step 3: Check ConcurrentExecutions:**
```
If concurrent executions increased → function is running longer → backing up
Self-reinforcing: Slow function → holds concurrency longer → more cold starts → even slower
```

**Step 4: CloudWatch Logs Insights — find what's slow:**
```sql
-- Find slow invocations and their details
fields @timestamp, @duration, @requestId
| filter @duration > 500
| sort @duration desc
| limit 20

-- If using structured logging:
fields @timestamp, @duration, db_query_time, external_api_time, processing_time
| filter @duration > 500
| stats avg(db_query_time), avg(external_api_time), avg(processing_time)
```

**Step 5: X-Ray trace analysis:**
```
Typical trace breakdown:
├── Lambda Initialization: 100ms (cold start only)
├── Handler Start
│   ├── DB Query 1: 50ms → 50ms (unchanged)
│   ├── DB Query 2: 30ms → 300ms (10x SLOWER! ← ROOT CAUSE)
│   ├── External API: 80ms → 80ms (unchanged)
│   └── Processing: 20ms → 20ms (unchanged)
└── Handler End

Root cause: DB Query 2 became slow after deployment
Possible reasons:
├── New query introduced without index
├── Table statistics stale after data migration
├── Connection pool cold (new Lambda env = new connections)
└── Downstream service throttling new code path
```

**Common deployment-related latency increases:**

| Cause | How to Detect | Fix |
|-------|---------------|-----|
| New unindexed DB query | X-Ray shows DB subsegment slow | Add index |
| Larger deployment package | InitDuration increased | Tree-shake, use layers |
| New dependency loaded | InitDuration increased | Lazy-load imports |
| Connection pool cold | First invocations slow, stabilizes | Connection reuse, provisioned concurrency |
| Increased payload size | Processing time up | Optimize serialization |
| New external API call added | X-Ray shows new subsegment | Async/cache the call |
| VPC cold start (new ENI) | InitDuration = 8-15s | Provisioned concurrency |

**Tricky**: Lambda `Duration` includes time waiting for downstream services (DB, APIs). A "slow Lambda" is often actually a "slow downstream." Always use X-Ray or structured logging to break down WHERE time is spent within the function.

---


**Q37: SCENARIO — ALB `HealthyHostCount` dropped from 4 to 2, but EC2 instances show as "running" with passing StatusChecks. Why are targets unhealthy?**

**A:**

**EC2 status checks and ALB health checks are COMPLETELY DIFFERENT!**

```
EC2 Status Checks (infrastructure level):
├── System check: Hardware/hypervisor OK?
├── Instance check: OS responsive? Network configured?
└── Result: Both passing ✓ (instance is "running")

ALB Health Check (application level):
├── Sends HTTP GET to configured path (e.g., /health)
├── Expects specific status code (200 by default)
├── Within timeout (default 5s)
├── Configured number of checks must pass
└── Result: FAILING! Application not responding properly
```

**Common causes of ALB health check failure with healthy EC2:**

1. **Application crashed but OS is fine:**
```
EC2 status: Running ✓ (Linux is up and responding to ARP)
App status: Crashed (nginx/java/node process died)
Health check: GET /health → Connection Refused → Unhealthy!

Fix: Restart application, check app logs
Prevent: Use systemd restart-on-failure, container orchestration
```

2. **Wrong port or path configured:**
```
App listens on: port 8080, path /api/health
ALB health check: port 80, path /health → 404 → Unhealthy!

Fix: Match health check config to actual app endpoint
```

3. **Application returning non-200 status:**
```
App responds but returns: 503 (maintenance mode after deployment)
ALB expects: 200
Result: Unhealthy!

Fix: Configure ALB to accept 200-299 range, or fix app response
Health check matcher: "200-299" instead of just "200"
```

4. **Security Group blocking health checks:**
```
ALB SG: Allows outbound to targets ✓
Target SG: Allows inbound from internet (port 443) ✓
Target SG: Does NOT allow inbound from ALB SG on health check port!
Health check comes from ALB IP → blocked → timeout → Unhealthy!

Fix: Allow inbound from ALB security group on health check port
```

5. **Health check timeout too aggressive:**
```
App startup time: 30 seconds
Health check interval: 10s, timeout: 5s, unhealthy threshold: 2
Result: App starting → first 2 checks fail → marked unhealthy BEFORE app is ready!

Fix: Increase healthy threshold or add health check grace period
ECS: healthCheckGracePeriodSeconds: 60
```

6. **Application memory/thread exhaustion:**
```
App is alive but TOO SLOW to respond within timeout
Health check timeout: 5 seconds
App response time: 8 seconds (under heavy load)
Result: Timeout → Unhealthy → ALB removes target → remaining targets MORE overloaded → cascade!

Fix: Separate lightweight health check endpoint (not affected by load)
     /health → just return 200 (no DB query, no business logic)
```

**Debugging commands:**
```bash
# Check target health with details
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/my-tg/abc123

# Output shows REASON for unhealthy:
# "Target.ResponseCodeMismatch" → Wrong status code
# "Target.Timeout" → Health check timed out
# "Target.FailedHealthChecks" → Health check failed
# "Elb.InitialHealthChecking" → Still in initial check period
```

**Tricky**: When ALB marks targets unhealthy, it stops sending traffic to them BUT continues health checking. If a target recovers, it must pass the "healthy threshold" number of consecutive checks before receiving traffic again. During this recovery period, you're still running on reduced capacity!

---


**Q38: SCENARIO — Your CloudWatch bill unexpectedly tripled this month. How do you identify what's costing so much?**

**A:**

**CloudWatch pricing components:**

| Component | Price | Common Cost Driver |
|-----------|-------|-------------------|
| Custom metrics | $0.30/metric/month | Publishing too many unique dimension combinations |
| API calls (GetMetricData) | $0.01/1,000 metrics requested | Dashboards refreshing frequently |
| Logs ingestion | $0.50/GB | Verbose application logging |
| Logs storage | $0.03/GB/month | Not setting retention policies! |
| Alarms | $0.10/standard, $0.30/high-res | Many alarms per resource |
| Dashboards | $3.00/dashboard/month | Many dashboards |
| Contributor Insights | $0.02/rule/1M events matched | Rules on high-volume services |
| Metric Streams | $0.003/metric update | Streaming everything to third party |

**Investigation:**

1. **Check AWS Cost Explorer with CloudWatch filter:**
```
Group by: Usage Type
Look for: CW:MetricMonitorUsage (custom metrics)
          CW:DataProcessing-Bytes (logs ingestion)
          CW:TimedStorage-ByteHrs (logs storage)
          CW:Requests (API calls)
```

2. **Most common cost explosions:**

**A. Custom metrics with high-cardinality dimensions:**
```
BAD: Publishing metric per request_id → millions of unique metrics!
cloudwatch.put_metric_data(
    MetricData=[{
        'MetricName': 'Latency',
        'Dimensions': [{'Name': 'RequestId', 'Value': unique_id}]  # DANGER!
    }]
)
# Each unique dimension combo = separate metric = $0.30/month

GOOD: Use bounded dimensions only
Dimensions: Service, Environment, StatusCode (limited values)
```

**B. CloudWatch Logs without retention:**
```
Default retention: NEVER EXPIRE (infinite storage!)
Fix: Set retention policy on all log groups

aws logs put-retention-policy \
  --log-group-name /ecs/my-app \
  --retention-in-days 30

# Common retention settings:
# Production: 30-90 days
# Development: 7 days
# Debug logs: 1-3 days
```

**C. Verbose logging (DEBUG level in production):**
```
Fix: Log at WARN/ERROR in production, DEBUG only in dev
Reduce: Don't log full request/response bodies (just IDs + status)
Compress: Use structured JSON logging (more efficient)
```

**D. Dashboard auto-refresh API calls:**
```
10 widgets × 20 metrics each × refresh every 60s = 288,000 GetMetricData calls/day
Cost: ~$86/month just for ONE dashboard!

Fix: 
├── Reduce refresh rate (5 min instead of 1 min)
├── Reduce time range (1 hour instead of 24 hours)
├── Consolidate widgets (fewer API calls)
└── Use cross-account dashboards instead of per-account
```

**Cost optimization checklist:**
```bash
# 1. Find log groups without retention (infinite storage!)
aws logs describe-log-groups --query 'logGroups[?retentionInDays==`null`].logGroupName'

# 2. Find log groups by size
aws logs describe-log-groups --query 'logGroups[*].[logGroupName,storedBytes]' \
  | sort -k2 -rn | head -10

# 3. Count custom metrics per namespace
aws cloudwatch list-metrics --namespace MyApp \
  | jq '.Metrics | length'

# 4. Check for unused alarms
aws cloudwatch describe-alarms --state-value INSUFFICIENT_DATA \
  | jq '.MetricAlarms[].AlarmName'
```

**Tricky**: EMF (Embedded Metric Format) is cheaper for high-volume custom metrics because you pay LOG INGESTION price ($0.50/GB) instead of PutMetricData API price. But EMF metrics still count as custom metrics ($0.30/metric/month). The savings is on the API call cost, not the metric storage.

---


**Q39: SCENARIO — You created a Composite Alarm combining 3 sub-alarms (CPU + Memory + 5xx errors) but it keeps flapping between OK and ALARM every few minutes. How do you stabilize it?**

**A:**

**Problem: Sub-alarms resolve at different times causing composite alarm oscillation.**

```
Timeline:
T+0:  CPU alarm → ALARM
T+1:  Memory alarm → ALARM  
T+2:  5xx alarm → ALARM
T+3:  Composite (ALL in alarm) → ALARM ← Triggers alert
T+4:  CPU drops → CPU alarm → OK
T+5:  Composite (not ALL in alarm) → OK ← Recovery alert
T+6:  CPU spikes again → CPU alarm → ALARM
T+7:  Composite → ALARM ← Another alert!
T+8:  Memory recovers → Composite → OK ← Another recovery!
...FLAPPING!
```

**Solutions:**

1. **Use `ALARM_ACTIONS_SUPPRESSOR` (Suppression — wait before transitioning):**
```bash
# Wait 5 minutes after composite enters ALARM before taking action
aws cloudwatch put-composite-alarm \
  --alarm-name "ServiceDegraded" \
  --alarm-rule 'ALARM(HighCPU) AND ALARM(High5xx)' \
  --actions-suppressor "WaitToFire" \
  --actions-suppressor-wait-period 300 \
  --actions-suppressor-extension-period 300
```

2. **Change composite logic from AND to OR with suppression:**
```
Instead of: ALARM when ALL alarms fire (too strict, keeps cycling)
Use: ALARM when ANY 2 of 3 fire (more stable signal)

Alarm Rule: 'ALARM(HighCPU) AND ALARM(High5xx) OR ALARM(HighCPU) AND ALARM(HighMemory) OR ALARM(High5xx) AND ALARM(HighMemory)'
```

3. **Add hysteresis to sub-alarms:**
```
CPU Alarm:
├── Enter ALARM: 3 out of 5 periods > 80% (hard to enter)
├── Return to OK: 5 out of 5 periods < 60% (hard to leave)
└── Gap between thresholds prevents oscillation

# Use "TreatMissingData: breaching" during suppression to avoid premature OK
```

4. **Use longer evaluation periods on sub-alarms:**
```
Before: CPU > 80% for 1 out of 1 periods (60s) → flappy
After:  CPU > 80% for 3 out of 5 periods (5min each, 25min window) → stable
```

5. **Add SNS filter for notification deduplication:**
```python
# Lambda behind SNS filters duplicate alerts
def handler(event, context):
    alarm_name = event['detail']['alarmName']
    state = event['detail']['state']['value']
    
    # Check DynamoDB: was this alarm already notified in last 30 min?
    last_alert = dynamodb.get_item(Key={'alarm': alarm_name})
    if last_alert and (now - last_alert['timestamp']) < 1800:
        return  # Suppress duplicate
    
    # Send alert and record
    send_pagerduty_alert(alarm_name, state)
    dynamodb.put_item(Item={'alarm': alarm_name, 'timestamp': now})
```

**Best practice composite alarm design:**
```
Tier 1: Warning (notify Slack)
  Rule: ALARM(CPU > 70%) OR ALARM(Memory > 80%) OR ALARM(5xx > 10/min)
  Action: Slack channel notification
  Suppression: 5 minutes (avoid noise)

Tier 2: Critical (page on-call)
  Rule: ALARM(CPU > 90% for 10min) AND ALARM(5xx > 100/min)
  Action: PagerDuty
  Suppression: 10 minutes (confirmed sustained issue)

Tier 3: Outage (page everyone)
  Rule: ALARM(HealthyHosts = 0) OR ALARM(5xx_rate > 50%)
  Action: PagerDuty + SMS + bridge call
  Suppression: 0 (immediate for full outage)
```

**Tricky**: Composite alarms themselves DON'T have evaluation periods or data point requirements. They change state IMMEDIATELY when sub-alarm states change. All stability must come from sub-alarm configuration or suppression settings.

---


**Q40: SCENARIO — Production is down. CloudWatch Alarms are all green (OK). How is this possible and how do you prevent it?**

**A:**

**The most terrifying scenario: "All lights green, everything's broken."**

**Why alarms can be green during an outage:**

1. **Metric stopped publishing (INSUFFICIENT_DATA treated as OK):**
```
Application crashed → no more metrics published → alarm state = INSUFFICIENT_DATA
If configured with TreatMissingData: "notBreaching" → alarm stays OK!

Fix: Set TreatMissingData: "breaching" for critical alarms
Or: Set INSUFFICIENT_DATA action to also alert
```

2. **Monitoring only infrastructure, not business:**
```
Alarms exist for: CPU, Memory, Disk → All OK (infra is fine!)
Missing alarms for: Order rate, Login success, API response time
Application bug returns 200 OK with empty/wrong data → infra metrics normal

Fix: Add business-level metrics:
├── Orders per minute (drops to 0 = outage!)
├── Successful logins per minute
├── Revenue per minute
└── Custom: "HeartbeatMetric" (app publishes "1" every minute)
```

3. **Health checks checking the wrong thing:**
```
Health endpoint: /health → returns 200 (checks if process alive)
But: Main endpoint /api/orders → returning 500 (DB connection pool exhausted)
ALB health check passes → targets stay "healthy" → no alarm

Fix: Deep health checks that verify actual functionality
GET /health/deep → checks DB, cache, downstream services
```

4. **Alarm threshold never reached:**
```
Alarm: 5xx > 1000/minute
Actual: 5xx = 999/minute (just below threshold) for hours!
Users are suffering but alarm doesn't fire.

Fix: Use percentage-based alarms: 5xx_rate > 5%
Or: Use Anomaly Detection (detects unusual patterns)
```

5. **Wrong dimension — alarm on wrong resource:**
```
Alarm configured on: InstanceId=i-old123 (was replaced during deployment!)
Current instance: i-new456 (has no alarm!)

Fix: Use ASG/Service-level metrics (not instance-level)
     Or use tag-based metric filtering
     Or use CloudFormation/Terraform to ensure alarms match resources
```

6. **CloudWatch delay (metrics not yet available):**
```
Some metrics have 1-5 minute publication delay
If outage started 2 minutes ago → metrics haven't arrived → alarms still OK

Fix: Add synthetic monitoring (actively probes) in addition to passive metrics
     CloudWatch Synthetics canary checks every 1 minute
```

**The "Dead Man's Switch" pattern:**
```python
# Application publishes "1" every minute
# Alarm: Metric < 1 for 3 consecutive periods = app is DEAD

def health_heartbeat():
    cloudwatch.put_metric_data(
        Namespace='MyApp/Heartbeat',
        MetricData=[{
            'MetricName': 'IsAlive',
            'Value': 1,
            'Unit': 'Count'
        }]
    )

# Alarm:
# If IsAlive Sum < 1 for 3 periods (3 minutes) → CRITICAL
# Treat missing data as "breaching" (no data = app dead!)
```

**Complete monitoring gap prevention checklist:**
```
□ Business metrics (orders, revenue, logins) — not just infra
□ Synthetic monitoring (active probing every 1-5 minutes)
□ Heartbeat metrics ("dead man's switch")
□ TreatMissingData = "breaching" for critical alarms
□ INSUFFICIENT_DATA state also triggers notification
□ Health checks verify actual functionality (not just process alive)
□ Percentage-based thresholds (not absolute numbers)
□ Anomaly Detection for baseline deviations
□ Dashboard with "last updated" timestamps (detect stale data)
□ External monitoring (Route53 health checks, third-party uptime tool)
```

**Tricky**: The worst outages are the ones where your monitoring agrees everything is fine. This happens when you only monitor SYMPTOMS you've seen before. You need ANOMALY DETECTION to catch the scenarios you haven't imagined yet. CloudWatch Anomaly Detection uses ML to baseline normal behavior and alerts on deviations.

---
