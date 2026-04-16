<p align="center"><img src="https://raw.githubusercontent.com/mockarty/releases/main/logo.svg" width="200" alt="Mockarty"></p>

<h1 align="center">Admin Node Guide</h1>

---

## Overview

The Admin Node (`mockarty`) is the central management server of the Mockarty platform. It provides:

- **Web UI** for managing mocks, importing OpenAPI specs, running API tests, chatting with an AI agent, and administering users
- **REST API** (`/api/v1/`) for programmatic mock management
- **MCP Server** (Model Context Protocol) on a dedicated port for AI tool integration
- **Coordinator** that registers and dispatches tasks to Mock Resolver and Runner Agent nodes via gRPC
- **Database** (PostgreSQL) as the single source of truth for all mocks, users, stores, and configuration
- **HTTPS/TLS** with automatic self-signed certificate generation or custom certificates
- **Authentication** with API tokens, OAuth 2.0 (Google, Yandex, VK), LDAP/AD, SAML SSO, and MFA
- **RBAC** with Casbin-based role and permission management
- **Backup and Restore** with scheduled and on-demand database backups

## System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| **PostgreSQL** | 14+ | 16+ |
| **RAM** | 512 MB | 2 GB+ |
| **Disk** | 100 MB (binary) | Depends on data volume |
| **CPU** | 1 core | 2+ cores |

### Default Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| `5770` | HTTP | Web UI, REST API, mock traffic |
| `5771` | HTTPS | TLS-encrypted Web UI, REST API, mock traffic |
| `5772` | HTTP (SSE) | MCP (Model Context Protocol) server |
| `5773` | gRPC | Coordinator for resolver/runner registration |

## Installation

### Binary

```bash
# Download for your platform
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-linux-amd64
chmod +x mockarty-linux-amd64
mv mockarty-linux-amd64 /usr/local/bin/mockarty
```

### Docker

```bash
docker pull mockarty/mockarty:latest

docker run -d \
  --name mockarty \
  -p 5770:5770 \
  -p 5771:5771 \
  -p 5772:5772 \
  -p 5773:5773 \
  -e DB_DSN="postgres://user:pass@host:5432/mockarty?sslmode=disable" \
  -e HTTP_BIND_ADDR="0.0.0.0" \
  mockarty/mockarty:latest
```

### Helm

```bash
helm repo add mockarty https://charts.mockarty.ru
helm repo update
helm install mockarty mockarty/mockarty \
  --set db.dsn="postgres://user:pass@host:5432/mockarty?sslmode=disable"
```

## Configuration

All configuration is done via environment variables. Create a `.env` file next to the binary or pass them directly.

### Core

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_DSN` | *(required)* | PostgreSQL connection string, e.g. `postgres://user:pass@localhost:5432/mockarty?sslmode=disable` |
| `DB_USE` | `pg` | Database engine: `pg`, `mysql`, `sqlite` |
| `DB_ENABLE_AUDIT` | `true` | Enable audit logging for all changes |
| `DB_ENABLE_VERSIONING` | `true` | Enable mock version history |
| `ENV` | `development` | Environment name (`development`, `staging`, `production`) |
| `LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |

### HTTP Server

| Variable | Default | Description |
|----------|---------|-------------|
| `HTTP_PORT` | `5770` | HTTP listen port |
| `HTTP_BIND_ADDR` | `localhost` | HTTP bind address. Set to `0.0.0.0` for Docker/remote access |
| `HTTPS_ENABLED` | `true` | Enable HTTPS server |
| `HTTPS_PORT` | `5771` | HTTPS listen port |
| `HTTPS_BIND_ADDR` | *(empty)* | HTTPS bind address (defaults to HTTP_BIND_ADDR) |
| `HTTPS_CERT_FILE` | *(empty)* | Path to TLS certificate file (auto-generated if empty) |
| `HTTPS_KEY_FILE` | *(empty)* | Path to TLS private key file (auto-generated if empty) |
| `HTTPS_TLS_DIR` | `.mockarty/tls` | Directory for auto-generated TLS certificates |
| `HTTP_KEEP_ALIVE_TIME` | `30s` | HTTP keep-alive time |
| `HTTP_KEEP_ALIVE_TIMEOUT` | `30s` | HTTP keep-alive timeout |
| `HTTP_REQUEST_BODY_SIZE_LIMIT_MB` | `5` | Maximum request body size in MB |

### MCP Server

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_MCP` | `false` | Enable the MCP (Model Context Protocol) server |
| `MCP_PORT` | `5772` | MCP server listen port |

