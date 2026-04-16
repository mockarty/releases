<p align="center"><img src="https://raw.githubusercontent.com/mockarty/releases/main/logo.svg" width="200" alt="Mockarty"></p>

<h1 align="center">Server Generator / Orchestrator Guide</h1>

---

## Overview

The Server Generator (`mockarty-server-generator`) is a code generation tool that produces standalone MCP and gRPC servers. These generated servers call back to a Mockarty Admin Node (or Resolver) to fetch mock responses at runtime, acting as protocol-native facades for your mock data.

Use cases:

- **MCP Servers** — generate an MCP server from a JSON config or OpenAPI/Swagger spec so AI tools (Claude, Cursor, etc.) can call your mocks via the Model Context Protocol
- **gRPC Servers** — generate a gRPC server from `.proto` files so gRPC clients can call your mocks using native protobuf messages
- **Proxy Mode** — generated servers can proxy requests to real backends, with Mockarty intercepting and optionally modifying responses

## Installation

### Binary

```bash
curl -LO https://github.com/mockarty/releases/releases/download/v1.0.0/mockarty-server-generator-linux-amd64
chmod +x mockarty-server-generator-linux-amd64
mv mockarty-server-generator-linux-amd64 /usr/local/bin/mockarty-server-generator
```

### Docker

```bash
docker pull mockarty/generator:latest

docker run --rm -v $(pwd):/workspace \
  mockarty/generator:latest \
  --input /workspace/openapi.yaml \
  --output /workspace/generated-server \
  --type mcp
```

### Build from Source

```bash
cd cmd/server-generator
go build -o mockarty-server-generator
```

## MCP Server Generation

### From JSON Config

Create a JSON configuration file describing the MCP tools, resources, and prompts:

```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "mockartyUrl": "http://localhost:5770",
  "namespace": "default",
  "tools": [
    {
      "name": "get_user",
      "description": "Retrieve a user by ID",
      "inputSchema": {
        "type": "object",
        "properties": {
          "userId": {
            "type": "string",
            "description": "The user ID"
          }
        },
        "required": ["userId"]
      }
    },
    {
      "name": "create_order",
      "description": "Create a new order",
      "inputSchema": {
        "type": "object",
        "properties": {
          "product": { "type": "string" },
          "quantity": { "type": "integer" }
        },
        "required": ["product", "quantity"]
      }
    }
  ],
  "resources": [
    {
      "uri": "config://settings",
      "name": "Application Settings",
      "description": "Current application configuration"
    }
  ]
}
```

Generate:

```bash
mockarty-server-generator \
  --input mcp-config.json \
  --output ./generated-mcp-server \
  --type mcp
```

Run the generated server:

```bash
cd generated-mcp-server
go run .
```

The generated server will start and listen for MCP connections. When a tool is called, it forwards the request to the Mockarty Admin Node and returns the mock response.

### From OpenAPI / Swagger

Generate an MCP server from an existing OpenAPI specification:

```bash
mockarty-server-generator \
  --input openapi.yaml \
  --output ./generated-mcp-server \
  --type mcp
```

The generator converts OpenAPI operations into MCP tools:

- Each API operation becomes an MCP tool
- Request parameters and body schema become the tool's `inputSchema`
- Operation descriptions and summaries are preserved

## gRPC Server Generation

### Proto File Requirements

Upload your `.proto` files as-is. `option go_package` is **not required** — if present, any format is accepted and normalized automatically. Language-specific options (`java_package`, `php_namespace`, `csharp_namespace`, etc.) are stripped automatically. Mockarty parses proto files via `protoreflect`, so you do **not** need to install `protoc`.

A minimal example:

```protobuf
syntax = "proto3";

package myservice;

service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string user_id = 1;
  string name = 2;
  string email = 3;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message ListUsersResponse {
  repeated GetUserResponse users = 1;
  string next_page_token = 2;
}
```

> `option go_package` is optional. When present, Mockarty accepts any form (`acme/svc`, `example.com/acme/svc;alias`, `/acme/svc`) and normalizes it automatically. When absent, Mockarty synthesizes a value from the `package` declaration and file path.

### Generate

```bash
mockarty-server-generator \
  --input ./protos/ \
  --output ./generated-grpc-server \
  --type grpc
```

If you have a single `.proto` file:

```bash
mockarty-server-generator \
  --input service.proto \
  --output ./generated-grpc-server \
  --type grpc
```

### Run the Generated gRPC Server

```bash
cd generated-grpc-server
go run . --mockarty-url http://localhost:5770 --port 50051
```

The generated server registers all services and methods from the proto files. When a gRPC client calls a method, the server:

1. Serializes the request to JSON
2. Sends it to the Mockarty Admin Node via HTTP
3. Receives the mock response
4. Deserializes it into the protobuf response message
5. Returns it to the gRPC client

### Multiple Proto Files

Place all `.proto` files in a directory and point `--input` to it:

