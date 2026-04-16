<p align="center"><img src="https://raw.githubusercontent.com/mockarty/releases/main/logo.svg" width="200" alt="Mockarty"></p>

<h1 align="center">Mockarty Deployment Guide</h1>

<p align="center">
  <i>Deploy Mockarty as a single node for evaluation or as a distributed cluster for production</i>
</p>

---

## Architecture Overview

Mockarty is a multi-component platform. Depending on your needs, you can deploy a single node or a full distributed cluster.

```
                          +---------------------+
                          |    Admin Node        |
    [Clients] --------->  |  Web UI + REST API   |
                          |  MCP + Coordinator   |
                          +----------+----------+
                                     |
                      +--------------+--------------+
                      |                             |
              +-------v-------+            +--------v--------+
              | Mock Resolver |            |  Runner Agent    |
              | (serves mocks)|            | (runs tests)     |
              | x N replicas  |            | x N replicas     |
              +---------------+            +-----------------+
```

| Component | Description | Docker Image |
|-----------|-------------|--------------|
| **Admin Node** | Central server: Web UI, REST API, MCP, mock management, coordinator | `mockarty/mockarty` |
| **Mock Resolver** | Lightweight mock-serving node for horizontal scaling | `mockarty/resolver` |
| **Runner Agent** | Distributed test/performance/chaos execution agent | `mockarty/runner` |

**Minimal deployment** requires only the Admin Node + PostgreSQL. Resolvers and runners are optional and added when you need horizontal scaling or distributed test execution.

## Deployment Methods

### Docker Compose

Best for: single-server deployments, evaluation, small-to-medium teams.

See **[DOCKER_COMPOSE.md](DOCKER_COMPOSE.md)** for the full guide.

**Quick start:**

```bash
# Download files
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/docker-compose.minimal.yml
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/.env.example

# Configure
cp .env.example .env
nano .env  # set POSTGRES_PASSWORD and FIRST_ADMIN_PASSWORD

# Start
docker compose -f docker-compose.minimal.yml up -d

# Open Web UI
open http://localhost:5770/ui/
```

### Kubernetes (Helm)

Best for: production clusters, auto-scaling, high availability.

See **[KUBERNETES.md](KUBERNETES.md)** for the full guide.

**Quick start:**

```bash
# Add Bitnami repo for PostgreSQL/Redis subcharts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Download the Helm chart from the latest release
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-helm-chart-1.0.0.tgz

# Install minimal (single admin + PostgreSQL)
helm install mockarty mockarty-helm-chart-1.0.0.tgz \
  --set admin.secrets.FIRST_ADMIN_PASSWORD="your-password" \
  --set postgresql.auth.password="db-password" \
  --namespace mockarty \
  --create-namespace
```

## Port Reference

| Port | Protocol | Component | Purpose |
|------|----------|-----------|---------|
| 5770 | HTTP | Admin | Web UI, REST API, mock HTTP traffic |
| 5771 | HTTPS | Admin | TLS-terminated HTTP |
| 5772 | TCP | Admin | MCP (Model Context Protocol) server |
| 5773 | gRPC | Admin | Coordinator (internal, resolver/runner registration) |
| 5780 | HTTP | Resolver | Mock HTTP traffic serving |
| 6780 | HTTP | Resolver | Resolver dashboard UI |
| 6770 | HTTP | Runner | Runner dashboard UI |

## Links

- [Documentation](https://mockarty.ru/docs)
- [Docker Hub](https://hub.docker.com/u/mockarty)
- [GitHub Releases](https://github.com/mockarty/releases/releases)
- [Website](https://mockarty.ru)

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
