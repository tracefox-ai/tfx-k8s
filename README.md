# TraceFox Kubernetes Infrastructure

This repository contains the Kubernetes infrastructure configuration for TraceFox, managed via Helm charts.

## Directory Structure

*   **`helm/`**: Contains the Helm charts and environment configurations.
    *   **`helm/clickhouse/`**: ClickHouse Helm chart for deploying ClickHouse clusters.
    *   **`helm/tracefox/`**: The main application Helm chart.

## Quick Start

### Prerequisites

*   Kubernetes cluster (e.g., Docker Desktop, EKS, GKE).
*   Helm 3+ installed.

### Deploying the Application

Go to the `helm` directory for detailed instructions:

[View Helm Documentation](helm/README.md)

### Deployment Examples

**Important**: Deploy components in this order:
1. ClickHouse Operator (required for ClickHouse)
2. ClickHouse (required for TraceFox)
3. TraceFox Application

#### Step 1: Deploy ClickHouse Operator

The ClickHouse Operator must be installed first as it manages ClickHouse resources. We use the official [Altinity Helm chart](https://github.com/Altinity/helm-charts/tree/main/charts/clickhouse-eks).

Add the Altinity Helm repository:
```bash
helm repo add altinity-operator https://docs.altinity.com/clickhouse-operator
```

Install the ClickHouse Operator:
```bash
# create the namespace
kubectl create namespace clickhouse

# install operator into namespace
helm install clickhouse-operator altinity/altinity-clickhouse-operator \
--namespace clickhouse
```

Verify the operator is running:
```bash
kubectl get pods -n clickhouse | grep clickhouse-operator
```

#### Step 2: Deploy ClickHouse

**Staging:**
```bash
cd helm
helm upgrade --install clickhouse ./clickhouse \
  -n clickhouse \
  -f clickhouse/values-staging.yaml
```

**Production:**
```bash
cd helm
helm upgrade --install clickhouse ./clickhouse \
  -n clickhouse \
  -f clickhouse/values-prod.yaml
```

#### Step 3: Deploy TraceFox Application

**Staging:**
```bash
cd helm
helm upgrade --install tracefox ./tracefox \
  --namespace staging --create-namespace \
  -f environments/values-staging.yaml
```

**Production:**
```bash
cd helm
helm upgrade --install tracefox ./tracefox \
  --namespace prod --create-namespace \
  -f environments/values-prod.yaml
```

For more details, see:
*   [ClickHouse Chart Documentation](helm/clickhouse/README.md)
*   [TraceFox Chart Documentation](helm/README.md)




kubectl -n clickhouse port-forward \
    services/clickhouse-clickhouse \
    8123:8123