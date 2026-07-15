# Kubernetes L3/L4 — Security & Version Update Deep-Dive Interview Q&A

## Table of Contents
1. [RBAC & Authentication](#rbac--authentication)
2. [Pod Security](#pod-security)
3. [Network Policies](#network-policies)
4. [Secrets Management](#secrets-management)
5. [Cluster Hardening](#cluster-hardening)
6. [Version Upgrade Strategy](#version-upgrade-strategy)
7. [Tricky Scenario Questions](#tricky-scenario-questions)

---

## RBAC & Authentication

**Q1: Explain the difference between Role, ClusterRole, RoleBinding, and ClusterRoleBinding. When would you use each?**

**A:**

| Resource | Scope | Purpose |
|----------|-------|---------|
| Role | Namespace-scoped | Defines permissions within a single namespace |
| ClusterRole | Cluster-wide | Defines permissions cluster-wide OR reusable across namespaces |
| RoleBinding | Namespace-scoped | Binds Role/ClusterRole to subjects within a namespace |
| ClusterRoleBinding | Cluster-wide | Binds ClusterRole to subjects across all namespaces |

**Tricky combinations:**
- ClusterRole + RoleBinding = Grant cluster-defined permissions in a SPECIFIC namespace only
- ClusterRole + ClusterRoleBinding = Grant permissions across ALL namespaces
- Role + ClusterRoleBinding = INVALID (Role is namespace-scoped, can't be bound cluster-wide)

```yaml
# ClusterRole that can be reused per-namespace via RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# Bind only in "dev" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-dev
  namespace: dev
subjects:
- kind: User
  name: developer1
roleRef:
  kind: ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

**Q2: A developer has `get` and `list` permissions on pods but can't run `kubectl logs`. Why?**

**A:** `kubectl logs` requires access to the `pods/log` subresource, not just `pods`. The RBAC rule must include:

```yaml
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```

**Common subresources that catch people off guard:**
- `pods/log` — for `kubectl logs`
- `pods/exec` — for `kubectl exec`
- `pods/portforward` — for `kubectl port-forward`
- `pods/attach` — for `kubectl attach`
- `deployments/scale` — for `kubectl scale`
- `nodes/proxy` — for kubelet API access

---

**Q3: How does Kubernetes authentication work? What are the different methods?**

**A:**

Kubernetes has NO built-in user management. It delegates to external identity providers:

| Method | How It Works | Use Case |
|--------|-------------|----------|
| X.509 Client Certificates | CN=username, O=group | Service accounts, initial cluster setup |
| Bearer Tokens (Static) | Token file on API server | Legacy, NOT recommended |
| OIDC (OpenID Connect) | External IdP (Okta, Azure AD, Google) | Production user authentication |
| Webhook Token Auth | External webhook validates token | Custom auth systems |
| Service Account Tokens | JWT mounted in pods | Pod-to-API-server communication |
| Bootstrap Tokens | Short-lived tokens | Node joining, kubeadm |

**Tricky**: Kubernetes can authenticate users but CANNOT revoke certificates! Once a client certificate is issued, it's valid until expiry. Mitigation:
1. Short certificate lifetimes
2. Use OIDC with token revocation support
3. Add certificate to CRL (if using intermediate CA)

---

**Q4: Explain Service Account Token best practices in Kubernetes 1.24+.**

**A:**

**Before 1.24**: Service account tokens were automatically mounted as non-expiring secrets.

**After 1.24**: Automatic secret creation is removed. Tokens are now:
- Projected via TokenRequestAPI (time-bound, audience-bound)
- Expire after 1 hour (configurable)
- Automatically rotated by kubelet

```yaml
# Modern approach - projected token volume
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false  # Disable auto-mount
  containers:
  - name: app
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: vault  # Specific audience
```

**Security best practices:**
1. Set `automountServiceAccountToken: false` on pods that don't need API access
2. Use separate service accounts per workload (not `default`)
3. Use audience-bound tokens for external services
4. Minimum RBAC permissions on each service account

---

## Pod Security

**Q5: Explain Pod Security Standards (PSS) and Pod Security Admission (PSA). How do they replace PodSecurityPolicy?**

**A:**

PodSecurityPolicy (PSP) was removed in Kubernetes 1.25. Replaced by:

**Pod Security Standards** (3 levels):
| Level | Description | Key Restrictions |
|-------|-------------|-----------------|
| Privileged | Unrestricted (for system workloads) | None |
| Baseline | Minimally restrictive | No hostNetwork, hostPID, privileged containers |
| Restricted | Heavily restricted (best practice) | Must run as non-root, drop ALL capabilities, read-only rootfs |

**Pod Security Admission** (enforcement modes):
| Mode | Behavior |
|------|----------|
| enforce | Reject violating pods |
| audit | Allow but log violations in audit log |
| warn | Allow but show warnings to user |

```yaml
# Namespace-level enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Tricky**: You can set different levels for different modes! Common pattern:
- `enforce: baseline` (blocks obviously dangerous pods)
- `warn: restricted` (shows warnings for non-compliant but allows)
- Gradually move to `enforce: restricted`

---

**Q6: A container must run as root for initial setup but the security policy blocks it. How do you solve this?**

**A:** Use an **init container** with elevated privileges while the main container runs restricted:

```yaml
apiVersion: v1
kind: Pod
spec:
  initContainers:
  - name: setup
    image: busybox
    command: ["sh", "-c", "chown -R 1000:1000 /data"]
    securityContext:
      runAsUser: 0  # Root for setup only
    volumeMounts:
    - name: data
      mountPath: /data
  containers:
  - name: app
    image: my-app
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

**Alternative approaches:**
1. Fix the image to not require root (best solution)
2. Use `fsGroup` in pod security context for volume permissions
3. Use a sidecar with specific capabilities instead of full root

---

**Q7: What are the most critical securityContext fields and why?**

**A:**

```yaml
securityContext:
  runAsNonRoot: true              # CRITICAL: Prevents root execution
  runAsUser: 1000                 # Explicit UID
  runAsGroup: 1000                # Explicit GID
  readOnlyRootFilesystem: true    # Prevents filesystem modification
  allowPrivilegeEscalation: false # Blocks setuid/setgid binaries
  capabilities:
    drop: ["ALL"]                 # Remove ALL Linux capabilities
    add: ["NET_BIND_SERVICE"]     # Add back ONLY what's needed
  seccompProfile:
    type: RuntimeDefault          # Enable seccomp filtering
```

**Why each matters:**
- `runAsNonRoot`: Container breakout as root = host root access
- `readOnlyRootFilesystem`: Prevents attackers from writing malware/shells
- `allowPrivilegeEscalation: false`: Blocks escalation via SUID binaries
- `drop: ALL`: Default Linux capabilities include dangerous ones (NET_RAW, SYS_PTRACE)
- `seccompProfile`: Reduces syscall attack surface (~300+ syscalls → only needed ones)

**Tricky**: `runAsNonRoot: true` will REJECT the pod if the container image's USER is root (UID 0). It validates at runtime, not build time. Always set explicit `runAsUser` to be safe.

---

## Secrets Management

**Q8: Kubernetes Secrets are base64 encoded. How do you actually secure them?**

**A:** Base64 is NOT encryption! Secrets are stored in etcd as plaintext (base64).

**Actual security measures:**

1. **Encryption at Rest** (etcd encryption):
```yaml
# EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}  # Fallback for reading old unencrypted secrets
```

2. **External Secret Managers:**
- AWS Secrets Manager + External Secrets Operator
- HashiCorp Vault + Vault Agent Injector
- Azure Key Vault + CSI Driver
- GCP Secret Manager

3. **RBAC on Secrets**: Restrict who can `get`/`list` secrets
4. **Audit Logging**: Track secret access
5. **Sealed Secrets**: Encrypt secrets for Git storage (GitOps)

**Tricky**: Anyone with `get` permission on Secrets in a namespace can read ALL secrets there. `list` permission exposes secret NAMES (metadata). Even `pods` permission can leak secrets via env vars or volume mounts!

---

## Cluster Hardening

**Q9: List the top 10 Kubernetes hardening measures for production.**

**A:**

1. **Disable anonymous authentication**: `--anonymous-auth=false` on API server
2. **Enable audit logging**: Track all API calls with audit policy
3. **Encrypt etcd**: Both at-rest encryption and mTLS between API server and etcd
4. **Network Policies**: Default-deny all traffic, allow explicitly
5. **Pod Security Admission**: Enforce `restricted` profile
6. **RBAC**: Least privilege, no `cluster-admin` for users
7. **Disable automount of service account tokens**: Unless needed
8. **Image scanning + admission controllers**: Only allow signed/scanned images
9. **Limit API server access**: Private API endpoint, firewall rules
10. **CIS Benchmark**: Run `kube-bench` regularly

**Admission Controllers to enable:**
- `PodSecurity` (replaces PSP)
- `NodeRestriction` (limits kubelet permissions)
- `AlwaysPullImages` (prevents cached image attacks)
- Custom webhooks (OPA Gatekeeper, Kyverno)

---

**Q10: What is OPA Gatekeeper vs Kyverno? When would you choose each?**

**A:**

| Feature | OPA Gatekeeper | Kyverno |
|---------|---------------|---------|
| Policy language | Rego (complex, powerful) | YAML (Kubernetes-native) |
| Learning curve | Steep | Easy |
| Mutation support | Limited (newer versions) | Full native support |
| Generation | No | Yes (can generate resources) |
| Background scanning | Yes | Yes |
| Audit | Yes | Yes |
| Performance | Higher memory usage | Lower footprint |

**Choose Gatekeeper when:**
- Complex cross-resource policies
- Team already knows Rego
- Need fine-grained logic (e.g., math operations)

**Choose Kyverno when:**
- Team prefers YAML over learning Rego
- Need mutation + generation (e.g., auto-add labels, create NetworkPolicies)
- Simpler policy requirements

---

## Version Upgrade Strategy

**Q11: Explain the Kubernetes version skew policy. What components must be upgraded in what order?**

**A:**

**Version Skew Policy:**
- `kube-apiserver`: Defines the cluster version
- `kubelet`: Can be up to 3 minor versions BEHIND apiserver (e.g., apiserver 1.28, kubelet 1.25-1.28)
- `kube-controller-manager` & `kube-scheduler`: Can be 1 minor version behind apiserver
- `kubectl`: Can be ±1 minor version from apiserver

**Upgrade Order (CRITICAL):**
```
1. etcd (if not managed)
2. kube-apiserver (control plane first)
3. kube-controller-manager
4. kube-scheduler
5. cloud-controller-manager
6. kubelet (node by node)
7. kube-proxy
```

**Tricky rules:**
- NEVER skip minor versions (1.26 → 1.28 is NOT allowed, must go 1.26 → 1.27 → 1.28)
- Upgrade control plane nodes BEFORE worker nodes
- In HA setup: upgrade one API server at a time
- Test with `kubectl convert` for deprecated API versions

---

**Q12: You need to upgrade a production cluster from 1.26 to 1.28. Describe the complete process.**

**A:**

**Phase 1: Preparation**
```bash
# 1. Check deprecated APIs
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
# Or use: kubectl deprecations (pluto tool)
pluto detect-all-in-cluster

# 2. Review changelogs for breaking changes
# 3. Test upgrade in staging cluster first
# 4. Backup etcd
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db

# 5. Verify PodDisruptionBudgets exist for critical workloads
kubectl get pdb --all-namespaces
```

**Phase 2: Upgrade 1.26 → 1.27**
```bash
# Control Plane (one node at a time in HA)
# On control plane node:
apt update && apt install -y kubeadm=1.27.x-*
kubeadm upgrade plan
kubeadm upgrade apply v1.27.x

# Upgrade kubelet on control plane
apt install -y kubelet=1.27.x-* kubectl=1.27.x-*
systemctl daemon-reload && systemctl restart kubelet

# Worker Nodes (one at a time - rolling)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# On node-1:
apt install -y kubeadm=1.27.x-* kubelet=1.27.x-*
kubeadm upgrade node
systemctl daemon-reload && systemctl restart kubelet
# Back on control plane:
kubectl uncordon node-1
# Verify node is Ready before proceeding to next node
```

**Phase 3: Repeat for 1.27 → 1.28**

**Phase 4: Post-upgrade validation**
```bash
kubectl get nodes  # All nodes on new version
kubectl get pods --all-namespaces  # No CrashLooping
kubectl cluster-info
# Run e2e tests
# Verify monitoring/alerting still works
```

---

**Q13: During a cluster upgrade, pods are getting evicted and causing downtime. What's wrong?**

**A:** Common causes:

1. **No PodDisruptionBudget (PDB)**: `kubectl drain` evicts ALL pods without limits
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2  # OR maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

2. **Single replica deployments**: If replica=1 and no PDB, drain evicts the only pod → downtime

3. **Pod using local storage without `--delete-emptydir-data`**: Drain hangs waiting

4. **DaemonSet pods blocking drain**: Use `--ignore-daemonsets`

5. **PDB too strict**: `minAvailable` equals total replicas → drain can never proceed

**Best practices for zero-downtime upgrades:**
- Minimum 2+ replicas for all production workloads
- PDB with `maxUnavailable: 1` or `minAvailable: N-1`
- Preemptive pod anti-affinity (spread across nodes)
- Graceful shutdown handling (preStop hooks + SIGTERM handling)
- Rolling upgrade strategy with `maxSurge` and `maxUnavailable`

---

## Tricky Scenario Questions

**Q14: You discover that a pod in production is running as root with all capabilities. It's a third-party image you can't modify. How do you secure it?**

**A:** Defense-in-depth approach without modifying the image:

```yaml
apiVersion: v1
kind: Pod
spec:
  # 1. Network isolation
  # Apply strict NetworkPolicy (separate manifest)
  
  # 2. Limit capabilities even if running as root
  containers:
  - name: third-party
    image: vendor/app:latest
    securityContext:
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]  # Only what's needed
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      seccompProfile:
        type: RuntimeDefault
    resources:
      limits:
        cpu: "1"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "256Mi"
    # 3. Read-only volumes where possible
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-run
      mountPath: /var/run
  
  volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 100Mi
  - name: var-run
    emptyDir:
      sizeLimit: 50Mi
```

**Additional measures:**
- Run in a dedicated namespace with strict NetworkPolicies
- Use RuntimeClass with gVisor/Kata containers for sandboxing
- Admission controller to enforce limits even on "privileged" workloads
- Falco for runtime threat detection
- Regular image scanning (Trivy, Snyk)

---

**Q15: How do you detect and respond to a compromised pod in a running cluster?**

**A:**

**Detection (Runtime Security):**
1. **Falco**: Detects anomalous syscalls, file access, network connections
2. **Audit Logs**: API server shows unusual RBAC usage
3. **Network monitoring**: Unexpected egress traffic (crypto mining, data exfiltration)

**Immediate Response:**
```bash
# 1. Isolate the pod via NetworkPolicy (deny all)
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-compromised
  namespace: target-ns
spec:
  podSelector:
    matchLabels:
      app: compromised-app
  policyTypes: ["Ingress", "Egress"]
  # Empty ingress/egress = deny all
EOF

# 2. Capture evidence before killing
kubectl exec compromised-pod -- ps aux > /evidence/processes.txt
kubectl exec compromised-pod -- netstat -tlnp > /evidence/network.txt
kubectl logs compromised-pod > /evidence/logs.txt

# 3. Cordon the node (prevent new scheduling)
kubectl cordon affected-node

# 4. Kill the pod
kubectl delete pod compromised-pod --grace-period=0 --force

# 5. Rotate secrets that the pod had access to
kubectl delete secret db-credentials -n target-ns
# Recreate with new values

# 6. Check for lateral movement
kubectl get pods --all-namespaces -o wide | grep affected-node
```

**Post-incident:**
- Review audit logs for API access from the compromised service account
- Check if the attacker escalated privileges
- Scan all images in the cluster
- Rotate ALL credentials the pod could access
- Root cause analysis: How did they get in?

---

**Q16: Explain the difference between `kubectl auth can-i` and actual runtime permissions. Can they diverge?**

**A:** Yes, they can diverge!

`kubectl auth can-i` checks RBAC rules only. But actual access depends on:

1. **Admission Controllers**: May deny even if RBAC allows
   - OPA/Gatekeeper can block resource creation
   - PodSecurity admission can reject pods
   - ResourceQuota can deny due to limits

2. **Network Policies**: RBAC allows pod communication, but NetworkPolicy blocks it

3. **Aggregated ClusterRoles**: `can-i` may not show permissions from aggregated roles correctly in all scenarios

4. **Impersonation**: Admin impersonating a user sees different permissions

```bash
# Check what a specific user can do
kubectl auth can-i --list --as=developer1 -n production

# Check specific permission
kubectl auth can-i create deployments -n production --as=developer1

# Check service account permissions
kubectl auth can-i get secrets --as=system:serviceaccount:default:my-sa

# BUT this won't show admission controller denials!
```

**Tricky**: A user might have RBAC permission to create a pod with `hostNetwork: true`, but PodSecurity admission will REJECT it at runtime. `can-i` shows "yes" but the action fails.

---

**Q17: How do you implement zero-trust networking in Kubernetes?**

**A:**

```
Zero Trust Principle: "Never trust, always verify"

1. IDENTITY: Every workload has a cryptographic identity
   ├── Service Mesh (Istio/Linkerd) provides mTLS between all pods
   ├── SPIFFE/SPIRE for workload identity
   └── No trust based on network location

2. NETWORK: Default-deny everything
   ├── NetworkPolicy: deny-all in every namespace
   ├── Allow only explicitly needed communication
   └── Egress policies (restrict what pods can talk to externally)

3. ACCESS: Least privilege everywhere
   ├── RBAC: Minimal permissions per service account
   ├── Secrets: Each pod only accesses its own secrets
   └── API server: Restricted access (private endpoint)

4. VERIFY: Continuous validation
   ├── Admission controllers validate every request
   ├── Runtime security (Falco) monitors behavior
   └── Audit logging tracks all actions
```

**Default-deny NetworkPolicy template:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # Applies to ALL pods
  policyTypes: ["Ingress", "Egress"]
  # No ingress/egress rules = deny everything
```

---

**Q18: What happens to existing connections when you apply a NetworkPolicy? Is it immediate?**

**A:** This is CNI-dependent and a common gotcha:

- **Calico**: Existing connections MAY be dropped (depends on connection tracking)
- **Cilium**: Existing connections are maintained until timeout (uses conntrack)
- **Weave**: Existing connections persist until closed

**Key insight**: NetworkPolicy is enforced by the CNI plugin, not Kubernetes core. Behavior varies!

**Best practice**: 
- Apply NetworkPolicies BEFORE deploying workloads
- If applying to existing workloads, be aware of potential connection drops
- Test in staging with actual traffic patterns
- Use `audit` mode (if available in your service mesh) before enforcing

---

**Q19: Your cluster upgrade failed midway. The API server is on 1.28 but some nodes are still on 1.26. What do you do?**

**A:**

**Assess the situation:**
```bash
kubectl get nodes -o wide  # Check versions
kubectl get pods --all-namespaces | grep -v Running  # Check broken pods
```

**The version skew policy ALLOWS this** (kubelet can be up to 3 minor versions behind since 1.28). So:

1. **If pods are running fine**: Continue the upgrade node by node
2. **If pods are crashing on old nodes**: Check if new API features are being used that old kubelets don't understand

**Recovery options:**
- **Continue forward**: Drain and upgrade remaining nodes one by one
- **Rollback API server** (risky): Only if critical breakage. Must restore etcd backup since schema migrations may have occurred.

**Tricky**: You CANNOT rollback from 1.28 → 1.27 without etcd restore because:
- etcd schema migrations happen during upgrade
- Storage versions of resources may have changed
- New features may have written data in formats old versions can't read

**Prevention**: Always take etcd snapshot before upgrade!

---

**Q20: Explain Kubernetes API deprecation and removal. What's the difference between deprecated and removed?**

**A:**

- **Deprecated**: Still works but will be removed. Usually shows warnings.
- **Removed**: API version is gone. Manifests using it will FAIL.

**Deprecation Policy:**
- GA APIs: Deprecated for minimum 12 months or 3 releases before removal
- Beta APIs: Can be removed after 3 releases of deprecation (since 1.22)
- Alpha APIs: Can be removed without notice

**Real examples that broke clusters:**
| API | Deprecated | Removed | Replacement |
|-----|-----------|---------|-------------|
| extensions/v1beta1 Ingress | 1.14 | 1.22 | networking.k8s.io/v1 |
| policy/v1beta1 PodSecurityPolicy | 1.21 | 1.25 | Pod Security Admission |
| batch/v1beta1 CronJob | 1.21 | 1.25 | batch/v1 |
| autoscaling/v2beta2 HPA | 1.23 | 1.26 | autoscaling/v2 |

**Detection before upgrade:**
```bash
# Use pluto to find deprecated APIs in cluster
pluto detect-all-in-cluster

# Check in Helm releases
pluto detect-helm

# Check YAML files
pluto detect-files -d ./manifests/

# API server metrics
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis
```
