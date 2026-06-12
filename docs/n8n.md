# n8n

Workflow automation, deployed Authentik-style: single-replica Deployment + Longhorn PVC, CNPG Postgres, Infisical secrets, Cloudflare tunnel for external access.

## Manifests

| File | Contents |
|------|----------|
| `apps/base/n8n/n8n.yaml` | Namespace, InfisicalSecrets, PVC, Deployment, Service, HTTPRoute, ServiceMonitor |
| `apps/base/n8n/n8n-postgres.yaml` | `postgres-n8n` namespace, CNPG `n8n-db` cluster (2 instances), daily snapshot backup |
| `apps/base/cloudflared/n8n.yaml` | Cloudflare tunnel (UI + webhooks, gated by Cloudflare Access) |

## URLs

| | |
|---|---|
| LAN | `http://n8n.kube.local.tnndev.com` |
| Public (canonical) | `https://n8n.tnndev.com` — UI behind Cloudflare Access, `/webhook/*` open |

## Required Infisical secrets (before pushing)

| Path | Keys |
|------|------|
| `/n8n/app` | `N8N_ENCRYPTION_KEY` — `openssl rand -base64 24`. **Back it up**: every credential saved in n8n is encrypted with it; if it's lost they are unrecoverable. |
| `/n8n/postgres` | `N8N_POSTGRES_USERNAME` (must be `n8n`, matches `initdb.owner`), `N8N_POSTGRES_PASSWORD` |
| `/cloudflared/n8n` | `CLOUDFLARE_TUNNEL_ID`, `CLOUDFLARE_TUNNEL_CREDENTIALS_JSON` (from `cloudflared tunnel create n8n-tunnel` + `cloudflared tunnel route dns n8n-tunnel n8n.tnndev.com`) |

## External access & Authentik OIDC

n8n's native OIDC/SAML SSO is a **paid feature** (Business/Enterprise license; community edition only has built-in email/password users). Instead, auth is enforced at the Cloudflare edge with **Authentik as the OIDC identity provider for Cloudflare Access**. The tunnel forwards all paths; Access decides who gets through.

> ⚠️ Deploy order matters: configure Cloudflare Access **before** pushing the tunnel manifest, otherwise the n8n UI is briefly exposed publicly (with only n8n's own login as protection).

### 1. Authentik: create the OIDC provider

1. Authentik admin → *Applications → Providers → Create* → **OAuth2/OpenID Provider**.
2. Authorization flow: implicit/explicit consent as preferred.
3. Redirect URI: `https://<your-team>.cloudflareaccess.com/cdn-cgi/access/callback` (team domain from Zero Trust → Settings → Custom Pages).
4. Note the Client ID, Client Secret, and the OpenID configuration URL (`https://authentik.tnndev.com/application/o/<app-slug>/`).
5. Create an Application bound to this provider; restrict via Authentik groups if wanted.

### 2. Cloudflare Zero Trust: add Authentik as IdP

*Settings → Authentication → Login methods → Add new → OpenID Connect*, fill in Client ID/Secret and the auth/token/certs URLs from Authentik's provider page. Test the connection.

### 3. Cloudflare Access: two applications for `n8n.tnndev.com`

| App | Path | Policy |
|-----|------|--------|
| `n8n-webhooks` | `n8n.tnndev.com/webhook` | **Bypass** → Everyone (webhook calls can't do an OIDC dance) |
| `n8n-ui` | `n8n.tnndev.com` | **Allow** → login method = Authentik OIDC, restrict to your email/group |

Order matters: the more specific webhook app must rank above the UI app. Add `/form` or `/webhook-waiting` bypass paths if workflows use forms or wait nodes externally.

n8n's own user management stays active behind Access — treat it as the second factor, not a redundancy to disable.

### Alternatives considered

- **n8n native OIDC**: cleanest (real user mapping) but needs a paid license. If you ever buy one, point n8n directly at Authentik and drop the Access Allow policy.
- **Community external-hook shim** ([cweagans/n8n-oidc](https://github.com/cweagans/n8n-oidc)): injects OIDC into community edition at runtime; unofficial and can break on n8n updates — not recommended for a set-and-forget homelab.
- **Envoy Gateway ext-auth with Authentik outpost**: works for LAN traffic but public traffic enters via the tunnel and never touches Envoy, so it doesn't cover the external path.

## Operational notes

- Image is `n8nio/n8n:latest` — consider pinning a version (weekly releases; `latest` jumps on every pod recreation).
- `GENERIC_TIMEZONE` is `Etc/UTC`; change it so Schedule triggers fire at local time.
- Metrics: `N8N_METRICS=true` + ServiceMonitor → already scraped by Prometheus; logs already collected by Alloy.
- Recovery: workflows/credentials live in Postgres (CNPG, 2 instances, daily snapshots); `/home/node/.n8n` PVC holds binary data. Pod restarts are safe; the encryption key in Infisical is the only unrecoverable piece.
- Scaling later: queue mode (Redis + workers) only if single-main throughput becomes a problem.
