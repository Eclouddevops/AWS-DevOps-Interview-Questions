# AWS Lambda + API Gateway + DDoS + WAF — Deep-Dive Interview Q&A

## Table of Contents
1. [AWS Lambda — Deep Dive](#1-aws-lambda--deep-dive)
2. [API Gateway — Deep Dive](#2-api-gateway--deep-dive)
3. [DDoS Protection (AWS Shield)](#3-ddos-protection-aws-shield)
4. [AWS WAF — Deep Dive](#4-aws-waf--deep-dive)
5. [Integration & Architecture Questions](#5-integration--architecture-questions)
6. [Tricky Scenario-Based Questions](#6-tricky-scenario-based-questions)

---

## 1. AWS Lambda — Deep Dive

### Basic to Intermediate


**Q1: What is a cold start in Lambda, and what factors affect it?**

**A:** A cold start occurs when AWS must provision a new execution environment (micro VM) for your function. Factors that affect cold start duration:

- **Runtime**: JVM-based (Java, .NET) have longer cold starts (~1-3s) vs. Python/Node.js (~100-300ms)
- **Package size**: Larger deployment packages increase cold start time
- **VPC attachment**: ENI provisioning adds latency (greatly improved with Hyperplane ENI since 2019)
- **Memory allocation**: More memory = more CPU = faster initialization
- **Number of dependencies**: More imports/modules = slower init
- **Provisioned Concurrency**: Eliminates cold starts by pre-warming environments

---


**Q2: Explain the Lambda execution lifecycle in detail.**

**A:**

```
INIT Phase → INVOKE Phase → SHUTDOWN Phase

INIT Phase (cold start only):
├── Extension Init (up to 10s)
├── Runtime Init (up to 10s)
└── Function Init (handler code outside handler function, up to 10s)

INVOKE Phase:
└── Handler execution (timeout applies)

SHUTDOWN Phase:
├── Runtime shutdown hook (up to 2s)
└── Extension shutdown (up to 2s)
```

**Key insight**: Code *outside* the handler runs during INIT and is reused across warm invocations. This is where you initialize DB connections, SDK clients, etc.

---


**Q3: What's the difference between synchronous and asynchronous invocation? What about event source mappings?**

**A:**

| Invocation Type | Retry Behavior | Error Handling | Examples |
|----------------|---------------|----------------|----------|
| **Synchronous** | No automatic retry; caller handles | Error returned to caller | API Gateway, ALB, SDK `Invoke` |
| **Asynchronous** | 2 automatic retries (total 3 attempts) | DLQ or On-Failure Destination | S3, SNS, EventBridge |
| **Event Source Mapping** | Retries until record expires (for streams) | Bisect batch, max retry, on-failure destination | SQS, Kinesis, DynamoDB Streams |

**Tricky follow-up**: With asynchronous invocation, Lambda places events into an internal queue. If the function errors, it retries after 1 minute, then 2 minutes. You can configure `MaximumRetryAttempts` (0-2) and `MaximumEventAgeInSeconds`.

---


**Q4: How does Lambda concurrency work? Explain reserved vs. provisioned concurrency.**

**A:**

- **Account-level limit**: 1,000 concurrent executions (soft limit, can be increased)
- **Reserved Concurrency**: Guarantees a fixed number of concurrent executions for a function AND caps it at that number. Other functions cannot steal this capacity.
- **Provisioned Concurrency**: Pre-initializes a specified number of execution environments (no cold starts). Charged even when idle.

**Tricky details:**
- Reserved concurrency of 0 = function is throttled (disabled)
- Reserved concurrency acts as both a floor (guaranteed) and a ceiling (max)
- Unreserved concurrency = Account limit - sum of all reserved concurrency
- Provisioned concurrency cannot exceed reserved concurrency (if set)

---


### Advanced / Tricky Lambda Questions

**Q5: You have a Lambda function connected to an SQS queue. Messages are failing and retrying infinitely. Why?**

**A:** With SQS as an event source:
- Lambda polls the queue and invokes your function with a batch
- If the function throws an error, the **entire batch** goes back to the queue after visibility timeout expires
- This creates an infinite loop if the error is persistent (poison pill message)

**Solutions:**
1. Configure a Dead Letter Queue (DLQ) on the SQS queue with `maxReceiveCount`
2. Use `ReportBatchItemFailures` — return only failed message IDs so successful ones aren't reprocessed
3. Implement try/catch per message within the batch

```python
def handler(event, context):
    batch_item_failures = []
    for record in event['Records']:
        try:
            process(record)
        except Exception:
            batch_item_failures.append({"itemIdentifier": record['messageId']})
    return {"batchItemFailures": batch_item_failures}
```

---


**Q6: What happens when Lambda hits a concurrency limit? How does it differ for sync vs async vs event source mapping?**

**A:**

- **Synchronous**: Returns 429 (TooManyRequestsException) immediately to caller
- **Asynchronous**: Throttled events are retried for up to 6 hours with backoff. If still failing after max age, goes to on-failure destination/DLQ
- **Event Source Mapping (Streams - Kinesis/DynamoDB)**: Lambda retries the entire batch (blocks the shard) until success, record expiry, or max retry. Does NOT lose data because the stream retains records
- **Event Source Mapping (SQS)**: Throttled messages return to queue after visibility timeout

---


**Q7: Explain Lambda@Edge vs CloudFront Functions. When would you choose one over the other?**

**A:**

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| **Runtime** | JavaScript only | Node.js, Python |
| **Execution time** | < 1ms | Up to 5s (viewer) / 30s (origin) |
| **Memory** | 2MB | 128-10240 MB |
| **Network access** | No | Yes |
| **Body access** | No | Yes (origin events) |
| **Triggers** | Viewer Request/Response only | All 4 CloudFront events |
| **Scale** | Millions of req/s | Thousands of req/s |
| **Price** | ~1/6th of Lambda@Edge | Higher |

**Use CloudFront Functions for**: URL rewrites, header manipulation, cache key normalization, simple A/B testing, simple JWT validation

**Use Lambda@Edge for**: Origin selection, complex auth, body manipulation, making network calls

---


**Q8: A Lambda function runs fine locally but times out in production. The function connects to an RDS database in a private subnet. What's happening?**

**A:** Classic VPC networking issue. Checklist:

1. Lambda is in a private subnet — does it have a route to the internet via NAT Gateway? (needed for AWS service calls)
2. Security group on Lambda's ENI — does it allow outbound to RDS port?
3. RDS security group — does it allow inbound from Lambda's security group?
4. **Most tricky**: If Lambda is in a **public** subnet, it still cannot access the internet because Lambda ENIs don't get public IPs regardless of subnet settings. It MUST be in a private subnet with a NAT Gateway.
5. Alternatively, use VPC Endpoints for AWS services to avoid NAT Gateway costs
6. DNS resolution — ensure the VPC has DNS resolution enabled

---


**Q9: How does Lambda handle database connections efficiently?**

**A:**

- **Problem**: Each execution environment opens its own connection. At scale (1000 concurrent Lambda = 1000 connections), you can exhaust the DB connection pool.
- **Solutions**:
  - **RDS Proxy**: Manages connection pooling, multiplexes Lambda connections to fewer DB connections. Supports IAM auth.
  - **Initialize connection outside handler**: Reused across warm invocations
  - **Provisioned Concurrency**: Limits max environments but adds cost
  - **Reserved Concurrency**: Caps concurrent Lambda = caps max connections

```python
import pymysql

# Connection initialized outside handler - reused in warm starts
conn = pymysql.connect(host='...', user='...', password='...')

def handler(event, context):
    with conn.cursor() as cur:
        cur.execute("SELECT ...")
        return cur.fetchall()
```

---


**Q10: What is Lambda Destinations vs DLQ? Why does AWS recommend Destinations?**

**A:**

- **DLQ** (SQS/SNS): Only captures **failed** async invocations. Limited metadata.
- **Destinations**: Can route both **success** and **failure** async results to SQS, SNS, Lambda, or EventBridge. Includes full invocation record (request payload, response, error, timestamps).

**DLQ limitations**: Only supports SQS/SNS; only on failure; minimal metadata. Destinations is strictly more powerful and flexible. DLQ is still supported for backward compatibility.

---


**Q11: What are Lambda Layers? What are the limits and best practices?**

**A:**

Lambda Layers allow you to share code and dependencies across multiple functions.

- **Limits**: Max 5 layers per function, total unzipped deployment package (function + layers) must be < 250 MB
- **Use cases**: Shared libraries, custom runtimes, common utility code
- **Versioning**: Layers are versioned and immutable. Functions reference a specific layer version.

**Best Practices:**
- Use layers for large dependencies (e.g., pandas, numpy) to keep function code small
- Pin layer versions in production (don't use "latest")
- Layer code is extracted to `/opt` directory at runtime
- Be cautious with version drift across functions using the same layer

**Tricky**: Layers don't reduce cold start time — the total unzipped size still matters. They only help with code organization and deployment size.

---

## 2. API Gateway — Deep Dive

### Core Concepts


**Q12: Explain the three types of API Gateway. When would you use each?**

**A:**

| Feature | REST API | HTTP API | WebSocket API |
|---------|----------|----------|---------------|
| **Cost** | Higher (~$3.50/million) | Lower (~$1.00/million) | Per-message pricing |
| **Latency** | Higher (~30ms overhead) | Lower (~10ms overhead) | Persistent connections |
| **Features** | Full (caching, WAF, usage plans, request validation, private APIs) | Minimal (JWT auth, CORS, OIDC) | Real-time bidirectional |
| **Auth** | IAM, Cognito, Lambda authorizer, API keys | IAM, JWT (native), Lambda | IAM, Lambda authorizer |
| **WAF support** | Yes | No | No |
| **Use case** | Enterprise APIs, caching, complex auth | Simple APIs, microservices, low-latency | Chat, notifications, gaming |

**Tricky**: HTTP API doesn't support WAF, usage plans, API keys, request/response transformation, or caching. If you need any of these, you must use REST API.

---


**Q13: Explain the API Gateway request/response flow for a REST API.**

**A:**

```
Client Request
    ↓
Method Request (auth, validation, API key check)
    ↓
Integration Request (mapping template, VTL transformation)
    ↓
Backend (Lambda, HTTP, AWS service, Mock)
    ↓
Integration Response (mapping template, status code mapping)
    ↓
Method Response (response models, headers)
    ↓
Client Response
```

Each stage can transform data. VTL (Velocity Template Language) is used in mapping templates for REST API. HTTP API uses simpler payload format versions (1.0/2.0).

---


**Q14: What are API Gateway throttling limits and how do they work?**

**A:**

- **Account-level**: 10,000 requests/second (steady-state), 5,000 burst (token bucket algorithm)
- **Stage-level**: Configurable per stage
- **Method-level**: Configurable per resource/method
- **Usage Plans**: Per API key throttling + quota

**Token Bucket Algorithm**:
- Bucket fills at the steady-state rate
- Burst capacity = bucket size
- Each request removes a token
- If bucket is empty → 429 Too Many Requests

**Tricky**: The 10,000 RPS is shared across ALL APIs in an account/region. A noisy API can throttle other APIs in the same account!

---


**Q15: How does API Gateway caching work? What are the gotchas?**

**A:**

- Cache is per-stage (not per method by default, but configurable per method)
- Cache size: 0.5GB to 237GB
- TTL: 0-3600 seconds (default 300s)
- Cache key: Full request URL by default; can include headers, query strings, path params
- Invalidation: `Cache-Control: max-age=0` header (requires IAM authorization or `Require authorization for cache control` disabled)
- **Cost**: Charged per hour based on cache size, even if empty

**Gotchas**:
1. Caching is only available for REST APIs (not HTTP APIs)
2. Cache invalidation requires IAM auth by default — clients without IAM can't flush cache
3. If you override method-level caching to "off," it uses the stage default. Set TTL=0 explicitly to disable.
4. Cache is not shared across stages or deployments

---


**Q16: Explain Lambda Authorizers (Custom Authorizers). What are the two types?**

**A:**

**Token-based (TOKEN type)**:
- Receives a bearer token (e.g., JWT from Authorization header)
- Returns an IAM policy document
- Token source specified in configuration

**Request-based (REQUEST type)**:
- Receives full request context (headers, query strings, path params, stage variables)
- Returns IAM policy document
- Can make decisions based on multiple request parameters

**Response format** (both types):
```json
{
  "principalId": "user123",
  "policyDocument": {
    "Version": "2012-10-17",
    "Statement": [{
      "Action": "execute-api:Invoke",
      "Effect": "Allow",
      "Resource": "arn:aws:execute-api:{region}:{account}:{api-id}/{stage}/{method}/{resource}"
    }]
  },
  "context": {
    "customKey": "customValue"
  }
}
```

**Caching behavior**:
- TOKEN authorizer: Cached by token value (TTL 0-3600s)
- REQUEST authorizer: Cached by ALL specified identity sources
- **Tricky**: If caching is enabled and the authorizer returns `Allow` for `arn:.../*`, ALL subsequent requests with the same token will be allowed without re-invoking the authorizer!

---


### Tricky API Gateway Questions

**Q17: You deploy an API but clients get 403 Forbidden. The Lambda function isn't being invoked at all. What's wrong?**

**A:** Common causes (in order of likelihood):

1. **Missing resource-based policy**: API Gateway doesn't have permission to invoke the Lambda. Need `lambda:InvokeFunction` permission.
2. **Incorrect stage deployment**: Changes made but stage not redeployed.
3. **WAF blocking**: If WAF is attached, a rule might be blocking.
4. **Resource policy on API**: IP-based or VPC-based access control blocking.
5. **API key required but not provided**: If method requires API key, `x-api-key` header is missing.
6. **Lambda authorizer returning Deny**: Authorizer runs before Lambda integration.

**Tricky diagnostic tips**:
- `{"message": "Forbidden"}` → Typically a resource policy issue
- `{"message": "Missing Authentication Token"}` → The resource/method doesn't exist (wrong URL or method not deployed)
- `{"message": "Unauthorized"}` → Authentication issue (Cognito/IAM)

---


**Q18: What's the maximum timeout for API Gateway integrations? How do you handle long-running operations?**

**A:**

- **REST API**: 29 seconds maximum (hard limit, cannot be increased)
- **HTTP API**: 30 seconds maximum
- **WebSocket API**: 29 seconds for route responses, 2 hours idle connection

**For long-running operations, use:**
1. **Async pattern**: API Gateway → Lambda (start execution, return task ID) → Client polls for status
2. **Step Functions integration**: Direct integration with Express (sync, max 5 min) or Standard (async) workflows
3. **WebSocket**: Push result to client when done
4. **SQS integration**: Direct API Gateway → SQS integration (no Lambda), return 200 immediately

---


**Q19: Explain the difference between Stage Variables and Environment Variables in the context of API Gateway + Lambda.**

**A:**

- **Stage Variables**: Defined in API Gateway, accessible via `$stageVariables` in mapping templates. Use cases: point different stages to different Lambda aliases, different backend URLs.
- **Lambda Environment Variables**: Defined on the Lambda function, accessible in function code via `os.environ`.

**Powerful pattern:**
```
API Gateway Stage: prod → Stage Variable: lambdaAlias = prod
API Gateway Stage: dev  → Stage Variable: lambdaAlias = dev

Integration URI: arn:aws:lambda:...:function:myFunc:${stageVariables.lambdaAlias}
```
This lets one API deployment point to different Lambda versions per stage.

---


**Q20: What is API Gateway payload format version 1.0 vs 2.0?**

**A:**

- **Version 1.0** (REST API default): Detailed event object with `httpMethod`, `resource`, `pathParameters`, `queryStringParameters`, `headers`, `body`, `requestContext`
- **Version 2.0** (HTTP API default): Simplified format with flattened structure, `rawPath`, `rawQueryString`, `cookies` array, `requestContext.http`

**Key differences in 2.0:**
- Multi-value headers/query params automatically handled
- Cookies in dedicated array
- Response can be simple string (auto-wrapped in 200 with JSON content-type)
- Lambda can return just a string/object instead of full `{statusCode, body, headers}` format

**Tricky**: If you migrate from REST API to HTTP API, your Lambda handlers may break due to different event formats!

---

## 3. DDoS Protection (AWS Shield)


**Q21: Explain AWS Shield Standard vs Shield Advanced.**

**A:**

| Feature | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **Cost** | Free (all AWS customers) | $3,000/month + data transfer |
| **Protection** | Layer 3/4 automatic | Layer 3/4/7 |
| **Applies to** | All AWS resources automatically | CloudFront, ALB, NLB, EIP, Global Accelerator, Route 53 |
| **DDoS Response Team** | No | Yes (24/7 SRT access) |
| **Cost Protection** | No | Yes (scaling credits during attacks) |
| **Visibility** | Basic CloudWatch | Real-time metrics, attack forensics, War Room |
| **WAF included** | No | WAF charges waived for protected resources |
| **Health-based detection** | No | Yes (uses Route 53 health checks) |
| **Automatic mitigation** | Layer 3/4 only | Layer 3/4 + optional automatic Layer 7 (with WAF) |

---


**Q22: How does Shield Advanced provide Layer 7 DDoS protection?**

**A:** Shield Advanced doesn't block Layer 7 attacks by itself. It works WITH WAF:

1. Shield Advanced detects anomalies in traffic patterns (baseline established over time)
2. DDoS Response Team (SRT) can create/update WAF rules on your behalf
3. **Automatic application layer DDoS mitigation**: Shield Advanced automatically creates/manages WAF rules when it detects Layer 7 attacks
4. Shield Advanced provides rate-based rules as first line of defense

**Tricky**: Without WAF associated, Shield Advanced only helps with Layer 3/4 for CloudFront/ALB. You MUST have WAF attached to get Layer 7 automated mitigation.

---


**Q23: Your application is behind CloudFront + ALB + Lambda. Describe the DDoS protection at each layer.**

**A:**

```
Internet → CloudFront → ALB → Lambda

CloudFront:
├── Shield Standard: Automatic SYN flood, UDP reflection protection
├── Shield Advanced: Traffic anomaly detection, SRT access
├── Geographic distribution absorbs volumetric attacks
├── WAF: Rate limiting, IP reputation, geo-blocking
└── Anycast network (hundreds of edge locations absorb traffic)

ALB:
├── Shield Standard: Basic Layer 3/4
├── Shield Advanced: DDoS detection + cost protection
├── Security Groups: Layer 4 filtering
└── WAF (if attached): Layer 7 protection

Lambda:
├── Concurrency limits act as natural throttle
├── Reserved concurrency prevents resource exhaustion
└── No direct internet exposure (only via API GW or ALB)
```

**Key insight**: CloudFront is the best first line of defense because:
- Absorbs volumetric attacks at the edge (distributed globally)
- SSL/TLS termination at edge reduces origin load
- Caching reduces origin requests
- WAF rules evaluated at edge (closer to attacker)

---


**Q24: What is a Slowloris attack? How does AWS protect against it?**

**A:** Slowloris keeps many connections open by sending partial HTTP requests slowly, exhausting server connection pools.

**AWS Protection:**
- **CloudFront**: Buffers complete requests before forwarding to origin. Slowloris connections tie up edge (massive capacity), not your origin.
- **ALB**: Has idle timeout (default 60s) and connection limits. Drops slow connections.
- **API Gateway**: Enforces 29-second timeout. Managed service absorbs connection exhaustion.
- **Shield Advanced**: Detects anomalous connection patterns.

**Tricky**: If your architecture exposes an EC2/NLB directly (no CloudFront/ALB), you're vulnerable. NLB passes TCP connections through without buffering.

---


**Q25: What are the different types of DDoS attacks and which AWS layer protects against each?**

**A:**

| Attack Type | Layer | Description | AWS Protection |
|-------------|-------|-------------|----------------|
| **UDP Flood** | L3 | Overwhelm with UDP packets | Shield Standard (automatic) |
| **SYN Flood** | L4 | Exhaust TCP connection table | Shield Standard (SYN proxy) |
| **DNS Amplification** | L3/L4 | Spoofed DNS requests to amplify traffic | Shield Standard + Route 53 |
| **HTTP Flood** | L7 | Overwhelming HTTP requests | WAF rate-based rules + Shield Advanced |
| **Slowloris** | L7 | Slow partial requests | CloudFront/ALB buffering |
| **Cache Busting** | L7 | Unique URLs to bypass cache | WAF + CloudFront origin shield |
| **SSL/TLS Abuse** | L4/L7 | Expensive handshake/renegotiation | CloudFront SSL termination |

---


**Q26: How do you architect for DDoS resilience on AWS?**

**A:** Defense-in-depth approach:

1. **Edge Layer**: CloudFront + WAF + Shield Advanced
2. **DNS Layer**: Route 53 (inherently DDoS-resilient, uses Anycast + shuffle sharding)
3. **Network Layer**: VPC with proper security groups + NACLs
4. **Application Layer**: Rate limiting, API Gateway throttling, Lambda concurrency limits
5. **Scaling**: Auto-scaling groups behind ALB (absorb what you can't block)
6. **Isolation**: Separate critical APIs with reserved concurrency
7. **Monitoring**: CloudWatch alarms, Shield Advanced metrics, WAF logging

**Key principle**: "Minimize attack surface, isolate resources, absorb the rest"

---


**Q27: What is Shield Advanced "Cost Protection" and how does it work?**

**A:**

Shield Advanced provides DDoS cost protection — if a DDoS attack causes your AWS bill to spike (due to auto-scaling, increased data transfer, etc.), AWS provides service credits for the charges attributable to the attack.

**Requirements:**
- Shield Advanced must be active on the resource BEFORE the attack
- The resource must have a Route 53 health check associated (for ALB, EIP)
- You must file a support case with the SRT within 15 days
- Only covers resources that are "protected" under Shield Advanced

**Tricky**: Cost protection doesn't cover Lambda invocation costs from API Gateway! It covers scaling costs for EC2, ALB, CloudFront data transfer, and Route 53 queries.

---

## 4. AWS WAF — Deep Dive

### Core Concepts


**Q28: Explain WAF rule evaluation order and how WCUs (Web ACL Capacity Units) work.**

**A:**

**Rule Priority** (lowest number = evaluated first):
```
Rules evaluated in priority order (0, 1, 2, ...)
├── First matching rule with ALLOW/BLOCK terminates evaluation
├── COUNT rules don't terminate — continue evaluation
├── CAPTCHA/Challenge: terminates if token invalid, continues if valid
└── Default action applied if no rules match
```

**WCUs (Web ACL Capacity Units):**
- Each Web ACL has a maximum of 5,000 WCUs (soft limit)
- Each rule consumes WCUs based on complexity:
  - Simple match (IP set, string match): 1 WCU
  - Regex: 25 WCUs per pattern
  - Rate-based rule: 2 WCUs + condition WCUs
  - Managed rule groups: Pre-defined WCU consumption (e.g., Core Rule Set = 700 WCUs)
- Rule groups also have WCU limits

**Tricky**: WCU is a capacity reservation, not runtime cost. A regex rule consuming 25 WCUs uses that regardless of traffic volume.

---


**Q29: Explain WAF rate-based rules in detail. What's the evaluation window?**

**A:**

- **Evaluation window**: 5 minutes (sliding window, updated every 30 seconds)
- **Minimum threshold**: 100 requests per 5 minutes
- **Scope**: Can aggregate by IP, forwarded IP, custom keys (header, query string, etc.)
- **Action when triggered**: Blocks requests from the aggregation key until rate drops below threshold

**Custom Keys (powerful feature):**
```json
{
  "AggregateKeyType": "CUSTOM_KEYS",
  "CustomKeys": [
    {"Header": {"Name": "Authorization"}},
    {"QueryString": {}}
  ]
}
```

**Tricky scenarios:**
- Rate-based rule with no scope-down statement: Counts ALL requests matching the rule
- IP-based rate limit behind CloudFront: Use "Forwarded IP" with `X-Forwarded-For` header
- Rate limit ONLY for specific paths: Use scope-down statement to filter
- 30-second update delay means short bursts can still get through

---


**Q30: What are WAF Managed Rule Groups? Explain the key AWS-managed ones.**

**A:**

| Rule Group | WCUs | Purpose |
|-----------|------|---------|
| AWSManagedRulesCommonRuleSet (Core) | 700 | OWASP Top 10: XSS, SQLi, path traversal |
| AWSManagedRulesKnownBadInputsRuleSet | 200 | Log4j/JNDI, known exploits |
| AWSManagedRulesSQLiRuleSet | 200 | SQL injection (advanced) |
| AWSManagedRulesAmazonIpReputationList | 25 | Known malicious IPs |
| AWSManagedRulesAnonymousIpList | 50 | VPNs, Tor, proxies, hosting providers |
| AWSManagedRulesBotControlRuleSet | 50 (common) / 725 (targeted) | Bot detection |
| AWSManagedRulesATPRuleSet | 50 | Account takeover prevention |

**Important**: Managed rules can be overridden per-rule:
- Set to COUNT instead of BLOCK for testing
- Use label-based logic: Rule adds label → subsequent custom rule evaluates label with additional conditions

---


**Q31: Explain WAF Labels. How do they enable complex logic?**

**A:** Labels are metadata that rules can add to requests. Subsequent rules can match on labels.

**Example — Allow known bots, block unknown bots:**
```
Rule 1 (Priority 0): Bot Control managed rule group
  → Adds labels like: awswaf:managed:aws:bot-control:bot:category:search_engine

Rule 2 (Priority 1): Custom rule
  → IF label "bot:category:search_engine" exists → ALLOW

Rule 3 (Priority 2): Custom rule
  → IF label "bot:detected" exists → BLOCK
```

**Power of labels:**
- Managed rules add labels without taking action (if overridden to COUNT)
- Custom rules can combine multiple labels with AND/OR logic
- Enables "detect in one rule, decide in another" pattern
- Labels exist only for the current request evaluation (not stored)
- Labels follow the format: `awswaf:managed:{vendor}:{rule-group}:{label-name}`

---


### Tricky WAF Questions

**Q32: You enable AWS Managed Rules Core Rule Set but legitimate file uploads are being blocked. How do you fix it without disabling the entire rule group?**

**A:** The issue is likely `SizeRestrictions_BODY` or `CrossSiteScripting_BODY` rules matching on uploaded file content.

**Solutions (from least to most permissive):**

1. **Rule-level override**: Set the specific problematic rule to COUNT using `RuleActionOverrides`
2. **Scope-down statement**: Apply the managed rule group only to non-upload paths
3. **Label + custom rule**: Override managed rule to COUNT, create custom rule that blocks based on the label UNLESS the path is `/upload`
4. **ExcludedRules**: Exclude specific rules from the group

```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesCommonRuleSet",
    "RuleActionOverrides": [
      {"Name": "SizeRestrictions_BODY", "ActionToUse": {"Count": {}}}
    ],
    "ScopeDownStatement": {
      "NotStatement": {
        "Statement": {
          "ByteMatchStatement": {
            "SearchString": "/upload",
            "FieldToMatch": {"UriPath": {}},
            "PositionalConstraint": "STARTS_WITH"
          }
        }
      }
    }
  }
}
```

---


**Q33: What's the difference between `Action` and `OverrideAction` in WAF rules?**

**A:** This is a common source of confusion:

- **Action**: Used by individual rules (not rule groups). Options: ALLOW, BLOCK, COUNT, CAPTCHA, Challenge
- **OverrideAction**: Used ONLY when referencing a Rule Group in a Web ACL. Options:
  - `"None": {}` — Use the actions defined within the rule group (pass-through)
  - `"Count": {}` — Override ALL rules in the group to COUNT (useful for testing)

**Tricky**: If you set `OverrideAction: Count`, EVERY rule in the group acts as COUNT regardless of its internal action. The rules will still add their labels, which you can use in subsequent custom rules.

Individual rules within a rule group have their own actions, and you can override specific rules using `RuleActionOverrides` (newer) or `ExcludedRules` (legacy, sets them to COUNT).

---


**Q34: How does WAF handle request body inspection? What are the limits?**

**A:**

- **Default body inspection limit**: First 8 KB for Regional (ALB, API Gateway), first 16 KB for CloudFront
- **Extended body inspection** (paid): Up to 64 KB
- **If body exceeds inspection limit** — configurable `OversizeHandling`:
  - `MATCH` — Treat as matching the rule (conservative/secure)
  - `NO_MATCH` — Skip inspection (permissive)
  - `CONTINUE` — Inspect what's available (default for most)

**Tricky**: If an attacker puts a SQL injection payload beyond 8KB in the body, default WAF won't detect it! Solutions:
1. Combine WAF with API Gateway request validation (reject large bodies)
2. Enable extended body inspection for sensitive endpoints
3. Set `OversizeHandling: MATCH` for security-critical rules
4. Use API Gateway model validation to enforce maximum body size

---


**Q35: How do you test WAF rules without blocking legitimate traffic?**

**A:** Multi-step safe deployment:

1. **COUNT mode**: Deploy new rules as COUNT initially. Monitor CloudWatch metrics and WAF logs.
2. **WAF Logging**: Enable full logging to S3/CloudWatch Logs/Kinesis Firehose. Analyze matched requests.
3. **Sampled Requests**: View sampled requests in console (last 3 hours, up to 5,000 samples).
4. **Labels**: Use managed rules in COUNT mode with labels, then custom rules that act on labels (easier to adjust).
5. **Gradual rollout**: Switch individual rules from COUNT to BLOCK one at a time.
6. **AWS WAF Bot Control**: Start with "common" level before "targeted" to avoid false positives.

**Best practice flow:**
```
Deploy as COUNT → Analyze logs (1-2 weeks) → Identify false positives → 
Add exceptions → Switch to BLOCK → Monitor → Iterate
```

---


**Q36: Where can you attach WAF? What are the regional vs global considerations?**

**A:**

**WAF can be attached to:**
- CloudFront distributions (Global — must use us-east-1 for WAF)
- Application Load Balancer (Regional)
- API Gateway REST API (Regional)
- AWS AppSync GraphQL API (Regional)
- Amazon Cognito User Pool (Regional)
- AWS App Runner service (Regional)
- AWS Verified Access instance (Regional)

**Key considerations:**
- One Web ACL per resource (but one Web ACL can be attached to multiple resources)
- CloudFront WAF MUST be created in `us-east-1`
- Regional WAF must be in the same region as the resource
- A Web ACL for CloudFront cannot be used for ALB and vice versa (scope difference: CLOUDFRONT vs REGIONAL)

---

## 5. Integration & Architecture Questions


**Q37: Design a secure, scalable API with DDoS protection using Lambda + API Gateway + WAF + Shield.**

**A:** Complete architecture:

```
Client
  ↓
Route 53 (Shield Standard, DNSSEC, health checks)
  ↓
CloudFront (Shield Advanced, WAF Web ACL attached here)
  ├── Edge caching reduces origin load
  ├── WAF: Rate limiting, IP reputation, Bot Control, Core Rules
  ├── Geographic restrictions
  └── Origin = API Gateway regional endpoint
  ↓
API Gateway (REST API)
  ├── Resource policy (restrict to CloudFront only)
  ├── Usage plans + API keys for partner APIs
  ├── Request validation (model validation)
  ├── Lambda authorizer (JWT validation + custom logic)
  └── Stage throttling: 1000 RPS steady, 2000 burst
  ↓
Lambda (Private subnet)
  ├── Reserved concurrency: 500 (protects downstream)
  ├── VPC for RDS access
  ├── RDS Proxy for connection pooling
  └── IAM role with least-privilege

Monitoring:
  ├── CloudWatch dashboards (Lambda errors, API GW 4xx/5xx, WAF blocks)
  ├── Shield Advanced attack notifications → SNS → PagerDuty
  ├── WAF logs → Kinesis Firehose → S3 → Athena (analysis)
  └── X-Ray tracing (end-to-end)
```

**Key design decisions:**
- WAF on CloudFront (not API Gateway) — evaluates at edge, blocks before reaching origin
- API Gateway restricted to CloudFront only — prevents direct origin access bypass
- Lambda reserved concurrency — prevents Lambda from overwhelming RDS
- Rate limiting at WAF layer — stops abusers before consuming Lambda execution time

---


**Q38: How do you prevent direct access to API Gateway, bypassing CloudFront + WAF?**

**A:** Multiple approaches:

**1. Custom origin header (recommended):**
- CloudFront adds a custom origin header (e.g., `x-origin-verify: <secret-value>`)
- API Gateway Lambda authorizer or WAF rule validates the header
- Rotate the secret periodically using AWS Secrets Manager

**2. API Gateway Resource Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "execute-api:Invoke",
    "Resource": "arn:aws:execute-api:*:*:*/*/*/*",
    "Condition": {
      "StringNotEquals": {
        "aws:Referer": "your-secret-string"
      }
    }
  }]
}
```

**3. WAF on API Gateway (defense in depth):**
- Attach a WAF Web ACL to API Gateway that only allows requests with the custom header

**4. AWS WAF + IP restriction (less reliable):**
- Restrict to CloudFront IP ranges (but these change and are shared)

**Tricky**: None of these are 100% foolproof individually. Layer multiple approaches for defense in depth.

---


**Q39: How do you implement rate limiting at multiple layers? Why is it necessary?**

**A:**

```
Layer 1: WAF Rate-Based Rules (per IP, per API key, per path)
  └── Blocks at edge before reaching your infrastructure
  └── 5-minute sliding window, minimum 100 requests

Layer 2: API Gateway Throttling (per stage, per method, per client)
  └── Token bucket algorithm: steady-state + burst
  └── Usage plans for per-client quotas (daily/weekly/monthly)

Layer 3: Lambda Concurrency (reserved concurrency)
  └── Protects downstream services (DB, third-party APIs)
  └── Natural rate limiting via execution environment cap

Layer 4: Application-Level (custom logic in Lambda)
  └── Business-logic rate limiting (e.g., 5 password resets per hour)
  └── Store counters in DynamoDB/ElastiCache
```

**Why multiple layers?**
- WAF: Stops volumetric abuse at the edge (cost-effective)
- API Gateway: Provides per-client fairness and quotas
- Lambda: Protects fragile downstream dependencies
- Application: Enforces business rules that infrastructure can't understand

---


**Q40: You notice your Lambda function is being invoked millions of times, causing a huge bill. The requests are coming through API Gateway. How do you investigate and mitigate?**

**A:**

**Investigation:**
1. Check WAF logs (if attached) — any patterns in blocked/allowed requests?
2. API Gateway access logs — identify source IPs, user agents, paths
3. CloudWatch metrics — when did the spike start? Which methods?
4. Check if it's a DDoS or a misconfigured client/retry loop

**Immediate Mitigation (in order of speed):**
1. **WAF rate-based rule**: Add IP-based rate limit immediately (takes effect in ~30s)
2. **Lambda reserved concurrency**: Set to lower value to cap invocations
3. **API Gateway stage throttling**: Reduce to emergency levels
4. **WAF IP block rule**: If attacker IPs identified, add to IP set (block immediately)
5. **API Gateway resource policy**: Block specific IPs/ranges

**Long-term fixes:**
- Enable WAF with proper rate-based rules
- Set up CloudWatch billing alarms
- Implement usage plans + API keys for all clients
- Add Shield Advanced for automatic detection
- Consider moving to CloudFront + WAF for edge-level protection

---

## 6. Tricky Scenario-Based Questions


**Q41: Your WAF is blocking requests with 403, but you can't figure out which rule is blocking them. How do you debug?**

**A:**

**Step-by-step debugging:**

1. **Enable WAF logging** (if not already):
   - Send to CloudWatch Logs, S3, or Kinesis Firehose
   - Logs include: timestamp, action, matching rule, request details, labels

2. **Check sampled requests** in WAF console:
   - Shows which rule matched and the action taken
   - Limited to last 3 hours and 5,000 samples

3. **Look at the `terminatingRule` field** in WAF logs:
   ```json
   {
     "terminatingRuleId": "AWS-AWSManagedRulesCommonRuleSet",
     "terminatingRuleMatchDetails": [{
       "conditionType": "XSS",
       "location": "BODY"
     }]
   }
   ```

4. **Use CloudWatch Metrics**:
   - `BlockedRequests` metric with `Rule` dimension shows which rule is blocking

5. **Switch suspicious rules to COUNT temporarily**:
   - Use `RuleActionOverrides` to set individual rules to COUNT
   - Monitor logs to confirm which rule was the culprit
   - Add proper exceptions, then switch back to BLOCK

---


**Q42: A Lambda function processes API Gateway requests. During a traffic spike, some users get 502 errors while others get 429. Explain the difference.**

**A:**

- **429 (Too Many Requests)**: Lambda concurrency limit reached. API Gateway receives `TooManyRequestsException` from Lambda and returns 429 to client.

- **502 (Bad Gateway)**: Lambda function error or timeout. Causes:
  - Function threw an unhandled exception
  - Function timed out (hit the configured timeout limit)
  - Function returned an invalid response format
  - Out of memory (Lambda killed the function)

**During a traffic spike, both can happen because:**
1. Some requests get through but Lambda is overwhelmed → timeout/error → 502
2. Other requests can't even start because concurrency is maxed → 429

**Solution:**
- Increase Lambda concurrency limit (or remove reserved concurrency cap)
- Add provisioned concurrency for consistent performance
- Implement API Gateway request throttling to smooth traffic
- Add caching for cacheable endpoints
- Use async processing pattern for non-critical operations

---


**Q43: Your company has a multi-tenant API. How do you prevent one tenant from consuming all resources (noisy neighbor)?**

**A:**

**API Gateway level:**
- **Usage Plans + API Keys**: Assign each tenant an API key with throttle limits (requests/second) and quotas (requests/day)
- **Per-method throttling**: Different limits for expensive vs cheap operations

**WAF level:**
- **Rate-based rules with custom keys**: Rate limit per API key header or tenant ID
- **Scope-down statements**: Different rate limits for different endpoints

**Lambda level:**
- **Separate Lambda functions per tier**: Premium tenants → dedicated Lambda with provisioned concurrency
- **Custom concurrency management**: Use DynamoDB atomic counters to track per-tenant concurrency

**Architecture patterns:**
```
Standard Tenant:
  API Key → Usage Plan (100 RPS, 10K/day) → Shared Lambda (reserved: 200)

Premium Tenant:
  API Key → Usage Plan (1000 RPS, unlimited) → Dedicated Lambda (provisioned: 50)
```

**Tricky**: API Gateway usage plans only work with REST API (not HTTP API). If using HTTP API, you must implement tenant isolation in Lambda or use WAF custom rate-based rules.

---


**Q44: You're migrating from a monolithic application to serverless (Lambda + API Gateway). What DDoS/security considerations change?**

**A:**

**What improves:**
- No servers to patch or manage (reduced attack surface)
- Auto-scaling handles legitimate traffic spikes
- API Gateway provides built-in throttling
- Lambda concurrency acts as natural circuit breaker
- Per-function IAM roles enable least privilege

**What gets riskier:**
- **Financial DDoS (denial of wallet)**: Attacker can run up your bill even if your app stays up
- **Cold starts during attacks**: Sudden spike = massive cold starts = degraded experience
- **Downstream overwhelm**: Lambda scales faster than your database can handle
- **Shared account limits**: Lambda concurrency is account-wide; one function can starve others
- **More endpoints = more attack surface**: Microservices expose more entry points

**New security considerations:**
- Every Lambda function needs proper IAM roles (principle of least privilege)
- Event injection: Attacker can craft malicious event payloads (e.g., SQL injection in DynamoDB query)
- Dependency vulnerabilities: Each function may have different dependencies to secure
- Secrets management: No long-lived servers means secrets need dynamic retrieval (Secrets Manager, Parameter Store)

---


**Q45: Explain "Denial of Wallet" (Economic DDoS). How do you protect against it in serverless?**

**A:**

**Denial of Wallet**: An attack that doesn't aim to take your service down, but to run up your cloud bill to unsustainable levels. Serverless is particularly vulnerable because every request costs money.

**Protection strategies:**

1. **WAF rate-based rules**: Block IPs exceeding thresholds (cheapest protection layer)
2. **API Gateway throttling**: Cap maximum requests regardless of source
3. **Lambda reserved concurrency**: Hard cap on concurrent executions
4. **AWS Budgets + CloudWatch Billing Alarms**: Alert when costs exceed thresholds
5. **Shield Advanced Cost Protection**: Reimburses DDoS-related scaling costs (limited scope)
6. **CloudFront caching**: Cached responses don't invoke Lambda (free at edge)
7. **API keys + usage plans**: Quota limits per client prevent runaway usage

**Monitoring formula:**
```
Estimated hourly cost = (Lambda invocations × avg_duration × memory_price) 
                      + (API Gateway requests × price_per_request)
                      + (Data transfer × price_per_GB)

Alert when: Current hourly rate > 3× baseline hourly rate
```

**Tricky**: Shield Advanced cost protection does NOT cover Lambda invocation costs or API Gateway request costs. It only covers CloudFront data transfer, ALB capacity units, and EC2 scaling.

---


**Q46: Your WAF is configured with both an IP whitelist (ALLOW) and a managed rule group (BLOCK). A request comes from a whitelisted IP but contains SQL injection. What happens?**

**A:** It depends on **rule priority** (evaluation order):

**Scenario 1: IP whitelist has LOWER priority number (evaluated first):**
- Request matches IP whitelist → Action = ALLOW → Evaluation STOPS
- SQL injection rule is never evaluated
- Request is ALLOWED (even with SQL injection!)

**Scenario 2: Managed rule group has LOWER priority number (evaluated first):**
- Request matches SQL injection rule → Action = BLOCK → Evaluation STOPS
- IP whitelist is never evaluated
- Request is BLOCKED (even from whitelisted IP!)

**Best Practice:**
```
Priority 0: IP Whitelist (ALLOW) — Trusted IPs skip all other checks
Priority 1: IP Blocklist (BLOCK) — Known bad actors
Priority 2: Rate-based rules (BLOCK) — Stop floods
Priority 3: Managed rule groups (BLOCK) — OWASP protections
Priority 99: Default: ALLOW or BLOCK (depending on security posture)
```

**Tricky decision**: Do you trust whitelisted IPs enough to skip security checks? If a trusted partner's system is compromised, you'd be vulnerable. Consider using COUNT for managed rules against whitelisted IPs and alerting instead.

---


**Q47: How does API Gateway handle CORS? What's a common mistake that causes CORS failures?**

**A:**

**How CORS works with API Gateway:**

- **Preflight (OPTIONS)**: Browser sends OPTIONS request before cross-origin requests
- **REST API**: Must explicitly configure OPTIONS method (mock integration) or enable CORS in console
- **HTTP API**: Built-in CORS configuration (simpler)

**Common mistakes:**

1. **Missing OPTIONS method**: REST API doesn't auto-create OPTIONS. Without it, preflight fails.
2. **Lambda not returning CORS headers**: Even if OPTIONS is configured, the actual Lambda response must include `Access-Control-Allow-Origin` header.
3. **Wildcard with credentials**: `Access-Control-Allow-Origin: *` doesn't work with `credentials: include`. Must specify exact origin.
4. **Gateway response not configured**: 4xx/5xx errors from API Gateway (not Lambda) don't include CORS headers by default → browser shows CORS error instead of actual error.

**Fix for Gateway Responses:**
```
Configure API Gateway "Gateway Responses" to add CORS headers
for 4XX and 5XX responses (DEFAULT_4XX, DEFAULT_5XX)
```

**Tricky**: When WAF blocks a request, the 403 response comes from WAF/CloudFront, NOT API Gateway. It won't have CORS headers → browser shows "CORS error" instead of "403 Forbidden". Solution: Configure CloudFront custom error responses or WAF custom response bodies with CORS headers.

---


**Q48: You need to deploy an API that handles 50,000 requests per second. Describe the architecture and potential bottlenecks.**

**A:**

**Architecture for 50K RPS:**

```
Route 53 → CloudFront (caching) → API Gateway → Lambda
                                                    ↓
                                              DynamoDB (on-demand)
```

**Bottlenecks and solutions:**

| Component | Default Limit | Solution |
|-----------|---------------|----------|
| API Gateway | 10,000 RPS (account) | Request limit increase, multiple accounts, or use CloudFront caching |
| Lambda | 1,000 concurrent (account) | Request increase to 10K+ concurrent, provisioned concurrency |
| Lambda burst | 3,000 immediate, then +500/min | Provisioned concurrency for predictable load |
| CloudFront | 250,000 RPS per distribution | No issue at 50K |
| DynamoDB | On-demand: virtually unlimited | Use on-demand mode or properly provisioned capacity |

**Key optimizations:**
1. **CloudFront caching**: If even 50% of requests are cacheable, origin only handles 25K RPS
2. **Lambda provisioned concurrency**: 50K RPS with 100ms avg duration = 5,000 concurrent needed
3. **API Gateway account limit**: Request increase well in advance (takes days)
4. **DynamoDB DAX**: Cache for read-heavy workloads (microsecond latency)
5. **Regional deployment**: Spread across multiple regions with Route 53 latency routing

**Tricky math:**
```
Required Lambda concurrency = RPS × Average Duration (in seconds)
50,000 RPS × 0.1s = 5,000 concurrent Lambdas needed
```

---


**Q49: How do you implement request signing/authentication that works with WAF?**

**A:**

**Challenge**: WAF evaluates requests BEFORE they reach your Lambda authorizer. You need auth logic that WAF can handle.

**Approaches:**

1. **JWT validation in Lambda Authorizer (most common):**
   - WAF handles rate limiting and bot detection
   - Lambda Authorizer validates JWT signature, expiry, claims
   - WAF can't natively validate JWTs (except HTTP API which has native JWT auth)

2. **API keys validated at both layers:**
   - WAF: Rate limit per API key (custom header inspection)
   - API Gateway: API key validation + usage plan enforcement

3. **AWS Cognito + WAF ATP (Account Takeover Prevention):**
   - WAF ATP rule group detects credential stuffing
   - Cognito handles user authentication
   - Combines security at edge with identity management

4. **mTLS (Mutual TLS):**
   - API Gateway validates client certificates
   - WAF cannot inspect mTLS (happens at TLS layer)
   - Must use custom domain with truststore

**Tricky**: WAF custom headers added by rules (via `CustomRequestHandling`) are visible to your backend but NOT to the client. You can use this to pass WAF decision metadata to your Lambda.

---


**Q50: Compare putting WAF on CloudFront vs on API Gateway directly. When would you do each?**

**A:**

| Consideration | WAF on CloudFront | WAF on API Gateway |
|---------------|-------------------|-------------------|
| **Evaluation location** | At edge (closer to user/attacker) | At region (after traffic reaches AWS region) |
| **Latency** | Lower (edge evaluation) | Higher (regional evaluation) |
| **Body inspection limit** | 16 KB (64 KB extended) | 8 KB (64 KB extended) |
| **Cost** | WAF charges at CloudFront pricing | WAF charges at regional pricing |
| **Protection scope** | Protects everything behind CloudFront | Only protects API Gateway |
| **DDoS defense** | Stops attacks before they reach origin | Attacks reach the region first |

**Use WAF on CloudFront when:**
- You have CloudFront in front of API Gateway
- You want maximum protection with minimum latency
- You need Shield Advanced integration
- You want to block attacks at the edge

**Use WAF on API Gateway when:**
- No CloudFront in architecture (e.g., internal APIs)
- Need API-specific rules separate from CDN rules
- Regional API not exposed through CloudFront
- Defense in depth (WAF at both layers)

**Both together (defense in depth):**
- CloudFront WAF: Rate limiting, IP reputation, geo-blocking (broad protection)
- API Gateway WAF: API-specific rules, request validation, business logic rules (targeted protection)

---


**Q51: What is AWS WAF Fraud Control? Explain ATP and ACFP.**

**A:**

**ATP (Account Takeover Prevention):**
- Detects credential stuffing and brute force login attempts
- Monitors login endpoints for stolen credentials (checks against a database of breached credentials)
- Tracks failed login attempts and blocks suspicious patterns
- Requires specifying your login URL and request body field mappings

**ACFP (Account Creation Fraud Prevention):**
- Detects fake account creation (bot sign-ups)
- Analyzes account creation patterns (speed, email domains, similarity)
- Blocks automated registration tools
- Requires specifying your registration URL and form field mappings

**Configuration required:**
```json
{
  "ManagedRuleGroupStatement": {
    "VendorName": "AWS",
    "Name": "AWSManagedRulesATPRuleSet",
    "ManagedRuleGroupConfigs": [{
      "LoginPath": "/api/login",
      "RequestInspection": {
        "PayloadType": "JSON",
        "UsernameField": {"Identifier": "/username"},
        "PasswordField": {"Identifier": "/password"}
      }
    }]
  }
}
```

**Tricky**: Both ATP and ACFP require JavaScript/iOS/Android SDK integration for full effectiveness (device fingerprinting, behavioral analysis). Without the SDK, only basic request analysis is performed.

---


**Q52: How does Lambda SnapStart work and what are its limitations?**

**A:**

**SnapStart** (Java only, as of now): Reduces cold start from seconds to ~200ms by:
1. Initializing the function once at publish time
2. Taking a Firecracker VM snapshot (memory + disk state)
3. Restoring from snapshot on cold start instead of full initialization

**Limitations:**
- **Java only** (Corretto 11+)
- **Uniqueness concern**: Random values, UUIDs, connections initialized before snapshot are IDENTICAL across all restored instances. Must re-initialize in handler.
- **Network connections**: Stale after snapshot restore. Must validate/reconnect.
- **14-day snapshot retention**: Snapshots expire if function not invoked for 14 days
- **Not compatible with**: Provisioned Concurrency, EFS, Ephemeral storage > 512MB, arm64, x86_64 only

**Tricky**: If you generate a unique ID during INIT phase (snapshot), EVERY cold-started instance will have the SAME ID. Use `CRaC` (Coordinated Restore at Checkpoint) hooks:
```java
public class Handler implements RequestHandler<Event, Response>, CracResource {
    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) {
        // Called before snapshot - clean up
    }
    @Override
    public void afterRestore(Context<? extends Resource> context) {
        // Called after restore - re-initialize unique values
    }
}
```

---


**Q53: Your API is under a sophisticated Layer 7 DDoS attack that mimics legitimate traffic. WAF rate-based rules aren't effective because the attack is distributed across thousands of IPs. What do you do?**

**A:**

**Immediate actions:**
1. **Engage Shield Response Team (SRT)** if you have Shield Advanced — they have visibility into attack patterns you don't
2. **Enable WAF Bot Control (targeted level)** — uses behavioral analysis, not just IP-based detection
3. **CAPTCHA/Challenge rules** — add WAF CAPTCHA action for suspicious paths to distinguish humans from bots

**Advanced WAF strategies:**
4. **Geographic restrictions** — if attack comes from specific regions where you have no users
5. **Custom rate-based rules with compound keys** — rate limit by combinations of headers + IP + path
6. **Token-based rate limiting** — rate limit by session token, API key, or user ID (not IP)
7. **Request pattern analysis** — look for abnormal User-Agent, Accept-Language, TLS fingerprint patterns

**Architecture changes:**
8. **Add CloudFront Challenge** — JavaScript challenge at CDN level (blocks simple bots)
9. **Implement request signing** — legitimate clients sign requests with timestamp + HMAC
10. **Reduce attack surface** — disable unused API methods, add strict request validation

**Shield Advanced specific:**
11. **Proactive engagement** — Shield automatically contacts SRT when health checks fail
12. **Automatic Layer 7 mitigation** — Shield creates/updates WAF rules based on attack patterns

**Tricky**: Distributed attacks with legitimate-looking traffic are the hardest to mitigate. There's no silver bullet — it requires multiple signals (behavioral, volumetric, geographic, fingerprinting) combined together.

---


**Q54: Explain API Gateway Private APIs. How do they work with VPC Endpoints?**

**A:**

**Private APIs** are REST APIs accessible only from within a VPC (or connected VPCs/on-premises via VPN/Direct Connect).

**Architecture:**
```
VPC
├── Private Subnet
│   ├── Lambda / EC2 / ECS (consumer)
│   └── VPC Endpoint (Interface type, for execute-api)
│       ├── ENI in each selected subnet
│       └── Private DNS enabled → resolves API GW domain to VPC endpoint IPs
└── API Gateway (Private)
    └── Resource Policy restricts access to specific VPC endpoint(s)
```

**Resource Policy (required):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "execute-api:Invoke",
    "Resource": "arn:aws:execute-api:region:account:api-id/*",
    "Condition": {
      "StringEquals": {
        "aws:sourceVpce": "vpce-0123456789abcdef0"
      }
    }
  }]
}
```

**Tricky considerations:**
- Private DNS must be enabled on the VPC endpoint for standard API Gateway URLs to resolve
- Without private DNS, use endpoint-specific URLs: `{api-id}-{vpce-id}.execute-api.{region}.amazonaws.com`
- WAF can still be attached to Private REST APIs
- Private APIs don't need custom domains (but can use them via Route 53 private hosted zones)
- Cannot be accessed from the internet even if the resource policy allows `*`

---


**Q55: What are the Lambda function URL features vs API Gateway? When would you use function URLs?**

**A:**

**Lambda Function URLs** provide a dedicated HTTPS endpoint for a Lambda function without API Gateway.

| Feature | Lambda Function URL | API Gateway |
|---------|-------------------|-------------|
| **Cost** | Free (only pay for Lambda) | $1-3.50/million requests |
| **Throttling** | No built-in (only Lambda concurrency) | Built-in token bucket |
| **Auth** | IAM or None | IAM, Cognito, Lambda Authorizer, API Keys |
| **WAF** | Not directly (use CloudFront in front) | Yes (REST API) |
| **Custom domain** | Not natively (use CloudFront) | Yes |
| **Request validation** | No | Yes |
| **Caching** | No | Yes (REST API) |
| **Rate limiting** | No | Yes |
| **Response streaming** | Yes | No (API GW buffers response) |

**Use Function URLs when:**
- Simple webhook receivers (GitHub, Stripe, Slack)
- Internal microservice-to-microservice calls
- Need response streaming (large data transfer)
- Cost-sensitive with high request volume
- Single-function APIs with no routing logic

**Don't use when:**
- Need WAF protection, rate limiting, or caching
- Multi-route APIs
- Need request/response transformation
- Require usage plans or API keys

**Tricky**: Function URLs are directly internet-accessible if auth is NONE. There's no built-in rate limiting — a DDoS can invoke your function without limit. Always put CloudFront + WAF in front for public-facing function URLs.

---


**Q56: How do you implement blue/green deployments with API Gateway and Lambda?**

**A:**

**Method 1: Lambda Aliases + API Gateway Stage Variables**
```
Lambda function: myFunction
├── Version 1 (blue - current production)
├── Version 2 (green - new code)
├── Alias "prod" → weighted routing:
│   ├── 90% → Version 1
│   └── 10% → Version 2 (canary)
└── Alias "prod" → 100% Version 2 (after validation)

API Gateway Integration: arn:aws:lambda:...:myFunction:${stageVariables.alias}
Stage Variable: alias = prod
```

**Method 2: API Gateway Canary Deployments**
```
Stage: prod
├── Base deployment (90% traffic) → Version 1
└── Canary deployment (10% traffic) → Version 2

Promote canary: canary → base (shifts 100% to new version)
```

**Method 3: Route 53 Weighted Routing**
```
custom-api.example.com
├── 90% → API-Gateway-Blue (us-east-1)
└── 10% → API-Gateway-Green (us-east-1, different stage)
```

**Tricky**: Lambda alias weighted routing + CodeDeploy enables automated rollback:
- Pre-traffic hook: Run integration tests against new version
- Traffic shifting: Linear10PercentEvery1Minute
- Rollback: If CloudWatch alarm triggers, auto-shift back to previous version

---


**Q57: What are WAF CAPTCHA and Challenge actions? How do they differ?**

**A:**

| Feature | CAPTCHA | Challenge |
|---------|---------|-----------|
| **User interaction** | Yes (puzzle to solve) | No (silent JavaScript challenge) |
| **Bot detection** | Human verification | Browser verification (not headless) |
| **User experience** | Disruptive (visible) | Transparent (invisible) |
| **Token validity** | Configurable (default 300s) | Configurable (default 300s) |
| **Effectiveness** | Stops most bots | Stops simple bots, headless browsers |

**How they work:**
1. Request arrives at WAF → matches rule with CAPTCHA/Challenge action
2. WAF checks for valid token (from previous successful challenge)
3. If token valid → request passes through (evaluation continues)
4. If token absent/expired → WAF returns interstitial page (202 status)
5. Client solves CAPTCHA/runs JavaScript → receives token → retries

**Tricky considerations:**
- API clients (mobile apps, CLI tools) can't solve CAPTCHA — only works for browser-based clients
- Challenge requires JavaScript execution — won't work for API-to-API calls
- Token immunity time (300s default) means subsequent requests pass without challenge
- If your frontend is a SPA calling APIs via fetch/XHR, you need the AWS WAF JavaScript SDK to handle the interstitial
- Cost: Additional per-CAPTCHA-attempt charges apply

---


**Q58: How do you handle Lambda function versioning and concurrent version execution during deployment?**

**A:**

**Lambda Versioning:**
- `$LATEST`: Mutable, always points to newest code
- Published versions (1, 2, 3...): Immutable snapshots of code + configuration
- Aliases: Named pointers to versions (e.g., "prod" → v3, "staging" → v4)

**Concurrent version execution during deployment:**
```
Time 0: Alias "prod" → Version 1 (100%)
Time 1: Deploy Version 2, shift alias:
         "prod" → Version 1 (90%) + Version 2 (10%)
Time 2: Both versions execute concurrently
Time 3: "prod" → Version 2 (100%)
```

**Key considerations:**
- Provisioned concurrency is PER ALIAS/VERSION — not shared
- During weighted routing, BOTH versions consume from the same function's concurrency pool
- If Version 2 has a bug, some users see errors while others don't (hard to debug)
- CloudWatch logs are separated by version ($LATEST, 1, 2) — check the right log stream
- Environment variables are version-specific (immutable once published)

**Tricky**: You CANNOT change environment variables on a published version. If your deployment requires env var changes, you must publish a new version with the updated env vars.

---


**Q59: What is the difference between API Gateway REST API "Edge-Optimized" vs "Regional" endpoints?**

**A:**

| Feature | Edge-Optimized | Regional |
|---------|---------------|----------|
| **Endpoint** | CloudFront distribution (AWS-managed) | Direct regional endpoint |
| **Latency** | Better for geographically distributed clients | Better for same-region clients |
| **Custom domain TLS cert** | Must be in us-east-1 (ACM) | Must be in same region as API |
| **CloudFront control** | No (AWS-managed, can't configure) | You manage your own CloudFront (optional) |
| **WAF** | Can't attach WAF to the managed CloudFront | Can attach WAF directly + your own CloudFront |
| **Caching** | Edge caching (AWS-managed CF) | API Gateway cache only |

**Tricky insights:**
- Edge-Optimized uses a HIDDEN CloudFront distribution you can't configure
- You CANNOT attach WAF to the hidden CloudFront of Edge-Optimized API — the WAF only attaches to the API Gateway regional component
- Best practice: Use **Regional** endpoint + your OWN CloudFront distribution. This gives you full control over CloudFront settings, WAF at edge, and custom caching rules.
- The hidden CloudFront doesn't cache by default — it only provides edge routing. It's misleading!

---


**Q60: Final Scenario — Design a production-ready serverless API with comprehensive security. Walk through every component.**

**A:**

```
┌─────────────────────────────────────────────────────────────────┐
│                     PRODUCTION ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Client → Route 53 (Alias to CloudFront)                         │
│              │                                                    │
│              ▼                                                    │
│  CloudFront (Regional endpoint as origin)                        │
│  ├── TLS 1.2+ only                                              │
│  ├── Shield Advanced enabled                                     │
│  ├── WAF Web ACL:                                                │
│  │   ├── Priority 0: IP Whitelist (internal) → ALLOW             │
│  │   ├── Priority 1: IP Blocklist → BLOCK                        │
│  │   ├── Priority 2: Rate-based (2000 req/5min per IP) → BLOCK   │
│  │   ├── Priority 3: AWS Bot Control → BLOCK/CAPTCHA             │
│  │   ├── Priority 4: IP Reputation List → BLOCK                  │
│  │   ├── Priority 5: Core Rule Set → BLOCK                       │
│  │   ├── Priority 6: SQL Injection Rules → BLOCK                 │
│  │   ├── Priority 7: Known Bad Inputs → BLOCK                    │
│  │   └── Default: ALLOW                                          │
│  ├── Custom header added: x-origin-verify: {secret}              │
│  └── Origin: API Gateway Regional endpoint                       │
│              │                                                    │
│              ▼                                                    │
│  API Gateway (REST API, Regional)                                │
│  ├── Resource Policy: Only allow with x-origin-verify header     │
│  ├── Lambda Authorizer: JWT validation + RBAC                    │
│  ├── Request Validation: JSON schema models                      │
│  ├── Usage Plans: Per-tenant throttling + quotas                 │
│  ├── Stage throttling: 5000 RPS / 10000 burst                   │
│  ├── Access logging: Full request/response to CloudWatch         │
│  └── X-Ray tracing enabled                                       │
│              │                                                    │
│              ▼                                                    │
│  Lambda Functions (VPC-attached, private subnets)                 │
│  ├── Reserved concurrency: 1000 per function                    │
│  ├── Provisioned concurrency: 100 (critical paths)              │
│  ├── 256-1024 MB memory (tuned per function)                    │
│  ├── 30s timeout (matches API Gateway limit)                     │
│  ├── IAM Role: Least-privilege per function                      │
│  ├── Environment: Encrypted with CMK                             │
│  └── Secrets via Parameter Store/Secrets Manager                 │
│              │                                                    │
│              ▼                                                    │
│  Data Layer                                                       │
│  ├── RDS Proxy → Aurora (Multi-AZ)                               │
│  ├── DynamoDB (on-demand, encryption at rest)                    │
│  ├── ElastiCache Redis (session/cache)                           │
│  └── S3 (objects, encrypted, versioned)                          │
│                                                                   │
│  Monitoring & Alerting                                            │
│  ├── CloudWatch: Dashboards, Alarms, Insights                    │
│  ├── WAF Logs → Firehose → S3 → Athena                          │
│  ├── X-Ray: Distributed tracing                                  │
│  ├── Shield Advanced: Attack visibility                          │
│  └── Billing Alarms: Cost anomaly detection                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Security layers summary:**
1. **Network**: CloudFront edge → absorbs volumetric attacks
2. **WAF**: Filters malicious requests at edge
3. **Shield**: DDoS detection and automatic mitigation
4. **API Gateway**: Throttling, auth, validation
5. **Lambda**: Isolated execution, least-privilege IAM
6. **Data**: Encryption at rest and in transit, VPC isolation

---

## Quick Reference — Key Numbers to Remember

| Service | Limit | Notes |
|---------|-------|-------|
| Lambda max timeout | 15 minutes | 900 seconds |
| API Gateway timeout | 29 seconds (REST) / 30s (HTTP) | Hard limit |
| Lambda deployment package | 50 MB (zipped) / 250 MB (unzipped) | Including layers |
| Lambda memory | 128 MB - 10,240 MB | CPU scales proportionally |
| Lambda /tmp storage | 512 MB - 10,240 MB | Ephemeral |
| Lambda concurrent executions | 1,000 (default) | Account-level, soft limit |
| API Gateway throttle | 10,000 RPS / 5,000 burst | Account-level, all APIs |
| WAF Web ACL capacity | 5,000 WCUs | Soft limit |
| WAF IP set | 10,000 addresses per set | IPv4 or IPv6 |
| WAF rate-based rule minimum | 100 requests / 5 minutes | Per aggregation key |
| Shield Advanced cost | $3,000/month | Per organization |
| Lambda layers | Max 5 per function | Total 250MB unzipped |
| API Gateway payload limit | 10 MB | Request/response |
| Lambda payload (sync) | 6 MB request / 6 MB response | Hard limit |
| Lambda payload (async) | 256 KB | Event payload limit |
