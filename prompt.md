# Kubernetes Upgrade Feasibility, Compatibility, and Risk Assessment

## Objective

You are acting as a Senior Kubernetes Platform Engineer performing a comprehensive upgrade readiness review.

Your task is to determine whether a Kubernetes cluster can be safely upgraded from:

```text
SOURCE_VERSION=<SOURCE_VERSION>
TARGET_VERSION=<TARGET_VERSION>
```

The analysis must be exhaustive, evidence-based, and conservative.

Your primary goal is to identify:

* Anything that can break during or after the upgrade
* Unsupported APIs
* Controller incompatibilities
* CRD incompatibilities
* Admission webhook risks
* Operator risks
* Runtime risks
* Networking risks
* Storage risks
* Security enforcement changes
* Workload restart risks
* Node upgrade risks
* Control plane risks

Do not assume compatibility unless verified.

If compatibility cannot be confirmed, classify it as a risk.

---

# Assessment Requirements

The assessment must combine:

1. Kubernetes release note analysis
2. Cluster state analysis
3. CRD analysis
4. Controller/operator analysis
5. API compatibility analysis
6. Add-on compatibility analysis
7. Runtime analysis
8. API deprecation scanning (kubent — cluster + Helm history, pluto — Helm releases + manifests)
9. Upgrade simulation
10. Failure scenario modeling

---

# Step 1 - Gather Cluster Information

Collect and analyze:

```bash
kubectl version
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
kubectl api-resources
kubectl get apiservices
```

### API Deprecation Scan (Read-Only)

These tools detect deprecated/removed APIs that `kubectl` alone cannot surface. `kubectl` silently converts old API versions to current ones, so a `kubectl get` may return `apps/v1` even when the object was originally created with `extensions/v1beta1`. These scanners inspect the stored manifest history (Helm release secrets, last-applied-config annotations) to find the true original API versions.

```bash
# kubent — scans live cluster resources + Helm v3 history (Secrets/ConfigMaps)
kubent -o json

# pluto — scans Helm releases (v2 + v3) running in the cluster
pluto detect-helm -o json --target-versions k8s=<TARGET_VERSION>

# Optional: scan static manifest files in your infrastructure-as-code repo
# pluto detect-files -d <path-to-manifests-directory>
```

Capture:

* Kubernetes version
* Node versions
* OS versions
* Container runtime versions
* Managed vs self-managed cluster
* HA topology
* API server version
* kubelet versions

---

# Step 2 - Inventory All Resources

Collect:

```bash
kubectl get all -A
kubectl get deploy -A
kubectl get sts -A
kubectl get ds -A
kubectl get jobs -A
kubectl get cronjobs -A
```

For every workload determine:

* Namespace
* Kind
* API version
* Age
* Restart history
* Criticality

---

# Step 3 - Discover All Installed CRDs

Collect:

```bash
kubectl get crd
kubectl get crd -o yaml
```

For every CRD capture:

* CRD name
* Group
* Kind
* API versions
* Served versions
* Storage version
* Conversion strategy
* Conversion webhooks
* Validation schemas

Generate a complete CRD inventory.

---

# Step 4 - Identify Controllers and Operators

Detect all installed operators/controllers including but not limited to:

* cert-manager
* ingress-nginx
* external-dns
* cluster-autoscaler
* metrics-server
* kube-prometheus-stack
* prometheus-operator
* argo-cd
* flux
* crossplane
* istio
* linkerd
* gatekeeper
* kyverno
* velero
* karpenter
* aws-load-balancer-controller
* ebs-csi-driver
* efs-csi-driver
* cilium
* calico
* antrea

Also detect:

* Vendor operators
* Custom operators
* Internal controllers

For every controller determine:

* Installed version
* Release version
* Supported Kubernetes versions
* Known incompatibilities

---

# Step 5 - Analyze Kubernetes Release Notes

Review ALL Kubernetes release notes between source and target versions.

Example:

```text
1.25 -> 1.26
1.26 -> 1.27
1.27 -> 1.28
1.28 -> 1.29
```

Do not skip intermediate versions.

Review:

* Changelogs
* Release notes
* Upgrade notes
* Deprecation notices
* Removal notices

---

# Step 6 - API Removal Analysis

Identify all APIs removed between source and target versions.

Examples:

```text
policy/v1beta1
batch/v1beta1
extensions/v1beta1
autoscaling/v2beta2
networking.k8s.io/v1beta1
admissionregistration.k8s.io/v1beta1
```

