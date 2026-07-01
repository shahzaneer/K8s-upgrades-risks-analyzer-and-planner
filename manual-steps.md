# Kubernetes Upgrade — Manual Assessment & Execution Plan Checklist

> **Audience:** Junior-to-mid DevOps/SRE engineers performing a cluster upgrade readiness review **by hand**, without an AI assistant.
>
> **Purpose:** This checklist mirrors every step in `prompt.md`. Fill it out manually, then use your findings to write `plans/plan.md` yourself. Afterward, run `prompt.md` with AI and compare results to catch gaps neither you nor the AI caught alone.

---

## Section 0 — Prerequisites

| Tool | Minimum Version | Installation Check |
|------|----------------|-------------------|
| `kubectl` | ≥ 1.25 | `kubectl version --client` |
| `jq` | ≥ 1.6 | `jq --version` |
| AWS CLI (`aws`) | v2 | `aws --version` (EKS only) |
| Google Cloud SDK (`gcloud`) | latest | `gcloud version` (GKE only) |
| Azure CLI (`az`) | latest | `az version` (AKS only) |
| `kubeadm` | — | `kubeadm version` (on-prem only) |
| `kubent` | ≥ 0.7 | `kubent --version` (recommended) |
| `pluto` | ≥ 5.0 | `pluto version` (recommended) |

**Access checklist before you begin:**

```
[ ] kubectl configured and can reach the cluster (kubectl cluster-info)
[ ] Can list nodes (kubectl get nodes → returns Ready nodes)
[ ] Can list all namespaces (kubectl get ns)
[ ] Platform CLI authenticated (aws sts get-caller-identity / gcloud auth list / az account show)
[ ] etcd access (on-prem only: can SSH to control plane nodes)
[ ] Read access to kube-system namespace
[ ] Read access to all application namespaces
```

---

## Section 1 — Platform Detection

| Command | Output (paste) |
|---------|---------------|
| `kubectl get nodes -o jsonpath='{.items[0].spec.providerID}'` | |
| `kubectl version --short 2>/dev/null \| grep Server` | |
| `kubectl get nodes -o jsonpath='{.items[0].metadata.labels}' \| jq '. \| with_entries(select(.key \| startswith("cloud.google.com/gke") or startswith("kubernetes.azure.com") or startswith("eks.amazonaws.com")))'` | |

**Classification:**

| Indicator Found? | Platform |
|------------------|----------|
| `providerID` contains `aws://` | [ ] EKS |
| `providerID` contains `gce://` | [ ] GKE |
| `providerID` contains `azure://` | [ ] AKS |
| `providerID` empty or contains `kubeadm://` | [ ] On-Prem |
| Node label `eks.amazonaws.com/` present | [ ] EKS |
| Node label `cloud.google.com/gke-` present | [ ] GKE |
| Node label `kubernetes.azure.com/` present | [ ] AKS |

> **MY PLATFORM: ________________________**

---

## Section 2 — Gather Cluster Information (`prompt.md` Step 1)

```bash
kubectl version
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
kubectl api-resources
kubectl get apiservices
kubectl get leases -A
kubectl get events -A --sort-by='.lastTimestamp' | tail -50
kubectl get componentstatuses 2>/dev/null || kubectl get --raw /healthz
```

### 2.1 Version Information

| Field | Value |
|-------|-------|
| Client Version | |
| Server Version (Control Plane) | |
| Platform Version (EKS only) | |

### 2.2 Node Inventory

| Node Name | Status | Role | Kubelet Version | Instance Type | OS Image | Kernel | Container Runtime |
|-----------|--------|------|----------------|---------------|----------|--------|-------------------|
| | | | | | | | |
| | | | | | | | |
| | | | | | | | |
| | | | | | | | |

**Total nodes: ___  |  Control plane nodes: ___  |  Worker nodes: ___**

### 2.3 Version Summary

