# MongoDB Helm Chart

This Helm chart deploys a MongoDB Replica Set using the MongoDB Community Operator.

## Prerequisites

- Kubernetes cluster (1.24+)
- Helm 3.x
- **MongoDB Community Operator** must be installed (managed via ArgoCD in `argocd/apps/infrastructure/mongodb-operator.yaml`)

## Installation

### Step 1: Verify MongoDB Operator is Running

The MongoDB Community Operator must be installed before deploying this chart:

```bash
# Check if operator is running
kubectl get pods -n mongodb | grep mongodb-kubernetes-operator
```

If not installed, it should be deployed via ArgoCD. See `argocd/apps/infrastructure/mongodb-operator.yaml`.

### Step 2: Install the Chart

For staging:
```bash
helm upgrade --install mongodb ./mongodb \
  -f ./mongodb/values-staging.yaml \
  -n staging --create-namespace \
  --set secret.password=<your-secure-password>
```

For production:
```bash
helm upgrade --install mongodb ./mongodb \
  -f ./mongodb/values.yaml \
  -n production --create-namespace \
  --set secret.password=<your-secure-password>
```

**Note**: The `--create-namespace` flag creates the namespace if it doesn't exist. Omit if the namespace already exists.

## Configuration

### Key Configuration Options

#### MongoDB Settings
- `mongodb.enabled` - Enable/disable MongoDB deployment
- `mongodb.name` - Name of the MongoDB resource
- `mongodb.members` - Number of replica set members (default: 3)
- `mongodb.version` - MongoDB version (default: "6.0.5")
- `mongodb.type` - Deployment type (ReplicaSet)

#### Security
- `mongodb.security.authentication.modes` - Authentication modes (default: ["SCRAM"])
- `mongodb.users` - User configuration with roles

#### Secret Configuration
- `secret.enabled` - Enable secret creation
- `secret.name` - Name of the password secret
- `secret.password` - User password (set via --set flag, not in Git!)

### Environment-Specific Values

**Staging** (`values-staging.yaml`):
- 3 replica set members
- Standard configuration

**Production** (`values.yaml`):
- 3 replica set members
- Production-ready configuration

## Accessing MongoDB

### From Within the Cluster

Applications in the same namespace can connect using:
```
mongodb://<mongodb-name>-svc.<namespace>.svc.cluster.local:27017
```

Example:
```
mongodb://mongodb-svc.production.svc.cluster.local:27017
```

### Port Forward for Local Access

```bash
kubectl port-forward -n production svc/mongodb-svc 27017:27017

# Then connect locally
mongosh "mongodb://localhost:27017" -u my-user -p <your-password>
```

## Verification

After installation, verify the deployment:

```bash
# Check MongoDB resource
kubectl get mongodbcommunity -n production

# Check pods
kubectl get pods -n production -l app=mongodb

# Check replica set status
kubectl exec -n production mongodb-0 -- mongosh --eval "rs.status()"
```

Expected output:
- 3 MongoDB pods running (mongodb-0, mongodb-1, mongodb-2)
- Replica set status showing PRIMARY and SECONDARY members

## Upgrading

To upgrade the chart:

```bash
helm upgrade mongodb ./mongodb \
  -f ./mongodb/values.yaml \
  -n production
```

## Uninstallation

To uninstall the chart:

```bash
helm uninstall mongodb -n production
```

**Warning**: This will delete the MongoDB replica set and all data. Make sure to backup data before uninstalling.

## Troubleshooting

### Pods Not Starting

```bash
# Check pod status
kubectl get pods -n production -l app=mongodb

# Check pod logs
kubectl logs -n production mongodb-0

# Describe pod for events
kubectl describe pod -n production mongodb-0
```

### Operator Not Found

```bash
# Verify operator is installed
kubectl get deployment -n mongodb mongodb-kubernetes-operator

# Check operator logs
kubectl logs -n mongodb -l app.kubernetes.io/name=mongodb-kubernetes-operator
```

### Replica Set Not Forming

```bash
# Check replica set status
kubectl exec -n production mongodb-0 -- mongosh --eval "rs.status()"

# Check MongoDB logs
kubectl logs -n production mongodb-0

# Verify network connectivity between pods
kubectl exec -n production mongodb-0 -- ping mongodb-1.mongodb-svc
```

### Authentication Issues

```bash
# Verify secret exists
kubectl get secret my-user-password -n production

# Check secret content (base64 encoded)
kubectl get secret my-user-password -n production -o yaml

# Verify user was created
kubectl exec -n production mongodb-0 -- mongosh --eval "db.getUsers()"
```

## Backup and Restore

### Backup

```bash
# Backup a specific database
kubectl exec -n production mongodb-0 -- mongodump --db=<database> --out=/tmp/backup

# Copy backup from pod
kubectl cp production/mongodb-0:/tmp/backup ./backup
```

### Restore

```bash
# Copy backup to pod
kubectl cp ./backup production/mongodb-0:/tmp/backup

# Restore database
kubectl exec -n production mongodb-0 -- mongorestore --db=<database> /tmp/backup/<database>
```

## Security Best Practices

1. **Strong Passwords** - Use strong, randomly generated passwords
2. **Secret Management** - Never commit passwords to Git; use External Secrets Operator
3. **Network Policies** - Restrict network access to MongoDB pods
4. **TLS/SSL** - Enable TLS for encrypted connections (configure in values)
5. **RBAC** - Use least-privilege MongoDB roles
6. **Regular Backups** - Implement automated backup strategy

## Performance Tuning

### Storage

For production workloads, use high-performance storage:
```yaml
mongodb:
  statefulSet:
    spec:
      volumeClaimTemplates:
        - spec:
            storageClassName: fast-ssd
            resources:
              requests:
                storage: 100Gi
```

### Resource Limits

Adjust based on workload:
```yaml
mongodb:
  statefulSet:
    spec:
      template:
        spec:
          containers:
            - resources:
                limits:
                  cpu: "4"
                  memory: "8Gi"
                requests:
                  cpu: "2"
                  memory: "4Gi"
```

## Values Reference

See `values.yaml` for all available configuration options with detailed comments.

## Additional Resources

- [MongoDB Community Operator Documentation](https://github.com/mongodb/mongodb-kubernetes-operator)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
