# Infisical Secrets Operator — Bootstrap Guide

The Infisical Secrets Operator is deployed via Flux (HelmRelease in `infrastructure/controllers/infisical.yaml`).
Once it is running, you use `InfisicalSecret` CRDs to sync secrets from Infisical Cloud into Kubernetes `Secret` objects.

This guide covers the one-time manual step that cannot be GitOps-managed: planting the Machine Identity credentials on the cluster.

---

## Step 1 — Create a Machine Identity in Infisical Cloud

1. Log in to [app.infisical.com](https://app.infisical.com).
2. Go to **Access Control → Machine Identities → Create**.
3. Name it something like `homelab-k3s`.
4. Under **Authentication**, choose **Universal Auth**.
5. Attach it to the project(s) and environment(s) you want the cluster to read.
6. Copy the **Client ID** and **Client Secret** — you will need them in Step 2.

---

## Step 2 — Plant the bootstrap Secret on the cluster

This Secret holds the Machine Identity credentials. It is the **only** secret you manage outside of GitOps.
Never commit it to this repository.

Create it **once** in the operator's own namespace so all apps can share it:

```bash
kubectl create secret generic infisical-universal-auth \
  --namespace infisical-operator-system \
  --from-literal=clientId=<CLIENT_ID> \
  --from-literal=clientSecret=<CLIENT_SECRET>
```

Every `InfisicalSecret` across every app namespace will reference this single secret via `credentialsRef.secretNamespace: infisical-operator-system`.

---

## Step 3 — Create an InfisicalSecret resource

Add an `InfisicalSecret` to your app manifest (committed to git — no sensitive data here):

```yaml
apiVersion: secrets.infisical.com/v1alpha1
kind: InfisicalSecret
metadata:
  name: <app-name>-secrets
  namespace: <app-namespace>
spec:
  hostAPI: https://app.infisical.com/api
  resyncInterval: 60         # seconds between re-syncs
  authentication:
    universalAuth:
      secretsScope:
        projectSlug: <infisical-project-slug>
        envSlug: prod          # dev | staging | prod (or your custom env slug)
        secretsPath: "/"       # path within the environment
        recursive: false
      credentialsRef:
        secretName: infisical-universal-auth
        secretNamespace: infisical-operator-system
  managedKubeSecretReferences:
    - secretName: <app-name>-secrets
      secretNamespace: <app-namespace>
      creationPolicy: Owner    # Operator owns the Secret lifecycle
```

The operator will create (and keep in sync) a Kubernetes `Secret` named `<app-name>-secrets` populated with every secret from the specified Infisical path.

---

## Verify

```bash
# Operator pods are healthy
kubectl get pods -n infisical-operator-system

# InfisicalSecret status
kubectl describe infisicalsecret <app-name>-secrets -n <app-namespace>

# Synced Kubernetes Secret
kubectl get secret <app-name>-secrets -n <app-namespace> -o yaml
```

A healthy `InfisicalSecret` will show `conditions[0].reason: Synced` in its status.
