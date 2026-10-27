# Operational Gotchas & Incident History

Hard-won lessons from real incidents on this cluster. Read this before touching node provisioning, DNS, or node sizing — each of these has already caused a real outage once.

---

## DNS: the `local.tnndev.com` search-domain landmine

**Symptom:** pods can't reach the public internet (GitHub, Helm chart repos, Cloudflare, etc.) — TLS handshakes fail with `connection reset by peer` / `SSL alert 80 (internal error)`, even though `nslookup`/`wget` from a shell sometimes look fine and DNS resolution for internal/cluster names works perfectly.

**Root cause:** the LAN's DHCP server hands every node a search domain of `local.tnndev.com` (this zone, served by the Technitium DNS server, deliberately has a wildcard so `*.kube.local.tnndev.com` ingress hostnames resolve to Caddy — that wildcard is **correct and must not be touched**). Kubernetes' default pod DNS policy (`ClusterFirst`) always sets `options ndots:5`, and kubelet appends the node's own search domains (including `local.tnndev.com`) to every pod's `/etc/resolv.conf`. For any hostname with fewer than 5 dots — i.e. basically every real-world hostname, like `github.com` — the resolver tries `github.com.local.tnndev.com` *before* trying `github.com` itself. Because `local.tnndev.com` is wildcarded, that bogus name resolves successfully (to Caddy's IP), so the resolver stops there and never tries the real name. Caddy then gets a TLS SNI/Host it doesn't recognize and resets the connection.

Busybox tools (`nslookup`, `wget`) can look deceptively healthy while debugging this because their simplified resolvers don't always respect `ndots`/`search` the way a real `getaddrinfo()` call (used by Go binaries, `curl`, `nc`, etc.) does.

**Fix:** point k3s's kubelet DNS config at a minimal resolv.conf that has the real nameserver but **no search domain**, instead of the host's DHCP-supplied one. This only changes what kubelet hands to *pods* — the host OS's own DNS, Technitium, and the wildcard zone are completely untouched, so nothing else on the LAN is affected.

On every node (`/etc/k3s-resolv.conf`):
```
nameserver 10.10.20.203
```

In `/etc/rancher/k3s/config.yaml`:
```yaml
resolv-conf: "/etc/k3s-resolv.conf"
```
(`k3s agent --help` / `k3s server --help` document this as the `--resolv-conf` flag.) Restart `k3s`/`k3s-agent` to apply. Any pod that was already running when you make this change keeps its old `resolv.conf` — you must recreate it (delete the pod; a Deployment/DaemonSet recreates it fresh) to pick up the fix.

**Don't** "fix" this by narrowing the wildcard on `local.tnndev.com` — that zone is serving other domains on the LAN besides the cluster's `*.kube.local.tnndev.com` slice, and narrowing it would break those.

---

## `multipathd` fighting Longhorn over its own disks

**Symptom:** a pod using a Longhorn PVC gets stuck with repeating kubelet events:
```
MountVolume.MountDevice failed ... mount failed: exit status 32
... /dev/longhorn/pvc-xxxx already mounted or mount point busy.
```
even though the volume isn't mounted anywhere, Longhorn reports it healthy, and deleting/recreating the pod, the CSI plugin pod, and the `VolumeAttachment` all reproduce the exact same error.

**Root cause:** `multipathd` (installed on the base Ubuntu image, not something Longhorn needs or uses) auto-claims the iSCSI-backed block devices Longhorn presents as raw `/dev/sdX` disks and wraps them in `dm-multipath` maps (visible as `mpathX` devices in `lsblk`/`multipath -ll`). Once `multipathd` has exclusive claim on the device, Longhorn's own attempt to mount it directly fails as "busy" — this can happen intermittently right as a new iSCSI session is established, so `multipath -ll` may look clean moments before or after the failure.

Confirm with `multipath -ll` on the node — if `pvc-xxxx`'s underlying `/dev/sdX` shows up wrapped in an `mpathX` map, this is the cause. Longhorn itself has a dedicated node condition for this (`nodes.longhorn.io <node> -o jsonpath='{.status.conditions[?(@.type=="Multipathd")]}'`), though the condition reports `False` just from `multipathd` *running*, independent of whether the blacklist fix below is applied — treat it as informational, not authoritative.

**Fix:** blacklist `sd*` devices from `multipathd` so it never claims them:
```bash
cat <<'EOF' >> /etc/multipath.conf
blacklist {
    devnode "^sd[a-z]+"
}
EOF
systemctl restart multipathd
multipath -F
```
Baked into the Ansible `common` role (`k3s-cluster/ansible/roles/common/tasks/main.yml`) so every new/rebuilt node gets this automatically.

---

## Longhorn rebuild storm after a node reboot

**Symptom:** shortly after a worker comes back from a reboot (or drain + restart), CNPG replicas elsewhere in the cluster start crash-looping with `could not open file "pg_wal/...": Input/output error` or `chmod .../pgdata: input/output error`, while the primary keeps running. On the node hosting the failing pod, `dmesg` shows `Buffer I/O error on dev sdX`, `Detected aborted journal`, and `Remounting filesystem read-only`. Longhorn still reports the volumes `attached` / `healthy` or `degraded`, so the data is intact — it's the live ext4 mount that's broken.

**Root cause:** while the node was down, every replica it held went stale. When it returns, Longhorn rebuilds all of them onto it in parallel — up to `concurrent-replica-rebuild-per-node-limit` (default **5**) at a time. On the slower hosts (`proxmox1` is laptop-class hardware), that saturates the returning node's disk (IO pressure `full` ~60%+, load ~20 on 4 CPUs): rebuilds time out (`context deadline exceeded`, `FailedRebuilding`) and retry, and the heavy sync load on the *source* replicas pushes their engines past `engineReplicaTimeout`. That surfaces as EIO on the live block device, ext4 aborts its journal and flips read-only, and Postgres can't start. A container restart reuses the same broken mount, so the pod just crash-loops.

Check what's rebuilding and how hard the destination node is hurting:
```bash
kubectl -n longhorn-system get events --sort-by=.lastTimestamp | grep -E 'Rebuilding|FailedRebuilding'
ssh <node> 'cat /proc/loadavg /proc/pressure/io'
```

**Fix:**
1. Throttle rebuilds to one per node — now set permanently via `defaultSettings.concurrentReplicaRebuildPerNodeLimit: 1` in `infrastructure/controllers/longhorn.yaml`. (Live, before Flux catches up: `kubectl -n longhorn-system patch settings.longhorn.io concurrent-replica-rebuild-per-node-limit --type merge -p '{"value":"1"}'`.)
2. Cordon the overloaded node until its rebuilds finish, so nothing new lands on it.
3. **Delete** (not restart) the crash-looping replica pods. Deleting forces kubelet to unmount the read-only filesystem, and the remount runs fsck. CNPG re-syncs them from the primary. Leave the primary alone.
4. Uncordon once no rebuilds target the node and its IO pressure is back to normal (`full avg60` well under 15%). A large volume (e.g. the 50 GiB Prometheus PVC) can keep the node busy for a while even at one rebuild at a time — that's expected and safe.

---

## Per-node resource reservations ("performance gate")

**Incident:** `worker-3` (the smallest node: 2 CPU / 4 GB) was driven to zero free memory and severe CPU/IO pressure after ~50 pods — including the *entire* Flux control plane, MetalLB, and the CNPG operator — piled onto it. Almost none of these had any resource `requests` set, so the scheduler had no signal that the node was actually full and kept stacking more on it.

**Fix, two parts:**

1. **Kubelet reservations per node**, scaled to each node's real capacity, so the OS/kubelet always keep guaranteed headroom and can never be starved to literal zero:
   ```yaml
   # /etc/rancher/k3s/config.yaml
   kubelet-arg:
     - "system-reserved=cpu=<X>,memory=<Y>"
     - "kube-reserved=cpu=<X>,memory=<Y>"
     - "eviction-hard=memory.available<<Z>"
   ```
   Current values live in `k3s-cluster/ansible/inventory/hosts.ini` as per-host vars (`kube_reserved_cpu`, `kube_reserved_memory`, `eviction_hard_memory`) and are applied by the `common` role.

2. **Resource requests/limits on every infra controller that had none** — Longhorn, MetalLB, the CNPG operator, Loki, and `external-snapshotter`'s `snapshot-controller` all shipped with zero requests. Fixed via each `HelmRelease`'s `values:` where the chart exposes a `resources:` key — and via Flux `postRenderers`/Kustomization `patches` where it doesn't (Longhorn's chart has **no** values knob for its own component resources at all; `csi-snapshot-controller` isn't a Helm chart, it's raw manifests from `kubernetes-csi/external-snapshotter`).

