# Servarr Migration Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Kubito servarr Helm chart with individual manifests using Hotio images.

**Architecture:** Each app gets its own `apps/<app>/` directory with deployment, service, ingress, and config PVC. Shared resources (namespace, NFS PV/PVC, network policy) live in `apps/servarr-common/`. Existing Longhorn PVCs are reused by name to preserve data.

**Tech Stack:** Kubernetes manifests, Flux CD, Hotio container images, Kustomize

**Spec:** `docs/superpowers/specs/2026-03-11-servarr-migration-design.md`

---

## Chunk 1: Shared Resources and First App

### Task 1: Create servarr-common

Shared namespace, NFS storage, and network policy.

**Files:**
- Create: `apps/servarr-common/kustomization.yaml`
- Create: `apps/servarr-common/namespace.yaml`
- Create: `apps/servarr-common/pv.yaml`
- Create: `apps/servarr-common/pvc-media.yaml`
- Create: `apps/servarr-common/networkpolicy.yaml`

- [ ] **Step 1: Create `apps/servarr-common/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - pv.yaml
  - pvc-media.yaml
  - networkpolicy.yaml
```

- [ ] **Step 2: Create `apps/servarr-common/namespace.yaml`**

Copy from existing `apps/servarr/namespace.yaml` (identical content):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: servarr
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
  labels:
    app.kubernetes.io/name: servarr
    pod-security.kubernetes.io/enforce: baseline
```

- [ ] **Step 3: Create `apps/servarr-common/pv.yaml`**

Copy from existing `apps/servarr/persistentvolume.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: servarr-jellyfin-media-nfs
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
  labels:
    app.kubernetes.io/name: servarr
spec:
  capacity:
    storage: 2000Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: pkb-nas-tns-1
    path: /mnt/hpool/media
```

- [ ] **Step 4: Create `apps/servarr-common/pvc-media.yaml`**

Copy from existing `apps/servarr/persistentvolumeclaim.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: servarr-jellyfin-media
  namespace: servarr
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
  labels:
    app.kubernetes.io/name: servarr
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  volumeName: servarr-jellyfin-media-nfs
  resources:
    requests:
      storage: 2000Gi
```

- [ ] **Step 5: Create `apps/servarr-common/networkpolicy.yaml`**

Based on existing policy, with added intra-namespace rule:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traefik-ingress
  namespace: servarr
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: traefik-system
    - from:
        - podSelector: {}
```

- [ ] **Step 6: Validate servarr-common**

Run: `kustomize build apps/servarr-common`
Expected: Renders 4 resources (Namespace, PV, PVC, NetworkPolicy) without errors.

- [ ] **Step 7: Commit**

```bash
git add apps/servarr-common/
git commit -m "Add servarr-common shared resources for manifest migration"
```

### Task 2: Create Sonarr app

First app — establishes the pattern all others follow.

**Files:**
- Create: `apps/sonarr/kustomization.yaml`
- Create: `apps/sonarr/pvc.yaml`
- Create: `apps/sonarr/deployment.yaml`
- Create: `apps/sonarr/service.yaml`
- Create: `apps/sonarr/ingress.yaml`

- [ ] **Step 1: Create `apps/sonarr/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - pvc.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

- [ ] **Step 2: Create `apps/sonarr/pvc.yaml`**

Reuses existing Longhorn PVC name. No `storageClassName` or `resources` — PVC already exists, these fields are immutable.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: servarr-servarr-sonarr
  namespace: servarr
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
  labels:
    app.kubernetes.io/name: sonarr
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
```

- [ ] **Step 3: Create `apps/sonarr/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarr
  namespace: servarr
  labels:
    app.kubernetes.io/name: sonarr
spec:
  replicas: 1
  revisionHistoryLimit: 3
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/name: sonarr
  template:
    metadata:
      labels:
        app.kubernetes.io/name: sonarr
    spec:
      securityContext:
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: sonarr
          image: ghcr.io/hotio/sonarr:release-4.0.16.2957
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: UMASK
              value: "002"
          ports:
            - name: http
              containerPort: 8989
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /ping
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ping
              port: http
            initialDelaySeconds: 15
            periodSeconds: 5
          volumeMounts:
            - name: config
              mountPath: /config
            - name: media
              mountPath: /data/media
          resources:
            limits:
              memory: 1Gi
              cpu: "1"
            requests:
              memory: 256Mi
              cpu: "0.2"
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: servarr-servarr-sonarr
        - name: media
          persistentVolumeClaim:
            claimName: servarr-jellyfin-media
```

- [ ] **Step 4: Create `apps/sonarr/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sonarr
  namespace: servarr
  labels:
    app.kubernetes.io/name: sonarr
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: sonarr
  ports:
    - name: http
      port: 8989
      targetPort: http
      protocol: TCP
```

