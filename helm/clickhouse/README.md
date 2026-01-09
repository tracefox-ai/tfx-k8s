# ClickHouse Helm Chart

This Helm chart deploys a ClickHouse cluster using the Altinity ClickHouse Operator.

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- **ClickHouse Operator must be installed separately** (see below)

## Installation

### Step 1: Install the ClickHouse Operator

The **ClickHouse Operator** must be installed before deploying this chart. The operator watches for `ClickHouseInstallation` and `ClickHouseKeeperInstallation` resources and creates the actual pods.

Install the operator from the bundle file:

```bash
kubectl apply -f ../clickhouse/01_clickhouse-operator-install-bundle.yaml
```

Or install from the official repository:

```bash
kubectl apply -f https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/deploy/dev/manifests/01-clickhouse-operator-install-bundle.yaml
```

Verify the operator is running:

```bash
kubectl get pods -n kube-system | grep clickhouse-operator
```

### Step 2: Install the Chart

The chart includes the ClickHouse Operator CRDs in the `crds/` directory. Helm 3+ will automatically install these CRDs before deploying the chart templates.

For staging:
```bash
helm upgrade --install clickhouse ./clickhouse \
  -f ./clickhouse/values-staging.yaml \
  -n clickhouse --create-namespace
```

For production:
```bash
helm upgrade --install clickhouse ./clickhouse \
  -f ./clickhouse/values-prod.yaml \
  -n clickhouse --create-namespace
```

**Note**: Using `helm upgrade --install` is recommended as it will install if the release doesn't exist, or upgrade if it does. The `--create-namespace` flag will create the namespace if it doesn't exist. If the namespace already exists (from a previous installation), you can omit the `--create-namespace` flag.

## Configuration

The chart supports extensive configuration through values files. Key parameters include:

### Keeper Configuration
- `keeper.enabled`: Enable/disable ClickHouse Keeper
- `keeper.replicasCount`: Number of Keeper replicas (default: 3)
- `keeper.resources`: Resource requests and limits
- `keeper.storage.size`: Storage size for Keeper data

### ClickHouse Configuration
- `clickhouse.enabled`: Enable/disable ClickHouse installation
- `clickhouse.cluster.shardsCount`: Number of shards
- `clickhouse.cluster.replicasCount`: Number of replicas per shard
- `clickhouse.resources`: Resource requests and limits
- `clickhouse.storage.size`: Storage size for ClickHouse data
- `clickhouse.users.admin.password`: Admin user password

### Environment-Specific Values

- **Staging**: `clickhouse/values-staging.yaml`
  - Smaller resource allocations
  - 2 shards, 2 replicas
  - ClusterIP service type

- **Production**: `clickhouse/values-prod.yaml`
  - Larger resource allocations
  - 4 shards, 3 replicas
  - LoadBalancer service type
  - Increased storage sizes

## Verification

After installation, verify the pods are running:

```bash
# Check operator
kubectl get pods -n kube-system | grep clickhouse-operator

# Check ClickHouse resources
kubectl get chi,chk -n clickhouse

# Check all pods
kubectl get pods -n clickhouse
```

Expected output:
- Operator pod: `clickhouse-operator-*` in `kube-system` namespace
- Keeper pods: `chk-clickhouse-keeper-cluster-0-0`, `chk-clickhouse-keeper-cluster-0-1`, `chk-clickhouse-keeper-cluster-0-2`
- ClickHouse pods: `chi-clickhouse-cluster-0-0-0`, `chi-clickhouse-cluster-0-1-0`, etc.

## Port Forwarding

To connect to ClickHouse:

```bash
kubectl port-forward -n clickhouse service/chi-clickhouse-cluster-0-0 8123:8123
```

Then connect using:
```bash
clickhouse-client --host localhost --port 8123 --user admin --password <password>
```

## Upgrading

To upgrade the chart:

```bash
helm upgrade clickhouse ./clickhouse \
  -f ./clickhouse/values-prod.yaml \
  -n clickhouse
```

## Uninstallation

To uninstall the chart:

```bash
helm uninstall clickhouse -n clickhouse
```

**Note**: This will remove the ClickHouse installation and Keeper. The operator will remain in the `kube-system` namespace and must be removed separately if needed.

## Values Reference

See `values.yaml` for all available configuration options.
