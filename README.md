# Fleet Infrastructure

[![Validate](https://github.com/rathix/fleet-infra/actions/workflows/validate.yaml/badge.svg)](https://github.com/rathix/fleet-infra/actions/workflows/validate.yaml)

GitOps repository for managing Kubernetes cluster infrastructure and applications using Flux CD.

**Cluster:** Production
**Domain:** kennyandries.com

## Overview

This repository contains the Infrastructure as Code (IaC) for a production Kubernetes cluster. It uses [Flux CD](https://fluxcd.io/) for GitOps-based continuous delivery, automatically syncing the cluster state with this repository.

## Tech Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **GitOps** | Flux CD v2 | Continuous delivery and cluster synchronization |
| **Ingress** | Traefik | Load balancing, TLS termination, routing |
| **Certificates** | cert-manager | Automatic TLS certificate management |
| **Storage** | Longhorn | Distributed block storage for persistent volumes |
| **Secrets** | Sealed Secrets | Encrypted secrets stored in Git |
| **VIP** | kube-vip | Virtual IP for control plane high availability |
| **Config Reload** | Reloader | Automatic pod restart on config/secret changes |

## Repository Structure

```
fleet-infra/
├── clusters/
│   └── production/           # Cluster bootstrap and configuration
│       ├── flux-system/      # Flux components
│       ├── cluster-config.yaml
│       ├── repositories.yaml
│       ├── infrastructure-controllers.yaml
│       ├── infrastructure-configs.yaml
│       └── apps.yaml
├── infrastructure/
│   ├── controllers/          # Infrastructure controllers (Helm releases)
│   └── configs/              # Infrastructure configurations
├── apps/
│   ├── base/                 # Base application definitions
│   └── production/           # Production-specific overrides
├── repositories/             # Helm repository definitions
├── docs/                     # Additional documentation
└── renovate.json            # Automated dependency updates
```

## Deployment Flow

Flux deploys resources in a specific order based on dependencies:

```
┌─────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐     ┌──────┐
│ Repositories│ ──▶ │ Infrastructure Controllers│ ──▶ │ Infrastructure Configs  │ ──▶ │ Apps │
└─────────────┘     └─────────────────────────┘     └─────────────────────────┘     └──────┘
     10m                      10m                            10m                       10m
```

Each stage waits for the previous to be healthy before proceeding.

## Applications

| Application | Description | URL |
|-------------|-------------|-----|
| **Homepage** | Dashboard with service links | `homepage.kennyandries.com` |
| **Traefik** | Ingress dashboard | `traefik.kennyandries.com` |
| **Longhorn** | Storage dashboard | `longhorn.kennyandries.com` |
| **Uptime Kuma** | Status monitoring | `uptime-kuma.kennyandries.com` |
| **Pocket-ID** | Identity provider | `auth.kennyandries.com` |
| **Servarr** | Media management suite | Various subdomains |
| **Trilium** | Note-taking | `trilium.kennyandries.com` |
| **Actualbudget** | Personal finance | `actualbudget.kennyandries.com` |
| **Beszel** | System monitoring | `beszel.kennyandries.com` |
| **SABnzbd** | Usenet downloader | `sabnzbd.kennyandries.com` |
| **Metrics Server** | Kubernetes resource metrics | N/A |

## Quick Start

### Prerequisites

- Kubernetes cluster (Talos OS recommended)
- `kubectl` configured
- `flux` CLI installed
- GitHub personal access token

### Bootstrap Flux

```bash
flux bootstrap github \
  --owner=rathix \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/production \
  --personal
```

### Check Status

```bash
# View all Flux resources
flux get all

# Check Kustomization status
flux get kustomizations

# Check HelmReleases
flux get helmreleases -A

# View Flux logs
flux logs
```

## Documentation

- [Architecture Overview](docs/architecture.md)
- [Infrastructure Components](docs/infrastructure.md)
- [Applications Guide](docs/apps.md)
- [Adding New Applications](docs/adding-apps.md)
- [Sealed Secrets Guide](docs/sealed-secrets.md)

## Dependency Management

This repository uses [Renovate](https://github.com/renovatebot/renovate) for automated dependency updates:

- Helm chart versions are automatically updated
- Minor and patch updates are auto-merged
- Major updates create PRs for review

## Directory Conventions

### Base vs Production

- `base/` - Generic, environment-agnostic definitions
- `production/` - Environment-specific patches, secrets, and overrides

### Kustomization Pattern

Each application follows this structure:

```
apps/
├── base/
│   └── my-app/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       ├── helmrelease.yaml
│       └── ingress.yaml
└── production/
    └── my-app/
        ├── kustomization.yaml
        ├── helmrelease-patch.yaml    # Version/config overrides
        ├── ingress-patch.yaml        # Domain/TLS settings
        └── sealedsecret.yaml         # Encrypted secrets
```

## Troubleshooting

### Force Reconciliation

```bash
# Reconcile a specific Kustomization
flux reconcile kustomization apps --with-source

# Reconcile a HelmRelease
flux reconcile helmrelease servarr -n flux-system
```

### View Resource Status

```bash
# Describe a failing HelmRelease
kubectl describe helmrelease servarr -n flux-system

# Check pod logs
kubectl logs -n <namespace> deployment/<name>

# View events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Suspend/Resume

```bash
# Suspend a HelmRelease (stop reconciliation)
flux suspend helmrelease servarr -n flux-system

# Resume
flux resume helmrelease servarr -n flux-system
```

## License

Private repository. All rights reserved.
