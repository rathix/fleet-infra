# Servarr Migration: Helm to Individual Manifests

## Goal

Replace the Kubito servarr Helm chart with individual Kubernetes manifests using Hotio images. Drop linuxserver images. Hotio uses s6-overlay (starts as root, drops to PUID/PGID) — the app process runs as non-root, but containers cannot use `runAsNonRoot: true`.

## Apps

| App | Image | Port |
|-|-|-|
| Jellyfin | ghcr.io/hotio/jellyfin | 8096 |
| Sonarr | ghcr.io/hotio/sonarr | 8989 |
| Radarr | ghcr.io/hotio/radarr | 7878 |
| Lidarr | ghcr.io/hotio/lidarr | 8686 |
| Prowlarr | ghcr.io/hotio/prowlarr | 9696 |
| qBittorrent | ghcr.io/hotio/qbittorrent | 8080 |
| Jellyseerr | docker.io/fallenbagel/jellyseerr | 5055 |

Dropped: FlareSolverr, Bazarr, Cleanuparr.

## Directory Structure

```
apps/servarr-common/
  kustomization.yaml
  namespace.yaml
  pv.yaml
  pvc-media.yaml
  networkpolicy.yaml

apps/<app>/
  kustomization.yaml
  pvc.yaml
  deployment.yaml
  service.yaml
  ingress.yaml
```

## Shared Resources (servarr-common)

### Namespace

`servarr` with `pod-security.kubernetes.io/enforce: baseline` and `kustomize.toolkit.fluxcd.io/prune: disabled`.

### NFS PV/PVC

Reuse existing:
- PV: `servarr-jellyfin-media-nfs` → `pkb-nas-tns-1:/mnt/hpool/media`, 2000Gi, ReadWriteMany
- PVC: `servarr-jellyfin-media` → bound to above PV

### Network Policy

Allow ingress from `traefik-system` namespace and intra-namespace communication. Deny all other ingress.

## Per-App Pattern

### Config PVCs (Longhorn)

Reuse existing PVC names so data is preserved:

| App | Existing PVC Name |
|-|-|
| Jellyfin | servarr-servarr-jellyfin-config |
| Sonarr | servarr-servarr-sonarr |
| Radarr | servarr-servarr-radarr |
| Lidarr | servarr-servarr-lidarr |
| Prowlarr | servarr-servarr-prowlarr |
| qBittorrent | servarr-servarr-qbittorrent |
| Jellyseerr | servarr-servarr-jellyseerr |

### Deployment

- Single replica, Recreate strategy (config volumes are RWO)
- `revisionHistoryLimit: 3`
- Hotio env: `PUID=1000`, `PGID=1000`, `UMASK=002`
- Two volume mounts: config PVC at `/config`, shared media PVC at `/data/media`
- Label: `app.kubernetes.io/name: <app>`
- Security context: `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault`
- Liveness/readiness probes: HTTP GET on `/` (or app-specific health endpoint)

**Jellyseerr exception:** Runs as non-root natively — use full security context (`runAsUser: 1000`, `runAsNonRoot: true`) without PUID/PGID env vars.

### Resource Limits

| App | Requests | Limits |
|-|-|-|
| Jellyfin | 512Mi / 0.5 | 8Gi / 4 |
| Sonarr | 256Mi / 0.2 | 1Gi / 1 |
| Radarr | 256Mi / 0.2 | 1Gi / 1 |
| Lidarr | 256Mi / 0.2 | 1Gi / 1 |
| Prowlarr | 128Mi / 0.1 | 512Mi / 0.5 |
| qBittorrent | 256Mi / 0.2 | 1Gi / 1 |
| Jellyseerr | 256Mi / 0.2 | 512Mi / 0.5 |

### Ingress

- `ingressClassName: traefik-system-traefik`
- No TLS block (wildcard cert at infrastructure level)
- Annotation: `traefik.ingress.kubernetes.io/router.middlewares: traefik-system-oidc@kubernetescrd` on all except Jellyfin and Jellyseerr
- Annotation: `external-dns.alpha.kubernetes.io/target: traefik.kennyandries.com`

### Service

ClusterIP, single port matching the app's default port.

## Migration Safety

Before removing the HelmRelease:

1. Annotate all existing PVCs with `helm.sh/resource-policy: keep`
2. Set all PVs to `persistentVolumeReclaimPolicy: Retain`
3. Remove the HelmRelease — Helm uninstall skips annotated PVCs
4. New manifests reference the same PVC names — Kubernetes binds to existing volumes

## Flux Wiring

- Remove `apps/servarr` from `apps/kustomization.yaml`
- Add `apps/servarr-common` and all 7 app directories (servarr-common listed first for resource ordering)
- Update `clusters/production/apps.yaml`: remove the servarr HelmRelease health check
- Remove Kubito HelmRepository from `repositories/repositories.yaml` if unused by other charts

## What Gets Deleted

- `apps/servarr/` directory (entire Helm deployment)
- FlareSolverr, Bazarr, Cleanuparr resources (dropped apps)
