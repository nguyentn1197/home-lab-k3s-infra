# Gateway API Migration Design

**Date:** 2026-05-08  
**Status:** Approved

---

## Goal

Replace ingress-nginx with Envoy Gateway as the cluster's HTTP routing layer, migrating from Kubernetes `Ingress` resources to Gateway API `HTTPRoute` resources. The external IP (`10.10.40.1`), domain pattern (`*.kube.local.tnndev.com`), and no-TLS-in-cluster behaviour are all preserved.

---

## Current State

| Resource | Current |
|----------|---------|
| Ingress controller | ingress-nginx (Helm, `ingress-nginx` namespace) |
| LoadBalancer IP | `10.10.40.1` (MetalLB annotation) |
| Routing resource | `networking.k8s.io/v1 Ingress` with `ingressClassName: nginx` |
| Longhorn UI | Enabled via `ingress.enabled: true`, `ingressClassName: nginx` in Helm values |
| App example | `hello-word.yaml` uses `Ingress` with nginx ssl-redirect annotation |
| TLS | Terminated upstream — no TLS inside cluster |

---

## Architecture

### Approach: Option A — CRDs separate from controller

Gateway API CRDs (upstream Kubernetes SIGs resource) are installed independently of the Envoy Gateway controller. This mirrors how MetalLB CRDs are handled today: controller in `infrastructure/controllers/`, CRD instances and config in `infrastructure/configs/`.

### Updated Flux Dependency Chain

```
infra-controllers               →   infra-configs                         →   apps
─────────────────────               ─────────────────────────────────────     ──────────────────
metallb                             gateway-api-crds (CRDs first)             hello-world HTTPRoute
longhorn                            gateway.yaml (GatewayClass + Gateway)     future app HTTPRoutes
envoy-gateway                       longhorn-httproute.yaml
                                    metallb-config (existing)
                                    local-path-sc-patch (existing)
```

Within `infra-configs`, Kustomize applies resources top-to-bottom. `gateway-api-crds` must appear before `gateway.yaml` in `kustomization.yaml`.

### Gateway ownership model

One shared `Gateway` resource lives in `infrastructure/configs/gateway.yaml`. All apps attach to it via `HTTPRoute.spec.parentRefs`. Apps never define their own Gateway.

---

## Component Design

### 1. `infrastructure/controllers/envoy-gateway.yaml` (new)

- **Namespace:** `envoy-gateway-system`
- **OCIRepository:** `oci://docker.io/envoyproxy/gateway-helm` with `layerSelector.mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip`, `operation: copy`, and `ref.tag: v1.7.2`
- **HelmRelease:** `releaseName: eg`, namespace `envoy-gateway-system`, `chartRef` to the OCIRepository
- **dependsOn:** `metallb` in `metallb-system` (same constraint ingress-nginx had)
- **values:** minimal — Envoy Gateway works out of the box; no special values needed

### 2. Remove `infrastructure/controllers/ingress-nginx.yaml`

Deleted entirely. Removed from `infrastructure/controllers/kustomization.yaml`.

### 3. `infrastructure/configs/gateway.yaml` (new)

Two resources:

**GatewayClass:**
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

**Gateway:**
```yaml
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

The `hostname` wildcard on the listener restricts which routes can attach — only routes matching `*.kube.local.tnndev.com` are accepted.

### 4. `infrastructure/configs/longhorn-httproute.yaml` (new)

```yaml
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

### 5. `infrastructure/controllers/longhorn.yaml` (modified)

In the HelmRelease `values`, change:
```yaml
# Before
ingress:
  enabled: true
  ingressClassName: nginx
  host: longhorn.kube.local.tnndev.com

# After
ingress:
  enabled: false
```

### 6. `apps/base/hello-word.yaml` (modified)

Remove the `Ingress` resource. Replace with:
```yaml
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

No nginx-specific annotations needed — TLS redirect is handled by the Gateway listener (HTTP only, no redirect configured).

### 7. `infrastructure/configs/kustomization.yaml` (modified)

Add new files in this order (order matters for CRD-before-instance dependency):
```yaml
resources:
  - gateway-api-crds.yaml      # ← new, must be first
  - gateway.yaml               # ← new
  - longhorn-httproute.yaml    # ← new
  - metallb-config.yaml        # existing
  - local-path-sc-patch.yaml   # existing
```

### 8. `infrastructure/controllers/kustomization.yaml` (modified)

Remove `ingress-nginx.yaml`, add `envoy-gateway.yaml`:
```yaml
resources:
  - metallb.yaml
  - longhorn.yaml
  - envoy-gateway.yaml   # ← new (replaces ingress-nginx.yaml)
```

### 9. `docs/contributing.md` and `AGENTS.md` (updated)

Update the "How to Add a New Application" sections to replace the `Ingress` template with an `HTTPRoute` template. Update all references to `ingressClassName: nginx` and nginx annotations.

---

## What Does NOT Change

- MetalLB, Longhorn, kube-vip — untouched
- LoadBalancer IP: `10.10.40.1`
- Domain pattern: `*.kube.local.tnndev.com`
- No TLS inside the cluster
- Flux reconciliation structure (`infra-controllers → infra-configs → apps`)
- All other Flux Kustomization files in `clusters/homelab/`

---

## Risks & Notes

- **Longhorn frontend service name:** Longhorn's Helm chart exposes the UI via a Service named `longhorn-frontend` in `longhorn-system`. This is verified from the Longhorn chart — confirm it hasn't changed if using a non-standard version.
- **OCI source pattern:** Envoy Gateway uses Flux's `OCIRepository` source with `layerSelector` and `ref.tag`, then a `HelmRelease` that references it via `chartRef`.
- **Downtime:** Since this is a clean cut-over (not a parallel run), there will be a brief window between removing ingress-nginx and Envoy Gateway becoming ready where HTTP routing is unavailable. Acceptable for a home lab.
- **`infra-configs` ordering:** Kustomize does not guarantee CRD readiness before dependent resources — Flux's `wait: true` on `infra-controllers` ensures the Envoy Gateway controller is ready, but within `infra-configs` the CRD Helm chart must complete before `GatewayClass`/`Gateway` are applied. Setting `wait: true` on the `infra-configs` Kustomization (or using `dependsOn` within the Kustomization) handles this.
