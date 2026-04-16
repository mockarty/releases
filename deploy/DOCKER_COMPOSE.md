<p align="center"><img src="https://raw.githubusercontent.com/mockarty/releases/main/logo.svg" width="200" alt="Mockarty"></p>

<h1 align="center">Docker Compose Deployment</h1>

---

## Prerequisites

- Docker Engine 24+ and Docker Compose v2
- At least 2 GB of available RAM
- Ports 5770-5773 available on the host (configurable)

## Deployment Modes

| Mode | Services | Best For |
|------|----------|----------|
| **Minimal** | Admin + PostgreSQL | Evaluation, development, small teams |
| **Full Stack** | Admin + PostgreSQL + Redis + Kafka + RabbitMQ | Teams using message queue mocking |
| **Distributed** | Admin + Resolver(s) + Runner(s) + PostgreSQL + Redis | Production, horizontal scaling |

---

## 1. Minimal Deployment

A single Mockarty admin node with PostgreSQL. This is the simplest way to get started.

### Download and Configure

```bash
# Download compose file and env template
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/docker-compose.minimal.yml
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/.env.example

# Create your configuration
cp .env.example .env
```

Edit `.env` and set the required values:

```dotenv
POSTGRES_PASSWORD=your-secure-db-password
FIRST_ADMIN_PASSWORD=your-admin-password
```

### Start

```bash
docker compose -f docker-compose.minimal.yml up -d
```

### Verify

```bash
# Check services are healthy
docker compose -f docker-compose.minimal.yml ps

# Check health endpoint
curl http://localhost:5770/health/live
```

### Access

| URL | Description |
|-----|-------------|
| http://localhost:5770/ui/ | Web UI |
| http://localhost:5770/ | REST API |
| http://localhost:5770/swagger/ | Swagger UI |
| https://localhost:5771/ | HTTPS (when TLS configured) |
| localhost:5772 | MCP server |

---

## 2. Full Stack Deployment

Extends the minimal setup with Redis, Kafka, and RabbitMQ for message queue mocking.

```bash
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/docker-compose.full.yml
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/.env.example
cp .env.example .env
# Edit .env with your passwords

docker compose -f docker-compose.full.yml up -d
```

Additional services:

| Service | Port | Purpose |
|---------|------|---------|
| Redis | 6379 | Caching layer |
| Kafka | 9092 | Kafka message queue mocking |
| RabbitMQ | 5672 / 15672 | RabbitMQ mocking + management UI |

---

## 3. Distributed Deployment

Deploy the full Mockarty platform with horizontally scalable resolver and runner nodes.

### Architecture

```
[clients] --> [admin:5770/5771/5772]
                  |
                  +-- coordinator:5773 (internal) <-- [mockarty-resolver x N]
                                                  <-- [mockarty-runner x N]
```

- **Admin Node** -- central coordinator, Web UI, API, mock management
- **Mockarty Resolver** -- serves mock HTTP traffic, scales horizontally
- **Mockarty Runner** -- executes test collections, scales horizontally

### Download and Configure

```bash
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/docker-compose.distributed.yml
curl -LO https://raw.githubusercontent.com/mockarty/releases/main/deploy/.env.example
cp .env.example .env
```

### Token Setup Workflow

Resolver and runner nodes authenticate to the admin coordinator using API tokens. Follow this bootstrap sequence:

**Step 1.** Start the admin node and infrastructure first:

```bash
docker compose -f docker-compose.distributed.yml up -d mockarty postgres redis
```

**Step 2.** Wait for the admin node to become healthy:

```bash
docker compose -f docker-compose.distributed.yml ps
# Wait until mockarty-admin shows "healthy"
```

**Step 3.** Open the admin UI and create tokens:

1. Navigate to http://localhost:5770/ui/
2. Log in with the admin credentials from your `.env`
3. Go to **Settings > API Tokens > Create Token**
4. Create one token with the **Resolver** role
5. Create one token with the **Runner** role

**Step 4.** Add tokens to your `.env` file:

```dotenv
RESOLVER_API_TOKEN=mk_your_resolver_token_here
RUNNER_API_TOKEN=mk_your_runner_token_here
```

**Step 5.** Start the full stack:

```bash
docker compose -f docker-compose.distributed.yml up -d
```

### Scaling Resolver and Runner Nodes

Resolver and runner services use `expose` (not `ports`) to avoid host port conflicts when running multiple replicas. Scale them independently:

```bash
# Scale to 3 resolvers and 2 runners
docker compose -f docker-compose.distributed.yml up -d \
  --scale mockarty-resolver=3 \
  --scale mockarty-runner=2
```

To expose resolver nodes externally, place a reverse proxy (Traefik, nginx, HAProxy) in front of them or run a single replica with explicit port bindings.

---

## Environment Variables

### Required

| Variable | Description | Default |
|----------|-------------|---------|
| `POSTGRES_PASSWORD` | PostgreSQL password | -- (required) |
| `FIRST_ADMIN_PASSWORD` | Initial admin UI password | -- (required) |

### Optional -- General