Scan all cluster resources. **Incorporate the kubent and pluto output from Step 1** — these tools detect deprecated APIs stored in Helm release history and last-applied-configuration annotations that `kubectl` silently converts to current API versions. A resource that appears to use `apps/v1` via `kubectl get` may have been originally deployed as `extensions/v1beta1` and will fail after the API is removed.

For every removed API found:

Report:

```text
Namespace:
Object:
Kind:
Current API:
Removal Version:
Impact:
Required Action:
```

Classify as:

```text
CRITICAL
```

if workload would fail after upgrade.

---

# Step 7 - Deprecated API Analysis

Identify APIs that are:

* Deprecated
* Scheduled for removal
* Behavior changed

Scan all workloads.

Provide:

```text
Namespace
Object
API Version
Risk Level
Migration Required
```

---

# Step 8 - CRD Compatibility Analysis

For every CRD:

Determine:

## API Compatibility

* Is the CRD API version still supported?
* Is the storage version still supported?
* Are served versions valid?

## Schema Compatibility

Check:

* OpenAPI schema changes
* Validation rule changes
* Structural schema requirements

## Conversion Compatibility

Check:

* Conversion webhooks
* Conversion strategies
* Version migration requirements

## Upgrade Impact

Determine:

* Will objects continue to deserialize?
* Will controllers continue to reconcile?
* Will upgrades require CRD migration?

Explicitly state:

```text
Could this CRD break after upgrade?
YES / NO

Reason:
```

---

# Step 9 - Controller and Operator Compatibility

For every controller/operator:

Review:

* Vendor documentation
* Release notes
* Compatibility matrix

Determine:

```text
Installed Version:
Supported Kubernetes Versions:
Target Compatibility:
```

Classify:

```text
PASS
GOOD
WARNING
HIGH RISK
CRITICAL
```

Determine whether controller upgrade is required:

```text
Before Kubernetes upgrade
After Kubernetes upgrade
Optional
```

---

# Step 10 - Admission Webhook Analysis

Collect:

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

Analyze:

* API compatibility
* TLS configuration
* FailurePolicy
* Side effects
* Version support

Determine:

```text
Can webhook failures block workloads?
YES / NO

Can webhook fail after upgrade?
YES / NO

Reason:
```

---

# Step 11 - Networking Compatibility Analysis

Review:

* Ingress controllers
* Service configuration
* DNS
* CoreDNS
* kube-proxy
* NetworkPolicies
* CNI plugins

Validate:

* Supported Kubernetes versions
* Known upgrade issues

Explicitly answer:

```text
Can networking break after upgrade?
YES / NO

Why?
```

---

# Step 12 - Storage Compatibility Analysis

Review:

* StorageClasses
* CSI drivers
* Volume snapshots
* PersistentVolumes
* PersistentVolumeClaims

Validate:

* Driver compatibility
* CSI API support
* Snapshot API support

Explicitly answer:

```text
Can storage become inaccessible?
YES / NO

Why?
```

---

# Step 13 - Security Compatibility Analysis

Review:

* PodSecurityPolicy usage
* Pod Security Admission
* Admission controllers
* RBAC
* Security policies

Identify:

* Privileged workloads
* Host networking
* HostPID
* HostIPC
* HostPath usage

Determine:

```text
Can security changes break workloads?
YES / NO

Why?
```

---

# Step 14 - Runtime Compatibility Analysis

Review node information:

```bash
kubectl get nodes -o yaml
```

Determine:

* Container runtime
* CRI compatibility
* Operating system support
* Kernel support
* kubelet support

Validate against target Kubernetes version.

---

# Step 15 - Resource Pressure Analysis

Collect:

```bash
kubectl top nodes
kubectl top pods -A
```

Review:

* CPU pressure
* Memory pressure
* Disk pressure
* Eviction risks

Determine whether upgrade operations may trigger:

* OOMKills
* Pod evictions
* Scheduling failures

---

# Step 16 - Upgrade Simulation

Perform a logical upgrade simulation.

Evaluate:

## Control Plane

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

## Nodes

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

## APIs

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

## CRDs

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

## Controllers

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

## Networking

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

## Storage

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

## Security

Status:

```text
PASS | GOOD | WARNING | HIGH RISK | CRITICAL
```

Reason:

---

# Step 17 - Failure Scenario Analysis

