# k3s-upgrade

Tooling and documentation for upgrading the `qb` K3s cluster from v1.21.6 to v1.36.1, rehearsed on a disposable OrbStack cluster before running against production.

- **[WRITEUP.md](WRITEUP.md)** — what happened: the approach, the production run, the two incidents (cgroup v1→v2, Azure disk-mount swap) and how they were handled, the outcome, and follow-ups. Start here.
- **[PLAN.md](PLAN.md)** — the original pre-upgrade assessment and the step-by-step procedure (pre-work, the staged hops, cert-manager handling, risk register).

## The repo

A [Taskfile](Taskfile.yaml) spins up a multi-node OrbStack VM cluster that mirrors the production starting state (K3s v1.21.6 + cert-manager v1.0.3 + SUC v0.6.2) and runs the upgrade through it. Every task defaults to the local playground but honours an exported `KUBECONFIG`, so the same commands run against production:

```bash
task up                              # create the rehearsal cluster
task upgrade:suc                     # upgrade system-upgrade-controller
task cert-manager:remove             # back up CRs + uninstall cert-manager
task k3s:upgrade -- v1.25.16+k3s4    # hop (repeat for 1.29, 1.36)
task cert-manager:install -- v1.18.2 # reinstall cert-manager + restore CRs

# against production:
KUBECONFIG=/path/to/qb task k3s:upgrade -- v1.25.16+k3s4
```

**Status:** ✅ Production upgraded to v1.36.1+k3s1 on 2026-06-24.