| Component | Current Version | Target Version | Gap |
|-----------|----------------|----------------|-----|
| Control Plane | | | |
| kubelet (lowest) | | | |
| kubelet (highest) | | | |
| Container Runtime | | | |
| CNI Plugin | | | |

### 2.4 Namespace Inventory

| Namespace | Purpose (brief) | Workload Count (est.) |
|-----------|-----------------|----------------------|
| | | |
| | | |

---

### Platform-Specific Commands

**EKS only:**
```bash
aws eks describe-cluster --name <cluster> --query "cluster.{Version:version,Status:status,Endpoint:endpoint,PlatformVersion:platformVersion}"
aws eks list-addons --cluster-name <cluster>
aws eks list-nodegroups --cluster-name <cluster>
aws eks list-updates --cluster-name <cluster> --query "updateIds[?status!='Successful']"
```

**GKE only:**
```bash
gcloud container clusters describe <cluster> --zone <zone> --format "yaml(currentMasterVersion,currentNodeVersion,status,releaseChannel,autopilot)"
gcloud container node-pools list --cluster <cluster> --zone <zone> --format "table(name,version,status,config.machineType)"
gcloud container get-server-config --zone <zone>
```

**AKS only:**
```bash
az aks show --resource-group <rg> --name <cluster> --query "{version:kubernetesVersion,status:powerState.code,provisioningState:provisioningState,autoUpgrade:autoUpgradeProfile.upgradeChannel}"
az aks nodepool list --resource-group <rg> --cluster-name <cluster> -o table
az aks get-upgrades --resource-group <rg> --name <cluster> -o table
```

**On-Prem only:**
```bash
kubeadm version
kubectl get cm -n kube-system kubeadm-config -o yaml
kubectl get cm -n kube-system kube-proxy -o yaml
kubectl get cm -n kube-system coredns -o yaml
crictl --version 2>/dev/null || containerd --version 2>/dev/null || docker version 2>/dev/null
kubeadm certs check-expiration
```

---

## Section 3 — Inventory All Resources (`prompt.md` Step 2)

```bash
kubectl get all -A
kubectl get deploy -A
kubectl get sts -A
kubectl get ds -A
kubectl get jobs -A
kubectl get cronjobs -A
```

### 3.1 Deployment Inventory

| Namespace | Deployment Name | Ready | API Version | Age | Critical? (Y/N) |
|-----------|----------------|-------|-------------|-----|-----------------|
| | | | | | |
| | | | | | |

### 3.2 StatefulSet Inventory

| Namespace | StatefulSet Name | Ready | API Version | Age | Critical? |
|-----------|-----------------|-------|-------------|-----|-----------|
| | | | | | |

### 3.3 DaemonSet Inventory

| Namespace | DaemonSet Name | Desired/Ready | API Version | Purpose |
|-----------|---------------|---------------|-------------|---------|
| | | | | |

**Pod Health Summary:**

```
Running pods: ___
Non-Running pods:
  CrashLoopBackOff: ___
  Pending: ___
  Error: ___
  ImagePullBackOff: ___
  Other: ___
```

---

## Section 4 — Discover All CRDs (`prompt.md` Step 3)

```bash
kubectl get crd
kubectl get crd -o yaml > /tmp/all-crds.yaml
```

### 4.1 CRD Inventory

| CRD Name | Group | Kind | Served Versions | Storage Version | Conversion Strategy | Conversion Webhook? |
|----------|-------|------|-----------------|-----------------|---------------------|---------------------|
| | | | | | | |
| | | | | | | |

**Total CRDs: ___**

---

## Section 5 — Identify Controllers & Operators (`prompt.md` Step 4)

```bash
# Helm releases (if using Helm)
helm ls -A 2>/dev/null

# Look for common controllers
kubectl get deploy -A | grep -iE "cert-manager|ingress|external-dns|autoscaler|metrics|prometheus|argo|flux|crossplane|istio|linkerd|gatekeeper|kyverno|velero|karpenter|load-balancer|ebs|efs|cilium|calico"

# Look for operator CRDs
kubectl get crd | grep -v "k8s.io$"
```

