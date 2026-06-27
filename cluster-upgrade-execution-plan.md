# Kubernetes Cluster Upgrade Execution Plan (Platform-Aware Template)

## Objective

You are acting as a Senior Kubernetes Platform Engineer. Your task is to read the assessment report `cluster-upgrade-feasibility-and-risks.md`, detect the **platform** from its Executive Summary, and produce a **platform-specific** step-by-step upgrade execution plan.

**Output file: `plan.md`** — a new file containing ONLY the upgrade plan for the detected platform. DO NOT overwrite this template file.

---

## Input File

```text
cluster-upgrade-feasibility-and-risks.md
```

Read its entire contents. Extract:

| Field | Source in Report |
|-------|-----------------|
| Platform | Executive Summary → `PLATFORM:` field |
| Source Version | Executive Summary → `SOURCE VERSION:` |
| Target Version | Executive Summary → `TARGET VERSION:` |
| Upgrade Decision | Executive Summary → `UPGRADE DECISION:` |
| Readiness Score | Executive Summary → `READINESS SCORE:` |
| Confidence Score | Executive Summary → `CONFIDENCE:` |
| Node Count | Step 1 |
| CNI Plugin | Step 11 |
| All CRITICAL / HIGH RISK findings | Risk Matrix + Mandatory Rule section |
| Deprecated API inventory | Step 6 & Step 7 |
| CRD compatibility findings | Step 8 |
| Controller upgrade timeline (before vs after) | Step 9 |
| Admission webhook risks | Step 10 |
| Resource pressure status | Step 15 |

---

## Section Applicability by Platform

Not all sections apply to all platforms. Generate ONLY the sections marked for the detected platform.

| Section | On-Prem | EKS | GKE | AKS |
|---------|---------|-----|-----|-----|
| 1. Pre-Flight Health Check | FULL | FULL | FULL | FULL |
| 2. Pre-Upgrade Remediation (APIs, CRDs, Controllers, Webhooks) | FULL | FULL | FULL | FULL |
| 3. Backup & Safety | FULL (etcd snapshot + manifest export) | LITE (manifest export only) | LITE (manifest export only) | LITE (manifest export only) |
| 4. Pre-Upgrade Configuration (alerts, autoscaler, PDBs) | FULL | FULL | FULL | FULL |
| 5. Control Plane Upgrade | FULL (kubeadm) | LITE (aws CLI) | LITE (gcloud CLI) | LITE (az CLI) |
| 6. Data Plane Upgrade (nodes) | FULL (manual drain + SSH × N) | LITE (managed node group command) | LITE (nodepool command) | LITE (nodepool command) |
| 7. CRD Storage Version Migration | YES | YES | YES | YES |
| 8. Post-Upgrade Validation | FULL | FULL | FULL | FULL |
| 9. Post-Upgrade Controller Upgrades | YES | YES | YES | YES |
| 10. Post-Upgrade Cleanup | FULL | FULL | FULL | FULL |
| 11. Rollback Procedure | FULL (etcd restore) | LITE (provider limitation note) | LITE (provider limitation note) | LITE (provider limitation note) |
| 12. Upgrade Completion Sign-Off | YES | YES | YES | YES |
| 13. 72-Hour Monitoring | YES | YES | YES | YES |

---

## Output Header (Always Generated)

```text
# Upgrade Execution Plan: <CLUSTER-NAME>

PLATFORM: <ON-PREM | EKS | GKE | AKS>
SOURCE: <SOURCE_VERSION>
TARGET: <TARGET_VERSION>
DECISION: <APPROVED | CONDITIONAL | NOT RECOMMENDED>
READINESS: <XX/100>
CONFIDENCE: <XX%>
EXPECTED DURATION: <calculated>
PLAN GENERATED: <YYYY-MM-DD HH:MM UTC>
```

---

## Section 1: Pre-Flight Health Check [ALL PLATFORMS]

```bash
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get events -A --sort-by='.lastTimestamp' | tail -30
kubectl get --raw /healthz
kubectl get --raw /livez
kubectl get deploy -A | grep -v "1/1\|2/2\|3/3"
kubectl get sts -A | grep -v "1/1\|2/2\|3/3"
kubectl get pods -A | grep CrashLoopBackOff || echo "No CrashLoopBackOff pods"
kubectl get pods -A | grep Pending || echo "No pending pods"
```

**GATE:** If anything fails, halt. Resolve before proceeding.

---

## Section 2: Pre-Upgrade Remediation [ALL PLATFORMS]

