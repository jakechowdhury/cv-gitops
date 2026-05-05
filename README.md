# cv-gitops

GitOps repository for deploying [cv-site](https://github.com/jakechowdhury/cv-site) and its supporting infrastructure to a Kubernetes cluster via ArgoCD.

## Applications

| App | Description |
|-----|-------------|
| `cv-site` | Personal CV site, served via nginx |
| `traefik` | Ingress controller, deployed from the Helm chart with values from this repo |
| `traefik-config` | IngressRoute and middleware configuration |
| `cloudflared` | Cloudflare Tunnel connector (two replicas for HA) |

## Structure

```
clusters/          ArgoCD Application manifests — one per app
apps/
  cv-site/         Deployment, Service, Kustomization
  traefik/         Helm values
  traefik-config/  IngressRoute
  cloudflared/     Deployment
.github/workflows/ CI — cosign signature verification on release PRs
```

## Deployment flow

1. A release tag (`v*.*.*`) is pushed to cv-site
2. cv-site's `release.yml` builds and signs the image with [cosign](https://github.com/sigstore/cosign) (keyless, via Sigstore)
3. cv-site opens a PR here labelled `cv-release`, bumping `apps/cv-site/kustomization.yaml` to the new image tag
4. The `verify-signature` workflow runs and confirms the image was signed by `release.yml`
5. Once the PR is merged, ArgoCD detects the change and rolls out the new image

Image updates for cv-site are intentionally excluded from Renovate — the cv-site pipeline owns that tag.

## CI

**Verify image signature** (`.github/workflows/verify-signature.yml`)
Runs on PRs labelled `cv-release` that touch `apps/cv-site/kustomization.yaml`. Pulls the image reference from the Kustomize overlay and runs `cosign verify` against the Sigstore transparency log, checking the certificate was issued to `release.yml` in the cv-site repo.

All other dependencies (Traefik chart version, cloudflared image, pre-commit hook revisions) are kept up to date by Renovate.

## Pre-commit hooks

```
pre-commit install
```

Hooks run: trailing whitespace, EOF, YAML lint, secret detection, Checkov (Kubernetes framework), and `kustomize build` validation for each overlay.