- [ ] **Step 5: Create `apps/sonarr/ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sonarr
  namespace: servarr
  labels:
    app.kubernetes.io/name: sonarr
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: traefik-system-oidc@kubernetescrd
    external-dns.alpha.kubernetes.io/target: traefik.kennyandries.com
spec:
  ingressClassName: traefik-system-traefik
  rules:
    - host: sonarr.kennyandries.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sonarr
                port:
                  number: 8989
```

- [ ] **Step 6: Validate Sonarr**

Run: `kustomize build apps/sonarr`
Expected: Renders 4 resources (PVC, Deployment, Service, Ingress) without errors.

Run: `kustomize build apps/sonarr | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.31.0 -schema-location default -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' -summary`
Expected: All resources valid.

- [ ] **Step 7: Commit**

```bash
git add apps/sonarr/
git commit -m "Add Sonarr manifest with Hotio image"
```

---

## Chunk 2: Remaining *arr Apps

These all follow the Sonarr pattern with app-specific differences (image, port, health path, resources). All Hotio deployments include container-level `securityContext` with `allowPrivilegeEscalation: false` and `capabilities.drop: [ALL]`.

### Task 3: Create Radarr app

**Files:** Create `apps/radarr/{kustomization,pvc,deployment,service,ingress}.yaml`

- [ ] **Step 1: Create all Radarr manifests**

Same pattern as Sonarr with these differences:
- Image: `ghcr.io/hotio/radarr:release-6.0.4.9564`
- Port: `7878`
- Health path: `/ping`
- PVC name: `servarr-servarr-radarr`
- Host: `radarr.kennyandries.com`
- Resources: same as Sonarr (1Gi/1 limit, 256Mi/0.2 request)

- [ ] **Step 2: Validate**

Run: `kustomize build apps/radarr`

- [ ] **Step 3: Commit**

```bash
git add apps/radarr/
git commit -m "Add Radarr manifest with Hotio image"
```

### Task 4: Create Lidarr app

**Files:** Create `apps/lidarr/{kustomization,pvc,deployment,service,ingress}.yaml`

- [ ] **Step 1: Create all Lidarr manifests**

Differences from Sonarr:
- Image: `ghcr.io/hotio/lidarr:release-3.1.0.4085`
- Port: `8686`
- Health path: `/ping`
- PVC name: `servarr-servarr-lidarr`
- Host: `lidarr.kennyandries.com`

- [ ] **Step 2: Validate**

Run: `kustomize build apps/lidarr`

- [ ] **Step 3: Commit**

```bash
git add apps/lidarr/
git commit -m "Add Lidarr manifest with Hotio image"
```

### Task 5: Create Prowlarr app

**Files:** Create `apps/prowlarr/{kustomization,pvc,deployment,service,ingress}.yaml`

- [ ] **Step 1: Create all Prowlarr manifests**

Differences from Sonarr:
- Image: `ghcr.io/hotio/prowlarr:release-2.3.0.5951`
- Port: `9696`
- Health path: `/ping`
- PVC name: `servarr-servarr-prowlarr`
- Host: `prowlarr.kennyandries.com`
- Resources: 512Mi/0.5 limit, 128Mi/0.1 request
- **No media volume mount** — Prowlarr only manages indexers

- [ ] **Step 2: Validate**

Run: `kustomize build apps/prowlarr`

- [ ] **Step 3: Commit**

```bash
git add apps/prowlarr/
git commit -m "Add Prowlarr manifest with Hotio image"
```

---

## Chunk 3: Media Apps

### Task 6: Create Jellyfin app

**Files:** Create `apps/jellyfin/{kustomization,pvc,deployment,service,ingress}.yaml`

- [ ] **Step 1: Create all Jellyfin manifests**

Differences from Sonarr:
- Image: `ghcr.io/hotio/jellyfin:release-10.11.6`
- Port: `8096`
- Health path: `/health`
- PVC name: `servarr-servarr-jellyfin-config`
- Host: `jellyfin.kennyandries.com`
- Resources: 8Gi/4 limit, 512Mi/0.5 request
- **No OIDC middleware** on ingress (public-facing media server)

- [ ] **Step 2: Validate**

Run: `kustomize build apps/jellyfin`

- [ ] **Step 3: Commit**

```bash
git add apps/jellyfin/
git commit -m "Add Jellyfin manifest with Hotio image"
```

### Task 7: Create qBittorrent app

**Files:** Create `apps/qbittorrent/{kustomization,pvc,deployment,service,ingress}.yaml`

- [ ] **Step 1: Create all qBittorrent manifests**

Differences from Sonarr:
- Image: `ghcr.io/hotio/qbittorrent:release-5.0.4`
- Port: `8080`
- Health probe: `tcpSocket` on port `8080` (HTTP API requires auth, so use TCP liveness/readiness)
- PVC name: `servarr-servarr-qbittorrent`
- Host: `qbittorrent.kennyandries.com`
- **Note:** Current image tag `20.04.1` is very old. Hotio uses `release-5.0.4` (latest stable). The old tag appears to be a linuxserver convention. Use current Hotio release.

- [ ] **Step 2: Validate**

Run: `kustomize build apps/qbittorrent`

