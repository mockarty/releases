<p align="center"><img src="https://raw.githubusercontent.com/mockarty/releases/main/logo.svg" width="200" alt="Mockarty"></p>

<h1 align="center">Mockarty Runner Guide</h1>

---

## Overview

The Mockarty Runner (`mockarty-runner`) is a distributed execution agent that picks up and runs tasks dispatched by the Admin Node coordinator. It enables horizontal scaling of compute-intensive operations:

- **API Tests** — execute collections of API test requests with assertions
- **Performance Tests** — run load tests with configurable VUs, duration, stages, and thresholds
- **Fuzzing** — execute fuzz testing campaigns against target endpoints
- **Chaos Engineering** — run chaos experiments (latency injection, error responses, etc.)

Mockarty Runners connect to the Admin Node coordinator, advertise their capabilities, and receive tasks in real time. Results are streamed back with live progress updates.

## Architecture

```
                        +-----------------+
                        |   Admin Node    |
                        |  (mockarty)     |
                        |                 |
                        | Task Queue,     |
                        | Coordinator,    |
                        | Result Storage  |
                        +--------+--------+
                                 |
                     gRPC stream (port 5773)
                                 |
              +------------------+------------------+
              |                  |                  |
     +--------+-------+ +-------+--------+ +-------+--------+
     | Mockarty Runner 1  | | Mockarty Runner 2  | | Mockarty Runner 3  |
     | capabilities:   | | capabilities:   | | capabilities:   |
     | api_test,       | | performance     | | fuzzing,        |
     | performance     | |                 | | chaos           |
     | (Dashboard :6770)| | (Dashboard :6770)| | (Dashboard :6770)|
     +-----------------+ +----------------+ +----------------+
```

The coordinator maintains a task queue in the database. When a user submits a test run, performance test, fuzz session, or chaos experiment via the Web UI or API, the coordinator assigns the task to an available runner based on capabilities, labels, and namespace assignments.

### Connection Modes

Mockarty Runners support three connection modes:

| Mode | Description | Use Case |
|------|-------------|----------|
| `grpc` | Traditional bidirectional gRPC stream | Standard deployments, same network |
| `poll` | HTTP long-polling from runner to coordinator | NAT/firewall environments, cloud functions |
| `reverse` | Coordinator connects TO the runner | Simplest configuration, runner exposes HTTP |

## Capabilities

Each runner advertises which task types it can execute:

| Capability | Description |
|------------|-------------|
| `api_test` | Execute API test collections with HTTP/gRPC/MCP requests and assertions |
| `performance` | Run performance/load tests with VU scheduling, stages, and thresholds |
| `fuzzing` | Execute fuzz testing campaigns with various strategies |
| `chaos` | Run chaos engineering experiments |

A runner can have one or more capabilities. The coordinator only assigns tasks to runners with matching capabilities.

## Installation

### Binary

```bash
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-runner-linux-amd64
chmod +x mockarty-runner-linux-amd64
mv mockarty-runner-linux-amd64 /usr/local/bin/mockarty-runner
```

### Docker

```bash
docker pull mockarty/runner:latest

docker run -d \
  --name runner \
  -p 6770:6770 \
  -e COORDINATOR_ADDR="admin-node:5773" \
  -e API_TOKEN="mki_your_integration_token" \
  -e CAPABILITIES="api_test,performance,fuzzing,chaos" \
  mockarty/runner:latest
```

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `COORDINATOR_ADDR` | *(required)* | Admin Node coordinator address, e.g. `admin-node:5773` |
| `API_TOKEN` | *(required)* | Integration API token created in the Admin Node |
| `UI_PORT` | `6770` | Dashboard UI port |
| `MAX_CONCURRENT_TASKS` | `3` | Maximum number of tasks to execute simultaneously |
| `CAPABILITIES` | `api_test,performance,fuzzing,chaos` | Comma-separated list of supported task types |
| `CONNECTION_MODE` | `grpc` | Connection mode: `grpc`, `poll`, or `reverse` |
| `LISTEN_ADDR` | *(empty)* | Listen address for reverse mode (e.g. `0.0.0.0:6771`) |
| `RUNNER_NAME` | *(auto-generated)* | Human-readable runner name |
| `LABELS` | *(empty)* | Comma-separated labels for task routing, e.g. `region=us-east,tier=heavy` |
| `LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |

### TLS Configuration

When the coordinator uses TLS, configure the runner accordingly:

| Variable | Default | Description |
|----------|---------|-------------|
| `TLS_ENABLED` | `false` | Enable TLS for coordinator connection |
| `TLS_CERT_FILE` | *(empty)* | Client TLS certificate file |
| `TLS_KEY_FILE` | *(empty)* | Client TLS private key file |
| `TLS_CA_CERT` | *(empty)* | CA certificate for verifying coordinator |
| `TLS_INSECURE_SKIP_VERIFY` | `false` | Skip TLS verification (development only) |

## Token Generation

Before starting a Mockarty Runner, create an integration token in the Admin Node:

1. Open the Admin Node Web UI at `http://admin-node:5770/ui/settings`
2. Navigate to **Integrations**
3. Click **Add Integration**
4. Select type **Runner**
5. Configure namespace assignments (which namespaces this runner can execute tasks for)
6. Copy the generated token (starts with `mki_`)

Use this token as the `API_TOKEN` environment variable.

## Running