### 5.1 Controller/Operator Inventory

| Name | Namespace | Installed Version | Image | Installed Via | Notes |
|------|-----------|------------------|-------|--------------|-------|
| | | | | | |
| | | | | | |

### 5.2 Compatibility Verification

For each controller above, look up the vendor's compatibility matrix and fill:

| Controller | Installed Ver | Supports Target K8s? (Y/N) | Min Version for Target | Source (URL) |
|------------|--------------|---------------------------|----------------------|--------------|
| | | | | |
| | | | | |

---

## Section 6 — Kubernetes Release Notes Review (`prompt.md` Step 5)

### 6.1 Version Path

```
SOURCE: ___
TARGET: ___
INTERMEDIATE: ___ (list all)
```

### 6.2 Per-Version Notes

| Version Bump | Key Deprecations | Key Removals | Behavior Changes | Anything Affecting Us? |
|-------------|-----------------|--------------|------------------|----------------------|
| | | | | |
| | | | | |

**Sources reviewed:**
```
[ ] https://kubernetes.io/blog/  (release announcements)
[ ] https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/
[ ] Platform-specific release notes (EKS docs / GKE release notes / AKS release notes)
```

---

## Section 7 — API Removal & Deprecation Analysis (`prompt.md` Steps 6 & 7)

### 7.1 Automated Scanner Run (Recommended)

```bash
# kubent — scan live cluster + Helm history for deprecated/removed APIs
kubent -o json > /tmp/kubent-report.json

# pluto — scan Helm releases against target version
pluto detect-helm -o json --target-versions k8s=<TARGET_VERSION> > /tmp/pluto-report.json

# pluto — scan static manifests (if you have an IaC repo)
# pluto detect-files -d <path-to-manifests> -o json --target-versions k8s=<TARGET_VERSION>
```

**kubent findings:**
```
Deprecated: ___
Removed: ___
```

**pluto findings:**
```
Deprecated: ___
Removed: ___
```

### 7.2 Manual API Scan (If Scanner Not Available)

Check each workload's `apiVersion` against the Kubernetes deprecation guide:

| Namespace | Object | Kind | Current API Version | Deprecation/Removal Version | Required Action |
|-----------|--------|------|--------------------|----------------------------|-----------------|
| | | | | | |
| | | | | | |

### 7.3 Known Removed APIs (Checklist)

Mark each API version that was removed between your SOURCE and TARGET:

```
[ ] batch/v1beta1          (CronJob)
[ ] extensions/v1beta1     (Deployment, DaemonSet, Ingress, etc.)
[ ] networking.k8s.io/v1beta1 (Ingress, IngressClass)
[ ] policy/v1beta1         (PodDisruptionBudget, PodSecurityPolicy)
[ ] autoscaling/v2beta1    (HorizontalPodAutoscaler)
[ ] autoscaling/v2beta2    (HorizontalPodAutoscaler)
[ ] rbac.authorization.k8s.io/v1beta1 (ClusterRole, Role)
[ ] admissionregistration.k8s.io/v1beta1 (Webhooks)
[ ] apiextensions.k8s.io/v1beta1 (CRDs)
[ ] certificates.k8s.io/v1beta1 (CSR)
[ ] coordination.k8s.io/v1beta1 (Lease)
[ ] events.k8s.io/v1beta1  (Event)
[ ] flowcontrol.apiserver.k8s.io/v1beta1 (FlowSchema, PriorityLevelConfiguration)
[ ] node.k8s.io/v1beta1    (RuntimeClass)
[ ] scheduling.k8s.io/v1beta1 (PriorityClass)
[ ] storage.k8s.io/v1beta1 (CSIDriver, CSINode, StorageClass)
[ ] discovery.k8s.io/v1beta1 (EndpointSlice)
```

---

## Section 8 — CRD Compatibility Analysis (`prompt.md` Step 8)