Explicitly answer all of the following.

## Could workloads fail to start?

YES / NO

Reason:

---

## Could controllers crash?

YES / NO

Reason:

---

## Could CRDs become unreadable?

YES / NO

Reason:

---

## Could CRD controllers stop reconciling?

YES / NO

Reason:

---

## Could admission webhooks block deployments?

YES / NO

Reason:

---

## Could storage become inaccessible?

YES / NO

Reason:

---

## Could networking break?

YES / NO

Reason:

---

## Could node upgrades fail?

YES / NO

Reason:

---

## Could kubelets fail to register?

YES / NO

Reason:

---

## Could the control plane fail?

YES / NO

Reason:

---

# Risk Matrix

Produce the following table.

| Area          | Status | Severity | Explanation |
| ------------- | ------ | -------- | ----------- |
| APIs          |        |          |             |
| CRDs          |        |          |             |
| Controllers   |        |          |             |
| Webhooks      |        |          |             |
| Networking    |        |          |             |
| Storage       |        |          |             |
| Security      |        |          |             |
| Runtime       |        |          |             |
| Nodes         |        |          |             |
| Control Plane |        |          |             |

---

# Readiness Score

Calculate:

```text
Readiness Score: XX/100
```

Scoring:

```text
90-100 = Ready

75-89 = Ready with remediation

50-74 = Significant risk

0-49 = Not recommended
```

---

# Confidence Score

Calculate:

```text
Confidence Score: XX%
```

Factors:

* Completeness of cluster inventory
* Availability of release notes
* Controller compatibility verification
* CRD verification
* Operator verification
* Unknown components

Confidence must never exceed available evidence.

---

# Executive Summary

Generate the following section.

```text
UPGRADE DECISION:
APPROVED / CONDITIONAL / NOT RECOMMENDED

SOURCE VERSION:
<SOURCE_VERSION>

TARGET VERSION:
<TARGET_VERSION>

READINESS SCORE:
XX/100

CONFIDENCE:
XX%

CRITICAL ISSUES:
- Item
- Item

HIGH RISKS:
- Item
- Item

WARNINGS:
- Item
- Item

REQUIRED ACTIONS BEFORE UPGRADE:
1.
2.
3.

RECOMMENDED UPGRADE ORDER:
1.
2.
3.

POST-UPGRADE VALIDATIONS:
1.
2.
3.

FINAL RECOMMENDATION:
<detailed recommendation>
```

---

# Mandatory Rule

If any incompatibility is discovered:

You MUST explicitly state:

```text
WHAT WILL BREAK:
WHEN IT WILL BREAK:
(Control Plane Upgrade / Node Upgrade / Immediately After Upgrade / First Reconciliation / First Deployment)

IMPACT:
(Outage / Partial Outage / Reconciliation Failure / Deployment Failure / Data Risk)

SEVERITY:
(Critical / High / Medium / Low)

REMEDIATION:
```

Never hide uncertainty.

Always separate:

* Verified Issues
* Probable Issues
* Possible Issues
* Unknown Risks

Unknown risks must reduce the final confidence score.

-- Give the final feasibility report in a new MD file: analysis/cluster-upgrade-feasibility-and-risks.md
-- Then read cluster-upgrade-execution-plan.md (the plan template in this repo) and use it to generate a platform-specific execution plan in a new MD file: plans/plan.md
-- Create the analysis/ and plans/ directories if they do not exist.
-- DO NOT overwrite prompt.md or cluster-upgrade-execution-plan.md — these are templates that stay unchanged.

---

# Platform Awareness & Cross-Provider Compatibility (On-Prem / EKS / GKE / AKS)

This prompt is **platform-agnostic by design** and works for all Kubernetes distributions. However, each platform has unique upgrade mechanics, managed add-ons, and provider-specific risks that must be incorporated. Below are the platform-specific enhancements required so the assessment is **production-ready for on-prem, Amazon EKS, Google GKE, and Azure AKS**.

---

## Platform Detection (Run First)

Before executing any step above, determine the platform:

```bash
kubectl get nodes -o json | jq '.items[0].metadata.labels | with_entries(select(.key | startswith("topology.kubernetes.io/") or startswith("kubernetes.io/") or startswith("node.kubernetes.io/")))'
kubectl get nodes -o jsonpath='{.items[0].spec.providerID}'
kubectl version --short 2>/dev/null | grep Server
```

