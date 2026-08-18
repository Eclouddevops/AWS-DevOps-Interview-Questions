# Production Incident Management & Live Troubleshooting — Tricky Interview Q&A

## Table of Contents
1. [On-Call & Incident Response](#on-call--incident-response)
2. [Live Production Debugging](#live-production-debugging)
3. [Database Production Issues](#database-production-issues)
4. [Memory & CPU Production Crises](#memory--cpu-production-crises)
5. [Networking Production Failures](#networking-production-failures)
6. [Deployment Gone Wrong](#deployment-gone-wrong)
7. [Data Loss & Recovery Scenarios](#data-loss--recovery-scenarios)

---

## On-Call & Incident Response

---

**Q1: It's 3 AM. PagerDuty wakes you up: "API latency p99 > 5s for 10 minutes." Walk me through your first 5 minutes.**

**A:**


**Minute 0-1: Triage & Acknowledge**
```
1. Acknowledge the alert (prevent escalation)
2. Open monitoring dashboard (Grafana/CloudWatch)
3. Check: Is this customer-impacting? (error rate, traffic volume)
4. Determine blast radius: one service? one region? global?
```

**Minute 1-3: Correlate & Identify**
```bash
# Check recent deployments (most common root cause - 70% of incidents)
git log --oneline --since="1 hour ago" --all
kubectl rollout history deployment/api-service

# Check infrastructure changes
aws cloudtrail lookup-events --lookup-attributes \
  AttributeKey=EventName,AttributeValue=UpdateService --max-results 10

# Check dependency health
curl -w "%{time_total}" https://internal-db-endpoint/health
curl -w "%{time_total}" https://redis-cluster:6379/ping

# Check resource saturation
kubectl top pods -n production --sort-by=memory
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name CPUUtilization --period 60 --statistics Average
```

**Minute 3-5: Mitigate (stop the bleeding)**
```bash
# If deployment-related: ROLLBACK IMMEDIATELY (fix later)
kubectl rollout undo deployment/api-service -n production

# If traffic spike: scale out
kubectl scale deployment/api-service --replicas=10 -n production

# If dependency failure: activate circuit breaker / failover
# Enable maintenance mode if customer data at risk
```

**Tricky**: Senior engineers DON'T debug at 3 AM. They **mitigate first, investigate later**. The goal is MTTR (Mean Time To Recovery), not root cause analysis. RCA happens in business hours. A junior will spend 45 minutes debugging while customers suffer.

---


**Q2: Your company uses microservices. Service A calls B calls C calls D. Customers report intermittent timeouts. How do you identify which service is the bottleneck?**

**A:**

**Step 1: Distributed Tracing (the real answer)**
```
Use Jaeger/X-Ray/Datadog APM to trace a single request:

Request → Service A (2ms) → Service B (3ms) → Service C (4500ms!) → Service D (1ms)
                                                    ↑ BOTTLENECK
```

**Step 2: If no tracing exists (common in older systems)**
```bash
# Check service-to-service latency from each service's metrics
# Service A logs:
grep "upstream_response_time" /var/log/nginx/access.log | awk '{print $NF}' | sort -n | tail

# Check connection pools — exhausted pools cause timeouts
curl http://service-b:8080/actuator/metrics/hikaricp.connections.active
curl http://service-c:8080/actuator/metrics/hikaricp.connections.pending

# Check thread pools
curl http://service-c:8080/actuator/threaddump | grep -c "WAITING"

# Network level — check TCP retransmissions between services
ss -ti dst service-c-ip | grep retrans
```

**Step 3: Common culprits in production**
```
1. Database slow query on Service C (missing index after data growth)
2. Connection pool exhaustion (configured for 10 connections, needs 50)
3. DNS resolution delays (intermittent = DNS TTL expiry + slow resolver)
4. Garbage collection pauses (Java services: check GC logs)
5. Noisy neighbor on shared infrastructure (another pod consuming node resources)
6. TCP connection timeout defaults too high (30s default, should be 3s with retry)
```

**Tricky**: "Intermittent" is the keyword. This often means **connection pool exhaustion** — works fine until pool fills up, then requests queue. Or **GC pauses** — fine 99% of the time, then JVM stops for 2s. Both look fine in average metrics but terrible in p99.

---

**Q3: Production database CPU is at 100%. Application is returning 500 errors. You have 60 seconds before the CEO calls. What do you do?**

**A:**


**Immediate (0-30 seconds):**
```sql
-- Find and kill the expensive queries
-- PostgreSQL:
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 10;

-- Kill the long-running query
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE duration > interval '30 seconds' AND state = 'active';

-- MySQL:
SHOW PROCESSLIST;
KILL <process_id>;
```

**Next 30 seconds:**
```bash
# If it's a read-heavy workload — shift traffic to read replica
# Update application config or DNS to point reads to replica
aws rds describe-db-instances --query 'DBInstances[].{ID:DBInstanceIdentifier,Role:ReadReplicaSourceDBInstanceIdentifier}'

# If it's a runaway batch job — identify and stop it
ps aux | grep -i "batch\|etl\|migration"

# Scale vertically if possible (RDS)
aws rds modify-db-instance --db-instance-identifier prod-db \
  --db-instance-class db.r6g.4xlarge --apply-immediately
# WARNING: This causes ~30s downtime for single-AZ!
```

**Root cause patterns in production:**
| Symptom | Likely Cause | Quick Fix |
|---------|-------------|-----------|
| Sudden CPU spike after deployment | New query without index | Rollback deployment |
| Gradual CPU increase over weeks | Table growth, stats stale | `ANALYZE` table, add index |
| CPU spike at specific time daily | Scheduled report/batch job | Reschedule to off-peak |
| CPU 100% with connection spike | Connection storm (app restart) | Implement connection pooling (PgBouncer) |

**Tricky**: In RDS, you CANNOT SSH into the database server. You must use Performance Insights, `pg_stat_activity`, or Enhanced Monitoring. Also, `modify-db-instance` for Multi-AZ does a failover (1-2 min downtime), NOT the 30s single-AZ restart. Know the difference!

---

## Live Production Debugging

---

**Q4: Your application works perfectly in staging but returns random 502 errors in production (only during peak hours, 5% of requests). Staging has identical code. How do you debug this?**

**A:**

**Why staging works but production doesn't (the real reasons):**
```
1. SCALE: Staging has 2 pods, production has 20. Race conditions appear at scale.
2. TRAFFIC PATTERNS: Real users do things test suites don't.
3. DATA VOLUME: Staging DB has 1GB, production has 500GB. Query plans differ.
4. RESOURCE CONTENTION: Shared nodes, noisy neighbors, network saturation.
5. CONFIGURATION DRIFT: "Identical" is never truly identical.
```

**Debugging 502 specifically:**
```bash
# 502 = upstream closed connection prematurely
# ALB/Nginx got no response from backend

# Step 1: Check where 502 originates
# Is it ALB → App, or App → Downstream?
aws elbv2 describe-target-health --target-group-arn $TG_ARN

# Step 2: Check if pods are being killed during requests
kubectl get events -n prod --field-selector reason=Killing
kubectl describe pod <pod> | grep -A5 "Last State"

# Step 3: Check if it's keepalive mismatch
# ALB keepalive = 60s (default), App keepalive = 55s?
# App closes connection, ALB sends request on closed connection = 502!
grep "keepalive_timeout" /etc/nginx/nginx.conf
# FIX: App keepalive MUST be > ALB idle timeout

# Step 4: Check resource limits causing OOMKill
kubectl top pods -n prod --containers | sort -k4 -h | tail -10
dmesg | grep -i "oom\|killed"

# Step 5: Check max connections / worker processes
# Nginx: worker_connections * worker_processes = max concurrent
# If exceeded during peak = 502
```

**The actual fix (90% of the time):**
```nginx
# nginx.conf - keepalive timeout MUST exceed load balancer idle timeout
keepalive_timeout 65;  # ALB default is 60s

# Increase upstream keepalive connections
upstream backend {
    server app:8080;
    keepalive 32;  # Reuse connections instead of creating new ones
}
```

**Tricky**: The keepalive race condition is the #1 cause of intermittent 502s in AWS. ALB idle timeout = 60s. If your app's keepalive is 60s too, there's a race where the app closes the connection at the exact moment ALB sends a request. Set app keepalive to 65s+ ALWAYS.

---


**Q5: A senior developer pushed a config change directly to production (bypassing CI/CD). Now the application is partially broken — some features work, some don't. You can't simply rollback because it also included a critical security patch. What do you do?**

**A:**

**Immediate Assessment:**
```bash
# What exactly changed?
git diff HEAD~1..HEAD --stat
git log --oneline -5

# Identify which features are broken
# Check error logs filtered to last 30 minutes
kubectl logs -l app=myservice --since=30m | grep -i "error\|exception\|fatal" | sort | uniq -c | sort -rn

# Check if it's config-related
kubectl get configmap app-config -o yaml
kubectl get secret app-secret -o yaml | base64 -d
```

**The Professional Response:**
```
1. Cherry-pick ONLY the security patch onto the previous stable version
   git checkout last-stable-tag
   git cherry-pick <security-commit-hash>
   
2. Deploy the cherry-picked version (stable + security fix)
   
3. Revert the broken changes in a NEW branch
   git revert <broken-commit> --no-commit
   # Keep security fix, undo the rest
   
4. Run through CI/CD properly this time
   
5. Post-incident: Implement branch protection rules
   - Require PR reviews
   - Require CI passing
   - No direct push to main/production
```

**Prevention (the real interview answer):**
```yaml
# GitHub branch protection
branches:
  - name: main
    protection:
      required_pull_request_reviews:
        required_approving_review_count: 2
      required_status_checks:
        strict: true
        contexts: ["ci/tests", "ci/security-scan"]
      enforce_admins: true  # Even admins can't bypass
      restrictions:
        users: []  # Nobody can push directly
```

**Tricky**: The interviewer is testing if you can handle politics + technical. The right answer isn't "yell at the developer." It's: fix production first, then implement guardrails that make it IMPOSSIBLE to repeat. Blameless culture + technical controls > process documents nobody reads.

---

**Q6: Your monitoring shows memory usage climbing 2% per hour on all production pods. It's not crashing yet, but will OOM in ~2 days. How do you investigate a slow memory leak in production WITHOUT disrupting service?**

**A:**

**Non-intrusive investigation:**
```bash
# Step 1: Confirm the pattern
kubectl top pods -n production -l app=myservice --containers
# Run this every 10 min to confirm growth

# Step 2: Check if it correlates with traffic/load
# Memory grows WITH requests = possible per-request leak
# Memory grows INDEPENDENT of traffic = background task leak

# Step 3: Get heap dump from ONE pod (Java)
kubectl exec -it myservice-pod-abc -- jmap -dump:live,format=b,file=/tmp/heap.hprof 1
kubectl cp myservice-pod-abc:/tmp/heap.hprof ./heap.hprof

# Step 4: For Node.js - take heap snapshot
kubectl exec -it myservice-pod-abc -- node -e "
  const v8 = require('v8');
  const fs = require('fs');
  const snapshotStream = v8.writeHeapSnapshot();
  console.log('Snapshot written to', snapshotStream);
"

# Step 5: For Python - use tracemalloc
kubectl exec -it myservice-pod-abc -- python3 -c "
import tracemalloc, requests
tracemalloc.start()
# ... trigger some requests
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:10]:
    print(stat)
"
```

**Common memory leak causes in production:**
```
1. Event listeners never removed (Node.js: EventEmitter leak)
2. Growing caches without TTL/eviction (in-memory dictionaries)
3. Unclosed database connections/cursors
4. Large objects held by closures
5. Prometheus metrics with high-cardinality labels (new time series per request)
6. Session stores growing unbounded (no cleanup job)
7. Buffer pools not released (file uploads stored in memory)
```

**Production-safe fix strategy:**
```bash
# Don't restart all pods at once!
# Use rolling restart to keep service available
kubectl rollout restart deployment/myservice -n production
# This replaces pods one by one (respects PodDisruptionBudget)

# Set resource limits to prevent OOM from taking down the node
resources:
  limits:
    memory: "1Gi"  # Pod gets OOMKilled, not the node
  requests:
    memory: "512Mi"
```

**Tricky**: Prometheus with high-cardinality labels is the sneakiest production memory leak. If you label metrics with `user_id` or `request_id`, every unique value creates a new time series permanently in memory. 1M users = 1M time series × metric size. This kills Prometheus AND your app's metrics client library.

---


## Database Production Issues

---

**Q7: Your production PostgreSQL RDS shows "too many connections" errors. Application has 20 pods, each with a connection pool of 10. RDS max_connections is 150. Do the math — what's wrong and how do you fix it WITHOUT downtime?**

**A:**

**The Math:**
```
20 pods × 10 connections = 200 connections needed
RDS max_connections = 150
DEFICIT = 50 connections (guaranteed failures)

But wait — it's WORSE:
- RDS reserves ~3 connections for replication/superuser
- Monitoring tools (Datadog/CloudWatch agent) use 2-5 connections
- Migration scripts, admin queries, cron jobs = 5-10 more
- ACTUAL available = ~135 connections

20 pods × 10 pool = 200 > 135 available = constant connection errors
```

**Immediate fix (no downtime):**
```bash
# Option 1: Reduce pool size per pod (quick)
# Change HikariCP/SQLAlchemy pool to 5 per pod
# 20 × 5 = 100 < 135 ✓
# Deploy with rolling update

# Option 2: Add PgBouncer as connection pooler (proper fix)
# PgBouncer sits between app and RDS
# App → PgBouncer (1000 connections OK) → RDS (50 actual connections)
```

**PgBouncer deployment pattern:**
```yaml
# Deploy PgBouncer as a sidecar or separate service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: pgbouncer
        image: edoburu/pgbouncer:1.20
        env:
        - name: DATABASE_URL
          value: "postgres://user:pass@prod-rds.xyz.rds.amazonaws.com:5432/mydb"
        - name: MAX_CLIENT_CONN
          value: "1000"      # Accept many app connections
        - name: DEFAULT_POOL_SIZE
          value: "50"        # Only 50 to RDS
        - name: POOL_MODE
          value: "transaction"  # Release connection after each transaction
```

**Tricky**: `POOL_MODE = transaction` vs `session` is critical. With `session` mode, a connection is held for the entire client session (useless for pooling). With `transaction` mode, connections are returned after each transaction — 50 RDS connections can serve 1000+ app connections. BUT: `transaction` mode breaks `SET` commands, prepared statements, and `LISTEN/NOTIFY`. If your app uses these, you need `session` mode for those specific connections.

---

**Q8: Friday 5 PM. A developer ran `DELETE FROM orders WHERE status = 'pending'` in production instead of staging. 50,000 orders deleted. No one noticed until Monday. Point-in-time recovery will lose 3 days of other valid data. What are your options?**

**A:**

**Option 1: Point-in-Time Recovery to a NEW instance (best approach)**
```bash
# Restore RDS to a new instance at the moment BEFORE the delete
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-db \
  --target-db-instance-identifier prod-db-recovery \
  --restore-time "2024-01-19T16:55:00Z"  # Just before the DELETE

# Now you have TWO databases:
# prod-db: current (missing deleted orders, has 3 days of new data)
# prod-db-recovery: has the deleted orders, but missing 3 days of new data

# Extract ONLY the deleted orders from recovery
pg_dump -h prod-db-recovery --table=orders --data-only \
  --column-inserts -f deleted_orders.sql

# Filter to only the records that are missing in prod
# Then INSERT them back into production
psql -h prod-db -c "
  INSERT INTO orders
  SELECT r.* FROM recovered_orders r
  WHERE r.id NOT IN (SELECT id FROM orders)
  AND r.status = 'pending';
"
```

**Option 2: If you have WAL archiving / Binary log replay**
```bash
# For PostgreSQL with WAL archiving:
# Replay WAL from backup to just before the DELETE statement
# Extract the deleted rows
# Replay remaining WAL to current

# For MySQL with binary logs:
mysqlbinlog --start-datetime="2024-01-19 16:50:00" \
  --stop-datetime="2024-01-19 17:00:00" \
  /var/log/mysql/mysql-bin.000123 | grep -B5 "DELETE FROM orders"
```

**Option 3: If you have audit logging / CDC (Change Data Capture)**
```bash
# If using AWS DMS with CDC to S3/Kinesis:
# Query the CDC stream for DELETE events on orders table
# Reconstruct the deleted records from the before-image
aws s3 ls s3://cdc-bucket/orders/2024/01/19/
```

**Prevention (what the interviewer really wants to hear):**
```sql
-- 1. NEVER give developers direct prod DB access
-- Use read replicas for queries, require PR for data changes

-- 2. Implement "soft delete" pattern
ALTER TABLE orders ADD COLUMN deleted_at TIMESTAMP NULL;
-- DELETE becomes: UPDATE orders SET deleted_at = NOW() WHERE...

-- 3. Require transactions with explicit COMMIT for destructive operations
BEGIN;
DELETE FROM orders WHERE status = 'pending';
-- Check row count: 50000 rows — WAIT, that seems like ALL of them!
ROLLBACK;  -- Oops, too many. Let me add more conditions.

-- 4. Database user permissions
GRANT SELECT, INSERT, UPDATE ON orders TO app_user;
-- NO DELETE permission for application user
-- DELETE requires DBA approval through runbook
```

**Tricky**: RDS Point-in-Time Recovery creates a NEW instance — it doesn't modify the existing one. This means you need to handle DNS/endpoint switching or extract data from the recovery instance. Also, PITR is only available within the backup retention period (default 7 days for RDS). If you discover the deletion after retention period, the data is GONE forever. This is why CDC/audit logs are critical.

---


## Memory & CPU Production Crises

---

**Q9: Your Java microservice in EKS suddenly shows GC pauses of 10-15 seconds during peak traffic. Service is returning timeouts. Kubernetes readiness probe fails, pod gets removed from service, traffic shifts to remaining pods (which also start failing). It cascades. How do you stop this death spiral?**

**A:**

**Immediate (stop the cascade):**
```bash
# Step 1: Increase readiness probe tolerance to stop pod removal
kubectl patch deployment java-service -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "java-service",
          "readinessProbe": {
            "failureThreshold": 10,
            "periodSeconds": 10
          }
        }]
      }
    }
  }
}'

# Step 2: Scale UP to absorb load (even if pods are unhealthy)
kubectl scale deployment java-service --replicas=30

# Step 3: Enable GC logging on one pod for diagnosis
kubectl set env deployment/java-service \
  JAVA_OPTS="-Xlog:gc*:file=/tmp/gc.log:time,uptime,level,tags"
```

**Root cause investigation:**
```bash
# Check JVM memory settings vs container limits
kubectl get pod java-pod -o jsonpath='{.spec.containers[0].resources}'
# Container limit: 2Gi, but JVM heap = ???

# CRITICAL: JVM doesn't know about container memory limits by default!
# Old JVMs see NODE memory (64GB) and set heap to 25% = 16GB
# Container only has 2GB → OOM immediately

# Check what JVM thinks it has:
kubectl exec java-pod -- java -XX:+PrintFlagsFinal -version | grep -i heap

# Fix: Set proper JVM flags
JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
# This tells JVM to use 75% of container limit as max heap
```

**GC tuning for production:**
```bash
# G1GC settings for low-latency services
JAVA_OPTS="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:G1HeapRegionSize=16m
  -XX:+UseContainerSupport
  -XX:MaxRAMPercentage=75.0
  -XX:+ParallelRefProcEnabled
  -XX:+AlwaysPreTouch
"

# For ultra-low latency (Java 17+), consider ZGC:
JAVA_OPTS="-XX:+UseZGC -XX:MaxRAMPercentage=75.0"
# ZGC: <1ms pause times regardless of heap size
```

**Tricky**: The "death spiral" pattern: Pod A becomes slow → fails readiness → removed from service → remaining pods get MORE traffic → they also become slow → cascade failure. Prevention: Set `minReadySeconds`, use PodDisruptionBudget, and NEVER set readiness probe `failureThreshold=1`. Also, Java's `-XX:+UseContainerSupport` was buggy before JDK 11. If you're on JDK 8u191+, it works, but older versions? JVM ignores cgroup limits entirely.

---

**Q10: Production alert: disk usage on a Kubernetes worker node hit 90%. Pods are being evicted. You investigate and find `/var/lib/docker` consuming 180GB on a 200GB disk. What's filling it up and how do you fix it?**

**A:**

**Immediate diagnosis:**
```bash
# SSH to the node (or use kubectl debug node)
kubectl debug node/worker-node-1 -it --image=ubuntu

# Check what's consuming space
du -sh /var/lib/docker/*
# Typical output:
# 120GB /var/lib/docker/overlay2  (image layers + container writable layers)
# 45GB  /var/lib/docker/containers  (container logs!)
# 15GB  /var/lib/docker/volumes

# Check container logs (often the #1 culprit)
find /var/lib/docker/containers -name "*.log" -exec ls -lh {} \; | sort -k5 -h | tail -10
# You'll find a container writing GB of logs without rotation!
```

**Immediate fix:**
```bash
# Truncate huge log files (instant space recovery)
truncate -s 0 /var/lib/docker/containers/<container-id>/<container-id>-json.log

# Clean up unused Docker resources
docker system prune -a --volumes -f
# WARNING: This removes ALL stopped containers, unused images, and volumes!

# Safer approach: only remove dangling resources
docker image prune -f
docker container prune -f
docker volume prune -f
```

**Permanent fix (prevent recurrence):**
```json
// /etc/docker/daemon.json - MUST configure log rotation
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "data-root": "/mnt/docker-data"  // Move to larger disk if needed
}
```

```bash
# Restart Docker to apply (will restart all containers on node!)
systemctl restart docker

# For Kubernetes: configure kubelet garbage collection
# /var/lib/kubelet/config.yaml
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
evictionHard:
  nodefs.available: "10%"
  imagefs.available: "15%"
```

**Tricky**: In EKS managed nodes, you can't easily SSH or modify `/etc/docker/daemon.json`. You need a custom launch template with a `userData` script that configures Docker BEFORE the node joins the cluster. Also, `docker system prune` in production can remove images that running pods need if they restart — those images would need to be pulled again, which fails if the registry is down. Only prune unused images on nodes AFTER confirming no pods reference them.

---


## Networking Production Failures

---

**Q11: Users in US-East can access your application but users in EU-West get timeouts. Both regions have the same infrastructure. What's your troubleshooting approach?**

**A:**

**Layer-by-layer debugging:**
```bash
# Layer 1: DNS Resolution
# Is EU DNS returning correct IPs?
dig +short api.company.com @8.8.8.8  # Global
dig +short api.company.com @eu-dns-resolver  # EU specific
# If Route53 geolocation/latency routing — check if EU record is correct

# Layer 2: Network Connectivity
# From EU region, check if target is reachable
traceroute api.company.com  # Look for packet loss or high latency hops
mtr --report api.company.com  # Better — shows packet loss per hop

# Layer 3: Certificate/TLS Issues
openssl s_client -connect api.company.com:443 -servername api.company.com
# Check: Is cert valid? Is it the right cert? SNI working?

# Layer 4: Load Balancer health
aws elbv2 describe-target-health --target-group-arn $EU_TG_ARN --region eu-west-1
# Are EU targets healthy?

# Layer 5: Security Groups / NACLs
aws ec2 describe-network-acls --region eu-west-1
# NACLs are STATELESS — check both inbound AND outbound rules
# Common mistake: outbound rule blocking ephemeral ports (1024-65535)
```

**Common production causes:**
```
1. Route53 health check failed → EU traffic sent to US (increased latency + timeout)
2. EU certificate expired (different cert per region, EU one not renewed)
3. EU ALB targets all unhealthy (deployment failed in EU only)
4. NACL blocking traffic (added new rule in EU, forgot ephemeral ports)
5. Security group referencing wrong source (US-East SG ID doesn't exist in EU-West)
6. Cross-region peering route table missing (if using Transit Gateway)
7. EU-West NAT Gateway exhausted — port allocation limit (55,000 per IP)
```

**Tricky**: NAT Gateway port exhaustion is a SILENT killer. Each NAT GW supports 55,000 simultaneous connections per destination IP. If your EU services all call one external API through a single NAT GW, you hit this limit during peak hours. CloudWatch metric: `ErrorPortAllocation`. Fix: Add more NAT GW IPs or use multiple NAT GWs across AZs.

---

**Q12: After a production deployment, your application can connect to the RDS database from some pods but not others. Same deployment, same config, same security groups. What's happening?**

**A:**

**The non-obvious causes:**
```bash
# Cause 1: DNS caching
# Some pods cached the OLD RDS endpoint IP (after failover/modification)
kubectl exec pod-working -- nslookup prod-db.xyz.rds.amazonaws.com
kubectl exec pod-failing -- nslookup prod-db.xyz.rds.amazonaws.com
# Different IPs? DNS TTL issue!
# Fix: Set ndots and DNS TTL in pod spec

# Cause 2: Subnet routing differences
# Pods on different nodes, nodes in different subnets
kubectl get pod pod-working -o wide  # Check NODE
kubectl get pod pod-failing -o wide   # Different node?
# Subnet A has route to RDS, Subnet B doesn't

# Cause 3: Network Policy blocking specific pods
kubectl get networkpolicy -n production
# A new NetworkPolicy might allow only certain pod labels

# Cause 4: RDS Security Group allows specific CIDR
# Node A is in 10.0.1.0/24 (allowed), Node B is in 10.0.3.0/24 (NOT allowed)
aws ec2 describe-security-groups --group-ids sg-xxxx \
  --query 'SecurityGroups[].IpPermissions[].IpRanges'

# Cause 5: Connection pool state
# Working pods had connections BEFORE a change
# Failing pods try to create NEW connections (which are now blocked)
# Long-lived connections survive SG changes!
```

**Actual resolution:**
```bash
# Verify with temporary debug pod in same namespace
kubectl run debug-pod --image=postgres:15 --rm -it -- \
  psql "host=prod-db.xyz.rds.amazonaws.com dbname=mydb user=app sslmode=require"

# If debug pod works → it's application-level (connection string, secrets)
# If debug pod fails → it's infrastructure-level (networking, SG, NACL)

# Check secrets are mounted correctly in failing pods
kubectl exec pod-failing -- env | grep DATABASE
kubectl exec pod-working -- env | grep DATABASE
# Secret might have been updated but pod not restarted!
```

**Tricky**: Security Group changes apply to NEW connections only. Existing TCP connections are NOT terminated when you modify an SG. So if you REMOVE an allow rule, pods with existing connections keep working until they close the connection. New pods (or pods that restart) can't connect. This creates the "some pods work, some don't" mystery.

---


## Deployment Gone Wrong

---

**Q13: You deployed a new version using a canary strategy (5% traffic). Metrics look fine — no errors, normal latency. You promote to 100%. Within 10 minutes, the application is overwhelmed and crashes. What went wrong?**

**A:**

**Why canary was misleading:**
```
1. CACHE WARMING: 5% traffic = most requests still hitting warm caches
   100% traffic on new version = cold cache stampede
   All requests hit database simultaneously = DB overload

2. CONNECTION POOLS: 5% = few new connections
   100% = ALL pods need new connections simultaneously (thundering herd)

3. RESOURCE SCALING: At 5%, autoscaler didn't trigger
   At 100%, traffic exceeds current capacity, autoscaler is too slow (2-3 min)

4. STATEFUL DEPENDENCIES: New version changed session handling
   5% = few sessions migrated
   100% = all sessions invalid simultaneously = mass re-authentication

5. RATE LIMITS: External API allows 1000 req/s
   5% canary = 50 req/s (well within limit)
   100% = 1000 req/s = hitting external rate limits
```

**How to do canary PROPERLY:**
```yaml
# Progressive delivery with Flagger/Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 5
      - pause: {duration: 5m}
      - setWeight: 20       # Don't jump to 100!
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 80
      - pause: {duration: 10m}
      # Monitor at EACH step before proceeding
      analysis:
        templates:
        - templateName: success-rate
          args:
          - name: service-name
            value: myservice
        - templateName: latency-p99
```

**Pre-warming strategy before full promotion:**
```bash
# Warm caches BEFORE promoting
# Route internal/synthetic traffic to new version
kubectl exec -it cache-warmer -- ./warm-cache.sh --target=new-version --keys=top-10000

# Pre-scale BEFORE promoting
kubectl scale deployment myservice-new --replicas=20  # Match expected load

# Gradual connection pool warming
# Start with 5% → wait for pools to fill → 20% → etc.
```

**Tricky**: This is called the "canary trap" — canary testing at low percentages doesn't catch problems that only appear at scale. The fix: Don't jump from 5% to 100%. Use 5% → 25% → 50% → 75% → 100% with pauses AND monitoring at each step. Also, load test the new version at full scale in staging before canary.

---

**Q14: A deployment completed successfully (all pods running, health checks passing), but customers are reporting they see stale data. The new version should show updated product prices, but old prices are still displaying. No errors anywhere. What's happening?**

**A:**

**Common stale data causes post-deployment:**
```
1. CDN CACHE: CloudFront/Cloudflare serving cached responses
   New backend returns new prices, but CDN serves old cached version
   
2. BROWSER CACHE: JavaScript bundle cached in user's browser
   Old JS makes API calls to old endpoints or formats differently
   
3. APPLICATION CACHE: Redis/Memcached still has old values
   New code reads from cache before database
   
4. API GATEWAY CACHE: AWS API Gateway response caching enabled
   Returns cached responses even though backend changed
   
5. DATABASE READ REPLICA LAG: Reads go to replica
   Writes went to primary, replica hasn't caught up yet
```

**Debugging and fixing each:**
```bash
# 1. CDN Cache — Invalidate
aws cloudfront create-invalidation \
  --distribution-id E1234567 \
  --paths "/api/products/*" "/*.js" "/*.css"

# 2. Browser Cache — Force new bundle
# Deploy with hashed filenames: main.abc123.js → main.def456.js
# Or set proper Cache-Control headers:
Cache-Control: no-cache, no-store, must-revalidate  # For API responses
Cache-Control: public, max-age=31536000, immutable  # For hashed static assets

# 3. Application Cache — Flush
redis-cli -h redis.cluster.local FLUSHDB
# Or better: use versioned cache keys
# Key: "products:v2:price:123" instead of "products:price:123"

# 4. API Gateway Cache — Flush
aws apigateway flush-stage-cache --rest-api-id abc123 --stage-name prod

# 5. Read Replica Lag
aws rds describe-db-instances --db-instance-identifier prod-replica \
  --query 'DBInstances[].StatusInfos[].Normal'
# Check ReplicaLag metric in CloudWatch
```

**Production deployment checklist (cache-aware):**
```bash
#!/bin/bash
# deploy.sh - production deployment with cache management

# 1. Deploy new version
kubectl apply -f k8s/deployment.yaml

# 2. Wait for rollout
kubectl rollout status deployment/myservice --timeout=300s

# 3. Invalidate application cache
redis-cli -h $REDIS_HOST FLUSHDB

# 4. Invalidate CDN
aws cloudfront create-invalidation --distribution-id $CF_DIST --paths "/*"

# 5. Verify fresh responses
for i in {1..5}; do
  curl -s -H "Cache-Control: no-cache" https://api.company.com/products/1 | jq '.price'
done
```

**Tricky**: CDN cache invalidation isn't instant. CloudFront invalidation can take 5-10 minutes to propagate to all edge locations globally. During this window, some users see old data, some see new. If you NEED instant consistency, don't cache dynamic data at CDN — use `Cache-Control: no-store` for API responses and only cache static assets with hashed filenames.

---


## Data Loss & Recovery Scenarios

---

**Q15: Your S3 bucket containing customer uploads (profile photos, documents) was accidentally made public for 2 hours before someone noticed. 500,000 objects exposed. Walk me through the incident response.**

**A:**

**Immediate containment (first 5 minutes):**
```bash
# Step 1: REMOVE PUBLIC ACCESS IMMEDIATELY
aws s3api put-public-access-block --bucket customer-uploads \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Step 2: Remove the bucket policy that caused it
aws s3api delete-bucket-policy --bucket customer-uploads

# Step 3: Verify — test public access fails
curl -I https://customer-uploads.s3.amazonaws.com/test-file.jpg
# Should return 403 Forbidden

# Step 4: Check CloudTrail for WHO made the change
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=PutBucketPolicy \
  --start-time $(date -d '3 hours ago' --iso-8601=seconds) \
  --query 'Events[].[Username,EventTime,CloudTrailEvent]'
```

**Assessment (what was accessed):**
```bash
# Check S3 server access logs for the 2-hour window
aws s3 ls s3://customer-uploads-access-logs/2024/01/19/

# Parse logs for external access (non-company IPs)
grep -v "10\.\|172\.\|192\.168\." access_logs/* | grep "GET " > external_access.log
wc -l external_access.log  # How many objects were actually accessed?

# Check CloudTrail data events (if enabled for S3)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::S3::Object \
  --start-time "2024-01-19T14:00:00Z" \
  --end-time "2024-01-19T16:00:00Z"
```

**Incident response process:**
```
1. CONTAIN: Block public access (done ✓)
2. ASSESS: Determine what was exposed and accessed
3. NOTIFY: 
   - Legal team (GDPR/compliance — 72-hour notification requirement)
   - Security team
   - Management
   - Affected customers (if PII was accessed)
4. REMEDIATE:
   - Rotate any exposed credentials/tokens stored in S3
   - If documents contain PII: determine breach notification requirements
5. PREVENT:
   - Enable S3 Block Public Access at ACCOUNT level
   - Add SCP (Service Control Policy) to prevent public S3 in entire org
   - Enable Config Rule: s3-bucket-public-read-prohibited
   - Add Macie to detect PII in S3 buckets
```

**Prevention (account-level):**
```bash
# Block public access for ENTIRE AWS account
aws s3control put-public-access-block --account-id 123456789012 \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# SCP to prevent anyone from making buckets public
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyPublicS3",
    "Effect": "Deny",
    "Action": ["s3:PutBucketPolicy", "s3:PutBucketAcl"],
    "Resource": "*",
    "Condition": {
      "StringEquals": {"s3:x-amz-acl": ["public-read", "public-read-write"]}
    }
  }]
}
```

**Tricky**: GDPR requires breach notification within 72 hours. If your S3 bucket contained EU citizen data (even just email addresses), the clock started when the bucket was made public — NOT when you discovered it. If you discover on Monday that the breach happened Friday, you have already used 48+ hours of your 72-hour window. This is why continuous monitoring (AWS Config, GuardDuty) is legally required, not just "nice to have."

---

**Q16: Your Terraform state file got corrupted (someone ran `terraform apply` with an older state version). Now Terraform wants to destroy 30 production resources and recreate them. How do you fix this without destroying anything?**

**A:**

**DO NOT RUN APPLY! Diagnosis first:**
```bash
# Step 1: Check what Terraform thinks vs reality
terraform plan
# Output will show: "30 resources to destroy, 30 to create"
# This means state says they don't exist, but they DO exist in AWS

# Step 2: Check state file version
terraform state list  # See what's currently in state
# Compare with what actually exists in AWS

# Step 3: Check if S3 versioning saved you
aws s3api list-object-versions --bucket terraform-state-bucket \
  --prefix "prod/terraform.tfstate" --max-items 5
# If versioning enabled, you can restore the previous version!
```

**Recovery approaches (safest to riskiest):**

```bash
# APPROACH 1: Restore previous state version (safest)
aws s3api get-object --bucket terraform-state-bucket \
  --key "prod/terraform.tfstate" \
  --version-id "OLD_VERSION_ID" \
  restored-state.tfstate

# Replace current state with the good version
aws s3 cp restored-state.tfstate s3://terraform-state-bucket/prod/terraform.tfstate

# Verify
terraform plan  # Should show "No changes"

# APPROACH 2: Import missing resources back into state
# For each resource Terraform wants to destroy:
terraform import aws_instance.web i-0abc123def
terraform import aws_rds_cluster.main arn:aws:rds:...
terraform import aws_s3_bucket.data my-bucket-name
# Tedious for 30 resources but safe

# APPROACH 3: Use terraform state rm + import selectively
# Remove the "ghost" resources that state thinks exist but don't
terraform state rm aws_instance.old_ghost
# Import the real resources that state doesn't know about
terraform import aws_instance.web i-real-instance-id

# After any approach:
terraform plan  # MUST show "No changes" before you feel safe
```

**Prevention:**
```hcl
# 1. ALWAYS enable S3 versioning on state bucket
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

# 2. Enable DynamoDB locking (prevents concurrent access)
# 3. Use Terraform Cloud/Enterprise for state management
# 4. Never run Terraform locally against production — use CI/CD only
# 5. Enable "prevent_destroy" on critical resources
resource "aws_rds_cluster" "main" {
  # ...
  lifecycle { prevent_destroy = true }
}
```

**Tricky**: `lifecycle { prevent_destroy = true }` only prevents `terraform destroy`. If Terraform decides to REPLACE a resource (destroy + create), `prevent_destroy` also blocks it. This can be frustrating during upgrades. Also, if you remove the resource from your config entirely and run apply, `prevent_destroy` will error — but if someone does `terraform state rm` first, the protection is gone. It's not foolproof.

---
