# Upgrading Talos & Kubernetes

This guide covers upgrading Talos OS and Kubernetes on the cluster. These are **independent operations** — you can upgrade one without the other, but they must stay within the [Talos support matrix](https://www.talos.dev/latest/introduction/support-matrix/).

## Prerequisites

- `talosctl` and `talhelper` installed
- `kubectl` access to the cluster
- Age key available (`SOPS_AGE_KEY_FILE` set)
- All nodes healthy and etcd quorum intact

## Pre-upgrade Checks

Always run these before any upgrade:

```bash
# All nodes Ready
kubectl get nodes -o wide

# etcd is healthy on all members
talosctl -n 192.168.1.11 etcd status

# No stuck Flux reconciliations
flux get kustomizations

# Validate talconfig
talhelper validate talconfig
```

## Upgrading Talos OS

Talos upgrades replace the OS image (kernel, initramfs, extensions) without touching Kubernetes components.

### 1. Update the version

Edit `talos/talconfig.yaml`:

```yaml
talosVersion: v1.12.0  # new version
```

### 2. Generate configs

```bash
cd talos
talhelper genconfig
```

### 3. Validate generated configs

```bash
talhelper validate nodeconfig clusterconfig/*.yaml
```

### 4. Generate upgrade commands

```bash
talhelper gencommand upgrade
```

This outputs a `talosctl upgrade` command per node with the correct factory image. Nodes with `machineSpec.secureboot: true` automatically get the `installer-secureboot` (UKI) image variant.

### 5. Upgrade one node at a time

**Order does not matter**, but only upgrade one node at a time — losing 2 of 3 etcd members kills quorum.

Run the generated command for the target node, e.g.:

```bash
talosctl upgrade -n 192.168.1.12 \
  --image factory.talos.dev/installer-secureboot/<schematic-id>:v1.12.0 \
  --preserve
```

The `--preserve` flag keeps data partitions intact. For nodes with TPM disk encryption (pkb-k3s-tal-1, pkb-k3s-tal-2), LUKS2 keys are re-sealed automatically during the upgrade.

### 6. Wait for the node to rejoin

```bash
# Watch the upgrade progress
talosctl -n 192.168.1.12 dmesg --follow

# Wait for health
talosctl -n 192.168.1.12 health --wait-timeout 10m

# Confirm node is Ready in Kubernetes
kubectl get nodes
```

### 7. Repeat for remaining nodes

Only proceed to the next node after the previous one is fully healthy and has rejoined etcd.

## Upgrading Kubernetes

Kubernetes upgrades roll the control plane components (API server, scheduler, controller-manager) and then the kubelets.

### 1. Update the version

Edit `talos/talconfig.yaml`:

```yaml
kubernetesVersion: v1.35.0  # new version
```

### 2. Regenerate and apply configs

```bash
cd talos
talhelper genconfig
talhelper gencommand apply
```

Apply the generated commands to each node so they have the updated config.

### 3. Run the Kubernetes upgrade

```bash
talosctl upgrade-k8s --to v1.35.0 -n 192.168.1.11
```

You only run this against **one node** — Talos coordinates the rolling upgrade of all control plane components and kubelets across the cluster automatically.

### 4. Verify

```bash
kubectl get nodes -o wide   # All nodes should show the new version
kubectl get pods -A          # Check for crashlooping pods
flux get kustomizations      # Flux should be reconciling normally
```

## Upgrading Both

When upgrading both Talos and Kubernetes:

1. Upgrade **Talos OS first** (all nodes, one at a time)
2. Then upgrade **Kubernetes** (single command)

This order ensures the OS supports the target Kubernetes version before the control plane components are rolled.

## Rollback

### Talos OS

Talos keeps the previous OS image. To roll back a node:

```bash
talosctl rollback -n 192.168.1.12
```

### Kubernetes

Kubernetes downgrades are not officially supported. If an upgrade fails, the safest path is to fix forward or restore from the previous Talos config:

```bash
# Revert talconfig.yaml to the previous kubernetesVersion
talhelper genconfig
# Re-apply configs and re-run upgrade-k8s with the old version
```

## Node Reference

| Node | IP | Secure Boot | TPM FDE | Image variant |
|-|-|-|-|-|
| pkb-k3s-tal-1 | 192.168.1.11 | Yes | Yes | `installer-secureboot` |
| pkb-k3s-tal-2 | 192.168.1.12 | Yes | Yes | `installer-secureboot` |
| pkv-k3s-tal-3 | 192.168.1.13 | No | No | `installer` |

## Troubleshooting

### Node stuck in upgrading state

```bash
# Check Talos service logs
talosctl -n <ip> dmesg | tail -100
talosctl -n <ip> logs kubelet
```

### etcd not healthy after upgrade

```bash
# Check etcd member list
talosctl -n <ip> etcd members

# If a member failed to rejoin, remove and re-add it
talosctl -n <ip> etcd remove-member <member-id>
talosctl -n <ip> etcd leave
# Then reapply config to the node
```

### Kubelet not starting

```bash
talosctl -n <ip> logs kubelet
talosctl -n <ip> service kubelet
```

Common cause: Kubernetes version incompatible with the Talos version. Check the support matrix.
