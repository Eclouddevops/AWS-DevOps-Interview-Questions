# Kubernetes Command Line — Deep-Dive Interview Q&A

## Table of Contents
1. [Essential kubectl Commands](#essential-kubectl-commands)
2. [Debugging & Troubleshooting Commands](#debugging--troubleshooting-commands)
3. [Advanced Operations](#advanced-operations)
4. [JSONPath & Custom Output](#jsonpath--custom-output)
5. [Resource Management](#resource-management)
6. [Tricky Command Scenarios](#tricky-command-scenarios)

---

## Essential kubectl Commands

**Q1: What is the difference between `kubectl apply`, `kubectl create`, and `kubectl replace`?**

**A:**

| Command | Behavior | Idempotent | Use Case |
|---------|----------|------------|----------|
| `create` | Creates resource. Fails if exists. | No | One-time creation |
| `apply` | Creates or updates. Uses 3-way merge. | Yes | Declarative management (GitOps) |
| `replace` | Deletes and recreates. Fails if not exists. | No | Full resource replacement |

**The 3-way merge in `apply`:**
```
apply compares:
1. Current live state (what's in cluster)
2. Last applied configuration (annotation: kubectl.kubernetes.io/last-applied-configuration)
3. New desired state (your YAML)

If a field exists in last-applied but NOT in new YAML → field is REMOVED
If a field exists in live but NOT in last-applied → field is PRESERVED (someone else added it)
```

**Tricky**: `kubectl apply` stores the entire manifest in an annotation. For large resources (ConfigMaps with big data), this can exceed the 256KB annotation limit. Use `--server-side` for server-side apply (no annotation needed).

---

**Q2: How do you switch between clusters and namespaces efficiently?**

**A:**

```bash
# View all contexts
kubectl config get-contexts

# Switch context (cluster)
kubectl config use-context production-cluster

# Set default namespace for current context
kubectl config set-context --current --namespace=production

# Quick tools (install kubectx/kubens)
kubectx production    # Switch cluster
kubens production     # Switch namespace

# One-liner: Run command in different context without switching
kubectl --context=staging --namespace=dev get pods

# Create context alias
kubectl config set-context prod-admin \
  --cluster=production \
  --user=admin \
  --namespace=default
```

**Tricky**: If KUBECONFIG environment variable has multiple files, they merge:
```bash
export KUBECONFIG=~/.kube/config:~/.kube/staging-config:~/.kube/prod-config
# All contexts from all files are available
```

---

**Q3: Explain `kubectl get` output formats and when to use each.**

**A:**

```bash
# Default table
kubectl get pods

# Wide - shows node, IP
kubectl get pods -o wide

# YAML - full resource definition
kubectl get pod my-pod -o yaml

# JSON - for programmatic parsing
kubectl get pod my-pod -o json

# JSONPath - extract specific fields
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Custom columns - table with specific fields
kubectl get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP'

# Name only
kubectl get pods -o name
# Output: pod/my-pod-abc123

# Go template
kubectl get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
```

**Power moves:**
```bash
# Sort by restart count (find problematic pods)
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# Sort by creation time
kubectl get pods --sort-by='.metadata.creationTimestamp'

# Show labels as columns
kubectl get pods -L app,version

# Field selectors (server-side filtering)
kubectl get pods --field-selector=status.phase=Running,spec.nodeName=node-1
```

---

## Debugging & Troubleshooting Commands

**Q4: A pod is in CrashLoopBackOff. Walk through the debugging commands.**

**A:**

```bash
# Step 1: Check pod events and status
kubectl describe pod crashing-pod
# Look at: Events, Last State, Exit Code, Reason

# Step 2: Check logs (current attempt)
kubectl logs crashing-pod

# Step 3: Check PREVIOUS container's logs (crashed instance)
kubectl logs crashing-pod --previous

# Step 4: Check all containers in pod (init + sidecar)
kubectl logs crashing-pod --all-containers=true

# Step 5: If container crashes too fast, use debug container
kubectl debug crashing-pod -it --image=busybox --target=main-container

# Step 6: Check resource limits (OOMKilled?)
kubectl get pod crashing-pod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# "OOMKilled" → increase memory limit

# Step 7: Check if image exists and is pullable
kubectl get pod crashing-pod -o jsonpath='{.status.containerStatuses[0].state}'

# Step 8: Check events in namespace for broader issues
kubectl get events --sort-by='.lastTimestamp' -n my-namespace | tail -20
```

**Exit code meanings:**
| Code | Meaning |
|------|---------|
| 0 | Success (container completed normally — maybe should be Job, not Deployment) |
| 1 | Application error |
| 137 | SIGKILL (OOMKilled or `kill -9`) |
| 139 | SIGSEGV (segmentation fault) |
| 143 | SIGTERM (graceful termination) |

---

**Q5: How do you debug a pod that is stuck in `Pending` state?**

**A:**

```bash
# Check why it's pending
kubectl describe pod pending-pod
# Look at Events section and Conditions

# Common reasons and their messages:

# 1. Insufficient resources
# "0/5 nodes are available: 5 Insufficient cpu"
kubectl describe nodes | grep -A5 "Allocated resources"
kubectl top nodes

# 2. No matching node (nodeSelector/affinity)
# "0/5 nodes are available: 5 node(s) didn't match Pod's node affinity"
kubectl get pod pending-pod -o yaml | grep -A5 nodeSelector

# 3. PVC not bound
# "persistentvolumeclaim "data" not found" or "unbound"
kubectl get pvc -n my-namespace
kubectl describe pvc my-pvc

# 4. Taints preventing scheduling
# "0/5 nodes are available: 5 node(s) had untolerated taint"
kubectl get nodes -o json | jq '.items[].spec.taints'
kubectl get pod pending-pod -o yaml | grep -A5 tolerations

# 5. ResourceQuota exceeded
kubectl get resourcequota -n my-namespace
kubectl describe resourcequota -n my-namespace

# 6. Too many pods on node (maxPods limit)
kubectl get nodes -o jsonpath='{.items[*].status.capacity.pods}'
```

---

**Q6: How do you exec into a pod with multiple containers? What if the container has no shell?**

**A:**

```bash
# Exec into specific container
kubectl exec -it my-pod -c sidecar-container -- /bin/bash

# If no bash, try sh
kubectl exec -it my-pod -c main -- /bin/sh

# If NO shell at all (distroless images):
# Use ephemeral debug container (K8s 1.23+)
kubectl debug -it my-pod --image=busybox --target=main

# This shares PID namespace with target container
# You can see processes of the main container from busybox

# Debug with full tools
kubectl debug -it my-pod --image=nicolaka/netshoot --target=main

# Copy pod for debugging (creates a clone with modifications)
kubectl debug my-pod -it --copy-to=debug-pod --container=debug --image=busybox

# Debug a node directly
kubectl debug node/my-node -it --image=busybox
# Creates a privileged pod on the node with host filesystem at /host
```

**Tricky**: `kubectl exec` requires `pods/exec` RBAC permission, not just `pods`. Many read-only roles forget this.

---

**Q7: How do you check resource usage and identify resource-hungry pods?**

**A:**

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage (requires metrics-server)
kubectl top pods --all-namespaces --sort-by=memory
kubectl top pods --all-namespaces --sort-by=cpu

# Top pods in a namespace with containers breakdown
kubectl top pods -n production --containers

# Find pods without resource limits (dangerous!)
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits == null) | .metadata.name'

# Find pods using most CPU relative to their requests
kubectl top pods -n production --no-headers | sort -k3 -rn | head -10

# Check node allocatable vs capacity
kubectl describe node my-node | grep -A6 "Allocated resources"

# Find unschedulable nodes
kubectl get nodes -o json | jq '.items[] | select(.spec.unschedulable==true) | .metadata.name'
```

---

## Advanced Operations

**Q8: How do you perform a rolling restart without changing the spec?**

**A:**

```bash
# Rolling restart (K8s 1.15+) - adds annotation to trigger rollout
kubectl rollout restart deployment/my-app

# Check rollout status
kubectl rollout status deployment/my-app

# Watch rollout progress
kubectl rollout status deployment/my-app --watch

# View rollout history
kubectl rollout history deployment/my-app

# View specific revision
kubectl rollout history deployment/my-app --revision=3

# Undo to previous version
kubectl rollout undo deployment/my-app

# Undo to specific revision
kubectl rollout undo deployment/my-app --to-revision=2

# Pause a rollout (for canary-like control)
kubectl rollout pause deployment/my-app
# Make changes...
kubectl rollout resume deployment/my-app
```

**Tricky**: `kubectl rollout restart` works by adding an annotation:
`kubectl.kubernetes.io/restartedAt: "2024-01-01T00:00:00Z"`
This changes the pod template → triggers new rollout. Old ReplicaSets are kept for rollback.

---

**Q9: How do you scale resources and manage autoscaling from CLI?**

**A:**

```bash
# Manual scale
kubectl scale deployment/my-app --replicas=5

# Scale multiple deployments
kubectl scale deployment/app1 deployment/app2 --replicas=3

# Scale statefulset
kubectl scale statefulset/my-db --replicas=3

# Create HPA
kubectl autoscale deployment/my-app --min=2 --max=10 --cpu-percent=70

# Check HPA status
kubectl get hpa my-app
# Shows: TARGETS (current/target), MINPODS, MAXPODS, REPLICAS

# Describe HPA for detailed metrics and events
kubectl describe hpa my-app

# Scale to zero (for dev environments)
kubectl scale deployment/my-app --replicas=0

# Conditional scale (only if current replicas match)
kubectl scale deployment/my-app --replicas=5 --current-replicas=3
# Fails if current replicas != 3 (prevents race conditions)
```

---

**Q10: Explain `kubectl patch` vs `kubectl edit` vs `kubectl apply`. When to use each?**

**A:**

```bash
# kubectl edit - Interactive editor (vim/nano)
# Use: Quick one-off changes, exploring resource structure
kubectl edit deployment/my-app
# Opens full YAML in editor, applies on save

# kubectl patch - Programmatic partial update
# Use: Scripts, automation, CI/CD pipelines
# Strategic Merge Patch (default):
kubectl patch deployment/my-app -p '{"spec":{"replicas":5}}'

# JSON Merge Patch:
kubectl patch deployment/my-app --type=merge -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","image":"app:v2"}]}}}}'

# JSON Patch (precise operations):
kubectl patch deployment/my-app --type=json -p='[{"op":"replace","path":"/spec/replicas","value":5}]'

# kubectl apply - Declarative full resource management
# Use: GitOps, maintaining desired state in YAML files
kubectl apply -f deployment.yaml
```

**Tricky difference:**
- `patch` with strategic merge: Arrays are MERGED by key (e.g., container name)
- `patch` with JSON merge: Arrays are REPLACED entirely
- `patch` with JSON patch: Individual array operations (add/remove/replace by index)

---

**Q11: How do you drain a node safely for maintenance?**

**A:**

```bash
# Step 1: Cordon node (prevent new scheduling)
kubectl cordon node-1
# Node shows SchedulingDisabled

# Step 2: Drain (evict existing pods)
kubectl drain node-1 \
  --ignore-daemonsets \          # Don't error on DaemonSet pods
  --delete-emptydir-data \       # Allow evicting pods with emptyDir
  --grace-period=60 \            # Give pods 60s to shutdown
  --timeout=300s \               # Max time to wait for drain
  --pod-selector='app!=critical' # Only drain matching pods (optional)

# Step 3: Perform maintenance...

# Step 4: Uncordon (allow scheduling again)
kubectl uncordon node-1

# Check which nodes are cordoned
kubectl get nodes | grep SchedulingDisabled

# Drain with force (for pods not managed by controller)
kubectl drain node-1 --force  # Will DELETE standalone pods permanently!
```

**Common drain failures and fixes:**
```bash
# "cannot delete Pods with local storage"
kubectl drain node-1 --delete-emptydir-data

# "cannot delete DaemonSet-managed Pods"
kubectl drain node-1 --ignore-daemonsets

# "Cannot evict pod as it would violate PDB"
# Check PDB: kubectl get pdb
# Either wait for other pods to be ready, or delete PDB temporarily

# Pod stuck in Terminating during drain
kubectl delete pod stuck-pod --grace-period=0 --force
```

---

## JSONPath & Custom Output

**Q12: Write kubectl commands using JSONPath to extract specific information.**

**A:**

```bash
# Get all pod names
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Get pod names with their IPs (formatted)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Get all container images in cluster
kubectl get pods --all-namespaces -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# Get nodes and their internal IPs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'

# Get pods that are NOT running
kubectl get pods -o jsonpath='{.items[?(@.status.phase!="Running")].metadata.name}'

# Get pods with restart count > 5
kubectl get pods -o json | jq -r '.items[] | select(.status.containerStatuses[0].restartCount > 5) | .metadata.name'

# Get all secrets of type Opaque
kubectl get secrets -o jsonpath='{.items[?(@.type=="Opaque")].metadata.name}'

# Get PVCs and their storage class
kubectl get pvc -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,CLASS:.spec.storageClassName,SIZE:.spec.resources.requests.storage'

# Get pods sorted by memory usage (using top + sort)
kubectl top pods --no-headers | sort -k4 -rn
```

---

**Q13: How do you export a running resource's YAML for backup or migration?**

**A:**

```bash
# Get YAML (includes status, managed fields - messy)
kubectl get deployment/my-app -o yaml

# Clean export (remove cluster-specific metadata)
kubectl get deployment/my-app -o yaml | \
  kubectl neat  # requires kubectl-neat plugin

# Manual cleanup with yq
kubectl get deployment/my-app -o yaml | \
  yq 'del(.metadata.uid, .metadata.resourceVersion, .metadata.creationTimestamp, .metadata.generation, .metadata.managedFields, .status)'

# Export all resources in a namespace
kubectl get all -n production -o yaml > production-backup.yaml

# Export specific resource types
for type in deployment service configmap secret ingress; do
  kubectl get $type -n production -o yaml > ${type}-backup.yaml
done

# Dry-run to generate YAML without applying
kubectl create deployment my-app --image=nginx --dry-run=client -o yaml > deployment.yaml

# Generate from existing with modifications
kubectl get deployment/my-app -o yaml | \
  sed 's/replicas: 3/replicas: 5/' | \
  kubectl apply -f -
```

**Tricky**: `kubectl get all` doesn't actually get ALL resources. It misses: ConfigMaps, Secrets, PVCs, NetworkPolicies, ServiceAccounts, Roles, etc. Use:
```bash
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I {} kubectl get {} -n production -o yaml
```

---

## Resource Management

**Q14: How do you manage resource quotas and limit ranges from CLI?**

**A:**

```bash
# Check resource quotas in namespace
kubectl get resourcequota -n production
kubectl describe resourcequota -n production

# Check limit ranges
kubectl get limitrange -n production
kubectl describe limitrange -n production

# See current resource usage vs quota
kubectl describe resourcequota compute-quota -n production
# Shows: Used / Hard for each resource

# Create quota imperatively
kubectl create quota my-quota -n dev \
  --hard=cpu=4,memory=8Gi,pods=20,services=10

# Check which pods are consuming most resources
kubectl top pods -n production --sort-by=cpu

# Find pods without resource requests (potential noisy neighbors)
kubectl get pods -n production -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.requests == null) | "\(.metadata.name) - \(.spec.containers[].name)"'

# Check node capacity and allocation
kubectl get nodes -o custom-columns='NODE:.metadata.name,CPU_CAP:.status.capacity.cpu,MEM_CAP:.status.capacity.memory,PODS_CAP:.status.capacity.pods'
```

---

**Q15: How do you troubleshoot RBAC permission issues from CLI?**

**A:**

```bash
# Check if current user can perform action
kubectl auth can-i create deployments -n production
# Output: yes/no

# Check all permissions for current user
kubectl auth can-i --list -n production

# Check permissions for a specific user
kubectl auth can-i get secrets --as=john -n production

# Check permissions for a service account
kubectl auth can-i get pods \
  --as=system:serviceaccount:default:my-service-account \
  -n production

# Check what groups a user belongs to
kubectl auth can-i --list --as=jane --as-group=developers -n production

# Find all roles/clusterroles bound to a user
kubectl get rolebindings,clusterrolebindings --all-namespaces -o json | \
  jq -r '.items[] | select(.subjects[]? | .name=="john") | "\(.metadata.namespace)/\(.metadata.name) → \(.roleRef.name)"'

# Find all roles that grant access to secrets
kubectl get roles,clusterroles --all-namespaces -o json | \
  jq -r '.items[] | select(.rules[]?.resources[]? == "secrets") | "\(.metadata.namespace)/\(.metadata.name)"'

# Create a role quickly for testing
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n dev
kubectl create rolebinding pod-reader-binding --role=pod-reader --user=john -n dev
```

---

## Tricky Command Scenarios

**Q16: How do you find all pods that are NOT controlled by a Deployment/ReplicaSet (orphan pods)?**

**A:**

```bash
# Find pods without ownerReferences (standalone/orphan pods)
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.metadata.ownerReferences == null) | "\(.metadata.namespace)/\(.metadata.name)"'

# Find pods whose controller no longer exists
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.metadata.ownerReferences != null) | "\(.metadata.namespace)/\(.metadata.name) owned by \(.metadata.ownerReferences[0].kind)/\(.metadata.ownerReferences[0].name)"' | \
  while read line; do
    # Check if owner exists
    echo "$line"
  done

# Find pods in Terminating state (stuck)
kubectl get pods --all-namespaces --field-selector=metadata.deletionTimestamp!='' 2>/dev/null || \
kubectl get pods --all-namespaces | grep Terminating

# Force delete stuck terminating pods
kubectl get pods --all-namespaces | grep Terminating | \
  awk '{print $1, $2}' | \
  while read ns pod; do
    kubectl delete pod $pod -n $ns --grace-period=0 --force
  done
```

---

**Q17: How do you copy files to/from a pod? What about between pods?**

**A:**

```bash
# Copy file from local to pod
kubectl cp ./local-file.txt my-pod:/tmp/file.txt

# Copy from pod to local
kubectl cp my-pod:/var/log/app.log ./app.log

# Copy with specific container
kubectl cp my-pod:/data ./backup -c sidecar

# Copy from one pod to another (via local)
kubectl cp pod-a:/data/file.txt ./temp.txt
kubectl cp ./temp.txt pod-b:/data/file.txt

# Copy entire directory
kubectl cp my-pod:/var/log/ ./logs/

# Namespace-aware copy
kubectl cp production/my-pod:/config ./config-backup

# Streaming (large files - avoids temp storage)
kubectl exec my-pod -- tar cf - /data | tar xf - -C ./backup/

# Copy to pod without kubectl cp (if tar not available in pod)
cat local-file.txt | kubectl exec -i my-pod -- tee /tmp/file.txt > /dev/null
```

**Tricky**: `kubectl cp` requires `tar` binary in the container. For distroless/minimal images, use:
```bash
# Write file to pod without tar
kubectl exec my-pod -i -- sh -c 'cat > /tmp/file.txt' < local-file.txt
```

---

**Q18: How do you quickly create test resources without YAML files?**

**A:**

```bash
# Create deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Create and expose in one command
kubectl create deployment web --image=nginx && \
kubectl expose deployment web --port=80 --type=NodePort

# Run a one-shot pod (like docker run)
kubectl run test --image=busybox --restart=Never --rm -it -- /bin/sh

# Run with specific command
kubectl run curl-test --image=curlimages/curl --restart=Never --rm -it -- curl http://my-service

# Create configmap from literal
kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2

# Create configmap from file
kubectl create configmap nginx-config --from-file=nginx.conf

# Create secret
kubectl create secret generic db-creds --from-literal=username=admin --from-literal=password=secret123

# Create service account
kubectl create serviceaccount my-sa

# Create job
kubectl create job my-job --image=busybox -- sh -c "echo hello && sleep 30"

# Create cronjob
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo "hello"

# Generate YAML without creating (dry-run)
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
```

---

**Q19: What are the most useful kubectl plugins? How do you install them?**

**A:**

```bash
# Install krew (plugin manager)
kubectl krew install krew

# Essential plugins:
kubectl krew install ctx        # Quick context switching
kubectl krew install ns         # Quick namespace switching
kubectl krew install neat       # Clean YAML output
kubectl krew install tree       # Show resource hierarchy
kubectl krew install images     # Show images used by pods
kubectl krew install access-matrix  # RBAC matrix view
kubectl krew install sniff      # Packet capture from pods
kubectl krew install node-shell # SSH into nodes
kubectl krew install resource-capacity  # Node resource overview
kubectl krew install who-can    # Who can perform action?
kubectl krew install deprecations  # Check deprecated APIs (pluto)

# Usage examples:
kubectl tree deployment my-app   # Shows RS → Pods hierarchy
kubectl who-can get secrets -n production
kubectl resource-capacity --sort cpu.util --pods
kubectl images -n production    # All images with versions
```

---

**Q20: In a CKA/CKAD exam scenario: Create a NetworkPolicy that allows ingress only from pods with label `role=frontend` on port 8080, and allows egress only to DNS (port 53) and pods with label `role=database` on port 5432. Do it using only kubectl.**

**A:**

```bash
# You can't create NetworkPolicy purely imperatively, but you can use dry-run + heredoc:
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: database
    ports:
    - protocol: TCP
      port: 5432
  - to: []  # Any destination
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF

# Verify it's applied
kubectl get networkpolicy backend-policy -n production -o yaml

# Test connectivity
kubectl exec frontend-pod -- curl backend-pod:8080  # Should work
kubectl exec other-pod -- curl backend-pod:8080     # Should fail
```

**Exam tips:**
- Use `kubectl explain networkpolicy.spec.ingress` to check field names
- Always specify BOTH Ingress and Egress in `policyTypes` if you want to restrict both
- Don't forget DNS egress (port 53 UDP+TCP) or pods can't resolve service names!



---

## Additional Scenario-Based Tricky Questions

---

**Q21: You need to find all pods that are consuming more than 80% of their memory limit across the entire cluster. Write the kubectl command and explain what you'd do with the results.**

**A:**

```bash
# Method 1: Using kubectl top + custom processing
kubectl top pods --all-namespaces --no-headers | while read ns name cpu mem; do
  # Get memory limit for each pod
  LIMIT=$(kubectl get pod "$name" -n "$ns" -o jsonpath='{.spec.containers[0].resources.limits.memory}' 2>/dev/null)
  if [ -n "$LIMIT" ]; then
    # Convert to Mi for comparison
    USAGE_MI=$(echo "$mem" | sed 's/Mi//')
    LIMIT_MI=$(echo "$LIMIT" | sed 's/Mi//; s/Gi/*1024/' | bc)
    if [ "$USAGE_MI" -gt 0 ] && [ "$LIMIT_MI" -gt 0 ]; then
      PCT=$((USAGE_MI * 100 / LIMIT_MI))
      if [ "$PCT" -gt 80 ]; then
        echo "WARNING: $ns/$name using ${PCT}% of memory limit ($mem / $LIMIT)"
      fi
    fi
  fi
done

# Method 2: One-liner with custom-columns + sorting
kubectl top pods -A --sort-by=memory --no-headers | head -20

# Method 3: Using Prometheus query (production approach)
# container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.8
```

**What to do with results:**
```bash
# 1. Check if pods are about to OOMKill
kubectl describe pod high-memory-pod -n prod | grep -A5 "Last State"
# If "OOMKilled" in history → memory leak or undersized limit

# 2. Get memory growth trend
kubectl exec -n monitoring prometheus-0 -- promtool query instant \
  'rate(container_memory_usage_bytes{pod="high-memory-pod"}[1h])'

# 3. Right-size or restart
# If growing steadily → memory leak → restart as temporary fix
kubectl rollout restart deployment/leaky-service -n prod

# If stable at 80% → increase limit (with 25% headroom)
kubectl patch deployment myservice -n prod -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"myservice","resources":{"limits":{"memory":"2Gi"}}}]}}}}'
```

**Tricky**: `kubectl top` shows CURRENT usage, not peak. A pod showing 50% now might spike to 95% during peak hours. For proper capacity planning, use Prometheus `container_memory_usage_bytes` with `max_over_time(... [7d])` to see the 7-day peak. Also, `kubectl top` shows working set memory (RSS + cache), while OOMKill triggers on RSS only in some kernel versions. The numbers might not match exactly.

---

**Q22: A node is NotReady and you need to safely move all workloads off it without losing any in-progress work. Walk through the complete drain procedure with all edge cases.**

**A:**

```bash
# Step 1: Investigate WHY it's NotReady (before draining)
kubectl describe node problem-node | grep -A 10 "Conditions"
# MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable?