For every CRD listed in Section 4.1, answer:

| CRD Name | API Version Still Supported? | Storage Version OK? | Schema Changes? | Conversion OK? | Could This Break After Upgrade? (YES/NO) | Reason |
|----------|-----------------------------|---------------------|-----------------|----------------|-------------------------------------------|--------|
| | | | | | | |
| | | | | | | |

**CRDs marked YES for breakage:**

| CRD Name | Issue | Required Action |
|----------|-------|-----------------|
| | | |

---

## Section 9 — Controller & Operator Compatibility (`prompt.md` Step 9)

For every controller from Section 5.1:

| Controller | Installed Ver | Compatible with Target? | Classification (PASS/GOOD/WARNING/HIGH RISK/CRITICAL) | Upgrade Timing (Before CP / After CP / Optional) | Min Target Version |
|------------|--------------|------------------------|--------------------------------------------------------|---------------------------------------------------|-------------------|
| | | | | | |
| | | | | | |

### 9.1 Critical Controller Issues

```
CONTROLLER: ________________________
  WHAT WILL BREAK:
  WHEN:
  IMPACT:
  REMEDIATION:
```

(Repeat for each CRITICAL or HIGH RISK controller)

---

## Section 10 — Admission Webhook Analysis (`prompt.md` Step 10)

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
```

### 10.1 Webhook Inventory

| Name | Type (Validating/Mutating) | FailurePolicy (Fail/Ignore) | Namespace Selector | Risk Level |
|------|---------------------------|----------------------------|--------------------|------------|
| | | | | |

### 10.2 Webhook Risk Assessment

```
Can webhook failures block workloads after upgrade?  [ ] YES  [ ] NO
Reason:

Can any webhook fail due to API changes in target version?  [ ] YES  [ ] NO
Reason:

Webhooks requiring action:
  1. ________________________________________________
  2. ________________________________________________
```

---

## Section 11 — Networking Compatibility Analysis (`prompt.md` Step 11)

### 11.1 Networking Component Inventory

| Component | Type | Version | Managed By | Notes |
|-----------|------|---------|------------|-------|
| CNI Plugin | | | | |
| kube-proxy | | | | |
| CoreDNS / kube-dns | | | | |
| Ingress Controller | | | | |
| Service Mesh (if any) | | | | |

### 11.2 NetworkPolicy Summary

Total NetworkPolicies: ___

### 11.3 Networking Risk Assessment

```
Can networking break after upgrade?  [ ] YES  [ ] NO
Why?
```

---

## Section 12 — Storage Compatibility Analysis (`prompt.md` Step 12)

### 12.1 StorageClass Inventory

| Name | Provisioner | Reclaim Policy | Volume Binding Mode | Is Default? |
|------|------------|----------------|---------------------|-------------|
| | | | | |

### 12.2 CSI Driver Inventory

| Driver Name | Version | Managed By | Node Plugin? | Controller Plugin? |
|------------|---------|------------|-------------|-------------------|
| | | | | |

### 12.3 Persistent Volume Summary

| Total PVs | Bound | Released | Failed |
|-----------|-------|----------|--------|
| | | | |

### 12.4 Storage Risk Assessment

```
Can storage become inaccessible after upgrade?  [ ] YES  [ ] NO
Why?
```

---

## Section 13 — Security Compatibility Analysis (`prompt.md` Step 13)

### 13.1 Security Component Inventory

```
PodSecurityPolicy in use?  [ ] YES  [ ] NO (removed in ≥1.25)
Pod Security Admission configured?  [ ] YES  [ ] NO
```

### 13.2 Privileged Workload Scan

```bash
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].securityContext.privileged==true) | "\(.metadata.namespace)/\(.metadata.name)"'
```

| Namespace | Pod | Privileged? | HostNetwork? | HostPID? | HostIPC? |
|-----------|-----|------------|-------------|----------|----------|
| | | | | | |

### 13.3 RBAC Summary

```bash
kubectl get clusterroles | wc -l
kubectl get clusterrolebindings | wc -l
kubectl get roles -A | wc -l
kubectl get rolebindings -A | wc -l
```

### 13.4 Security Risk Assessment

```
Can security changes break workloads?  [ ] YES  [ ] NO
Why?
```

---

## Section 14 — Runtime Compatibility Analysis (`prompt.md` Step 14)

```bash
kubectl get nodes -o yaml
```

### 14.1 Runtime Matrix

| Component | Current Version | Target K8s Requires Min | Compatible? | Action Needed |
|-----------|----------------|------------------------|-------------|---------------|
| Container Runtime | | | | |
| Kernel | | | | |
| OS | | | | |
| kubelet | | | | |

### 14.2 OS EOL Check

| OS | Current Version | EOL Date | Action Needed? |
|----|----------------|----------|----------------|
| | | | |

---

## Section 15 — Resource Pressure Analysis (`prompt.md` Step 15)

```bash
kubectl top nodes
kubectl top pods -A
```

### 15.1 Node Resource Usage

| Node | CPU Cores | CPU% | Memory (Mi) | Memory% | Resource Pressure? |
|------|----------|------|-------------|---------|--------------------|
| | | | | | |
| | | | | | |

### 15.2 Capacity Headroom

```
Total cluster CPU:    ___ cores
Total CPU used:       ___ cores
Total CPU headroom:   ___ cores (___%)

