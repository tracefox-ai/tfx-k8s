# Install argocd on local k8s cluster
* We will use managed ArgoCD on EKS in staging and production
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

## Deploying TraceFox Applications

The Application manifests for Staging and Production are located in this directory:
- `tracefox-staging.yaml`
- `tracefox-prod.yaml`

### Prerequisites

1. **Update Repository URL**:
   Edit both `tracefox-staging.yaml` and `tracefox-prod.yaml`. Replace the `repoURL` placeholder with the HTTPS URL of your git repository containing this code.
   ```yaml
   spec:
     source:
       repoURL: https://github.com/YOUR_ORG/YOUR_REPO.git  # <--- Update this
   ```

2. **Commit Changes**:
   It is recommended to commit and push these changes so ArgoCD can pull the matching revision.

### Apply Manifests

Run the following commands to create the applications in ArgoCD:

```bash
# Deploy Staging
kubectl apply -f tracefox-staging.yaml

# Deploy Production
kubectl apply -f tracefox-prod.yaml
```

### Verify

Log in to the ArgoCD UI (forward port 8080 if running locally as shown above) to check the application status.

