<p align="center"><img src="https://raw.githubusercontent.com/mockarty/releases/main/logo.svg" width="200" alt="Mockarty"></p>

<h1 align="center">Mock Resolver Guide</h1>

---

## Overview

The Mock Resolver (`mockarty-resolver`) is a lightweight, stateless node that serves mock HTTP traffic. It connects to the Admin Node's coordinator via gRPC to register itself and synchronize mocks, then serves incoming requests independently with high throughput.

Use Mock Resolvers when you need to:

- **Scale mock traffic horizontally** without adding load to the Admin Node
- **Isolate mock serving** from management and administration
- **Deploy close to consumers** in different regions or availability zones
- **Handle high-volume load testing** where a single node is insufficient

## Architecture

```
                        +-----------------+
                        |   Admin Node    |
                        |  (mockarty)     |
                        |                 |
                        | Web UI, API,    |
                        | Database (PG),  |
                        | Coordinator     |
                        +--------+--------+
                                 |
                          gRPC (port 5773)
                                 |
              +------------------+------------------+
              |                  |                  |
     +--------+-------+ +-------+--------+ +-------+--------+
     | Mock Resolver 1 | | Mock Resolver 2 | | Mock Resolver 3 |
     | (HTTP :5780)    | | (HTTP :5780)    | | (HTTP :5780)    |
     | (Dashboard :6780)| | (Dashboard :6780)| | (Dashboard :6780)|
     +-----------------+ +----------------+ +----------------+
              |                  |                  |
     =========================================================
              Load Balancer (Traefik / nginx / cloud LB)
     =========================================================
              |
        Mock Consumers
     (tests, services, CI/CD)
```

Mock Resolvers register with the Admin Node coordinator on startup. They receive mock definitions, store state, and configuration updates over the gRPC stream. All mock traffic is served locally without hitting the Admin Node.

## Installation

### Binary

```bash
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-resolver-linux-amd64
chmod +x mockarty-resolver-linux-amd64
mv mockarty-resolver-linux-amd64 /usr/local/bin/mockarty-resolver
```

### Docker

```bash
docker pull mockarty/resolver:latest

docker run -d \
  --name resolver \
  -p 5780:5780 \
  -p 6780:6780 \
  -e COORDINATOR_ADDR="mockarty:5773" \
  -e API_TOKEN="mki_your_integration_token" \
  mockarty/resolver:latest
```

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `COORDINATOR_ADDR` | *(required)* | Admin Node coordinator address, e.g. `mockarty:5773` |
| `API_TOKEN` | *(required)* | Integration API token created in the Admin Node |
| `DB_DSN` | *(optional)* | Direct database connection for read-through (bypasses gRPC sync) |
| `HTTP_PORT` | `5780` | HTTP port for mock traffic |
| `HTTP_BIND_ADDR` | `0.0.0.0` | HTTP bind address |
| `UI_PORT` | `6780` | Dashboard UI port |
| `STORE_MODE` | `read_write` | Store access mode: `read_only` or `read_write` |
| `CACHE_SYNC_INTERVAL` | `30s` | Interval for syncing mock cache from the coordinator |
| `LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `USE_CACHE` | `true` | Enable local cache |
| `CACHE_TYPE` | `inmemory` | Cache backend: `inmemory` or `redis` |

## Token Generation

Before starting a Mock Resolver, you must create an integration token in the Admin Node:

1. Open the Admin Node Web UI at `http://mockarty:5770/ui/settings`
2. Navigate to **Integrations**
3. Click **Add Integration**
4. Select type **Resolver**
5. Copy the generated token (starts with `mki_`)

Use this token as the `API_TOKEN` environment variable.

## Running

### Standalone Binary

```bash
export COORDINATOR_ADDR="mockarty:5773"
export API_TOKEN="mki_your_integration_token"
./mockarty-resolver
```

### Docker Compose

