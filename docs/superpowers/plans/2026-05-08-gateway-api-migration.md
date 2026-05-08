# Gateway API Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace ingress-nginx with Envoy Gateway, migrating all routing from `Ingress` resources to Gateway API `HTTPRoute` resources, while preserving the same external IP (`10.10.40.1`) and domain pattern (`*.kube.local.tnndev.com`).

**Architecture:** Envoy Gateway is installed as a Helm controller in `infrastructure/controllers/`. The Gateway API CRDs are installed separately as a Helm chart in `infrastructure/configs/` (must apply before GatewayClass/Gateway resources). One shared `Gateway` object in `envoy-gateway-system` receives all traffic; apps attach via `HTTPRoute` resources in their own namespaces.

**Tech Stack:** Flux CD v2, Envoy Gateway (Helm OCI chart `oci://docker.io/envoyproxy/gateway-helm`), Gateway API CRDs (Helm chart from `https://kubernetes-sigs.github.io/gateway-api`), MetalLB (unchanged), Longhorn (unchanged), K3s.

---

## File Map

| Action | File | What changes |
|--------|------|-------------|
| Create | `infrastructure/controllers/envoy-gateway.yaml` | New: Namespace + HelmRepository (OCI) + HelmRelease for Envoy Gateway |
| Delete | `infrastructure/controllers/ingress-nginx.yaml` | Removed entirely |
| Modify | `infrastructure/controllers/kustomization.yaml` | Remove ingress-nginx, add envoy-gateway |
| Modify | `infrastructure/controllers/longhorn.yaml` | Set `ingress.enabled: false`, remove ingressClassName/host |
| Create | `infrastructure/configs/gateway-api-crds.yaml` | New: HelmRepository + HelmRelease for Gateway API CRDs |
| Create | `infrastructure/configs/gateway.yaml` | New: GatewayClass + Gateway resources |
| Create | `infrastructure/configs/longhorn-httproute.yaml` | New: HTTPRoute for Longhorn UI |
| Modify | `infrastructure/configs/kustomization.yaml` | Add 3 new files (CRDs first), preserve existing 2 |
| Modify | `clusters/homelab/configs.yaml` | Add `wait: true` so CRD HelmRelease completes before GatewayClass applies |
| Modify | `apps/base/hello-word.yaml` | Replace Ingress with HTTPRoute |
| Modify | `docs/contributing.md` | Update app template from Ingress to HTTPRoute |
| Modify | `AGENTS.md` | Update "How to Add" sections to reference HTTPRoute |

---

## Task 1: Add `wait: true` to infra-configs Kustomization

`infra-configs` currently lacks `wait: true`. Without it, Flux may attempt to apply `GatewayClass`/`Gateway` before the Gateway API CRD HelmRelease has finished installing the CRDs, causing a "no matches for kind" error.

**Files:**
- Modify: `clusters/homelab/configs.yaml`

- [ ] **Step 1: Open `clusters/homelab/configs.yaml` and add `wait: true`**

Replace the file content with:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infra-configs
  namespace: flux-system
spec:
  dependsOn:
    - name: infra-controllers
  interval: 10m
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/configs
  prune: true
  wait: true
```

- [ ] **Step 2: Verify the diff looks correct**

```bash
git diff clusters/homelab/configs.yaml
```

Expected output: one line added — `  wait: true` at the end of the spec block.

- [ ] **Step 3: Commit**

```bash
git add clusters/homelab/configs.yaml
git commit -m "feat(flux): add wait: true to infra-configs Kustomization"
```

---

## Task 2: Create Envoy Gateway controller manifest

**Files:**
- Create: `infrastructure/controllers/envoy-gateway.yaml`

- [ ] **Step 1: Create the file**

Create `infrastructure/controllers/envoy-gateway.yaml` with this exact content:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: envoy-gateway-system
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: envoy-gateway
  namespace: flux-system
spec:
  interval: 24h
  type: oci
  url: oci://docker.io/envoyproxy/gateway-helm
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: envoy-gateway
  namespace: envoy-gateway-system
spec:
  interval: 1h
  dependsOn:
    - name: metallb
      namespace: metallb-system
  chart:
    spec:
      chart: gateway-helm
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: envoy-gateway
        namespace: flux-system
      interval: 12h
```

- [ ] **Step 2: Verify the file looks correct**

```bash
cat infrastructure/controllers/envoy-gateway.yaml
```

Expected: 3 YAML documents separated by `---` (Namespace, HelmRepository, HelmRelease). HelmRepository has `type: oci` and URL starting with `oci://`.

- [ ] **Step 3: Commit**

```bash
git add infrastructure/controllers/envoy-gateway.yaml
git commit -m "feat(infra): add Envoy Gateway controller manifest"
```

---