Before copying a chart's `resources:` key path from memory or a quick guess: **check the actual chart** (`helm show values <repo>/<chart> --version <x>`) first. Several charts nest `resources` per-container (e.g. `csi-driver-smb`'s `controller.resources.{csiProvisioner,csiResizer,smb,livenessProbe}`, each with its own chart-supplied defaults already) rather than as one flat block — setting a flat override there is a silent no-op at best.

---

## Worker nodes are not identical — check before assuming

| Node | CPU | Memory | Proxmox host |
|------|-----|--------|--------------|
| `k3s-worker-1` | 4 | 4 GB | proxmox3 |
| `k3s-worker-2` | 4 | 8 GB | proxmox1 |
| `k3s-worker-3` | 2 | 4 GB | proxmox2 |

`worker-3` in particular has half the CPU of the other two — it's the node most likely to show pressure first under any cluster-wide load spike. Don't assume uniform capacity when reasoning about scheduling or diagnosing a single node's symptoms.

---

## Proxmox VM CPU type can silently hide host CPU features

**Symptom:** a specific container binary crashes outright with something like `This program can only be run on AMD64 processors with v2 microarchitecture support`, and only on one particular node — `grep -o 'avx2\|bmi2\|sse4_2' /proc/cpuinfo` on that node comes back empty.