```yaml
version: "3.8"
services:
  resolver:
    image: mockarty/resolver:latest
    ports:
      - "5780:5780"
      - "6780:6780"
    environment:
      COORDINATOR_ADDR: "mockarty:5773"
      API_TOKEN: "mki_your_integration_token"
    deploy:
      replicas: 3
```

### Scaling with Docker Compose

```bash
# Scale to 5 resolver instances
docker compose up -d --scale resolver=5
```

When scaling, use a load balancer in front of the resolver instances. Each resolver registers independently with the coordinator.

## Dashboard UI

The Mock Resolver exposes a dashboard at `http://resolver-host:6780` showing:

- Connection status to the coordinator
- Number of loaded mocks
- Request statistics (total, per-second, latency percentiles)
- Store state
- Health information
- Version

## Health Check

```
GET http://resolver-host:5780/health
```

Returns:

```json
{
  "status": "ok",
  "releaseId": "1.0.0",
  "coordinator": "connected",
  "mocks": 42
}
```

## Scaling

### Horizontal Scaling

Mock Resolvers are stateless and designed for horizontal scaling. Each instance:

1. Connects to the coordinator on startup
2. Receives the full mock set
3. Receives incremental updates (create, update, delete) in real time
4. Serves traffic independently

There is no coordination between resolver instances. You can run as many as needed.

### Load Balancing

Place a load balancer in front of resolver instances:

**Traefik (Docker labels):**

```yaml
services:
  resolver:
    image: mockarty/resolver:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.resolver.rule=Host(`mocks.example.com`)"
      - "traefik.http.services.resolver.loadbalancer.server.port=5780"
    deploy:
      replicas: 3
```

**nginx:**

```nginx
upstream resolvers {
    server resolver-1:5780;
    server resolver-2:5780;
    server resolver-3:5780;
}

server {
    listen 80;
    server_name mocks.example.com;

    location / {
        proxy_pass http://resolvers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Store Modes

| Mode | Description |
|------|-------------|
| `read_write` | Resolver can read and write to Global and Chain stores. Changes are propagated back to the coordinator. |
| `read_only` | Resolver can only read store values. Writes are rejected. Use this for scenarios where resolvers should not mutate state. |

## Monitoring

### Prometheus Metrics

```
GET http://resolver-host:5780/metrics
```

Key metrics:

- `mockarty_resolver_requests_total` — total requests served
- `mockarty_resolver_request_duration_seconds` — request latency histogram
- `mockarty_resolver_mocks_loaded` — number of mocks currently loaded
- `mockarty_resolver_cache_hits_total` — cache hit counter
- `mockarty_resolver_coordinator_connected` — coordinator connection status (1/0)

## Troubleshooting

### Cannot connect to coordinator

```
Error: failed to connect to coordinator at mockarty:5773
```

- Verify the Admin Node is running and the coordinator is enabled (`RUNNER_COORDINATOR_ENABLED=true`)
- Check that port `5773` is reachable from the resolver (firewalls, security groups)
- If using TLS on the coordinator, configure the resolver's TLS settings accordingly

### Token rejected

```
Error: authentication failed: invalid token
```

- Ensure the token was created as a **Resolver** integration in the Admin Node
- Tokens start with `mki_` — do not use regular API tokens (`mk_`)
- The token may have been revoked or the integration deleted

### Mocks not loading

- Check the dashboard at `http://resolver-host:6780` for connection status
- Set `LOG_LEVEL=debug` and look for sync errors
- Verify the Admin Node has mocks in the namespaces the resolver is assigned to
- Check `CACHE_SYNC_INTERVAL` — the first full sync happens on startup

### High memory usage

- Reduce `REPO_INMEMORY_MAX_SIZE_MB` if using in-memory cache
- Consider using Redis (`CACHE_TYPE=redis`) for large mock sets
- Monitor the number of mocks — each mock with large response payloads adds to memory

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