```bash
protos/
  user.proto
  order.proto
  payment.proto

mockarty-server-generator \
  --input ./protos/ \
  --output ./generated-grpc-server \
  --type grpc
```

All services from all proto files will be included in the generated server.

## Configuration

### Generator CLI Flags

| Flag | Description |
|------|-------------|
| `--input` | Path to the input file or directory (JSON config, OpenAPI spec, or `.proto` files) |
| `--output` | Output directory for the generated server code |
| `--type` | Generation type: `mcp` or `grpc` |
| `--mockarty-url` | Mockarty Admin Node URL (default: `http://localhost:5770`) |
| `--namespace` | Mockarty namespace to use (default: `default`) |
| `--port` | Listen port for the generated server |

### Generated Server Environment Variables

The generated servers read configuration from environment variables at runtime:

| Variable | Description |
|----------|-------------|
| `MOCKARTY_URL` | Mockarty Admin Node or Resolver URL |
| `MOCKARTY_API_TOKEN` | API token for authenticating with Mockarty |
| `MOCKARTY_NAMESPACE` | Namespace for mock resolution |
| `PORT` | Listen port for the generated server |
| `LOG_LEVEL` | Logging level |

## Proxy Mode

Generated servers can be configured to proxy requests to real backends. When proxy mode is enabled:

1. The generated server receives a client request
2. It checks Mockarty for a matching mock
3. If a mock exists, the mock response is returned
4. If no mock exists, the request is forwarded to the real backend
5. The real backend response is returned to the client

This enables gradual migration from real services to mocks and vice versa.

```bash
# Run generated gRPC server in proxy mode
cd generated-grpc-server
MOCKARTY_URL=http://localhost:5770 \
PROXY_TARGET=real-service:50051 \
go run .
```

## Orchestrator UI

When a generated server is running and connected to the Admin Node, it appears in the Admin Node's Web UI under the **Integrations** section. The generated server exposes a lightweight dashboard at its `/` route showing:

- Connection status to Mockarty
- Registered services and methods (gRPC) or tools and resources (MCP)
- Request statistics
- Health status and version

The version is fetched from the `/health` endpoint of the generated server.

## Examples

### End-to-End: OpenAPI to MCP Server

```bash
# 1. Generate MCP server from OpenAPI spec
mockarty-server-generator \
  --input petstore.yaml \
  --output ./petstore-mcp \
  --type mcp \
  --mockarty-url http://localhost:5770

# 2. Create mocks in Mockarty for each operation
curl -X POST http://localhost:5770/api/v1/mocks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "get-pet",
    "mcp": {
      "serverName": "petstore-mcp",
      "method": "get_pet_by_id"
    },
    "response": {
      "payload": {
        "id": "$.req.petId",
        "name": "$.fake.FirstName",
        "status": "available"
      }
    }
  }'

# 3. Run the generated server
cd petstore-mcp
go run .

# 4. Connect from Claude Desktop or any MCP client
```

### End-to-End: Proto to gRPC Server

```bash
# 1. Generate gRPC server from proto
mockarty-server-generator \
  --input ./protos/user.proto \
  --output ./user-grpc \
  --type grpc

# 2. Create mocks in Mockarty
curl -X POST http://localhost:5770/api/v1/mocks \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "get-user-grpc",
    "grpc": {
      "serviceName": "myservice.UserService",
      "method": "GetUser"
    },
    "response": {
      "payload": {
        "user_id": "$.req.user_id",
        "name": "$.fake.FullName",
        "email": "$.fake.Email"
      }
    }
  }'

# 3. Run the generated server
cd user-grpc
MOCKARTY_URL=http://localhost:5770 go run . --port 50051

# 4. Test with grpcurl
grpcurl -plaintext -d '{"user_id": "123"}' \
  localhost:50051 myservice.UserService/GetUser
```

## Troubleshooting

### Proto parsing fails

- Check that all `import` statements resolve to files inside the uploaded directory (imports are resolved from the directory structure you provide)
- Google well-known types (`google/protobuf/*`) and `validate/validate.proto` are bundled and do not need to be uploaded
- You do **not** need `protoc` or any protobuf toolchain installed — Mockarty parses proto files programmatically via `jhump/protoreflect`

### Generated server cannot reach Mockarty

- Verify `MOCKARTY_URL` points to a running Admin Node or Resolver
- Check that the API token is valid and has access to the target namespace
- Ensure network connectivity between the generated server and Mockarty

### MCP tool calls return empty responses

- Verify mocks exist in Mockarty for each MCP tool
- Check the `serverName` and `method` fields match the generated tool names
- Set `LOG_LEVEL=debug` on the generated server to see request details

### gRPC method not found

- Ensure the proto file's service and method names match the mocks in Mockarty
- `go_package` does not need to be present or formatted in any particular way — Mockarty normalizes it automatically
- Rebuild the generated server after updating proto files

---

<p align="center"><sub>&copy; 2026 Mockarty. All rights reserved.</sub></p>
