# TraceFox ArgoCD GitOps

This directory contains ArgoCD Application manifests for deploying TraceFox and its dependencies using the **App of Apps** pattern.

## 📁 Directory Structure

```
argocd/
├── README.md                          # This file
├── bootstrap/                         # Bootstrap entry point
│   └── root.yaml                      # Root App of Apps (apply this first)
├── projects/                          # ArgoCD Projects for RBAC
│   ├── infrastructure.yaml            # Infrastructure project (operators)
│   ├── staging.yaml                   # Staging environment project
│   └── production.yaml                # Production environment project
└── apps/                              # All applications
    ├── infrastructure/                # Cluster-wide infrastructure
    │   ├── app-of-apps.yaml          # Infrastructure App of Apps
    │   ├── clickhouse-operator.yaml
    │   └── mongodb-operator.yaml
    ├── staging/                       # Staging environment
    │   ├── app-of-apps.yaml          # Staging App of Apps
    │   ├── tracefox.yaml
    │   ├── clickhouse.yaml
    │   └── mongodb.yaml
    └── production/                    # Production environment
        ├── app-of-apps.yaml          # Production App of Apps
        ├── tracefox.yaml
        ├── clickhouse.yaml
        └── mongodb.yaml
```

## 🚀 Quick Start

### Prerequisites

1. **ArgoCD installed** on your Kubernetes cluster
2. **Git repository access** - Update `repoURL` in all manifests to your repository URL
3. **kubectl** configured to access your cluster

### Initial Setup

#### 1. Install ArgoCD (if not already installed)

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI (optional)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

#### 2. Update Repository URLs

Before deploying, update the `repoURL` in all manifests to point to your Git repository:

```bash
# Replace YOUR_ORG and YOUR_REPO with your values
find argocd -name "*.yaml" -type f -exec sed -i '' 's|https://github.com/tracefox-ai/tfx-k8s.git|https://github.com/YOUR_ORG/YOUR_REPO.git|g' {} +
```

Or manually edit each file to update the `repoURL` field.

#### 3. Deploy ArgoCD Projects

ArgoCD Projects provide RBAC and multi-tenancy. Deploy them first:

```bash
kubectl apply -f argocd/projects/
```

#### 4. Bootstrap Everything

Apply the root App of Apps to deploy everything:

```bash
kubectl apply -f argocd/bootstrap/root.yaml
```

This single command will:
- Create the root application
- Deploy infrastructure App of Apps → ClickHouse & MongoDB operators
- Deploy staging App of Apps → Staging TraceFox, ClickHouse, MongoDB
- Deploy production App of Apps → Production TraceFox, ClickHouse, MongoDB

#### 5. Verify Deployment

```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Watch application sync status
kubectl get applications -n argocd -w

# Or use ArgoCD UI at http://localhost:8080 (if port-forwarded)
```

## 🏗️ Architecture

### App of Apps Pattern

The **App of Apps** pattern allows ArgoCD to manage applications declaratively:

1. **Root Application** (`bootstrap/root.yaml`) - Manages environment-level App of Apps
2. **Environment App of Apps** - Each environment has an App of Apps that manages its applications
   - Infrastructure App of Apps → Operators
   - Staging App of Apps → Staging applications
   - Production App of Apps → Production applications
3. **Leaf Applications** - Individual applications (TraceFox, ClickHouse, MongoDB)

**Benefits:**
- Single command deployment
- Declarative application management
- Automatic sync and self-healing
- Easy to add/remove applications

### ArgoCD Projects

Projects provide RBAC and multi-tenancy:

- **infrastructure** - Cluster-wide operators with cluster-scoped permissions
- **staging** - Staging environment, restricted to `staging` namespace
- **production** - Production environment, restricted to `production` namespace

**Benefits:**
- Namespace isolation
- Source repository restrictions
- Resource type restrictions
- RBAC enforcement

## 📦 Components

### Infrastructure (Operators)

Cluster-wide operators that manage custom resources:

- **ClickHouse Operator** - Altinity ClickHouse Operator for managing ClickHouse clusters
- **MongoDB Operator** - MongoDB Community Operator for managing MongoDB clusters