# Step 2: Cordon first (prevent new scheduling)
kubectl cordon problem-node
# Node is now SchedulingDisabled — no new pods will land here

# Step 3: Check what's running on the node
kubectl get pods --all-namespaces --field-selector spec.nodeName=problem-node
# Note: DaemonSets, StatefulSets, pods with local storage, pods without PDB

# Step 4: Drain with safety flags
kubectl drain problem-node \
  --ignore-daemonsets \           # Don't try to evict DaemonSet pods
  --delete-emptydir-data \        # Allow deletion of pods using emptyDir
  --grace-period=120 \            # Give pods 2 minutes to shutdown gracefully
  --timeout=600 \                 # Wait max 10 minutes for drain to complete
  --pod-selector='app!=critical-singleton'  # Skip specific pods if needed

# Step 5: If drain is stuck (common!)
kubectl get pods --field-selector spec.nodeName=problem-node
# PodDisruptionBudget might be blocking eviction

# Check PDB status
kubectl get pdb --all-namespaces
# If PDB says "Allowed disruptions: 0" → drain is blocked
```

**Edge cases that block drain:**
```bash
# Case 1: PDB with maxUnavailable=0 or minAvailable=100%
# Fix: Temporarily relax PDB
kubectl patch pdb my-pdb -p '{"spec":{"maxUnavailable": 1}}'

