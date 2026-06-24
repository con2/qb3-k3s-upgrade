# K3s Upgrade Assessment

## TL;DR

The production cluster (`qb`) runs K3s v1.21.6+k3s1 — 15 minor versions behind the current stable v1.36.1+k3s1. The cluster is in better shape than its age suggests: all API objects have already been migrated to non-deprecated versions, so there is **no object-level migration work** before the upgrade. The blockers are entirely in the tooling and a handful of ghost resources left over from decommissioned Longhorn.

**Do the pre-work first, then upgrade in two hops: v1.21 → v1.25 → v1.29 → v1.36.**

---

## Current State

| Item | Value |
|---|---|
| K3s version | v1.21.6+k3s1 (November 2021) |
| Kubernetes gap | 15 minor versions to v1.36.1 |
| Nodes | qb1–qb3: etcd/control-plane · qb4: worker |
| OS | Ubuntu 20.04 (Azure), kernel 5.15 |
| system-upgrade-controller | v0.6.2 (latest: v0.19.2) |
| cert-manager | v1.0.3 (latest: v1.17.x) |
| ingress-nginx | v1.12.1 — modern, already supports K8s 1.32+ |
| Old ARC (summerwind) | v0.17.0 in `actions-runner-system` |
| New ARC | Present in `arc-systems` (migration in progress) |
| containerd | 1.4.11-k3s1 — replaced automatically by K3s upgrade |

All commands below assume:

```bash
export KUBECONFIG=/path/to/qb
```

---

## What's Already in Good Shape

The cluster has been kept up-to-date at the object level even though K3s itself wasn't upgraded. None of the common API migration blockers apply here:

| Check | Status |
|---|---|
| All Ingress objects | `networking.k8s.io/v1` ✓ |
| All CronJobs | `batch/v1` ✓ |
| All webhook configurations | `admissionregistration.k8s.io/v1` ✓ |
| All CRDs | `apiextensions.k8s.io/v1` ✓ |
| HorizontalPodAutoscaler resources | None exist ✓ |

This means the upgrade path has no etcd-level object migrations to perform. The risk is concentrated in component compatibility (cert-manager, SUC) and ghost resources that need cleaning.

---

## Pre-Work

Complete all six items below before changing any K3s version.

### 1. Upgrade system-upgrade-controller (v0.6.2 → v0.19.2)

The SUC is the tool that performs the K3s upgrade itself. v0.6.2 is four years old and its job templates are incompatible with modern K3s upgrade images. This must be upgraded first.

```bash
# Verify current version
kubectl get deployment system-upgrade-controller -n system-upgrade \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Delete the old ClusterRoleBinding first — see note below
kubectl delete clusterrolebinding system-upgrade --ignore-not-found

# Apply the new release — CRDs are bundled in the manifest
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.19.2/system-upgrade-controller.yaml

# Wait for rollout
kubectl rollout status deployment/system-upgrade-controller -n system-upgrade
```

> **Why delete the ClusterRoleBinding first?** v0.6.2 binds the `system-upgrade` ServiceAccount to `cluster-admin`; v0.19.2 rebinds it to a scoped `system-upgrade-controller` role. A binding's `roleRef` is immutable, so `kubectl apply` fails with `cannot change roleRef` on that object (the Deployment still updates, but the task exits non-zero and RBAC stays on the old `cluster-admin` binding). Deleting the binding first lets the new, correctly-scoped one be created. Verified in the playground.

### 2. Upgrade cert-manager — in two phases

cert-manager v1.0.3 is 4.5 years old. Its cainjector patches webhook CA bundles at runtime; if it calls any API removed in v1.22 during the upgrade window, TLS breaks for every service on the cluster. So it must be modernised before the K3s upgrade crosses v1.22.

**But it cannot go straight to latest.** The current cert-manager chart enforces `kubeVersion: >= 1.22.0-0` and refuses to install on v1.21. Verified chart floors:

| cert-manager | requires k8s |
|---|---|
| 1.10.x | ≥ 1.20 |
| **1.11.x** | **≥ 1.21** ← newest line that runs on 1.21 |
| 1.12.x and later (incl. latest) | ≥ 1.22 |

So cert-manager is upgraded in two phases, straddling the first K3s hop. Both phases use **static manifests** — the production and playground installs are static-manifest (not Helm), and `helm upgrade --install` cannot adopt a static install without ownership conflicts.

**Phase A — now, while on k8s 1.21** (before the first K3s hop). Target v1.11.5: it runs on 1.21 and is modern enough (v1 webhook/CRD APIs) to survive the v1.22 API removals during the 1.21 → 1.25 hop.

