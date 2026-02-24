# Adding New Applications

This guide walks through the process of adding a new application to the cluster.

## Prerequisites

- `kubectl` access to the cluster
- `kubeseal` CLI installed (for secrets)
- Understanding of Helm charts or Kubernetes manifests

## Option 1: Helm-based Application

### Step 1: Add Helm Repository (if needed)

If the chart comes from a new repository, add it to `repositories/`:

```yaml
# repositories/my-repo.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-repo
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.example.com
```

Update `repositories/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - existing-repo.yaml
  - my-repo.yaml  # Add new repo
```

### Step 2: Create Application Directory

```bash
mkdir -p apps/my-app
```

**Namespace** (`apps/my-app/namespace.yaml`):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
```

**HelmRelease** (`apps/my-app/helmrelease.yaml`):

All values are defined inline -- no separate patch files.

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: flux-system
spec:
  releaseName: my-app
  targetNamespace: my-app
  interval: 10m
  chart:
    spec:
      chart: my-app
      version: "1.2.3"
      sourceRef:
        kind: HelmRepository
        name: my-repo
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  values:
    # All values inline
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
        cpu: 500m
```

**Ingress** (`apps/my-app/ingress.yaml`):

TLS is handled by a wildcard certificate (`*.kennyandries.com`), so no per-ingress cert-manager annotations are needed.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: my-app
spec:
  ingressClassName: traefik-system-traefik
  rules:
    - host: my-app.kennyandries.com
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

**Network Policy** (`apps/my-app/networkpolicy.yaml`) (optional):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: traefik-system
```

**Kustomization** (`apps/my-app/kustomization.yaml`):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrelease.yaml
  - ingress.yaml
  - networkpolicy.yaml
```

### Step 3: Add to Apps Kustomization

Update `apps/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./existing-app
  - ./my-app  # Add new app
```

### Step 4: Test and Deploy

```bash
# Validate locally
kustomize build apps/my-app

# Commit and push
git add apps/
git commit -m "Add my-app application"
git push
```

Flux will automatically detect the changes and deploy the application.

---

## Option 2: Raw Manifest Application

For applications without Helm charts:

### Step 1: Create Application Directory

```bash
mkdir -p apps/my-app
```

**Deployment** (`apps/my-app/deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: myregistry/my-app:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: 128Mi
              cpu: 50m
            limits:
              memory: 256Mi
              cpu: 200m
```

**Service** (`apps/my-app/service.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

Create namespace, ingress, network policy, and kustomization files following the same pattern as above.

---

## Adding Secrets

### Step 1: Create the Secret Locally

```bash
kubectl create secret generic my-app-secret \
  --namespace my-app \
  --from-literal=api-key=your-secret-key \
  --dry-run=client -o yaml > /tmp/secret.yaml
```

### Step 2: Seal the Secret

```bash
kubeseal --format yaml \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets \
  < /tmp/secret.yaml \
  > apps/my-app/sealedsecret.yaml
```

### Step 3: Reference in Kustomization

```yaml
# apps/my-app/kustomization.yaml
resources:
  - namespace.yaml
  - helmrelease.yaml
  - ingress.yaml
  - sealedsecret.yaml
```

### Step 4: Use in Deployment

```yaml
env:
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: my-app-secret
        key: api-key
```

---

## Adding Persistent Storage

### Using Longhorn (Recommended)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

Mount in deployment:

```yaml
spec:
  containers:
    - name: my-app
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-app-data
```

---

## Verification

### Check Flux Status

```bash
# Watch Kustomization reconciliation
flux get kustomizations -w

# Check HelmRelease status
flux get helmreleases -A

# View specific HelmRelease
kubectl describe helmrelease my-app -n flux-system
```

### Check Application

```bash
# View pods
kubectl get pods -n my-app

# View logs
kubectl logs -n my-app deployment/my-app

# Test ingress
curl -I https://my-app.kennyandries.com
```

---

## Checklist

- [ ] Helm repository added (if needed)
- [ ] Application directory created in `apps/<app>/`
- [ ] All manifests co-located (namespace, helmrelease/deployment, service, ingress, networkpolicy, sealedsecret)
- [ ] Added to `apps/kustomization.yaml`
- [ ] Secrets sealed and committed
- [ ] DNS record created (if applicable)
- [ ] Tested locally with `kustomize build`
- [ ] Committed and pushed
- [ ] Verified deployment via Flux