Populated from the feasibility report findings. Generate these subsections:

### 2.1 Deprecated API Migration

For every deprecated API object from the report:

```text
ACTION [CRITICAL]: <object> in <namespace>
  Current API: <deprecated-api>
  Must migrate to: <replacement-api>

  kubectl get <kind> <object> -n <namespace> -o yaml > /tmp/<object>-backup.yaml
  # Edit /tmp/<object>-backup.yaml: apiVersion → <replacement-api>
  kubectl replace -f /tmp/<object>-backup.yaml --force

  VERIFY: kubectl get <kind> <object> -n <namespace> -o jsonpath='{.apiVersion}'
  STATUS: [ ] COMPLETE
```

If the report found zero deprecated APIs, write: **NO DEPRECATED APIS FOUND — SKIP.**

### 2.2 CRD Upgrades

For every CRD flagged with "YES" for breakage:

```text
ACTION: Upgrade CRD <crd-name>
  Issue: <from report>
  Required version: <from report>

  kubectl apply -f <crd-manifest>

  VERIFY: kubectl get crd <crd-name> -o jsonpath='{.spec.versions[*].name}'
  STATUS: [ ] COMPLETE
```

If zero CRDs flagged, write: **NO CRD ISSUES FOUND — SKIP.**

### 2.3 Controller/Operator Upgrades (Pre-Kubernetes)

For every controller classified as "Upgrade BEFORE Kubernetes":

```text
ACTION: Upgrade <controller> from <installed-ver> to <target-ver>
  <upgrade-command>

  VERIFY: kubectl get deploy <controller> -n <ns> -o jsonpath='{.spec.template.spec.containers[0].image}'
  STATUS: [ ] COMPLETE
```

### 2.4 Admission Webhook Remediation

For every webhook flagged:

```text
ACTION: <remediation from report>

  STATUS: [ ] COMPLETE
```

### 2.5 Resource Pressure Resolution

If resource pressure detected in report:

```text
ACTION: Free capacity or scale up before upgrade
  CPU headroom needed: <XX cores>
  Memory headroom needed: <XX GB>

  STATUS: [ ] COMPLETE
```

---

## Section 3: Pre-Upgrade Backup & Safety

### ON-PREM — FULL BACKUP

```bash
# 3.1 etcd snapshot
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# 3.2 Verify snapshot integrity
etcdctl snapshot status /backup/etcd-snapshot-*.db --write-out=table

# 3.3 etcd defrag (if >2GB)
etcdctl defrag --endpoints=https://127.0.0.1:2379

# 3.4 Certificate expiry check
kubeadm certs check-expiration

# 3.5 Resource manifest export
kubectl get all -A -o yaml > /tmp/cluster-backup-all-$(date +%Y%m%d-%H%M%S).yaml
kubectl get crd -o yaml > /tmp/cluster-backup-crds-$(date +%Y%m%d-%H%M%S).yaml
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -o yaml > /tmp/cluster-backup-webhooks-$(date +%Y%m%d-%H%M%S).yaml
kubectl get pv,pvc,sc -A -o yaml > /tmp/cluster-backup-storage-$(date +%Y%m%d-%H%M%S).yaml

# 3.6 Namespace-level backups
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get all -n $ns -o yaml > /tmp/ns-backup-${ns}-$(date +%Y%m%d).yaml
done
```

### EKS / GKE / AKS — LITE BACKUP

```bash
# Managed platforms: control plane is provider-managed. No etcd access.
# Export manifests only.

kubectl get all -A -o yaml > /tmp/cluster-backup-all-$(date +%Y%m%d-%H%M%S).yaml
kubectl get crd -o yaml > /tmp/cluster-backup-crds-$(date +%Y%m%d-%H%M%S).yaml
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -o yaml > /tmp/cluster-backup-webhooks-$(date +%Y%m%d-%H%M%S).yaml
kubectl get pv,pvc,sc -A -o yaml > /tmp/cluster-backup-storage-$(date +%Y%m%d-%H%M%S).yaml

echo "Ensure platform-native backup exists:"
echo "  EKS: Verify Velero backup or AWS Backup plan is current"
echo "  GKE: gcloud container backup-restore list --project=<project> | confirm recent success"
echo "  AKS: az backup vault show --resource-group <rg> --name <vault> | confirm recent success"
```

---

## Section 4: Pre-Upgrade Configuration [ALL PLATFORMS]

### 4.1 Silence Alerts

