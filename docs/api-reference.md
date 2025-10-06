# API Reference

Gate22 provides three main API services for different functionalities. All APIs use REST principles with JSON payloads.

## Base URLs

### Local Development
- **Control Plane**: `http://localhost:8000/v1/control-plane`
- **MCP Service**: `http://localhost:8001/v1/mcp`
- **Virtual MCP Service**: `http://localhost:8002/v1/virtual`

### Production
- **Control Plane**: `https://api.aci.dev/v1/control-plane`
- **MCP Service**: `https://api.aci.dev/v1/mcp`
- **Virtual MCP Service**: `https://api.aci.dev/v1/virtual`

## Authentication

All API requests (except authentication endpoints) require a JWT access token in the Authorization header:

```bash
Authorization: Bearer <access_token>
```

## Control Plane API

The main management API for users, organizations, and MCP configurations.

### Authentication

#### POST /auth/google

Exchange Google OAuth code for access tokens.

**Request:**
```json
{
  "code": "google_auth_code"
}
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer",
  "expires_in": 900
}
```

#### POST /auth/login

Email/password authentication.

**Request:**
```json
{
  "email": "user@example.com",
  "password": "secure_password"
}
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer"
}
```

#### POST /auth/refresh

Refresh access token using refresh token.

**Request:**
```json
{
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGc..."
}
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer",
  "expires_in": 900
}
```

### Users

#### GET /users/me

Get current user profile.

**Response:**
```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "email": "user@example.com",
  "name": "John Doe",
  "identity_provider": "google",
  "email_verified": true,
  "created_at": "2024-01-01T00:00:00Z"
}
```

#### PATCH /users/me

Update current user profile.

**Request:**
```json
{
  "name": "John Smith"
}
```

### Organizations

#### GET /organizations

List user's organizations.

**Query Parameters:**
- `offset` (integer): Pagination offset (default: 0)
- `limit` (integer): Results per page (default: 100)

**Response:**
```json
{
  "data": [
    {
      "id": "org-123",
      "name": "Acme Corp",
      "slug": "acme-corp",
      "role": "admin",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 1,
  "offset": 0,
  "limit": 100
}
```

#### POST /organizations

Create a new organization.

**Request:**
```json
{
  "name": "My Organization",
  "slug": "my-org"
}
```

### MCP Servers

#### GET /mcp-servers

List available MCP servers.

**Query Parameters:**
- `offset` (integer): Pagination offset
- `limit` (integer): Results per page
- `category` (string): Filter by category
- `search` (string): Search by name/description

**Response:**
```json
{
  "data": [
    {
      "id": "server-123",
      "name": "AIRTABLE",
      "url": "https://mcp.aci.dev/virtual/mcp?server_name=AIRTABLE",
      "transport_type": "streamable_http",
      "description": "Airtable MCP server",
      "logo": "https://example.com/logo.svg",
      "categories": ["Data & Analytics"],
      "auth_configs": [
        {
          "type": "api_key",
          "location": "header",
          "name": "Authorization",
          "prefix": "Bearer"
        }
      ],
      "server_metadata": {
        "is_virtual_mcp_server": true
      }
    }
  ],
  "total": 100,
  "offset": 0,
  "limit": 100
}
```

#### GET /mcp-servers/{server_id}

Get MCP server details.

**Response:**
```json
{
  "id": "server-123",
  "name": "AIRTABLE",
  "url": "https://mcp.aci.dev/virtual/mcp?server_name=AIRTABLE",
  "description": "Airtable MCP server",
  "logo": "https://example.com/logo.svg",
  "categories": ["Data & Analytics"],
  "auth_configs": [...],
  "tools_count": 15
}
```

#### POST /mcp-servers/{server_id}/refresh-tools

Refresh tools from MCP server.

**Response:**
```json
{
  "added": 5,
  "updated": 3,
  "removed": 2,
  "unchanged": 10,
  "total": 15
}
```

### MCP Server Configurations

#### GET /mcp-server-configurations

List MCP server configurations.

**Query Parameters:**
- `organization_id` (string): Filter by organization
- `offset` (integer): Pagination offset
- `limit` (integer): Results per page

