# home-lab-k3s-infra

GitOps repository for my home-lab Kubernetes cluster — a 3-master, 3-worker K3s HA cluster running on Proxmox VMs, managed by Flux CD.

---

## What's Running

| Layer | Tool | Purpose |
|-------|------|---------|
| Cluster provisioning | Ansible | Installs K3s on bare Ubuntu VMs |
| High-availability VIP | kube-vip | Provides a stable API server IP (`10.10.30.20`) |
| GitOps | Flux CD v2 | Reconciles this repo → cluster state |
| Load balancer | MetalLB | Assigns IPs from `10.10.40.1 – 10.10.40.250` to LoadBalancer services |
| Gateway / ingress | Envoy Gateway | Gateway API HTTP routing at `10.10.40.1` |
| Storage | Longhorn | Replicated block storage (default StorageClass) |
| Databases | CloudNativePG | PostgreSQL operator with Longhorn CSI snapshot backups |
| Secrets operator | Infisical Secrets Operator | Syncs external secrets into Kubernetes Secrets |
| Apps | hello-world, authentik | Example nginx app and Authentik identity provider |

---

## Cluster Topology

```
Proxmox host
├── k3s-master-1  10.10.30.21  (control plane + etcd)
├── k3s-master-2  10.10.30.22  (control plane + etcd)
├── k3s-master-3  10.10.30.23  (control plane + etcd)
│         └── VIP: 10.10.30.20  (kube-vip)
├── k3s-worker-1  10.10.30.31  (workloads + Longhorn disk)
├── k3s-worker-2  10.10.30.32  (workloads + Longhorn disk)
└── k3s-worker-3  10.10.30.33  (workloads + Longhorn disk)
```

Masters have a `CriticalAddonsOnly=true:NoExecute` taint — regular workloads only run on workers.

---

## Repository Layout

```
home-lab-k3s-infra/
├── clusters/homelab/         # Flux entrypoint — declares what gets deployed
├── infrastructure/
│   ├── controllers/          # Helm releases and controller sources
│   └── configs/              # CRD instances, snapshot classes, routes, secrets, SC patch
├── apps/
│   ├── base/                 # App manifests (environment-agnostic)
│   └── homelab/              # Homelab overlays
└── k3s-cluster/ansible/      # Ansible playbooks + roles for node provisioning
```

See `AGENTS.md` for a detailed breakdown and `docs/contributing.md` for how to add apps or infrastructure.

---

## Flux Reconciliation Order

```
infra-controllers → infra-configs → apps
```

Flux enforces this dependency chain. Each layer waits for the previous one to be healthy before deploying.

### Current Flux Layers

| Layer | Path | Resources |
|-------|------|-----------|
| `infra-controllers` | `infrastructure/controllers/` | MetalLB, Longhorn, Envoy Gateway, Infisical Secrets Operator, CloudNativePG, external-snapshotter Flux source |
| `infra-configs` | `infrastructure/configs/` | Shared `homelab` Gateway, MetalLB L2 pool, Longhorn HTTPRoute, Longhorn backup secret sync, Longhorn backup `VolumeSnapshotClass`, CNPG backup retention, local-path StorageClass patch, Infisical smoke test |
| `apps` | `apps/homelab/` | Homelab overlay for `apps/base/`; currently deploys `hello-world`, Authentik, and Authentik's PostgreSQL cluster |

---

## Quick Start

### 1. Provision nodes with Ansible

```bash
cd k3s-cluster/ansible

# Phase 1: Prepare all nodes (OS, packages, kernel settings)
ansible-playbook playbook.yml -i inventory/hosts.ini --tags common

# Phase 2: Install masters (sequential — order matters for etcd init)
ansible-playbook playbook.yml -i inventory/hosts.ini --tags k3s-master

# Phase 3: Install workers (parallel)
ansible-playbook playbook.yml -i inventory/hosts.ini --tags k3s-worker
```

> **Note:** Update `inventory/hosts.ini` with your node IPs and generate a fresh `k3s_token` before running:
> ```bash
> openssl rand -hex 32
> ```

### 2. Bootstrap Flux

```bash
export GITHUB_TOKEN=<your-token>
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=home-lab-k3s-infra \
  --branch=main \
  --path=clusters/homelab \
  --personal
```

### 3. Verify

```bash
# Watch Flux reconcile
flux get kustomizations --watch

# Check all resources (read-only)
kubectl get all -A
```

---

## Adding Apps

See `docs/contributing.md` — the short version:

1. Add a YAML file to `apps/base/<app-name>.yaml`
2. Register it in `apps/base/kustomization.yaml`
3. Commit and push — Flux deploys within 10 minutes

---

## PostgreSQL Notes

- CloudNativePG is installed as a cluster-wide operator in `cnpg-system`.
- CSI snapshot support is reconciled from the upstream `kubernetes-csi/external-snapshotter` repo at tag `v7.0.2`.
- Longhorn backup snapshots use `VolumeSnapshotClass/longhorn-backup-vsc`.
- Reusable examples and restore notes live in `docs/cloudnative-pg.md`.

---

## Authentik Notes

- Authentik is deployed from `apps/base/authentik/authentik.yaml`.
- The Helm chart is pinned to the `2026.2.x` release line.
- It uses the CNPG service `authentik-db-rw.postgres-authentik.svc.cluster.local`.
- The UI is exposed through Gateway API at `authentik.kube.local.tnndev.com`.
- Required Infisical keys:
  - `/authentik/app`: `AUTHENTIK_SECRET_KEY`
  - `/authentik/postgres`: `AUTHENTIK_POSTGRES_USERNAME`, `AUTHENTIK_POSTGRES_PASSWORD`
- Optional chart values are listed, commented out, in `apps/base/authentik/authentik-values-options.yaml`.

---

## Networking Notes

- TLS is terminated **upstream** (by a reverse proxy or router) — Envoy Gateway handles plain HTTP inside the cluster.
- The shared Gateway is `envoy-gateway-system/homelab`, with a wildcard listener for `*.kube.local.tnndev.com`.
- Envoy Gateway is assigned `10.10.40.1` through MetalLB.
- MetalLB IP pool: `10.10.40.1 – 10.10.40.250`. Make sure this range doesn't overlap with your router's DHCP range.
- Current HTTPRoutes:
  - `hello.kube.local.tnndev.com` → `hello-world/hello-world`
  - `longhorn.kube.local.tnndev.com` → `longhorn-system/longhorn-frontend`

## Secrets Notes

- The Infisical Secrets Operator is installed from `infrastructure/controllers/infisical.yaml`.
- `infrastructure/configs/longhorn-backup.yaml` syncs CIFS backup credentials into `longhorn-system/longhorn-backup-cifs` for Longhorn backups.
- `infrastructure/configs/infisical-test.yaml` is a smoke-test sync for validating the operator and can be removed once it is no longer needed.
- Do not commit plaintext secrets. Bootstrap credentials such as `infisical-universal-auth` must be created out-of-band.
