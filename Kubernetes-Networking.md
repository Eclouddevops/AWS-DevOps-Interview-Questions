# Kubernetes Networking — Deep-Dive Interview Q&A

## Table of Contents
1. [Networking Fundamentals](#networking-fundamentals)
2. [Services & Service Discovery](#services--service-discovery)
3. [Ingress & Load Balancing](#ingress--load-balancing)
4. [CNI Plugins](#cni-plugins)
5. [DNS in Kubernetes](#dns-in-kubernetes)
6. [Network Policies](#network-policies)
7. [Service Mesh](#service-mesh)
8. [Troubleshooting Scenarios](#troubleshooting-scenarios)

---

## Networking Fundamentals

**Q1: Explain the 4 fundamental Kubernetes networking requirements.**

**A:**

1. **Pod-to-Pod**: Every pod can communicate with every other pod without NAT (flat network)
2. **Pod-to-Service**: Pods access services via ClusterIP (virtual IP)
3. **External-to-Service**: External traffic reaches pods via NodePort/LoadBalancer/Ingress
4. **Pod-to-External**: Pods can reach external services (SNAT via node IP)

**Key constraint**: Kubernetes assumes a flat network — every pod gets a unique IP routable within the cluster. No NAT between pods.

---

**Q2: How does pod-to-pod communication work across nodes? Explain the packet flow.**

**A:**

```
Pod A (Node 1: 10.244.1.5) → Pod B (Node 2: 10.244.2.8)

1. Pod A sends packet: src=10.244.1.5, dst=10.244.2.8
2. Packet hits pod's veth pair → goes to node's network bridge (cbr0/cni0)
3. Node 1 routing table: 10.244.2.0/24 → via Node 2 IP
4. Encapsulation (depends on CNI):
   - VXLAN: Original packet wrapped in UDP (port 4789)
   - IP-in-IP: Original packet wrapped in outer IP header
   - BGP (no encap): Routes advertised via BGP to physical network
5. Packet arrives at Node 2
6. Decapsulation → routing to local bridge → veth pair → Pod B
```

**CNI comparison for cross-node:**
| CNI | Method | Overhead | Performance |
|-----|--------|----------|-------------|
| Calico (default) | IP-in-IP | Low (~20 bytes) | Good |
| Calico (no encap) | BGP routes | Zero | Best |
| Flannel (VXLAN) | VXLAN | Medium (~50 bytes) | Moderate |
| Cilium | VXLAN/Geneve/Native | Configurable | Best (eBPF) |
| AWS VPC CNI | Native VPC routing | Zero | Best (AWS) |

---

**Q3: What is the difference between VXLAN, IP-in-IP, and native routing in Kubernetes networking?**

**A:**

**VXLAN (Virtual Extensible LAN):**
- Encapsulates L2 frame in UDP packet
- ~50 bytes overhead per packet
- Works across any network (even non-routable)
- Higher CPU usage for encap/decap
- Used by: Flannel, Calico (cross-subnet), Cilium

**IP-in-IP:**
- Encapsulates IP packet in another IP packet
- ~20 bytes overhead
- Requires IP protocol 4 support (some cloud providers block it)
- Used by: Calico (default mode)

**Native/Direct Routing:**
- No encapsulation — pod CIDR routes advertised to physical network
- Zero overhead — best performance
- Requires L3 network infrastructure support (BGP peering or cloud VPC routes)
- Used by: Calico (BGP mode), AWS VPC CNI, GKE native routing

**Tricky**: On AWS, IP-in-IP doesn't work between VPCs (blocked by security groups). Must use VXLAN or native VPC CNI.

---

## Services & Service Discovery

**Q4: Explain how ClusterIP service works internally. What happens when you curl a ClusterIP?**

**A:**

```
Pod → ClusterIP (10.96.0.10:80) → [kube-proxy/eBPF] → Pod endpoint (10.244.1.5:8080)

Step-by-step:
1. Pod sends packet to ClusterIP 10.96.0.10:80
2. Packet hits iptables/IPVS/eBPF rules on the NODE
3. DNAT: dst changes from 10.96.0.10:80 → 10.244.1.5:8080 (random backend pod)
4. Packet routed to backend pod (normal pod-to-pod routing)
5. Response: SNAT reverses the translation
```

**kube-proxy modes:**
| Mode | Mechanism | Performance | Features |
|------|-----------|-------------|----------|
| iptables (default) | iptables rules | O(n) rules, degrades at scale | Random selection |
| IPVS | Linux IPVS | O(1) lookup, hash table | Round-robin, least-conn, etc. |
| eBPF (Cilium) | Kernel eBPF | Fastest, no iptables | Full load balancing |

**Tricky**: ClusterIP is a VIRTUAL IP — it doesn't exist on any interface. It's only a target for iptables DNAT rules. You can't ping it (no ICMP rule), only connect to the declared port.

---

**Q5: What's the difference between ClusterIP, NodePort, LoadBalancer, and ExternalName services?**

**A:**

```
ClusterIP (default):
├── Virtual IP accessible only within cluster
├── Port range: any
└── Use: Internal service communication

NodePort:
├── Extends ClusterIP
├── Opens a port (30000-32767) on EVERY node
├── External access via <NodeIP>:<NodePort>
└── Use: Development, on-prem without cloud LB

LoadBalancer:
├── Extends NodePort
├── Provisions cloud load balancer (AWS ELB/NLB, GCP LB)
├── External IP assigned by cloud provider
└── Use: Production external access

ExternalName:
├── DNS CNAME record only (no proxy/IP)
├── Maps service name to external DNS name
├── No ClusterIP, no port
└── Use: Abstract external services (e.g., RDS endpoint)
```

**Tricky questions:**
- **Q: Can you have a LoadBalancer service without NodePort?** A: Yes! Set `spec.allocateLoadBalancerNodePorts: false` (K8s 1.24+). Traffic goes directly to pod IPs (requires cloud support).
- **Q: What happens if you delete a LoadBalancer service?** A: The cloud LB is ALSO deleted (controller manages lifecycle).
- **Q: Can NodePort conflict across services?** A: Yes! Each NodePort must be unique cluster-wide.

---

**Q6: Explain headless services. When and why would you use them?**

**A:**

A headless service has `clusterIP: None`. Instead of a virtual IP, DNS returns the individual pod IPs.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  clusterIP: None  # Headless!
  selector:
    app: postgres
  ports:
  - port: 5432
```

**DNS behavior:**
```
# Normal service: returns ClusterIP
dig my-service.default.svc.cluster.local
→ 10.96.0.10 (single VIP)

# Headless service: returns all pod IPs
dig my-db.default.svc.cluster.local
→ 10.244.1.5
→ 10.244.2.8
→ 10.244.3.12
```

**Use cases:**
1. **StatefulSets**: Each pod needs a stable DNS name (`pod-0.my-db.default.svc.cluster.local`)
2. **Client-side load balancing**: Application chooses which backend to connect to
3. **Service discovery**: Get all endpoints for custom routing logic
4. **Databases**: Connect to specific replicas (read from secondaries)

---

**Q7: What is the `externalTrafficPolicy` and why does it matter?**

**A:**

| Policy | Behavior | Pros | Cons |
|--------|----------|------|------|
| `Cluster` (default) | Traffic distributed across ALL nodes, then to any pod | Even distribution | Extra hop, source IP lost (SNAT) |
| `Local` | Traffic only goes to pods on the receiving node | Preserves source IP, no extra hop | Uneven distribution, connection refused if no local pod |

**Critical for:**
- **Source IP preservation**: WAF, geo-routing, audit logs need real client IP
- **Performance**: Avoids extra network hop between nodes
- **Health checks**: LoadBalancer only sends traffic to nodes with pods

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # Preserves client IP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: web
```

**Tricky**: With `Local` policy + insufficient pod distribution, some nodes will return 503 (no local endpoints). Must ensure pods are spread across all nodes (use topology spread constraints or pod anti-affinity).

---

## Ingress & Load Balancing

**Q8: Explain the difference between Ingress, Gateway API, and Service Mesh ingress.**

**A:**

| Feature | Ingress (legacy) | Gateway API (modern) | Service Mesh (Istio/Linkerd) |
|---------|-----------------|---------------------|------------------------------|
| API maturity | GA (stable) | GA (v1.0+) | Vendor-specific |
| Multi-tenancy | Poor (one controller) | Native (GatewayClass) | Full (VirtualService) |
| Protocol support | HTTP/HTTPS only | HTTP, gRPC, TCP, UDP | All + mTLS |
| Traffic splitting | Limited (annotations) | Native (HTTPRoute weights) | Full (canary, mirror) |
| Header-based routing | Annotation-dependent | Native | Full |
| Cross-namespace | No | Yes (ReferenceGrant) | Yes |
| TLS termination | Yes | Yes + passthrough | Yes + mTLS |

**Gateway API architecture:**
```
GatewayClass (infra provider defines)
    └── Gateway (cluster operator creates)
        └── HTTPRoute/TCPRoute (app developer creates)
            └── Backend Services
```

**Tricky**: Ingress annotations are controller-specific and NOT portable. Moving from nginx to traefik means rewriting all annotations. Gateway API solves this with standardized spec.

---

**Q9: You have 5 microservices behind an Ingress. One service is getting 10x more traffic. How do you handle this?**

**A:**

**Immediate fixes:**
1. **HPA on the hot service**: Auto-scale pods based on requests/CPU
2. **Ingress rate limiting** (nginx annotation):
```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
```

3. **Separate Ingress Controller** for the hot service:
```yaml
# Dedicated ingress class for high-traffic service
spec:
  ingressClassName: nginx-high-traffic
```

4. **CDN/Caching**: If traffic is cacheable, put CloudFront in front

**Architecture consideration:**
- Don't share ingress controllers for services with vastly different traffic patterns
- One noisy service can exhaust connections for all others
- Use separate IngressClass or dedicated node pools for high-traffic services

---

## CNI Plugins

**Q10: Compare Calico, Cilium, and AWS VPC CNI. When would you choose each?**

**A:**

| Feature | Calico | Cilium | AWS VPC CNI |
|---------|--------|--------|-------------|
| Dataplane | iptables/eBPF | eBPF (native) | AWS ENI/VPC |
| Network Policy | Full | Full + extended (L7, DNS) | Basic (with Calico addon) |
| Encryption | WireGuard | WireGuard/IPsec | VPC encryption |
| Observability | Flow logs | Hubble (full visibility) | VPC Flow Logs |
| Performance | Good | Best | Best (native VPC) |
| Multi-cluster | Yes (Typha) | Yes (ClusterMesh) | Limited |
| IP management | Overlay (CIDR) | Overlay or native | VPC IP per pod |

**Choose Calico when:**
- On-premises or multi-cloud
- Need BGP peering with physical network
- Familiar with iptables/networking

**Choose Cilium when:**
- Need L7 visibility without service mesh
- Want eBPF performance
- Need advanced network policies (DNS-based, HTTP-aware)
- Observability is priority (Hubble)

**Choose AWS VPC CNI when:**
- Running on EKS
- Need pod-level security groups
- Need direct VPC integration (pods get VPC IPs)
- Tradeoff: Limited IPs per node (based on ENI limits)

---

**Q11: AWS VPC CNI has an IP exhaustion problem. Explain it and how to solve it.**

**A:**

**Problem**: Each pod gets a real VPC IP address. IPs are pre-allocated per node based on instance type's ENI capacity.

```
Instance Type → Max ENIs → Max IPs per ENI → Max Pods
t3.medium    → 3 ENIs   → 6 IPs/ENI      → 17 pods max
m5.large     → 3 ENIs   → 10 IPs/ENI     → 29 pods max
m5.xlarge    → 4 ENIs   → 15 IPs/ENI     → 58 pods max
```

**Solutions:**

1. **Prefix Delegation** (recommended):
```bash
# Enable /28 prefix assignment (16 IPs per prefix slot)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
# m5.large: 3 ENIs × 10 slots × 16 IPs = 480 pods max!
```

2. **Secondary CIDR ranges**: Add more CIDR to VPC (100.64.0.0/16 for pods)

3. **Custom networking**: Pods use different subnet than nodes:
```bash
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

4. **Larger instance types**: More ENIs = more IPs

**Tricky**: If you run out of IPs, new pods stay in `ContainerCreating` with error: `failed to assign an IP address to container`. Existing pods are fine but no new ones can schedule.

---

## DNS in Kubernetes

**Q12: Explain Kubernetes DNS resolution. What happens when a pod resolves `my-service`?**

**A:**

```
Pod resolves "my-service"
    ↓
/etc/resolv.conf in pod:
  nameserver 10.96.0.10 (CoreDNS ClusterIP)
  search default.svc.cluster.local svc.cluster.local cluster.local
    ↓
DNS queries (in order):
  1. my-service.default.svc.cluster.local → Found? Return ClusterIP
  2. my-service.svc.cluster.local → Found? Return
  3. my-service.cluster.local → Found? Return
  4. my-service → External DNS resolution
```

**DNS record formats:**
```
Services:  <service>.<namespace>.svc.cluster.local → ClusterIP
Pods:      <pod-ip-dashed>.<namespace>.pod.cluster.local → Pod IP
StatefulSet: <pod-name>.<service>.<namespace>.svc.cluster.local → Pod IP
```

**Tricky**: The search domains mean a simple `curl my-service` triggers 4-5 DNS queries! For external domains, use FQDN with trailing dot: `curl google.com.` (avoids unnecessary search domain queries).

---

**Q13: CoreDNS is returning NXDOMAIN for a service that exists. How do you troubleshoot?**

**A:**

```bash
# Step 1: Verify service exists
kubectl get svc my-service -n target-ns

# Step 2: Check endpoints (service might have no backends)
kubectl get endpoints my-service -n target-ns
# Empty endpoints = DNS exists but no IPs returned (not NXDOMAIN though)

# Step 3: Test DNS from within a pod
kubectl run dnstest --image=busybox --restart=Never -- sleep 3600
kubectl exec dnstest -- nslookup my-service.target-ns.svc.cluster.local

# Step 4: Check CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Step 5: Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Step 6: Verify CoreDNS ConfigMap
kubectl get cm coredns -n kube-system -o yaml

# Step 7: Check if pod's DNS policy is correct
kubectl get pod my-pod -o yaml | grep dnsPolicy
```

**Common causes:**
1. Pod using `dnsPolicy: Default` (uses node's DNS, not cluster DNS)
2. CoreDNS crashed/OOMKilled (check resources)
3. NetworkPolicy blocking DNS traffic (port 53 UDP/TCP to kube-dns)
4. Service in different namespace (need FQDN: `svc.other-ns.svc.cluster.local`)
5. CoreDNS ConfigMap misconfigured

---

**Q14: How do you optimize DNS performance in a large Kubernetes cluster?**

**A:**

1. **NodeLocal DNSCache** (DaemonSet):
```
Pod → Node-local cache (169.254.20.10) → CoreDNS (only on cache miss)
```
Benefits: Reduces inter-node traffic, faster resolution, connection tracking improvements

2. **Increase CoreDNS replicas**:
```yaml
# Scale based on cluster size
# Rule of thumb: 1 CoreDNS replica per 500 pods
kubectl scale deployment coredns -n kube-system --replicas=5
```

3. **Use FQDN with trailing dot**: Eliminates search domain lookups
```
# Bad: "external-service" → 5 DNS queries
# Good: "external-service.example.com." → 1 DNS query
```

4. **Tune ndots in pods**:
```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"  # Default is 5 (too many search expansions)
```

5. **Autopath plugin** in CoreDNS: Responds with FQDN on first try

---

## Troubleshooting Scenarios

**Q15: Pod A can reach Pod B, but Pod B cannot reach Pod A. What could cause asymmetric connectivity?**

**A:**

1. **NetworkPolicy**: Ingress policy on Pod A blocks Pod B, but no egress policy on Pod B
```yaml
# Pod A has this - blocks incoming from Pod B
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-pod-a
spec:
  podSelector:
    matchLabels:
      app: pod-a
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: allowed-app  # Pod B not in this list!
```

2. **iptables/IPVS asymmetry**: Corrupted conntrack entries (rare)
3. **Node-level firewall**: One node has iptables rules blocking return traffic
4. **MTU mismatch**: One direction works (small packets), other fails (large packets due to PMTU issues)
5. **CNI bug**: Pod on one node has broken networking

**Debugging:**
```bash
# Test connectivity from both sides
kubectl exec pod-a -- curl pod-b-ip:port
kubectl exec pod-b -- curl pod-a-ip:port

# Check network policies affecting each pod
kubectl get networkpolicy -n namespace

# Check conntrack entries
conntrack -L | grep <pod-ip>

# Packet capture on node
tcpdump -i any host <pod-ip> -nn
```

---

**Q16: Services work within the cluster but external clients get intermittent timeouts via LoadBalancer. What's happening?**

**A:**

Common causes of intermittent external timeouts:

1. **Connection draining during deployments**: Pod gets killed before finishing request
   - Fix: `terminationGracePeriodSeconds` + preStop hook + readiness probe removal

2. **SNAT port exhaustion** (with `externalTrafficPolicy: Cluster`):
   - Node runs out of SNAT ports for return traffic
   - Fix: Switch to `externalTrafficPolicy: Local` or increase conntrack table

3. **Health check failures**: LB marks nodes unhealthy intermittently
   - Check LB health check path and timeout settings

4. **TCP idle timeout mismatch**:
   ```
   Client keep-alive: 300s
   LB idle timeout: 60s (AWS ALB default)
   → LB silently closes connection after 60s
   → Client sends on dead connection → timeout
   ```
   Fix: Set application keep-alive < LB idle timeout

5. **Insufficient pod resources**: OOMKill or CPU throttling under load
   - Check `kubectl top pods` and resource limits

**Quick diagnostics:**
```bash
# Check node health from LB perspective
kubectl get nodes -o wide  # All Ready?

# Check endpoints
kubectl get endpoints my-service  # All pods listed?

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20

# Check for pod restarts
kubectl get pods -o wide | grep -v "1/1"
```

---

**Q17: Explain the difference between `kubectl port-forward` and `kubectl proxy`. What are the security implications?**

**A:**

| Feature | port-forward | proxy |
|---------|-------------|-------|
| Target | Specific pod or service | API server |
| Protocol | TCP (raw) | HTTP (API proxy) |
| Auth | Uses your kubeconfig | Uses your kubeconfig |
| Access | Direct to pod port | Access to any API resource |
| Use case | Debug a specific pod | Dashboard access, API exploration |

**Security implications:**
- `port-forward`: Opens direct tunnel to pod. If pod is compromised, attacker has reverse shell via your tunnel.
- `proxy`: Exposes ENTIRE Kubernetes API on localhost. Anyone with access to your machine can use it.

```bash
# port-forward: Only specific pod/port
kubectl port-forward pod/my-pod 8080:80
# Only localhost:8080 → pod:80

# proxy: Full API access
kubectl proxy --port=8001
# localhost:8001/api/v1/namespaces/default/secrets → ALL SECRETS!
```

**Best practice**: Never run `kubectl proxy` on shared machines or with `--address=0.0.0.0`

---

**Q18: What happens to in-flight requests when a pod is terminated? How do you achieve zero-downtime deployments?**

**A:**

**Default termination sequence:**
```
1. Pod marked for deletion (metadata.deletionTimestamp set)
2. Pod removed from Service endpoints (simultaneously with step 3!)
3. preStop hook runs (if defined)
4. SIGTERM sent to container
5. terminationGracePeriodSeconds countdown (default: 30s)
6. SIGKILL sent (forced kill)
```

**The race condition problem:**
- Step 2 (endpoint removal) and Step 4 (SIGTERM) happen IN PARALLEL
- kube-proxy/iptables update is NOT instant (propagation delay)
- Requests can still arrive at the pod AFTER it receives SIGTERM

**Solution - Zero-downtime deployment:**
```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]  # Wait for endpoint removal propagation
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      # Once readiness fails, pod is removed from endpoints
```

**Application side:**
- Handle SIGTERM: Stop accepting NEW connections, finish IN-FLIGHT requests
- Return non-ready on readiness probe during shutdown
- Connection draining timeout < terminationGracePeriodSeconds

---

**Q19: How does kube-proxy IPVS mode differ from iptables mode? When does iptables become a problem?**

**A:**

**iptables mode problems at scale:**
- O(n) rules per service: 10,000 services = 40,000+ iptables rules
- Rule update locks the iptables table (all traffic blocked during update!)
- Latency increases linearly with number of services
- 20,000+ rules: Service updates can take seconds (causes packet drops)

**IPVS mode advantages:**
- Hash table lookup: O(1) regardless of service count
- Supports multiple load balancing algorithms (rr, lc, dh, sh, sed, nq)
- Incremental updates (no full-table lock)
- Better performance at scale (100K+ services)

**Switch to IPVS:**
```yaml
# kube-proxy ConfigMap
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"  # round-robin
```

**Tricky**: IPVS still uses iptables for SNAT and masquerading. It's not "zero iptables" — it just offloads the service routing rules to IPVS.

---

**Q20: A pod can reach the internet but cannot resolve external DNS names. Internal cluster DNS works fine. What's wrong?**

**A:**

This means CoreDNS is working for cluster-internal names but failing to forward external queries.

**Check CoreDNS Corefile:**
```
kubectl get cm coredns -n kube-system -o yaml
```

**Common causes:**
1. **Upstream DNS unreachable**: CoreDNS forwards to node's `/etc/resolv.conf` which may point to unreachable DNS
```
# In Corefile
forward . /etc/resolv.conf  # Node's resolv.conf may have wrong DNS
# Fix: forward . 8.8.8.8 8.8.4.4
```

2. **NetworkPolicy blocking egress DNS**: Pod can reach CoreDNS (internal) but CoreDNS can't reach upstream (external)
```yaml
# Need to allow CoreDNS egress to external DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-coredns-egress
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      k8s-app: kube-dns
  egress:
  - to: []  # Allow all egress for CoreDNS
    ports:
    - port: 53
      protocol: UDP
```

3. **VPC DNS throttling** (AWS): VPC DNS resolver has rate limits. High query volume → dropped packets.

4. **Wrong dnsPolicy on pod**: `dnsPolicy: None` without custom `dnsConfig`

**Quick fix test:**
```bash
kubectl exec test-pod -- nslookup google.com 8.8.8.8
# If this works, problem is CoreDNS upstream forwarding
```



---

## Additional Scenario-Based Tricky Questions

---

**Q21: Your pods can reach external services (internet) but cannot communicate with pods in another namespace. NetworkPolicies exist in both namespaces. How do you debug cross-namespace communication?**

**A:**

**Debugging steps:**
```bash
# Step 1: Check NetworkPolicies in BOTH namespaces
kubectl get networkpolicy -n namespace-a
kubectl get networkpolicy -n namespace-b

# Step 2: The crucial detail — NetworkPolicies are ADDITIVE for allow but DEFAULT DENY
# If namespace-b has ANY NetworkPolicy with podSelector matching the target pod,
# then ALL ingress not explicitly allowed is DENIED

# Check if namespace-b allows ingress from namespace-a:
kubectl get networkpolicy -n namespace-b -o yaml | grep -A 20 "ingress"

# Step 3: Test connectivity
kubectl exec -n namespace-a test-pod -- curl -v http://service-b.namespace-b.svc.cluster.local:8080
```

**Common cross-namespace NetworkPolicy mistakes:**
```yaml
# WRONG: This only allows traffic from SAME namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
  namespace: namespace-b
spec:
  podSelector:
    matchLabels:
      app: service-b
  ingress:
  - from:
    - podSelector:          # No namespaceSelector = same namespace only!
        matchLabels:
          app: service-a

# CORRECT: Allow cross-namespace traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cross-namespace
  namespace: namespace-b
spec:
  podSelector:
    matchLabels:
      app: service-b
  ingress:
  - from:
    - namespaceSelector:      # Must specify namespace!
        matchLabels:
          kubernetes.io/metadata.name: namespace-a
      podSelector:
        matchLabels:
          app: service-a
```

**Tricky**: In NetworkPolicy, `podSelector` without `namespaceSelector` means "pods in THIS namespace only." To allow cross-namespace, you MUST include `namespaceSelector`. Also, the `-` (dash) before `namespaceSelector` matters for AND vs OR logic. With dash: each item is OR'd. Without dash (same list item): AND'd together. This subtle YAML difference is the #1 cause of broken NetworkPolicies.

---

**Q22: After enabling Calico as your CNI, some pods can reach the internet but others cannot. All pods are in the same subnet. The ones that fail are always on specific nodes. What's happening?**

**A:**

**The node-specific networking issue:**
```bash
# Check which nodes have failing pods
kubectl get pods -o wide | grep -v Running

# Check iptables on failing nodes
ssh node-3 "iptables -t nat -L POSTROUTING -n | grep -i masq"
# If MASQUERADE rule is missing → pods can't reach internet (no SNAT)

# Check Calico node status
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
# Is calico-node pod Running on the failing nodes?

# Check BGP peering (if using Calico BGP mode)
kubectl exec -n kube-system calico-node-xxx -- birdcl show protocols
```

**Common causes:**
```
1. calico-node DaemonSet not running on specific nodes
   - Node taint preventing scheduling
   - Node label mismatch with DaemonSet nodeSelector
   - Fix: Check DaemonSet tolerations and node selectors

2. IP-in-IP tunnel broken on specific nodes
   - MTU mismatch (node network MTU < tunnel overhead)
   - Firewall blocking protocol 4 (IPIP) between nodes
   - Fix: Use VXLAN mode instead (UDP, passes through firewalls)

3. iptables rules not applied
   - kube-proxy mode mismatch (iptables vs IPVS)
   - calico-node crashed and left stale rules
   - Fix: Restart calico-node on affected nodes

4. BGP route not propagated
   - New node joined but BGP session not established
   - Route reflector unreachable from specific nodes
   - Fix: Check BGP peer status with calicoctl
```

**Fix for IPIP/VXLAN issues:**
```yaml
# Switch from IPIP to VXLAN (works through most firewalls)
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-pool
spec:
  cidr: 192.168.0.0/16
  ipipMode: Never          # Disable IPIP
  vxlanMode: Always        # Use VXLAN instead
  natOutgoing: true        # Enable SNAT for internet access
```

**Tricky**: Calico's IPIP mode uses IP protocol 4 (not TCP/UDP). Many corporate firewalls, cloud security groups, and network ACLs only allow TCP/UDP/ICMP. IPIP packets get silently dropped → pods on nodes behind restrictive firewalls can't communicate with pods on other nodes. VXLAN uses UDP port 4789 — much more firewall-friendly. If you see partial connectivity (some nodes work, others don't), suspect protocol filtering.

---

**Q23: Your Kubernetes service is load-balancing unevenly — one pod gets 80% of traffic while other 4 pods get 5% each. The Service type is ClusterIP with no session affinity. Why?**

**A:**

**Causes of uneven load distribution:**
```
1. KEEPALIVE CONNECTIONS (most common):
   - Client maintains persistent HTTP/2 or gRPC connections
   - kube-proxy load-balances at CONNECTION creation, not per-request
   - Client creates 1 connection → all requests go to same pod
   - 5 clients, but 4 connect to pod-1 by chance = 80% traffic

2. NODE-LOCAL SERVICE ROUTING:
   - externalTrafficPolicy: Local
   - Traffic enters through one node → only sent to pods on THAT node
   - If 4 pods on node-1, 1 pod on node-2, and LB sends 80% to node-1...

3. IPTABLES PROBABILITY:
   - kube-proxy iptables uses random probability
   - With 5 pods: each rule has 1/5, 1/4, 1/3, 1/2, 1/1 probability
   - Statistically fair but for low request counts, can appear uneven

4. SLOW POD:
   - One pod responds fast (10ms) → handles more concurrent requests
   - Other pods slow (500ms) → fewer requests complete
   - Monitoring shows "more requests" to fast pod (actually better!)
```

**Fix for keepalive imbalance:**
```yaml
# Option 1: Use headless service with client-side load balancing (gRPC)
apiVersion: v1
kind: Service
metadata:
  name: my-grpc-service
spec:
  clusterIP: None  # Headless — returns all pod IPs
  selector:
    app: my-grpc-service

# gRPC client then load-balances across all returned IPs per-request

# Option 2: Use service mesh (Istio/Linkerd) for L7 load balancing
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-service-lb
spec:
  host: my-service
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN  # Per-request, not per-connection
    connectionPool:
      http:
        maxRequestsPerConnection: 100  # Force connection recycling

# Option 3: For non-mesh, add a sidecar proxy (envoy) that does L7 LB
```

**Tricky**: kube-proxy (iptables mode) does Layer 4 load balancing — it picks a backend when the TCP connection is established. For HTTP/1.1 with keepalive or HTTP/2 (multiplexed), all requests on one connection go to the SAME pod. This is by design, not a bug. To get per-request balancing, you need an L7 proxy (Ingress controller, service mesh, or sidecar). IPVS mode with `--ipvs-scheduler=lc` (least connections) is slightly better but still connection-based.

---
