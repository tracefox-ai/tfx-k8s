# Install argocd on local k8s cluster
* We will use managed ArgoCD on EKS in staging and production
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

## Deploying Applications

The Application manifests for Staging and Production are located in this directory:

### TraceFox Applications
- `tracefox-staging.yaml` - TraceFox observability platform for staging
- `tracefox-prod.yaml` - TraceFox observability platform for production

### ClickHouse Applications
- `clickhouse-staging.yaml` - ClickHouse cluster for staging
- `clickhouse-prod.yaml` - ClickHouse cluster for production

### MongoDB Applications
- `mongodb-staging.yaml` - MongoDB cluster for staging
- `mongodb-prod.yaml` - MongoDB cluster for production

### Operators
- `clickhouse-operator.yaml` - Altinity ClickHouse Operator (cluster-wide operator for managing ClickHouse resources)
- `mongodb-operator.yaml` - MongoDB Community Operator (cluster-wide operator for managing MongoDB resources)

**Note**: The operators are deployed via ArgoCD using Helm repositories. After deploying the ClickHouse operator, you can create ClickHouse resources using Altinity ClickHouse Operator CRDs. Similarly, after deploying the MongoDB operator, you can create MongoDB resources using MongoDB Community Operator CRDs.

### Prerequisites

1. **Update Repository URL**:
   Edit all application manifests. Replace the `repoURL` placeholder with the HTTPS URL of your git repository containing this code.
   ```yaml
   spec:
     source:
       repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git  # <--- Update this
   ```

2. **Helm Repositories for Operators**:
   ArgoCD can access public Helm repositories directly. If you encounter issues, you may need to add them to ArgoCD's repository list:
   ```bash
   # Add ClickHouse Operator repository
   argocd repo add https://docs.altinity.com/clickhouse-operator --type helm
   
   # Add MongoDB Operator repository
   argocd repo add https://mongodb.github.io/helm-charts --type helm
   ```

3. **Commit Changes**:
   It is recommended to commit and push these changes so ArgoCD can pull the matching revision.

### Apply Manifests

Run the following commands to create the applications in ArgoCD:

```bash
# Deploy Operators (cluster-wide, deploy once)
kubectl apply -f clickhouse-operator.yaml
kubectl apply -f mongodb-operator.yaml

# Deploy Staging Applications
kubectl apply -f tracefox-staging.yaml
kubectl apply -f clickhouse-staging.yaml
kubectl apply -f mongodb-staging.yaml

# Deploy Production Applications
kubectl apply -f tracefox-prod.yaml
kubectl apply -f clickhouse-prod.yaml
kubectl apply -f mongodb-prod.yaml
```

### Verify

Log in to the ArgoCD UI (forward port 8080 if running locally as shown above) to check the application status.