# Case 2: Pod with no controller (bare pod)
# kubectl drain won't evict pods not managed by ReplicaSet/Deployment
# Fix: Use --force flag (pod will be deleted, not rescheduled!)
kubectl drain problem-node --force

# Case 3: Local PersistentVolume
# Pod using hostPath or local PV can't be rescheduled elsewhere
# Fix: Handle manually — backup data, delete pod, recreate on new node

# Case 4: StatefulSet pod with "ordinal stuck"
# Pod redis-2 won't be recreated until redis-2 is fully terminated
# If termination takes long (finalizer stuck): patch to remove finalizer
kubectl patch pod redis-2 -p '{"metadata":{"finalizers":null}}'

# Case 5: Pods in Terminating state (stuck)
kubectl get pods --field-selector spec.nodeName=problem-node,status.phase=Running
# Force delete stuck terminating pods:
kubectl delete pod stuck-pod --grace-period=0 --force
```

**Tricky**: `kubectl drain` respects PodDisruptionBudgets (PDB). If a PDB says "minimum 2 pods must be available" and only 2 pods exist, drain CANNOT evict — it will hang forever. Before draining, always check: (1) Are there enough replicas across other nodes? (2) Is the PDB configured to allow disruption? (3) Are there bare pods without controllers? A common production mistake: setting `minAvailable: 100%` which prevents ALL voluntary disruptions including drains and upgrades.

---

**Q23: Your team needs to quickly identify all resources (pods, services, configmaps, secrets) associated with a specific application across all namespaces. The label `app=payment-service` is used but not consistently. How do you find everything?**

**A:**

```bash
# Method 1: Label-based search (finds labeled resources)
kubectl get all,configmap,secret,pvc,ingress,networkpolicy,serviceaccount \
  --all-namespaces -l app=payment-service

