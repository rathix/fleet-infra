# Fleet Infrastructure

[![Validate](https://github.com/rathix/fleet-infra/actions/workflows/validate.yaml/badge.svg)](https://github.com/rathix/fleet-infra/actions/workflows/validate.yaml)

GitOps repository for managing Kubernetes cluster infrastructure and applications using Flux CD.

**Cluster:** Production
**Domain:** kennyandries.com

## Overview

This repository contains the Infrastructure as Code (IaC) for a production Kubernetes cluster. It uses [Flux CD](https://fluxcd.io/) for GitOps-based continuous delivery, automatically syncing the cluster state with this repository.

## Tech Stack

| Component | Technology | Purpose |
|-|-|-|
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
│       ├── infrastructure.yaml
│       └── apps.yaml
├── infrastructure/           # Infrastructure services
│   ├── cert-manager/
│   ├── traefik/
│   ├── longhorn/
│   ├── sealed-secrets/
│   ├── kube-vip/
│   ├── reloader/
│   ├── metrics-server/
│   └── notifications/
├── apps/                     # Application definitions
│   ├── homepage/
│   ├── servarr/
│   ├── uptime-kuma/
│   └── ...
├── repositories/             # Helm repository definitions
└── docs/                     # Additional documentation
```

## Deployment Flow

Flux deploys resources in a specific order based on dependencies:

```
Repositories → Infrastructure → Apps
```

Each stage waits for the previous to be healthy before proceeding.

## Applications

| Application | Description | URL |
|-|-|-|
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

- [Adding New Applications](docs/adding-apps.md)
- [Sealed Secrets Guide](docs/sealed-secrets.md)

## Dependency Management

This repository uses [Renovate](https://github.com/renovatebot/renovate) for automated dependency updates:

- Helm chart versions are automatically updated
- Minor and patch updates are auto-merged
- Major updates create PRs for review

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