```text
ACTION: Silence monitoring alerts for the maintenance window
  Window: <START_TIME> → <END_TIME>
  STATUS: [ ] COMPLETE
```

### 4.2 Pause Autoscaler

```text
ON-PREM ONLY:
  kubectl scale deploy cluster-autoscaler -n kube-system --replicas=0
  STATUS: [ ] COMPLETE

EKS / GKE / AKS:
  SKIP — Cloud provider handles scaling during managed upgrades.
```

### 4.3 Validate PDBs

```bash
kubectl get pdb -A
# For any PDB with ALLOWED DISRUPTIONS = 0, verify it won't block drains.
# Temporarily relax minAvailable if necessary.
```

### 4.4 Notify Stakeholders

```text
NOTIFICATION SENT:
  - [ ] Platform Engineering
  - [ ] Application Teams
  - [ ] On-Call
STATUS: [ ] COMPLETE
```

---

## Section 5: Control Plane Upgrade

### ON-PREM (kubeadm)

```bash
# 5.1 Dry-run upgrade plan
kubeadm upgrade plan v<TARGET_VERSION>

# 5.2 Apply upgrade on primary control plane
kubeadm upgrade apply v<TARGET_VERSION>

# 5.3 Upgrade kubelet + kubectl on all control plane nodes
for node in <cp-node-1> <cp-node-2> <cp-node-3>; do
  ssh $node "kubeadm upgrade node"
  ssh $node "apt-get update && apt-get install -y kubelet=<TARGET_VERSION>-00 kubectl=<TARGET_VERSION>-00"
  ssh $node "systemctl daemon-reload && systemctl restart kubelet"
done

# 5.4 Verify control plane
kubectl get nodes
kubectl version
```

### EKS

```bash
# 5.1 Verify EKS add-ons are at target-compatible versions FIRST
aws eks describe-addon --cluster-name <cluster> --addon-name vpc-cni
aws eks describe-addon --cluster-name <cluster> --addon-name coredns
aws eks describe-addon --cluster-name <cluster> --addon-name kube-proxy

# 5.2 Upgrade EKS control plane
aws eks update-cluster-version --cluster-name <cluster> --version <TARGET_VERSION>
aws eks describe-update --cluster-name <cluster> --update-id <id> --query "update.status"
# Wait for "Successful"

# 5.3 Upgrade legacy EKS add-ons to target versions
aws eks update-addon --cluster-name <cluster> --addon-name vpc-cni --addon-version <cni-ver>
aws eks update-addon --cluster-name <cluster> --addon-name coredns --addon-version <coredns-ver>
aws eks update-addon --cluster-name <cluster> --addon-name kube-proxy --addon-version <kubeproxy-ver>
aws eks update-addon --cluster-name <cluster> --addon-name aws-ebs-csi-driver --addon-version <ebs-ver>

# 5.4 Verify
aws eks describe-cluster --name <cluster> --query "cluster.version"
```

### GKE

```bash
# 5.1 Verify target is available in release channel
gcloud container get-server-config --zone <zone> --format "yaml(validMasterVersions)"

# 5.2 Disable auto-upgrade temporarily (if coordinating manual upgrade)
gcloud container clusters update <cluster> --zone <zone> --no-enable-autoupgrade

# 5.3 Upgrade control plane
gcloud container clusters upgrade <cluster> --zone <zone> --master --cluster-version <TARGET_VERSION>

# 5.4 Wait for completion
gcloud container operations list --cluster <cluster> --zone <zone> --filter="status=RUNNING"
# Proceed when no running operations

# 5.5 Verify
gcloud container clusters describe <cluster> --zone <zone> --format "value(currentMasterVersion)"
```

### AKS

```bash
# 5.1 Verify target available
az aks get-upgrades --resource-group <rg> --name <cluster> --output table

# 5.2 Validate no maintenance window conflicts
az aks show --resource-group <rg> --name <cluster> --query "maintenanceWindow"

# 5.3 Upgrade control plane
az aks upgrade --resource-group <rg> --name <cluster> --kubernetes-version <TARGET_VERSION> --no-wait
az aks wait --resource-group <rg> --name <cluster> --updated

# 5.4 Verify
az aks show --resource-group <rg> --name <cluster> --query "kubernetesVersion"
```

### Post Control-Plane Gate [ALL PLATFORMS]

```bash
kubectl version
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get apiservices | grep False
```

**GATE:** If any `apiservice` reports `False`, pause. Do NOT proceed to data plane.

---

## Section 6: Data Plane Upgrade (Nodes)

