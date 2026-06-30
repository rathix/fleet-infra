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
kustomize build infrastructure
kustomize build apps

# Validate against Kubernetes schemas (v1.35.0)
kustomize build apps | kubeconform -strict -ignore-missing-schemas -kubernetes-version 1.35.0 \
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
Repositories → Infrastructure → Apps
```

Each stage uses `dependsOn` and health checks to wait for the previous stage.

### Directory Structure

- `clusters/production/` - Flux bootstrap and sync configuration
- `infrastructure/` - Core services, each in `infrastructure/<service>/` with all manifests co-located (HelmRelease, middlewares, certificates, ClusterIssuers, etc.)
- `apps/` - Application definitions, each in a flat `apps/<app>/` directory with all manifests and network policies co-located
- `repositories/` - Helm repository definitions (single multi-doc YAML file)
- `docs/` - Additional documentation

### Kustomization Pattern

Both infrastructure and apps use a flat structure:
- `infrastructure/<service>/` contains all manifests for that service
- `apps/<app>/` contains all manifests directly (namespace, HelmRelease or Deployment/Service, Ingress, NetworkPolicy, SealedSecrets)
- No base/overlay split; values and configuration are defined inline

### Key Configuration

- HelmReleases deployed to `flux-system` namespace with `targetNamespace` pointing to app namespace
- TLS uses a wildcard certificate (`*.${domain}`) managed by cert-manager at `infrastructure/traefik/certificate.yaml`; individual ingresses do not need `cert-manager.io/cluster-issuer` annotations
- metrics-server is in `infrastructure/metrics-server/`

### Cluster-wide variables (Flux postBuild substitution)

Both the `apps` and `infrastructure` Flux Kustomizations apply `postBuild.substituteFrom` the `cluster-config` ConfigMap (`clusters/production/cluster-config.yaml`). Use these tokens in manifests instead of hardcoding:

- `${domain}` → `kennyandries.com` (e.g. `host: sonarr.${domain}`)
- `${storage_class}` → `longhorn` (e.g. `storageClassName: ${storage_class}`)
- `${ingress_class}` → `traefik-system-traefik` (e.g. `ingressClassName: ${ingress_class}`)

These are substituted by Flux at apply time, NOT by `kustomize build` — local builds and CI show the literal `${...}` strings, which is expected.

**Escaping:** substitution scans the **entire rendered output, including YAML comments and Helm-values block scalars**, so any literal `$` (shell scripts, env-var expansion in app configs, regex, comments) must be doubled as `$$`. Two failure modes:
- A valid-looking `${VAR}` that isn't in `cluster-config` is replaced with an empty string (silent breakage).
- An **invalid** variable name — e.g. `${...}` (dots aren't legal name chars) — makes envsubst fail with `unable to parse variable name` and **blocks the whole Kustomization** until fixed. This actually happened: a `${...}` left in a *comment* stalled the entire `infrastructure` Kustomization (fixed in `95bf7d5`).

Examples in-repo: `infrastructure/crowdsec/helmrelease.yaml` (`$${REGISTRATION_TOKEN}`), `apps/home-assistant/speech-to-phrase.yaml` (`$$HASS_TOKEN`). Prefer inlining values over shell variables, and don't write `${...}` placeholders in comments.

### Naming conventions

Principle: **identity at the boundary, role on the inside.** Outer names carry the app identity; inner names describe the role (so you never get `vaultwarden-vaultwarden`).

| Element | Convention | Example |
|-|-|-|
| Namespace | one per app, = app name, kebab-case | `vaultwarden`, `home-assistant` |
| HelmRelease / release | = app name, in `flux-system` | `name: vaultwarden` |
| Controller (app-template) | `main` for the primary workload; role-named for extras | `main`, `worker` |
| Container | `app` for the primary; role for sidecars/init | `app`, `exporter`, `init-db` |
| Service | `app` for primary; role otherwise | `app` |
| Service port | protocol/role, not a number | `http`, `metrics` |
| Persistence (volume) | by purpose, **never** the app name | `config`, `data`, `media`, `cache`, `tmp` |
| Resulting PVC | auto-named `<release>-<purpose>` by app-template | `vaultwarden-data`, `sonarr-config` |
| Labels | let app-template set `app.kubernetes.io/{name,instance}` | (automatic) |

- **Namespaces:** default to one per app (it's the boundary for NetworkPolicy, PSS, RBAC). `servarr` is a deliberate exception — those apps share the media PVC and talk to each other.
- **Apps with a good upstream chart** (e.g. uptime-kuma) use that chart, not app-template; the standard is "every app is a `HelmRelease`", not "every app is app-template".
- See `apps/vaultwarden/` for the reference app-template layout.

### Shared app-template defaults

Universal pod security/options live once in `apps/_shared/app-template-defaults.yaml` (a ConfigMap in `flux-system`). app-template HelmReleases pull it in with `spec.valuesFrom` and add only their specifics inline (inline `values` deep-merge over the ConfigMap and win on conflicts). Apps supply identity (`runAsUser`/`runAsGroup`/`fsGroup`) and any overrides (e.g. `runAsNonRoot: false` for root workloads).

Don't re-specify app-template/k8s defaults: controller `type: deployment`, `replicas: 1`, and service `targetPort` (= `port`) are implicit. Do keep `probes.*.enabled: true` (probes are disabled by default), `revisionHistoryLimit` (k8s default is 10), and ingress `pathType`.

## CI Validation

GitHub Actions runs on all PRs and pushes to main:
1. YAML lint (relaxed rules, no line-length limit)
2. Kustomize build for infrastructure, apps
3. Kubeconform schema validation against Kubernetes 1.35.0 (with CRD schema support)
4. Flux resource validation
5. Kustomize diff summary (PRs only)

## Adding New Applications

Standard: **every app is a `HelmRelease`.** Prefer the app-template pattern below for self-built/simple apps; use a dedicated upstream chart when one exists (e.g. uptime-kuma). Follow the naming conventions and `${...}` substitution variables above. `apps/vaultwarden/` is the reference layout.

### app-template-based (preferred default)
1. Create `apps/<app>/` with:
   - `namespace.yaml` (one namespace per app, with the appropriate `pod-security.kubernetes.io/*` labels)
   - `helmrelease.yaml` (chart `app-template` from the `bjw-s` repo; `spec.valuesFrom` the `app-template-defaults` ConfigMap; app-specific `values` inline)
   - `networkpolicy.yaml`
   - `pvc.yaml` only if reusing an existing PVC via `existingClaim`; otherwise declare storage under `persistence` and let app-template create it
   - `sealedsecret.yaml` if needed
   - `kustomization.yaml`
2. Add to `apps/kustomization.yaml`
3. Test with `kustomize build apps/<app>`

### Upstream-chart-based
1. Add the Helm repository to `repositories/repositories.yaml` if needed
2. Create `apps/<app>/` with `namespace.yaml`, `helmrelease.yaml` (values inline), `networkpolicy.yaml`, sealed secrets if needed, `kustomization.yaml`
3. Add to `apps/kustomization.yaml`; test with `kustomize build apps/<app>`

### Manifest-based (legacy)
Existing apps may still use raw `deployment.yaml`/`service.yaml`/`ingress.yaml`. New apps should use a HelmRelease instead.

See [docs/adding-apps.md](docs/adding-apps.md) for detailed examples.

## Talos Cluster Management

Talos node configurations are managed with [talhelper](https://github.com/budimanjojo/talhelper) in the `talos/` directory. Secrets are encrypted with SOPS + age.

### Prerequisites

```bash
brew install talhelper sops age
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt  # also in ~/.zshrc
```

The age private key at `~/.config/sops/age/keys.txt` is required to decrypt secrets. Back this up securely.

### Directory Structure

- `talos/talconfig.yaml` - Talhelper config defining all nodes and patches
- `talos/talsecret.sops.yaml` - Encrypted cluster secrets (PKI, tokens, etcd CA)
- `talos/clusterconfig/` - Generated output (gitignored, never commit)
- `.sops.yaml` - SOPS encryption rules (age public key)

### Generate Configs

```bash
cd talos
talhelper genconfig
```

This reads `talconfig.yaml` + `talsecret.sops.yaml` and outputs per-node configs into `clusterconfig/`.

### Validate

```bash
talhelper validate talconfig                          # Check talconfig.yaml syntax
talhelper validate nodeconfig clusterconfig/*.yaml     # Check generated node configs
```

### Apply Config to a Node

```bash
# Generate talosctl apply-config commands for all nodes
talhelper gencommand apply

# Or apply to a specific node
talosctl apply-config -n 192.168.1.11 -f clusterconfig/production-pkb-k3s-tal-1.yaml
```

### Upgrade Talos Version

1. Update `talosVersion` in `talconfig.yaml`
2. Regenerate configs: `talhelper genconfig`
3. Generate upgrade commands: `talhelper gencommand upgrade`
4. Upgrade nodes one at a time, waiting for each to rejoin before proceeding

### Upgrade Kubernetes Version

1. Update `kubernetesVersion` in `talconfig.yaml`
2. Regenerate and apply configs
3. Run: `talosctl upgrade-k8s --to <version> -n 192.168.1.11`

### Rotate Secrets

```bash
talhelper gensecret > talsecret.sops.yaml
sops -e -i talsecret.sops.yaml
talhelper genconfig
# Apply to all nodes
```

### Cluster Nodes

| Hostname | IP | Type | Notes |
|-|-|-|-|
| pkb-k3s-tal-1 | 192.168.1.11 | Bare-metal CP | TPM disk encryption |
| pkb-k3s-tal-2 | 192.168.1.12 | Bare-metal CP | TPM disk encryption |
| pkv-k3s-tal-3 | 192.168.1.13 | VM CP | No FDE (host has FDE) |

VIP: `192.168.1.3` (kube.kennyandries.com)

## Backup & Disaster Recovery

The cluster is fully IaC — rebuildable from Git + two offline keys (age private key, Sealed Secrets controller key). No etcd backups needed. See [docs/backup-and-disaster-recovery.md](docs/backup-and-disaster-recovery.md) for full procedures.
