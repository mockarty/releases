<p align="center">
  <img src="logo.svg" width="200" alt="Mockarty">
</p>

<h1 align="center">Mockarty Platform Releases</h1>

<p align="center">
  <i>Official release binaries for all <a href="https://mockarty.ru">Mockarty</a> platform components</i>
</p>

<p align="center">
  <a href="https://mockarty.ru">Website</a> ·
  <a href="https://mockarty.ru/docs">Docs</a> ·
  <a href="https://github.com/mockarty">GitHub</a>
</p>

---

[Mockarty](https://mockarty.ru) is a powerful mock server platform for HTTP, gRPC, MCP, GraphQL, SOAP, SSE, WebSocket, Kafka, RabbitMQ, and SMTP. It provides dynamic response generation with Faker and JsonPath, sophisticated condition matching, stateful stores, proxy mode with delay injection, AI assistant integration, and distributed test execution with horizontal scaling.

## Components

| Component | Binary | Description | Docker Image |
|-----------|--------|-------------|--------------|
| **Admin Node** | `mockarty` | Central management server: Web UI, REST API, database, MCP server, coordinator for resolvers and runners | `mockarty/mockarty` |
| **Mock Resolver** | `mockarty-resolver` | Lightweight mock-serving node for horizontal scaling of HTTP traffic | `mockarty/mockarty-resolver` |
| **Runner** | `mockarty-runner` | Distributed execution agent for API tests, performance tests, fuzzing, and chaos engineering | `mockarty/mockarty-runner` |
| **Server Generator** | `mockarty-server-generator` | Generates standalone MCP/gRPC servers that call back to Mockarty for mock data | `mockarty/generator` |
| **CLI** | `mockarty-cli` | Command-line tool for mocking, testing, fuzzing, performance, chaos, and contract testing | `mockarty/cli` |

## Download

### Latest Release

Download the latest version from the [Releases](https://github.com/mockarty/releases/releases/latest) page.

### Platform Matrix

Each server component is built for the following platforms:

| OS | Architecture | Suffix |
|----|-------------|--------|
| Linux | amd64 (x86_64) | `-linux-amd64` |
| Linux | arm64 (aarch64) | `-linux-arm64` |
| macOS | amd64 (Intel) | `-darwin-amd64` |
| macOS | arm64 (Apple Silicon) | `-darwin-arm64` |
| Windows | amd64 | `-windows-amd64.exe` |

### Download URLs

```
https://github.com/mockarty/releases/releases/download/v{VERSION}/{binary}-{os}-{arch}
```

### Quick Install

```bash
# Admin Node — Linux amd64
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-linux-amd64
chmod +x mockarty-linux-amd64

# Admin Node — macOS ARM (Apple Silicon)
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-darwin-arm64
chmod +x mockarty-darwin-arm64

# Mock Resolver — Linux amd64
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-resolver-linux-amd64
chmod +x mockarty-resolver-linux-amd64

# Runner — Linux amd64
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-runner-linux-amd64
chmod +x mockarty-runner-linux-amd64

# Server Generator — Linux amd64
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-server-generator-linux-amd64
chmod +x mockarty-server-generator-linux-amd64

# CLI — macOS ARM (CLI has its own repo)
curl -LO https://github.com/mockarty/mockarty-cli/releases/download/v1.0.0/mockarty-cli-darwin-arm64
chmod +x mockarty-cli-darwin-arm64
```

### CLI One-liner Install

```bash
curl -sSL https://mockarty.ru/install.sh | sh
```

Or via Homebrew:

```bash
brew install mockarty/tap/mockarty-cli
```

## Docker Images

```bash
# Admin Node
docker pull mockarty/mockarty:latest
docker pull mockarty/mockarty:1.0.0

# Mock Resolver
docker pull mockarty/resolver:latest
docker pull mockarty/resolver:1.0.0

# Runner Agent
docker pull mockarty/runner:latest
docker pull mockarty/runner:1.0.0

# Server Generator
docker pull mockarty/generator:latest
docker pull mockarty/generator:1.0.0

# CLI
docker pull mockarty/cli:latest
docker pull mockarty/cli:1.0.0
```

### Minimal Docker Compose

```yaml
version: "3.8"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mockarty
      POSTGRES_USER: mockarty
      POSTGRES_PASSWORD: mockarty
    volumes:
      - pgdata:/var/lib/postgresql/data

  mockarty:
    image: mockarty/mockarty:latest
    ports:
      - "5770:5770"   # HTTP
      - "5771:5771"   # HTTPS
      - "5772:5772"   # MCP
      - "5773:5773"   # Coordinator gRPC
    environment:
      DB_DSN: "postgres://mockarty:mockarty@postgres:5432/mockarty?sslmode=disable"
      HTTP_BIND_ADDR: "0.0.0.0"
    depends_on:
      - postgres

volumes:
  pgdata:
```

## Component Guides

| Guide | Description |
|-------|-------------|
| [Admin Node Guide](docs/ADMIN_GUIDE.md) | Central management server setup, configuration, Web UI, API, and administration |
| [Mock Resolver Guide](docs/RESOLVER_GUIDE.md) | Lightweight mock-serving nodes for horizontal scaling |
| [Runner Agent Guide](docs/RUNNER_GUIDE.md) | Distributed execution agents for tests, fuzzing, performance, and chaos |
| [Orchestrator Guide](docs/ORCHESTRATOR_GUIDE.md) | Standalone MCP/gRPC server generation |
| [Docker Compose Deployment](deploy/DOCKER_COMPOSE.md) | Full platform deployment with Docker Compose |
| [Kubernetes Deployment](deploy/KUBERNETES.md) | Helm chart and Kubernetes operator deployment |

## Verification

All release archives include SHA256 checksums in the `checksums.txt` file attached to each GitHub release.

```bash
# Download checksums
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/checksums.txt

# Verify a binary
sha256sum -c checksums.txt --ignore-missing

# Or verify a single file manually
sha256sum mockarty-linux-amd64
# Compare the output with the corresponding line in checksums.txt
```

On macOS, use `shasum -a 256` instead of `sha256sum`.

## Links

- [Documentation](https://mockarty.ru/docs)
- [CLI Repository](https://github.com/mockarty/mockarty-cli)
- [Go SDK](https://github.com/mockarty/mockarty-go)
- [Homebrew Tap](https://github.com/mockarty/homebrew-tap)
- [Docker Hub](https://hub.docker.com/u/mockarty)
- [Website](https://mockarty.ru)

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
