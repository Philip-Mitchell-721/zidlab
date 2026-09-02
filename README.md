# ⛵ zidlab

A three-node bare-metal Kubernetes home lab, managed entirely with GitOps.

Everything in [`kubernetes/`](./kubernetes) is reconciled onto the cluster by Flux from this
repository. There is no click-ops — a change reaches the cluster by being committed, or it
doesn't reach the cluster at all.

## 📖 What this is

A **learning project**, first and foremost. The goal was to get hands-on with infrastructure
topics — Kubernetes, networking, storage, identity, observability — that my day job as an
application developer doesn't touch.

Some honest context, since it shapes what this repo demonstrates:

- It was **bootstrapped from [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template)**,
  which supplied the Talos configuration, the task automation, and a working networking and
  GitOps core (Flux, Cilium, Envoy Gateway, cert-manager, Cloudflare Tunnel, and friends).
  I didn't design those pieces — I inherited them working and have maintained them since.
- Most of the configuration here was **written with AI assistance**. I decided what to build
  and why, drove the troubleshooting, and run it day to day.
- What I added on top of the template: storage (Longhorn, SMB CSI), managed PostgreSQL
  (CloudNativePG), self-hosted SSO (Authentik), monitoring (Prometheus/Grafana/Alertmanager),
  and roughly thirty self-hosted applications.

Running it for months has taught me more than building it did — upgrades, failures at
inconvenient times, and the gap between "working" and "reliable."

## 🖥️ Cluster

