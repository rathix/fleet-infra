# Backup & Disaster Recovery

## Overview

The cluster is fully IaC — rebuildable from Git and two offline keys. No etcd backups are needed because Flux re-syncs all resources from Git and Longhorn restores volume data from NFS.

## What's Backed Up

| Layer | Backup method | Recovery method |
|-|-|-|
| App/infra manifests | Git (fleet-infra repo) | Flux auto-reconciles from `main` |
| Talos node configs | Git (`talos/talconfig.yaml` + SOPS-encrypted secrets) | `talhelper genconfig` + `talosctl apply-config` |
| PVC data (app volumes) | Longhorn snapshots → NFS | Restore via Longhorn UI/API |
| Kubernetes secrets | SealedSecrets committed in Git | Flux re-applies, controller decrypts |
| Cluster state (etcd) | Not backed up — fully rebuildable | Fresh `talosctl bootstrap`, Flux re-syncs |

## Critical Keys

These two keys are the **only pieces not stored in Git**. Without them, recovery requires recreating all secrets from scratch. Back them up once to a password manager (1Password, Bitwarden, etc.).

### 1. Age Private Key

Located at `~/.config/sops/age/keys.txt`. Used to decrypt `talos/talsecret.sops.yaml` (Talos cluster PKI, tokens, etcd CA).

If lost: you must regenerate all Talos cluster secrets and re-bootstrap the cluster.

### 2. Sealed Secrets Controller Key

The private key the controller uses to decrypt all SealedSecret resources in Git.

Export it:

```bash
kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-key-backup.yaml
# Store contents in password manager, then delete the local file
rm sealed-secrets-key-backup.yaml
```

If lost: every SealedSecret in the repo becomes undecryptable. You'd need to recreate all application secrets from their original values and re-seal them.

## Disaster Recovery Procedures

### Total Cluster Loss (all nodes)

All nodes are destroyed or unrecoverable.

1. Flash Talos on all nodes (USB installer or PXE)
2. Generate configs:
   ```bash
   cd talos
   talhelper genconfig
   ```
3. Apply config to each node:
   ```bash
   talosctl apply-config --insecure -n 192.168.1.11 -f clusterconfig/production-pkb-k3s-tal-1.yaml
   talosctl apply-config --insecure -n 192.168.1.12 -f clusterconfig/production-pkb-k3s-tal-2.yaml
   talosctl apply-config --insecure -n 192.168.1.13 -f clusterconfig/production-pkv-k3s-tal-3.yaml
   ```
4. Bootstrap etcd on **one** node only:
   ```bash
   talosctl bootstrap -n 192.168.1.11
   ```
5. Wait for all nodes to become Ready:
   ```bash
   kubectl get nodes --watch
   ```
6. Restore Sealed Secrets controller key (from password manager backup):
   ```bash
   kubectl apply -f sealed-secrets-key-backup.yaml
   ```
7. Bootstrap Flux:
   ```bash
   flux bootstrap github \
     --owner=<github-user> \
     --repository=fleet-infra \
     --branch=main \
     --path=clusters/production
   ```
   Flux will reconcile all resources from Git automatically.
8. Restore Longhorn volumes from NFS backups via the Longhorn UI once Longhorn is running.

**Expected recovery time:** ~15-20 minutes for cluster + Flux, then application-dependent for volume restores.

### Single Node Loss

One node is destroyed; the other two are healthy.

1. Flash Talos on replacement hardware
2. Apply config:
   ```bash
   talosctl apply-config --insecure -n <node-ip> -f clusterconfig/<node-config>.yaml
   ```
3. Node auto-joins the existing etcd cluster
4. Flux automatically schedules workloads onto the new node

No manual intervention needed beyond applying the config.

### Sealed Secrets Key Loss (controller redeployed without key)

The Sealed Secrets controller was redeployed and lost its signing key.

1. Restore the key from your password manager:
   ```bash
   kubectl apply -f sealed-secrets-key-backup.yaml
   ```
2. Restart the controller to pick up the restored key:
   ```bash
   kubectl rollout restart deployment -n sealed-secrets sealed-secrets-controller
   ```
3. Flux will automatically reconcile all SealedSecrets — no further action needed.

### Age Key Loss (no backup available)

The age private key is lost and no backup exists. This is the worst-case scenario.

1. Generate new cluster secrets:
   ```bash
   cd talos
   talhelper gensecret > talsecret.sops.yaml
   sops -e -i talsecret.sops.yaml
   ```
2. Regenerate and apply configs:
   ```bash
   talhelper genconfig
   # Apply to all nodes
   ```
3. This changes all cluster PKI — effectively a full cluster rebuild. All nodes must rejoin with new certificates.
4. Back up the new age key immediately.

## Maintenance Checklist

- [ ] Age private key stored in password manager
- [ ] Sealed Secrets controller key stored in password manager
- [ ] Longhorn NFS backup target is on a separate physical machine
- [ ] Longhorn backup schedule is configured and tested
- [ ] Test a Longhorn volume restore periodically
- [ ] After rotating Sealed Secrets key, update the password manager backup
