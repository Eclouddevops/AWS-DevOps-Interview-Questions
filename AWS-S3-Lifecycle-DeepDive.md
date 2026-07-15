# AWS S3 Lifecycle & Buckets — Deep-Dive Interview Q&A

## Table of Contents
1. [S3 Storage Classes](#s3-storage-classes)
2. [Lifecycle Policies](#lifecycle-policies)
3. [Security & Access Control](#security--access-control)
4. [Performance & Optimization](#performance--optimization)
5. [Replication & Versioning](#replication--versioning)
6. [Tricky Scenarios](#tricky-scenarios)

---

## S3 Storage Classes


**Q1: Compare all S3 storage classes. When would you use each?**

**A:**

| Storage Class | Durability | Availability | Min Duration | Retrieval | Use Case |
|---------------|-----------|-------------|-------------|-----------|----------|
| Standard | 99.999999999% (11 9s) | 99.99% | None | Instant | Frequently accessed data |
| Intelligent-Tiering | 11 9s | 99.9% | None | Instant | Unknown/changing access patterns |
| Standard-IA | 11 9s | 99.9% | 30 days | Instant | Infrequent but needs fast access |
| One Zone-IA | 11 9s | 99.5% | 30 days | Instant | Re-creatable data, single AZ |
| Glacier Instant | 11 9s | 99.9% | 90 days | Instant (ms) | Archives needing instant access |
| Glacier Flexible | 11 9s | 99.99% | 90 days | Minutes-hours | Archives, 1-5 min retrieval |
| Glacier Deep Archive | 11 9s | 99.99% | 180 days | 12-48 hours | Compliance, rarely accessed |

**Intelligent-Tiering automatic tiers:**
```
Frequent Access (default) → 30 days no access → Infrequent Access
→ 90 days → Archive Instant Access
→ 90-180 days → Archive Access (optional)
→ 180+ days → Deep Archive Access (optional)
```

**Tricky**: Minimum storage duration charges apply! If you upload to Glacier and delete after 1 day, you're still charged for 90 days. Same for Standard-IA (30 days).

---

**Q2: What are S3 retrieval fees and how can they surprise you?**

**A:**

| Class | Retrieval Cost | Per-Request Cost | Minimum Object Size |
|-------|---------------|-----------------|-------------------|
| Standard | Free | $0.0004/1000 GET | None |
| Standard-IA | $0.01/GB | $0.001/1000 GET | 128 KB charged |
| One Zone-IA | $0.01/GB | $0.001/1000 GET | 128 KB charged |
| Glacier Instant | $0.03/GB | $0.01/1000 GET | 128 KB charged |
| Glacier Flexible | $0.01-0.03/GB | $0.05/1000 GET | 40 KB charged |
| Deep Archive | $0.02/GB | $0.10/1000 GET | 40 KB charged |

**Surprise cost scenarios:**
1. **Many small files in IA**: 10,000 files × 1KB each → charged as 10,000 × 128KB = 1.25GB storage!
2. **Frequent access to IA objects**: CloudFront origin hitting IA objects → massive retrieval fees
3. **Lifecycle transition costs**: Moving 1 million objects to Glacier = $0.05/1000 × 1000 = $50 in transition requests alone
4. **Glacier restore + download**: Pay retrieval fee + data transfer fee

**Tricky**: S3 Intelligent-Tiering has NO retrieval fees but charges a monthly monitoring fee ($0.0025 per 1,000 objects). For millions of objects, this adds up. Best for objects > 128KB with unpredictable access.

---

## Lifecycle Policies

**Q3: Explain S3 lifecycle rules. What transitions are allowed and not allowed?**

**A:**

**Allowed transitions (waterfall — only downward):**
```
Standard → Standard-IA → Intelligent-Tiering → One Zone-IA → Glacier Instant → Glacier Flexible → Deep Archive
```

**NOT allowed:**
- Cannot transition FROM a lower tier to a higher tier via lifecycle
- Cannot transition from Glacier back to Standard (must restore + copy)
- Cannot transition from One Zone-IA to Standard-IA (must copy)

**Lifecycle rule example:**
```json
{
  "Rules": [{
    "ID": "archive-old-logs",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "Expiration": {"Days": 2555},
    "NoncurrentVersionTransitions": [
      {"NoncurrentDays": 30, "StorageClass": "GLACIER"}
    ],
    "NoncurrentVersionExpiration": {"NoncurrentDays": 90}
  }]
}
```

**Tricky constraints:**
- Minimum 30 days in Standard before transitioning to IA
- Minimum 30 days in IA before transitioning to Glacier
- BUT you CAN go directly from Standard to Glacier (skipping IA) after 1 day
- Objects < 128KB are never transitioned to IA (AWS skips them silently)

---

**Q4: How do lifecycle rules work with versioning? Explain current vs noncurrent.**

**A:**

```
Object: report.pdf
├── Version 3 (CURRENT) ← Transitions/Expiration rules apply
├── Version 2 (NONCURRENT) ← NoncurrentVersionTransitions/NoncurrentVersionExpiration apply
└── Version 1 (NONCURRENT)

When current version is deleted (with versioning ON):
├── A "delete marker" becomes CURRENT version
├── Previous current version becomes NONCURRENT
└── Object appears deleted but all versions remain
```

**Complete lifecycle with versioning:**
```json
{
  "Transitions": [
    {"Days": 90, "StorageClass": "GLACIER"}
  ],
  "NoncurrentVersionTransitions": [
    {"NoncurrentDays": 30, "StorageClass": "GLACIER"}
  ],
  "NoncurrentVersionExpiration": {
    "NoncurrentDays": 90,
    "NewerNoncurrentVersions": 3
  },
  "ExpiredObjectDeleteMarker": true,
  "AbortIncompleteMultipartUpload": {
    "DaysAfterInitiation": 7
  }
}
```

**Tricky**: `ExpiredObjectDeleteMarker: true` cleans up delete markers that have no noncurrent versions behind them. Without this, you accumulate orphaned delete markers that increase LIST operation costs.

---

**Q5: Design a cost-optimized lifecycle strategy for a data lake with different data tiers.**

**A:**

```
Bucket: company-data-lake

Rule 1: Hot data (prefix: raw/incoming/)
├── Transition to IA after 7 days
├── Transition to Glacier after 30 days
└── Expire after 365 days

Rule 2: Processed data (prefix: processed/)
├── Transition to Intelligent-Tiering after 1 day
├── Enable Archive Access tier (90 days)
└── Enable Deep Archive tier (180 days)

Rule 3: Compliance data (prefix: audit/)
├── Transition to Glacier Instant after 1 day
├── Transition to Deep Archive after 365 days
├── Object Lock: GOVERNANCE mode, 7 years
└── NEVER expire

Rule 4: Temp/staging (prefix: tmp/)
├── Expire after 1 day
└── Abort incomplete multipart after 1 day

Rule 5: All objects (no prefix)
├── Abort incomplete multipart uploads after 7 days
├── Expire old noncurrent versions after 30 days
└── Clean up delete markers
```

**Cost savings example:**
```
100TB in Standard: $2,300/month
After lifecycle optimization:
├── 10TB Standard: $230
├── 30TB Standard-IA: $375
├── 40TB Glacier: $160
└── 20TB Deep Archive: $20
Total: $785/month (66% savings)
```

---

## Security & Access Control

**Q6: Explain S3 bucket policy vs ACL vs IAM policy. When does each take precedence?**

**A:**

| Type | Scope | Attached To | Cross-Account |
|------|-------|-------------|---------------|
| IAM Policy | User/Role permissions | IAM entity | Need both IAM + resource policy |
| Bucket Policy | Bucket-level resource policy | S3 bucket | Can grant directly |
| ACL | Legacy per-object/bucket | Object or bucket | Limited (predefined groups) |
| Access Points | Simplified per-application access | Access point | Scoped policies |

**Evaluation logic:**
```
1. Explicit DENY anywhere → DENY
2. Bucket policy ALLOW + IAM ALLOW (same account) → ALLOW
3. Bucket policy ALLOW (cross-account) + IAM ALLOW in other account → ALLOW
4. Only IAM ALLOW (no bucket policy) same account → ALLOW
5. Nothing allows → DENY
```

**Tricky**: For cross-account access, BOTH sides must allow:
- Account A's bucket policy must allow Account B
- Account B's IAM policy must allow S3 access
- If EITHER denies, access is denied

---

**Q7: How do you make an S3 bucket truly private and prevent accidental public exposure?**

**A:**

**Layer 1: Block Public Access (account-level + bucket-level):**
```json
{
  "BlockPublicAcls": true,
  "IgnorePublicAcls": true,
  "BlockPublicPolicy": true,
  "RestrictPublicBuckets": true
}
```

**Layer 2: Bucket Policy (deny non-SSL, deny non-VPC):**
```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
      "Condition": {"Bool": {"aws:SecureTransport": "false"}}
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {"StringNotEquals": {"aws:sourceVpc": "vpc-123456"}}
    }
  ]
}
```

**Layer 3: VPC Endpoint (keep traffic private):**
- Gateway endpoint for S3 (free, no internet transit)
- Endpoint policy restricts which buckets are accessible

**Layer 4: SCP (organization-level prevention):**
- Deny `s3:PutBucketPublicAccessBlock` with non-blocking values
- Deny `s3:PutBucketPolicy` without encryption conditions

---

**Q8: Explain S3 Object Lock and how it prevents data deletion.**

**A:**

**Modes:**
| Mode | Can Delete? | Can Overwrite? | Who Can Override? |
|------|-------------|----------------|-------------------|
| Governance | Yes (with permission) | Yes (with permission) | Users with `s3:BypassGovernanceRetention` |
| Compliance | NO (nobody, not even root) | NO | NOBODY until retention expires |

**Legal Hold**: Separate from retention. Prevents deletion regardless of retention period. Can be set/removed independently.

```bash
# Set retention
aws s3api put-object-retention --bucket my-bucket --key file.txt \
  --retention '{"Mode":"COMPLIANCE","RetainUntilDate":"2030-01-01T00:00:00Z"}'

# Set legal hold
aws s3api put-object-legal-hold --bucket my-bucket --key file.txt \
  --legal-hold '{"Status":"ON"}'
```

**Tricky**: 
- Object Lock requires versioning enabled (can't disable versioning once Object Lock is on)
- Compliance mode: Even the AWS account ROOT user CANNOT delete the object!
- You CANNOT shorten a Compliance retention period (only extend it)
- Once a bucket has Object Lock enabled, it CANNOT be disabled

---

## Performance & Optimization

**Q9: How do you optimize S3 performance for high-throughput workloads?**

**A:**

**S3 performance limits per prefix:**
- 3,500 PUT/COPY/POST/DELETE requests per second per prefix
- 5,500 GET/HEAD requests per second per prefix

**Optimization strategies:**

1. **Randomize prefixes** (parallelize across partitions):
```
# BAD: All operations on one prefix
s3://bucket/2024/01/15/file1.csv

# GOOD: Distribute across prefixes
s3://bucket/a1b2/2024/01/15/file1.csv
s3://bucket/c3d4/2024/01/15/file2.csv
```

2. **Multipart upload** (for files > 100MB):
```bash
# Auto-multipart with AWS CLI
aws s3 cp large-file.zip s3://bucket/ --expected-size 5368709120
# Or configure: aws configure set s3.multipart_threshold 100MB
```

3. **S3 Transfer Acceleration** (for global uploads):
```
bucket.s3-accelerate.amazonaws.com
# Uses CloudFront edge locations for upload speedup
```

4. **Byte-range fetches** (parallel downloads):
```
GET /large-file Range: bytes=0-999999
GET /large-file Range: bytes=1000000-1999999
# Download parts in parallel, combine locally
```

5. **S3 Select / Glacier Select** (query in place):
```sql
SELECT s.name, s.age FROM s3object s WHERE s.age > 30
-- Only returns matching data (save bandwidth)
```

**Tricky**: The old guidance about adding random prefixes for performance was relaxed in 2018. S3 now auto-partitions based on request patterns. But for extreme throughput (>10K RPS), distributing across prefixes still helps.

---

**Q10: Explain S3 event notifications and how to build event-driven architectures.**

**A:**

**Event destinations:**
- Lambda functions
- SQS queues
- SNS topics
- EventBridge (most flexible — filtering, multiple targets)

**Event types:**
```
s3:ObjectCreated:*          (Put, Post, Copy, CompleteMultipartUpload)
s3:ObjectRemoved:*          (Delete, DeleteMarkerCreated)
s3:ObjectRestore:*          (Glacier restore initiated/completed)
s3:Replication:*            (Replication failed/completed)
s3:LifecycleTransition
s3:IntelligentTiering
s3:ObjectTagging:*
s3:ObjectAcl:Put
```

**Event-driven architecture:**
```
S3 Upload → EventBridge → Step Functions
                            ├── Lambda: Validate file
                            ├── Lambda: Extract metadata
                            ├── Lambda: Transcode/process
                            ├── Lambda: Update DynamoDB catalog
                            └── SNS: Notify users
```

**Tricky**: S3 event notifications deliver AT LEAST ONCE (duplicates possible). Your processing Lambda must be idempotent! Use DynamoDB conditional writes or deduplication logic.

---

## Replication & Versioning

**Q11: Explain S3 Cross-Region Replication (CRR) vs Same-Region Replication (SRR). What are the gotchas?**

**A:**

| Feature | CRR | SRR |
|---------|-----|-----|
| Purpose | DR, compliance, latency | Log aggregation, dev/prod sync |
| Regions | Different regions | Same region |
| Versioning required | Yes (both buckets) | Yes (both buckets) |
| Existing objects | NOT replicated (must enable Batch Replication) | NOT replicated |
| Delete markers | Optional (can replicate or not) | Optional |
| Storage class | Can change on replica | Can change on replica |
| Encryption | Supports SSE-S3, SSE-KMS | Same |

**Gotchas:**
1. **Existing objects NOT replicated**: Only NEW objects after enabling replication
2. **Delete markers**: By default NOT replicated (source delete doesn't delete replica)
3. **Lifecycle actions NOT replicated**: Each bucket needs its own lifecycle rules
4. **Object Lock retention NOT replicated** by default
5. **Chaining not supported**: A→B→C doesn't work (B's replicas don't replicate to C)
6. **KMS encrypted objects**: Need explicit configuration + KMS key in destination region

**Batch Replication** (for existing objects):
```bash
aws s3control create-job \
  --account-id 123456789 \
  --operation '{"S3ReplicateObject":{}}' \
  --manifest '{"Spec":{"Format":"S3BatchOperations_CSV_20180820"}}'
```

---

**Q12: How does S3 versioning work with deletes? What are delete markers?**

**A:**

```
Without versioning:
DELETE object → Object GONE (permanent)

With versioning:
DELETE object → Creates "delete marker" (object appears deleted)
                Original versions still exist!

GET object → 404 (delete marker is "current")
GET object?versionId=abc → Returns original version!

To permanently delete:
DELETE object?versionId=abc → Removes specific version
DELETE object?versionId=deleteMarkerID → Removes delete marker (undeletes!)
```

**MFA Delete** (extra protection):
- Requires MFA to permanently delete versions or change versioning state
- Can only be enabled by root account via CLI (not console!)
- Prevents accidental or malicious permanent deletion

```bash
# Enable MFA Delete (root account only)
aws s3api put-bucket-versioning \
  --bucket my-bucket \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456:mfa/root-device 123456"
```

**Tricky**: Suspending versioning doesn't delete existing versions! It just stops creating new versions. Old versions remain and still cost money. You must explicitly delete them or use lifecycle rules for noncurrent versions.

---

## Tricky Scenarios

**Q13: Your S3 bucket has 10 million objects and you need to delete all of them. What's the fastest approach?**

**A:**

**Option 1: Lifecycle rule (easiest, but slow — up to 48 hours):**
```json
{
  "Rules": [{
    "ID": "delete-all",
    "Status": "Enabled",
    "Filter": {},
    "Expiration": {"Days": 1}
  }]
}
```

**Option 2: S3 Batch Operations (faster, controlled):**
```bash
# Generate inventory first
# Then create batch delete job
aws s3control create-job \
  --operation '{"S3DeleteObject":{}}' \
  --manifest-location '...'
```

**Option 3: Parallel CLI delete (fast for moderate counts):**
```bash
# Fastest CLI approach
aws s3 rm s3://bucket/ --recursive
# Uses 10 concurrent connections by default

# Faster with max concurrency
aws configure set s3.max_concurrent_requests 100
aws s3 rm s3://bucket/ --recursive
```

**Option 4: Delete the bucket (if you don't need to keep it):**
```bash
# Must empty bucket first (or use force-delete tools)
# Fastest: lifecycle rule → wait → delete bucket
```

**Tricky**: With versioning enabled, `aws s3 rm --recursive` only creates delete markers! All versions remain. To truly delete everything:
```bash
# Delete all versions + delete markers
aws s3api list-object-versions --bucket my-bucket --output json | \
  jq -r '.Versions[]?, .DeleteMarkers[]? | "\(.Key)\t\(.VersionId)"' | \
  while IFS=$'\t' read key vid; do
    aws s3api delete-object --bucket my-bucket --key "$key" --version-id "$vid"
  done
```

---

**Q14: S3 presigned URLs — how do they work and what are the security considerations?**

**A:**

```python
import boto3
s3 = boto3.client('s3')

# Generate presigned URL (default: 1 hour expiry)
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'private/file.pdf'},
    ExpiresIn=3600  # seconds
)
# URL contains: signature, expiry, credentials reference
```

**Security considerations:**
1. **Anyone with the URL can access** (no additional auth needed)
2. **URL is valid until expiry** even if you revoke IAM credentials (if STS session is still valid)
3. **Permissions checked at URL USE time** (not creation time)
4. **If creator's permissions are revoked** → URL stops working immediately
5. **Maximum expiry**: 7 days (IAM user), role session duration (for roles)

**Tricky scenarios:**
- Presigned URL created by IAM role with 1h session → URL won't work after session expires (even if ExpiresIn=7d)
- Bucket policy denying VPC access → presigned URL from outside VPC fails
- URL includes region → doesn't work against bucket in different region

**Best practices:**
- Short expiry times (minutes, not days)
- Use CloudFront signed URLs for distribution (more features)
- Add conditions: IP restriction, content-type enforcement

---

**Q15: Explain S3 Select vs Athena vs Redshift Spectrum for querying S3 data.**

**A:**

| Feature | S3 Select | Athena | Redshift Spectrum |
|---------|-----------|--------|-------------------|
| Query scope | Single object | Entire data lake | Entire data lake |
| Language | Subset of SQL | Full ANSI SQL | Full SQL |
| Performance | Fast (single object) | Moderate (serverless) | Fast (dedicated cluster) |
| Cost model | Per GB scanned | Per TB scanned ($5/TB) | Cluster + per TB scanned |
| Setup | None | Glue Catalog | Redshift cluster needed |
| Joins | No | Yes | Yes |
| Format support | CSV, JSON, Parquet | CSV, JSON, Parquet, ORC, Avro | Same as Athena |
| Use case | Filter before download | Ad-hoc analytics | Regular analytics + warehouse |

**S3 Select use case:**
```python
# Instead of downloading 1GB CSV and filtering locally
# S3 Select returns only matching rows (save bandwidth)
result = s3.select_object_content(
    Bucket='data-lake',
    Key='logs/2024/01/access.csv.gz',
    Expression="SELECT * FROM s3object s WHERE s.status_code = '500'",
    ExpressionType='SQL',
    InputSerialization={'CSV': {'FileHeaderInfo': 'USE'}, 'CompressionType': 'GZIP'},
    OutputSerialization={'JSON': {}}
)
```

**Tricky**: S3 Select works on a SINGLE object. For querying across thousands of objects, use Athena. S3 Select is ideal when your application needs specific rows from a known file (e.g., Lambda processing individual files from S3 events).
