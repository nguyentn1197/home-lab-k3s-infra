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
# HTTPRoute (Gateway API)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app-name>
  namespace: <app-name>
spec:
  parentRefs:
    - name: homelab
      namespace: envoy-gateway-system
  hostnames:
    - "<app-name>.kube.local.tnndev.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: <app-name>
          port: 80
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
kubectl get httproute -n <app-name>
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

## Adding a CloudNativePG PostgreSQL Cluster

PostgreSQL clusters are managed by the CloudNativePG operator. Use the example in
`docs/examples/cloudnative-pg-cluster.yaml` as the starting point.

1. Copy it to `apps/base/<purpose>-postgres.yaml`.
2. Rename the namespace to `postgres-<purpose>` and the cluster to `<purpose>-db`.
3. Replace database names, owner names, storage sizes, and the referenced Secret.
4. Keep `backup.volumeSnapshot.className: longhorn-backup-vsc` for Longhorn backup snapshots.
5. Keep the `ScheduledBackup` label and annotation `homelab.tnndev.com/cnpg-retention: 7d` if the 7-day cleanup job should manage its backups.
6. Add the file to `apps/base/kustomization.yaml`.
7. Commit and push.

Do not commit database credentials. Sync the referenced Secret with Infisical or
create it out-of-band before the cluster initializes.

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
  - metallb.yaml
  - longhorn.yaml
  - envoy-gateway.yaml
  - <tool-name>.yaml    # ← add this line
```

### Step 3: (Optional) Add CRD instances to configs

If the controller installs CRDs that need instances (like MetalLB's `IPAddressPool`), create a config file:

```
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