### Staging Environment

Applications deployed to the `staging` namespace:

- **TraceFox** - Observability platform (staging configuration)
- **ClickHouse** - Time-series database (staging configuration)
- **MongoDB** - Document database (staging configuration)

### Production Environment

Applications deployed to the `production` namespace:

- **TraceFox** - Observability platform (production configuration)
- **ClickHouse** - Time-series database (production configuration)
- **MongoDB** - Document database (production configuration)

## 🔧 Common Operations

### Deploy a Specific Environment

If you only want to deploy one environment:

```bash
# Deploy only infrastructure
kubectl apply -f argocd/projects/infrastructure.yaml
kubectl apply -f argocd/apps/infrastructure/app-of-apps.yaml

# Deploy only staging
kubectl apply -f argocd/projects/staging.yaml
kubectl apply -f argocd/apps/staging/app-of-apps.yaml

# Deploy only production
kubectl apply -f argocd/projects/production.yaml
kubectl apply -f argocd/apps/production/app-of-apps.yaml
```

### Add a New Application

1. Create the application manifest in the appropriate directory:
   - Infrastructure → `apps/infrastructure/`
   - Staging → `apps/staging/`
   - Production → `apps/production/`

2. Ensure the application uses the correct ArgoCD Project

3. Commit and push to Git

4. ArgoCD will automatically detect and sync the new application

### Remove an Application

1. Delete the application manifest from Git
2. Commit and push
3. ArgoCD will automatically prune the application (if `prune: true` is set)

### Sync Applications Manually

```bash
# Sync all applications
argocd app sync -l app.kubernetes.io/instance=root

# Sync specific environment
argocd app sync infrastructure
argocd app sync staging
argocd app sync production

# Sync specific application
argocd app sync tracefox-staging
```

### Disable Auto-Sync

To prevent automatic syncing (useful for production):

```bash
# Edit the application manifest and remove/comment out syncPolicy.automated
# Or use ArgoCD CLI:
argocd app set tracefox-prod --sync-policy none
```

## 🐛 Troubleshooting

### Application Not Syncing

```bash
# Check application status
kubectl get application <app-name> -n argocd -o yaml

# Check sync status
argocd app get <app-name>

# Force sync
argocd app sync <app-name> --force
```

### Repository Access Issues

```bash
# Verify repository is accessible
argocd repo list

# Add repository if needed
argocd repo add https://github.com/YOUR_ORG/YOUR_REPO.git
```

### Permission Issues

```bash
# Check ArgoCD Project permissions
kubectl get appproject -n argocd <project-name> -o yaml

# Verify application is using correct project
kubectl get application <app-name> -n argocd -o jsonpath='{.spec.project}'
```

### Operator Not Creating Resources

```bash
# Verify operator is running
kubectl get pods -n clickhouse
kubectl get pods -n mongodb

# Check operator logs
kubectl logs -n clickhouse -l app.kubernetes.io/name=clickhouse-operator
kubectl logs -n mongodb -l app.kubernetes.io/name=mongodb-kubernetes-operator
```

## 📚 Best Practices

1. **Always commit changes to Git** - ArgoCD syncs from Git, not local files
2. **Use ArgoCD Projects** - Enforce RBAC and namespace isolation
3. **Pin operator versions** - Use semantic versioning (e.g., `0.23.*`) instead of `*`
4. **Enable auto-sync with caution** - Consider manual sync for production
5. **Use App of Apps** - Manage applications declaratively
6. **Monitor sync status** - Regularly check ArgoCD UI or CLI
7. **Test in staging first** - Validate changes before deploying to production

## 🔗 References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [Altinity ClickHouse Operator](https://github.com/Altinity/clickhouse-operator)
- [MongoDB Community Operator](https://github.com/mongodb/mongodb-kubernetes-operator)

## 📝 Notes

- All applications use automated sync with `prune: true` and `selfHeal: true`
- Operators are deployed to dedicated namespaces (`clickhouse`, `mongodb`)
- Staging and production applications are isolated in their respective namespaces
- The root application manages all environment-level App of Apps
- Changes to Git repository automatically trigger syncs (when auto-sync is enabled)