Total cluster Memory: ___ Gi
Total Memory used:    ___ Gi
Total Memory headroom: ___ Gi (___%)

Estimated surge nodes needed during upgrade: ___
Sufficient headroom for surge?  [ ] YES  [ ] NO
```

### 15.3 Resource Pressure Assessment

```
[ ] OOMKill risk during drain/eviction
[ ] Scheduling failure risk
[ ] Disk pressure risk
[ ] PID pressure risk
```

---

## Section 16 — Upgrade Simulation (`prompt.md` Step 16)

For each area, assess and explain your reasoning.

| Area | Status (PASS/GOOD/WARNING/HIGH RISK/CRITICAL) | Reason |
|------|------------------------------------------------|--------|
| Control Plane | | |
| Nodes | | |
| APIs | | |
| CRDs | | |
| Controllers | | |
| Networking | | |
| Storage | | |
| Security | | |

---

## Section 17 — Failure Scenario Analysis (`prompt.md` Step 17)

Answer ALL of the following. If the answer is YES, explain WHAT and HOW.

| Scenario | Could Happen? (YES/NO) | Reason |
|----------|------------------------|--------|
| Could workloads fail to start? | | |
| Could controllers crash? | | |
| Could CRDs become unreadable? | | |
| Could CRD controllers stop reconciling? | | |
| Could admission webhooks block deployments? | | |
| Could storage become inaccessible? | | |
| Could networking break? | | |
| Could node upgrades fail? | | |
| Could kubelets fail to register? | | |
| Could the control plane fail? | | |

---

## Section 18 — Risk Matrix

Populate from all sections above.

| Area | Status | Severity (Low/Medium/High/Critical) | Explanation |
|------|--------|-------------------------------------|-------------|
| APIs | | | |
| CRDs | | | |
| Controllers | | | |
| Webhooks | | | |
| Networking | | | |
| Storage | | | |
| Security | | | |
| Runtime | | | |
| Nodes | | | |
| Control Plane | | | |

---

## Section 19 — Readiness & Confidence Scores

### 19.1 Readiness Score Worksheet

```
Start at 100.
Subtract for each issue found:
  - Critical issue:     -10 each (list): _________________________
  - High risk:          -6 each  (list): _________________________
  - Warning:            -3 each  (list): _________________________
  - Unknown component:  -3 each  (list): _________________________

