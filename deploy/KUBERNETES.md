<p align="center"><img src="https://raw.githubusercontent.com/mockarty/releases/main/logo.svg" width="200" alt="Mockarty"></p>

<h1 align="center">Kubernetes Deployment</h1>

---

## Prerequisites

- Kubernetes 1.24+
- Helm 3.10+
- `kubectl` configured with cluster access
- Persistent volume provisioner (for PostgreSQL and Redis storage)

---

## Install the Helm Chart

### From GitHub Releases

```bash
# Download the chart archive from the latest release
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-helm-chart-1.0.0.tgz

# Add Bitnami repo for PostgreSQL and Redis subcharts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### From OCI Registry (Docker Hub)

```bash
helm pull oci://registry-1.docker.io/mockarty/helm-chart --version 1.0.0
```

---

## Minimal Install (Single Admin + PostgreSQL)

Deploy a single admin node with an embedded PostgreSQL instance. Suitable for evaluation, development, and small teams.

```bash
helm install mockarty mockarty-helm-chart-1.0.0.tgz \
  --set admin.secrets.FIRST_ADMIN_PASSWORD="your-password" \
  --set postgresql.auth.password="db-password" \
  --namespace mockarty \
  --create-namespace
```

Minimal values profile disables resolvers, runners, orchestrator, and Redis:

```bash
helm install mockarty mockarty-helm-chart-1.0.0.tgz \
  -f values.minimal.yaml \
  --set admin.secrets.FIRST_ADMIN_PASSWORD="your-password" \
  --set postgresql.auth.password="db-password" \
  --namespace mockarty \
  --create-namespace
```

### Verify Installation

```bash
kubectl get pods -n mockarty
kubectl get svc -n mockarty

# Wait for admin to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/component=admin -n mockarty --timeout=120s

# Check health
kubectl port-forward svc/mockarty-admin 5770:5770 -n mockarty
curl http://localhost:5770/health/live
```

---

## Full Cluster Install

Deploy the complete Mockarty platform: 2 admin nodes, 2 resolvers, 2 runners, 1 orchestrator, PostgreSQL, and Redis with auto-scaling and pod disruption budgets.

```bash
helm install mockarty mockarty-helm-chart-1.0.0.tgz \
  -f values.cluster.yaml \
  --set admin.secrets.FIRST_ADMIN_PASSWORD="your-password" \
  --set postgresql.auth.password="db-password" \
  --namespace mockarty \
  --create-namespace
```

The **token bootstrap Job** runs automatically after install and creates integration tokens (`mki_*` prefix) for resolver, runner, and orchestrator nodes. No manual token setup is needed.

### Verify Cluster

```bash
# All pods should be Running
kubectl get pods -n mockarty

# Expected output:
# mockarty-admin-0              1/1   Running
# mockarty-admin-1              1/1   Running
# mockarty-resolver-xxxxx       1/1   Running
# mockarty-resolver-xxxxx       1/1   Running
# mockarty-runner-xxxxx         1/1   Running
# mockarty-runner-xxxxx         1/1   Running
# mockarty-orchestrator-xxxxx   1/1   Running
# mockarty-postgresql-0         1/1   Running
# mockarty-redis-master-0       1/1   Running
# mockarty-token-bootstrap-xxx  0/1   Completed
```

---

## Configuration Reference

### Global

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.registry` | Docker image registry prefix | `""` (Docker Hub) |
| `global.imagePullPolicy` | Image pull policy | `IfNotPresent` |
| `global.imagePullSecrets` | Image pull secrets for private registries | `[]` |

### Admin Node

| Parameter | Description | Default |
|-----------|-------------|---------|
| `admin.enabled` | Deploy admin node | `true` |
| `admin.replicaCount` | Number of admin replicas | `1` |
| `admin.image.repository` | Admin Docker image | `mockarty/mockarty` |
| `admin.image.tag` | Admin image tag | `latest` |
| `admin.secrets.FIRST_ADMIN_PASSWORD` | Initial admin password | `change_me` |
| `admin.env.CACHE_TYPE` | Cache backend (`inmemory` or `redis`) | `redis` |
| `admin.env.CLUSTER_MODE` | Enable multi-replica cluster mode | `true` |
| `admin.env.LOG_LEVEL` | Logging level | `info` |
| `admin.env.ENABLE_MCP` | Enable MCP server | `true` |
| `admin.env.RUNNER_COORDINATOR_ENABLED` | Enable coordinator for resolvers/runners | `true` |
| `admin.env.ADK_ENABLED` | Enable Agent Development Kit | `true` |
| `admin.env.A2A_ENABLED` | Enable Agent-to-Agent protocol | `true` |
| `admin.ingress.enabled` | Enable ingress for admin | `false` |
| `admin.autoscaling.enabled` | Enable HPA for admin | `false` |
| `admin.podDisruptionBudget.enabled` | Enable PDB for admin | `false` |
| `admin.resources.requests.cpu` | CPU request | `200m` |
| `admin.resources.requests.memory` | Memory request | `256Mi` |
| `admin.resources.limits.cpu` | CPU limit | `1000m` |
| `admin.resources.limits.memory` | Memory limit | `1Gi` |

