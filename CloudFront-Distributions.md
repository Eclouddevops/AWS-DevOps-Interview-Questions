# CloudFront Distributions — Deep-Dive Interview Q&A

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Caching & Performance](#caching--performance)
3. [Security](#security)
4. [Advanced Features](#advanced-features)
5. [Tricky Scenarios](#tricky-scenarios)

---

## Core Concepts


**Q1: Explain CloudFront architecture — Edge Locations, Regional Edge Caches, and Origin Shield.**

**A:**

```
Client → Edge Location (450+ globally)
           ↓ (cache miss)
         Regional Edge Cache (13 locations — larger cache)
           ↓ (cache miss)
         Origin Shield (single cache layer — optional)
           ↓ (cache miss)
         Origin (S3, ALB, EC2, custom HTTP)
```

| Layer | Purpose | Cache Size | Latency |
|-------|---------|-----------|---------|
| Edge Location | Closest to user, first cache check | Smaller | Lowest |
| Regional Edge Cache | Larger cache, reduces origin load | Larger | Low |
| Origin Shield | Single point of entry to origin | Large | Medium |
| Origin | Source of truth | N/A | Highest |

**Origin Shield benefits:**
- Reduces origin load (only one cache checks origin)
- Better cache hit ratio (aggregates requests from all edge locations)
- Reduces origin costs (fewer requests)
- Best for: Origins with limited capacity, expensive to serve

**Tricky**: Origin Shield adds latency on cache miss (extra hop). Don't use if your origin is fast and cheap. Use when origin is slow/expensive (dynamic media transcoding, API with rate limits).

---

**Q2: How does CloudFront caching work? Explain cache keys and cache policies.**

**A:**

**Cache key** determines what makes a cached object "unique":
```
Default cache key: URL path only
Example: https://example.com/images/logo.png → key = /images/logo.png

Custom cache key can include:
├── URL path (always included)
├── Query strings (all, specific, or none)
├── Headers (specific headers like Accept-Language)
├── Cookies (all, specific, or none)
└── Result: More key components = more cache variants = lower hit ratio
```

**Cache Policy (what's in cache key):**
```json
{
  "Name": "Custom-Policy",
  "MinTTL": 60,
  "MaxTTL": 86400,
  "DefaultTTL": 3600,
  "ParametersInCacheKeyAndForwardedToOrigin": {
    "HeadersConfig": {"Headers": ["Accept-Language", "CloudFront-Is-Mobile-Viewer"]},
    "QueryStringsConfig": {"QueryStrings": ["page", "sort"]},
    "CookiesConfig": {"Cookies": ["session_id"]}
  }
}
```

**Origin Request Policy** (what's forwarded but NOT in cache key):
- Forward headers/cookies to origin for processing
- But don't cache different variants for them

**Tricky**: If you include `Authorization` header in cache key, EVERY user gets their own cache copy (0% hit ratio!). Instead: Forward Authorization to origin via Origin Request Policy, but exclude from Cache Policy.

---

**Q3: Explain CloudFront cache invalidation vs versioned URLs. When to use each?**

**A:**

| Approach | Speed | Cost | Complexity |
|----------|-------|------|-----------|
| Invalidation | 5-15 minutes | $0.005 per path (first 1000/month free) | Simple |
| Versioned URLs | Instant | Free | Requires build system changes |

**Invalidation:**
```bash
aws cloudfront create-invalidation \
  --distribution-id E1234 \
  --paths "/images/*" "/css/style.css" "/*"
```

**Versioned URLs (preferred):**
```html
<!-- Old: /css/style.css (must invalidate on change) -->
<!-- New: /css/style.v2.1.css (different URL = no cache to invalidate) -->
<!-- Or: /css/style.css?v=abc123hash -->
<link rel="stylesheet" href="/css/style.abc123.css">
```

**Tricky**: 
- `/*` invalidation removes EVERYTHING from all edge caches (expensive at scale)
- Invalidation is per-distribution, not per-path — you can't target specific edge locations
- Invalidation doesn't remove from Regional Edge Caches immediately
- Wildcard `*` only works at the end: `/images/logo*` works, `/*/logo.png` does NOT

---

## Security

**Q4: How do you restrict access to an S3 origin so only CloudFront can access it?**

**A:**

**Method 1: Origin Access Control (OAC) — recommended:**
```json
{
  "S3OriginConfig": {
    "OriginAccessIdentity": ""
  },
  "OriginAccessControlId": "E1234ABCDEF"
}
```

S3 bucket policy:
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456:distribution/E1234"
      }
    }
  }]
}
```

**Method 2: Origin Access Identity (OAI) — legacy:**
- Creates special CloudFront identity
- S3 grants access to that identity
- Limited: Doesn't support SSE-KMS, no POST/PUT

**OAC advantages over OAI:**
- Supports SSE-KMS encrypted objects
- Supports S3 PUT/POST (upload via CloudFront)
- Supports all S3 regions
- Per-distribution granularity

**Tricky**: If S3 bucket has Block Public Access enabled AND no OAC/OAI configured, CloudFront gets 403. Must configure OAC and update bucket policy.

---

**Q5: Explain CloudFront signed URLs vs signed cookies. When to use each?**

**A:**

| Feature | Signed URLs | Signed Cookies |
|---------|------------|----------------|
| Scope | Single file | Multiple files (entire path) |
| URL modification | Yes (adds signature params) | No (cookie-based) |
| RTMP streaming | Supported | Not supported |
| Restrict by | IP, date/time, path | IP, date/time, path pattern |
| Use case | Individual downloads, API access | Media streaming, premium content areas |

**Signed URL:**
```python
from botocore.signers import CloudFrontSigner
import rsa

def sign_url(url, key_pair_id, private_key, expiry):
    signer = CloudFrontSigner(key_pair_id, rsa_signer)
    return signer.generate_presigned_url(url, date_less_than=expiry)
```

**Signed Cookie (for multiple files):**
```
Set-Cookie: CloudFront-Policy=<base64-encoded-policy>
Set-Cookie: CloudFront-Signature=<signature>
Set-Cookie: CloudFront-Key-Pair-Id=<key-pair-id>
```

**Tricky**: Signed URLs override signed cookies. If both are present, the URL signature is used. Also: Signed URLs don't work with CORS preflight (OPTIONS requests can't carry URL signatures).

---

**Q6: How do you implement geo-restriction and geo-routing with CloudFront?**

**A:**

**Geo-Restriction (built-in):**
- Whitelist: Only allow specific countries
- Blacklist: Block specific countries
- Based on: MaxMind GeoIP database (CloudFront-Viewer-Country header)

**Advanced geo-routing (Lambda@Edge/CloudFront Functions):**
```javascript
// CloudFront Function: Route to nearest origin based on country
function handler(event) {
    var request = event.request;
    var country = request.headers['cloudfront-viewer-country'].value;

    if (['US', 'CA', 'MX'].includes(country)) {
        request.origin = { custom: { domainName: 'us-origin.example.com' }};
    } else if (['GB', 'DE', 'FR'].includes(country)) {
        request.origin = { custom: { domainName: 'eu-origin.example.com' }};
    }
    return request;
}
```

**Tricky**: Geo-restriction returns 403 to blocked countries. But users with VPNs bypass it easily. For stronger restriction, combine with WAF geo-match rules (can also check additional signals).

---

## Advanced Features

**Q7: Explain CloudFront Functions vs Lambda@Edge. When to use each?**

**A:**

| Feature | CloudFront Functions | Lambda@Edge |
|---------|---------------------|-------------|
| Runtime | JavaScript (ES 5.1) | Node.js, Python |
| Execution time | < 1ms | 5s (viewer) / 30s (origin) |
| Memory | 2 MB | 128-10240 MB |
| Network access | No | Yes |
| Request body access | No | Yes (origin events) |
| Execution location | Edge (all 450+ locations) | Regional Edge Cache (13 locations) |
| Scale | Millions RPS | Thousands RPS |
| Triggers | Viewer request/response | All 4 events |
| Price | $0.10/million | $0.60/million + duration |

**CloudFront Functions use cases:**
- URL rewrites/redirects
- Header manipulation (add security headers, CORS)
- Cache key normalization (lowercase, sort query params)
- Simple A/B testing (cookie-based routing)
- JWT validation (simple token check)

**Lambda@Edge use cases:**
- Origin selection (route to different backends)
- Image resizing/optimization
- Bot detection with external API calls
- Complex authentication (calling auth service)
- Server-side rendering

---

**Q8: How do you implement multi-origin routing with CloudFront?**

**A:**

```
Distribution:
├── Behavior 1: /api/* → Origin: ALB (API servers)
├── Behavior 2: /static/* → Origin: S3 bucket (static assets)
├── Behavior 3: /media/* → Origin: Media S3 bucket
├── Behavior 4: /ws/* → Origin: WebSocket server (NLB)
└── Default (*) → Origin: S3 bucket (SPA index.html)

Origin Groups (failover):
├── Primary: ALB us-east-1
└── Secondary: ALB us-west-2
    Failover on: 500, 502, 503, 504, 403, 404
```

**Origin Groups for HA:**
```json
{
  "OriginGroup": {
    "Members": {
      "Items": [
        {"OriginId": "primary-alb"},
        {"OriginId": "failover-alb"}
      ]
    },
    "FailoverCriteria": {
      "StatusCodes": {"Items": [500, 502, 503, 504]}
    }
  }
}
```

**Tricky**: Origin failover only triggers on HTTP status codes from origin. If the primary origin is unreachable (timeout), CloudFront retries 3 times before failing over. This adds ~30s latency during failover. Set lower origin timeout for faster failover.

---

## Tricky Scenarios

**Q9: CloudFront is serving stale content after deployment. How do you force fresh content?**

**A:**

**Diagnosis:**
```bash
# Check cache hit/miss status
curl -I https://example.com/app.js
# Look for: X-Cache: Hit from cloudfront (stale)
# Or: X-Cache: Miss from cloudfront (fresh)

# Check which edge served it
# Look for: X-Amz-Cf-Pop: IAD89-P1 (edge location)
```

**Solutions (in order of preference):**

1. **Versioned filenames** (best — no invalidation needed):
```
app.v2.1.js instead of app.js
app.abc123.css (hash-based)
```

2. **Reduce TTL** for frequently changing content:
```
Cache-Control: max-age=60, s-maxage=3600
# Browsers cache 60s, CloudFront caches 3600s
```

3. **Invalidation** (last resort):
```bash
aws cloudfront create-invalidation --distribution-id E1234 --paths "/*"
# Wait 5-15 minutes for propagation to all edge locations
```

4. **Cache-Control: no-cache from origin**:
```
# Origin responds with: Cache-Control: no-cache
# CloudFront still caches but revalidates with origin every time (conditional GET)
```

**Tricky**: `Cache-Control: no-store` = never cache. `Cache-Control: no-cache` = cache but revalidate. These are frequently confused! Use `no-cache` for HTML pages (allows CDN caching with instant freshness), `no-store` for truly uncacheable data.

---

**Q10: You need to serve a single-page application (SPA) from S3 via CloudFront. What are the common issues?**

**A:**

**Issue 1: 403/404 on page refresh** (direct URL access):
```
User navigates to /dashboard → CloudFront looks for /dashboard in S3 → 404!
Because SPA routes don't correspond to S3 objects.
```
**Fix**: Custom error response:
```json
{
  "ErrorCode": 403,
  "ResponseCode": 200,
  "ResponsePagePath": "/index.html",
  "ErrorCachingMinTTL": 0
}
```

**Issue 2: Caching HTML vs Assets differently:**
```
/index.html → Cache-Control: no-cache (always fresh)
/static/js/app.hash.js → Cache-Control: max-age=31536000 (1 year)
```
Create separate cache behaviors:
- `/index.html` → short TTL or no-cache
- `/static/*` → long TTL

**Issue 3: CORS errors for API calls:**
- CloudFront must forward `Origin` header to API origin
- API must return `Access-Control-Allow-Origin`
- Configure CloudFront to cache based on `Origin` header

**Issue 4: HTTP to HTTPS redirect:**
- Set viewer protocol policy to "Redirect HTTP to HTTPS"
- Add `Strict-Transport-Security` header via CloudFront Function

---

**Q11: How do you monitor CloudFront performance and troubleshoot latency issues?**

**A:**

**Built-in metrics (CloudWatch):**
- `Requests`: Total requests
- `BytesDownloaded/BytesUploaded`: Data transfer
- `4xxErrorRate` / `5xxErrorRate`: Error percentages
- `TotalErrorRate`: Combined
- `CacheHitRate`: Cache efficiency (aim for >90%)
- `OriginLatency`: Time for origin to respond

**Real-time logs (Kinesis Data Streams):**
```
Fields: timestamp, edge-location, response-result-type,
        time-to-first-byte, client-ip, edge-response-result-type
```

**Troubleshooting high latency:**
1. **Low cache hit rate** → Review cache policy (too many cache key variants?)
2. **High origin latency** → Enable Origin Shield, optimize origin
3. **TTFB high at edge** → Check if origin is far from Regional Edge Cache
4. **SSL handshake slow** → Enable TLS session resumption, use TLS 1.3
5. **Large response size** → Enable compression (gzip/brotli)

**Enable compression:**
```
Behavior setting: Compress Objects Automatically = Yes
Works for: text/html, text/css, application/javascript, etc.
Requires: Origin sends without Content-Encoding, viewer sends Accept-Encoding
```

**Tricky**: CloudFront won't compress if origin already compressed the response (has Content-Encoding header). Remove compression at origin and let CloudFront handle it for optimal edge caching.



---

## Additional Scenario-Based Tricky Questions

---

**Q12: After deploying a new version of your React SPA, some users see the old version and some see the new version. Clearing browser cache doesn't help for affected users. What's happening?**

**A:**

**The cache layering problem:**
```
User → CloudFront Edge (cached old index.html) → Origin (has new index.html)

Timeline:
1. Old version deployed: index.html + main.abc123.js (hashed)
2. New version deployed: index.html + main.def456.js (hashed)
3. User A: CloudFront edge has cached old index.html (TTL not expired)
   → Gets old index.html → loads old main.abc123.js → sees OLD version
4. User B: Different edge location, cache expired
   → Gets new index.html → loads new main.def456.js → sees NEW version
```

**Why "clear browser cache" doesn't help:**
```
The problem is CloudFront cache, not browser cache.
Even if user clears browser cache:
- Request goes to CloudFront edge → edge returns cached OLD index.html
- Browser gets old index.html → references old JS bundle
- Old JS bundle also cached at edge → serves old version
```

**Fix:**
```bash
# Immediate: Invalidate CloudFront cache
aws cloudfront create-invalidation --distribution-id E123456 \
  --paths "/index.html" "/asset-manifest.json" "/service-worker.js"
# NOTE: Only invalidate non-hashed files. Hashed files (main.abc123.js) are safe
# because new index.html will reference main.def456.js (new hash)

# Better: Set proper cache headers per file type
# Origin (S3/ALB) should set:
# index.html: Cache-Control: no-cache, no-store, must-revalidate
# *.js, *.css (hashed): Cache-Control: public, max-age=31536000, immutable
# images (hashed): Cache-Control: public, max-age=31536000, immutable

# CloudFront behavior configuration:
# Path: /index.html → TTL: 0 (always revalidate with origin)
# Path: /static/* → TTL: 31536000 (1 year, content-hashed)
# Path: *.html → TTL: 0

# In CloudFront distribution:
aws cloudfront update-distribution --id E123456 \
  --default-cache-behavior '{
    "DefaultTTL": 0,
    "MaxTTL": 0,
    "MinTTL": 0,
    "ForwardedValues": {"QueryString": false, "Cookies": {"Forward": "none"}}
  }'
```

**Proper SPA deployment strategy:**
```bash
#!/bin/bash
# deploy-spa.sh

# 1. Upload hashed assets FIRST (they're new URLs, no conflict)
aws s3 sync build/static/ s3://my-bucket/static/ \
  --cache-control "public,max-age=31536000,immutable"

# 2. Upload index.html LAST (switches to new version)
aws s3 cp build/index.html s3://my-bucket/index.html \
  --cache-control "no-cache,no-store,must-revalidate"

# 3. Invalidate index.html at CloudFront edge
aws cloudfront create-invalidation --distribution-id $CF_DIST \
  --paths "/index.html"

# 4. Wait for invalidation (takes 30s-5min)
aws cloudfront wait invalidation-completed --distribution-id $CF_DIST \
  --id $INVALIDATION_ID
```

**Tricky**: CloudFront invalidation costs money after 1,000 paths/month ($0.005 per path). Invalidating `/*` counts as ONE path but invalidates everything (cache efficiency destroyed for all assets). The optimal approach: NEVER invalidate hashed assets, ONLY invalidate `index.html` and other non-hashed entry points. This preserves cache for 99% of requests (static assets) while ensuring fresh content for the HTML shell.

---

**Q13: Your CloudFront distribution serves API responses (dynamic content) with `Cache-Control: no-store`. But you're still paying significant data transfer costs through CloudFront. A colleague asks: "If nothing is cached, why use CloudFront at all?" What's your answer?**

**A:**

**Benefits of CloudFront even without caching:**
```
1. LOWER DATA TRANSFER COST:
   - EC2/ALB → Internet: $0.09/GB
   - CloudFront → Internet: $0.085/GB (US/EU)
   - Savings: 5-15% on data transfer alone
   - At 100TB/month: saves $500-$1,500/month

2. GLOBAL ACCELERATION:
   - User connects to nearest CloudFront edge (200+ locations)
   - Edge → Origin uses AWS backbone network (faster than public internet)
   - Reduces latency by 20-60% for distant users
   - TCP connection reuse between edge and origin

3. DDoS PROTECTION (included free):
   - CloudFront = AWS Shield Standard (automatic L3/L4 protection)
   - Absorbs volumetric attacks at the edge (before reaching origin)
   - No extra cost for Shield Standard on CloudFront

4. SSL TERMINATION AT EDGE:
   - TLS handshake at nearest edge (low latency)
   - HTTP/2 and HTTP/3 at edge (even if origin only speaks HTTP/1.1)
   
5. WAF INTEGRATION:
   - CloudFront + WAF blocks malicious traffic before it reaches origin
   - Origin only sees legitimate traffic = less compute needed

6. CONNECTION COALESCING:
   - 10,000 users connect to edge → edge maintains 10 connections to origin
   - Origin handles 10 connections instead of 10,000
   - Massive reduction in origin load
```

**When to NOT use CloudFront for APIs:**
```
- WebSocket connections (CloudFront has 30-second idle timeout)
- Server-Sent Events (SSE) — CloudFront buffers responses
- APIs that need real-time with <50ms latency to origin (edge adds 1-5ms)
- Internal APIs (within same VPC) — use ALB directly
```

**Tricky**: Even with `Cache-Control: no-store`, CloudFront can still serve a "conditional" cache using `If-Modified-Since` or `If-None-Match` headers. If origin returns 304 Not Modified, CloudFront serves from cache → reduces origin load and bandwidth. This is different from "no caching" — it's "revalidation caching." For truly no-cache behavior, both the origin must respond with no-store AND CloudFront cache policy must have TTL=0.

---

**Q14: CloudFront is returning stale content even after you invalidated the cache. The invalidation shows "Completed" in the console. Users in some regions get old content while others get new content. Why?**

**A:**

**Why invalidation "completed" but content is stale:**
```
1. MULTIPLE CACHE LAYERS:
   CloudFront has TWO cache tiers:
   - Regional Edge Caches (13 locations) ← Invalidation hits these
   - Edge Locations (200+ locations) ← May still have old content
   
   Invalidation propagates: API → Regional → Edge locations
   It can take up to 10-15 minutes for ALL 200+ edges to clear
   "Completed" means AWS accepted the request, not that all edges cleared

2. BROWSER/ISP CACHE:
   - CloudFront sent "max-age=86400" with old response
   - User's browser cached for 24 hours
   - CloudFront invalidation doesn't affect browser cache!
   - Users must hard-refresh (Ctrl+Shift+R) or wait for TTL expiry

3. QUERY STRING VARIATION:
   - You invalidated: /api/products
   - But users request: /api/products?v=123
   - If CloudFront forwards query strings → different cache key!
   - Must invalidate: /api/products* (with wildcard)

4. CUSTOM CACHE KEY (cookies, headers):
   - Cache policy includes "Accept-Language" header
   - Invalidation clears /index.html for ALL variations
   - But if policy uses cookies → may need cookie-specific invalidation
```

**Proper invalidation:**
```bash
# Invalidate with wildcard (catches all query string variations)
aws cloudfront create-invalidation --distribution-id E123456 \
  --paths "/api/products*" "/static/*"

# For truly instant global update — use versioned URLs
# Old: /api/v1/products → Cache-Control: max-age=300
# New: /api/v2/products → Completely new cache key
# No invalidation needed! New URL = cache miss everywhere

# Or use Cache-Control: s-maxage=0 + stale-while-revalidate
# Response: Cache-Control: s-maxage=0, stale-while-revalidate=300
# CloudFront serves stale immediately but revalidates in background
# Next request gets fresh content
```

**Tricky**: CloudFront `Invalidation Completed` is misleading. It means the invalidation request was processed by the CloudFront control plane, NOT that all 200+ edge locations have purged their cache. Full global propagation can take up to 15 minutes after "Completed" status. For business-critical updates, use versioned URLs (cache-busting) instead of invalidation. Also, invalidating `/*` takes longer than specific paths because it must traverse all objects in the distribution.

---
