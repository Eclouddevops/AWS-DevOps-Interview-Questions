# AWS Networking & VPC — Production Tricky Interview Q&A

## Table of Contents
1. [VPC Design & Subnetting in Production](#vpc-design--subnetting-in-production)
2. [Transit Gateway & Multi-Account Networking](#transit-gateway--multi-account-networking)
3. [PrivateLink & VPC Endpoints](#privatelink--vpc-endpoints)
4. [NAT Gateway & Internet Connectivity Issues](#nat-gateway--internet-connectivity-issues)
5. [DNS & Route 53 Production Problems](#dns--route-53-production-problems)
6. [Network Troubleshooting War Stories](#network-troubleshooting-war-stories)
7. [Security Groups vs NACLs — Production Gotchas](#security-groups-vs-nacls--production-gotchas)

---

## VPC Design & Subnetting in Production

---


**Q1: Your company is growing from 3 AWS accounts to 50. The current VPC uses 10.0.0.0/16 in every account. Why is this a ticking time bomb and how do you fix it?**

**A:**

**The problem:**
```
Account A: VPC 10.0.0.0/16
Account B: VPC 10.0.0.0/16
Account C: VPC 10.0.0.0/16

When you try to peer these VPCs or connect via Transit Gateway:
ERROR: "Cannot create peering - CIDR blocks overlap"

You CANNOT peer VPCs with overlapping CIDRs. Period.
50 accounts with same CIDR = complete networking isolation. 
No cross-account communication possible without ugly workarounds.
```

**The proper IPAM (IP Address Management) strategy:**
```
Company supernet: 10.0.0.0/8 (16 million IPs)

Allocation scheme:
├── 10.0.0.0/16   → Production account (65,536 IPs)
├── 10.1.0.0/16   → Staging account
├── 10.2.0.0/16   → Development account
├── 10.3.0.0/16   → Shared Services (CI/CD, monitoring)
├── 10.4.0.0/16   → Security/Audit account
├── 10.10.0.0/16  → Team A production
├── 10.11.0.0/16  → Team A staging
...
├── 10.50.0.0/16  → Team Z production
└── 172.16.0.0/12 → Reserved for on-premises connectivity

Within each VPC (10.0.0.0/16):
├── 10.0.0.0/20   → Public subnet AZ-a (4,096 IPs)
├── 10.0.16.0/20  → Public subnet AZ-b
├── 10.0.32.0/20  → Public subnet AZ-c
├── 10.0.48.0/19  → Private subnet AZ-a (8,192 IPs)
├── 10.0.80.0/19  → Private subnet AZ-b
├── 10.0.112.0/19 → Private subnet AZ-c
├── 10.0.144.0/20 → Database subnet AZ-a
├── 10.0.160.0/20 → Database subnet AZ-b
└── 10.0.176.0/20 → Database subnet AZ-c
```

**AWS VPC IPAM (the modern solution):**
```bash
# Create IPAM pool hierarchy
aws ec2 create-ipam-pool --ipam-scope-id scope-xxx \
  --address-family ipv4 --locale us-east-1

# Allocate CIDRs from the pool (prevents overlap automatically)
aws ec2 allocate-ipam-pool-cidr --ipam-pool-id pool-xxx \
  --netmask-length 16
# Returns: 10.5.0.0/16 (next available, guaranteed non-overlapping)
```

**Tricky**: Once a VPC CIDR is assigned, you CANNOT change it. You can only ADD secondary CIDRs (up to 5). If you started with 10.0.0.0/16 everywhere, your only options are: (1) migrate workloads to new VPCs with proper CIDRs, (2) use ALB/NLB as a bridge between overlapping VPCs, or (3) use PrivateLink to expose services across overlapping VPCs. Option 3 is the most common "fix" in production.

---

**Q2: Your EKS cluster is running out of IP addresses. Pods can't be scheduled because "insufficient IP addresses in subnet." The VPC has 10.0.0.0/16 (65k IPs). How are you already out?**

**A:**

**Why EKS consumes IPs so fast:**
```
Default AWS VPC CNI behavior:
- Each EC2 node reserves ENIs (Elastic Network Interfaces)
- Each ENI gets multiple secondary IPs
- Each pod gets a REAL VPC IP address (not overlay)

Example: m5.large
- Max ENIs: 3
- Max IPs per ENI: 10
- Max pods: (3 × 10) - 3 = 27 pods per node (minus ENI primary IPs)
- BUT: VPC CNI pre-allocates warm IPs (WARM_IP_TARGET=1 or WARM_ENI_TARGET=1)

With 50 nodes in a /24 subnet (254 IPs):
- 50 node primary IPs = 50
- 50 × 3 ENIs × 10 IPs = 1500 IPs needed!
- But subnet only has 254 → EXHAUSTION
```

**Solutions (used in production):**
```bash
# Solution 1: Use /19 or /18 subnets (8192+ IPs per AZ)
# This requires VPC redesign

# Solution 2: Enable prefix delegation (most common fix)
# Each ENI slot gets a /28 prefix (16 IPs) instead of 1 IP
kubectl set env daemonset/aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1
# Result: m5.large goes from 27 pods to 110 pods!

# Solution 3: Use secondary CIDR (100.64.0.0/16) for pods
# Add secondary CIDR to VPC
aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxx --cidr-block 100.64.0.0/16
# Configure VPC CNI to use secondary CIDR for pods
kubectl set env daemonset/aws-node -n kube-system \
  AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
# Create ENIConfig per AZ pointing to secondary subnets

# Solution 4: Use overlay networking (Calico/Cilium) instead of VPC CNI
# Pods get IPs from an overlay network — no VPC IPs consumed
# Trade-off: lose direct VPC integration, need NAT for pod-to-AWS-service traffic
```

**Tricky**: AWS reserves 5 IPs in every subnet (network address, VPC router, DNS, future use, broadcast). So a /24 has 251 usable IPs, not 256. A /28 (minimum) has only 11 usable IPs — barely enough for a NAT Gateway subnet. Also, the `100.64.0.0/16` CIDR (RFC 6598) is specifically designed for this use case and doesn't conflict with RFC 1918 private addressing. But it CANNOT be routed to on-premises networks over VPN/Direct Connect.

---


## Transit Gateway & Multi-Account Networking

---

**Q3: You're architecting networking for a company with 30 AWS accounts, 5 on-premises data centers, and 3 regions. They currently have 200+ VPC peering connections and it's unmanageable. Design the Transit Gateway architecture.**

**A:**

**Current mess (VPC Peering at scale):**
```
With N VPCs, you need N×(N-1)/2 peering connections
30 accounts × 3 regions = 90 VPCs
Peering needed: 90 × 89 / 2 = 4,005 connections!
Each needs route table entries in BOTH VPCs = 8,010 route entries
THIS IS UNMANAGEABLE.
```

**Transit Gateway architecture (hub-and-spoke):**
```
                    ┌─────────────────────────────────────┐
                    │         Transit Gateway (TGW)        │
                    │         Regional Hub                 │
                    └──┬───┬───┬───┬───┬───┬───┬───┬──┘
                       │   │   │   │   │   │   │   │
         ┌─────────────┤   │   │   │   │   │   │   ├─────────────┐
         │             │   │   │   │   │   │   │   │             │
    VPC-Prod-1    VPC-Prod-2  ...  VPC-Dev-1  VPN-DC1  VPN-DC2  DX-OnPrem
    
    Route table segmentation:
    ├── Production RT: Can reach other Prod VPCs + Shared Services + On-Prem
    ├── Development RT: Can reach other Dev VPCs + Shared Services ONLY
    ├── Shared Services RT: Reachable by ALL (monitoring, CI/CD, DNS)
    └── On-Premises RT: Can reach Prod + Shared, NOT Dev
```

**Implementation:**
```bash
# Create Transit Gateway
aws ec2 create-transit-gateway \
  --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,\
DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable

# Create route tables for segmentation
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-xxx  # Production
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-xxx  # Development
aws ec2 create-transit-gateway-route-table --transit-gateway-id tgw-xxx  # Shared

# Attach VPCs
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id tgw-xxx \
  --vpc-id vpc-prod-1 \
  --subnet-ids subnet-az1 subnet-az2 subnet-az3

# Associate with correct route table
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id tgw-rtb-prod \
  --transit-gateway-attachment-id tgw-attach-prod1

# Share TGW across accounts using AWS RAM
aws ram create-resource-share --name "TGW-Share" \
  --resource-arns arn:aws:ec2:us-east-1:111111:transit-gateway/tgw-xxx \
  --principals 222222222222 333333333333  # Other account IDs
```

**Multi-region with TGW peering:**
```
US-East-1 TGW ←──── TGW Peering ────→ EU-West-1 TGW
     │                                        │
     ├── US Prod VPCs                        ├── EU Prod VPCs
     ├── US Dev VPCs                         ├── EU Dev VPCs
     └── US On-Prem (VPN)                   └── EU On-Prem (DX)
```

**Tricky**: Transit Gateway charges per attachment ($0.05/hour = $36/month per VPC) PLUS data processing ($0.02/GB). With 90 VPCs: $3,240/month just for attachments + data costs. Also, TGW has a 50 Gbps bandwidth limit per VPC attachment (shared across all AZs). If one VPC needs more than 50 Gbps to TGW, you need multiple attachments or Direct Connect. Finally, TGW peering between regions does NOT support route propagation — you must add static routes manually.

---

**Q4: Your application in VPC-A needs to call an API in VPC-B (different AWS account, same region). VPC-B team doesn't want to expose their service to VPC-A's entire CIDR range. What's the production-grade solution?**

**A:**

**AWS PrivateLink (the answer for cross-account service exposure):**
```
VPC-B (Service Provider):
┌────────────────────────────────────────────────┐
│  NLB (Network Load Balancer)                    │
│    ↓                                            │
│  Target Group (Service instances)               │
│    ↓                                            │
│  VPC Endpoint Service                           │
│  (Allowlist: Only Account A can connect)        │
└────────────────────────────────────────────────┘
         │ PrivateLink (ENI in consumer VPC)
         ↓
VPC-A (Consumer):
┌────────────────────────────────────────────────┐
│  VPC Interface Endpoint                         │
│  (Gets private IP: 10.0.5.23)                  │
│    ↑                                            │
│  Application pods call 10.0.5.23:443            │
└────────────────────────────────────────────────┘
```

**Why PrivateLink over VPC Peering here:**
```
VPC Peering:
❌ Exposes entire VPC-B network to VPC-A
❌ Requires non-overlapping CIDRs
❌ Bidirectional (VPC-B can reach VPC-A too)
❌ All routes must be managed in both VPCs

PrivateLink:
✓ Only exposes ONE specific service
✓ Works with overlapping CIDRs (!)
✓ Unidirectional (VPC-B can't reach VPC-A)
✓ No route table changes needed
✓ Service provider controls who can connect
✓ Traffic stays on AWS backbone (never touches internet)
```

**Setup in VPC-B (provider):**
```bash
# Create NLB for the service
aws elbv2 create-load-balancer --name internal-api-nlb \
  --type network --scheme internal \
  --subnets subnet-1 subnet-2 subnet-3

# Create VPC Endpoint Service
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:...:loadbalancer/net/internal-api-nlb/xxx \
  --acceptance-required  # Manual approval for new consumers

# Allowlist Account A
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-xxx \
  --add-allowed-principals arn:aws:iam::ACCOUNT-A-ID:root
```

**Setup in VPC-A (consumer):**
```bash
# Create Interface Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-aaa \
  --service-name com.amazonaws.vpce.us-east-1.vpce-svc-xxx \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-a1 subnet-a2 \
  --security-group-ids sg-consumer

# Get the endpoint DNS name
aws ec2 describe-vpc-endpoints --query 'VpcEndpoints[].DnsEntries'
# Use this DNS name in your application config
```

**Tricky**: PrivateLink uses ENIs in the CONSUMER's VPC, consuming the consumer's IPs. Each AZ where you create the endpoint uses one IP. Also, PrivateLink traffic costs: $0.01/GB processed + $0.013/hour per AZ. For high-throughput services (TB/day), this adds up fast. Evaluate whether TGW is cheaper at scale. The break-even is approximately 5-10 services — below that PrivateLink is cheaper, above that TGW is more economical.

---


## NAT Gateway & Internet Connectivity Issues

---

**Q5: Your Lambda functions in a private subnet suddenly can't reach the internet (they need to call an external API). They worked yesterday. No infrastructure changes were made. What happened?**

**A:**

**Debugging steps:**
```bash
# Step 1: Check NAT Gateway status
aws ec2 describe-nat-gateways --filter Name=state,Values=available
# If state = "failed" → NAT GW is dead (rare but happens)

# Step 2: Check Elastic IP associated with NAT GW
aws ec2 describe-addresses --filter Name=association-id,Values=eipassoc-xxx
# If EIP was released → NAT GW loses internet connectivity

# Step 3: Check route table for private subnet
aws ec2 describe-route-tables --route-table-ids rtb-private
# Look for: 0.0.0.0/0 → nat-xxx
# If route is missing or pointing to blackhole → no internet

# Step 4: Check NAT Gateway's subnet route table
# NAT GW must be in PUBLIC subnet with route to IGW
aws ec2 describe-route-tables --route-table-ids rtb-public
# Must have: 0.0.0.0/0 → igw-xxx

# Step 5: Check NAT GW CloudWatch metrics
# ErrorPortAllocation → port exhaustion (55,000 limit)
# PacketsDropCount → packets being dropped
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
  --metric-name ErrorPortAllocation --period 300 --statistics Sum \
  --dimensions Name=NatGatewayId,Value=nat-xxx
```

**The actual cause (90% of the time):**
```
Cause 1: NAT Gateway port exhaustion
- Each NAT GW supports 55,000 simultaneous connections PER destination IP
- If Lambda functions all call the same API endpoint → 55k limit hit
- Fix: Use multiple NAT GWs or add more EIPs to the NAT GW

Cause 2: Route table was modified by another team's Terraform
- Someone added a more specific route that matches the destination
- e.g., 0.0.0.0/0 → nat-gw, but new route 52.0.0.0/8 → blackhole
- The API endpoint is in 52.x.x.x range → takes the more specific route

Cause 3: Security Group on Lambda ENI
- Lambda in VPC gets an ENI with a security group
- SG allows outbound to 0.0.0.0/0 on port 443
- But someone added a DENY rule (wait — SGs don't have deny rules!)
- Actually: NACL on the subnet was modified (NACLs DO have deny rules)

Cause 4: "No changes made" but DNS changed
- API endpoint resolved to new IP
- NACL allows old IP range but not new IP range
- Or: Route table has specific route for old IP but not new IP
```

**Lambda VPC networking diagram:**
```
Lambda Function (private subnet)
    ↓ ENI (Security Group)
Private Subnet (NACL)
    ↓ Route Table: 0.0.0.0/0 → NAT Gateway
NAT Gateway (public subnet)
    ↓ Route Table: 0.0.0.0/0 → Internet Gateway
Internet Gateway
    ↓
External API

ANY break in this chain = Lambda can't reach internet
```

**Tricky**: Lambda functions in a VPC lose access to ALL AWS services (S3, DynamoDB, SQS, etc.) unless you either: (1) have a NAT Gateway, or (2) create VPC Endpoints for each service. VPC endpoints are cheaper and faster than routing through NAT GW for AWS services. But here's the gotcha: Lambda's default timeout is 3 seconds. If the ENI attachment takes time (cold start), the request might timeout before network is ready. Set Lambda timeout to at least 30s for VPC-attached functions.

---

**Q6: Your production NAT Gateway bill jumped from $500/month to $8,000/month. No traffic increase. What's happening and how do you investigate?**

**A:**

**NAT Gateway pricing breakdown:**
```
- Hourly charge: $0.045/hour per NAT GW (~$32/month)  ← negligible
- Data processing: $0.045/GB                          ← THIS IS EXPENSIVE

$8000/month ÷ $0.045/GB = 177 TB of data through NAT GW!
That's 5.9 TB/day. Something is VERY wrong.
```

**Investigation:**
```bash
# Step 1: Enable VPC Flow Logs (if not already)
aws ec2 create-flow-logs --resource-type VPC --resource-ids vpc-xxx \
  --traffic-type ALL --log-destination-type s3 \
  --log-destination arn:aws:s3:::flow-logs-bucket

# Step 2: Query flow logs to find top talkers
# Using Athena:
SELECT srcaddr, dstaddr, dstport, SUM(bytes) as total_bytes
FROM vpc_flow_logs
WHERE action = 'ACCEPT' AND interface_id IN (
  SELECT interface_id FROM nat_gateway_interfaces
)
GROUP BY srcaddr, dstaddr, dstport
ORDER BY total_bytes DESC
LIMIT 20;

# Step 3: Check NAT GW metrics for data volume by AZ
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination --period 3600 --statistics Sum \
  --dimensions Name=NatGatewayId,Value=nat-xxx
```

**Common culprits in production:**
```
1. S3 TRAFFIC THROUGH NAT GW (most common!)
   - Application in private subnet accessing S3
   - Without S3 VPC Endpoint, all S3 traffic goes through NAT GW
   - S3 VPC Gateway Endpoint = FREE, zero data processing charge
   - Fix: aws ec2 create-vpc-endpoint --service-name com.amazonaws.us-east-1.s3

2. ECR IMAGE PULLS THROUGH NAT GW
   - Every pod restart pulls container images through NAT GW
   - 500MB image × 100 pods × 3 restarts/day = 150GB/day
   - Fix: Create ECR VPC endpoint + S3 endpoint (ECR uses S3 for layers)

3. CLOUDWATCH LOGS THROUGH NAT GW
   - Every log line from every container goes through NAT GW
   - Fix: Create CloudWatch Logs VPC endpoint

4. CROSS-AZ TRAFFIC VIA NAT GW
   - NAT GW in AZ-a, workloads in AZ-b route through it
   - Cross-AZ data transfer: $0.01/GB BOTH WAYS + NAT processing
   - Fix: One NAT GW per AZ (costs more hourly but saves on data)

5. DATABASE REPLICATION THROUGH NAT GW
   - RDS Multi-AZ replication should use VPC internal routing
   - But if RDS is in different VPC and routes through NAT... expensive
```

**The $7,500 savings fix:**
```bash
# Create FREE VPC Gateway Endpoints (S3, DynamoDB)
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-private-a rtb-private-b rtb-private-c

aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --route-table-ids rtb-private-a rtb-private-b rtb-private-c

# Create Interface Endpoints for other services
aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-a subnet-b

aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-a subnet-b

aws ec2 create-vpc-endpoint --vpc-id vpc-xxx \
  --service-name com.amazonaws.us-east-1.logs \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-a subnet-b
```

**Tricky**: S3 Gateway Endpoint is FREE (no hourly charge, no data charge). S3 Interface Endpoint costs $0.01/GB + $0.013/hour per AZ. ALWAYS use Gateway Endpoint for S3 unless you specifically need DNS resolution from on-premises (Gateway Endpoints only work from within the VPC). Also, after creating the S3 Gateway Endpoint, existing connections still go through NAT GW until they close and reconnect. You won't see immediate cost reduction — it takes 24-48 hours for all connections to use the new route.

---


## DNS & Route 53 Production Problems

---

**Q7: Your application uses Route 53 private hosted zones. After migrating to a new VPC, services can resolve public DNS names but NOT the internal ones (e.g., `database.internal.company.com`). What's wrong?**

**A:**

**Root cause:**
```
Private Hosted Zone (PHZ) must be ASSOCIATED with each VPC that needs to resolve it.

Old VPC: PHZ "internal.company.com" → Associated ✓
New VPC: PHZ "internal.company.com" → NOT Associated ✗

Services in new VPC try to resolve "database.internal.company.com"
→ Route 53 doesn't serve PHZ records to unassociated VPCs
→ Falls through to public DNS
→ NXDOMAIN (doesn't exist publicly)
```

**Fix:**
```bash
# Associate PHZ with new VPC
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z1234567890 \
  --vpc VPCRegion=us-east-1,VPCId=vpc-new

# If VPC is in different account — requires cross-account authorization
# Step 1 (PHZ owner account):
aws route53 create-vpc-association-authorization \
  --hosted-zone-id Z1234567890 \
  --vpc VPCRegion=us-east-1,VPCId=vpc-new-in-other-account

# Step 2 (VPC owner account):
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id Z1234567890 \
  --vpc VPCRegion=us-east-1,VPCId=vpc-new-in-other-account
```

**DNS resolution order in VPC:**
```
Query: database.internal.company.com

1. VPC DNS resolver (AmazonProvidedDNS at VPC CIDR + 2)
2. Check Route 53 Private Hosted Zones associated with this VPC
3. Check Route 53 Resolver Rules (forwarding rules)
4. If no match → forward to public DNS (Route 53 public hosted zones / internet)

Priority: PHZ wins over public hosted zone (same name)
```

**Tricky**: If you have BOTH a public hosted zone AND a private hosted zone for `internal.company.com`, resources in associated VPCs will ALWAYS get the private zone answers. Resources outside the VPC get public zone answers. This means you can have different DNS responses for the same name depending on WHERE the query originates. This is powerful for split-horizon DNS but confusing to debug if you don't know it exists.

---

**Q8: You set up Route 53 health checks with failover routing (primary in us-east-1, secondary in eu-west-1). During a real us-east-1 outage, failover DIDN'T happen. Users got errors for 45 minutes. Post-mortem: what went wrong?**

**A:**

**Common failover failures in production:**
```
Failure 1: Health check checking the WRONG thing
- Health check monitors: https://api.company.com/health
- But api.company.com resolves to... the Route 53 record itself!
- Health check hits the healthy region → always passes
- FIX: Health check should target the SPECIFIC regional endpoint
  Check: https://us-east-1.api.company.com/health (direct, not via DNS)

Failure 2: Health check from wrong locations
- Route 53 health checkers are in specific AWS regions
- If us-east-1 is down, health checkers IN us-east-1 can't reach your endpoint
- But health checkers in other regions CAN reach your endpoint (it's fine from their perspective)
- Result: 2/3 health checkers say healthy → no failover
- FIX: Health check must target a resource INSIDE the region
  e.g., ALB in us-east-1 → health checker must reach into us-east-1

Failure 3: Insufficient "failure threshold" time
- Health check interval: 30s, failure threshold: 3
- Time to detect: 30s × 3 = 90 seconds
- Then DNS TTL: 60 seconds (client caches old record)
- Total failover time: 90s + 60s = 2.5 minutes minimum
- If TTL was set to 300s (5 min) → 7.5 minutes before clients switch!

Failure 4: Health check endpoint has its own issues
- /health endpoint depends on external service
- That external service is also down in us-east-1
- But Route 53 health checker calls from outside → can reach /health
- /health returns 200 because it can reach the external service from its location
- FIX: /health must test dependencies from the APPLICATION'S perspective
```

**Proper failover configuration:**
```bash
# Health check targeting regional ALB directly (not DNS name)
aws route53 create-health-check --caller-reference unique-id \
  --health-check-config '{
    "IPAddress": "1.2.3.4",  
    "Port": 443,
    "Type": "HTTPS",
    "ResourcePath": "/health",
    "RequestInterval": 10,
    "FailureThreshold": 2,
    "Regions": ["us-east-1", "eu-west-1", "ap-southeast-1"]
  }'

# DNS record with low TTL for fast failover
aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch '{
  "Changes": [{
    "Action": "UPSERT",
    "ResourceRecordSet": {
      "Name": "api.company.com",
      "Type": "A",
      "SetIdentifier": "primary",
      "Failover": "PRIMARY",
      "TTL": 30,  # LOW TTL for fast failover
      "ResourceRecords": [{"Value": "1.2.3.4"}],
      "HealthCheckId": "health-check-id"
    }
  }]
}'
```

**Tricky**: Route 53 itself is globally distributed and highly available. But the health checks run from specific locations. If all health check locations are in the region that's having issues, you get a blind spot. Always configure health checks from at least 3 diverse regions. Also, remember that DNS caching is client-side — even after Route 53 switches the record, clients with cached DNS won't see the change until their TTL expires. Java applications are notorious for caching DNS forever (JVM's `networkaddress.cache.ttl` defaults to -1 in some versions = cache forever).

---


## Network Troubleshooting War Stories

---

**Q9: Intermittent connection timeouts between microservices in the same VPC. Happens randomly, maybe 0.1% of requests. All security groups are correct. How do you find the cause?**

**A:**

**The 0.1% mystery — common causes:**
```
1. CONNTRACK TABLE FULL (Linux kernel connection tracking)
   - Each TCP connection uses a conntrack entry
   - Default limit: 65,536 entries per node
   - High-traffic pods can exhaust this → new connections silently dropped
   
2. ARP TABLE OVERFLOW
   - Kubernetes nodes with many pods hit ARP cache limits
   - Default gc_thresh3: 1024 entries
   - More pods = more IPs = more ARP entries needed

3. NETWORK INTERFACE QUEUE OVERFLOW
   - ENI packet-per-second limit: varies by instance type
   - c5.large: 750,000 PPS max
   - Exceeded → packets dropped silently (no error returned)

4. TCP TIME_WAIT EXHAUSTION
   - Short-lived connections pile up in TIME_WAIT
   - Ephemeral port range: 32768-60999 (28,231 ports)
   - If all ports in TIME_WAIT → new connections fail

5. DNS RESOLUTION TIMEOUT
   - VPC DNS resolver has rate limit: 1024 packets/second per ENI
   - High pod count with frequent DNS queries → dropped DNS packets
   - Results in 5-second delay (DNS retry timeout)
```

**Diagnosis commands:**
```bash
# Check conntrack table
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
# If count ≈ max → FULL! Connections being dropped

# Check for packet drops
netstat -s | grep -i "drop\|overflow\|pruned"
cat /proc/net/softnet_stat  # Column 2 = dropped packets

# Check TIME_WAIT connections
ss -s  # Shows TCP state summary
ss -tan state time-wait | wc -l

# Check ARP table
ip neigh show | wc -l
cat /proc/sys/net/ipv4/neigh/default/gc_thresh3

# Check ENI packet stats (instance-level)
ethtool -S eth0 | grep -i "drop\|err\|miss"

# VPC Flow Logs for dropped packets
# Filter: action = REJECT (security group/NACL)
# If packets are dropped but flow logs show ACCEPT → it's instance/kernel level
```

**Fixes:**
```bash
# Fix conntrack exhaustion
sysctl -w net.netfilter.nf_conntrack_max=524288
echo "net.netfilter.nf_conntrack_max=524288" >> /etc/sysctl.conf

# Fix ARP table overflow
sysctl -w net.ipv4.neigh.default.gc_thresh1=4096
sysctl -w net.ipv4.neigh.default.gc_thresh2=8192
sysctl -w net.ipv4.neigh.default.gc_thresh3=16384

# Fix TIME_WAIT exhaustion
sysctl -w net.ipv4.tcp_tw_reuse=1
# NOTE: tcp_tw_recycle is REMOVED in Linux 4.12+ (caused issues with NAT)

# Fix DNS rate limiting
# Use NodeLocal DNSCache in Kubernetes
kubectl apply -f https://k8s.io/examples/admin/dns/nodelocaldns.yaml
# This caches DNS on each node, reducing queries to VPC resolver

# For EKS: Use larger instance types with higher PPS limits
# c5.xlarge: 1,000,000 PPS vs c5.large: 750,000 PPS
```

**Tricky**: VPC Flow Logs only show ACCEPT or REJECT at the security group/NACL level. If a packet is dropped by the KERNEL (conntrack full, buffer overflow, PPS limit), flow logs still show ACCEPT because it passed AWS networking — it was dropped after reaching the instance. This is why 0.1% packet loss with "correct security groups" is so hard to debug. You need instance-level metrics (Enhanced Monitoring, CloudWatch Agent, or node_exporter).

---

**Q10: Your application connects to an on-premises database via VPN. Performance was fine for months, but now you're seeing 200ms latency (was 20ms). The VPN tunnel is UP. What's happening?**

**A:**

**Systematic VPN performance investigation:**
```bash
# Step 1: Is it the VPN tunnel or the network behind it?
# Ping the VPN tunnel endpoint (should be <5ms for same-region)
ping 169.254.x.x  # Tunnel inside address

# Ping the actual on-prem database
ping 192.168.1.100  # On-prem DB IP

# If tunnel latency is high → VPN issue
# If tunnel is fine but DB is slow → on-prem network issue

# Step 2: Check VPN tunnel status and metrics
aws ec2 describe-vpn-connections --vpn-connection-ids vpn-xxx \
  --query 'VpnConnections[].VgwTelemetry[]'
# Check: StatusMessage, OutsideIpAddress, AcceptedRouteCount

# Step 3: Check CloudWatch VPN metrics
# TunnelDataIn/Out, TunnelState
aws cloudwatch get-metric-statistics --namespace AWS/VPN \
  --metric-name TunnelDataIn --period 300 --statistics Sum \
  --dimensions Name=VpnId,Value=vpn-xxx
```

**Common causes:**
```
1. VPN FAILOVER TO BACKUP TUNNEL
   - AWS VPN has 2 tunnels for HA
   - Primary tunnel goes down (maintenance) → failover to second tunnel
   - Second tunnel terminates in different AZ (or different AWS endpoint)
   - Extra hop adds latency
   - Check: Which tunnel is active?

2. BGP ROUTE CHANGE
   - On-premises router started advertising a less-optimal route
   - Traffic now goes: VPC → TGW → VPN → longer path → on-prem
   - vs before: VPC → VPN → direct path → on-prem
   
3. VPN THROUGHPUT LIMIT HIT
   - Each VPN tunnel: ~1.25 Gbps max
   - If throughput exceeded → packets queue → latency increases
   - Check: TunnelDataIn > 1 Gbps?
   - Fix: Use multiple VPN connections or Direct Connect

4. MTU / FRAGMENTATION
   - VPN adds overhead (ESP header): reduces effective MTU
   - Standard MTU: 1500, VPN effective MTU: ~1400
   - If app sends 1500-byte packets → fragmentation → reassembly delay
   - Fix: Set MSS clamping: iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1360

5. ISP ROUTING CHANGE
   - Your VPN goes over public internet
   - ISP changed peering/routing → longer path
   - traceroute shows extra hops
   - Fix: Move to AWS Direct Connect (private, consistent path)
```

**Direct Connect vs VPN comparison (production decision):**
| Factor | VPN | Direct Connect |
|--------|-----|----------------|
| Latency | Variable (internet) | Consistent (private fiber) |
| Bandwidth | 1.25 Gbps/tunnel | 1/10/100 Gbps |
| Setup time | Minutes | 2-4 weeks (physical cross-connect) |
| Cost | ~$36/month | $0.30/GB + port fee ($220-$10,800/month) |
| Encryption | Built-in IPSec | No encryption (add MACsec or VPN over DX) |
| Reliability | Internet-dependent | 99.99% SLA (with redundant connections) |

**Tricky**: AWS performs VPN tunnel endpoint maintenance WITHOUT notification. During maintenance, traffic fails over to the second tunnel. If your on-premises router doesn't support ACTIVE/ACTIVE VPN (most don't — they use ACTIVE/PASSIVE), there's a 10-30 second blackout during failover. For zero-downtime, configure both tunnels as ACTIVE on both sides with ECMP (Equal Cost Multi-Path routing). Or better: use Direct Connect with VPN as backup.

---


## Security Groups vs NACLs — Production Gotchas

---

**Q11: A new developer added a NACL rule to block a specific IP address that was scanning your infrastructure. Now half your application is broken. The security group rules haven't changed. What happened?**

**A:**

**The NACL gotcha that catches everyone:**
```
Security Groups: STATEFUL
- If you allow inbound traffic, return traffic is automatically allowed
- You only need to worry about ONE direction

NACLs: STATELESS
- You must explicitly allow BOTH inbound AND outbound
- Including the EPHEMERAL PORT RANGE for return traffic!
```

**What the developer did:**
```
NACL Inbound Rules:
100: ALLOW TCP 443 from 0.0.0.0/0  ← HTTPS in
150: DENY ALL from 1.2.3.4/32      ← Block bad IP (NEW RULE)
200: ALLOW ALL from 0.0.0.0/0      ← Allow everything else

NACL Outbound Rules:
100: ALLOW TCP 443 to 0.0.0.0/0    ← HTTPS out
200: ALLOW ALL to 0.0.0.0/0        ← Allow everything else
                                     ← BUT WAIT...
```

**The problem:**
```
The developer added an outbound rule too:
50: DENY ALL to 1.2.3.4/32   ← Block bad IP outbound

But legitimate RESPONSE traffic uses ephemeral ports (1024-65535)
If a client from IP range near 1.2.3.4 sends a request, the 
response goes OUT to that IP = BLOCKED by the DENY rule!

OR more commonly:
The developer blocked ALL traffic from the IP using rule number 50 (lower = evaluated first)
Rule 50 evaluates BEFORE rule 100
ALL traffic from network near that IP = blocked (including legitimate responses)
```

**NACL rule evaluation:**
```
Rules evaluated LOWEST number first:
50: DENY 1.2.3.4/32      ← Evaluated FIRST (blocks everything from this IP)
100: ALLOW TCP 443        ← Never reached for 1.2.3.4
200: ALLOW ALL            ← Never reached for 1.2.3.4
*: DENY ALL (default)     ← Implicit deny

If the "bad IP" was actually a CIDR range like 1.2.3.0/24...
you just blocked 254 legitimate users!
```

**Proper way to block IPs in production:**
```bash
# Option 1: AWS WAF (best for web traffic)
aws wafv2 create-ip-set --name "blocked-scanners" --scope REGIONAL \
  --ip-address-version IPV4 --addresses "1.2.3.4/32" "5.6.7.8/32"

# Option 2: Security Group (simpler, but can't DENY)
# SGs only ALLOW — you control access by NOT allowing
# Works for: restrict access to specific IPs only

# Option 3: NACL (if you must) — do it correctly:
# Block ONLY inbound from the specific IP
# Rule number between existing ALLOW rules
aws ec2 create-network-acl-entry --network-acl-id acl-xxx \
  --rule-number 50 --protocol -1 --rule-action deny \
  --cidr-block 1.2.3.4/32 --ingress
# Do NOT add outbound deny (breaks responses to nearby IPs)
```

**Tricky**: NACLs apply to the SUBNET, not to individual instances. If you add a NACL rule, it affects ALL resources in that subnet — every EC2 instance, Lambda ENI, RDS instance, etc. Security Groups apply per-ENI (per instance). For blocking specific IPs, WAF is always the right choice at the application layer. NACLs should only be used for broad network-level controls (block entire country CIDRs, block known-bad networks).

---

**Q12: Your security team runs a scan and finds that your RDS database is "publicly accessible." But it's in a private subnet with no internet gateway route. Is it actually exposed? How do you verify?**

**A:**

**Understanding "publicly accessible" in RDS:**
```
RDS "Publicly Accessible = Yes" means:
- RDS instance gets a PUBLIC DNS name that resolves to a public IP
- BUT this doesn't mean it's reachable from the internet!

For actual internet access, you need ALL of these:
1. PubliclyAccessible = Yes (public DNS/IP assigned) ✓ (configured)
2. Subnet has route to Internet Gateway ✗ (private subnet — NO route)
3. Security Group allows inbound from 0.0.0.0/0 ✗ (restricted to app SG)
4. NACL allows inbound traffic ✓ (probably)

Missing #2 means: NOT actually reachable from internet
The "Publicly Accessible" flag is MISLEADING
```

**Verification:**
```bash
# Check what DNS resolves to
nslookup prod-db.xxx.us-east-1.rds.amazonaws.com
# If public IP but can't connect → network path is blocked

# Check from outside AWS
timeout 3 nc -zv prod-db.xxx.us-east-1.rds.amazonaws.com 5432
# Should timeout if truly unreachable

# Check subnet route table
aws ec2 describe-route-tables --filters \
  "Name=association.subnet-id,Values=$(aws rds describe-db-instances \
  --db-instance-identifier prod-db \
  --query 'DBInstances[0].DBSubnetGroup.Subnets[0].SubnetIdentifier' --output text)" \
  --query 'RouteTables[].Routes[]'
# Look for 0.0.0.0/0 → igw-xxx (if missing = not internet accessible)

# Check security group
aws ec2 describe-security-groups --group-ids sg-rds-xxx \
  --query 'SecurityGroups[].IpPermissions[]'
# Should NOT have 0.0.0.0/0 in any source
```

**Fix (defense in depth):**
```bash
# Set PubliclyAccessible to false (modifies instance — no downtime for Multi-AZ)
aws rds modify-db-instance --db-instance-identifier prod-db \
  --no-publicly-accessible --apply-immediately
# This removes the public IP and changes DNS to resolve to private IP only

# Verify
aws rds describe-db-instances --db-instance-identifier prod-db \
  --query 'DBInstances[0].PubliclyAccessible'
# Should return: false
```

**Tricky**: Setting `PubliclyAccessible=false` changes the DNS resolution. If your application uses the RDS endpoint DNS name (which it should), the DNS will now resolve to a PRIVATE IP instead of public. If your application is OUTSIDE the VPC (e.g., in another VPC without peering, or on-premises without VPN), it will stop working. Always verify application connectivity after changing this setting. Also, for Multi-AZ deployments, changing PubliclyAccessible triggers a failover (brief downtime).

---