| | |
|---|---|
| **Nodes** | 3 × bare metal, all control-plane (etcd quorum, tolerates losing one) |
| **OS** | [Talos Linux](https://www.talos.dev) — immutable, no SSH, no shell, configured over a gRPC API |
| **Kubernetes** | Managed via [talhelper](https://budimanjojo.github.io/talhelper/latest/); node config is declarative and patch-composed |
| **Networking** | Static IPs, floating VIP for the API server, Cilium as CNI with kube-proxy disabled |
| **Storage** | NVMe per node (Longhorn) + an SMB network share for bulk media |

Node addresses and the domain live in `cluster.yaml` / `nodes.yaml`, which are gitignored —
so are all key material and the rendered Talos configs.

## 📦 What's running

**Platform**

| Component | Role |
|---|---|
| [Flux](https://fluxcd.io) | GitOps reconciliation — this repo is the source of truth |
| [Cilium](https://cilium.io) | eBPF CNI, kube-proxy replacement, L2 announcements for LAN LoadBalancer IPs |
| [Envoy Gateway](https://gateway.envoyproxy.io) | Gateway API ingress, split into internal (LAN) and external listeners |
| [cloudflared](https://github.com/cloudflare/cloudflared) | Outbound tunnel — public services with no inbound ports open |
| [external-dns](https://github.com/kubernetes-sigs/external-dns) + [k8s-gateway](https://github.com/k8s-gateway/k8s_gateway) | Public and split-horizon LAN DNS |
| [cert-manager](https://cert-manager.io) | Wildcard TLS via Let's Encrypt DNS-01 |
| [Longhorn](https://longhorn.io) | Replicated block storage, the default StorageClass |
| [csi-driver-smb](https://github.com/kubernetes-csi/csi-driver-smb) | RWX mounts of the bulk media share |
| [CloudNativePG](https://cloudnative-pg.io) | Operator-managed PostgreSQL, one cluster per app |
| [Authentik](https://goauthentik.io) | Self-hosted OIDC provider — single sign-on across the lab |
| [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts) | Prometheus, Grafana, Alertmanager |
| [Spegel](https://github.com/spegel-org/spegel) · [Reloader](https://github.com/stakater/Reloader) | P2P image mirror · restart-on-config-change |
| [SOPS](https://github.com/getsops/sops) + age | Secrets encrypted at rest in Git |

**Applications** — ~30 across `default`, `media`, and `network`:

- **Media** — Plex/Jellyfin (on a separate box, see below), Sonarr, Radarr, Bazarr, Prowlarr,
  SABnzbd, Seerr, Tautulli, Recyclarr, Audiobookshelf, Wizarr
- **Home** — Immich (photos), Mealie (recipes), Actual (budgeting), Karakeep (bookmarks),
  Donetick (chores), Rallly (scheduling), LubeLogger (vehicles), Kimai (time tracking)
- **Infrastructure** — AdGuard Home, ntfy + Apprise (notifications), Homepage (dashboard)

### A deliberate exception

Plex, Jellyfin and AdGuard run as **native systemd services on a separate small Linux box**,
not in the cluster. Hardware transcoding is simpler outside Kubernetes, and DNS needs to keep
resolving when the cluster is down. It's the clearest case here of deciding *not* to use the
platform.

## 📂 Repository layout

```
kubernetes/
├── apps/              # one directory per app, grouped by namespace
│   └── <namespace>/
│       └── <app>/
│           ├── ks.yaml           # Flux Kustomization — registers the app
│           └── app/
│               ├── helmrelease.yaml
│               ├── kustomization.yaml
│               └── *.sops.yaml   # encrypted secrets
├── components/        # reusable Kustomize components
└── flux/              # Flux's own bootstrap configuration
talos/                 # Talos machine config + patches (talhelper)
.taskfiles/            # task automation
.github/workflows/     # CI — manifest diffing, linting
```

Adding an app means creating that folder and listing it in the namespace's
`kustomization.yaml`. Flux picks it up on the next reconcile.

## 🔄 How a change ships

```
edit YAML → PR → CI renders + diffs the manifests → merge → webhook → Flux applies
```

CI runs [flux-local](https://github.com/allenporter/flux-local), which renders every
HelmRelease and Kustomization and diffs it against the branch — so a broken manifest fails
the PR rather than the cluster. [Renovate](https://www.mend.io/renovate) opens dependency
PRs on a weekly schedule.

---

# 🛠️ Operations

Kept from the upstream template, adapted to this cluster. Commands assume
[mise](https://mise.jdx.dev) is activated (`mise activate`) so `task`, `talosctl`, `flux`
and friends are on `PATH`.

## 🔁 Force a reconcile

Flux syncs on a webhook, with an hourly poll as a fallback. To push it:

```sh
task reconcile
```

## ⚙️ Updating Talos node configuration

> [!TIP]
> Update `talconfig.yaml` and any patches first. Some changes need **both** a config apply
> *and* a Talos upgrade to take effect.

```sh
# (Re)generate the Talos config
task talos:generate-config

# Apply it to one node
task talos:apply-node IP=<node-ip> MODE=auto
```

## ⬆️ Updating Talos and Kubernetes versions

> [!TIP]
> Set `talosVersion` / `kubernetesVersion` in `talenv.yaml` first — Renovate usually opens
> this PR for you.

```sh
# One node at a time. Wait for Ready before moving to the next —
# three nodes is a bare etcd quorum.
task talos:upgrade-node IP=<node-ip>
```

```sh
# Kubernetes itself, cluster-wide
task talos:upgrade-k8s
```

## ➕ Adding a node

An **odd number** of control-plane nodes is required for quorum. No re-bootstrap needed:

1. Boot the new node into maintenance mode.
2. Grab its disk and MAC address:

   ```sh
   talosctl get disks -n <ip> --insecure
   talosctl get links -n <ip> --insecure
   ```

3. Add the node to `talconfig.yaml` ([talhelper docs](https://budimanjojo.github.io/talhelper/latest/)).
4. Generate and apply:

   ```sh
   task talos:generate-config
   task talos:apply-node IP=<node-ip>
   ```

It joins automatically and workloads schedule once it reports Ready.

## 🐛 Debugging

General flow when a workload isn't showing up, or a pod is `CrashLoopBackOff` / `Pending`:

1. **Are the Flux resources healthy and current?**

   ```sh
   flux get sources git -A
   flux get ks -A
   flux get hr -A
   ```

2. **Does the pod exist?**

   ```sh
   kubectl -n <namespace> get pods -o wide
   ```

3. **What do its logs say?**

   ```sh
   kubectl -n <namespace> logs <pod-name> -f
   ```

4. **Describe the resource** — mount failures and scheduling problems surface here, not in
   the logs:

   ```sh
   kubectl -n <namespace> describe <resource> <name>
   ```

5. **Check namespace events:**

   ```sh
   kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
   ```

For node-level problems, Talos has no SSH — use the API:

```sh
talosctl --nodes <node-ip> dmesg
talosctl --nodes <node-ip> services
talosctl --nodes <node-ip> health
talosctl --nodes <node-ip> dashboard
```

## 💥 Reset

Destroys the cluster and returns the nodes to maintenance mode. Prompts first; no undo.

```sh
task talos:reset
```

---

## 🙏 Credits

Built from [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) — the
Talos configuration, task automation, and the initial networking and GitOps stack all come
from there. It's an excellent project and worth a look if you're starting your own cluster.

The [Home Operations](https://discord.gg/home-operations) Discord community is where most of
the answers came from.

## 📄 License

MIT — see [LICENSE](./LICENSE). Original copyright © 2025 onedr0p.