```bash
# Back up all cert-manager-managed resources first
kubectl get certificates,certificaterequests,issuers,clusterissuers,orders,challenges \
  -A -o yaml > cert-manager-backup-$(date +%Y%m%d).yaml

# Upgrade via static manifest (includes CRDs); v1.11.5 is the k8s-1.21 ceiling
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.5/cert-manager.yaml

kubectl rollout status deployment/cert-manager \
  deployment/cert-manager-cainjector \
  deployment/cert-manager-webhook \
  -n cert-manager --timeout=180s
```

**Phase B — after the K3s 1.25 hop** (cluster now on k8s ≥ 1.22). Upgrade to latest:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

kubectl rollout status deployment/cert-manager \
  deployment/cert-manager-cainjector \
  deployment/cert-manager-webhook \
  -n cert-manager --timeout=180s
```

> **Verify after each phase.** The webhook needs ~60–90s to warm up after a rollout — certificate issuance returns `context deadline exceeded` until it's ready, then recovers. Confirm with a self-signed Issuer + Certificate and check it reaches `Ready` (allow ≥90s), or just confirm existing certs stay valid: `kubectl get certificates -A`. The two-phase path (v1.0.3 → v1.11.5 on k8s 1.21, then → latest) was validated end-to-end in the playground.

### 3. Remove ghost Longhorn resources

Longhorn is fully decommissioned. These three objects are orphaned remnants. The PSP is the most important: it is the **only** PodSecurityPolicy in the cluster and would be a hard blocker at v1.25 where PSPs are removed.

```bash
# Delete the only PodSecurityPolicy in the cluster
kubectl delete podsecuritypolicy longhorn-nfs-provisioner

# Delete the orphaned StorageClass
kubectl delete storageclass longhorn-nfs

# The outline-ropecon PVC is stuck in Terminating because the Longhorn
# finalizer controller is gone. Strip the finalizer so Kubernetes can finish.
kubectl patch pvc redis-data -n outline-ropecon \
  -p '{"metadata":{"finalizers":null}}' --type=merge
```

### 4. Remove ghost Crunchydata CRDs

PGO v4 was decommissioned. No resources exist for any of these CRDs; they are dead API surface.

```bash
kubectl delete crd \
  pgclusters.crunchydata.com \
  pgreplicas.crunchydata.com \
  pgtasks.crunchydata.com \
  pgpolicies.crunchydata.com
```

### 5. Fix: kubernetes-secret-generator crash loop

This deployment has been crash-looping for a very long time (28,000+ restarts). It adds noise to health checks during the upgrade and should be resolved beforehand.

```bash
# Review crash logs
kubectl logs -n kubernetes-secret-generator \
  deployment/kubernetes-secret-generator --previous

kubectl describe deployment kubernetes-secret-generator \
  -n kubernetes-secret-generator
```

The running image is `quay.io/mittwald/kubernetes-secret-generator:v3.1.0`, built on operator-sdk v0.16.0 (Go 1.15, ~2020). It crashes during component registration, most likely due to API incompatibility with the current API server version. Check the [mittwald/kubernetes-secret-generator releases](https://github.com/mittwald/kubernetes-secret-generator/releases) for a current image tag and update the deployment.

### 6. Fix: tm namespace stuck Terminating

The `tm` namespace has been stuck in `Terminating` for 81 days. The namespace controller is blocked because `metrics.k8s.io/v1beta1` returns an auth error during API group discovery, preventing it from confirming all resources are gone. All content has actually been deleted; only the namespace object itself is stuck.

Force-unblock it by calling the finalize endpoint directly:

```bash
kubectl get namespace tm -o json \
  | python3 -c "
import sys, json
ns = json.load(sys.stdin)
ns['spec']['finalizers'] = []
print(json.dumps(ns))
" | kubectl replace --raw /api/v1/namespaces/tm/finalize -f -
```

This clears the finalizer list without touching any resources (they are already gone), which lets the namespace controller complete the deletion.

---

## Upgrade Path

**Recommended: 2-stop staged upgrade**

```
v1.21.6+k3s1  →  v1.25.x  →  v1.29.x  →  v1.36.1+k3s1
```

### Why these stops

- **v1.25** is the boundary where `policy/v1beta1` (PodSecurityPolicy) and `batch/v1beta1` CronJobs are removed. The PSP will already be deleted and all CronJobs already use `batch/v1`, so this stop is a verification checkpoint, not a migration step. Confirm cert-manager, ingress-nginx, and all application services are healthy before continuing. **This is also where cert-manager Phase B runs** (pre-work item 2): the cluster is now on k8s ≥ 1.22, so cert-manager can finally go to latest.

- **v1.29** crosses the v1.26 removal of `autoscaling/v2beta2` HPA (no HPAs exist in this cluster) and several flow-control API promotions. A second verification point before the final jump.

- **v1.36.1** is the current latest stable. v1.33–v1.36 still have RC patch releases in flight; if you prefer a more settled target, v1.32.13+k3s1 is the most recent fully-stable patch series.

### Upgrade mechanics

Each hop is performed by applying a `server-plan` (control-plane nodes) and an `agent-plan` (workers) to the system-upgrade-controller. The SUC cordons each node, runs the K3s upgrade binary, and uncordons — one node at a time, servers before agents.

**Example: first hop to v1.25**

```bash
kubectl apply -f - <<'EOF'
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/master, operator: In, values: ["true"]}
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.25.16+k3s4
---
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
      - {key: node-role.kubernetes.io/master, operator: DoesNotExist}
  prepare:
    args: ["prepare", "server-plan"]
    image: rancher/k3s-upgrade:v1.25.16-k3s4
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.25.16+k3s4
EOF