- [ ] **Step 3: Commit**

```bash
git add apps/qbittorrent/
git commit -m "Add qBittorrent manifest with Hotio image"
```

### Task 8: Create Jellyseerr app

**Files:** Create `apps/jellyseerr/{kustomization,pvc,deployment,service,ingress}.yaml`

- [ ] **Step 1: Create all Jellyseerr manifests**

Different from the Hotio pattern — uses official image which runs non-root natively:
- Image: `docker.io/fallenbagel/jellyseerr:2.7.3`
- Port: `5055`
- Health path: `/api/v1/status`
- PVC name: `servarr-servarr-jellyseerr`
- Host: `jellyseerr.kennyandries.com`
- Resources: 512Mi/0.5 limit, 256Mi/0.2 request
- **No OIDC middleware** on ingress (has its own auth)
- **No media volume mount** — Jellyseerr only talks to Sonarr/Radarr/Jellyfin via API
- **No PUID/PGID env vars** — not a Hotio image
- **Full security context:** `runAsUser: 1000`, `runAsGroup: 1000`, `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`

- [ ] **Step 2: Validate**

Run: `kustomize build apps/jellyseerr`

- [ ] **Step 3: Commit**

```bash
git add apps/jellyseerr/
git commit -m "Add Jellyseerr manifest with official image"
```

---

## Chunk 4: Wiring and Cleanup

### Task 9: Update Flux wiring

**Files:**
- Modify: `apps/kustomization.yaml`
- Modify: `clusters/production/apps.yaml`
- Modify: `repositories/repositories.yaml`

- [ ] **Step 1: Update `apps/kustomization.yaml`**

Replace `servarr` entry with the new directories:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - actualbudget
  - beszel
  - configarr
  - homepage
  - jellyfin
  - jellyseerr
  - lidarr
  - pocket-id
  - prowlarr
  - qbittorrent
  - radarr
  - sabnzbd
  - servarr-common
  - sonarr
  - trilium
  - uptime-kuma
```

- [ ] **Step 2: Update `clusters/production/apps.yaml`**

Remove the servarr HelmRelease health check:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infrastructure
  interval: 10m
  retryInterval: 2m
  timeout: 15m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps
  prune: true
  wait: true
  healthChecks:
    - apiVersion: helm.toolkit.fluxcd.io/v2
      kind: HelmRelease
      name: uptime-kuma
      namespace: flux-system
```

- [ ] **Step 3: Remove Kubito HelmRepository from `repositories/repositories.yaml`**

Delete the kubito entry (lines containing `name: kubito` and its spec). No other chart uses it.

- [ ] **Step 4: Validate full apps build**

Run: `kustomize build apps`
Expected: All resources render without errors.

Run: `kustomize build apps | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.31.0 -schema-location default -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' -summary`
Expected: All resources valid.

- [ ] **Step 5: Commit**

```bash
git add apps/kustomization.yaml clusters/production/apps.yaml repositories/repositories.yaml
git commit -m "Update Flux wiring for servarr manifest migration"
```

### Task 10: Remove old servarr Helm deployment

**Files:**
- Delete: `apps/servarr/` (entire directory)

- [ ] **Step 1: Delete old directory**

```bash
rm -rf apps/servarr/
```

- [ ] **Step 2: Validate**

Run: `kustomize build apps`
Expected: Still renders cleanly without the old servarr directory.

- [ ] **Step 3: Commit**

```bash
git rm -r apps/servarr/
git commit -m "Remove old servarr Helm chart deployment"
```

### Task 11: Final validation

- [ ] **Step 1: Run full validation suite**

```bash
# YAML lint
yamllint .

# Kustomize build
kustomize build infrastructure
kustomize build apps

# Schema validation
kustomize build apps | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.31.0 \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -summary
```

Expected: All pass.

---

## Pre-Deployment: Migration Safety (manual, requires cluster access)

Before pushing to main, run these commands against the live cluster to protect existing PVC data:

```bash
# Annotate all config PVCs to survive Helm uninstall
for pvc in servarr-servarr-jellyfin-config servarr-servarr-sonarr servarr-servarr-radarr \
  servarr-servarr-lidarr servarr-servarr-prowlarr servarr-servarr-qbittorrent \
  servarr-servarr-jellyseerr; do
  kubectl annotate pvc "$pvc" -n servarr helm.sh/resource-policy=keep
done

# Set all Longhorn PVs backing config PVCs to Retain
for pvc in servarr-servarr-jellyfin-config servarr-servarr-sonarr servarr-servarr-radarr \
  servarr-servarr-lidarr servarr-servarr-prowlarr servarr-servarr-qbittorrent \
  servarr-servarr-jellyseerr; do
  pv=$(kubectl get pvc "$pvc" -n servarr -o jsonpath='{.spec.volumeName}')
  kubectl patch pv "$pv" -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
done

# Verify all PVs are now Retain
kubectl get pv -o custom-columns='NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy' | grep servarr
```

After confirming PVCs are protected, push the branch and merge.
