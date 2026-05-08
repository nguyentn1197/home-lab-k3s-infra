# Repo Agentic Setup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Configure this K3s home-lab GitOps repository for effective AI-agent collaboration by adding AGENTS.md, a comprehensive README, opencode.json, and contributing workflow documentation.

**Architecture:** Four new files are added to the repo root and `.opencode/` directory. No existing files are modified. All content is documentation/configuration only — no Kubernetes manifests are changed.

**Tech Stack:** Markdown, JSON (opencode.json), GitOps (Flux CD v2), K3s, Ansible, Kustomize, Helm

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `AGENTS.md` | Create | AI agent instructions: repo structure, conventions, GitOps-only rule, how to add apps/infra, secrets |
| `README.md` | Overwrite (currently empty) | Human-readable overview of the cluster, tech stack, directory layout, and quick-start |
| `docs/contributing.md` | Create | Step-by-step workflows for adding apps, infrastructure controllers, and Ansible roles |
| `.opencode/opencode.json` | Create | opencode tool permissions and agent settings |

---

### Task 1: Create `.opencode/opencode.json`

**Files:**
- Create: `.opencode/opencode.json`

- [ ] **Step 1: Create the `.opencode/` directory and `opencode.json`**

Create `.opencode/opencode.json` with the following content:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "theme": "opencode",
  "model": "anthropic/claude-sonnet-4-5",
  "autoshare": false,
  "permissions": {
    "bash": [
      { "permission": "allow", "pattern": "git status *" },
      { "permission": "allow", "pattern": "git diff *" },
      { "permission": "allow", "pattern": "git log *" },
      { "permission": "allow", "pattern": "git show *" },
      { "permission": "allow", "pattern": "git branch *" },
      { "permission": "allow", "pattern": "git ls-files *" },
      { "permission": "allow", "pattern": "git blame *" },
      { "permission": "allow", "pattern": "git rev-parse *" },
      { "permission": "allow", "pattern": "git remote *" },
      { "permission": "allow", "pattern": "git fetch *" },
      { "permission": "ask",   "pattern": "git stash *" },
      { "permission": "ask",   "pattern": "git pull *" },
      { "permission": "ask",   "pattern": "git checkout *" },
      { "permission": "ask",   "pattern": "git switch *" },
      { "permission": "ask",   "pattern": "git merge *" },
      { "permission": "ask",   "pattern": "git rebase *" },
      { "permission": "ask",   "pattern": "git cherry-pick *" },
      { "permission": "ask",   "pattern": "git add *" },
      { "permission": "ask",   "pattern": "git commit *" },
      { "permission": "ask",   "pattern": "git tag *" },
      { "permission": "deny",  "pattern": "git push *" },
      { "permission": "deny",  "pattern": "git reset --hard *" },
      { "permission": "deny",  "pattern": "git clean *" },
      { "permission": "deny",  "pattern": "git push --force *" },
      { "permission": "deny",  "pattern": "kubectl apply *" },
      { "permission": "deny",  "pattern": "kubectl delete *" },
      { "permission": "deny",  "pattern": "kubectl create *" },
      { "permission": "deny",  "pattern": "kubectl exec *" },
      { "permission": "deny",  "pattern": "kubectl edit *" },
      { "permission": "deny",  "pattern": "kubectl patch *" },
      { "permission": "deny",  "pattern": "kubectl scale *" },
      { "permission": "allow", "pattern": "kubectl get *" },
      { "permission": "allow", "pattern": "kubectl describe *" },
      { "permission": "allow", "pattern": "kubectl logs *" },
      { "permission": "allow", "pattern": "kubectl top *" },
      { "permission": "allow", "pattern": "kubectl config *" },
      { "permission": "allow", "pattern": "kubectl cluster-info *" },
      { "permission": "allow", "pattern": "kubectl version *" },
      { "permission": "allow", "pattern": "kubectl api-resources *" },
      { "permission": "deny",  "pattern": "helm install *" },
      { "permission": "deny",  "pattern": "helm upgrade *" },
      { "permission": "deny",  "pattern": "helm delete *" },
      { "permission": "deny",  "pattern": "helm uninstall *" },
      { "permission": "allow", "pattern": "helm list *" },
      { "permission": "allow", "pattern": "helm status *" },
      { "permission": "allow", "pattern": "helm history *" },
      { "permission": "allow", "pattern": "helm get *" },
      { "permission": "allow", "pattern": "cat *" },
      { "permission": "allow", "pattern": "head *" },
      { "permission": "allow", "pattern": "tail *" },
      { "permission": "allow", "pattern": "grep *" },
      { "permission": "allow", "pattern": "rg *" },
      { "permission": "allow", "pattern": "ls *" },
      { "permission": "allow", "pattern": "tree *" },
      { "permission": "allow", "pattern": "pwd" },
      { "permission": "allow", "pattern": "wc *" },
      { "permission": "allow", "pattern": "find *" },
      { "permission": "allow", "pattern": "fd *" },
      { "permission": "allow", "pattern": "jq *" },
      { "permission": "allow", "pattern": "yq *" },
      { "permission": "allow", "pattern": "diff *" },
      { "permission": "allow", "pattern": "file *" },
      { "permission": "allow", "pattern": "stat *" },
      { "permission": "allow", "pattern": "echo *" },
      { "permission": "allow", "pattern": "python *" },
      { "permission": "allow", "pattern": "python3 *" },
      { "permission": "allow", "pattern": "node *" },
      { "permission": "deny",  "pattern": "curl *" },
      { "permission": "deny",  "pattern": "wget *" },
      { "permission": "deny",  "pattern": "ssh *" },
      { "permission": "deny",  "pattern": "sudo *" },
      { "permission": "deny",  "pattern": "rm *" },
      { "permission": "deny",  "pattern": "*" }
    ]
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add .opencode/opencode.json
git commit -m "chore: add opencode.json with agent tool permissions"
```

---

### Task 2: Create `AGENTS.md`

**Files:**
- Create: `AGENTS.md`

- [ ] **Step 1: Create `AGENTS.md`**

Create `AGENTS.md` at the repo root with the following content:

```markdown
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
│   │   ├── ingress-nginx.yaml # Ingress controller (LoadBalancer IP: 10.10.40.1)
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
infra-controllers  →  infra-configs  →  apps
(metallb, nginx,        (metallb IP        (hello-world,
 longhorn)               pool, SC patch)    future apps)
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
| Ingress controller | ingress-nginx |
| Ingress IP | `10.10.40.1` |
| MetalLB IP pool | `10.10.40.0 – 10.10.40.250` |
| Storage | Longhorn (default StorageClass, 2 replicas) |
| Base domain | `*.kube.local.tnndev.com` |
| TLS | Terminated upstream (no TLS in-cluster) |

---

## Naming Conventions

### Namespaces
- Infrastructure controllers use their own namespaces: `metallb-system`, `ingress-nginx`, `longhorn-system`.
- Applications get their own namespace named after the app: `hello-world`, `grafana`, etc.

### Ingress hostnames
Pattern: `<app-name>.kube.local.tnndev.com`
Examples: `hello.kube.local.tnndev.com`, `longhorn.kube.local.tnndev.com`

### File names
- One logical resource group per file (e.g., one HelmRelease + its HelmRepository + Namespace in one file).
- Files named after the tool/app: `metallb.yaml`, `ingress-nginx.yaml`.

### Helm chart versions
- Use semver range constraints, not pinned versions: `"0.14.x"`, `"4.x"`, `"1.7.x"`.
- Flux's Helm controller will pick up patch/minor updates automatically within the range.

---

## How to Add a New Application

See `docs/contributing.md` for the full workflow. Short version:

1. Create `apps/base/<app-name>.yaml` with Namespace + Deployment/StatefulSet + Service + Ingress.
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
```

- [ ] **Step 2: Commit**

```bash
git add AGENTS.md
git commit -m "docs: add AGENTS.md with AI agent instructions and repo conventions"
```

---

### Task 3: Write `README.md`

**Files:**
- Modify: `README.md` (currently empty — overwrite entirely)

- [ ] **Step 1: Write the README**

Overwrite `README.md` with the following content:

```markdown
# home-lab-k3s-infra

GitOps repository for my home-lab Kubernetes cluster — a 3-master, 3-worker K3s HA cluster running on Proxmox VMs, managed by Flux CD.

---

## What's Running

| Layer | Tool | Purpose |
|-------|------|---------|
| Cluster provisioning | Ansible | Installs K3s on bare Ubuntu VMs |
| High-availability VIP | kube-vip | Provides a stable API server IP (`10.10.30.20`) |
| GitOps | Flux CD v2 | Reconciles this repo → cluster state |
| Load balancer | MetalLB | Assigns IPs from `10.10.40.0/24` to LoadBalancer services |
| Ingress | ingress-nginx | HTTP routing at `10.10.40.1` |
| Storage | Longhorn | Replicated block storage (default StorageClass) |
| Apps | hello-world | Example nginx app at `hello.kube.local.tnndev.com` |

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
│   ├── controllers/          # Helm releases: MetalLB, ingress-nginx, Longhorn
│   └── configs/              # CRD instances: MetalLB IP pool, StorageClass patch
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

# Check all resources
kubectl get all -A
```

---

## Adding Apps

See `docs/contributing.md` — the short version:

1. Add a YAML file to `apps/base/<app-name>.yaml`
2. Register it in `apps/base/kustomization.yaml`
3. Commit and push — Flux deploys within 10 minutes

---

## Networking Notes

- TLS is terminated **upstream** (by a reverse proxy or router) — ingress-nginx runs plain HTTP inside the cluster.
- All services use the domain pattern `<app>.kube.local.tnndev.com`.
- MetalLB IP pool: `10.10.40.0 – 10.10.40.250`. Make sure this range doesn't overlap with your router's DHCP range.
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: write comprehensive README with cluster overview and quick start"
```

---

### Task 4: Create `docs/contributing.md`

**Files:**
- Create: `docs/contributing.md`

- [ ] **Step 1: Create `docs/contributing.md`**

Create `docs/contributing.md` with the following content:

```markdown
# Contributing — How to Work with This Repo

This document explains the step-by-step workflows for making changes to the cluster via GitOps.

**Important:** All cluster changes go through Git. Never run `kubectl apply` or `helm install` directly.

---

## Adding a New Application

Applications live in `apps/base/` (environment-agnostic manifests) and optionally `apps/homelab/` (overlays).

### Step 1: Create the app manifest

Create `apps/base/<app-name>.yaml`. Every app needs at minimum:

```yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: <app-name>
---
# Workload (Deployment or StatefulSet)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: <app-name>
spec:
  replicas: 1
  selector:
    matchLabels:
      app: <app-name>
  template:
    metadata:
      labels:
        app: <app-name>
    spec:
      containers:
        - name: <app-name>
          image: <image>:<tag>
          ports:
            - containerPort: <port>
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
  namespace: <app-name>
spec:
  selector:
    app: <app-name>
  ports:
    - port: 80
      targetPort: <port>
---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app-name>
  namespace: <app-name>
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - host: <app-name>.kube.local.tnndev.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <app-name>
                port:
                  number: 80
```

### Step 2: Register the app in the base kustomization

Edit `apps/base/kustomization.yaml` and add the new file to the `resources` list:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - hello-word.yaml
  - <app-name>.yaml    # ← add this line
```

### Step 3: (Optional) Add homelab-specific overrides

If the app needs homelab-specific patches (e.g., different replica count, extra env vars), add a patch file in `apps/homelab/` and reference it in `apps/homelab/kustomization.yaml`.

### Step 4: Commit and push

```bash
git add apps/base/<app-name>.yaml apps/base/kustomization.yaml
git commit -m "feat(apps): add <app-name>"
git push
```

Flux reconciles every 10 minutes. To trigger immediately (requires `flux` CLI):

```bash
flux reconcile kustomization apps --with-source
```

### Step 5: Verify

```bash
kubectl get all -n <app-name>
kubectl get ingress -n <app-name>
```

---

## Adding an Application via Helm

If the app is distributed as a Helm chart, put it in `apps/base/<app-name>.yaml` as a HelmRelease:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: <app-name>
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: <app-name>
  namespace: flux-system
spec:
  interval: 24h
  url: https://<chart-repo-url>
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <app-name>
  namespace: <app-name>
spec:
  interval: 1h
  chart:
    spec:
      chart: <chart-name>
      version: "<semver-range>"   # e.g. "1.x" or "1.2.x"
      sourceRef:
        kind: HelmRepository
        name: <app-name>
        namespace: flux-system
      interval: 12h
  values:
    # chart-specific values here
```

Then follow Steps 2–5 above.

---

## Adding a New Infrastructure Controller

Infrastructure controllers (cluster-wide tools like MetalLB, ingress-nginx, Longhorn) live in `infrastructure/controllers/`.

### Step 1: Create the controller manifest

Create `infrastructure/controllers/<tool-name>.yaml`:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: <tool-name>-system
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: <tool-name>
  namespace: flux-system
spec:
  interval: 24h
  url: https://<chart-repo-url>
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <tool-name>
  namespace: <tool-name>-system
spec:
  interval: 1h
  chart:
    spec:
      chart: <chart-name>
      version: "<semver-range>"
      sourceRef:
        kind: HelmRepository
        name: <tool-name>
        namespace: flux-system
      interval: 12h
  values:
    # chart-specific values
```

### Step 2: Register in the controllers kustomization

Edit `infrastructure/controllers/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ingress-nginx.yaml
  - metallb.yaml
  - longhorn.yaml
  - <tool-name>.yaml    # ← add this line
```

### Step 3: (Optional) Add CRD instances to configs

If the controller installs CRDs that need instances (like MetalLB's `IPAddressPool`), create a config file:

```bash
infrastructure/configs/<tool-name>-config.yaml
```

Then add it to `infrastructure/configs/kustomization.yaml`.

### Step 4: Commit and push

```bash
git add infrastructure/controllers/<tool-name>.yaml infrastructure/controllers/kustomization.yaml
git commit -m "feat(infra): add <tool-name> controller"
git push
```

---

## Modifying Ansible Roles

The Ansible playbooks in `k3s-cluster/ansible/` are used **only during initial node provisioning**. They are not continuously reconciled by Flux.

To apply Ansible changes to existing nodes, you must re-run the playbook manually:

```bash
cd k3s-cluster/ansible
ansible-playbook playbook.yml -i inventory/hosts.ini
```

### Role responsibilities

| Role | What it does |
|------|-------------|
| `common` | OS prep: packages, kernel modules, sysctl, UFW firewall, swap off, hostname, reboot |
| `k3s-master` | Installs K3s server; deploys kube-vip manifests; first master does `--cluster-init` |
| `k3s-worker` | Installs K3s agent; formats and mounts the Longhorn data disk at `/var/lib/longhorn` |

---

## Secrets

**Do not commit secrets.** If an app needs a Kubernetes Secret:

1. Create the secret manually on the cluster:
   ```bash
   kubectl create secret generic <secret-name> \
     --namespace <app-name> \
     --from-literal=key=value
   ```
2. Reference it in your app manifest.
3. Document in this file that the secret must be created out-of-band.

For a fully GitOps-managed secrets workflow, adopt SOPS + Age encryption (not yet set up — see `AGENTS.md` for guidance).
```

- [ ] **Step 2: Commit**

```bash
git add docs/contributing.md
git commit -m "docs: add contributing guide for apps, infra, and Ansible workflows"
```

---

## Self-Review

### Spec coverage check

| Requirement | Covered by |
|-------------|-----------|
| AGENTS.md with repo structure | Task 2 |
| AGENTS.md GitOps-only rule | Task 2 |
| AGENTS.md how to add apps | Task 2 |
| AGENTS.md how to add infra | Task 2 |
| AGENTS.md naming conventions | Task 2 |
| AGENTS.md secrets handling | Task 2 |
| README with repo overview | Task 3 |
| opencode.json config | Task 1 |
| Contributing / workflow docs | Task 4 |

### Placeholder scan
- No TBD, TODO, or "implement later" in any task.
- All file contents are fully specified.

### Type consistency
- N/A — this is a documentation/config plan, no code types involved.
