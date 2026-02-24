# Applications Guide

This document details the applications deployed in the cluster.

## Overview

| Application | Type | Description | URL |
|-------------|------|-------------|-----|
| Homepage | Manifest | Dashboard/landing page | `homepage.kennyandries.com` |
| Servarr | Helm | Media management suite | Various |
| Uptime Kuma | Helm | Status monitoring | `uptime-kuma.kennyandries.com` |
| Pocket-ID | Manifest | OIDC identity provider | `auth.kennyandries.com` |
| Trilium | Helm | Note-taking app | `trilium.kennyandries.com` |
| Actualbudget | Helm | Personal finance | `actualbudget.kennyandries.com` |
| Beszel | Manifest | System monitoring | `beszel.kennyandries.com` |
| SABnzbd | Manifest | Usenet downloader | `sabnzbd.kennyandries.com` |
| ConfigArr | CronJob | Servarr configuration sync | N/A |



---

## Homepage

**Description**: Customizable dashboard displaying service links, bookmarks, and cluster status.

**Type**: Raw Kubernetes manifests (Deployment, Service, Ingress, RBAC)

**Features**:
- Service autodiscovery via Kubernetes API
- Custom bookmarks and widgets
- Real-time cluster resource monitoring

**Configuration Files**:
```
apps/homepage/
├── kustomization.yaml
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── rbac.yaml
├── configmap.yaml          # bookmarks, services, widgets, settings
└── networkpolicy.yaml
```

**Key ConfigMap Sections**:
- `bookmarks.yaml` - External links organized by category
- `services.yaml` - Service definitions with icons
- `widgets.yaml` - Dashboard widgets (clock, search, etc.)
- `kubernetes.yaml` - Cluster integration settings

---

## Servarr Stack

**Description**: Media automation suite for managing TV shows, movies, and downloads.

**Chart**: `kubito/servarr`
**Version**: 1.2.x

**Included Applications**:
- **Jellyfin** - Media server (`jellyfin.kennyandries.com`)
- **Jellyseerr** - Media request management (`jellyseerr.kennyandries.com`)
- **Sonarr** - TV show management (`sonarr.kennyandries.com`)
- **Radarr** - Movie management (`radarr.kennyandries.com`)
- **Lidarr** - Music management (`lidarr.kennyandries.com`)
- **Prowlarr** - Indexer management (`prowlarr.kennyandries.com`)
- **Bazarr** - Subtitle management (`bazarr.kennyandries.com`)
- **qBittorrent** - Torrent client (`qbittorrent.kennyandries.com`)
- **Flaresolverr** - CAPTCHA solver (no ingress)

**Storage**:
- Config volumes per application
- Shared media volume (NFS or Longhorn)

---

## Uptime Kuma

**Description**: Self-hosted monitoring tool for tracking service availability.

**Chart**: `dirsigler/uptime-kuma`
**Version**: 2.24.x

**Features**:
- HTTP/HTTPS monitoring
- TCP/Ping checks
- Status pages
- Notifications (Discord, Telegram, etc.)

---

## Pocket-ID

**Description**: Lightweight OIDC identity provider for single sign-on.

**Type**: Raw Kubernetes manifests (Deployment, Service, Ingress, PVC)

**Usage**:
- Provides authentication for Traefik-protected services
- Configured as OIDC provider at `auth.kennyandries.com`

**Protected Services**:
- Traefik Dashboard
- Longhorn UI
- Sonarr, Radarr, Lidarr, Prowlarr, Bazarr, qBittorrent
- Homepage

---

## Trilium

**Description**: Hierarchical note-taking application with rich features.

**Chart**: `triliumnext/trilium`
**Version**: 1.3.x

**Features**:
- Tree-structured notes
- Rich text editing
- Code blocks with syntax highlighting
- Note encryption

---

## Actualbudget

**Description**: Privacy-focused personal finance and budgeting tool.

**Chart**: `actualbudget`
**Version**: 1.8.x

**Features**:
- Envelope budgeting
- Bank sync (optional)
- Reports and analytics
- Multi-device sync

---

## Beszel

**Description**: Lightweight system monitoring with a DaemonSet agent.

**Type**: Raw Kubernetes manifests (Deployment + DaemonSet)
**Version**: 0.18.x

**Components**:
- **Hub** (Deployment) - Central dashboard
- **Agent** (DaemonSet) - Runs on each node

**Metrics Collected**:
- CPU, memory, disk usage
- Network I/O
- Process information

---

## SABnzbd

**Description**: Usenet binary newsreader for automated downloads.

**Type**: Raw Kubernetes manifests
**Version**: Latest

**Storage**:
- Config volume (Longhorn)
- Downloads volume (NFS recommended)

---

## ConfigArr

**Description**: Automated configuration management for Servarr applications.

**Type**: CronJob
**Schedule**: Hourly (`0 * * * *`)

**Purpose**:
- Syncs configuration across Servarr apps
- Applies predefined quality profiles
- Manages indexer settings

---

## Application Structure

Each application uses a flat directory structure with all manifests co-located:

```
apps/
└── <app-name>/
    ├── kustomization.yaml    # Lists all resources
    ├── namespace.yaml        # App namespace
    ├── helmrelease.yaml      # Helm chart with values inline (or deployment.yaml + service.yaml)
    ├── ingress.yaml          # Ingress route (if applicable)
    ├── networkpolicy.yaml    # Network policy (if applicable)
    └── sealedsecret.yaml     # Encrypted secrets (if applicable)
```

## Common Patterns

### Ingress

TLS is handled by a wildcard certificate (`*.kennyandries.com`) managed at `infrastructure/configs/traefik/certificate.yaml`. Individual ingresses do not need TLS blocks or cert-manager annotations.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
spec:
  ingressClassName: traefik-system-traefik
  rules:
    - host: myapp.kennyandries.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

Middleware annotations can be added directly to the ingress:

```yaml
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-security-headers@kubernetescrd,traefik-system-rate-limit@kubernetescrd
```

### Persistent Volume Claim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

### HelmRelease with Values

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 10m
  chart:
    spec:
      chart: my-app
      version: "1.x.x"
      sourceRef:
        kind: HelmRepository
        name: my-repo
  values:
    replicaCount: 1
    image:
      tag: latest
    persistence:
      enabled: true
      storageClass: longhorn
```