## Task 3: Update controllers kustomization (swap ingress-nginx → envoy-gateway)

**Files:**
- Modify: `infrastructure/controllers/kustomization.yaml`
- Delete: `infrastructure/controllers/ingress-nginx.yaml`

- [ ] **Step 1: Update `infrastructure/controllers/kustomization.yaml`**

Replace the file content with:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - metallb.yaml
  - longhorn.yaml
  - envoy-gateway.yaml
```

(ingress-nginx.yaml removed; envoy-gateway.yaml added)

- [ ] **Step 2: Delete the ingress-nginx manifest**

```bash
git rm infrastructure/controllers/ingress-nginx.yaml
```

- [ ] **Step 3: Verify**

```bash
git diff --cached infrastructure/controllers/kustomization.yaml
git status
```

Expected: `kustomization.yaml` modified (ingress-nginx line gone, envoy-gateway line added), `ingress-nginx.yaml` staged for deletion.

- [ ] **Step 4: Commit**

```bash
git add infrastructure/controllers/kustomization.yaml
git commit -m "feat(infra): replace ingress-nginx with envoy-gateway in controllers"
```

---

## Task 4: Disable Longhorn's built-in ingress

**Files:**
- Modify: `infrastructure/controllers/longhorn.yaml` (lines 55–58)

- [ ] **Step 1: Update the `ingress` section in `infrastructure/controllers/longhorn.yaml`**

Find the current `ingress:` block at the bottom of the file (lines 55–58):

```yaml
    ingress:
      enabled: true
      ingressClassName: nginx
      host: longhorn.kube.local.tnndev.com
```

Replace it with:

```yaml
    ingress:
      enabled: false
```

The full file after the change should end with:

```yaml
      recurringJobSelector:
        enable: true
        jobList: '[{"name":"default","isGroup":true}]'
    ingress:
      enabled: false
```

- [ ] **Step 2: Verify the diff**

```bash
git diff infrastructure/controllers/longhorn.yaml
```

Expected: 3 lines removed (`ingressClassName: nginx`, `host: longhorn.kube.local.tnndev.com`, and `enabled: true` replaced by `enabled: false`).

- [ ] **Step 3: Commit**

```bash
git add infrastructure/controllers/longhorn.yaml
git commit -m "feat(infra): disable Longhorn built-in ingress (will use HTTPRoute)"
```

---

## Task 5: Create Gateway API CRDs manifest

**Files:**
- Create: `infrastructure/configs/gateway-api-crds.yaml`

- [ ] **Step 1: Create the file**

Create `infrastructure/configs/gateway-api-crds.yaml` with this exact content:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: gateway-api
  namespace: flux-system
spec:
  interval: 24h
  url: https://kubernetes-sigs.github.io/gateway-api
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: gateway-api-crds
  namespace: envoy-gateway-system
spec:
  interval: 1h
  chart:
    spec:
      chart: gateway-api
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: gateway-api
        namespace: flux-system
      interval: 12h
```

Note: No `dependsOn` needed here — `infra-configs` already `dependsOn: infra-controllers`, so Envoy Gateway controller is ready before any config is applied.

- [ ] **Step 2: Verify**

```bash
cat infrastructure/configs/gateway-api-crds.yaml
```

Expected: 2 YAML documents — HelmRepository (standard HTTP URL, no `type: oci`) and HelmRelease targeting namespace `envoy-gateway-system`.

- [ ] **Step 3: Commit**

```bash
git add infrastructure/configs/gateway-api-crds.yaml
git commit -m "feat(infra): add Gateway API CRDs HelmRelease"
```

---

## Task 6: Create GatewayClass and Gateway resources

**Files:**
- Create: `infrastructure/configs/gateway.yaml`

- [ ] **Step 1: Create the file**

Create `infrastructure/configs/gateway.yaml` with this exact content:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: homelab
  namespace: envoy-gateway-system
  annotations:
    metallb.universe.tf/loadBalancerIPs: 10.10.40.1
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: "*.kube.local.tnndev.com"
```

- [ ] **Step 2: Verify**

```bash
cat infrastructure/configs/gateway.yaml
```

Expected: 2 YAML documents — GatewayClass named `envoy` and Gateway named `homelab` in `envoy-gateway-system`, with MetalLB annotation pinning IP to `10.10.40.1`.

- [ ] **Step 3: Commit**

```bash
git add infrastructure/configs/gateway.yaml
git commit -m "feat(infra): add GatewayClass and shared Gateway"
```

---

## Task 7: Create Longhorn HTTPRoute

**Files:**
- Create: `infrastructure/configs/longhorn-httproute.yaml`

- [ ] **Step 1: Create the file**

Create `infrastructure/configs/longhorn-httproute.yaml` with this exact content:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: longhorn
  namespace: longhorn-system
spec:
  parentRefs:
    - name: homelab
      namespace: envoy-gateway-system
  hostnames:
    - "longhorn.kube.local.tnndev.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: longhorn-frontend
          port: 80
```