# Method 2: Name-based search (finds resources by naming convention)
kubectl get all --all-namespaces | grep -i payment
kubectl get configmap --all-namespaces | grep -i payment
kubectl get secret --all-namespaces | grep -i payment
kubectl get ingress --all-namespaces | grep -i payment

# Method 3: Find by owner references (follow the chain)
# Deployment → ReplicaSet → Pods
DEPLOY=$(kubectl get deploy -A -l app=payment-service -o name)
kubectl get rs -A -o json | jq -r ".items[] | select(.metadata.ownerReferences[]?.name == \"payment-service\") | .metadata.name"

# Method 4: Using kubectl-tree plugin (shows resource hierarchy)
kubectl tree deployment payment-service -n production
# Output:
# NAMESPACE    NAME                                    READY
# production   Deployment/payment-service              True
# production   ├── ReplicaSet/payment-service-abc123   True
# production   │   ├── Pod/payment-service-abc123-x1   True
# production   │   └── Pod/payment-service-abc123-x2   True
# production   └── ReplicaSet/payment-service-old456   False

# Method 5: Find all resources that REFERENCE this service
# Find ConfigMaps/Secrets mounted by the pods
kubectl get pod -n production -l app=payment-service -o json | \
  jq -r '.items[].spec | 
    (.volumes[]? | select(.configMap) | "ConfigMap: " + .configMap.name),
    (.volumes[]? | select(.secret) | "Secret: " + .secret.secretName),
    (.containers[].envFrom[]? | select(.configMapRef) | "ConfigMap: " + .configMapRef.name),
    (.containers[].envFrom[]? | select(.secretRef) | "Secret: " + .secretRef.name)'

# Method 6: Full audit with Helm (if Helm-managed)
helm list -A | grep payment
helm get all payment-service -n production  # Shows ALL resources managed by this release
```

**Tricky**: `kubectl get all` does NOT actually get ALL resources. It only returns: pods, services, deployments, replicasets, statefulsets, daemonsets, jobs, cronjobs. It MISSES: configmaps, secrets, PVCs, ingresses, networkpolicies, serviceaccounts, roles, rolebindings, HPAs, and custom resources. For a complete inventory, you must explicitly list resource types or use `kubectl api-resources --verbs=list -o name | xargs -I {} kubectl get {} -A -l app=payment-service`.

---
