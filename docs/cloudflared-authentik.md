# Cloudflare Tunnel for Authentik

Exposes Authentik publicly at `https://authentik.tnndev.com` via a Cloudflare Tunnel while:

- **Blocking** `/if/admin/` (admin UI) and `/django-admin/` (Django admin) for tunnel traffic
- **Allowing** full access via the local URL `https://authentik.kube.local.tnndev.com` (Envoy Gateway, unchanged)

The tunnel resources live in the `authentik` namespace alongside the app itself.

---

## Architecture

```
Internet
  │
  ▼
Cloudflare Edge (TLS terminated, sets X-Forwarded-Proto: https)
  │
  │  cloudflared-authentik Deployment (namespace: authentik, 2 replicas)
  │
  ├─ /if/admin/.*     → 403 Forbidden
  ├─ /django-admin/.* → 403 Forbidden
  └─ everything else  → authentik-server.authentik.svc:9000

Local Network
  │
  ▼
Envoy Gateway (10.10.40.1) → authentik.kube.local.tnndev.com
  │
  └─ full access (no path restrictions)
```

Traffic via the tunnel receives the correct `X-Forwarded-Proto: https` and `X-Forwarded-For` headers from the Cloudflare edge. Authentik trusts private CIDR ranges by default (`10.0.0.0/8` included), so no additional proxy trust configuration is needed.

---

## GitOps files

| File | Purpose |
|------|---------|
| [apps/base/cloudflared/authentik.yaml](../apps/base/cloudflared/authentik.yaml) | InfisicalSecret + ConfigMap + Deployment |
| [apps/base/kustomization.yaml](../apps/base/kustomization.yaml) | Includes `cloudflared/authentik.yaml` |

---

## One-time Setup (run locally, not via GitOps)

### 1. Install cloudflared CLI

```sh
# macOS
brew install cloudflared
# Or: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
```

### 2. Authenticate to your Cloudflare account

```sh
cloudflared tunnel login
# Opens browser — authorise for the tnndev.com zone
```

### 3. Create the named tunnel

```sh
cloudflared tunnel create authentik-tunnel
# → Created tunnel authentik-tunnel with id <TUNNEL-UUID>
# Credentials file written to ~/.cloudflared/<TUNNEL-UUID>.json
```

Note the UUID and keep the credentials file — both go into Infisical in the next step.

### 4. Store credentials in Infisical

In Infisical project `home-lab-k3s`, environment `prod`, path `/cloudflared/authentik`:

| Key | Value |
|-----|-------|
| `CLOUDFLARE_TUNNEL_ID` | The tunnel UUID printed in step 3 |
| `CLOUDFLARE_TUNNEL_CREDENTIALS_JSON` | Full contents of `~/.cloudflared/<TUNNEL-UUID>.json` |

The Infisical operator renders `config.yaml` (with the tunnel UUID embedded) directly into the K8s Secret via Go template — no ConfigMap or init container needed.

### 5. Create the Cloudflare DNS record

```sh
cloudflared tunnel route dns authentik-tunnel authentik.tnndev.com
# Adds CNAME: authentik.tnndev.com → <TUNNEL-UUID>.cfargotunnel.com
```

### 6. Push — Flux reconciles automatically

No further action needed. Flux picks up the commit within ~10 minutes. The tunnel connects outbound — no firewall ports need to be opened.

---

## Verifying the Tunnel

Check cloudflared pod logs:

```sh
kubectl logs -n authentik -l app=cloudflared-authentik --tail=50
```

Verify admin paths are blocked from the public URL:

```sh
curl -I https://authentik.tnndev.com/if/admin/
# Expect: HTTP/2 403

curl -I https://authentik.tnndev.com/if/flow/initial-setup/
# Expect: HTTP/2 200 (or redirect to flow)
```

Verify local access is unrestricted:

```sh
curl -I http://authentik.kube.local.tnndev.com/if/admin/
# Expect: 200 (admin UI loads normally)
```

---

## Blocked Routes

| Path | Reason |
|------|--------|
| `/if/admin/.*` | Authentik admin web interface |
| `/django-admin/.*` | Django super-admin panel |

The REST API at `/api/v3/` is **not** blocked because authentication flows (login, MFA, password reset) depend on it. Admin-specific API endpoints within `/api/v3/` require valid admin credentials that cannot be obtained via the tunnel (admin UI is blocked).

---

## Adding More Blocked Paths

Edit the `ingress` section of the ConfigMap in [authentik.yaml](../apps/base/cloudflared/authentik.yaml) and add a rule **before** the catch-all service rule:

```yaml
- hostname: authentik.tnndev.com
  path: /some/sensitive/path/.*
  service: http_status:403
```

Commit and push — Flux reconciles automatically.

---

## Adding a Tunnel for Another App

See the "How to Add a New Cloudflare Tunnel" section in [AGENTS.md](../AGENTS.md) for the full pattern. Short version: create `apps/base/cloudflared/<app-name>.yaml`, store credentials in Infisical at `/cloudflared/<app-name>`, add to kustomization.
