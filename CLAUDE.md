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
kustomize build apps

# Validate against Kubernetes schemas (v1.31.0)
kustomize build apps | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.31.0 \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -summary
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
  --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets \
  > apps/my-app/sealedsecret.yaml
```

## Architecture

### Deployment Order (Flux Dependencies)

```
Repositories → Infrastructure Controllers → Infrastructure Configs → Apps
```

Each stage uses `dependsOn` and health checks to wait for the previous stage.

### Directory Structure

- `clusters/production/` - Flux bootstrap and sync configuration
- `infrastructure/controllers/` - Core services: Traefik, cert-manager, Longhorn, Sealed Secrets, kube-vip, Reloader, metrics-server
- `infrastructure/configs/` - Post-install configs: ClusterIssuers, middlewares, certificates, notifications
- `apps/` - Application definitions, each in a flat `apps/<app>/` directory with all manifests and network policies co-located
- `repositories/` - Helm repository definitions
- `docs/` - Additional documentation

### Kustomization Pattern

Applications use a flat structure:
- `apps/<app>/` contains all manifests directly (namespace, HelmRelease or Deployment/Service, Ingress, NetworkPolicy, SealedSecrets)
- No base/overlay split; values and configuration are defined inline

### Key Configuration

- HelmReleases deployed to `flux-system` namespace with `targetNamespace` pointing to app namespace
- TLS uses a wildcard certificate (`*.kennyandries.com`) managed by cert-manager at `infrastructure/configs/traefik/certificate.yaml`; individual ingresses do not need `cert-manager.io/cluster-issuer` annotations
- Ingress uses `ingressClassName: traefik-system-traefik`
- Storage uses `storageClassName: longhorn`
- metrics-server is deployed as an infrastructure controller in `infrastructure/controllers/metrics-server/`

## CI Validation

GitHub Actions runs on all PRs and pushes to main:
1. YAML lint (relaxed rules, no line-length limit)
2. Kustomize build for infrastructure/controllers, infrastructure/configs, apps
3. Kubeconform schema validation against Kubernetes 1.31.0 (with CRD schema support)
4. Flux resource validation
5. Kustomize diff summary (PRs only)

## Adding New Applications

### Helm-based
1. Add Helm repository to `repositories/` if needed
2. Create `apps/<app>/` with kustomization.yaml, namespace.yaml, helmrelease.yaml (with values inline), ingress.yaml
3. Add network policy if needed (`networkpolicy.yaml`)
4. Add sealed secrets if needed
5. Add to `apps/kustomization.yaml`
6. Test with `kustomize build apps/<app>`

### Manifest-based
1. Create `apps/<app>/` with kustomization.yaml, namespace.yaml, deployment.yaml, service.yaml, ingress.yaml
2. Add network policy and sealed secrets if needed
3. Add to `apps/kustomization.yaml`
4. Test with `kustomize build apps/<app>`

See [docs/adding-apps.md](docs/adding-apps.md) for detailed examples.