**Classification:**
| Indicator | Platform |
|-----------|----------|
| `providerID` contains `aws://` | Amazon EKS |
| `providerID` contains `gce://` | Google GKE |
| `providerID` contains `azure://` | Azure AKS |
| `providerID` is empty or `kubeadm://` | On-Premises / Self-Managed |
| Node labels `eks.amazonaws.com/` | Amazon EKS |
| Node labels `cloud.google.com/gke-` | Google GKE |
| Node labels `kubernetes.azure.com/` | Azure AKS |

---

## On-Premises (kubeadm / Self-Managed)

### Additional Cluster State Collection

```bash
kubeadm version
kubectl get cm -n kube-system kubeadm-config -o yaml
kubectl get cm -n kube-system kube-proxy -o yaml
kubectl get cm -n kube-system coredns -o yaml
cat /etc/kubernetes/manifests/kube-apiserver.yaml
etcdctl version
crictl --version 2>/dev/null || containerd --version 2>/dev/null || docker version 2>/dev/null
kubectl -n kube-system get cm coredns -o jsonpath='{.data.Corefile}'
```

### On-Prem Specific Risks

| Risk Area | What to Validate |
|-----------|-----------------|
| **etcd** | Backup strategy, data directory size, snapshot freshness. `etcdctl snapshot save` must succeed before upgrade. |
| **kubeadm** | Verify `kubeadm upgrade plan` output. Confirm `kubeadm` version matches target. |
| **Container Runtime** | Validate CRI compatibility with target Kubernetes version (containerd 1.6+ for 1.24+, Docker Engine deprecated past 1.20). |
| **CNI** | Manual CNI upgrade path (Calico, Cilium, Flannel). Each has its own compatibility matrix. |
| **Control Plane HA** | Verify all control plane nodes drainable, load balancer targets healthy, and API server certificates valid through the target timeframe. |
| **OS / Kernel** | Ensure kernel >= 4.15 for cgroups v2 support if upgrading to 1.25+. Verify OS EOL dates. |
| **kube-proxy** | Confirm kube-proxy mode (iptables vs IPVS). IPVS has stricter kernel module requirements. |
| **Certificate Expiry** | `kubeadm certs check-expiration` — ensure certs are valid beyond the upgrade window. |
| **Node Draining** | Manual node-by-node drain required. No managed node group rolling update. Budget enough time. |
| **etcd Defrag** | If etcd is > 2GB, defrag before upgrade to reduce snapshot time. |

### On-Prem Upgrade Order (Mandatory Sequence)

```text
1. etcd backup (snapshot + data directory backup)
2. kubeadm upgrade plan (dry-run)
3. Control plane node 1 (kubeadm upgrade apply)
4. Control plane node 2,3... (kubeadm upgrade node)
5. kubelet upgrade on all control plane nodes
6. kubectl upgrade on all control plane nodes
7. Drain and upgrade worker node 1 (kubeadm upgrade node + kubelet + kubectl)
8. Uncordon and validate
9. Repeat drain-upgrade-uncordon for each worker node
10. Force kube-proxy and CoreDNS upgrade if kubeadm did not
11. CNI upgrade (if required)
```

---

## Amazon EKS

### Additional Cluster State Collection

```bash
eksctl get cluster --name <cluster-name> -o yaml 2>/dev/null || aws eks describe-cluster --name <cluster-name>
aws eks list-addons --cluster-name <cluster-name>
aws eks describe-addon --cluster-name <cluster-name> --addon-name vpc-cni
aws eks describe-addon --cluster-name <cluster-name> --addon-name coredns
aws eks describe-addon --cluster-name <cluster-name> --addon-name kube-proxy
aws eks describe-addon --cluster-name <cluster-name> --addon-name aws-ebs-csi-driver
kubectl -n kube-system get ds aws-node -o yaml
kubectl -n kube-system get deploy coredns -o yaml
kubectl -n kube-system get ds kube-proxy -o yaml
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <nodegroup>
aws eks list-nodegroups --cluster-name <cluster-name>
```

### EKS-Specific Risks

