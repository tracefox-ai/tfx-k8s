# TraceFox Kubernetes Infrastructure (tfx-k8s)

> **Complete guide to deploying and managing TraceFox on Kubernetes using GitOps principles**

This repository contains all Kubernetes infrastructure for TraceFox - an observability platform for logs, metrics, and traces. Everything is managed declaratively using **Helm charts** and **ArgoCD** for GitOps-based deployments.

---

## 📚 Table of Contents

- [What is This Repository?](#what-is-this-repository)
- [Architecture Overview](#architecture-overview)
- [Directory Structure](#directory-structure)
- [Prerequisites](#prerequisites)
- [Quick Start Guide](#quick-start-guide)
- [Detailed Deployment Guide](#detailed-deployment-guide)
- [Secret Management](#secret-management)
- [Environment Configuration](#environment-configuration)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [FAQ](#faq)

---

## What is This Repository?

**tfx-k8s** is the infrastructure-as-code repository for TraceFox. It contains:

1. **Helm Charts** (`helm/`) - Kubernetes application definitions
2. **ArgoCD Applications** (`argocd/`) - GitOps deployment configurations

**Why Helm?** Helm is a package manager for Kubernetes that allows us to:
- Template Kubernetes manifests with variables
- Manage different configurations per environment (staging, production)
- Version and rollback deployments
- Share reusable components

**Why ArgoCD?** ArgoCD is a GitOps tool that:
- Automatically deploys changes from Git to Kubernetes
- Keeps your cluster in sync with this repository
- Provides a UI to visualize deployments
- Enables declarative, auditable infrastructure

---

## Architecture Overview

TraceFox consists of several components that work together:

```
┌─────────────────────────────────────────────────────────────┐
│                        TraceFox Stack                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  TraceFox    │  │  ClickHouse  │  │   MongoDB    │       │
│  │  Application │  │   Database   │  │   Database   │       │
│  │              │  │              │  │              │       │
│  │ • Web UI     │  │ • Time-series│  │ • Metadata   │       │
│  │ • API Server │  │ • Analytics  │  │ • Config     │       │
│  │ • OTLP       │  │ • Logs/Traces│  │ • Users      │       │
│  │   Gateway    │  │ • Metrics    │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                  │                  │             │
│         └──────────────────┴──────────────────┘             │
│                            │                                │
└────────────────────────────┼────────────────────────────────┘
                             │
                    ┌────────┴────────┐
                    │   Kubernetes    │
                    │     Cluster     │
                    └─────────────────┘
```

### Component Dependencies

**Critical Order:** Components must be deployed in this order because of dependencies:

1. **Operators** (Infrastructure)
   - ClickHouse Operator - Manages ClickHouse clusters
   - MongoDB Operator - Manages MongoDB replica sets
   
2. **Databases** (Data Layer)
   - ClickHouse - Stores telemetry data (logs, traces, metrics)
   - MongoDB - Stores application metadata and configuration
   
3. **Application** (Application Layer)
   - TraceFox - Main observability platform

**Why this order?** 
- Operators must exist before creating database resources (they watch for Custom Resource Definitions)
- Databases must be running before TraceFox starts (TraceFox connects to them on startup)

---

## Directory Structure

```
tfx-k8s/
├── README.md                   # This file - comprehensive guide
├── argocd/                     # ArgoCD GitOps configurations
│   ├── README.md               # ArgoCD setup and usage guide
│   ├── bootstrap/              # App of Apps entry point
│   │   └── root.yaml           # Root application (deploy this first)
│   ├── projects/               # ArgoCD Projects (RBAC & isolation)
│   │   ├── infrastructure.yaml # Operators project
│   │   ├── staging.yaml        # Staging environment project
│   │   └── production.yaml     # Production environment project
│   └── apps/                   # Application manifests
│       ├── infrastructure/     # Cluster-wide operators
│       │   ├── app-of-apps.yaml
│       │   ├── clickhouse-operator.yaml
│       │   └── mongodb-operator.yaml
│       ├── staging/            # Staging environment apps
│       │   ├── app-of-apps.yaml
│       │   ├── tracefox.yaml
│       │   ├── clickhouse.yaml
│       │   └── mongodb.yaml
│       └── production/         # Production environment apps
│           ├── app-of-apps.yaml
│           ├── tracefox.yaml
│           ├── clickhouse.yaml
│           └── mongodb.yaml
└── helm/                       # Helm charts
    ├── tracefox/               # TraceFox application chart
    │   ├── README.md           # Detailed chart documentation
    │   ├── Chart.yaml          # Chart metadata
    │   ├── values.yaml         # Default configuration
    │   ├── values-staging.yaml # Staging overrides
    │   ├── values-prod.yaml    # Production overrides
    │   └── templates/          # Kubernetes manifest templates
    ├── clickhouse/             # ClickHouse database chart
    │   ├── README.md           # Detailed chart documentation
    │   ├── Chart.yaml          # Chart metadata
    │   ├── values.yaml         # Default configuration
    │   ├── values-staging.yaml # Staging overrides
    │   ├── values-prod.yaml    # Production overrides
    │   └── templates/          # Kubernetes manifest templates
    └── mongodb/                # MongoDB database chart
        ├── README.md           # Detailed chart documentation
        ├── Chart.yaml          # Chart metadata
        ├── values.yaml         # Default configuration
        ├── values-staging.yaml # Staging overrides
        └── templates/          # Kubernetes manifest templates
```

### Key Files Explained

- **`Chart.yaml`** - Metadata about the Helm chart (name, version, description)
- **`values.yaml`** - Default configuration values (base template)
- **`values-staging.yaml`** - Staging-specific overrides (smaller resources, ClusterIP services)
- **`values-prod.yaml`** - Production-specific overrides (larger resources, LoadBalancer services)
- **`templates/`** - Kubernetes YAML templates with Go templating syntax
- **`NOTES.txt`** - Post-install instructions shown after `helm install`
- **`.helmignore`** - Files to exclude from Helm packages (like `.gitignore`)

---

## Prerequisites

### Required Tools

| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| **kubectl** | 1.24+ | Kubernetes CLI | [Install Guide](https://kubernetes.io/docs/tasks/tools/) |
| **helm** | 3.0+ | Kubernetes package manager | [Install Guide](https://helm.sh/docs/intro/install/) |
| **ArgoCD CLI** | Latest | ArgoCD management (optional) | [Install Guide](https://argo-cd.readthedocs.io/en/stable/cli_installation/) |

### Kubernetes Cluster

You need a running Kubernetes cluster. Options:

**Local Development:**
- **Docker Desktop** - Easiest for Mac/Windows ([Download](https://www.docker.com/products/docker-desktop))
- **Minikube** - Lightweight local cluster ([Install](https://minikube.sigs.k8s.io/docs/start/))
- **Kind** - Kubernetes in Docker ([Install](https://kind.sigs.k8s.io/docs/user/quick-start/))

**Cloud Production:**
- **AWS EKS** - Amazon Elastic Kubernetes Service
- **GCP GKE** - Google Kubernetes Engine
- **Azure AKS** - Azure Kubernetes Service

### Verify Prerequisites

```bash
# Check kubectl is installed and connected
kubectl version --client
kubectl cluster-info

# Check Helm is installed
helm version

# Check you can access the cluster
kubectl get nodes
```

**Expected output:**
```
NAME                 STATUS   ROLES           AGE   VERSION
docker-desktop       Ready    control-plane   10d   v1.28.2
```

---

## Quick Start Guide

> **For experienced users:** Deploy everything with ArgoCD in 3 commands

### 1. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 2. Access ArgoCD UI (Optional)

```bash
# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Open browser to https://localhost:8080
# Username: admin
# Password: (from command above)
```

### 3. Deploy Everything

```bash
# Update repository URL in all manifests (replace with your repo)
cd argocd
find . -name "*.yaml" -type f -exec sed -i '' 's|https://github.com/tracefox-ai/tfx-k8s.git|https://github.com/YOUR_ORG/YOUR_REPO.git|g' {} +

# Deploy ArgoCD Projects (RBAC)
kubectl apply -f projects/

# Deploy root App of Apps (this deploys everything)
kubectl apply -f bootstrap/root.yaml

# Watch deployment progress
kubectl get applications -n argocd -w
```

**What just happened?**
1. ArgoCD Projects created (infrastructure, staging, production)
2. Root application deployed
3. Root app created 3 App of Apps (infrastructure, staging, production)
4. Each App of Apps deployed its applications
5. Everything syncs automatically from Git

> **📘 For Docker Desktop local development:** See [DOCKER_DESKTOP.md](DOCKER_DESKTOP.md) for detailed instructions on deploying to Docker Desktop with production disabled.

---

## Detailed Deployment Guide

> **For beginners:** Step-by-step manual deployment with explanations

### Understanding the Deployment Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Deployment Flow                           │
└─────────────────────────────────────────────────────────────┘

Step 1: Install ArgoCD
   │
   ├─→ ArgoCD runs in your cluster
   │   Watches Git repository for changes
   │   Automatically deploys when you push to Git
   │
Step 2: Deploy ArgoCD Projects
   │
   ├─→ Creates RBAC boundaries
   │   Isolates infrastructure/staging/production
   │   Restricts what can be deployed where
   │
Step 3: Deploy Operators (Infrastructure)
   │
   ├─→ ClickHouse Operator installed
   │   MongoDB Operator installed
   │   These watch for database Custom Resources
   │
Step 4: Deploy Databases
   │
   ├─→ ClickHouse cluster created
   │   MongoDB replica set created
   │   Operators manage the actual pods
   │
Step 5: Deploy TraceFox Application
   │
   └─→ TraceFox connects to databases
       Application ready to use
```

### Step 1: Install ArgoCD

**What is ArgoCD?** ArgoCD is a GitOps continuous delivery tool. It watches this Git repository and automatically deploys changes to your Kubernetes cluster.

```bash
# Create a dedicated namespace for ArgoCD
kubectl create namespace argocd

# Install ArgoCD using official manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Wait for ArgoCD to be ready:**
```bash
# This command waits up to 5 minutes for ArgoCD to start
kubectl wait --for=condition=available --timeout=300s \
  deployment/argocd-server -n argocd

# Verify all pods are running
kubectl get pods -n argocd
```

**Expected output:**
```
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-application-controller-0       1/1     Running   0          2m
argocd-dex-server-xxx                 1/1     Running   0          2m
argocd-redis-xxx                      1/1     Running   0          2m
argocd-repo-server-xxx                1/1     Running   0          2m
argocd-server-xxx                     1/1     Running   0          2m
```

**Access ArgoCD UI (Optional but Recommended):**
```bash
# Forward port 8080 to ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# In another terminal, get the admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Open your browser to `https://localhost:8080`:
- **Username:** `admin`
- **Password:** (from command above)

**⚠️ Security Note:** Change the default password after first login!

### Step 2: Update Repository URLs

**Why?** The ArgoCD manifests reference `https://github.com/tracefox-ai/tfx-k8s.git`. You need to update this to your actual Git repository URL.

```bash
cd argocd

# Option 1: Using sed (Mac)
find . -name "*.yaml" -type f -exec sed -i '' \
  's|https://github.com/tracefox-ai/tfx-k8s.git|https://github.com/YOUR_ORG/YOUR_REPO.git|g' {} +

# Option 2: Using sed (Linux)
find . -name "*.yaml" -type f -exec sed -i \
  's|https://github.com/tracefox-ai/tfx-k8s.git|https://github.com/YOUR_ORG/YOUR_REPO.git|g' {} +

# Option 3: Manually edit each file
# Search for "repoURL:" and replace with your repository URL
```

**Commit and push changes:**
```bash
git add .
git commit -m "Update repository URLs for ArgoCD"
git push
```

### Step 3: Deploy ArgoCD Projects

**What are ArgoCD Projects?** Projects provide:
- **RBAC** - Role-based access control
- **Isolation** - Separate infrastructure/staging/production
- **Restrictions** - Control what can be deployed where

```bash
# Deploy all three projects
kubectl apply -f projects/
```

**Verify projects were created:**
```bash
kubectl get appproject -n argocd
```

**Expected output:**
```
NAME             AGE
default          10m
infrastructure   1m
staging          1m
production       1m
```

**What each project does:**

| Project | Purpose | Allowed Namespaces | Allowed Resources |
|---------|---------|-------------------|-------------------|
| **infrastructure** | Cluster-wide operators | `clickhouse`, `mongodb`, `argocd` | All (including cluster-scoped) |
| **staging** | Staging environment | `staging` only | Namespace-scoped only |
| **production** | Production environment | `production` only | Namespace-scoped only |

### Step 4: Deploy Using App of Apps Pattern

**What is App of Apps?** Instead of deploying each application manually, we deploy one "root" application that manages all other applications. This is called the "App of Apps" pattern.

```mermaid
graph TD
    A[Root Application] --> B[Infrastructure App of Apps]
    A --> C[Staging App of Apps]
    A --> D[Production App of Apps]
    B --> E[ClickHouse Operator]
    B --> F[MongoDB Operator]
    C --> G[TraceFox Staging]
    C --> H[ClickHouse Staging]
    C --> I[MongoDB Staging]
    D --> J[TraceFox Production]
    D --> K[ClickHouse Production]
    D --> L[MongoDB Production]
```

**Deploy the root application:**
```bash
kubectl apply -f bootstrap/root.yaml
```

**What happens next?** (Automatically)
1. Root app creates 3 App of Apps (infrastructure, staging, production)
2. Infrastructure App of Apps deploys operators
3. Staging App of Apps deploys staging databases and TraceFox
4. Production App of Apps deploys production databases and TraceFox

**Watch the deployment:**
```bash
# Watch all applications
kubectl get applications -n argocd -w

# Or use ArgoCD UI at https://localhost:8080
```

**Check sync status:**
```bash
# List all applications
kubectl get applications -n argocd

# Check specific application
kubectl get application tracefox-staging -n argocd -o yaml
```

### Step 5: Verify Deployment

**Check operators are running:**
```bash
# ClickHouse Operator
kubectl get pods -n clickhouse | grep operator

# MongoDB Operator
kubectl get pods -n mongodb | grep operator
```

**Check databases are running:**
```bash
# ClickHouse
kubectl get chi -n staging  # ClickHouseInstallation custom resource
kubectl get pods -n staging | grep clickhouse

# MongoDB
kubectl get mongodbcommunity -n staging
kubectl get pods -n staging | grep mongodb
```

**Check TraceFox is running:**
```bash
kubectl get pods -n staging | grep tracefox
kubectl get svc -n staging | grep tracefox
```

**Access TraceFox UI:**
```bash
# Port forward to TraceFox
kubectl port-forward -n staging svc/tracefox-nginx 8080:80

# Open browser to http://localhost:8080
```

---

## Secret Management

> **🔴 CRITICAL SECURITY SECTION - READ CAREFULLY**

### The Problem

Kubernetes applications need secrets (passwords, API keys, tokens). **NEVER commit secrets to Git!**

**Why?**
- Git history is permanent (even if you delete the file later)
- Anyone with repository access can see secrets
- Leaked secrets can compromise your entire system

### Our Approach

All secrets have been **removed** from values files. You have 4 options to provide secrets:

### Option 1: External Secrets Operator (Recommended for Production)

**What is it?** External Secrets Operator fetches secrets from external secret stores (AWS Secrets Manager, Vault, GCP Secret Manager, etc.) and creates Kubernetes Secrets automatically.

**Advantages:**
- ✅ Secrets never touch Git
- ✅ Centralized secret management
- ✅ Automatic rotation
- ✅ Audit logging
- ✅ Production-grade security

**Setup:**

1. **Install External Secrets Operator:**
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets-system --create-namespace
```

2. **Create a SecretStore** (example for AWS Secrets Manager):
```yaml
# secretstore.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: staging
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

```bash
kubectl apply -f secretstore.yaml
```

3. **Store secrets in AWS Secrets Manager:**
```bash
# Create secrets in AWS
aws secretsmanager create-secret \
  --name tracefox/staging/hyperdx-api-key \
  --secret-string "your-api-key-here"

aws secretsmanager create-secret \
  --name tracefox/staging/clickhouse-password \
  --secret-string "your-password-here"
```

4. **Enable external secrets in Helm:**
```bash
helm upgrade --install tracefox ./helm/tracefox \
  -f ./helm/tracefox/values-staging.yaml \
  --set externalSecrets.enabled=true \
  --set externalSecrets.secretStore=aws-secrets-manager \
  -n staging
```

**How it works:**
1. External Secrets Operator watches for ExternalSecret resources
2. It fetches secrets from AWS Secrets Manager
3. It creates Kubernetes Secrets automatically
4. TraceFox uses these Kubernetes Secrets

### Option 2: Helm --set Flags (Good for Development)

**What is it?** Pass secrets as command-line arguments when installing Helm charts.

**Advantages:**
- ✅ Simple and quick
- ✅ No additional tools needed
- ✅ Good for local development

**Disadvantages:**
- ⚠️ Secrets visible in shell history
- ⚠️ Secrets visible in Helm release metadata
- ⚠️ Not suitable for production

**Usage:**
```bash
helm upgrade --install tracefox ./helm/tracefox \
  -f ./helm/tracefox/values-staging.yaml \
  --set secrets.hyperdxApiKey=my-secret-key \
  --set secrets.clickhouseAdminPassword=my-password \
  --set secrets.mongoUri=mongodb://user:pass@host:27017/db \
  -n staging
```

### Option 3: Separate Secrets File (Good for Team Development)

**What is it?** Create a separate `secrets.yaml` file that is **NOT committed to Git**.

**Advantages:**
- ✅ Clean separation of secrets and config
- ✅ Easy to manage multiple environments
- ✅ Can be encrypted with tools like SOPS

**Disadvantages:**
- ⚠️ Must distribute secrets file securely to team
- ⚠️ Risk of accidentally committing to Git

**Setup:**

1. **Create secrets file:**
```yaml
# secrets-staging.yaml (DO NOT COMMIT!)
secrets:
  hyperdxApiKey: "actual-api-key-here"
  clickhouseAdminPassword: "actual-password-here"
  mongoUri: "mongodb://user:pass@host:27017/db"
```

2. **Add to .gitignore:**
```bash
echo "secrets-*.yaml" >> .gitignore
```

3. **Use with Helm:**
```bash
helm upgrade --install tracefox ./helm/tracefox \
  -f ./helm/tracefox/values-staging.yaml \
  -f ./secrets-staging.yaml \
  -n staging
```

### Option 4: Sealed Secrets (Good for GitOps)

**What is it?** Encrypt secrets so they can be safely committed to Git. Only the cluster can decrypt them.

**Advantages:**
- ✅ Secrets can be in Git (encrypted)
- ✅ Full GitOps workflow
- ✅ Audit trail in Git

**Disadvantages:**
- ⚠️ Requires Sealed Secrets controller
- ⚠️ More complex setup

**Setup:**

1. **Install Sealed Secrets controller:**
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml
```

2. **Install kubeseal CLI:**
```bash
# Mac
brew install kubeseal

# Linux
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64 -O kubeseal
chmod +x kubeseal
sudo mv kubeseal /usr/local/bin/
```

3. **Create and seal a secret:**
```bash
# Create regular secret
kubectl create secret generic tracefox-secrets \
  --from-literal=hyperdxApiKey=my-key \
  --from-literal=clickhousePassword=my-pass \
  --dry-run=client -o yaml > secret.yaml

# Seal it (encrypt)
kubeseal -f secret.yaml -w sealed-secret.yaml

# Commit sealed secret to Git
git add sealed-secret.yaml
git commit -m "Add sealed secrets"
```

4. **Apply sealed secret:**
```bash
kubectl apply -f sealed-secret.yaml -n staging
```

### Recommendation by Environment

| Environment | Recommended Approach | Why |
|-------------|---------------------|-----|
| **Local Development** | `--set` flags | Quick and simple |
| **Team Development** | Separate secrets file | Easy to share securely |
| **Staging** | External Secrets Operator | Practice production setup |
| **Production** | External Secrets Operator | Enterprise-grade security |

---

## Environment Configuration

### Understanding Values Files

Helm uses a hierarchy of values files:

```
values.yaml (base)
    ↓
values-staging.yaml (overrides for staging)
    ↓
values-prod.yaml (overrides for production)
    ↓
--set flags (highest priority)
```

**Example:**

```yaml
# values.yaml (base)
app:
  replicaCount: 1
  resources:
    memory: "512Mi"

# values-staging.yaml (staging overrides)
app:
  replicaCount: 2  # Override: 2 replicas for staging
  # resources not specified, uses base value

# values-prod.yaml (production overrides)
app:
  replicaCount: 5  # Override: 5 replicas for production
  resources:
    memory: "2Gi"  # Override: more memory for production
```

### Staging vs Production Differences

| Configuration | Staging | Production | Rationale |
|---------------|---------|------------|-----------|
| **Replicas** | 2 | 5 | Production needs high availability |
| **Resources** | Smaller (512Mi RAM) | Larger (2Gi RAM) | Production handles more load |
| **Service Type** | ClusterIP | LoadBalancer | Production needs external access |
| **Log Level** | `info` | `error` | More debugging in staging |
| **ClickHouse Shards** | 2 | 4 | Production needs more data capacity |
| **Storage Size** | 5Gi | 100Gi | Production stores more data |

### Customizing Configuration

**To change a value:**

1. **Edit the appropriate values file:**
```bash
# For staging
vim helm/tracefox/values-staging.yaml

# For production
vim helm/tracefox/values-prod.yaml
```

2. **Commit and push:**
```bash
git add helm/tracefox/values-staging.yaml
git commit -m "Increase staging replicas to 3"
git push
```

3. **ArgoCD auto-syncs** (if enabled) or manually sync:
```bash
argocd app sync tracefox-staging
```

**To override without changing files:**
```bash
helm upgrade tracefox ./helm/tracefox \
  -f ./helm/tracefox/values-staging.yaml \
  --set app.replicaCount=3 \
  -n staging
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Pods Not Starting

**Symptom:**
```bash
kubectl get pods -n staging
NAME                        READY   STATUS             RESTARTS   AGE
tracefox-app-xxx            0/1     CrashLoopBackOff   5          3m
```

**Diagnosis:**
```bash
# Check pod logs
kubectl logs -n staging tracefox-app-xxx

# Check pod events
kubectl describe pod -n staging tracefox-app-xxx

# Check previous logs if pod restarted
kubectl logs -n staging tracefox-app-xxx --previous
```

**Common Causes:**

| Error Message | Cause | Solution |
|---------------|-------|----------|
| `Connection refused` | Database not ready | Wait for database pods to be Running |
| `Secret not found` | Secrets not configured | See [Secret Management](#secret-management) |
| `ImagePullBackOff` | Cannot pull image | Check image name and registry access |
| `OOMKilled` | Out of memory | Increase memory limits in values file |

#### 2. ArgoCD Application OutOfSync

**Symptom:**
```bash
kubectl get application -n argocd
NAME               SYNC STATUS   HEALTH STATUS
tracefox-staging   OutOfSync     Healthy
```

**Diagnosis:**
```bash
# Check what's different
argocd app diff tracefox-staging

# View application details
kubectl get application tracefox-staging -n argocd -o yaml
```

**Solutions:**

**Manual sync:**
```bash
argocd app sync tracefox-staging
```

**Enable auto-sync:**
```yaml
# In argocd/apps/staging/tracefox.yaml
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### 3. Cannot Connect to Database

**Symptom:**
```
Error: dial tcp 10.96.0.1:8123: connect: connection refused
```

**Diagnosis:**
```bash
# Check if ClickHouse is running
kubectl get chi -n staging
kubectl get pods -n staging | grep clickhouse

# Check if service exists
kubectl get svc -n staging | grep clickhouse

# Test connection from another pod
kubectl run -it --rm debug --image=busybox --restart=Never -n staging -- sh
# Inside pod:
wget -O- http://clickhouse-tracefox:8123/?query=SELECT%201
```

**Solutions:**

1. **Wait for database to be ready:**
```bash
kubectl wait --for=condition=ready pod -l app=clickhouse -n staging --timeout=300s
```

2. **Check operator is running:**
```bash
kubectl get pods -n clickhouse | grep operator
```

3. **Check operator logs:**
```bash
kubectl logs -n clickhouse -l app.kubernetes.io/name=clickhouse-operator
```

#### 4. Helm Install Fails

**Symptom:**
```
Error: INSTALLATION FAILED: unable to build kubernetes objects from release manifest
```

**Diagnosis:**
```bash
# Dry-run to see what would be created
helm install tracefox ./helm/tracefox \
  -f ./helm/tracefox/values-staging.yaml \
  --dry-run --debug

# Lint the chart
helm lint ./helm/tracefox
```

**Common Causes:**

| Error | Cause | Solution |
|-------|-------|----------|
| `invalid type for...` | Wrong value type in values file | Check values file syntax |
| `template: ... undefined` | Missing required value | Add missing value to values file |
| `namespace not found` | Namespace doesn't exist | Add `--create-namespace` flag |

#### 5. External Secrets Not Working

**Symptom:**
```bash
kubectl get externalsecret -n staging
NAME                 STORE               STATUS
tracefox-secrets     aws-secrets-manager SecretSyncedError
```

**Diagnosis:**
```bash
# Check ExternalSecret status
kubectl describe externalsecret tracefox-secrets -n staging

# Check SecretStore
kubectl describe secretstore aws-secrets-manager -n staging

# Check External Secrets Operator logs
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets
```

**Solutions:**

1. **Verify IAM permissions** (for AWS):
```bash
# Service account must have IAM role with SecretsManager permissions
kubectl get sa external-secrets-sa -n staging -o yaml
```

2. **Check secret exists in backend:**
```bash
aws secretsmanager get-secret-value --secret-id tracefox/staging/hyperdx-api-key
```

3. **Verify secret paths match:**
```yaml
# In values file
externalSecrets:
  hyperdxApiKeyPath: "tracefox/staging/hyperdx-api-key"  # Must match AWS secret name
```

### Getting Help

**Check logs:**
```bash
# Application logs
kubectl logs -n staging -l app.kubernetes.io/name=tracefox --tail=100

# All pods in namespace
kubectl logs -n staging --all-containers=true --tail=100

# Follow logs in real-time
kubectl logs -n staging -l app.kubernetes.io/name=tracefox -f
```

**Describe resources:**
```bash
# Pod details and events
kubectl describe pod <pod-name> -n staging

# Service details
kubectl describe svc <service-name> -n staging

# Application details
kubectl describe application tracefox-staging -n argocd
```

**Interactive debugging:**
```bash
# Shell into a pod
kubectl exec -it <pod-name> -n staging -- /bin/sh

# Run debug pod
kubectl run -it --rm debug --image=busybox --restart=Never -n staging -- sh
```

---

## Best Practices

### 1. Git Workflow

**Always use Git for changes:**
```bash
# Make changes
vim helm/tracefox/values-staging.yaml

# Commit with descriptive message
git add helm/tracefox/values-staging.yaml
git commit -m "Increase staging replicas from 2 to 3 for load testing"

# Push to trigger ArgoCD sync
git push
```

**Use branches for major changes:**
```bash
# Create feature branch
git checkout -b feature/increase-clickhouse-storage

# Make changes
vim helm/clickhouse/values-prod.yaml

# Commit and push
git add helm/clickhouse/values-prod.yaml
git commit -m "Increase ClickHouse storage from 100Gi to 500Gi"
git push origin feature/increase-clickhouse-storage

# Create pull request for review
# After approval, merge to main
```

### 2. Testing Changes

**Always test in staging first:**
```bash
# 1. Deploy to staging
helm upgrade tracefox ./helm/tracefox \
  -f ./helm/tracefox/values-staging.yaml \
  -n staging

# 2. Run tests
helm test tracefox -n staging

# 3. Verify manually
kubectl port-forward -n staging svc/tracefox-nginx 8080:80
# Open http://localhost:8080 and test

# 4. If successful, deploy to production
helm upgrade tracefox ./helm/tracefox \
  -f ./helm/tracefox/values-prod.yaml \
  -n production
```

### 3. Monitoring Deployments

**Watch ArgoCD applications:**
```bash
# Watch all applications
watch kubectl get applications -n argocd

# Watch specific application
argocd app get tracefox-staging --watch
```

**Check pod status:**
```bash
# Watch pods
watch kubectl get pods -n staging

# Check pod readiness
kubectl get pods -n staging -o wide
```

### 4. Rollback Strategy

**Helm rollback:**
```bash
# List releases
helm list -n staging

# See release history
helm history tracefox -n staging

# Rollback to previous version
helm rollback tracefox -n staging

# Rollback to specific revision
helm rollback tracefox 3 -n staging
```

**ArgoCD rollback:**
```bash
# View history
argocd app history tracefox-staging

# Rollback to previous version
argocd app rollback tracefox-staging

# Rollback to specific revision
argocd app rollback tracefox-staging 5
```

### 5. Resource Management

**Set resource limits:**
```yaml
# In values file
app:
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "2000m"
```

**Why?**
- **Requests** - Guaranteed resources (used for scheduling)
- **Limits** - Maximum resources (prevents resource hogging)

**Monitor resource usage:**
```bash
# Pod resource usage
kubectl top pods -n staging

# Node resource usage
kubectl top nodes
```

### 6. Security Best Practices

✅ **DO:**
- Use External Secrets Operator for production
- Enable RBAC with ArgoCD Projects
- Use least-privilege service accounts
- Scan images for vulnerabilities
- Enable network policies
- Use TLS for all external traffic
- Rotate secrets regularly

❌ **DON'T:**
- Commit secrets to Git
- Use default passwords
- Run containers as root
- Expose services publicly without authentication
- Use `latest` image tags in production
- Disable security features for convenience

---

## FAQ

### General Questions

**Q: What is the difference between Helm and ArgoCD?**

**A:** 
- **Helm** is a package manager - it templates and installs Kubernetes manifests
- **ArgoCD** is a GitOps tool - it watches Git and automatically deploys changes
- They work together: ArgoCD uses Helm to deploy charts from Git

**Q: Do I need ArgoCD, or can I just use Helm?**

**A:** You can use just Helm, but ArgoCD provides:
- Automatic deployments when you push to Git
- Visual UI to see deployment status
- Automatic drift detection and correction
- Audit trail of all changes

**Q: What is GitOps?**

**A:** GitOps means:
1. All configuration is in Git
2. Git is the single source of truth
3. Changes are made via Git commits
4. Tools (like ArgoCD) automatically sync Git to cluster

### Deployment Questions

**Q: In what order should I deploy components?**

**A:** Always follow this order:
1. Operators (ClickHouse, MongoDB)
2. Databases (ClickHouse, MongoDB)
3. Application (TraceFox)

**Q: Can I deploy just one component?**

**A:** Yes, but respect dependencies:
```bash
# Deploy only ClickHouse operator
kubectl apply -f argocd/apps/infrastructure/clickhouse-operator.yaml

# Deploy only staging ClickHouse
kubectl apply -f argocd/apps/staging/clickhouse.yaml
```

**Q: How do I deploy to a new environment (e.g., dev)?**

**A:**
1. Create new values file: `helm/tracefox/values-dev.yaml`
2. Create new ArgoCD project: `argocd/projects/dev.yaml`
3. Create new App of Apps: `argocd/apps/dev/app-of-apps.yaml`
4. Create application manifests in `argocd/apps/dev/`

### Configuration Questions

**Q: How do I change a configuration value?**

**A:**
1. Edit the appropriate values file (e.g., `helm/tracefox/values-staging.yaml`)
2. Commit and push to Git
3. ArgoCD will auto-sync (if enabled) or manually sync

**Q: What's the difference between values.yaml and values-staging.yaml?**

**A:**
- `values.yaml` - Base configuration (defaults)
- `values-staging.yaml` - Overrides for staging environment
- `values-prod.yaml` - Overrides for production environment

Values are merged: base → environment-specific → --set flags

**Q: Can I use the same chart for multiple environments?**

**A:** Yes! That's the whole point of Helm. Use different values files:
```bash
# Staging
helm install tracefox ./helm/tracefox -f values-staging.yaml -n staging

# Production
helm install tracefox ./helm/tracefox -f values-prod.yaml -n production
```

### Troubleshooting Questions

**Q: My pods are in CrashLoopBackOff. What do I do?**

**A:**
1. Check logs: `kubectl logs <pod-name> -n <namespace>`
2. Check events: `kubectl describe pod <pod-name> -n <namespace>`
3. Common causes: missing secrets, database not ready, wrong configuration

**Q: ArgoCD shows OutOfSync but I didn't change anything. Why?**

**A:** Possible causes:
- Someone made manual changes with `kubectl`
- Helm release was modified outside ArgoCD
- Automatic sync is disabled

Solution: Enable auto-sync or manually sync

**Q: How do I debug connection issues between pods?**

**A:**
```bash
# Run debug pod
kubectl run -it --rm debug --image=busybox -n staging -- sh

# Inside pod, test connection
wget -O- http://clickhouse-tracefox:8123/?query=SELECT%201
ping mongodb-svc
```

### Secret Management Questions

**Q: I accidentally committed a secret to Git. What do I do?**

**A:**
1. **Immediately rotate the secret** (change password/key)
2. Remove from Git history:
```bash
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/secret/file" \
  --prune-empty --tag-name-filter cat -- --all
```
3. Force push: `git push origin --force --all`
4. Notify your team

**Q: Which secret management approach should I use?**

**A:**
- **Local dev:** `--set` flags
- **Team dev:** Separate secrets file
- **Production:** External Secrets Operator

**Q: How do I rotate secrets?**

**A:**
1. Update secret in secret store (AWS Secrets Manager, etc.)
2. External Secrets Operator will automatically sync
3. Restart pods to use new secret:
```bash
kubectl rollout restart deployment/tracefox-app -n staging
```

---

## Additional Resources

### Documentation

- **Helm Charts:**
  - [TraceFox Chart](helm/tracefox/README.md)
  - [ClickHouse Chart](helm/clickhouse/README.md)
  - [MongoDB Chart](helm/mongodb/README.md)

- **ArgoCD:**
  - [ArgoCD Setup Guide](argocd/README.md)
  - [Official ArgoCD Docs](https://argo-cd.readthedocs.io/)

- **External Tools:**
  - [Helm Documentation](https://helm.sh/docs/)
  - [Kubernetes Documentation](https://kubernetes.io/docs/)
  - [External Secrets Operator](https://external-secrets.io/)

### Support

- **GitHub Issues:** https://github.com/tracefox-ai/tfx-k8s/issues
- **Documentation:** https://docs.tracefox.ai
- **Community:** https://community.tracefox.ai

---

## Contributing

We welcome contributions! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly in staging
5. Submit a pull request

---

## License

[Your License Here]

---

**Last Updated:** 2026-01-10

**Maintained by:** TraceFox Team (support@tracefox.ai)