| Variable | Description | Default |
|----------|-------------|---------|
| `VERSION` | Docker image tag | `latest` |
| `POSTGRES_USER` | PostgreSQL user | `mockarty` |
| `POSTGRES_DB` | PostgreSQL database name | `mockarty` |
| `POSTGRES_PORT` | PostgreSQL host port | `5432` |
| `HTTP_PORT` | Admin HTTP port | `5770` |
| `HTTPS_PORT` | Admin HTTPS port | `5771` |
| `MCP_PORT` | Admin MCP port | `5772` |
| `COORDINATOR_PORT` | Coordinator gRPC port | `5773` |
| `LOG_LEVEL` | Logging level (`debug`, `info`, `warn`, `error`) | `info` |
| `FIRST_ADMIN_LOGIN` | Initial admin username | `admin` |
| `AUTH_BASE_URL` | Base URL for OAuth redirects | `http://localhost:5770` |

### Optional -- Cache

| Variable | Description | Default |
|----------|-------------|---------|
| `REDIS_PORT` | Redis host port | `6379` |
| `REDIS_PASSWORD` | Redis password (empty = no auth) | -- |

### Optional -- Distributed Mode

| Variable | Description | Default |
|----------|-------------|---------|
| `RESOLVER_API_TOKEN` | API token for resolver nodes | -- (required for distributed) |
| `RUNNER_API_TOKEN` | API token for runner agents | -- (required for distributed) |
| `MAX_CONCURRENT_TASKS` | Max concurrent tasks per runner | `3` |
| `RUNNER_CAPABILITIES` | Runner capabilities | `api_test,performance` |

---

## Port Reference

| Port | Service | Purpose |
|------|---------|---------|
| 5770 | Admin | HTTP -- Web UI, REST API, mock traffic |
| 5771 | Admin | HTTPS -- TLS-terminated HTTP |
| 5772 | Admin | MCP -- Model Context Protocol server |
| 5773 | Admin | Coordinator gRPC (internal) |
| 5780 | Resolver | HTTP mock server |
| 6780 | Resolver | Dashboard UI |
| 6770 | Runner | Dashboard UI |
| 5432 | PostgreSQL | Database |
| 6379 | Redis | Cache |

---

## Health Checks

All Mockarty services expose health check endpoints:

```bash
# Admin node
curl http://localhost:5770/health/live

# Resolver (from within Docker network)
docker exec <resolver-container> curl -f http://localhost:5780/health/live

# Runner (from within Docker network)
docker exec <runner-container> curl -f http://localhost:6770/health
```

The compose files include built-in Docker health checks. Use `docker compose ps` to monitor service health status.

---

## Upgrading

### Pull New Images

```bash
# Update to a specific version
VERSION=1.2.0 docker compose -f docker-compose.minimal.yml pull
VERSION=1.2.0 docker compose -f docker-compose.minimal.yml up -d
```

Or set `VERSION` in your `.env` file and run:

```bash
docker compose -f docker-compose.minimal.yml pull
docker compose -f docker-compose.minimal.yml up -d
```

Database migrations are applied automatically on startup.

### Zero-Downtime Upgrade (Distributed)

For distributed deployments, upgrade components in order:

1. Pull new images: `docker compose pull`
2. Upgrade admin first: `docker compose up -d --no-deps mockarty`
3. Wait for admin to be healthy
4. Upgrade resolvers: `docker compose up -d --no-deps mockarty-resolver`
5. Upgrade runners: `docker compose up -d --no-deps mockarty-runner`

---

## Backup

### Database Backup

```bash
# Create a backup
docker exec mockarty-db pg_dump -U mockarty mockarty > backup_$(date +%Y%m%d).sql

# Restore from backup
docker exec -i mockarty-db psql -U mockarty mockarty < backup_20260327.sql
```

### Volume Backup

```bash
# Backup all volumes
docker run --rm \
  -v mockarty_postgres_data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/postgres_data.tar.gz -C /data .
```

---

## Troubleshooting

### Admin node fails to start

Check database connectivity:

```bash
docker compose logs mockarty
docker compose logs postgres
```

Common causes:
- `POSTGRES_PASSWORD` not set or mismatched
- PostgreSQL not yet healthy when admin starts (the `depends_on` condition handles this, but if using an external DB, verify connectivity)

### Resolver/Runner cannot connect to coordinator

```bash
docker compose logs mockarty-resolver
docker compose logs mockarty-runner
```

Common causes:
- API token not set or invalid (regenerate in admin UI)
- Admin node not healthy yet (wait and retry)
- Network isolation (ensure all services are on the same Docker network)

### Port conflicts

If default ports are in use, override them in `.env`:

```dotenv
HTTP_PORT=8080
HTTPS_PORT=8443
MCP_PORT=8772
POSTGRES_PORT=15432
```

### Viewing logs

```bash
# All services
docker compose -f docker-compose.minimal.yml logs -f

# Specific service
docker compose -f docker-compose.minimal.yml logs -f mockarty

# Last 100 lines
docker compose -f docker-compose.minimal.yml logs --tail=100 mockarty
```

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