| Risk Area | What to Validate |
|-----------|-----------------|
| **EKS Version Skew Policy** | EKS enforces `<major>.<minor>` skew. Control plane and data plane cannot differ beyond 2 minor versions. Same minor version may differ by 1 patch. |
| **EKS Add-ons** | vpc-cni, coredns, kube-proxy, and aws-ebs-csi-driver have EKS-specific version compatibility matrices. These do NOT auto-upgrade with the control plane — they must be upgraded separately. |
| **vpc-cni** | Versions 1.11+ contain breaking changes. ENI trunking, custom networking, and prefix delegation modes are version-sensitive. |
| **coredns** | EKS maintains a patched CoreDNS. Standard upstream compatibility rules do not apply. |
| **kube-proxy** | EKS kube-proxy add-on is version-locked to the control plane. |
| **Managed Node Groups** | Node groups upgrade via launch template versioning. In-place upgrade (AMI update) vs blue/green (new ASG). Max unavailable / max surge config matters. |
| **Self-Managed Node Groups** | Nodes are NOT automatically upgraded. You must cordon/drain/terminate manually or use a node termination handler. |
| **Fargate** | Fargate pods restart automatically on control plane upgrade. No drain needed. Verify Fargate profile selectors still match after upgrade. |
| **IAM Roles for Service Accounts (IRSA)** | No breaking changes expected, but IAM OIDC provider must remain valid. |
| **AWS Load Balancer Controller** | Version must be compatible with target EKS version. Upgrading the controller BEFORE the cluster is often required. |
| **EBS/EFS CSI Driver** | CSI API removals may require driver upgrade. EBS CSI migration (in-tree to CSI) is version-bound. |
| **Secrets Encryption** | If using KMS envelope encryption, verify KMS key policy and rotation. |
| **Cluster Security Group** | Security group rules are not modified by EKS upgrade, but validate no rules reference deprecated protocols. |
| **Upgrade Time Budget** | EKS control plane upgrade: ~10-15 min per step. Data plane (managed node groups): ~20-40 min per node group (depends on pod count). |

### EKS Upgrade Order (Mandatory Sequence)

```text
1. Review EKS Kubernetes versions FAQ and upgrade notes for each intermediate version
2. Upgrade all EKS add-ons to versions compatible with TARGET (do this FIRST)
   - vpc-cni
   - coredns
   - kube-proxy
   - aws-ebs-csi-driver
3. Upgrade third-party controllers (ALB controller, cluster-autoscaler, metrics-server, etc.)
4. Upgrade CRDs for all operators to target-compatible versions
5. EKS control plane upgrade (one minor version at a time, no skipping)
6. Wait for control plane to report Active
7. Upgrade managed node groups (one minor version at a time) OR terminate self-managed nodes
8. Upgrade Fargate profiles (if applicable)
9. Upgrade cluster autoscaler to match new node group AMI
10. Validate all add-ons match target version
11. Run post-upgrade validations
```

---

## Google GKE

### Additional Cluster State Collection

```bash
gcloud container clusters describe <cluster-name> --zone <zone> --format yaml
gcloud container node-pools list --cluster <cluster-name> --zone <zone>
gcloud container operations list --cluster <cluster-name> --zone <zone> --filter="operationType=UPGRADE_CLUSTER"
kubectl -n kube-system get ds konnectivity-agent -o yaml 2>/dev/null
kubectl -n kube-system get ds gke-metrics-agent -o yaml 2>/dev/null
kubectl -n kube-system get deploy event-exporter-gke -o yaml 2>/dev/null
kubectl -n kube-system get cm cluster-autoscaler-status -o yaml 2>/dev/null
kubectl -n kube-system get deploy kube-dns -o yaml
kubectl -n kube-system get deploy kube-dns-autoscaler -o yaml
gcloud container get-server-config --zone <zone>
```

### GKE-Specific Risks