READINESS SCORE: ___/100
```

### 19.2 Readiness Classification

```
[ ] 90–100 = Ready
[ ] 75–89  = Ready with remediation
[ ] 50–74  = Significant risk
[ ] 0–49   = Not recommended
```

### 19.3 Confidence Score Worksheet

```
Start at 100. Subtract:

  - Incomplete resource inventory:      -10  (if Section 3 has unknowns)
  - Release notes not reviewed:         -15  (if Section 6 is empty)
  - Controller matrix not verified:     -10  (if Section 9 has gaps)
  - CRD schemas not validated:          -10  (if Section 8 is surface-level)
  - Unknown workloads/components:        -5  per unknown (list):
  - API deprecation scan not run:       -15  (if kubent/pluto were skipped)

CONFIDENCE SCORE: ___%
```

---

## Section 20 — Mandatory Finding Detail Template

For every CRITICAL or HIGH finding discovered above, fill one of these:

```
FINDING: ________________________________________________

WHAT WILL BREAK:
________________________________________________

WHEN IT WILL BREAK:
(Control Plane Upgrade / Node Upgrade / Immediately After Upgrade / First Reconciliation / First Deployment)

IMPACT:
(Outage / Partial Outage / Reconciliation Failure / Deployment Failure / Data Risk)

SEVERITY:
(Critical / High / Medium / Low)

REMEDIATION:
1. ________________________________________________
2. ________________________________________________
3. ________________________________________________
```

(Copy and repeat for each finding.)

---

## Section 21 — Write Your Own Execution Plan

Now that you have filled out all sections above, use these findings to manually write `plans/plan.md`.

### 21.1 Plan Template

Open a new file at `plans/plan.md` and fill in:

```markdown
# Upgrade Execution Plan: <CLUSTER-NAME>

PLATFORM: <your platform from Section 1>
SOURCE: <current version>
TARGET: <target version>
DECISION: <APPROVED / CONDITIONAL / NOT RECOMMENDED>
READINESS: <your score>/100
CONFIDENCE: <your score>%
EXPECTED DURATION: <your estimate>
PLAN GENERATED: <today's date>

---

## Section 1: Pre-Flight Health Check

(Paste the gate commands from this checklist Section 2, verify cluster is healthy.)

## Section 2: Pre-Upgrade Remediation

(For every finding from Sections 7, 8, 9, 10 — write the exact kubectl/helm commands to fix each.)

## Section 3: Backup

(Based on your platform — etcd snapshot for on-prem, manifest export + platform backup for EKS/GKE/AKS.)

## Section 4: Pre-Upgrade Configuration

(Silence alerts, pause autoscaler, validate PDBs, notify stakeholders.)

## Section 5: Control Plane Upgrade

(Paste the platform-specific commands from `cluster-upgrade-execution-plan.md` Section 5.)

## Section 6: Data Plane Upgrade

(Order nodes by criticality, paste the platform-specific upgrade commands, add your gates.)

## Section 7: CRD Storage Version Migration

(List CRDs needing migration — from your Section 8 findings.)

## Section 8: Post-Upgrade Validation

(Run every validation: nodes, pods, DNS, storage, webhooks, CRD controllers, smoke tests.)

## Section 9: Post-Upgrade Controller Upgrades

(For controllers flagged "Upgrade AFTER" in Section 9.)

## Section 10: Cleanup

(Re-enable alerts, autoscaler, auto-upgrade.)

## Section 11: Rollback Procedure

(Platform-specific — etcd restore for on-prem, provider limitation for managed.)

## Section 12: Sign-Off

(Checklist from this document.)

## Section 13: 72-Hour Monitoring Plan

(24h, 48h, 72h checks.)
```

### 21.2 Key Rules for Manual Plan Writing

1. **Every finding from this checklist must appear in the plan.**
2. **Platform-specific commands only** — don't mix on-prem commands into an EKS plan.
3. **If a section has no findings, write:** `NO ISSUES FOUND — PROCEED.`
4. **If you're missing information, write:** `INCOMPLETE: Need <X> before execution.`
5. **Reference the `cluster-upgrade-execution-plan.md` template** in this repo for command syntax.

---

## Section 22 — AI Gap Analysis (Dual-Flow Validation)

### 22.1 Generate AI Output

Now that you have your manual findings, run the prompts:

```bash
# Step A — Generate the AI assessment report
# (Feed prompt.md to an AI assistant with your SOURCE_VERSION and TARGET_VERSION)
# Output saved to: analysis/cluster-upgrade-feasibility-and-risks.md

