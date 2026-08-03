# Kubernetes Production — Tricky Scenarios Interview Q&A

## Table of Contents
1. [Pod Scheduling & Resource Issues](#pod-scheduling--resource-issues)
2. [Networking & Service Mesh Problems](#networking--service-mesh-problems)
3. [Storage & StatefulSet Failures](#storage--statefulset-failures)
4. [Cluster Upgrades & Node Management](#cluster-upgrades--node-management)
5. [RBAC & Security Production Issues](#rbac--security-production-issues)
6. [HPA & Autoscaling Gotchas](#hpa--autoscaling-gotchas)

---

## Pod Scheduling & Resource Issues

---

**Q1: Production alert: pods in `Pending` state for 20 minutes. Cluster has 50 nodes with available capacity according to `kubectl top nodes`. Why won't pods schedule?**

**A:**

**Diagnosis:**
```bash
kubectl describe pod pending-pod-xyz | grep -A 20 "Events"
# Look for scheduler messages

# Common "capacity available but can't schedule" causes:
```

| Symptom | Cause | Fix |
|---------|-------|-----|
| "0/50 nodes: insufficient cpu" | Requests > allocatable (despite low actual usage) | Nodes are RESERVED even if not used. Reduce requests or add nodes |
| "didn't match node affinity" | nodeSelector/affinity mismatch | Check labels on nodes |
| "had taint that pod didn't tolerate" | Node taints without pod tolerations | Add toleration or remove taint |
| "didn't have free ports" | hostPort conflict | Remove hostPort usage |
| "Topology spread constraint not satisfied" | Can't satisfy even distribution | Relax topologySpreadConstraints |

**The sneaky one — resource fragmentation:**
```
Node 1: 4 CPU allocatable, 3.8 CPU requested (0.2 free)
Node 2: 4 CPU allocatable, 3.9 CPU requested (0.1 free)
...
Node 50: 4 CPU allocatable, 3.7 CPU requested (0.3 free)

Total "free" CPU: 50 × 0.2 avg = 10 CPU
Pod requesting: 1 CPU

BUT: No single node has 1 CPU free!
The capacity exists in aggregate but NOT on any single node.
This is "resource fragmentation" — the #1 hidden scheduling issue.
```

**Fix:**
```bash
# Option 1: Use Descheduler to rebalance
# Evicts pods from overloaded nodes, scheduler redistributes
kubectl apply -f https://github.com/kubernetes-sigs/descheduler/releases/latest/download/descheduler.yaml

# Option 2: Right-size resource requests
# Most teams OVER-request CPU/memory
kubectl top pods -n production --containers | \
  awk '{if($3 > 0) printf "%s: request=%s actual=%s ratio=%.0f%%\n", $2, $4, $3, ($3/$4)*100}'

# Option 3: Add cluster autoscaler / Karpenter
# Karpenter provisions RIGHT-SIZED nodes for pending pods
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
spec:
  requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: ["on-demand", "spot"]
  - key: node.kubernetes.io/instance-type
    operator: In
    values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]
  limits:
    resources:
      cpu: 1000
      memory: 1000Gi
```

**Tricky**: `kubectl top nodes` shows ACTUAL usage. The scheduler uses REQUESTS (reserved capacity), not actual usage. A node can show 20% CPU usage but be 95% reserved. This is the biggest confusion for engineers new to Kubernetes. The formula: `Allocatable - Sum(Pod Requests) = Schedulable capacity`, NOT `Allocatable - Actual Usage`.

---

**Q2: Your production cluster runs fine for weeks, then suddenly pods start getting OOMKilled across multiple services simultaneously. No deployment happened. What's causing cluster-wide OOM?**

**A:**


**Cluster-wide OOM causes (no deployment):**
```
1. MEMORY LEAK REACHING THRESHOLD (most common)
   - Multiple services have slow memory leaks
   - All reach their limits around the same time (deployed together weeks ago)
   
2. TRAFFIC SPIKE (seasonal/viral)
   - More requests = more in-flight data in memory
   - All services scale memory usage simultaneously
   
3. NODE PRESSURE
   - One large pod got OOMKilled → Kubelet starts evicting other pods
   - Eviction cascades across co-located pods
   
4. LOG/METRIC ACCUMULATION
   - In-memory log buffers growing (Fluentd sidecar backpressure)
   - Prometheus scraping causes memory growth in app metrics endpoints
   
5. KERNEL MEMORY ACCOUNTING CHANGE
   - OS update changed cgroup memory accounting
   - Previously "hidden" memory now counts toward limit
```

**Investigation:**
```bash
# Check which nodes have memory pressure
kubectl get nodes -o custom-columns='NAME:.metadata.name,MEMORY_PRESSURE:.status.conditions[?(@.type=="MemoryPressure")].status'

# Check OOM events cluster-wide
kubectl get events --all-namespaces --field-selector reason=OOMKilling --sort-by='.lastTimestamp'

# Check node-level OOM killer
kubectl debug node/worker-1 -it --image=ubuntu -- dmesg | grep -i "oom\|killed"

# Check if it's a specific node problem
kubectl get pods --all-namespaces -o wide --field-selector status.phase=Failed | grep OOMKilled

# Memory usage vs limits
kubectl top pods --all-namespaces --sort-by=memory | head -20
```

**Immediate mitigation:**
```bash
# Scale horizontally to distribute memory pressure
kubectl scale deployment --all --replicas=+2 -n production

# If specific pods are the culprits, restart them (clears leaked memory)
kubectl rollout restart deployment/leaky-service -n production

# Add memory headroom
kubectl patch deployment myservice -p '{"spec":{"template":{"spec":{"containers":[{"name":"myservice","resources":{"limits":{"memory":"1.5Gi"}}}]}}}}'
```

**Tricky**: In Kubernetes, when a NODE runs out of memory (not just a pod), the Linux OOM killer picks victims based on `oom_score_adj`. Pods with no memory limits get highest oom_score (killed first). BestEffort pods die first, then Burstable, then Guaranteed (pods where requests=limits). This is why setting memory limits CORRECTLY (not too low, not unlimited) is critical. Also, the container's memory limit includes ALL memory: heap, stack, mmap, shared libs, AND kernel memory (network buffers, filesystem cache). Your app might show "200MB heap" but the container uses 800MB total.

---

## Networking & Service Mesh Problems

---

**Q3: After upgrading CoreDNS, random pods get `NXDOMAIN` for internal service names. External DNS works fine. Only affects ~5% of queries intermittently. How do you debug?**

**A:**

```bash
# Step 1: Check CoreDNS pods are healthy
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100 | grep -i "error\|fail"

# Step 2: Test DNS resolution from affected pod
kubectl exec affected-pod -- nslookup kubernetes.default.svc.cluster.local
kubectl exec affected-pod -- nslookup my-service.my-namespace.svc.cluster.local
kubectl exec affected-pod -- cat /etc/resolv.conf

# Step 3: Check resolv.conf
# Expected:
# nameserver 10.96.0.10 (CoreDNS ClusterIP)
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# Step 4: Check CoreDNS metrics
kubectl exec -n kube-system coredns-pod -- wget -qO- http://localhost:9153/metrics | grep dns_requests_total
```

**Common causes after CoreDNS upgrade:**
```
1. ndots:5 + external lookups = 5x DNS queries
   Pod resolves "api.external.com":
   - api.external.com.default.svc.cluster.local → NXDOMAIN
   - api.external.com.svc.cluster.local → NXDOMAIN
   - api.external.com.cluster.local → NXDOMAIN
   - api.external.com.<search-domain> → NXDOMAIN
   - api.external.com. → SUCCESS (finally!)
   Under load: CoreDNS overwhelmed by 5x queries
   
2. CoreDNS pod resource limits too low after upgrade
   New version uses more memory → OOMKilled → DNS blackout
   
3. Conntrack table full on CoreDNS node
   UDP connections fill conntrack → packets dropped
   
4. CoreDNS Corefile misconfiguration
   New plugin order breaks resolution chain
```

**Fix:**
```yaml
# Increase CoreDNS resources
kubectl edit deployment coredns -n kube-system
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    memory: 512Mi

# Scale CoreDNS (often under-provisioned!)
kubectl scale deployment coredns -n kube-system --replicas=5

# Add NodeLocal DNSCache (eliminates CoreDNS as SPOF)
# Each node runs a DNS cache, pods query local cache first
kubectl apply -f nodelocaldns.yaml

# Fix ndots for external-heavy services
# In pod spec:
dnsConfig:
  options:
  - name: ndots
    value: "2"  # Reduces unnecessary cluster lookups
```

**Tricky**: The default `ndots:5` means ANY domain with fewer than 5 dots goes through ALL search domains first. `api.stripe.com` has 2 dots (< 5), so Kubernetes tries 4 internal lookups before the real one. For services making many external calls, set `ndots:2` or use FQDNs with trailing dot: `api.stripe.com.` (trailing dot = absolute, skip search domains). This alone can reduce DNS traffic by 80%.

---

**Q4: Your service mesh (Istio) is causing 200ms latency overhead on every request. Before Istio: 5ms service-to-service. After Istio: 205ms. How do you fix this without removing Istio?**

**A:**

```bash
# Step 1: Identify WHERE the latency is added
# Check envoy sidecar stats
kubectl exec pod-name -c istio-proxy -- pilot-agent request GET stats | grep -i "latency\|timeout\|retry"

# Step 2: Check if mTLS handshake is the culprit
# First request to a new destination = TLS handshake (expensive)
# Subsequent requests should reuse connection
kubectl exec pod-name -c istio-proxy -- pilot-agent request GET clusters | grep "cx_active"
# If cx_active is always 0/1 → connections not being reused!

# Step 3: Common latency causes in Istio
```

| Cause | Latency Added | Fix |
|-------|--------------|-----|
| mTLS handshake (no connection reuse) | 100-200ms | Enable connection pooling |
| Envoy filter chain processing | 1-5ms | Remove unnecessary filters |
| Mixer policy checks (Istio < 1.5) | 50-100ms | Upgrade Istio (Mixer removed) |
| Sidecar resource limits too low | 50-200ms | Increase CPU limits on sidecar |
| Outlier detection retry storms | Variable | Tune retry budgets |
| DNS resolution through sidecar | 5-50ms | Use Istio DNS proxying |

```yaml
# Fix: Connection pooling (prevent new TLS handshake per request)
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: optimize-connections
spec:
  host: my-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 5s
      http:
        h2UpgradePolicy: DEFAULT
        maxRequestsPerConnection: 0  # Unlimited (keep connection alive)
    tls:
      mode: ISTIO_MUTUAL

# Fix: Increase sidecar resources
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    defaultConfig:
      concurrency: 2  # Number of worker threads
  values:
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m     # Low CPU limit = sidecar throttled = latency!
            memory: 256Mi

# Fix: Scope sidecar to only needed services (reduce config size)
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: my-service
  namespace: production
spec:
  egress:
  - hosts:
    - "./*"                              # Same namespace
    - "istio-system/*"                   # Istio infra
    - "database-namespace/postgres.database-namespace.svc.cluster.local"  # Only specific external service
    # NOT: "~/*" (all namespaces) — this makes envoy config HUGE
```

**Tricky**: In large clusters (200+ services), each Envoy sidecar receives the configuration for ALL services in the mesh. With 200 services × routes × endpoints, the Envoy config can be 50MB+. Every config push from Istiod causes Envoy to recompute its routing table → CPU spike → latency spike. Use `Sidecar` resources to limit what each pod's envoy knows about (only the services it actually calls). This alone can reduce Envoy memory by 90% and eliminate config-push latency.

---

## Storage & StatefulSet Failures

---

**Q5: A StatefulSet pod (`redis-cluster-2`) is stuck in `Pending` because its PVC can't be bound. The PV exists and shows `Available`. Why won't it bind?**

**A:**

```bash
# Check PVC and PV details
kubectl describe pvc data-redis-cluster-2
kubectl describe pv pv-redis-2

# Common reasons PV Available but PVC won't bind:
```

| Check | Command | Gotcha |
|-------|---------|--------|
| Storage class mismatch | `kubectl get pv pv-redis-2 -o yaml \| grep storageClassName` | PV class must EXACTLY match PVC class (empty ≠ "default") |
| Access mode mismatch | Check ReadWriteOnce vs ReadWriteMany | EBS only supports RWO |
| Capacity mismatch | PVC requests 10Gi, PV only has 5Gi | PV must be ≥ PVC request |
| Node affinity | PV has `nodeAffinity` restricting zones | Pod must schedule in same AZ |
| Already claimed | PV `claimRef` points to old/deleted PVC | Manual clear needed |
| Label selector | PVC has `selector.matchLabels` | PV must have matching labels |

**The AZ problem (most common in production):**
```
EBS volume az: us-east-1a
Pod scheduled on node in: us-east-1b
EBS volumes CANNOT cross AZs!

StatefulSet pod previously ran on node in 1a
Node was terminated (spot instance / scaling)
New pod scheduled on node in 1b → CAN'T mount volume from 1a
```

**Fix:**
```yaml
# Option 1: Force pod to same AZ using topology constraints
apiVersion: apps/v1
kind: StatefulSet
spec:
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule

# Option 2: Use StorageClass with volumeBindingMode
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer  # Create volume in same AZ as pod
parameters:
  type: gp3

# Option 3: Clear stale claimRef (if PV shows "Released" not "Available")
kubectl patch pv pv-redis-2 -p '{"spec":{"claimRef": null}}'
# WARNING: This might cause data to be mounted by wrong pod!
```

**Tricky**: `volumeBindingMode: WaitForFirstConsumer` should be the DEFAULT for all cloud storage classes but often isn't. Without it, the provisioner creates the EBS volume in a random AZ. Then the pod MUST schedule in that AZ. If that AZ has no capacity → stuck forever. With `WaitForFirstConsumer`, the scheduler picks a node FIRST, then creates the volume in the same AZ. Also, PV reclaim policy `Retain` means deleted PVCs leave PVs in `Released` state — they won't auto-bind to new PVCs without manual intervention.

---

## Cluster Upgrades & Node Management

---

**Q6: You need to upgrade an EKS cluster from 1.27 to 1.29. The cluster runs 200 production services with zero-downtime requirement. What's your upgrade strategy?**

**A:**

**EKS upgrade constraints:**
```
- Can only upgrade ONE minor version at a time (1.27→1.28→1.29)
- Control plane upgrades first, then node groups
- Control plane upgrade: 10-15 minutes (managed by AWS, brief API server unavailability)
- Node group upgrade: rolling replacement of all nodes (could take hours)
- Addon compatibility must be verified at each step
```

**Production upgrade runbook:**
```bash
# PHASE 1: Pre-upgrade validation (days before)
# Check deprecated APIs that will break
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
# Or use kubent (Kube No Trouble)
kubent  # Shows deprecated APIs in use

# Check addon compatibility
aws eks describe-addon-versions --kubernetes-version 1.28 \
  --addon-name vpc-cni --query 'addons[].addonVersions[].addonVersion'

# Check PodDisruptionBudgets
kubectl get pdb --all-namespaces
# Any PDB with minAvailable=100% or maxUnavailable=0 will BLOCK node drain!

# PHASE 2: Upgrade control plane
aws eks update-cluster-version --name prod-cluster --kubernetes-version 1.28
aws eks wait cluster-active --name prod-cluster

# PHASE 3: Upgrade addons (order matters!)
aws eks update-addon --cluster-name prod-cluster --addon-name vpc-cni --addon-version v1.15.0
aws eks update-addon --cluster-name prod-cluster --addon-name kube-proxy --addon-version v1.28.1
aws eks update-addon --cluster-name prod-cluster --addon-name coredns --addon-version v1.10.1

# PHASE 4: Upgrade node groups (the risky part)
# Strategy: Blue-Green node groups (safest)
# 1. Create NEW node group with 1.28
aws eks create-nodegroup --cluster-name prod-cluster \
  --nodegroup-name workers-1-28 --node-role $ROLE_ARN \
  --instance-types m5.xlarge --scaling-config minSize=10,maxSize=50,desiredSize=20

# 2. Cordon old nodes (prevent new scheduling)
kubectl get nodes -l eks.amazonaws.com/nodegroup=workers-1-27 -o name | \
  xargs -I {} kubectl cordon {}

# 3. Drain old nodes one by one (respects PDB)
kubectl get nodes -l eks.amazonaws.com/nodegroup=workers-1-27 -o name | \
  xargs -I {} kubectl drain {} --ignore-daemonsets --delete-emptydir-data --grace-period=120

# 4. Verify all pods running on new nodes
kubectl get pods --all-namespaces -o wide | grep "workers-1-27"  # Should be empty

# 5. Delete old node group
aws eks delete-nodegroup --cluster-name prod-cluster --nodegroup-name workers-1-27

# PHASE 5: Repeat for 1.28→1.29
```

**Tricky**: During EKS control plane upgrade, the API server has brief unavailability (seconds to minutes). This means: `kubectl` commands fail temporarily, HPA can't scale, cluster autoscaler can't provision nodes, and any deployment in progress might stall. NEVER upgrade control plane during peak traffic. Also, the kubelet on existing nodes continues running — pods are fine, you just can't make changes. Schedule upgrades during low-traffic windows and ensure enough headroom that you don't need autoscaling during the upgrade.

---

## RBAC & Security Production Issues

---

**Q7: A developer reports they can `kubectl get pods` but `kubectl logs` returns "forbidden." They have the role `pod-reader` which includes `get`, `list`, `watch` on pods. What's missing?**

**A:**

```bash
# Check what they can actually do
kubectl auth can-i get pods --as=developer@company.com -n production  # ✓ yes
kubectl auth can-i get pods/log --as=developer@company.com -n production  # ✗ no!

# The problem: "logs" is a SUB-RESOURCE of pods, not part of "pods" permission
```

**Kubernetes RBAC sub-resources:**
```yaml
# Pod sub-resources that need SEPARATE permissions:
# pods/log       → kubectl logs
# pods/exec      → kubectl exec
# pods/portforward → kubectl port-forward
# pods/attach    → kubectl attach
# pods/status    → patch pod status
# pods/eviction  → evict pods

# Correct role for a developer who needs logs:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-with-logs
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]    # Sub-resource!
  verbs: ["get"]
- apiGroups: [""]
  resources: ["pods/exec"]   # If they need exec too
  verbs: ["create"]          # exec is a CREATE operation, not GET!
```

**Tricky**: `kubectl exec` requires `create` permission on `pods/exec`, not `get`. This is because exec creates a new connection/session. Similarly, `kubectl port-forward` requires `create` on `pods/portforward`. Many RBAC setups miss this because it's counterintuitive. Also, `kubectl logs --follow` requires BOTH `get` on `pods/log` AND a persistent connection — if NetworkPolicies block long-lived connections to the API server, `--follow` fails while regular `logs` works.

---

## HPA & Autoscaling Gotchas

---

**Q8: Your HPA is configured to scale between 3-50 pods based on CPU at 70% target. During a traffic spike, HPA scales to 50 pods but response times are WORSE than with 3 pods. More pods = worse performance. What's happening?**

**A:**

```
Why more pods can HURT performance:
```

| Cause | Explanation | Fix |
|-------|-------------|-----|
| Database connection exhaustion | 50 pods × 10 connections = 500 (DB limit: 100) | Use connection pooler (PgBouncer) |
| Thundering herd on cold cache | 47 new pods all miss cache simultaneously | Warm cache before adding pods |
| Downstream rate limiting | External API limit: 100 req/s, shared across pods | Implement client-side rate limiting |
| DNS/service discovery storm | 50 new pods all resolve DNS simultaneously | Use NodeLocal DNS cache |
| Node resource pressure | 50 pods on same node, all competing for CPU/network | Set pod anti-affinity + resource requests |
| Load balancer slow propagation | ALB takes 30-60s to register new targets | Configure slow start on target group |

```yaml
# Fix: Controlled scaling with behavior
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60  # Wait before scaling up more
      policies:
      - type: Pods
        value: 5          # Max 5 new pods per minute (not 47 at once!)
        periodSeconds: 60
      - type: Percent
        value: 50         # Or max 50% increase per minute
        periodSeconds: 60
      selectPolicy: Min   # Use the MORE conservative policy
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 3
        periodSeconds: 60

# Fix: Pod topology + anti-affinity (spread across nodes)
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["myservice"]
        topologyKey: kubernetes.io/hostname
```

**Tricky**: HPA uses AVERAGE CPU across all pods. When new pods start, they have 0% CPU (haven't received traffic yet). This DROPS the average below target → HPA doesn't scale more. Then load balancer sends traffic → new pods spike to 100% → average goes up → HPA adds more pods → repeat oscillation. This is called "flapping." Fix: Use `stabilizationWindowSeconds` and `scaleUp.policies` to prevent rapid oscillation. Also, consider scaling on custom metrics (requests per second) instead of CPU — it's more predictable.

---