| Risk Area | What to Validate |
|-----------|-----------------|
| **Release Channels** | Static, Rapid, Regular, Stable channels gate which versions are available. A target version may be unavailable in your channel. Check with `gcloud container get-server-config`. |
| **Auto-Upgrade** | If node auto-upgrade is enabled, GKE may upgrade nodes on its own schedule. Coordinate manual upgrade window to avoid race conditions. |
| **Maintenance Windows & Exclusions** | GKE respects maintenance windows. Upgrades outside the window may be rejected. Check for active exclusions. |
| **Node Pools** | Each node pool upgrades independently. Node pool upgrade strategy (SURGE vs RECREATE) determines pod disruption. SURGE creates new nodes before deleting old ones (cost: extra capacity). RECREATE deletes then creates (cost: downtime). |
| **GKE Dataplane v2** | If eBPF-based GKE Dataplane is enabled, additional compatibility checks with CNI-aware controllers apply. |
| **GKE Managed Add-ons** | GKE manages kube-dns, kube-dns-autoscaler, metrics-server, konnectivity-agent, and GMP components. These upgrade automatically with the control plane. Do NOT manually override. |
| **Workload Identity** | GKE Workload Identity (GKE WI) requires the GKE metadata server. Version compatibility is maintained by GKE but validate WI-enabled pods post-upgrade. |
| **GKE Sandbox (gVisor)** | gVisor runtime has separate version compatibility. Check if sandboxed node pools need separate upgrades. |
| **Confidential Computing Nodes** | If using Confidential GKE nodes, verify target version supports Confidential Compute. |
| **Binary Authorization** | Policy enforcement via Binary Authorization is version-independent but validate post-upgrade. |
| **GKE Backup for GKE** | If using Backup for GKE, verify backup plan continues to function with target version. |
| **Control Plane IP** | If using authorized networks, control plane public endpoint IP may change during upgrade. |
| **Upgrade Time Budget** | GKE control plane: ~15-20 min. Node pools (SURGE): ~10-15 min per node in pool. |

### GKE Upgrade Order (Mandatory Sequence)

```text
1. gcloud container get-server-config --zone=<zone> (confirm target version available in channel)
2. Review GKE release notes for each intermediate version
3. Validate maintenance window or schedule manual upgrade window
4. Disable node auto-upgrade temporarily (if coordinating manual upgrade)
5. Upgrade all third-party controllers/operators/CRDs to target-compatible versions FIRST
6. GKE control plane upgrade (gcloud container clusters upgrade) — one minor version at a time
7. Verify control plane upgrade completed (gcloud container operations list)
8. Upgrade node pools one at a time (gcloud container clusters upgrade with --node-pool)
   - STRATEGY: SURGE if zero-downtime required, RECREATE if cost-sensitive
   - Order: non-critical pools before critical pools
9. If using cluster autoscaler, restart it after node pool upgrades
10. Re-enable node auto-upgrade if it was disabled
11. Run post-upgrade validations
```

---

## Azure AKS

### Additional Cluster State Collection

```bash
az aks show --resource-group <rg> --name <cluster> -o yaml
az aks get-upgrades --resource-group <rg> --name <cluster> -o table
az aks nodepool list --resource-group <rg> --cluster-name <cluster> -o table
az aks nodepool show --resource-group <rg> --cluster-name <cluster> --name <nodepool>
kubectl -n kube-system get ds azure-cni -o yaml 2>/dev/null
kubectl -n kube-system get ds azure-cni-networkmonitor -o yaml 2>/dev/null
kubectl -n kube-system get deploy coredns -o yaml
kubectl -n kube-system get deploy coredns-autoscaler -o yaml 2>/dev/null
kubectl -n kube-system get deploy metrics-server -o yaml
kubectl -n kube-system get deploy konnectivity-agent -o yaml 2>/dev/null
kubectl -n kube-system get ds csi-azuredisk-node -o yaml 2>/dev/null
kubectl -n kube-system get ds csi-azurefile-node -o yaml 2>/dev/null
az aks show --resource-group <rg> --name <cluster> --query "autoUpgradeProfile" -o yaml
```

### AKS-Specific Risks

