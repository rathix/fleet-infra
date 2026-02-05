# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

GitOps repository for managing a production Kubernetes cluster using Flux CD v2. All manifests are declarative Kubernetes YAML; changes pushed to `main` are automatically synced to the cluster.

**Domain:** kennyandries.com
**Tech Stack:** Flux CD, Traefik (ingress), cert-manager (TLS), Longhorn (storage), Sealed Secrets

## Common Commands

### Validate Manifests Locally

```bash
# Build and verify Kustomization renders correctly
kustomize build infrastructure/controllers
kustomize build infrastructure/configs
kustomize build apps/production

# Validate against Kubernetes schemas (v1.31.0)
kustomize build apps/production | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.31.0
```

### Check Flux Status (requires cluster access)

```bash
flux get all                           # View all Flux resources
flux get kustomizations                # Check sync status
flux get helmreleases -A               # Check Helm deployments
flux reconcile kustomization apps --with-source  # Force sync
```

### Create Sealed Secrets

```bash
# Create plain secret (dry-run), then seal it
kubectl create secret generic my-secret \
  --namespace my-app \
  --from-literal=key=value \
  --dry-run=client -o yaml | \
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=flux-system \
  > apps/production/my-app/sealedsecret.yaml
```

## Architecture

### Deployment Order (Flux Dependencies)

```
Repositories → Infrastructure Controllers → Infrastructure Configs → Apps
```

Each stage uses `dependsOn` and health checks to wait for the previous stage.

### Directory Structure

- `clusters/production/` - Flux bootstrap and sync configuration
- `infrastructure/controllers/` - Core services: Traefik, cert-manager, Longhorn, Sealed Secrets, kube-vip, Reloader
- `infrastructure/configs/` - Post-install configs: ClusterIssuers, middlewares, network policies
- `apps/base/` - Environment-agnostic application definitions
- `apps/production/` - Production-specific patches, secrets, and overrides
- `repositories/` - Helm repository definitions

### Kustomization Pattern

Applications follow base/overlay pattern:
- `apps/base/<app>/` contains namespace, HelmRelease, and Ingress
- `apps/production/<app>/` patches versions, domains, TLS, and adds SealedSecrets

### Key Configuration

- HelmReleases deployed to `flux-system` namespace with `targetNamespace` pointing to app namespace
- Ingress uses `ingressClassName: traefik` with `cert-manager.io/cluster-issuer: letsencrypt` annotation
- Storage uses `storageClassName: longhorn`

## CI Validation

GitHub Actions runs on all PRs and pushes to main:
1. YAML lint (relaxed rules, no line-length limit)
2. Kustomize build for infrastructure/controllers, infrastructure/configs, apps/production
3. Kubeconform schema validation against Kubernetes 1.31.0
4. Flux resource validation

## Adding New Applications

1. Add Helm repository to `repositories/` if needed
2. Create `apps/base/<app>/` with kustomization.yaml, namespace.yaml, helmrelease.yaml, ingress.yaml
3. Create `apps/production/<app>/` with kustomization.yaml and patches for version, domain, TLS
4. Add sealed secrets if needed
5. Add to `apps/production/kustomization.yaml`
6. Test with `kustomize build apps/production/<app>`