# Step B — Generate the AI execution plan
# (Feed cluster-upgrade-execution-plan.md with the feasibility report as context)
# Output saved to: plans/plan.md
```

### 22.2 Gap Comparison

Create a third file at `analysis/manual-vs-ai-gaps.md`:

```markdown
# Manual vs AI Gap Analysis

| Area | Manual Finding | AI Finding | Match? | Action |
|------|---------------|------------|--------|--------|
| Controllers found | (list) | (list) | Y/N | |
| Deprecated APIs | (list) | (list) | Y/N | |
| CRDs flagged | (list) | (list) | Y/N | |
| Severity ratings | (list) | (list) | Y/N | |
| Readiness Score | /100 | /100 | diff | |
| Confidence Score | % | % | diff | |

## Discrepancies to Investigate

1. AI found X but manual missed it: ________________________________________________
2. Manual found Y but AI missed it: ________________________________________________
3. Severity disagreement on Z: ________________________________________________
4. Both missed something (found by neither): ________________________________________________

## Final Merged Report

After resolving all gaps, update:
- [ ] analysis/cluster-upgrade-feasibility-and-risks.md (merge findings)
- [ ] plans/plan.md (update with resolved findings)
- [ ] Document residual unknowns: ________________________________________________
```

### 22.3 Gap Resolution Rules

1. **AI missed something you found** → Add it to the AI report. Your manual scan is ground truth.
2. **You missed something AI found** → Verify it exists, then update your manual checklist.
3. **Disagreement on severity** → Research both perspectives. Default to the **higher** severity.
4. **Both missed something** → Document in `manual-vs-ai-gaps.md` as an unknown risk. Reduce confidence by 5 points.
5. **Unknown unknowns remain** → Accept that no process catches everything. Flag them.

---

## Quick Reference — Common Fixes for Common Findings

| Problem | Fix |
|---------|-----|
| `extensions/v1beta1` Ingress | `kubectl get ingress -A` → re-create with `networking.k8s.io/v1` |
| `batch/v1beta1` CronJob | `kubectl get cronjob -A` → re-create with `batch/v1` |
| `policy/v1beta1` PDB | `kubectl get pdb -A` → re-create with `policy/v1` |
| Old kube-proxy version (EKS) | `kubectl set image ds/kube-proxy -n kube-system kube-proxy=<image>` |
| Old CoreDNS version (EKS) | `kubectl set image deploy/coredns -n kube-system coredns=<image>` |
| EKS managed add-on out of date | `aws eks update-addon --cluster-name X --addon-name Y --addon-version Z` |
| VPC CNI outdated | Same as above — it's an EKS add-on |
| etcd snapshot needed (on-prem) | `etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db` |
| kube-proxy mode IPVS | Verify kernel modules: `lsmod \| grep ip_vs` |
| PodDisruptionBudget blocking drain | `kubectl get pdb -A` → temporarily relax `minAvailable` |

---

## Completion Checklist

```
[ ] All 22 sections filled out
[ ] Risk matrix populated
[ ] Readiness & confidence scores calculated
[ ] Every CRITICAL/HIGH finding has a Mandatory Finding Detail
[ ] Execution plan (plans/plan.md) written from findings
[ ] AI output generated (analysis/cluster-upgrade-feasibility-and-risks.md)
[ ] Gap analysis completed (analysis/manual-vs-ai-gaps.md)
[ ] Final merged reports updated

ASSESSOR: ________________________
DATE: ________________________
CLUSTER: ________________________
```
