# Docker Desktop Deployment (Local Development)

> **For local development using Docker Desktop Kubernetes**

## Important Configuration Note

**Current Setup:** This repository is configured for **multi-cluster deployment**:
- **Staging** → Separate cluster (currently Docker Desktop for development)
- **Production** → Separate cluster (not yet provisioned)
- **Both deploy to `default` namespace** (not separate namespaces in same cluster)

**Production is currently DISABLED** in the root App of Apps to avoid deployment on Docker Desktop.

## Prerequisites

1. **Docker Desktop installed** with Kubernetes enabled
   - Open Docker Desktop → Settings → Kubernetes → ☑ Enable Kubernetes
   - Wait for Kubernetes to start (green indicator in Docker Desktop)

2. **Verify connection:**
   ```bash
   kubectl config current-context
   # Should show: docker-desktop
   
   kubectl get nodes
   # Should show: docker-desktop   Ready    control-plane
   ```

3. **Recommended Resources:**
   - Docker Desktop → Settings → Resources
   - **CPUs:** 4 or more
   - **Memory:** 8GB or more
   - **Swap:** 2GB
   - **Disk:** 60GB or more

## Quick Deploy to Docker Desktop

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Wait for ArgoCD to be ready (takes 2-3 minutes)
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# 3. Verify ArgoCD is running
kubectl get pods -n argocd
# All pods should be Running

# 4. Deploy staging and infrastructure projects
kubectl apply -f argocd/projects/staging.yaml
kubectl apply -f argocd/projects/infrastructure.yaml

# 5. Deploy everything (infrastructure + staging only, production disabled)
kubectl apply -f argocd/bootstrap/root.yaml

# 6. Watch deployment progress
kubectl get applications -n argocd -w
# Press Ctrl+C to stop watching
```

## Access ArgoCD UI

```bash
# Port forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# In another terminal, get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Open browser to https://localhost:8080
# Username: admin
# Password: (from command above)
```

## What Gets Deployed on Docker Desktop

By default, only **staging** environment is deployed:

**✅ Infrastructure** (cluster-wide operators):
- ClickHouse Operator → `clickhouse` namespace
- MongoDB Operator → `mongodb` namespace

**✅ Staging** (all in `default` namespace):
- TraceFox application
- ClickHouse database cluster
- MongoDB replica set

**❌ Production** (disabled):
- Not deployed to Docker Desktop
- Will be deployed to separate production cluster when ready

## Enabling/Disabling Production Deployment

### Current Configuration: Production is DISABLED

File: `argocd/bootstrap/root.yaml`
```yaml
# Current configuration (production disabled)
spec:
  source:
    path: argocd/apps
    directory:
      recurse: true
      include: '{infrastructure,staging}/app-of-apps.yaml'  # Exclude production
```

### To Enable Production

**When you have a production cluster ready:**

1. Edit `argocd/bootstrap/root.yaml`:
   ```yaml
   # Change to this (production enabled)
   spec:
     source:
       path: argocd/apps
       directory:
         recurse: true
         include: '*/app-of-apps.yaml'  # Include all environments
   ```

2. Commit and push to Git:
   ```bash
   git add argocd/bootstrap/root.yaml
   git commit -m "Enable production deployment"
   git push
   ```

3. ArgoCD will automatically sync and deploy production

### Why Disable Production on Docker Desktop?

- ✅ Saves local machine resources
- ✅ Prevents accidental production deployments
- ✅ Docker Desktop is for development/testing only
- ✅ Production requires real cloud cluster (EKS, GKE, AKS)

## Verify Deployment on Docker Desktop

```bash
# Check all ArgoCD applications
kubectl get applications -n argocd

# Expected output:
# NAME             SYNC STATUS   HEALTH STATUS
# root             Synced        Healthy
# infrastructure   Synced        Healthy
# staging          Synced        Healthy
# (production should NOT appear)

# Check pods in default namespace (staging apps)
kubectl get pods -n default

# Expected pods:
# - tracefox-app-xxx
# - tracefox-otlp-gateway-xxx
# - tracefox-collector-xxx
# - tracefox-nginx-xxx
# - chi-clickhouse-xxx (ClickHouse pods)
# - mongodb-xxx (MongoDB pods)

# Check operators
kubectl get pods -n clickhouse  # ClickHouse operator
kubectl get pods -n mongodb     # MongoDB operator

# Check ClickHouse cluster
kubectl get chi -n default

# Check MongoDB replica set
kubectl get mongodbcommunity -n default
```

## Access TraceFox UI on Docker Desktop

```bash
# Port forward to TraceFox nginx service
kubectl port-forward -n default svc/tracefox-nginx 8080:80

# Open browser to http://localhost:8080
```

## Troubleshooting Docker Desktop

### Pods Stuck in Pending

**Cause:** Not enough resources allocated to Docker Desktop

**Solution:**
```bash
# Check pod status
kubectl describe pod <pod-name> -n default

# If you see "Insufficient cpu" or "Insufficient memory":
# 1. Open Docker Desktop → Settings → Resources
# 2. Increase CPUs to 4+ and Memory to 8GB+
# 3. Click "Apply & Restart"
# 4. Wait for Kubernetes to restart
```

### ImagePullBackOff

**Cause:** Docker Desktop pulling images for first time

**Solution:**
```bash
# Check image pull status
kubectl describe pod <pod-name> -n default | grep -A 10 Events

# Usually resolves itself after a few minutes
# Docker Desktop will retry automatically
```

### ArgoCD Application OutOfSync

**Cause:** Manual changes or sync disabled

**Solution:**
```bash
# Check application status
kubectl get application <app-name> -n argocd -o yaml

# Force sync via kubectl
kubectl patch application <app-name> -n argocd \
  --type merge \
  -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Or sync via ArgoCD UI
# https://localhost:8080 → Click application → Sync
```

### ClickHouse or MongoDB Not Starting

**Cause:** Operators not ready or CRDs not installed

**Solution:**
```bash
# Check if operators are running
kubectl get pods -n clickhouse
kubectl get pods -n mongodb

# Check operator logs
kubectl logs -n clickhouse -l app.kubernetes.io/name=clickhouse-operator
kubectl logs -n mongodb -l app.kubernetes.io/name=mongodb-kubernetes-operator

# If operators are missing, check ArgoCD application
kubectl get application infrastructure -n argocd
```

## Clean Up Docker Desktop

To remove everything:

```bash
# Delete root application (cascades to all apps)
kubectl delete application root -n argocd

# Wait for all apps to be deleted
kubectl get applications -n argocd -w

# Delete ArgoCD
kubectl delete namespace argocd

# Delete all namespaces
kubectl delete namespace default clickhouse mongodb --force --grace-period=0
```