### 6.1 Node Upgrade Order [ALL PLATFORMS]

Based on workload criticality from the assessment report:

```text
TIER 1 — NON-CRITICAL (upgrade first, validate):
  <node-list>

TIER 2 — CRITICAL (upgrade one at a time, verify between each):
  <node-list>
```

### ON-PREM — Node-by-Node Manual Upgrade

```bash
NODE="<node-name>"

# 1. Cordon
kubectl cordon $NODE

# 2. Drain (respects PDBs, graceful termination)
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --grace-period=120 --timeout=600s
# If drain hangs, identify blocking pod/PDB and resolve

# 3. SSH and upgrade
ssh $NODE "kubeadm upgrade node"
ssh $NODE "apt-get update && apt-get install -y kubelet=<TARGET_VERSION>-00 kubectl=<TARGET_VERSION>-00"
ssh $NODE "systemctl daemon-reload && systemctl restart kubelet"

# 4. Uncordon
kubectl uncordon $NODE

# 5. Wait for Ready
kubectl wait --for=condition=Ready node/$NODE --timeout=300s

# 6. Validate
kubectl get node $NODE
kubectl describe node $NODE | grep -i "ready\|version"
kubectl get pods -A --field-selector spec.nodeName=$NODE | grep -v Running | grep -v Completed

# 7. REPEAT for next node — one at a time
```

**ON-PREM TIME ESTIMATE:** ~5-10 min per worker node. Multiply by node count.

### EKS — Managed Node Groups

```bash
NODEGROUP="<nodegroup-name>"

# In-place update (recommended for managed node groups):
aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name $NODEGROUP --kubernetes-version <TARGET_VERSION>

# Monitor:
aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name $NODEGROUP --query "nodegroup.status"
# Poll until status = Active (~20-40 min per node group)

# Verify:
kubectl get nodes -l "eks.amazonaws.com/nodegroup=$NODEGROUP"
```

If using self-managed (non-EKS) node groups:
```bash
# Cordon → Drain → Terminate instance (ASG replaces it with updated AMI)
NODE="<node-name>"
kubectl cordon $NODE
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=600s
aws ec2 terminate-instances --instance-ids $(kubectl get node $NODE -o jsonpath='{.spec.providerID}' | cut -d/ -f5)
```

### GKE — Node Pools

```bash
NODEPOOL="<nodepool-name>"

# Upgrade (SURGE strategy — creates new nodes before removing old):
gcloud container clusters upgrade <cluster> --zone <zone> --node-pool $NODEPOOL

# Monitor:
gcloud container operations list --cluster <cluster> --zone <zone> --filter="status=RUNNING"
# Proceed when no running operations

# Verify:
gcloud container node-pools describe $NODEPOOL --cluster <cluster> --zone <zone> --format "value(version)"
```

### AKS — Node Pools

```bash
NODEPOOL="<nodepool-name>"

# System pools FIRST:
az aks nodepool upgrade --resource-group <rg> --cluster-name <cluster> --name <system-pool> --kubernetes-version <TARGET_VERSION> --no-wait
az aks nodepool wait --resource-group <rg> --cluster-name <cluster> --name <system-pool> --updated

# User pools next:
az aks nodepool upgrade --resource-group <rg> --cluster-name <cluster> --name $NODEPOOL --kubernetes-version <TARGET_VERSION> --no-wait
az aks nodepool wait --resource-group <rg> --cluster-name <cluster> --name $NODEPOOL --updated

# Verify:
az aks nodepool show --resource-group <rg> --cluster-name <cluster> --name $NODEPOOL --query "orchestratorVersion"
```

### Node Upgrade Gates [ALL PLATFORMS]

After each node/nodegroup, verify before proceeding:

```bash
# Gate 1: Node Ready
kubectl get node <node> -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep True

# Gate 2: Correct kubelet version
kubectl get node <node> -o jsonpath='{.status.nodeInfo.kubeletVersion}' | grep <TARGET_VERSION>

# Gate 3: No pod failures on this node
kubectl get pods -A --field-selector spec.nodeName=<node> | grep -E "Error|CrashLoopBackOff|ImagePullBackOff" || echo "No issues"

# Gate 4: No new cluster-wide errors
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

**IF GATE FAILS:** Stop. Diagnose. Do NOT proceed to next node.

---

## Section 7: CRD Storage Version Migration [ALL PLATFORMS]

```bash
# Check stored versions still on old APIs
kubectl get crd -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.storedVersions}{"\n"}{end}' | grep -v <STORAGE_VERSION>

