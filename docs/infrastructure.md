# Infrastructure Components

This document details the infrastructure components deployed in the cluster.

## Overview

| Component | Version | Namespace | Purpose |
|-----------|---------|-----------|---------|
| Traefik | 38.0.2 | traefik-system | Ingress controller |
| cert-manager | 1.19.2 | cert-manager | Certificate management |
| Sealed Secrets | 2.18.0 | flux-system | Secret encryption |
| Longhorn | 1.11.0 | longhorn-system | Distributed storage |
| kube-vip | 1.0.3 | kube-system | Virtual IP |
| Reloader | 2.2.7 | reloader | Config reload |
| Metrics Server | 3.13.0 | kube-system | Resource metrics |

---

## Traefik

**Purpose**: Ingress controller for routing external traffic to services.

**Features**:
- Automatic HTTPS with Let's Encrypt
- HTTP to HTTPS redirect
- HTTP/3 support
- OIDC authentication middleware
- Dashboard at `traefik.kennyandries.com`

**Key Configuration**:

```yaml
# infrastructure/controllers/traefik/helmrelease.yaml
spec:
  values:
    providers:
      kubernetesCRD:
        allowCrossNamespace: true
    ingressRoute:
      dashboard:
        enabled: true
```

**Middlewares**:
- `redirect-https` - Redirects HTTP to HTTPS
- `traefik-oidc` - OIDC authentication for protected routes

---

## cert-manager

**Purpose**: Automated TLS certificate management using Let's Encrypt.

**Components**:
- cert-manager controller
- Hetzner DNS webhook (for DNS-01 challenges)

**ClusterIssuer**:

```yaml
# infrastructure/configs/cert-manager/clusterissuer.yaml
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    solvers:
      - dns01:
          webhook:
            groupName: acme.cert-manager.io
            solverName: hetzner
```

**Usage in Ingress**:

```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt
```

---

## Sealed Secrets

**Purpose**: Encrypt secrets for safe storage in Git.

**How it works**:
1. Generate a SealedSecret using `kubeseal`
2. Commit the encrypted secret to Git
3. Controller decrypts it in-cluster

**Creating a Sealed Secret**:

```bash
# Create a regular secret
kubectl create secret generic my-secret \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml > secret.yaml

# Seal it
kubeseal --format yaml < secret.yaml > sealedsecret.yaml
```

**Key Rotation**: The controller manages its own keys. Backup the private key for disaster recovery.

---

## Longhorn

**Purpose**: Cloud-native distributed storage for persistent volumes.

**Features**:
- Replicated storage across nodes
- Automated backups to NFS
- Snapshots and restore
- Dashboard at `longhorn.kennyandries.com`

**Storage Class**:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
```

**Backup Configuration**:

```yaml
# infrastructure/base/longhorn/values.yaml
defaultSettings:
  backupTarget: nfs://192.168.1.21:/mnt/hpool/backups/longhorn
```

---

## kube-vip

**Purpose**: Virtual IP for Kubernetes control plane high availability.

**Configuration**:
- Virtual IP: `192.168.1.3`
- Mode: DaemonSet on control plane nodes
- Protocol: ARP

**DaemonSet**:

```yaml
# infrastructure/controllers/kube-vip/daemonset.yaml
spec:
  containers:
    - name: kube-vip
      env:
        - name: vip_address
          value: "192.168.1.3"
        - name: vip_interface
          value: "eth0"
```

---

## Reloader

**Purpose**: Automatically restart pods when ConfigMaps or Secrets change.

**Usage**: Add annotation to Deployment:

```yaml
metadata:
  annotations:
    reloader.stakater.com/auto: "true"
```

Or watch specific resources:

```yaml
annotations:
  configmap.reloader.stakater.com/reload: "my-configmap"
  secret.reloader.stakater.com/reload: "my-secret"
```

---

## Metrics Server

**Purpose**: Provides resource metrics for Horizontal Pod Autoscaling (HPA) and `kubectl top`.

**Usage**:

```bash
# View node metrics
kubectl top nodes

# View pod metrics
kubectl top pods -A
```

---

## Directory Structure

```
infrastructure/
├── base/                    # Base definitions (namespace, common config)
│   ├── cert-manager/
│   ├── kube-vip/
│   ├── longhorn/
│   ├── sealed-secrets/
│   └── traefik/
├── controllers/             # HelmReleases for each component
│   ├── cert-manager/
│   ├── kube-vip/
│   ├── longhorn/
│   ├── reloader/
│   ├── sealed-secrets/
│   └── traefik/
└── configs/                 # Post-install configurations
    ├── cert-manager/        # ClusterIssuers
    └── traefik/             # Middlewares, default certs
```

## Health Checks

Flux monitors these HelmReleases before proceeding to the next stage:

- `sealed-secrets`
- `cert-manager`
- `traefik`
- `longhorn`

If any fails, dependent Kustomizations will not be applied.
