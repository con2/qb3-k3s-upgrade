# node-maintenance

Host-level (systemd) maintenance units for the `qb` nodes — applied directly on each Hyper-V VM, not via Kubernetes.

## k3s-image-prune (weekly image GC)

Background: kubelet's built-in image GC only trims the image filesystem down to ~80% (its low threshold), so `/var/lib/rancher` rides near 80% on busy nodes. This timer runs a full `crictl rmi --prune` weekly, which reclaims *all* unused image layers (nodes drop to ~40–55%), giving much more headroom than kubelet GC alone — the interim measure until the Winter 2026–2027 cluster rotation. See [../WRITEUP.md](../WRITEUP.md) follow-ups.

Install on **every** node (server and agent):
```bash
sudo cp k3s-image-prune.service k3s-image-prune.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now k3s-image-prune.timer

# verify
systemctl list-timers k3s-image-prune.timer
# run once now to confirm it works:
sudo systemctl start k3s-image-prune.service
journalctl -u k3s-image-prune.service --no-pager | tail
```

Notes:
- `k3s crictl` works on both server and agent nodes (the `k3s` binary and the containerd socket are present on all of them).
- The unit runs as root, niced and with idle I/O priority so it never starves etcd/workload I/O on the shared SSD.
- Complement with a disk-free alert on `/var/lib/rancher` (< ~15 GB free) — image GC can't reclaim non-image growth (etcd DB, local-path PVs, logs), so an alert catches what the prune can't.
