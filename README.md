# Coflnet Organization Shared Workflows

## Container CI (`container-ci.yml`)

Reusable workflow: **Build on GitHub → Push to GHCR → Signal Argo via OIDC webhook**

### Architecture

```
GitHub Runner (ephemeral)          Cluster (trusted boundary)
┌──────────────────────┐          ┌──────────────────────────────────┐
│ checkout → buildx     │          │ Argo Events (webhook + mTLS)      │
│   ↓                   │  OIDC    │   ↓                               │
│ push to GHCR          │──JWT───→│ verify JWT (JWKS from GitHub)    │
│   ↓                   │          │   ↓                               │
│ request OIDC token    │          │ skopeo pull GHCR → Trivy scan    │
│ POST webhook          │          │   ↓ (gate: HIGH+ CVEs block)     │
└──────────────────────┘          │ skopeo push → Docker Hub         │
                                  │   ↓                               │
                                  │ promote (PR to fleet/k8s)         │
                                  └──────────────────────────────────┘
```

### Why this design?

- **No cluster secrets on GitHub runners**: Docker Hub credentials live only in OpenBao inside the cluster. A compromised workflow cannot exfiltrate them.
- **Build isolation**: The build runs on GitHub's ephemeral runners — nothing on the same host as the cluster.
- **OIDC authenticity**: The JWT is signed by GitHub's private key. The cluster verifies it against `token.actions.githubusercontent.com` JWKS. A developer cannot forge the `repository` claim.
- **Authoritative Trivy scan**: The cluster runs Trivy against the exact image before it reaches Docker Hub — the scan cannot be skipped or tampered with.

### How to use (caller repo)

Create `.github/workflows/ci.yml` in your repo:

```yaml
name: CI

on:
  push:
    branches: [main, release, develop, preprod]

permissions:
  id-token: write
  packages: write
  contents: read

jobs:
  container:
    uses: Coflnet/.github/.github/workflows/container-ci.yml@main
    with:
      # Optional: path to Dockerfile (default: ./Dockerfile)
      dockerfile: ./src/Dockerfile
      # Optional: build context (default: .)
      context: ./src
      # Optional: path in fleet repo to update on promotion
      k8s_file: sky/chart/charts/sky-proxy/values.yaml
```

### Required secrets

All build-time secrets are **organization-level** secrets in the Coflnet org.
No per-repo setup needed. Create these in the org settings:

| Secret               | Purpose                                       |
|----------------------|-----------------------------------------------|
| `NUGET_USERNAME`     | NuGet private feed authentication (optional)  |
| `NUGET_PASSWORD`     | NuGet private feed authentication (optional)  |
| `NUXT_UI_PRO_LICENSE`| Nuxt UI Pro license key (optional)            |
| `HF_TOKEN`           | HuggingFace API token for model downloads     |

### Cluster-side setup

See `Coflnet/fleet` repo:
- `argo-events/kustomize/talos-eu-hcloud/` — webhook eventsource + sensor
- `argo-workflow/workflow-templates/kustomize/` — verify, pull-scan-push, promote templates

OpenBao manual step (not in git):
- Add `ghcr_username` / `ghcr_token` to `kv/data/argo-workflows/promote`
  (org fine-grained PAT with `packages:read` scope)