**Response:**
```json
{
  "data": [
    {
      "id": "config-123",
      "mcp_server_id": "server-123",
      "organization_id": "org-123",
      "name": "Airtable Production",
      "description": "Production Airtable access",
      "auth_config": {
        "type": "api_key",
        "location": "header",
        "name": "Authorization",
        "prefix": "Bearer"
      },
      "allowed_tools": ["AIRTABLE__LIST_BASES", "AIRTABLE__GET_BASE"],
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 1
}
```

#### POST /mcp-server-configurations

Create MCP server configuration.

**Request:**
```json
{
  "mcp_server_id": "server-123",
  "name": "Airtable Production",
  "description": "Production Airtable access",
  "auth_config_index": 0,
  "allowed_tools": ["AIRTABLE__LIST_BASES", "AIRTABLE__GET_BASE"]
}
```

#### PATCH /mcp-server-configurations/{config_id}

Update MCP server configuration.

**Request:**
```json
{
  "name": "Updated Name",
  "allowed_tools": ["TOOL_1", "TOOL_2"]
}
```

### MCP Server Bundles

#### GET /mcp-server-bundles

List user's bundles.

**Response:**
```json
{
  "data": [
    {
      "id": "bundle-123",
      "user_id": "user-123",
      "name": "My Production Bundle",
      "description": "Production tools bundle",
      "bundle_key": "bnd_abc123def456",
      "configurations": [
        {
          "id": "config-123",
          "name": "Airtable Production",
          "mcp_server": {
            "id": "server-123",
            "name": "AIRTABLE",
            "logo": "https://example.com/logo.svg"
          }
        }
      ],
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 1
}
```

#### POST /mcp-server-bundles

Create a new bundle.

**Request:**
```json
{
  "name": "My Bundle",
  "description": "Bundle description",
  "mcp_server_configuration_ids": ["config-123", "config-456"]
}
```

**Response:**
```json
{
  "id": "bundle-123",
  "name": "My Bundle",
  "description": "Bundle description",
  "bundle_key": "bnd_abc123def456",
  "endpoint_url": "https://mcp.aci.dev/bundle/bnd_abc123def456",
  "created_at": "2024-01-01T00:00:00Z"
}
```

#### DELETE /mcp-server-bundles/{bundle_id}

Delete a bundle.

**Response:**
```json
{
  "success": true
}
```

### MCP Tools

#### GET /mcp-tools

List MCP tools.

**Query Parameters:**
- `mcp_server_id` (string): Filter by server
- `search` (string): Search by name/description
- `offset` (integer): Pagination offset
- `limit` (integer): Results per page

**Response:**
```json
{
  "data": [
    {
      "id": "tool-123",
      "name": "AIRTABLE__LIST_BASES",
      "description": "List all bases in Airtable",
      "mcp_server_id": "server-123",
      "input_schema": {
        "type": "object",
        "properties": {...}
      },
      "tool_metadata": {
        "type": "rest",
        "method": "GET",
        "endpoint": "https://api.airtable.com/v0/meta/bases"
      }
    }
  ],
  "total": 15
}
```

#### GET /mcp-tools/{tool_name}

Get tool details by name.

**Response:**
```json
{
  "id": "tool-123",
  "name": "AIRTABLE__LIST_BASES",
  "description": "List all bases in Airtable",
  "mcp_server": {
    "id": "server-123",
    "name": "AIRTABLE",
    "logo": "https://example.com/logo.svg"
  },
  "input_schema": {...},
  "tool_metadata": {...}
}
```

### Connected Accounts

#### GET /connected-accounts

List user's connected accounts.

**Response:**
```json
{
  "data": [
    {
      "id": "account-123",
      "mcp_server_configuration_id": "config-123",
      "ownership": "user",
      "auth_type": "oauth2",
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 1
}
```

#### POST /connected-accounts

Create connected account (initiate OAuth flow).

**Request:**
```json
{
  "mcp_server_configuration_id": "config-123"
}
```

**Response:**
```json
{
  "authorize_url": "https://provider.com/oauth/authorize?client_id=...",
  "state": "random_state_value"
}
```

## MCP Service API

Implements the MCP protocol for bundle endpoints.

### MCP Protocol Endpoints

#### POST /bundle/{bundle_key}

MCP protocol endpoint for tools.

**Headers:**
- `Authorization: Bearer <bundle_key>`