### Coordinator (gRPC)

| Variable | Default | Description |
|----------|---------|-------------|
| `RUNNER_COORDINATOR_ENABLED` | `true` | Enable the coordinator for resolver/runner registration |
| `RUNNER_GRPC_PORT` | `5773` | Coordinator gRPC listen port |
| `RUNNER_GRPC_BIND_ADDR` | `0.0.0.0` | Coordinator gRPC bind address |
| `RUNNER_HEARTBEAT_TIMEOUT` | `30s` | Heartbeat timeout before marking a runner/resolver offline |
| `RUNNER_TASK_TIMEOUT` | `30m` | Maximum task execution time |
| `RUNNER_GRPC_TLS_ENABLED` | `false` | Enable TLS for coordinator gRPC |
| `RUNNER_GRPC_TLS_CERT_FILE` | *(empty)* | gRPC TLS certificate file |
| `RUNNER_GRPC_TLS_KEY_FILE` | *(empty)* | gRPC TLS private key file |

### Caching

| Variable | Default | Description |
|----------|---------|-------------|
| `USE_CACHE` | `true` | Enable caching layer for performance |
| `CACHE_TYPE` | `inmemory` | Cache backend: `inmemory` or `redis` |
| `REPO_INMEMORY_MAX_SIZE_MB` | `200` | Maximum in-memory cache size in MB |
| `REPO_REDIS_HOST` | *(empty)* | Redis host (required when `CACHE_TYPE=redis`) |
| `REPO_REDIS_PORT` | *(empty)* | Redis port (required when `CACHE_TYPE=redis`) |
| `REPO_REDIS_PASSWORD` | *(empty)* | Redis password |
| `REPO_LOG_LIMIT` | `3` | Number of recent request logs to keep per mock |

### Authentication

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTH_MODE` | `full` | Auth mode: `full` (multi-user) or `local` (single-user, no login) |
| `AUTH_SESSION_EXPIRATION` | `24` | Session expiration in hours |
| `AUTH_BASE_URL` | `http://localhost:5770` | Base URL for OAuth redirect URIs |
| `FIRST_ADMIN_LOGIN` | `admin` | First admin username (created on initial startup) |
| `FIRST_ADMIN_PASSWORD` | `admin` | First admin password |
| `FIRST_ADMIN_EMAIL` | *(empty)* | First admin email |

#### OAuth 2.0

| Variable | Default | Description |
|----------|---------|-------------|
| `OAUTH_GOOGLE_ENABLED` | `false` | Enable Google OAuth |
| `OAUTH_GOOGLE_CLIENT_ID` | *(empty)* | Google OAuth client ID |
| `OAUTH_GOOGLE_SECRET` | *(empty)* | Google OAuth client secret |
| `OAUTH_YANDEX_ENABLED` | `false` | Enable Yandex OAuth |
| `OAUTH_YANDEX_CLIENT_ID` | *(empty)* | Yandex OAuth client ID |
| `OAUTH_YANDEX_SECRET` | *(empty)* | Yandex OAuth client secret |
| `OAUTH_VK_ENABLED` | `false` | Enable VK OAuth |
| `OAUTH_VK_CLIENT_ID` | *(empty)* | VK OAuth client ID |
| `OAUTH_VK_SECRET` | *(empty)* | VK OAuth client secret |

#### LDAP / Active Directory

| Variable | Default | Description |
|----------|---------|-------------|
| `LDAP_ENABLED` | `false` | Enable LDAP authentication |
| `LDAP_URL` | *(empty)* | LDAP server URL, e.g. `ldaps://ldap.company.com:636` |
| `LDAP_BASE_DN` | *(empty)* | Base DN for user search |
| `LDAP_BIND_DN` | *(empty)* | Bind DN for LDAP connection |
| `LDAP_BIND_PASSWORD` | *(empty)* | Bind password |
| `LDAP_USER_FILTER` | *(empty)* | User search filter, e.g. `(sAMAccountName=%s)` |
| `LDAP_USER_ATTR` | `sAMAccountName` | Username attribute |
| `LDAP_EMAIL_ATTR` | `mail` | Email attribute |
| `LDAP_NAME_ATTR` | `displayName` | Display name attribute |
| `LDAP_TLS_INSECURE` | `false` | Skip TLS certificate verification |

### Security