# For CRDs flagged in assessment report that need migration:
kubectl patch crd <crd-name> --type='json' -p='[{"op": "replace", "path": "/spec/versions/0/storage", "value": true}]'
# Most operators handle this automatically — validate controller logs
```

---

## Section 8: Post-Upgrade Validation [ALL PLATFORMS]

Every item must pass before sign-off.

### 8.1 Core Infrastructure

```bash
# All nodes Ready at target
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[?(@.type=="Ready")].status,VERSION:.status.nodeInfo.kubeletVersion

# All pods healthy
kubectl get pods -A --no-headers | grep -vE "Running|Completed" && echo "WARNING" || echo "All pods healthy"

# All apiservices Available
kubectl get apiservices | grep False && echo "WARNING" || echo "All apiservices healthy"

# All deployments ready
kubectl get deploy -A --no-headers | awk '{if ($2 != $3 && $2 != "READY") print}' | grep -E "^" && echo "WARNING" || echo "All deployments ready"

# All statefulsets ready
kubectl get sts -A --no-headers | awk '{if ($2 != $3) print}' | grep -E "^" && echo "WARNING" || echo "All StatefulSets ready"

# All daemonsets ready
kubectl get ds -A --no-headers | awk '{if ($2 != $3 || $2 != $4) print}' | grep -E "^" && echo "WARNING" || echo "All DaemonSets ready"
```

### 8.2 DNS

```bash
kubectl run dns-test-$(date +%s) --rm -i --restart=Never --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local
kubectl run dns-ext-test-$(date +%s) --rm -i --restart=Never --image=busybox:1.36 -- nslookup google.com
```

### 8.3 Storage

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: upgrade-test-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 1Gi } }
EOF

kubectl run storage-test --rm -i --restart=Never --image=busybox:1.36 -- sh -c "echo ok > /data/test && cat /data/test" \
  --overrides='{"spec":{"containers":[{"name":"t","image":"busybox:1.36","command":["sh","-c","echo ok > /data/test && cat /data/test"],"volumeMounts":[{"mountPath":"/data","name":"v"}]}],"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"upgrade-test-pvc"}}]}}'

kubectl delete pvc upgrade-test-pvc
```

### 8.4 Admission Webhooks

```bash
kubectl create deployment webhook-test --image=nginx:alpine --replicas=1 --dry-run=client -o yaml | kubectl apply -f -
kubectl wait --for=condition=Available deploy/webhook-test --timeout=120s
kubectl delete deploy webhook-test
```

### 8.5 CRD Controller Reconciliation

For each critical CRD controller from the assessment report:

```bash
kubectl logs -n <ns> deploy/<controller> --tail=30 | grep -i "error\|fail" || echo "No errors"
```

### 8.6 Application Smoke Tests

```text
APPLICATION: <name>
  ENDPOINT: <url>
  EXPECTED: 200
  RESULT: [ ] PASS [ ] FAIL
```

### 8.7 Security

```bash
kubectl auth can-i list pods --as=system:serviceaccount:<ns>:<sa>
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].securityContext.privileged==true) | "\(.metadata.namespace)/\(.metadata.name)"'
```

---

## Section 9: Post-Upgrade Controller Upgrades [ALL PLATFORMS]

For controllers classified as "Upgrade AFTER Kubernetes" from the assessment report:

```text
ACTION: Upgrade <controller> from <old-ver> to <new-ver>
  VERIFY: kubectl get deploy <controller> -n <ns> -o jsonpath='{.spec.template.spec.containers[0].image}'
  STATUS: [ ] COMPLETE
```

---

## Section 10: Post-Upgrade Cleanup [ALL PLATFORMS]

```bash
# Un-silence alerts
echo "Re-enable alerting"

# Unpause autoscaler (on-prem only)
kubectl scale deploy cluster-autoscaler -n kube-system --replicas=<original> 2>/dev/null || true

# EKS: No action (managed node groups handle scaling)
# GKE: Re-enable auto-upgrade if it was disabled
gcloud container clusters update <cluster> --zone <zone> --enable-autoupgrade 2>/dev/null || true
# AKS: Re-enable auto-upgrade channel
az aks update --resource-group <rg> --name <cluster> --auto-upgrade-channel <channel> 2>/dev/null || true

# Clean old ReplicaSets
kubectl get rs -A | grep "0\s" | awk '{print $1, $2}' | xargs -n2 sh -c 'kubectl delete rs $1 -n $0' 2>/dev/null || true
```

