# Documentation Index

This folder contains detailed documentation for the fleet-infra repository.

## Contents

| Document | Description |
|----------|-------------|
| [Architecture](architecture.md) | GitOps model, deployment order, network/storage diagrams |
| [Infrastructure](infrastructure.md) | Traefik, cert-manager, Longhorn, Sealed Secrets, etc. |
| [Applications](apps.md) | Deployed applications and their configuration |
| [Adding Apps](adding-apps.md) | Step-by-step guide for deploying new applications |
| [Sealed Secrets](sealed-secrets.md) | How to create and manage encrypted secrets |

## Quick Links

### Common Tasks

- **Add a new app**: [Adding Apps Guide](adding-apps.md)
- **Create a secret**: [Sealed Secrets Guide](sealed-secrets.md#creating-sealed-secrets)
- **Check deployment status**: See [README.md](../README.md#check-status)
- **Troubleshoot issues**: See [README.md](../README.md#troubleshooting)

### Architecture Decisions

- **Why Flux?** - GitOps with strong Helm support, mature ecosystem
- **Why Traefik?** - Native Kubernetes CRDs, middleware support, OIDC plugin
- **Why Longhorn?** - Distributed storage without cloud dependencies
- **Why Sealed Secrets?** - Simple, GitOps-friendly secret management

## Contributing

When making changes:

1. Test manifests locally: `kustomize build apps/production/my-app`
2. Validate YAML syntax
3. Commit with descriptive messages
4. Push and monitor Flux reconciliation