# Watch progress
kubectl get plans -n system-upgrade -w
kubectl get nodes -o wide -w
```

Note the image tag format difference: the `prepare` image uses `v1.25.16-k3s4` (hyphen) while the `version` field uses `v1.25.16+k3s4` (plus).

Repeat with updated version strings for the subsequent hops. The `task k3s:upgrade -- <version>` command in this repo's Taskfile handles the substitution automatically.

> **Expected during each hop — don't mistake it for a hang.** SUC upgrades each node one at a time, servers before agents, and the upgrade job runs *on the node it's upgrading*. When k3s restarts on that node, the kubelet stops reporting the pod that triggered the restart, so it's orphaned into `Unknown` status and the node briefly shows `Ready,SchedulingDisabled` (cordoned). SUC then runs a replacement pod that finalizes and uncordons. A node sitting cordoned mid-hop is normal; confirm completion via `kubectl get plans -n system-upgrade -o wide` showing `COMPLETE=True`, not by watching individual pods. After the hop, one `Unknown` ghost pod per node remains in `system-upgrade` — harmless. Clear them with `kubectl delete jobs -n system-upgrade --all` (the plans are CRDs and survive; only the completed jobs and their ghost pods are removed).

### Health check after each hop

Run this once `plans` show `COMPLETE=True`, before proceeding to the next version:

```bash
# All nodes Ready and on the new version
kubectl get nodes -o wide

# Unhealthy pods, excluding the system-upgrade ghosts (Unknown-status job pods are expected)
kubectl get pods -A --field-selector=metadata.namespace!=system-upgrade --no-headers \
  | grep -v -E "Running|Completed|Succeeded" || echo "All pods healthy."

# Certificates still valid
kubectl get certificates -A

# Ingress responding (spot-check)
curl -sI https://kompassi.eu | head -1
```

---

## Testing in OrbStack

This repo's Taskfile spins up an OrbStack VM cluster that replicates the production starting state: K3s v1.21.6+k3s1, cert-manager v1.0.3, and system-upgrade-controller v0.6.2. Use it to run through the full upgrade sequence before touching production.

```bash
task up          # create VMs + fetch kubeconfig + mirror production state
task k3s:status  # confirm starting version

task upgrade:suc                # SUC -> v0.19.2
task upgrade:cert-manager:pre   # cert-manager -> v1.11.5 (k8s 1.21 ceiling)

task k3s:upgrade -- v1.25.16+k3s4   # first hop
task k3s:upgrade:watch               # follow progress
task k3s:health                      # verify

task upgrade:cert-manager:final # now on k8s >= 1.22: cert-manager -> latest

task k3s:upgrade -- v1.29.14+k3s1   # second hop
task k3s:upgrade -- v1.36.1+k3s1    # final hop

task down        # tear down when done
```

The cloud-init scripts pin the K3s version via `INSTALL_K3S_VERSION=v1.21.6+k3s1` passed to the K3s installer, giving a faithful replica of the production starting point.

---

## Risk Register

| Risk | Severity | Mitigation | Status |
|---|---|---|---|
| cert-manager v1.0.3 runtime compatibility across v1.22 API removals | High | Two-phase upgrade: → v1.11.5 before first hop (latest can't install on k8s 1.21), → latest after the 1.25 hop | Pre-work item 2 |
| system-upgrade-controller v0.6.2 incompatible with modern K3s upgrade images | High | Upgrade SUC to v0.19.2 first | Pre-work item 1 |
| PodSecurityPolicy removed at v1.25 | High (if not deleted) | Delete `longhorn-nfs-provisioner` PSP before starting | Pre-work item 3 |
| Old summerwind ARC v0.17.0 compatibility with K8s 1.22+ | Medium | Test in playground; complete migration to new ARC before upgrade | Needs investigation |
| Ubuntu 20.04 EOL (April 2025) | Medium | Plan OS upgrade after K3s is current | Separate work item |
| containerd 1.4.11 (very old) | Low–Medium | Replaced automatically during K3s node upgrade | Handled by K3s upgrade |
| kubernetes-secret-generator crash loop | Low | Upgrade image; investigate root cause | Pre-work item 5 |
| tm namespace stuck Terminating | Low | Force-clear finalizers via finalize endpoint | Pre-work item 6 |
