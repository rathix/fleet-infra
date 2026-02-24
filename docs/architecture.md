# Architecture Overview

This document describes the overall architecture of the fleet-infra GitOps system.

## GitOps Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              GitHub Repository                               │
│                               (fleet-infra)                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ Git Poll (10m)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Flux Source Controller                             │
│                          (GitRepository resource)                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ Triggers
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Flux Kustomize Controller                            │
│                        (Kustomization resources)                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
            ┌───────────┐     ┌───────────────┐   ┌──────────┐
            │ Helm Ctrl │     │ Kustomize Ctrl│   │ Raw YAML │
            └───────────┘     └───────────────┘   └──────────┘
                    │                 │                 │
                    └─────────────────┴─────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────────┐
                    │        Kubernetes Cluster           │
                    │         (Production)                │
                    └─────────────────────────────────────┘
```

## Deployment Order

Flux uses `dependsOn` to ensure resources are deployed in the correct order:

### 1. Repositories (First)

Registers all Helm chart repositories so charts can be fetched.

```yaml
# clusters/production/repositories.yaml
spec:
  path: ./repositories
  interval: 10m
```

### 2. Infrastructure Controllers

Deploys core infrastructure components:

- **Sealed Secrets** - Encrypted secrets management
- **cert-manager** - TLS certificate automation
- **Traefik** - Ingress controller
- **Longhorn** - Distributed storage
- **kube-vip** - Virtual IP management
- **Reloader** - Config/secret change detection
- **metrics-server** - Kubernetes resource metrics

```yaml
# clusters/production/infrastructure-controllers.yaml
spec:
  dependsOn:
    - name: repositories
  healthChecks:
    - kind: HelmRelease
      name: sealed-secrets
    - kind: HelmRelease
      name: cert-manager
    - kind: HelmRelease
      name: traefik
    - kind: HelmRelease
      name: longhorn
```

### 3. Infrastructure Configs

Applies configurations that depend on controllers:

- **ClusterIssuers** - Let's Encrypt certificate issuers
- **Middlewares** - OIDC authentication (depends on Sealed Secrets for client credentials)
- **Wildcard certificate** - `*.kennyandries.com` via cert-manager
- **Notifications** - Flux GitHub status reporting

```yaml
# clusters/production/infrastructure-configs.yaml
spec:
  dependsOn:
    - name: infrastructure-controllers
```

### 4. Applications (Last)

Deploys user-facing applications after all infrastructure is ready:

```yaml
# clusters/production/apps.yaml
spec:
  dependsOn:
    - name: infrastructure-configs
  healthChecks:
    - kind: HelmRelease
      name: servarr
    - kind: HelmRelease
      name: uptime-kuma
```

## Network Architecture

```
                              Internet
                                  │
                                  ▼
                          ┌───────────────┐
                          │   Cloudflare  │
                          │   (DNS/CDN)   │
                          └───────────────┘
                                  │
                                  ▼
                          ┌───────────────┐
                          │   Hetzner     │
                          │   DNS API     │
                          └───────────────┘
                                  │
                                  ▼ traefik.kennyandries.com
                          ┌───────────────┐
                          │   kube-vip    │
                          │  192.168.1.3  │
                          └───────────────┘
                                  │
                                  ▼
                          ┌───────────────┐
                          │    Traefik    │
                          │   (Ingress)   │
                          └───────────────┘
                                  │
           ┌──────────────────────┼──────────────────────┐
           ▼                      ▼                      ▼
    ┌────────────┐        ┌────────────┐        ┌────────────┐
    │  Homepage  │        │  Servarr   │        │   Other    │
    │   :3000    │        │   :8080    │        │   Apps     │
    └────────────┘        └────────────┘        └────────────┘
```

## Storage Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Longhorn Manager                          │
│                     (Distributed Storage)                        │
└─────────────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
    ┌────────────┐       ┌────────────┐       ┌────────────┐
    │   Node 1   │       │   Node 2   │       │   Node 3   │
    │  Replica   │       │  Replica   │       │  Replica   │
    └────────────┘       └────────────┘       └────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │   NFS Backup Target │
                    │   192.168.1.21      │
                    │   /mnt/hpool/backups│
                    └─────────────────────┘
```

## Security Architecture

### TLS/SSL

1. **cert-manager** watches for Certificate resources
2. Creates ACME challenges via Hetzner DNS webhook
3. Stores certificates as Kubernetes Secrets
4. Traefik uses certificates for HTTPS termination

### Secrets Management

1. Secrets are encrypted locally using `kubeseal`
2. SealedSecrets are stored in Git
3. Sealed Secrets controller decrypts them in-cluster
4. Only the cluster can decrypt (private key never leaves cluster)

### Authentication

- **OIDC Provider**: Pocket-ID at `auth.kennyandries.com`
- **Protected Services**: Traefik dashboard, Longhorn UI
- **Middleware**: Traefik forward-auth plugin

## Cluster Configuration

Central configuration is stored in a ConfigMap:

```yaml
# clusters/production/cluster-config.yaml
data:
  cluster_name: production
  domain: kennyandries.com
  ingress_class: traefik
  storage_class: longhorn
  external_dns_target: traefik.kennyandries.com
```

This ConfigMap can be referenced by other resources for consistent configuration.