| Risk Area | What to Validate |
|-----------|-----------------|
| **AKS Version Availability** | AKS does not offer all Kubernetes patch versions. Use `az aks get-upgrades` to see available targets. |
| **Planned Maintenance** | AKS planned maintenance windows can override manual upgrade schedules. Check for conflicts. |
| **Auto-Upgrade Channels** | `patch`, `stable`, `rapid`, `node-image` channels. If set, AKS may upgrade automatically. Disable or coordinate. |
| **Max Surge** | AKS node pool upgrade uses `max-surge` to provision extra nodes before cordon/drain. Verify you have capacity headroom. |
| **Node Image** | AKS nodes use a specific image version (AKS Ubuntu / Azure Linux). This is upgraded independently of the control plane. Check `--node-image-only` upgrades. |
| **Managed AKS Add-ons** | Some add-ons (monitoring, HTTP application routing, Azure Policy) are AKS-managed and upgrade with the cluster. Do NOT override. |
| **Azure CNI** | Azure CNI v1 (per-pod IP from subnet) vs Azure CNI v2 (Cilium-based overlay) have different upgrade paths. Subnet IP exhaustion can block node creation during surge upgrades. |
| **Azure Disk/File CSI** | Managed by AKS. Verify CSI driver versions match target control plane. In-tree to CSI migration is automatic but verify post-upgrade. |
| **Pod Identity / Workload Identity** | AAD Pod Identity is deprecated for Workload Identity. If still using AAD Pod Identity, migration may be forced at certain versions. |
| **Azure Policy Add-on** | Gatekeeper-based Azure Policy may block workloads if constraint templates are version-incompatible. |
| **Private Cluster** | Private AKS clusters require careful DNS/firewall validation. Control plane FQDN may change IP during upgrade. |
| **Azure RBAC** | If Azure AD integration with Kubernetes RBAC, validate group membership resolves post-upgrade. |
| **Multiple Node Pools** | AKS system node pools and user node pools can use different upgrade schedules. System pools should be upgraded BEFORE user pools. |
| **Spot Node Pools** | Spot/transient node pools have different eviction behavior during upgrades. Validate eviction policies. |
| **Upgrade Time Budget** | AKS control plane: ~10-15 min. Node pools (max-surge): ~15-25 min per pool. |

### AKS Upgrade Order (Mandatory Sequence)

```text
1. az aks get-upgrades --resource-group <rg> --name <cluster> (confirm target available)
2. Review AKS release notes and known issues for each intermediate version
3. Validate planned maintenance windows do not conflict
4. Validate subnet IP capacity for surge nodes (Azure CNI v1)
5. Upgrade all third-party controllers/operators/CRDs to target-compatible versions FIRST
6. AKS control plane upgrade (az aks upgrade) — one minor version at a time
7. Verify control plane upgrade completed
8. Upgrade system node pools FIRST (az aks nodepool upgrade)
9. Upgrade user node pools one at a time
   - Order: non-critical pools before critical
   - Max surge must be configured to avoid capacity issues
10. Verify node image updated (az aks nodepool show --query "nodeImageVersion")
11. Run post-upgrade validations
```

---

## Cross-Platform Universal Upgrade Checklist

Regardless of platform, these steps apply:

```text
BEFORE UPGRADE:
  [ ] Full etcd snapshot / managed cluster backup
  [ ] All controllers upgraded to target-compatible versions
  [ ] All CRDs upgraded to target-compatible versions
  [ ] Deprecated API usage remediated (kubent, pluto, or kubepug scan)
  [ ] All PodDisruptionBudgets validated (ensure they won't block drains)
  [ ] Cluster auto-scaler paused (to prevent scaling during upgrade)
  [ ] Alerting silenced or acknowledged for the maintenance window
  [ ] Runbook updated with rollback procedure
  [ ] Critical application health check endpoint reviewed
  [ ] Node cordon/drain procedure documented for operator

DURING UPGRADE:
  [ ] Monitor control plane status (platform-specific commands)
  [ ] Monitor node pool upgrade progress
  [ ] Watch pod eviction success (kubectl get events -A)
  [ ] Validate new nodes register as Ready
  [ ] Check for CrashLoopBackOff on newly scheduled pods
  [ ] Verify CoreDNS/kube-dns endpoint resolution

AFTER UPGRADE:
  [ ] kubectl version (verify client/server match expectations)
  [ ] kubectl get nodes (all Ready, correct version)
  [ ] kubectl get pods -A (all Running/Completed, zero errors)
  [ ] kubectl get events -A (no unexpected warnings)
  [ ] CoreDNS functional test (nslookup from test pod)
  [ ] Ingress controller health check
  [ ] Storage validation (create/delete test PVC)
  [ ] Admission webhook validation (test deployment)
  [ ] CRD controller reconciliation test
  [ ] Application smoke tests (critical business flows)
  [ ] Monitoring/alerting re-enabled
  [ ] Cluster auto-scaler unpaused
  [ ] Node auto-upgrade re-enabled (if applicable)
```

---

## Platform-Specific Rollback Procedures

### On-Prem Rollback
- Restore etcd snapshot on all control plane nodes
- Re-run `kubeadm init` with previous version if control plane corrupted
- Revert kubelet binary and restart on all nodes

### EKS Rollback
- EKS **does not support control plane downgrade** (you can only go forward)
- For data plane: roll back managed node group launch template to previous AMI
- Mitigation: test upgrades in a non-production EKS cluster first

