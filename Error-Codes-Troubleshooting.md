# Application & Infrastructure Error Codes — Deep-Dive Interview Q&A

## Table of Contents
1. [HTTP Status Codes](#http-status-codes)
2. [AWS Infrastructure Error Codes](#aws-infrastructure-error-codes)
3. [Application-Level Errors](#application-level-errors)
4. [Troubleshooting Methodology](#troubleshooting-methodology)
5. [Tricky Scenarios](#tricky-scenarios)

---

## HTTP Status Codes


**Q1: Categorize HTTP status codes. What does each category mean?**

**A:**

| Range | Category | Meaning |
|-------|----------|---------|
| 1xx | Informational | Request received, continuing |
| 2xx | Success | Request successful |
| 3xx | Redirection | Further action needed |
| 4xx | Client Error | Problem with request (client's fault) |
| 5xx | Server Error | Server failed to fulfill valid request (server's fault) |

**Critical codes to know:**

| Code | Name | Common Cause | Who Fixes |
|------|------|-------------|-----------|
| 200 | OK | Success | N/A |
| 201 | Created | Resource created (POST) | N/A |
| 204 | No Content | Success, no body (DELETE) | N/A |
| 301 | Moved Permanently | URL changed forever, cached by browsers | DevOps |
| 302 | Found (Temp Redirect) | Temporary redirect | DevOps |
| 304 | Not Modified | Cache valid (conditional GET) | N/A |
| 400 | Bad Request | Malformed request syntax | Developer |
| 401 | Unauthorized | No/invalid authentication | Developer |
| 403 | Forbidden | Authenticated but not authorized | DevOps/Dev |
| 404 | Not Found | Resource doesn't exist | Developer |
| 405 | Method Not Allowed | Wrong HTTP method (POST to GET-only) | Developer |
| 408 | Request Timeout | Client took too long to send | Network |
| 413 | Payload Too Large | Body exceeds limit | DevOps/Dev |
| 429 | Too Many Requests | Rate limited/throttled | DevOps |
| 500 | Internal Server Error | Unhandled exception | Developer |
| 502 | Bad Gateway | Proxy/LB got invalid response from upstream | DevOps |
| 503 | Service Unavailable | Overloaded or maintenance | DevOps |
| 504 | Gateway Timeout | Upstream didn't respond in time | DevOps |

---

**Q2: Explain the difference between 401 vs 403, and 502 vs 504. These are the most confused pairs.**

**A:**

**401 Unauthorized vs 403 Forbidden:**
```
401: "Who are you?" — No credentials provided or credentials invalid
     → Solution: Provide valid authentication (token, API key, login)
     → Should include WWW-Authenticate header in response

403: "I know who you are, but you can't do this" — Authenticated but not authorized
     → Solution: Request permission/role change, or access different resource
     → Re-authenticating won't help (credentials are valid, permissions aren't)
```

**502 Bad Gateway vs 504 Gateway Timeout:**
```
502: Proxy/LB received INVALID response from upstream server
     → Backend crashed, returned malformed HTTP, or closed connection unexpectedly
     → Check: Application logs, backend health, response format

504: Proxy/LB received NO response from upstream (timed out)
     → Backend is too slow or unresponsive
     → Check: Backend processing time, timeout settings, network connectivity
```

**Tricky**: In AWS:
- ALB 502 → Target returned invalid response OR connection refused
- ALB 504 → Target didn't respond within idle timeout (default 60s)
- CloudFront 502 → Origin returned error OR SSL handshake failed
- API Gateway 504 → Lambda/backend exceeded 29s timeout

---

## AWS Infrastructure Error Codes

**Q3: Explain common AWS ALB/NLB error codes and their causes.**

**A:**

**ALB-specific errors:**

| Code | ALB Cause | Solution |
|------|-----------|----------|
| 400 | Request malformed (invalid header, URI too long) | Fix client request format |
| 401 | Authentication action failed (OIDC/Cognito) | Check IdP configuration |
| 403 | WAF blocked request | Review WAF rules |
| 460 | Client closed connection before LB could respond | Client timeout too short |
| 463 | X-Forwarded-For has too many IPs (>30) | Reduce proxy chain |
| 500 | ALB internal error | AWS issue (rare) |
| 502 | Target closed connection or returned invalid response | Fix target app |
| 503 | No registered targets or all targets unhealthy | Check target health |
| 504 | Target didn't respond within timeout | Increase idle timeout or optimize backend |
| 561 | IdP returned error during authentication | Check OIDC/Cognito config |

**NLB-specific behaviors:**
- NLB doesn't generate HTTP errors (it's Layer 4)
- Connection timeout = TCP RST to client
- Unhealthy target = connection refused or timeout

**How to diagnose:**
```bash
# Enable ALB access logs (to S3)
# Fields: target_processing_time, response_processing_time, elb_status_code, target_status_code

# target_status_code = "-" → ALB couldn't connect to target (502)
# target_processing_time = "-" → No target received the request (503)
```

---

**Q4: Explain CloudFront error codes. What do 4xx vs 5xx mean from CloudFront specifically?**

**A:**

| Code | CloudFront Meaning | Root Cause |
|------|-------------------|-----------|
| 400 | Bad request | Request exceeds CloudFront limits (URI > 8192 chars) |
| 403 | Forbidden | Geo-restriction, WAF block, OAC denied, signed URL invalid |
| 404 | Not found | Object not in S3/origin |
| 405 | Method not allowed | HTTP method not allowed for behavior |
| 414 | URI too long | Request URI exceeds limit |
| 500 | Internal error | CloudFront internal issue |
| 502 | Bad Gateway | Origin returned invalid response, SSL mismatch, origin unreachable |
| 503 | Service unavailable | CloudFront capacity issue (rare) or origin overloaded |
| 504 | Gateway timeout | Origin didn't respond within timeout (30s default) |

**CloudFront 502 deep dive (most common):**
1. Origin SSL certificate invalid/expired/self-signed
2. Origin not listening on expected port
3. Origin security group blocks CloudFront IPs
4. Origin returned response that CloudFront can't parse
5. DNS resolution failure for custom origin domain

**Custom Error Responses (return friendly pages):**
```json
{
  "ErrorCode": 404,
  "ResponseCode": 200,
  "ResponsePagePath": "/error/404.html",
  "ErrorCachingMinTTL": 300
}
```

---

**Q5: Explain common API Gateway error codes and their solutions.**

**A:**

| Error | Message | Cause | Solution |
|-------|---------|-------|----------|
| 403 | Forbidden | Resource policy, WAF, missing API key | Check resource policy, WAF rules |
| 403 | Missing Authentication Token | Wrong URL/method (resource doesn't exist) | Verify URL path and HTTP method |
| 401 | Unauthorized | Cognito/IAM auth failed | Check auth token/credentials |
| 429 | Too Many Requests | Throttled (account/stage/usage plan limit) | Increase throttle limit or implement backoff |
| 500 | Internal Server Error | Integration error (Lambda crash, mapping template error) | Check Lambda logs, mapping templates |
| 502 | Bad Gateway | Lambda returned invalid response format | Ensure Lambda returns {statusCode, body, headers} |
| 503 | Service Unavailable | AWS throttling or service issue | Retry with backoff |
| 504 | Endpoint Request Timed Out | Lambda exceeded 29s or backend timeout | Optimize Lambda, use async pattern |

**API Gateway 502 (most common interview question):**
```python
# BAD - Lambda response causes 502
def handler(event, context):
    return "Hello"  # String not valid! Must be dict with statusCode

# GOOD - Proper response format
def handler(event, context):
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"message": "Hello"})
    }
```

---

**Q6: Explain ECS/Fargate common errors and their solutions.**

**A:**

| Error | Cause | Solution |
|-------|-------|----------|
| CannotPullContainerError | ECR access denied, no NAT/VPC endpoint | Add ECR permissions, fix networking |
| ResourceNotFoundException | Task definition doesn't exist | Check family name and revision |
| STOPPED (Essential container exited) | Main container crashed | Check container logs in CloudWatch |
| STOPPED (OutOfMemory) | Container exceeded memory limit | Increase task memory or fix memory leak |
| PENDING (no capacity) | No EC2 instances available (EC2 type) | Scale ASG or use Capacity Providers |
| Service unhealthy (health check failed) | Health check endpoint not responding | Fix health check path/port/timing |
| CannotStartContainerError | Entrypoint/CMD error or port conflict | Check Dockerfile ENTRYPOINT, port mappings |

**Task Stopped Reason debugging:**
```bash
# Check why task stopped
aws ecs describe-tasks --cluster my-cluster --tasks <task-arn> \
  --query 'tasks[0].{reason:stoppedReason,containers:containers[*].{name:name,reason:reason,exitCode:exitCode}}'
```

---

## Application-Level Errors

**Q7: Common application errors and their infra vs code causes:**

**A:**

| Symptom | Infra Cause | Code Cause |
|---------|-------------|-----------|
| Connection timeout | Security group, NACL, route table | App not listening on port |
| Connection refused | Service not running, wrong port | App crashed on startup |
| Intermittent 500s | Memory pressure, CPU throttle | Unhandled exceptions, race conditions |
| Slow responses | Disk I/O, network latency, DNS | N+1 queries, no caching, memory leaks |
| SSL errors | Certificate expired, wrong CN | Wrong SSL library version |
| DNS resolution failure | Route53 issue, VPC DHCP config | Hardcoded IPs in code |
| Out of memory | Container/instance too small | Memory leak, unbounded caching |
| Disk full | EBS/EFS volume full | Logs not rotated, temp files not cleaned |

---

**Q8: How do you systematically troubleshoot a "website is down" incident?**

**A:**

```
Layer-by-layer diagnosis (outside-in):

1. DNS Layer:
   $ dig example.com
   → Is DNS resolving? Correct IP?
   → Check Route53 health checks

2. Network Layer:
   $ curl -I https://example.com
   → Can you reach the endpoint?
   → Check CloudFront/ALB status

3. Load Balancer Layer:
   → Check ALB target group health
   → Check 5xx error rate in CloudWatch
   → Review ALB access logs

4. Application Layer:
   → Check application logs (CloudWatch Logs)
   → Check container/instance health
   → Memory/CPU metrics

5. Database Layer:
   → RDS connections count
   → Slow query log
   → Storage full?

6. External Dependencies:
   → Third-party API status
   → AWS service health dashboard
```

**Quick checklist:**
```bash
# 1. Is it DNS?
dig +short example.com

# 2. Is it network?
curl -o /dev/null -s -w "%{http_code} %{time_total}s\n" https://example.com

# 3. Is it the LB?
aws elbv2 describe-target-health --target-group-arn <arn>

# 4. Is it the app?
aws logs tail /ecs/my-app --since 5m

# 5. Is it resources?
aws cloudwatch get-metric-statistics --namespace AWS/ECS --metric-name CPUUtilization ...
```

---

## Tricky Scenarios

**Q9: Users report "504 Gateway Timeout" but only during peak hours. Infrastructure looks healthy. What's happening?**

**A:**

**Possible causes:**

1. **Database connection pool exhaustion:**
   - App waits for DB connection → exceeds LB timeout → 504
   - Check: RDS `DatabaseConnections` metric vs max_connections
   - Fix: Connection pooling (RDS Proxy), increase pool size

2. **Lambda cold starts + API Gateway 29s limit:**
   - Cold start + execution > 29s → API GW returns 504
   - Fix: Provisioned concurrency, optimize cold start

3. **Downstream service slow under load:**
   - Microservice calls another service that's overloaded
   - Fix: Circuit breaker pattern, timeouts, async processing

4. **Thread/worker pool exhaustion:**
   - All workers busy handling slow requests → new requests queue → timeout
   - Fix: Increase workers, add request timeout, async processing

5. **NAT Gateway bandwidth limit:**
   - NAT GW: 45 Gbps max, but connections have limits
   - Fix: Multiple NAT GWs, VPC endpoints for AWS services

**Debugging approach:**
```
Check metrics in order:
1. ALB target response time → If high, backend is slow
2. ECS CPU/Memory → If maxed, scale out
3. RDS connections/CPU → If maxed, DB bottleneck
4. VPC Flow Logs → If drops, network issue
```

---

**Q10: Application returns 200 but with empty/incorrect response body. Status code is misleading. How do you handle this?**

**A:**

**This is a "silent failure" — hardest to detect:**

1. **Health check passes but app is broken:**
   - Health endpoint returns 200 but main functionality is down
   - Fix: Deep health checks that verify database, cache, dependencies
   ```json
   GET /health/deep
   {
     "status": "degraded",
     "database": "connected",
     "cache": "unreachable",
     "api_v2": "timeout"
   }
   ```

2. **Try-catch swallowing errors:**
   ```python
   # BAD - hides errors
   try:
       result = process_data()
   except Exception:
       return {"statusCode": 200, "body": ""}  # Silent failure!
   
   # GOOD - return proper error
   except Exception as e:
       logger.error(f"Processing failed: {e}")
       return {"statusCode": 500, "body": json.dumps({"error": str(e)})}
   ```

3. **Monitoring empty responses:**
   - CloudWatch custom metric: Track response body size
   - ALB access logs: Filter responses with body size = 0
   - Synthetic monitoring: Validate response content, not just status code

4. **Circuit breaker pattern:**
   - If downstream returns empty responses, trip breaker
   - Return 503 with meaningful error instead of empty 200

---

**Q11: Explain the "thundering herd" problem and cache stampede. How do you prevent them?**

**A:**

**Thundering Herd**: Cache expires → ALL requests simultaneously hit origin → origin overwhelmed

**Cache Stampede**: Popular cached item expires → hundreds of concurrent requests try to regenerate cache simultaneously

```
Normal: 10,000 RPS → Cache (99% hits) → Origin: 100 RPS
Cache expires: 10,000 RPS → ALL hit origin → Origin: 10,000 RPS → CRASH

Solutions:

1. Staggered TTL (jitter):
   TTL = base_ttl + random(0, 60)  # Different expiry times

2. Cache-aside with lock:
   if cache.get(key) is None:
       if acquire_lock(key):  # Only one request regenerates
           value = fetch_from_db()
           cache.set(key, value)
           release_lock(key)
       else:
           wait_for_cache()  # Others wait

3. Background refresh (proactive):
   TTL = 3600, Refresh_at = 3000  # Refresh 10 min before expiry
   Background worker refreshes cache before it expires

4. Stale-while-revalidate:
   Serve stale cache while regenerating in background
   CloudFront: Use Origin Shield (single request to origin)
   
5. Request collapsing:
   Multiple identical requests → only ONE forwarded to origin
   CloudFront does this automatically (request collapsing)
```

---

**Q12: What Linux/system-level errors cause application issues? How do you diagnose?**

**A:**

| Error/Symptom | System Cause | Diagnostic Command |
|---------------|-------------|-------------------|
| Connection refused | Process not running, port not bound | `ss -tlnp \| grep <port>` |
| Connection timeout | Firewall, routing, overloaded | `iptables -L`, `traceroute` |
| Too many open files | File descriptor limit | `ulimit -n`, `ls /proc/<pid>/fd \| wc -l` |
| Out of memory | OOM killer triggered | `dmesg \| grep -i oom`, `journalctl -k` |
| Disk full | Logs, temp files, undeleted files | `df -h`, `du -sh /*`, `lsof +L1` |
| CPU 100% | Process stuck, crypto mining | `top`, `perf top` |
| Load average high | Too many processes, I/O wait | `uptime`, `iostat`, `vmstat 1` |
| Permission denied | File permissions, SELinux | `ls -la`, `getenforce`, `ausearch -m AVC` |
| Cannot allocate memory | Virtual memory exhausted | `free -h`, `sysctl vm.overcommit_memory` |
| Network unreachable | Route missing, interface down | `ip route`, `ip link show` |

**Quick system health check:**
```bash
# One-liner health check
echo "=== Load ===" && uptime && \
echo "=== Memory ===" && free -h && \
echo "=== Disk ===" && df -h && \
echo "=== Top processes ===" && ps aux --sort=-%mem | head -5 && \
echo "=== Network ===" && ss -s && \
echo "=== OOM ===" && dmesg | grep -i "oom\|killed" | tail -5
```