| Variable | Default | Description |
|----------|---------|-------------|
| `COOKIE_SECURE` | `true` | Set Secure flag on session cookies (disable for local HTTP) |
| `MCP_TLS_INSECURE_SKIP_VERIFY` | `false` | Skip TLS verification for outbound MCP connections |
| `ALLOW_PROXY_TO_PRIVATE_IPS` | `false` | Allow proxy mode to forward to private/internal IPs |

### LLM Integration

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_LLM` | `false` | Enable LLM-powered AI agent |
| `OLLAMA_URL` | `http://localhost:11434` | Ollama server URL |
| `OLLAMA_MODEL` | `qwen3:1.7b` | Ollama model name |

### Cluster Mode

| Variable | Default | Description |
|----------|---------|-------------|
| `CLUSTER_MODE` | `false` | Enable leader election via PostgreSQL advisory locks |
| `NODE_ID` | *(auto-generated)* | Unique node ID for cluster membership |

### Licensing

| Variable | Default | Description |
|----------|---------|-------------|
| `LICENSE_SERVER_URL` | *(empty)* | Comma-separated license server URLs (empty = offline mode) |
| `LICENSE_HEARTBEAT_INTERVAL` | `3600` | License heartbeat interval in seconds |
| `LICENSE_AIR_GAPPED` | `false` | Air-gapped mode: never contact license servers |

### Email Notifications

| Variable | Default | Description |
|----------|---------|-------------|
| `EMAIL_ENABLED` | `false` | Enable outbound email notifications |
| `SMTP_HOST` | *(empty)* | SMTP server hostname |
| `SMTP_PORT` | `587` | SMTP server port |
| `SMTP_USERNAME` | *(empty)* | SMTP username |
| `SMTP_PASSWORD` | *(empty)* | SMTP password |
| `SMTP_USE_IMPLICIT_TLS` | `false` | Use implicit TLS (port 465) |
| `EMAIL_FROM` | `no-reply@mockarty.ru` | Sender email address |

### Kubernetes

| Variable | Default | Description |
|----------|---------|-------------|
| `K8S_MANAGEMENT` | `auto` | Kubernetes management: `auto`, `true`, `false` |
| `K8S_NAMESPACE` | *(auto-detected)* | Kubernetes namespace |
| `K8S_CLUSTER_NAME` | `mockarty` | MockartyCluster CR name |
| `K8S_KUBECONFIG` | *(empty)* | Path to kubeconfig (empty = in-cluster) |

## First Run

### 1. Run Database Migrations

Migrations are built into the binary and run automatically on startup. To run them manually:

```bash
# Apply all pending migrations
mockarty --migrate up

# Check current version
mockarty --migrate version

# Dry run (print SQL without executing)
mockarty --migrate up --migrate-dry-run
```

### 2. Start the Server

```bash
# Minimal startup
export DB_DSN="postgres://mockarty:mockarty@localhost:5432/mockarty?sslmode=disable"
./mockarty

# With explicit bind to all interfaces
export HTTP_BIND_ADDR="0.0.0.0"
./mockarty
```

### 3. Access the Web UI

Open your browser and navigate to:

- **HTTP**: `http://localhost:5770/ui/`
- **HTTPS**: `https://localhost:5771/ui/`

Login with the default credentials:

- **Username**: `admin`
- **Password**: `admin`

> Change the default password immediately after first login.

## Web UI Overview

| Page | URL | Description |
|------|-----|-------------|
| **Mocks** | `/ui/mocks` | Create, edit, and manage mocks for all protocols |
| **OpenAPI Import** | `/ui/openapi` | Import OpenAPI/Swagger specs to generate mocks |
| **API Tester** | `/ui/api-tester` | Postman-like HTTP/gRPC/MCP client for testing |
| **Agent Chat** | `/ui/agent-chat` | AI assistant for generating and managing mocks |
| **Test Runs** | `/ui/test-runs` | View API test execution results |
| **Performance** | `/ui/performance` | Performance test configuration and results |
| **Fuzzing** | `/ui/fuzzing` | Fuzz testing configuration and results |
| **Chaos** | `/ui/chaos` | Chaos engineering experiments |
| **Contract Testing** | `/ui/contract-testing` | Consumer-driven contract tests |
| **Settings** | `/ui/settings` | Users, roles, namespaces, integrations, backup |
| **Recorder** | `/ui/recorder` | Record live traffic and convert to mocks |

## REST API

The full REST API is available at `/api/v1/`. Interactive Swagger documentation is served at `/swagger/`.

### Authentication

All API requests require an API token in the `Authorization` header:

```bash
curl -H "Authorization: Bearer mk_your_api_token_here" \
  http://localhost:5770/api/v1/mocks
```

API tokens are created in the Web UI under **Settings > API Tokens**.

### Quick Examples

```bash
# List all mocks
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:5770/api/v1/mocks

# Create a mock
curl -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:5770/api/v1/mocks \
  -d '{
    "id": "get-user",
    "http": {
      "route": "/api/users/:id",
      "httpMethod": "GET"
    },
    "response": {
      "statusCode": 200,
      "payload": {
        "id": "$.pathParam.id",
        "name": "$.fake.FirstName",
        "email": "$.fake.Email"
      }
    }
  }'

# Delete a mock
curl -X DELETE -H "Authorization: Bearer $TOKEN" \
  http://localhost:5770/api/v1/mocks/get-user
```

## MCP Server

When `ENABLE_MCP=true`, Mockarty exposes an MCP (Model Context Protocol) server on port `5772`. This allows AI tools and agents (Claude, Cursor, etc.) to interact with your mocks programmatically.

```bash
# Enable MCP in .env
ENABLE_MCP=true
MCP_PORT=5772
```

The MCP server exposes tools and resources for listing, creating, and managing mocks.

## HTTPS / TLS Configuration

HTTPS is enabled by default (`HTTPS_ENABLED=true`). On first startup, Mockarty generates a self-signed certificate in the `HTTPS_TLS_DIR` directory.

### Using Custom Certificates

```bash
HTTPS_CERT_FILE=/path/to/cert.pem
HTTPS_KEY_FILE=/path/to/key.pem
```

### Disabling HTTPS

```bash
HTTPS_ENABLED=false
```

## Backup and Restore

Backups are managed in the Web UI under **Settings > Backup**. You can:

- Create on-demand backups (full database export)
- Schedule automatic backups with cron expressions
- Download backup archives
- Restore from a backup file

Backups include all mocks, stores, users, roles, namespaces, and configuration.

## Monitoring

### Health Check

```
GET /health
```

Returns JSON with component status and the release version:

```json
{
  "status": "ok",
  "releaseId": "1.0.0",
  "database": "ok",
  "cache": "ok"
}
```

### Prometheus Metrics

```
GET /metrics
```

Exposes Prometheus-compatible metrics for request counts, latencies, mock resolution times, cache hit rates, and more.

## Upgrading

1. Download the new binary or pull the new Docker image
2. Stop the running instance
3. Run migrations (automatic on startup, or manual with `--migrate up`)
4. Start the new version
5. Verify via `/health` endpoint

```bash
# Check current version
curl -s http://localhost:5770/health | jq .releaseId

# Upgrade
docker pull mockarty/mockarty:1.1.0
docker stop mockarty
docker rm mockarty
docker run -d --name mockarty ... mockarty/mockarty:1.1.0
```

## Troubleshooting

### Cannot connect to database

```
Error: database is required. Please set DB_DSN
```

Ensure `DB_DSN` is set and the PostgreSQL server is reachable. Test connectivity:

```bash
psql "postgres://user:pass@host:5432/mockarty?sslmode=disable"
```

### Port already in use

```
Error: listen tcp :5770: bind: address already in use
```

Another process is using the port. Either stop it or change `HTTP_PORT`:

```bash
lsof -i :5770
HTTP_PORT=5780 ./mockarty
```

### Web UI not accessible from remote machines

By default, the HTTP server binds to `localhost`. Set `HTTP_BIND_ADDR=0.0.0.0` to listen on all interfaces.

### HTTPS certificate errors in browser

On first run, Mockarty generates a self-signed certificate. Browsers will show a warning. Either:

- Accept the self-signed certificate in the browser
- Provide custom trusted certificates via `HTTPS_CERT_FILE` and `HTTPS_KEY_FILE`
- Disable HTTPS with `HTTPS_ENABLED=false` (development only)

### License errors (403 Forbidden)

If you see 403 errors related to licensing, the binary may have been built with an incorrect public key. Rebuild using the official build script or download a pre-built binary from the releases page.

### Migrations fail with "dirty" state

```bash
# Check current state
mockarty --migrate version

# Force a specific version to fix dirty state
mockarty --migrate force <version_number>

# Then retry
mockarty --migrate up
```

### Slow mock resolution

- Enable caching: `USE_CACHE=true` (enabled by default)
- For distributed deployments, use Redis: `CACHE_TYPE=redis`
- Consider deploying Mock Resolver nodes for horizontal scaling

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