### Mock Resolver

| Parameter | Description | Default |
|-----------|-------------|---------|
| `resolver.enabled` | Deploy resolver nodes | `false` |
| `resolver.replicaCount` | Number of resolver replicas | `2` |
| `resolver.image.repository` | Resolver Docker image | `mockarty/resolver` |
| `resolver.image.tag` | Resolver image tag | `latest` |
| `resolver.apiToken` | Resolver integration token | `""` (use tokenBootstrap) |
| `resolver.env.CACHE_SYNC_INTERVAL` | Cache sync interval | `60s` |
| `resolver.env.STORE_MODE` | Store access mode | `read_only` |
| `resolver.env.CACHE_MAX_SIZE_MB` | Max cache size in MB | `512` |
| `resolver.autoscaling.enabled` | Enable HPA for resolver | `false` |
| `resolver.podDisruptionBudget.enabled` | Enable PDB for resolver | `false` |

### Runner Agent

| Parameter | Description | Default |
|-----------|-------------|---------|
| `runner.enabled` | Deploy runner agents | `false` |
| `runner.replicaCount` | Number of runner replicas | `1` |
| `runner.image.repository` | Runner Docker image | `mockarty/runner` |
| `runner.image.tag` | Runner image tag | `latest` |
| `runner.apiToken` | Runner integration token | `""` (use tokenBootstrap) |
| `runner.env.RUNNER_MODE` | Runner mode (`grpc`, `poll`, `reverse`) | `grpc` |
| `runner.env.MAX_CONCURRENT_TASKS` | Max concurrent tasks | `3` |
| `runner.env.CAPABILITIES` | Runner capabilities | `api_test,performance` |
| `runner.autoscaling.enabled` | Enable HPA for runner | `false` |

### Orchestrator

| Parameter | Description | Default |
|-----------|-------------|---------|
| `orchestrator.enabled` | Deploy orchestrator | `false` |
| `orchestrator.replicaCount` | Orchestrator replicas | `1` |
| `orchestrator.image.repository` | Orchestrator Docker image | `mockarty/generator` |
| `orchestrator.image.tag` | Orchestrator image tag | `latest` |
| `orchestrator.env.MODE` | Mode (`full` or `api`) | `full` |
| `orchestrator.persistence.enabled` | Enable persistent storage | `true` |
| `orchestrator.persistence.size` | Storage size | `5Gi` |

### Database

| Parameter | Description | Default |
|-----------|-------------|---------|
| `postgresql.enabled` | Deploy embedded PostgreSQL | `true` |
| `postgresql.auth.username` | PostgreSQL username | `mockarty` |
| `postgresql.auth.password` | PostgreSQL password | `change_me` |
| `postgresql.auth.database` | PostgreSQL database name | `mockarty` |
| `postgresql.primary.persistence.size` | PostgreSQL storage size | `10Gi` |
| `externalDatabase.host` | External DB host (when `postgresql.enabled=false`) | `""` |
| `externalDatabase.dsn` | Full DSN (takes precedence over host/port) | `""` |

### Cache

| Parameter | Description | Default |
|-----------|-------------|---------|
| `redis.enabled` | Deploy embedded Redis | `true` |
| `redis.auth.enabled` | Enable Redis authentication | `false` |
| `redis.master.persistence.size` | Redis storage size | `2Gi` |
| `externalRedis.host` | External Redis host (when `redis.enabled=false`) | `""` |

### Token Bootstrap

| Parameter | Description | Default |
|-----------|-------------|---------|
| `tokenBootstrap.enabled` | Auto-create integration tokens | `false` |
| `tokenBootstrap.image` | Job container image | `bitnami/kubectl:latest` |

### Network Policies

| Parameter | Description | Default |
|-----------|-------------|---------|
| `networkPolicy.enabled` | Create NetworkPolicy resources | `false` |

---

## Value Profiles

| Profile | File | Description |
|---------|------|-------------|
| Default | `values.yaml` | Base values, all optional components disabled |
| Minimal | `values.minimal.yaml` | Single admin + PostgreSQL, no Redis/resolver/runner |
| Cluster | `values.cluster.yaml` | Full HA: 2 admin, resolvers, runners, orchestrator, PG, Redis |

