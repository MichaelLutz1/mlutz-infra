# mlutz-infra

GitOps infrastructure repository for a Kubernetes homelab cluster managed with Argo CD.

## What this repo does

- Bootstraps Argo CD with a root app (`app-of-apps`) pattern.
- Manages Argo CD itself from this same repo (self-managed Argo).
- Deploys workload applications via Argo CD:
  - `picture-it` web app
  - `cloudflared` tunnel connector

## Repository layout

- `bootstrap/root-app/root-app.yaml`
  - One-time bootstrap manifest for the Argo CD `root` Application.
  - Points Argo CD to `argocd/apps` in this repository.
- `argocd/apps/`
  - Argo CD Application definitions and kustomization for managed apps.
  - `argocd-self.yaml`: installs Argo CD Helm chart using values in this repo.
  - `picture-it.yaml`: deploys `apps/picture-it/base`.
  - `cloudflared.yaml`: deploys `apps/cloudflared`.
- `apps/argocd/helm/values.yaml`
  - Argo CD Helm values (external URL + ingress settings).
- `apps/picture-it/base/`
  - Kubernetes manifests for the `picture-it` app (namespace, deployment, service, ingress).
- `apps/cloudflared/`
  - Kubernetes manifests for Cloudflare Tunnel connector (namespace + deployment).

## Deployment architecture

1. Apply `bootstrap/root-app/root-app.yaml` to an existing Argo CD installation.
2. The `root` Application syncs `argocd/apps/kustomization.yaml`.
3. Argo CD then manages:
   - itself (`argocd-self` via Helm),
   - `picture-it`,
   - `cloudflared`.

This creates a standard GitOps flow where commits to `main` become desired cluster state.

## Managed applications

### Argo CD (self-managed)

- Chart: `argo-cd` from `https://argoproj.github.io/argo-helm`
- Version: `9.4.2`
- Namespace: `argocd`
- Ingress host: `argocd.michaellutz.org`
- Ingress class: `traefik`
- Important settings:
  - `configs.cm.url` set to external Argo CD URL
  - `server.insecure: "true"` (TLS termination expected at ingress/proxy)
  - Sync policy: automated with prune + self-heal enabled
  - `ServerSideApply=true`

### picture-it

- Namespace: `picture-it`
- Deployment:
  - Image: `ghcr.io/mlutz-devops/picture-it:v0.1.0`
  - Replicas: `1`
  - Container port: `8080`
- Service:
  - Name: `picture-it-service`
  - Type: `ClusterIP`
  - Port mapping: `80 -> 8080`
- Ingress:
  - Host: `picture-it.michaellutz.org`
  - Path: `/`

### cloudflared

- Namespace: `cloudflared`
- Deployment:
  - Image: `cloudflare/cloudflared:latest`
  - Replicas: `2`
  - Runs `cloudflared tunnel run`
  - Exposes metrics/readiness endpoint on `:2000`
  - Liveness probe on `/ready`
- Required secret:
  - Secret name: `tunnel-token`
  - Key: `token`
  - Mounted as env var `TUNNEL_TOKEN`

## Sync behavior notes

- `argocd-self` has automated sync (`prune: true`, `selfHeal: true`).
- `picture-it` and `cloudflared` currently have `syncPolicy: null` (manual sync unless changed in Argo CD).
- `root` app has automated sync enabled, but `prune` and `selfHeal` are currently null.

## Prerequisites

- A running Kubernetes cluster.
- Argo CD installed in namespace `argocd` for initial bootstrap.
- Ingress controller compatible with `ingressClassName: traefik` (as configured).
- DNS records for:
  - `argocd.michaellutz.org`
  - `picture-it.michaellutz.org`
- Cloudflare Tunnel token stored as Kubernetes secret in `cloudflared` namespace.

## Bootstrap quick start

```bash
kubectl apply -f bootstrap/root-app/root-app.yaml
```

After applying, use the Argo CD UI/CLI to verify app sync and health.
