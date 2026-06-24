# K3s Upgrade Write-up: qb cluster, v1.21.6 → v1.36.1

**Date:** 2026-06-24 · **Cluster:** `qb` (4 nodes, Hyper-V, Ubuntu 20.04) · **Outcome:** ✅ success; limited end-user impact (one full outage during an incident — see below)

## TL;DR

The production cluster had been stuck on K3s **v1.21.6+k3s1** (Kubernetes 1.21, ~4.5 years old) for a long time. We upgraded it to **v1.36.1+k3s1** — 15 minor versions — in staged hops via the system-upgrade-controller, after rehearsing the entire procedure on a throwaway OrbStack cluster.

All four nodes are now on **v1.36.1+k3s1** (containerd 2.2.3), etcd quorum healthy, all workloads running. Removing cert-manager for the duration caused **no certificate impact** — TLS Secrets persist independently of cert-manager, so existing HTTPS kept working. **End-user impact was limited overall, but not zero:** the disk-swap incident on qb2 caused a full services outage for its duration because public DNS points at qb2 (a known ingress single-point-of-failure) — details under [End-user impact](#end-user-impact).

Two incidents occurred during the production run (a cgroup v1→v2 incompatibility and a Hyper-V disk-mount swap that masqueraded as etcd data loss). Both were resolved without data loss; neither was reproducible in the playground because both are OS/hardware-level. They are the most useful part of this write-up.

## Why this was non-trivial

- **15-version jump.** K8s supports a +1 minor skew between components; you can't go 1.21→1.36 in one shot. We staged it: **1.21 → 1.25 → 1.29 → 1.36**, with health checks between hops.
- **Ancient supporting components.** system-upgrade-controller v0.6.2 and cert-manager v1.0.3 were as old as the cluster and needed handling before/around the upgrade.
- **It's production.** Real TLS for ~30 sites, a 3-node embedded-etcd control plane, and StatefulSet data (MinIO, Redis) on node-local storage.

The cluster was in better shape than its age suggested: all API objects had already been migrated to non-deprecated versions (Ingress `networking.k8s.io/v1`, CronJobs `batch/v1`, etc.), so there was no object-level migration to do — the risk was concentrated in tooling and node-level specifics.

## Approach: rehearse first, then execute

We built a disposable OrbStack VM cluster (this repo's Taskfile) that mirrored the production starting state — K3s v1.21.6, cert-manager v1.0.3 (Helm-managed, matching prod), SUC v0.6.2 — and drove the whole upgrade there before touching production. Every task is `KUBECONFIG`-overridable, so the *same commands* we rehearsed ran against prod by pointing at the real kubeconfig.

### What the rehearsal caught (and saved us from in prod)

1. **SUC self-upgrade RBAC.** Upgrading SUC v0.6.2 → v0.19.2 fails because the old release binds its ServiceAccount to `cluster-admin` and the new one rebinds to a scoped role — and a binding's `roleRef` is immutable. Fix: delete the old ClusterRoleBinding first.
2. **cert-manager can't leap to latest.** The current chart requires k8s ≥ 1.22 (can't install on 1.21), and cert-manager ≥ v1.18 ships CRDs using `selectableFields` (needs k8s ≥ 1.30). A direct `helm upgrade` from the ancient v1.0.3 also fails on a CRD-conversion merge (cert-manager doesn't support skipping minors).
3. **The cert-manager strategy we settled on:** remove cert-manager entirely before the hops and reinstall a current version fresh at the end. This is safe because **cert-manager does not own the TLS Secrets** — they survive its uninstall, so ingress-nginx keeps serving HTTPS while cert-manager is gone. We back up the CR objects (Issuers/Certificates) and re-apply them after reinstall; they reconcile against the surviving Secrets with no re-issuance.
4. **SUC "ghost" pods.** After each hop, SUC leaves one `Unknown`-status pod per node (orphaned when k3s restarts on the node it's upgrading). Harmless; the health check filters them out.

## Production execution

Pre-work (all validated in the playground first):
- Upgraded SUC v0.6.2 → v0.19.2 (with the ClusterRoleBinding fix).
- Removed decommissioned ghosts: the lone `longhorn-nfs-provisioner` PodSecurityPolicy (a hard blocker at v1.25), the `longhorn-nfs` StorageClass, and unused Crunchydata PGO CRDs.
- Fully decommissioned the old summerwind actions-runner-controller (namespace, CRDs, orphaned webhooks, and RBAC — carefully keeping the live new ARC's shared ClusterRole).
- Force-finalized three stuck-`Terminating` namespaces (blocked by a transient `metrics.k8s.io` discovery failure).
- Checked certificate expiries (so the cert-manager-absent window was safe) and removed cert-manager, backing up its 31 CRs.

The hops, each `task k3s:upgrade -- <version>` then watch + health-check:
- **1.21 → 1.25** ✅ (crossed the PSP / `batch/v1beta1` removals cleanly)
- **1.25 → 1.29** ✅
- **1.29 → 1.36** — this is where the two incidents happened (below)

Finally, reinstalled cert-manager **v1.18.2** and restored the 31 CRs → 29/30 certs back to `Ready` (the one exception was a pre-existing failure, see Follow-ups).

## The two production-only incidents

### 1. Kubernetes 1.36 refuses to run on cgroup v1

On the final hop, qb1's kubelet died on startup with `kubelet is configured to not run on a host using cgroup v1`. **K8s 1.36 hard-fails on cgroup v1**; Ubuntu 20.04 boots the cgroup v1 hierarchy by default. 1.25 and 1.29 had tolerated it; 1.36 made it fatal.

**Fix:** switch each node to the unified (v2) hierarchy with the `systemd.unified_cgroup_hierarchy=1` kernel cmdline flag and reboot — no OS upgrade required (kernel 5.15 + k3s 1.36's bundled containerd/runc all support cgroup v2). We did qb1 first (already down), confirmed it rejoined on 1.36, then prepared the rest before continuing.

The playground missed this because OrbStack VMs already run a modern kernel on cgroup v2.

### 2. Hyper-V disk reordering swapped a node's mounts (looked like etcd loss)

When qb2 rebooted for the cgroup-v2 change, its k3s crash-looped with `etcd cluster join failed: duplicate node name found` and an *empty* `/var/lib/rancher`. It looked like the etcd data was gone.

Root cause: `/etc/fstab` mounted the data disks by **unstable `/dev/sdX` names**, and Hyper-V had reordered the SCSI disks across the reboot — so the wrong (empty 1000G) disk got mounted at `/var/lib/rancher`, and the real 100G rancher+etcd disk landed at `/mnt/big`. The data was never lost, just mis-mounted.

**Fix:** remount the correct disk (by UUID) at `/var/lib/rancher` and rewrite fstab to use `UUID=` for the data mounts. k3s then rejoined as its *existing* etcd member (the data carried the right member ID, and the failed "fresh join" attempts had never actually altered cluster membership) — no etcd surgery, no data loss. We then UUID-pinned fstab on **all** nodes before any further reboots.

Throughout this, etcd quorum held on the other two control-plane nodes (2/3), so the cluster API stayed up and there was no service impact.

## Verification

- All 4 nodes `Ready` on **v1.36.1+k3s1**, etcd 3/3 voters.
- No unhealthy workloads; the node-local StatefulSets (MinIO, Redis) recovered onto their home nodes.
- cert-manager v1.18.2 healthy; 29/30 Certificates `Ready` against their original Secrets (no re-issuance).
- TLS Secrets were byte-identical across the whole upgrade — cert-manager's absence caused no certificate disruption.

## End-user impact

Limited, but not zero:

- **Hop 1 (1.21 → 1.25):** brief slowness and request timeouts while control-plane components and ingress pods rolled. Cleared on their own; no intervention needed.
- **qb2 disk-swap incident:** a **full services outage for its duration.** Although ingress-nginx runs on every node (DaemonSet), the public DNS names point at **qb2 specifically** — a known single point of failure — so with qb2 down, all ingress traffic was black-holed. This is independent of the upgrade and of cert-manager (the TLS Secrets were fine); it's purely the ingress SPOF.
  - A failover solution (load-balanced VIP / DNS across all ingress nodes) was **planned but never implemented**. It's slated for the **full cluster rotation in Winter 2026–2027**; until then, qb2 going down takes ingress with it.
- The deliberate cert-manager-absent window caused **no user impact** on its own — existing certs kept serving from their Secrets.

## Follow-ups (deferred, none blocking)

1. **Cluster-wide HTTP-01 cert renewal was broken — now fixed.** Surfaced via the already-expired `static/ingress-letsencrypt`. Root cause: **ingress-nginx v1.12 defaults `strict-validate-path-type: true`, which rejects cert-manager's HTTP-01 solver Ingress** for the `/.well-known/acme-challenge/...` path (the challenge Ingress is never admitted). **Pre-existing and unrelated to the K3s upgrade** — ingress-nginx v1.12.1 predates it, which is why `static` (first to hit renewal, ~April 2026) failed while the rest stayed `Ready`; all 30 certs share the issuer and would have failed as they each reached renewal. **Fix applied:** set `strict-validate-path-type: "false"` in the `ingress-nginx-controller` ConfigMap and re-triggered the Order → all 13 challenges validated, `static` renewed (now `notAfter` 2026-09-22), **30/30 certs Ready**. (The earlier SAN/DNS/rate-limit hunch was wrong — every SAN validated fine once the solver Ingress was admitted.)
2. ~~**Orphaned Longhorn PV** `pvc-2d7f2f26…`~~ — **done.** It had been stuck `Terminating` since 2022, blocked by a dead `external-attacher/driver-longhorn-io` finalizer; cleared the finalizer and it GC'd. No Longhorn PVs remain.
3. **Disk usage** on `/var/lib/rancher` — grows per upgrade hop (old images linger); prune unused images (`k3s crictl rmi --prune`). qb2's `/var/lib/rancher` is at **83% (18G free) on its 100G disk** and still needs attention; note its `/mnt/big` (1000G) sits nearly empty, so expanding/relocating the rancher volume is an option. Check qb1/qb3/qb4 too.
4. **Ubuntu 20.04 is EOL** — worth a distro upgrade as separate work (not required for K3s; the cgroup-v2 boot flag was sufficient).
5. **cert-manager ≥ v1.18 defaults `rotationPolicy: Always`** — certificates will get new private keys at their next renewal (not immediately).

## Lessons

- **Rehearse on a faithful replica** — it caught four real failure modes before prod. But know its blind spots: a playground on a different OS/kernel can't surface OS-level issues (cgroup version, disk/mount behaviour). Both prod incidents were exactly there.
- **Mount data disks by UUID, never `/dev/sdX`** — especially on Hyper-V / VM platforms that can reorder SCSI disks across reboots.
- **TLS Secrets outlive cert-manager** — that fact made the "remove it for the duration" strategy safe and turned cert-manager from a blocker into a non-event.
- **One etcd node at a time, and never tunnel your kube-API through a node you're about to reboot** (we lost API access mid-incident when the SSH tunnel ran through the node being recovered).
- **Staged hops with quorum awareness** kept a multi-version jump safe even when a node fell over mid-upgrade.

---

*See [PLAN.md](PLAN.md) for the original pre-upgrade assessment and the step-by-step procedure. The Taskfile in this repo spins up the OrbStack rehearsal cluster and runs the upgrade (against the playground by default, or production via `KUBECONFIG`).*
