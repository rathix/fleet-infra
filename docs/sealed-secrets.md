# Sealed Secrets Guide

This document explains how to manage secrets securely in the fleet-infra repository.

## Overview

[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) allows you to encrypt Kubernetes Secrets so they can be safely stored in Git. Only the Sealed Secrets controller running in the cluster can decrypt them.

## How It Works

```
┌─────────────────┐     kubeseal      ┌─────────────────┐
│  Plain Secret   │ ───────────────▶  │  SealedSecret   │
│  (local only)   │                   │  (safe for Git) │
└─────────────────┘                   └─────────────────┘
                                              │
                                              │ git push
                                              ▼
                                      ┌─────────────────┐
                                      │    Git Repo     │
                                      └─────────────────┘
                                              │
                                              │ Flux sync
                                              ▼
                                      ┌─────────────────┐
                                      │   Controller    │
                                      │   (decrypts)    │
                                      └─────────────────┘
                                              │
                                              ▼
                                      ┌─────────────────┐
                                      │ Kubernetes      │
                                      │ Secret          │
                                      └─────────────────┘
```

## Prerequisites

### Install kubeseal CLI

**macOS**:
```bash
brew install kubeseal
```

**Linux**:
```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | jq -r .tag_name | cut -c 2-)
wget "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz
sudo install kubeseal /usr/local/bin/
```

### Verify Controller

```bash
kubectl get pods -n flux-system -l app.kubernetes.io/name=sealed-secrets
```

## Creating Sealed Secrets

### Method 1: From Literal Values

```bash
# Create a regular secret (don't apply it!)
kubectl create secret generic my-secret \
  --namespace my-app \
  --from-literal=username=admin \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml > /tmp/secret.yaml

# Seal it
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=flux-system \
  < /tmp/secret.yaml \
  > apps/production/my-app/sealedsecret.yaml

# Clean up plain secret
rm /tmp/secret.yaml
```

### Method 2: From Files

```bash
# Create secret from file
kubectl create secret generic my-tls \
  --namespace my-app \
  --from-file=tls.crt=./certificate.crt \
  --from-file=tls.key=./certificate.key \
  --dry-run=client -o yaml > /tmp/secret.yaml

# Seal it
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=flux-system \
  < /tmp/secret.yaml \
  > apps/production/my-app/tls-sealedsecret.yaml
```

### Method 3: From .env File

```bash
# Create secret from env file
kubectl create secret generic my-env \
  --namespace my-app \
  --from-env-file=.env \
  --dry-run=client -o yaml > /tmp/secret.yaml

# Seal it
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=flux-system \
  < /tmp/secret.yaml \
  > apps/production/my-app/env-sealedsecret.yaml
```

## Sealed Secret Structure

A SealedSecret looks like this:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: my-secret
  namespace: my-app
spec:
  encryptedData:
    username: AgBy8h...  # Encrypted value
    password: AgCtr2...  # Encrypted value
  template:
    metadata:
      name: my-secret
      namespace: my-app
    type: Opaque
```

## Scopes

Sealed Secrets support different scopes that control where the secret can be decrypted:

### Strict (Default)

Secret can only be decrypted in the exact namespace and with the exact name.

```bash
kubeseal --scope strict ...
```

### Namespace-wide

Secret can be decrypted with any name, but only in the specified namespace.

```bash
kubeseal --scope namespace-wide ...
```

### Cluster-wide

Secret can be decrypted anywhere in the cluster.

```bash
kubeseal --scope cluster-wide ...
```

## Updating Secrets

To update a sealed secret:

1. Create the new secret with updated values
2. Re-seal it (this generates new encrypted data)
3. Replace the old SealedSecret file
4. Commit and push

```bash
# Update a single key
kubectl create secret generic my-secret \
  --namespace my-app \
  --from-literal=password=newpassword \
  --dry-run=client -o yaml | \
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=flux-system \
  --merge-into apps/production/my-app/sealedsecret.yaml
```

## Backup & Recovery

### Backup the Sealing Key

The controller generates a sealing key pair. **If lost, all existing SealedSecrets become undecryptable.**

```bash
# Backup the private key
kubectl get secret -n flux-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key-backup.yaml

# Store this securely (NOT in Git!)
```

### Restore the Sealing Key

```bash
kubectl apply -f sealed-secrets-key-backup.yaml
kubectl rollout restart deployment sealed-secrets -n flux-system
```

## Fetching the Public Key

If you need to seal secrets without cluster access:

```bash
# Fetch and save public key
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=flux-system \
  > sealed-secrets-cert.pem

# Use offline
kubeseal --cert sealed-secrets-cert.pem --format yaml < secret.yaml > sealedsecret.yaml
```

## Troubleshooting

### Check Controller Logs

```bash
kubectl logs -n flux-system deployment/sealed-secrets
```

### Verify Decryption

```bash
# Check if the Secret was created
kubectl get secret my-secret -n my-app

# View decrypted values (base64 encoded)
kubectl get secret my-secret -n my-app -o jsonpath='{.data.password}' | base64 -d
```

### Common Issues

**"no key could decrypt secret"**
- The SealedSecret was created with a different sealing key
- Re-seal the secret with the current controller's key

**"namespace mismatch"**
- The SealedSecret has `scope: strict` but you're trying to use it in a different namespace
- Re-seal with the correct namespace or use `--scope namespace-wide`

## Best Practices

1. **Never commit plain secrets** - Always seal before committing
2. **Clean up temp files** - Remove `/tmp/secret.yaml` after sealing
3. **Backup sealing keys** - Store securely outside the cluster
4. **Use strict scope** - Unless you specifically need flexibility
5. **Rotate secrets** - Re-seal periodically for security
6. **Document secrets** - Add comments about what each secret contains

## Example Workflow

```bash
# 1. Create and seal a new secret
kubectl create secret generic api-keys \
  --namespace my-app \
  --from-literal=GITHUB_TOKEN=ghp_xxxx \
  --from-literal=SLACK_WEBHOOK=https://hooks.slack.com/xxx \
  --dry-run=client -o yaml | \
kubeseal --format yaml \
  --controller-name=sealed-secrets \
  --controller-namespace=flux-system \
  > apps/production/my-app/api-keys-sealedsecret.yaml

# 2. Add to kustomization
echo "  - api-keys-sealedsecret.yaml" >> apps/production/my-app/kustomization.yaml

# 3. Commit and push
git add apps/production/my-app/
git commit -m "Add API keys secret for my-app"
git push

# 4. Verify deployment
flux reconcile kustomization apps --with-source
kubectl get secret api-keys -n my-app
```