The backend service name `longhorn-frontend` is the name Longhorn's Helm chart uses for the UI service in `longhorn-system`. If using a non-standard Longhorn version, verify with `kubectl get svc -n longhorn-system`.

- [ ] **Step 2: Verify**

```bash
cat infrastructure/configs/longhorn-httproute.yaml
```

Expected: HTTPRoute in `longhorn-system` namespace, `parentRefs` pointing to `homelab` Gateway in `envoy-gateway-system`, backend `longhorn-frontend:80`.

- [ ] **Step 3: Commit**

```bash
git add infrastructure/configs/longhorn-httproute.yaml
git commit -m "feat(infra): add HTTPRoute for Longhorn UI"
```

---

## Task 8: Update configs kustomization

**Files:**
- Modify: `infrastructure/configs/kustomization.yaml`

- [ ] **Step 1: Update the file**

Replace the file content with:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gateway-api-crds.yaml
  - gateway.yaml
  - longhorn-httproute.yaml
  - metallb-config.yaml
  - local-path-sc-patch.yaml
```

**Order matters:** `gateway-api-crds.yaml` must come before `gateway.yaml` so Kustomize applies the CRD HelmRelease first. Combined with `wait: true` on the `infra-configs` Flux Kustomization (added in Task 1), this ensures the CRDs are installed before `GatewayClass`/`Gateway` are applied.

- [ ] **Step 2: Verify the diff**

```bash
git diff infrastructure/configs/kustomization.yaml
```

Expected: 3 lines added at the top of the resources list (`gateway-api-crds.yaml`, `gateway.yaml`, `longhorn-httproute.yaml`). Existing 2 entries unchanged.

- [ ] **Step 3: Commit**

```bash
git add infrastructure/configs/kustomization.yaml
git commit -m "feat(infra): register gateway CRDs, gateway, and longhorn HTTPRoute in configs kustomization"
```

---

## Task 9: Migrate hello-world app from Ingress to HTTPRoute

**Files:**
- Modify: `apps/base/hello-word.yaml`

- [ ] **Step 1: Update `apps/base/hello-word.yaml`**

Replace the entire `Ingress` resource block (lines 40–58) with an `HTTPRoute`. The full file after the change:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: hello-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: hello-world
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: nginxdemos/hello:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello-world
  namespace: hello-world
spec:
  parentRefs:
    - name: homelab
      namespace: envoy-gateway-system
  hostnames:
    - "hello.kube.local.tnndev.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: hello-world
          port: 80
```

Note: No nginx annotations needed. The Gateway listener is HTTP-only with no redirect configured.

- [ ] **Step 2: Verify the diff**

```bash
git diff apps/base/hello-word.yaml
```

Expected: `Ingress` resource removed (including `nginx.ingress.kubernetes.io/ssl-redirect` annotation and `ingressClassName: nginx`). `HTTPRoute` added with `parentRefs` to `homelab`/`envoy-gateway-system`.

- [ ] **Step 3: Commit**

```bash
git add apps/base/hello-word.yaml
git commit -m "feat(apps): migrate hello-world from Ingress to HTTPRoute"
```

---

## Task 10: Update docs/contributing.md

**Files:**
- Modify: `docs/contributing.md`

- [ ] **Step 1: Replace the Ingress template in the "Adding a New Application" section**

In `docs/contributing.md`, find the `# Ingress` block inside the Step 1 template (around lines 60–81) and replace it with an HTTPRoute block. Also update the Step 5 verify command.

The new app template (Step 1 in the doc) should read:

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

- [ ] **Step 2: Update the Step 5 verify command in the same section**

Find (around line 116):
```bash
kubectl get all -n <app-name>
kubectl get ingress -n <app-name>
```

Replace with:
```bash
kubectl get all -n <app-name>
kubectl get httproute -n <app-name>
```

- [ ] **Step 3: Update the "Adding a New Infrastructure Controller" section (Step 2 example)**

Find the example kustomization in Step 2 of that section (around line 218):
```yaml
resources:
  - ingress-nginx.yaml
  - metallb.yaml
  - longhorn.yaml
  - <tool-name>.yaml    # ← add this line
```

Replace with:
```yaml
resources:
  - metallb.yaml
  - longhorn.yaml
  - envoy-gateway.yaml
  - <tool-name>.yaml    # ← add this line
```

- [ ] **Step 4: Verify the diff**

```bash
git diff docs/contributing.md
```

Expected: `Ingress` template replaced with `HTTPRoute` template; `kubectl get ingress` replaced with `kubectl get httproute`; kustomization example updated to remove ingress-nginx and show envoy-gateway.