---

## Section 11: Rollback Procedure

### ON-PREM — FULL ROLLBACK

```bash
# Control plane failure: Restore etcd snapshot
etcdctl snapshot restore /backup/etcd-snapshot-<timestamp>.db \
  --name <cp-node-name> \
  --initial-cluster <cp-node-name>=https://<ip>:2380 \
  --initial-advertise-peer-urls https://<ip>:2380 \
  --data-dir /var/lib/etcd-restored
# Update kube-apiserver.yaml static pod manifest to use restored data dir
# Restart kubelet to re-create apiserver with restored etcd

# Node failure:
ssh <node> "apt-get install -y kubelet=<OLD_VERSION>-00 && systemctl restart kubelet"
kubectl uncordon <node>
```

### EKS / GKE / AKS — MITIGATION ONLY (NO CONTROL PLANE DOWNGRADE)

```text
CRITICAL: <EKS/GKE/AKS> does NOT support control plane downgrade.

IF CONTROL PLANE FAILS:
  - Open a <AWS/Google Cloud/Azure> support ticket immediately
  - Prepare to restore workloads to a new cluster at the old version from backup

IF NODE UPGRADE FAILS:
  - Cordon the failed node: kubectl cordon <node>
  - Drain workloads: kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
  - The managed node group will automatically replace the failed node
  - Verify the replacement node joins and is Ready
```

---

## Section 12: Upgrade Completion Sign-Off [ALL PLATFORMS]

```text
COMPLETION CHECKLIST:
  [ ] All nodes at target version
  [ ] All pods Running/Completed
  [ ] All deployments/StatefulSets/DaemonSets ready
  [ ] DNS resolution functional
  [ ] Storage functional (PVC create/mount/write/read/delete)
  [ ] Admission webhooks functional
  [ ] CRD controllers reconciling without errors
  [ ] Application smoke tests passed
  [ ] Alerts re-enabled
  [ ] Autoscaler restored (on-prem only)
  [ ] Node auto-upgrade re-enabled (GKE/AKS)
  [ ] No new events/errors

START TIME: <timestamp>
END TIME:   <timestamp>
DURATION:   <HH:MM>
ISSUES:     <none / list>

OPERATOR SIGN-OFF: __________________
```

---

## Section 13: 72-Hour Post-Upgrade Monitoring [ALL PLATFORMS]

```text
24-HOUR CHECK:  [ ] CLEAN / [ ] ISSUES
  kubectl get nodes && kubectl get pods -A | grep -v Running | grep -v Completed
  kubectl get events -A --sort-by='.lastTimestamp' | tail -30
  Issues: <details>

48-HOUR CHECK:  [ ] CLEAN / [ ] ISSUES
  Issues: <details>

72-HOUR CHECK:  [ ] CLEAN / [ ] ISSUES
  Issues: <details>

FINAL CLASSIFICATION: [ ] SUCCESS [ ] PARTIAL [ ] FAILURE
```

---

## Generation Rules

### Rule 0: Output File

Write the generated execution plan to **`plan.md`** (a new file). DO NOT overwrite `cluster-upgrade-execution-plan.md` (this template) or `prompt.md`. Both templates remain unchanged.

### Rule 1: Platform-Only Output

Read `cluster-upgrade-feasibility-and-risks.md` → Extract `PLATFORM:` field → Generate ONLY the sections applicable to that platform. Do NOT include sections for other platforms.

### Rule 2: Finding-Driven Content

Every CRITICAL, HIGH, and WARNING from the assessment report must appear in the corresponding section of this plan. If a section has zero findings, write:

```text
SECTION <N>: NO ISSUES FOUND — PROCEED
```

### Rule 3: On-Prem = Heavy, Managed = Light

- **On-Prem:** etcd backup, kubeadm steps, manual SSH drain/upgrade per node, cert expiry checks, rollback via etcd restore — full detail.
- **EKS / GKE / AKS:** Provider CLI commands, managed node group/pool operations, no etcd access, no cert management, simplified rollback (node replacement). Managed clusters require **significantly fewer steps**.

### Rule 4: Missing Information

If the feasibility report lacks information needed for a section:

```text
SECTION <N>: INCOMPLETE
Missing: <exactly what is missing>
Action: Operator must manually provide <information>.
```

### Rule 5: Completeness Enforcement

Every finding from `cluster-upgrade-feasibility-and-risks.md` must appear somewhere in this execution plan. Silent omission is not allowed.