Download value profiles from the [releases page](https://github.com/mockarty/releases/releases) and customize:

```bash
helm install mockarty mockarty-helm-chart-1.0.0.tgz \
  -f values.cluster.yaml \
  -f my-overrides.yaml \
  --namespace mockarty \
  --create-namespace
```

---

## Ingress Configuration

### nginx Ingress Controller

```yaml
admin:
  ingress:
    enabled: true
    className: "nginx"
    annotations:
      nginx.ingress.kubernetes.io/proxy-body-size: "50m"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    hosts:
      - host: mockarty.example.com
        paths:
          - path: /
            pathType: Prefix
```

### TLS with cert-manager

```yaml
admin:
  ingress:
    enabled: true
    className: "nginx"
    annotations:
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    hosts:
      - host: mockarty.example.com
        paths:
          - path: /
            pathType: Prefix
    tls:
      - secretName: mockarty-tls
        hosts:
          - mockarty.example.com
```

Make sure cert-manager is installed in your cluster:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## Network Policies

When `networkPolicy.enabled=true`, the chart creates NetworkPolicy resources that restrict traffic:

- **Admin** -- accepts ingress on HTTP/HTTPS/MCP/Coordinator ports; egress to PostgreSQL, Redis
- **Resolver** -- accepts ingress on HTTP/gRPC/UI ports; egress to PostgreSQL, Admin (coordinator)
- **Runner** -- accepts ingress on UI port; egress to Admin (coordinator)
- **PostgreSQL** -- accepts ingress only from Admin and Resolver pods
- **Redis** -- accepts ingress only from Admin pods

Enable in your values:

```yaml
networkPolicy:
  enabled: true
```

---

## Horizontal Pod Autoscaler (HPA)

Enable auto-scaling for any component:

```yaml
resolver:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

runner:
  autoscaling:
    enabled: true
    minReplicas: 1
    maxReplicas: 5
    targetCPUUtilizationPercentage: 70

admin:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
```

Requires the [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) to be installed in your cluster.

---

## Pod Disruption Budget (PDB)

Ensure minimum availability during voluntary disruptions (node drains, upgrades):

```yaml
admin:
  podDisruptionBudget:
    enabled: true
    minAvailable: 1

resolver:
  podDisruptionBudget:
    enabled: true
    minAvailable: 1
```

---

## Upgrading

### Upgrade All Components

```bash
helm upgrade mockarty mockarty-helm-chart-1.2.0.tgz \
  --reuse-values \
  --namespace mockarty
```

### Upgrade to a Specific Image Tag

```bash
helm upgrade mockarty mockarty-helm-chart-1.0.0.tgz \
  --reuse-values \
  --set admin.image.tag="1.2.0" \
  --set resolver.image.tag="1.2.0" \
  --set runner.image.tag="1.2.0" \
  --set orchestrator.image.tag="1.2.0" \
  --namespace mockarty
```

The RollingUpdate strategy with `maxSurge=1, maxUnavailable=0` ensures zero-downtime upgrades.

Database migrations are applied automatically on pod startup.

---

## Rollback

```bash
# List revision history
helm history mockarty -n mockarty

# Rollback to a specific revision
helm rollback mockarty <revision> -n mockarty
```

The chart retains the last 5 ReplicaSets (`revisionHistoryLimit: 5`) for instant rollback.

---

## Monitoring with Prometheus

Mockarty exposes Prometheus metrics at `/metrics` on the admin HTTP port. Scrape configuration:

```yaml
# ServiceMonitor (requires prometheus-operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mockarty
  namespace: mockarty
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: mockarty
      app.kubernetes.io/component: admin
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

Or add a Prometheus scrape annotation to admin pods:

```yaml
admin:
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "5770"
    prometheus.io/path: "/metrics"
```

---

## Uninstalling

```bash
# Remove the Helm release
helm uninstall mockarty --namespace mockarty

# PVCs are NOT deleted automatically -- remove them manually if desired
kubectl delete pvc -l app.kubernetes.io/instance=mockarty -n mockarty

# Optionally delete the namespace
kubectl delete namespace mockarty
```

---

## Kubernetes Operator (Advanced)

For advanced users, Mockarty provides a Kubernetes Operator that manages the platform via Custom Resource Definitions (CRDs).

### MockartyCluster CRD

The operator watches `MockartyCluster` resources and reconciles the desired state:

```yaml
apiVersion: mockarty.io/v1alpha1
kind: MockartyCluster
metadata:
  name: production
  namespace: mockarty
spec:
  version: "1.2.0"

  admin:
    replicas: 2
    resources:
      requests:
        cpu: 500m
        memory: 512Mi

  resolver:
    replicas: 3
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 10
      targetCPU: 70

  runner:
    replicas: 2
    capabilities:
      - api_test
      - performance
      - chaos

  postgresql:
    storageSize: 20Gi

  redis:
    enabled: true

  ingress:
    host: mockarty.example.com
    tls: true
    issuer: letsencrypt-prod
```

### Install the Operator

```bash
# Install the operator CRDs and controller
kubectl apply -f https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-operator.yaml

# Deploy a cluster
kubectl apply -f mockarty-cluster.yaml

# Check status
kubectl get mockartycluster -n mockarty
```

### Operator Features

- Automatic token bootstrap for resolver/runner/orchestrator nodes
- Rolling upgrades with health validation
- Auto-scaling integration
- Namespace-scoped multi-tenant deployments
- CLI integration (`mockarty-cli cluster status`)

> **Note:** The Kubernetes Operator is available in Mockarty Enterprise editions. Contact [support@mockarty.ru](mailto:support@mockarty.ru) for details.

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