- [ ] **Step 5: Commit**

```bash
git add docs/contributing.md
git commit -m "docs: update contributing guide for Gateway API (HTTPRoute replaces Ingress)"
```

---

## Task 11: Update AGENTS.md

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Update the Directory Layout section**

In the `### Directory Layout` code block (around lines 37–44), update the controllers listing:

Find:
```
│   │   ├── ingress-nginx.yaml # Ingress controller (LoadBalancer IP: 10.10.40.1)
```

Replace with:
```
│   │   ├── envoy-gateway.yaml # Envoy Gateway controller (LoadBalancer IP: 10.10.40.1)
```

- [ ] **Step 2: Update the Dependency Order section**

Find (around lines 67–71):
```
infra-controllers  →  infra-configs  →  apps
(metallb, nginx,        (metallb IP        (hello-world,
 longhorn)               pool, SC patch)    future apps)
```

Replace with:
```
infra-controllers  →  infra-configs               →  apps
(metallb,              (gateway-api-crds,              (hello-world HTTPRoute,
 longhorn,              gateway, longhorn-httproute,    future app HTTPRoutes)
 envoy-gateway)         metallb-config, SC patch)
```

- [ ] **Step 3: Update the Cluster Facts table**

Find (around line 86):
```
| Ingress controller | ingress-nginx |
```

Replace with:
```
| Ingress controller | Envoy Gateway (Gateway API) |
```

- [ ] **Step 4: Update the Naming Conventions section**

Find (around lines 101–103):
```
### Ingress hostnames
Pattern: `<app-name>.kube.local.tnndev.com`
Examples: `hello.kube.local.tnndev.com`, `longhorn.kube.local.tnndev.com`
```

Replace with:
```
### HTTPRoute hostnames
Pattern: `<app-name>.kube.local.tnndev.com`
Examples: `hello.kube.local.tnndev.com`, `longhorn.kube.local.tnndev.com`
```

- [ ] **Step 5: Update the "How to Add a New Application" section**

Find (around line 119):
```
1. Create `apps/base/<app-name>.yaml` with Namespace + Deployment/StatefulSet + Service + Ingress.
```

Replace with:
```
1. Create `apps/base/<app-name>.yaml` with Namespace + Deployment/StatefulSet + Service + HTTPRoute (Gateway API). The HTTPRoute must reference the shared `homelab` Gateway in `envoy-gateway-system`.
```

- [ ] **Step 6: Verify the diff**

```bash
git diff AGENTS.md
```

Expected: `ingress-nginx.yaml` → `envoy-gateway.yaml` in directory layout; dependency order updated; cluster facts table updated; "Ingress hostnames" → "HTTPRoute hostnames"; app-adding instruction updated.

- [ ] **Step 7: Commit**

```bash
git add AGENTS.md
git commit -m "docs: update AGENTS.md for Gateway API migration"
```

---

## Self-Review

### Spec coverage check

| Spec requirement | Task |
|-----------------|------|
| Create `envoy-gateway.yaml` with OCI HelmRepository + HelmRelease | Task 2 |
| Delete `ingress-nginx.yaml` | Task 3 |
| Update `controllers/kustomization.yaml` | Task 3 |
| Disable Longhorn built-in ingress | Task 4 |
| Create `gateway-api-crds.yaml` | Task 5 |
| Create `gateway.yaml` (GatewayClass + Gateway) | Task 6 |
| Create `longhorn-httproute.yaml` | Task 7 |
| Update `configs/kustomization.yaml` (CRDs first) | Task 8 |
| Migrate hello-world to HTTPRoute | Task 9 |
| Update `docs/contributing.md` | Task 10 |
| Update `AGENTS.md` | Task 11 |
| Add `wait: true` to `infra-configs` | Task 1 ✓ (spec noted this was needed) |

All spec requirements covered.

### Risks to watch during execution

1. **Longhorn service name:** The HTTPRoute in Task 7 targets `longhorn-frontend`. Verify this service exists: `kubectl get svc -n longhorn-system`. If the name differs, update `longhorn-httproute.yaml` accordingly.
2. **OCI HelmRepository:** Flux must have OCI support enabled (it does by default in Flux v2.x). The `type: oci` field on the HelmRepository is required.
3. **Downtime window:** After Task 3 is committed and Flux reconciles, ingress-nginx is removed. HTTP routing is unavailable until Envoy Gateway is ready (Tasks 5–8). For a home lab this is acceptable, but plan for ~5–10 minutes of downtime.
4. **Gateway API CRD version:** The `gateway-api` Helm chart version `"1.x"` must align with Envoy Gateway 1.x requirements. Both are pinned to `"1.x"` semver ranges, so they should stay compatible automatically.