**Root cause:** a VM's `cpu:` field in Proxmox defaults to `None`, which resolves to a conservative baseline model (`kvm64`/`qemu64`) that deliberately strips modern instruction-set extensions (AVX2, BMI2, SSE4.2, i.e. x86-64-v2) for live-migration safety across potentially dissimilar hardware — **even when the underlying physical CPU fully supports them** (confirmed via `pvesh get /nodes/<node>/status` → `cpuinfo.flags` on the Proxmox host itself).

**Fix:** set an explicit `cpu:` type on the VM (`host` to pass through the physical CPU 1:1, simplest for a small homelab cluster where live-migrating between dissimilar hosts isn't a design goal) via the Proxmox UI or API, then **fully stop and start** the VM — a CPU model change does not take effect on a soft reboot, only a cold start. Rebuilding/re-cloning a VM from a template does **not** preserve a previous VM's `cpu:` override; re-check and re-apply it after any VM rebuild.

---

## k3s worker install: version skew and a dead node-label flag

Two gotchas hit together when rebuilding `k3s-worker-2` from scratch:

- `curl -sfL https://get.k3s.io | sh -s - agent` with no `INSTALL_K3S_VERSION` pulls whatever is currently "stable" — which can easily be **newer** than the control plane, violating Kubernetes' version-skew policy (a kubelet must never be newer than the API server). Always pin `INSTALL_K3S_VERSION` to match the masters' current version (`kubectl get nodes` to check). The Ansible `k3s-worker` role now does this via the `k3s_version` var in `hosts.ini` — keep it in sync with the masters.
- The old role passed `--node-label "node-role.kubernetes.io/worker=true"` to the agent install. Kubelet's `--node-labels` flag has always rejected any key in the reserved `kubernetes.io`/`k8s.io` namespace that isn't on its allow-list — older k3s/kubelet silently dropped the flag (none of the existing workers actually carry this label, despite it being in the install command for who knows how long), but newer versions make it a hard `kubelet exited: failed to validate kubelet flags` failure instead. Removed.

A stale `k3s-worker-2.node-password.k3s` Secret (and the old `Node` object) left over from the *previous* VM also blocks a rebuilt node with the same hostname from rejoining (`Node password rejected, duplicate hostname...`) — delete both (`kubectl -n kube-system delete secret <node>.node-password.k3s` and `kubectl delete node <node>`) before the fresh agent can register.