**Request (list tools):**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "AIRTABLE__LIST_BASES",
        "description": "List all bases in Airtable",
        "inputSchema": {
          "type": "object",
          "properties": {...}
        }
      }
    ]
  }
}
```

**Request (call tool):**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "AIRTABLE__LIST_BASES",
    "arguments": {
      "header": {
        "Content-Type": "application/json"
      }
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"bases\": [...]}"
      }
    ]
  }
}
```

## Virtual MCP Service API

Executes REST API tools through MCP protocol.

### Tool Execution

The Virtual MCP service translates MCP tool calls to REST API requests based on the tool's metadata.

**Example Tool Metadata:**
```json
{
  "type": "rest",
  "method": "POST",
  "endpoint": "https://api.example.com/v1/items"
}
```

**MCP Call:**
```json
{
  "name": "EXAMPLE__CREATE_ITEM",
  "arguments": {
    "body": {
      "name": "New Item",
      "price": 99.99
    }
  }
}
```

**Translated REST Call:**
```http
POST https://api.example.com/v1/items
Content-Type: application/json
Authorization: Bearer <user_token>

{
  "name": "New Item",
  "price": 99.99
}
```

## Error Responses

All APIs use standard HTTP status codes and return errors in this format:

```json
{
  "detail": "Error message",
  "code": "ERROR_CODE",
  "status": 400
}
```

### Common Error Codes

| Code | Status | Description |
|------|--------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid authentication |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 422 | Invalid request data |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |

## Rate Limiting

API rate limits (per user):
- **Control Plane**: 1000 requests/hour
- **MCP Service**: 10000 requests/hour
- **Virtual MCP Service**: 5000 requests/hour

Rate limit headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

## Pagination

List endpoints support pagination:

**Query Parameters:**
- `offset`: Number of items to skip (default: 0)
- `limit`: Number of items to return (default: 100, max: 100)

**Response Format:**
```json
{
  "data": [...],
  "total": 500,
  "offset": 0,
  "limit": 100
}
```

## Interactive API Documentation

For interactive API exploration:

- **Control Plane**: [http://localhost:8000/v1/control-plane/docs](http://localhost:8000/v1/control-plane/docs)
- **MCP Service**: [http://localhost:8001/v1/mcp/docs](http://localhost:8001/v1/mcp/docs)
- **Virtual MCP Service**: [http://localhost:8002/v1/virtual/docs](http://localhost:8002/v1/virtual/docs)

## Code Examples

### Python

```python
import httpx

# Authenticate
async with httpx.AsyncClient() as client:
    response = await client.post(
        "http://localhost:8000/v1/control-plane/auth/login",
        json={"email": "user@example.com", "password": "password"}
    )
    tokens = response.json()
    
    # Use access token
    headers = {"Authorization": f"Bearer {tokens['access_token']}"}
    
    # List MCP servers
    servers = await client.get(
        "http://localhost:8000/v1/control-plane/mcp-servers",
        headers=headers
    )
    print(servers.json())
```

### TypeScript

```typescript
// Authenticate
const response = await fetch('http://localhost:8000/v1/control-plane/auth/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'user@example.com',
    password: 'password'
  })
});

const { access_token } = await response.json();

// List MCP servers
const servers = await fetch('http://localhost:8000/v1/control-plane/mcp-servers', {
  headers: { 'Authorization': `Bearer ${access_token}` }
});

console.log(await servers.json());
```

### cURL

```bash
# Authenticate
curl -X POST http://localhost:8000/v1/control-plane/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"password"}'

# List MCP servers
curl http://localhost:8000/v1/control-plane/mcp-servers \
  -H "Authorization: Bearer <access_token>"
```

## WebSocket Support (Planned)

Future versions will support WebSocket for real-time MCP protocol:

```javascript
const ws = new WebSocket('wss://mcp.aci.dev/bundle/<bundle_key>');

ws.onopen = () => {
  ws.send(JSON.stringify({
    jsonrpc: '2.0',
    id: 1,
    method: 'tools/list',
    params: {}
  }));
};

ws.onmessage = (event) => {
  const response = JSON.parse(event.data);
  console.log(response.result);
};
```

## Next Steps

- [Architecture Overview](architecture/overview.md)
- [Authentication Guide](architecture/authentication.md)
- [Backend Development](backend/overview.md)
