# TraceFox Helm Chart

This Helm chart deploys the TraceFox Observability Platform, a comprehensive solution for logs, metrics, and traces.

## Prerequisites

- Kubernetes cluster (1.24+)
- Helm 3.x
- **ClickHouse** - Deploy using the `clickhouse` chart in this repository
- **MongoDB** - Deploy using the `mongodb` chart in this repository
- **ClickHouse Operator** - Must be installed (managed via ArgoCD)
- **MongoDB Community Operator** - Must be installed (managed via ArgoCD)

## Architecture

TraceFox consists of several components:

- **App** - Main application server (API + Web UI)
- **OTLP Gateway** - OpenTelemetry Protocol gateway for ingesting telemetry
- **Collector Shards** - Multiple OpenTelemetry Collector instances for data processing
- **Nginx** - Reverse proxy and load balancer
- **MongoDB** - Metadata and configuration storage (separate chart)
- **ClickHouse** - Time-series data storage (separate chart)

## Installation

### Quick Start

For staging environment:
```bash
helm upgrade --install tracefox ./tracefox \
  -f ./tracefox/values-staging.yaml \
  -n staging --create-namespace \
  --set secrets.hyperdxApiKey=<your-api-key> \
  --set secrets.clickhouseAdminPassword=<clickhouse-password>
```

For production environment:
```bash
helm upgrade --install tracefox ./tracefox \
  -f ./tracefox/values-prod.yaml \
  -n production --create-namespace \
  --set secrets.hyperdxApiKey=<your-api-key> \
  --set secrets.clickhouseAdminPassword=<clickhouse-password>
```

### Using External Secrets Operator (Recommended)

1. Install External Secrets Operator:
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
```

2. Create a SecretStore (example for AWS Secrets Manager):
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
```

3. Install TraceFox with external secrets enabled:
```bash
helm upgrade --install tracefox ./tracefox \
  -f ./tracefox/values-prod.yaml \
  --set externalSecrets.enabled=true \
  --set externalSecrets.secretStore=aws-secrets-manager
```

## Configuration

### Key Configuration Options

#### Global Settings
- `global.domain` - Domain name for TraceFox
- `global.protocol` - Protocol (http/https)

#### Application
- `app.replicaCount` - Number of app replicas
- `app.env.HYPERDX_LOG_LEVEL` - Log level (debug, info, error)

#### OTLP Gateway
- `otlpGateway.replicaCount` - Number of gateway replicas
- `otlpGateway.service.ports` - gRPC (4317) and HTTP (4318) ports

#### Collector
- `collector.shards` - Number of collector shards for horizontal scaling
- `collector.image` - Collector image configuration

#### Nginx
- `nginx.enabled` - Enable/disable nginx
- `nginx.service.type` - Service type (LoadBalancer, ClusterIP)

#### Secrets
- `secrets.hyperdxApiKey` - API key for TraceFox
- `secrets.clickhouseAdminPassword` - ClickHouse admin password
- `secrets.mongoUri` - MongoDB connection URI

#### External Secrets
- `externalSecrets.enabled` - Enable External Secrets Operator
- `externalSecrets.secretStore` - Name of SecretStore
- `externalSecrets.hyperdxApiKeyPath` - Path to API key in secret backend
- `externalSecrets.clickhousePasswordPath` - Path to ClickHouse password
- `externalSecrets.mongoUriPath` - Path to MongoDB URI

### Environment-Specific Values

**Staging** (`values-staging.yaml`):
- 2 app replicas
- 2 collector shards
- ClusterIP service
- Info log level

**Production** (`values-prod.yaml`):
- 5 app replicas
- 4 collector shards
- LoadBalancer service
- Error log level

## Accessing TraceFox

### LoadBalancer (Production)
```bash
export SERVICE_IP=$(kubectl get svc tracefox-nginx -n production -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "TraceFox UI: http://$SERVICE_IP"
```

### Port Forward (Staging/Development)
```bash
kubectl port-forward svc/tracefox-nginx -n staging 8080:80
# Access at http://localhost:8080
```

## Upgrading

To upgrade the chart:
```bash
helm upgrade tracefox ./tracefox \
  -f ./tracefox/values-prod.yaml \
  -n production
```

## Uninstallation

To uninstall the chart:
```bash
helm uninstall tracefox -n production
```

**Note**: This will remove TraceFox but not ClickHouse or MongoDB. Uninstall those separately if needed.

## Troubleshooting

### Pods not starting
```bash
# Check pod status
kubectl get pods -n production

# Check pod logs
kubectl logs -n production -l app.kubernetes.io/name=tracefox

# Describe pod for events
kubectl describe pod -n production <pod-name>
```

### Cannot connect to ClickHouse
```bash
# Verify ClickHouse is running
kubectl get chi -n production

# Test connection from app pod
kubectl exec -n production <app-pod> -- curl http://clickhouse-tracefox:8123/?query=SELECT%201
```

### Cannot connect to MongoDB
```bash
# Verify MongoDB is running
kubectl get mongodbcommunity -n production

# Test connection
kubectl exec -n production <app-pod> -- mongosh mongodb://mongodb-svc:27017 --eval "db.version()"
```

### Secrets not loading
```bash
# Check if secrets exist
kubectl get secrets -n production

# If using external-secrets, check ExternalSecret status
kubectl get externalsecret -n production
kubectl describe externalsecret tracefox-secrets -n production
```

## Security Best Practices

1. **Never commit secrets to Git** - Use External Secrets Operator or `--set` flags
2. **Use TLS** - Configure ingress with TLS certificates
3. **Network Policies** - Restrict pod-to-pod communication
4. **RBAC** - Use least-privilege service accounts
5. **Image Scanning** - Scan images for vulnerabilities
6. **Regular Updates** - Keep chart and images up to date

## Values Reference

See `values.yaml` for all available configuration options with detailed comments.

## Support

For issues and questions:
- GitHub Issues: https://github.com/tracefox-ai/tfx-k8s/issues
- Documentation: https://docs.tracefox.ai