### GKE Rollback
- GKE **does not support control plane downgrade**
- Node pools can be rolled back (create new pool at old version, migrate workloads, delete upgraded pool)
- Mitigation: leverage GKE release channel staging clusters

### AKS Rollback
- AKS **does not support control plane downgrade**
- Node pools can be rolled back via `az aks nodepool upgrade` to a previous version if still available
- Mitigation: use AKS cluster snapshot/restore or blue/green cluster patterns

---

## Additional Platform-Aware Commands Required in Step 1

Add the following to Step 1 "Gather Cluster Information" based on detected platform:

```bash
# UNIVERSAL
kubectl get leases -A
kubectl get events -A --sort-by='.lastTimestamp' | tail -50
kubectl get componentstatuses 2>/dev/null || echo "ComponentStatus deprecated"

# EKS
aws eks describe-cluster --name <cluster> --query "cluster.{Version:version,Status:status,Endpoint:endpoint,PlatformVersion:platformVersion}"
aws eks list-addons --cluster-name <cluster>
aws eks list-updates --cluster-name <cluster> --query "updateIds[?status!='Successful']"

# GKE
gcloud container clusters describe <cluster> --zone <zone> --format "yaml(currentMasterVersion,currentNodeVersion,status,releaseChannel,autopilot)"
gcloud container node-pools list --cluster <cluster> --zone <zone> --format "table(name,version,status,config.machineType)"

# AKS
az aks show --resource-group <rg> --name <cluster> --query "{version:kubernetesVersion,status:powerState.code,provisioningState:provisioningState,autoUpgrade:autoUpgradeProfile.upgradeChannel}"
az aks nodepool list --resource-group <rg> --cluster-name <cluster> --query "[].{name:name,version:orchestratorVersion,status:powerState.code,provisioningState:provisioningState}"
```

---

## Final Platform-Aware Assessment Section

Add this to the Executive Summary:

```text
PLATFORM:
<ON-PREM | EKS | GKE | AKS | OTHER>

PLATFORM-SPECIFIC NOTES:
- <notes>

UPGRADE STRATEGY:
<On-Prem: kubeadm sequential>
<EKS: Control plane -> Add-ons -> Node groups>
<GKE: Control plane -> Node pools (SURGE/RECREATE)>
<AKS: Control plane -> System node pools -> User node pools>

MAXIMUM VERSION SKIP:
<On-Prem: 1 minor version at a time (mandatory)>
<EKS: 1 minor version at a time (enforced by EKS)>
<GKE: 1 minor version at a time (enforced by GKE)>
<AKS: 1 minor version at a time (enforced by AKS)>

UPGRADE WINDOW ESTIMATE:
<HH:MM based on node count x per-node-upgrade-time>

ROLLBACK STRATEGY:
<platform-specific rollback or mitigation>

PLATFORM-SPECIFIC CRITICAL WARNINGS:
- <any platform-specific warnings discovered>
```

---

# Output Files

This assessment produces **two separate markdown files** inside dedicated output directories (neither overwrites the templates in this repo):

1. **analysis/cluster-upgrade-feasibility-and-risks.md** — the complete feasibility assessment and risk report as defined above.

2. **plans/plan.md** — a **platform-specific** step-by-step upgrade execution plan. This plan is generated by reading `cluster-upgrade-execution-plan.md` (the plan template in this repository), detecting the platform from the assessment, and producing ONLY the relevant plan sections for that platform. The execution plan is **lightweight for managed clusters** (EKS/GKE/AKS use provider CLI commands with a few steps) and **heavy for on-prem** (kubeadm manual drain/SSH/etcd backup/kubelet upgrade per node). Read the template file first, then populate it with all findings from this assessment — no finding should be silently dropped.

```text
OUTPUT DIRECTORIES (gitignored — never committed):
  analysis/  → cluster-upgrade-feasibility-and-risks.md
  plans/     → plan.md

FILE 1: analysis/cluster-upgrade-feasibility-and-risks.md
CONTENTS: Full 17-step assessment with Risk Matrix, Readiness Score, Confidence Score, and Executive Summary.

FILE 2: plans/plan.md
CONTENTS: Platform-specific execution plan generated from the template in this repo.
          Only includes sections applicable to the detected platform.
          On-Prem ≈ heavy, EKS/GKE/AKS ≈ light.

BOTH FILES ARE NEW — prompt.md and cluster-upgrade-execution-plan.md (the templates) remain unchanged.
```