### Standalone Binary

```bash
export COORDINATOR_ADDR="admin-node:5773"
export API_TOKEN="mki_your_integration_token"
export CAPABILITIES="api_test,performance"
./mockarty-runner
```

### Docker Compose

```yaml
version: "3.8"
services:
  runner:
    image: mockarty/runner:latest
    ports:
      - "6770:6770"
    environment:
      COORDINATOR_ADDR: "admin-node:5773"
      API_TOKEN: "mki_your_integration_token"
      CAPABILITIES: "api_test,performance,fuzzing,chaos"
      MAX_CONCURRENT_TASKS: "5"
    deploy:
      replicas: 3
```

### Scaling with Docker Compose

```bash
# Scale to 10 runner instances
docker compose up -d --scale runner=10
```

## Dashboard UI

The Mockarty Runner exposes a dashboard at `http://runner-host:6770` showing:

- Connection status to the coordinator
- Runner capabilities and labels
- Active tasks with live progress
- Completed task history
- Resource utilization
- Version

## Task Execution Flow

1. **Submission** — A user submits a task via the Web UI or API on the Admin Node
2. **Queuing** — The coordinator stores the task in the database queue
3. **Assignment** — The coordinator selects an available runner with matching capabilities and namespace access
4. **Dispatch** — The task payload is sent to the runner over the gRPC stream (or via HTTP poll/reverse)
5. **Execution** — The runner executes the task, streaming progress updates back to the coordinator
6. **Completion** — Results are sent back and stored in the database
7. **Notification** — The Web UI receives live SSE updates; optional email/webhook notifications are sent

### Progress Streaming

During execution, the runner sends real-time progress events:

```json
{
  "taskId": "task-abc-123",
  "status": "running",
  "progressPct": 45,
  "message": "Executing request: GET /api/users",
  "totalRequests": 20,
  "completedRequests": 9,
  "passedTests": 8,
  "failedTests": 1
}
```

These events are relayed to the Web UI via SSE for live progress bars and result updates.

## Scaling

### Horizontal Scaling

Mockarty Runners are independent and stateless. Scale them based on:

- **Task volume** — more runners = more concurrent task execution
- **Capabilities** — deploy specialized runners (e.g., heavy performance runners on high-CPU instances)
- **Geography** — deploy runners close to target services for accurate latency measurements

### Sizing Recommendations

| Workload | CPU | RAM | Max Concurrent |
|----------|-----|-----|----------------|
| API tests only | 1 core | 512 MB | 5-10 |
| Performance tests | 4+ cores | 4+ GB | 2-3 |
| Fuzzing | 2+ cores | 2+ GB | 3-5 |
| Mixed | 4+ cores | 4+ GB | 3-5 |

### Label-Based Routing

Use labels to route specific task types to appropriate runners:

```bash
# Heavy runner for performance tests
LABELS="tier=heavy,region=us-east"
CAPABILITIES="performance"
MAX_CONCURRENT_TASKS=2

# Lightweight runner for API tests
LABELS="tier=light,region=us-east"
CAPABILITIES="api_test"
MAX_CONCURRENT_TASKS=10
```

## Monitoring

### Health Check

```
GET http://runner-host:6770/health
```

Returns:

```json
{
  "status": "ok",
  "releaseId": "1.0.0",
  "coordinator": "connected",
  "activeTasks": 2,
  "capabilities": ["api_test", "performance", "fuzzing", "chaos"]
}
```

### Prometheus Metrics

```
GET http://runner-host:6770/metrics
```

Key metrics:

- `mockarty_runner_tasks_total` — total tasks executed (by type and status)
- `mockarty_runner_task_duration_seconds` — task execution duration histogram
- `mockarty_runner_active_tasks` — currently running tasks gauge
- `mockarty_runner_coordinator_connected` — coordinator connection status (1/0)

## Troubleshooting

### Cannot connect to coordinator

```
Error: failed to connect to coordinator at admin-node:5773
```

- Verify the Admin Node is running and the coordinator is enabled (`RUNNER_COORDINATOR_ENABLED=true`)
- Check that port `5773` is reachable from the runner (firewalls, security groups)
- If behind NAT, consider using `CONNECTION_MODE=poll` instead of `grpc`

### Token rejected

```
Error: authentication failed: invalid token
```

- Ensure the token was created as a **Runner** integration in the Admin Node
- Tokens start with `mki_` — do not use regular API tokens (`mk_`)
- The token may have been revoked or the integration deleted

### Tasks not being assigned

- Verify the runner's `CAPABILITIES` include the task type being submitted
- Check that the runner is assigned to the correct namespace in the Admin Node
- Review `MAX_CONCURRENT_TASKS` — if all slots are full, tasks wait in the queue
- Set `LOG_LEVEL=debug` to see assignment decisions

### Performance test results are inaccurate

- Deploy the runner close to the target service (same region/network)
- Do not overload the runner — use `MAX_CONCURRENT_TASKS=1` for performance tests
- Ensure the runner has sufficient CPU and RAM (see sizing recommendations above)
- Check for network throttling or container resource limits

### Runner keeps disconnecting

- Check `RUNNER_HEARTBEAT_TIMEOUT` on the Admin Node (default `30s`)
- Ensure network is stable between runner and coordinator
- Increase heartbeat timeout if running over high-latency links
- Check runner logs for gRPC stream errors

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
