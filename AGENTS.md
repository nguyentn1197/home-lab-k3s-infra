# AGENTS.md — AI Agent Instructions for home-lab-k3s-infra

This file tells AI agents (Claude, Copilot, Gemini, etc.) how to work safely and effectively with this repository.

---

## Golden Rule: GitOps Only

**Never run `kubectl apply`, `kubectl delete`, `helm install`, `helm upgrade`, or any other imperative cluster command.**

All changes to the cluster happen exclusively through Git commits. Flux CD watches this repository and reconciles the cluster state automatically. If you need to deploy something, write the YAML and commit it — Flux handles the rest.

Read-only `kubectl` commands (`kubectl get`, `kubectl describe`, `kubectl logs`) are fine for inspection.

---

## Repository Overview

This is a GitOps-managed K3s home-lab cluster. The repo has two main concerns:

1. **Cluster provisioning** (`k3s-cluster/`) — Ansible playbooks that install K3s on bare VMs.
2. **Cluster workloads** (`clusters/`, `infrastructure/`, `apps/`) — Flux CD manifests that declare what runs on the cluster.

### Directory Layout

```
home-lab-k3s-infra/
├── clusters/
│   └── homelab/              # Flux entrypoint for the homelab environment
│       ├── flux-system/      # Flux CD bootstrap manifests (auto-generated, don't edit)
│       ├── infrastructure.yaml   # Flux Kustomization: deploys infrastructure/controllers/
│       ├── configs.yaml          # Flux Kustomization: deploys infrastructure/configs/
│       └── apps.yaml             # Flux Kustomization: deploys apps/homelab/
│
├── infrastructure/
│   ├── controllers/          # Helm releases for cluster-wide controllers
│   │   ├── metallb.yaml      # Layer-2 load balancer (IP pool: 10.10.40.0-10.10.40.250)
│   │   ├── envoy-gateway.yaml # Envoy Gateway controller (LoadBalancer IP: 10.10.40.1)
│   │   ├── longhorn.yaml     # Distributed block storage
│   │   └── kustomization.yaml
│   └── configs/              # CRD instances and patches that depend on controllers
│       ├── metallb-config.yaml   # IPAddressPool + L2Advertisement
│       ├── local-path-sc-patch.yaml  # Demotes local-path StorageClass from default
│       └── kustomization.yaml
│
├── apps/
│   ├── base/                 # Environment-agnostic app manifests
│   │   ├── hello-word.yaml   # Example app (nginxdemos/hello)
│   │   └── kustomization.yaml
│   └── homelab/              # Homelab-specific overlays/additions
│       └── kustomization.yaml  # Currently just re-exports ../base
│
└── k3s-cluster/
    └── ansible/
        ├── playbook.yml
        ├── ansible.cfg
        ├── inventory/
        │   └── hosts.ini     # 3 masters (10.10.30.21-23), 3 workers (10.10.30.31-33)
        └── roles/
            ├── common/       # OS prep: packages, kernel modules, sysctl, UFW, swap
            ├── k3s-master/   # K3s server install + kube-vip manifests
            └── k3s-worker/   # K3s agent install + Longhorn disk setup
```

### Dependency Order (Flux)

```
infra-controllers  →  infra-configs               →  apps
(metallb,              (gateway-api-crds,              (hello-world HTTPRoute,
 longhorn,              gateway, longhorn-httproute,    future app HTTPRoutes)
 envoy-gateway)         metallb-config, SC patch)
```

Each layer `dependsOn` the previous one. Flux will not deploy a layer until its dependency is healthy.

---

## Cluster Facts

| Property | Value |
|----------|-------|
| Kubernetes distribution | K3s (HA, embedded etcd) |
| Control-plane VIP | `10.10.30.20` (kube-vip) |
| Master nodes | `10.10.30.21`, `.22`, `.23` |
| Worker nodes | `10.10.30.31`, `.32`, `.33` |
| GitOps tool | Flux CD v2 |
| Ingress controller | Envoy Gateway (Gateway API) |
| Ingress IP | `10.10.40.2` |
| MetalLB IP pool | `10.10.40.0 – 10.10.40.250` |
| Storage | Longhorn (default StorageClass, 2 replicas) |
| Base domain | `*.kube.local.tnndev.com` |
| TLS | Terminated upstream (no TLS in-cluster) |

---

## Naming Conventions

### Namespaces
- Infrastructure controllers use their own namespaces: `metallb-system`, `envoy-gateway-system`, `longhorn-system`.
- Applications get their own namespace named after the app: `hello-world`, `grafana`, etc.

### HTTPRoute hostnames
Pattern: `<app-name>.kube.local.tnndev.com`
Examples: `hello.kube.local.tnndev.com`, `longhorn.kube.local.tnndev.com`

### File names
- One logical resource group per file (e.g., one HelmRelease + its HelmRepository + Namespace in one file).
- Files named after the tool/app: `metallb.yaml`, `envoy-gateway.yaml`.

### Helm chart versions
- Use semver range constraints, not pinned versions: `"0.14.x"`, `"4.x"`, `"1.7.x"`.
- Flux's Helm controller will pick up patch/minor updates automatically within the range.

---

## How to Add a New Application

See `docs/contributing.md` for the full workflow. Short version:

1. Create `apps/base/<app-name>.yaml` with Namespace + Deployment/StatefulSet + Service + HTTPRoute (Gateway API) that references the shared `homelab` Gateway in `envoy-gateway-system`.
2. Add the file to `apps/base/kustomization.yaml` resources list.
3. If the app needs homelab-specific overrides, add a patch in `apps/homelab/`.
4. Commit and push — Flux reconciles within 10 minutes (or force-reconcile with `flux reconcile`).

---

## How to Add a New Infrastructure Controller

1. Create `infrastructure/controllers/<tool-name>.yaml` with Namespace + HelmRepository + HelmRelease.
2. Add the file to `infrastructure/controllers/kustomization.yaml`.
3. If the controller needs CRD instances after install, add them to `infrastructure/configs/`.
4. Update `infrastructure/configs/kustomization.yaml` to include the new config file.
5. Commit and push.

---

## Secrets Handling

**There are currently no secrets in this repository.** The cluster token in `k3s-cluster/ansible/inventory/hosts.ini` is a provisioning-time secret used only by Ansible — it is not a Kubernetes secret.

For future Kubernetes secrets:
- **Do not commit plaintext secrets.**
- Use [SOPS](https://github.com/getsops/sops) with Age encryption, or [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets).
- If SOPS is adopted, add a `.sops.yaml` config file and document the key management approach here.

---

## What Agents Should NOT Do

- ❌ Run `kubectl apply`, `kubectl delete`, `helm install`, `helm upgrade`
- ❌ Commit secrets, tokens, or credentials to any file
- ❌ Edit files under `clusters/homelab/flux-system/` (auto-generated by Flux bootstrap)
- ❌ Change the MetalLB IP pool without confirming it doesn't conflict with the home router's DHCP range
- ❌ Change the Longhorn replica count below 2 (data loss risk)
- ❌ Remove the `CriticalAddonsOnly=true:NoExecute` taint from master nodes (workloads should run on workers)
