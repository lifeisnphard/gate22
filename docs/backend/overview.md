# Backend Architecture

The Gate22 backend is built with Python and FastAPI, providing a robust, scalable platform for MCP gateway functionality.

## Project Structure

```
backend/
├── aci/                        # Main application package
│   ├── alembic/               # Database migrations
│   ├── cli/                   # CLI commands for admin operations
│   ├── common/                # Shared utilities and models
│   │   ├── db/               # Database models and utilities
│   │   ├── schemas/          # Pydantic schemas
│   │   └── enums.py          # Shared enums
│   ├── control_plane/         # Control Plane service
│   │   ├── routes/           # API endpoints
│   │   ├── services/         # Business logic
│   │   ├── middleware/       # Request/response interceptors
│   │   └── main.py           # FastAPI app
│   ├── mcp/                   # MCP service (Remote MCPs)
│   │   ├── routes/           # MCP protocol endpoints
│   │   ├── protocol/         # MCP protocol implementation
│   │   └── main.py           # FastAPI app
│   └── virtual_mcp/           # Virtual MCP service
│       ├── routes/           # Virtual MCP endpoints
│       └── main.py           # FastAPI app
├── mcp_servers/               # Remote MCP server definitions
│   └── {server_name}/
│       ├── server.json       # Server metadata
│       └── tools.json        # Tool definitions
├── virtual_mcp_servers/       # Virtual MCP server definitions
│   └── {server_name}/
│       ├── server.json       # Server metadata
│       └── tools.json        # Tool definitions
├── pyproject.toml             # Python dependencies
├── uv.lock                    # Dependency lock file
└── compose.yml                # Docker Compose configuration
```

## Core Components

### 1. Control Plane Service

The central management API for the platform.

**Port:** 8000  
**Path:** `/v1/control-plane`

#### Key Routes

```python
# aci/control_plane/routes/

auth.py                    # Authentication endpoints
users.py                   # User management
organizations.py           # Organization management
mcp_servers.py            # MCP server management
mcp_server_configurations.py  # Server configurations
mcp_server_bundles.py     # Bundle management
mcp_tools.py              # Tool management
connected_accounts.py     # OAuth credential management
```

#### Services Layer

```python
# aci/control_plane/services/

user_service.py           # User CRUD operations
organization_service.py   # Organization operations
mcp_server_service.py    # MCP server operations
bundle_service.py        # Bundle operations
auth_service.py          # Authentication logic
```

#### Database Models

```python
# aci/common/db/sql_models.py

class User(Base):
    """User account"""
    id: UUID
    email: str
    name: str
    identity_provider: UserIdentityProvider
    password_hash: str | None
    
class Organization(Base):
    """Organization/workspace"""
    id: UUID
    name: str
    slug: str
    
class MCPServer(Base):
    """MCP server definition"""
    id: UUID
    name: str
    url: str
    transport_type: MCPServerTransportType
    description: str
    logo: str
    categories: list[str]
    auth_configs: list[dict]  # JSON field
    
class MCPServerConfiguration(Base):
    """Configured instance of MCP server"""
    id: UUID
    mcp_server_id: UUID
    organization_id: UUID
    name: str
    description: str
    allowed_tools: list[str] | None  # Function allow-list
    
class MCPServerBundle(Base):
    """User's bundle of MCP configurations"""
    id: UUID
    user_id: UUID
    name: str
    description: str
    bundle_key: str  # Authentication key
    configurations: list[UUID]  # Array of config IDs
```

### 2. MCP Service

Handles MCP protocol for remote MCP servers.

**Port:** 8001  
**Path:** `/v1/mcp`

#### Protocol Implementation

```python
# aci/mcp/protocol/server.py

class MCPServer:
    """MCP protocol server implementation"""
    
    async def handle_initialize(self, params: dict) -> dict:
        """Initialize MCP session"""
        return {
            "protocolVersion": "2024-11-05",
            "capabilities": {
                "tools": {"listChanged": True},
                "logging": {},
            },
            "serverInfo": {
                "name": "Gate22 MCP Gateway",
                "version": "0.1.0",
            },
        }
    
    async def handle_tools_list(self, params: dict) -> dict:
        """List available tools from bundle"""
        # Get bundle configuration
        bundle = await get_bundle_by_key(self.bundle_key)
        
        # Collect tools from all configurations
        tools = []
        for config in bundle.configurations:
            config_tools = await get_tools_for_config(config.id)
            tools.extend(config_tools)
        
        # Filter by permissions
        allowed_tools = await filter_tools_by_permissions(bundle, tools)
        
        return {"tools": [tool.to_mcp_format() for tool in allowed_tools]}
    
    async def handle_tools_call(self, params: dict) -> dict:
        """Execute tool"""
        tool_name = params["name"]
        arguments = params.get("arguments", {})
        
        # Find tool and its server
        tool = await get_tool_by_name(tool_name)
        server = await get_mcp_server(tool.mcp_server_id)
        
        # Execute on appropriate backend
        if server.is_virtual:
            result = await execute_virtual_tool(tool, arguments)
        else:
            result = await execute_remote_tool(server, tool, arguments)
        
        # Log execution
        await create_audit_log(
            user_id=self.user_id,
            action="tool_execute",
            tool_name=tool_name,
            result=result,
        )
        
        return result
```

#### Authentication Manager

```python
# aci/common/mcp_auth_manager.py

class MCPAuthManager(httpx.Auth):
    """Manages authentication for MCP server requests"""
    
    def __init__(
        self,
        mcp_server: MCPServer,
        auth_config: AuthConfig,
        auth_credentials: AuthCredentials,
    ):
        self.mcp_server_metadata = MCPServerMetadata.model_validate(
            mcp_server.server_metadata
        )
        self.auth_config = auth_config
        self.auth_credentials = auth_credentials
    
    async def async_auth_flow(
        self, request: httpx.Request
    ) -> AsyncGenerator[httpx.Request, httpx.Response]:
        # Add authentication based on type
        if isinstance(self.auth_config.root, OAuth2Config):
            # OAuth2 authentication
            request.headers["Authorization"] = (
                f"Bearer {self.auth_credentials.root.access_token}"
            )
        elif isinstance(self.auth_config.root, APIKeyConfig):
            # API Key authentication
            if self.auth_config.root.location == HttpLocation.HEADER:
                key = self.auth_config.root.name
                value = self._format_api_key(
                    self.auth_config.root.prefix,
                    self.auth_credentials.root.api_key,
                )
                request.headers[key] = value
        
        yield request
```

### 3. Virtual MCP Service

Converts REST APIs into MCP tools.

**Port:** 8002  
**Path:** `/v1/virtual`

#### REST Tool Execution

```python
# aci/virtual_mcp/routes/mcp.py

async def execute_rest_tool(
    tool: MCPTool,
    arguments: dict,
    auth_credentials: AuthCredentials,
) -> dict:
    """Execute a REST API tool"""
    
    # Parse tool metadata
    tool_metadata = RestToolMetadata.model_validate(tool.tool_metadata)
    
    # Build request
    url = tool_metadata.endpoint
    method = tool_metadata.method
    headers = arguments.get("header", {})
    query_params = arguments.get("query", {})
    body = arguments.get("body", {})
    
    # Add authentication
    auth_manager = MCPAuthManager(
        mcp_server=tool.mcp_server,
        auth_config=auth_config,
        auth_credentials=auth_credentials,
    )
    
    # Execute request
    async with httpx.AsyncClient(auth=auth_manager) as client:
        response = await client.request(
            method=method,
            url=url,
            headers=headers,
            params=query_params,
            json=body if method in ["POST", "PUT", "PATCH"] else None,
        )
    
    # Format response
    return {
        "content": [
            {
                "type": "text",
                "text": json.dumps(response.json(), indent=2),
            }
        ],
        "isError": response.status_code >= 400,
    }
```

## Database Architecture

### Schema Design

Gate22 uses PostgreSQL with the pgvector extension for vector search capabilities.

#### Core Tables

```sql
-- Users and authentication
users
user_refresh_tokens
user_verifications

-- Organizations and teams
organizations
organization_memberships
teams
team_memberships
organization_invitations

-- MCP infrastructure
mcp_servers                 -- Server definitions
mcp_server_configurations   -- Configured instances
mcp_server_bundles          -- User bundles
mcp_tools                   -- Tool definitions

-- Authentication & credentials
connected_accounts          -- OAuth credentials
auth_configs               -- Auth method configs

-- Audit & monitoring
audit_logs                 -- Activity logs
tool_execution_logs        -- Tool usage tracking (planned)
```

#### Relationships

```
Organization (1) -----> (*) MCPServerConfiguration
    |
    └--> (*) OrganizationMembership -----> (1) User
    
User (1) -----> (*) MCPServerBundle
    |
    └--> (*) ConnectedAccount
    
MCPServer (1) -----> (*) MCPServerConfiguration
         |
         └--> (*) MCPTool
         
MCPServerBundle (*) -----> (*) MCPServerConfiguration
```

### Migrations

Database migrations are managed with Alembic.

```bash
# Check for detected changes
docker compose exec runner alembic check

# Generate migration
docker compose exec runner alembic revision --autogenerate -m "description"

# Apply migrations
docker compose exec runner alembic upgrade head

# Rollback
docker compose exec runner alembic downgrade -1
```

## CLI Commands

The CLI provides administrative functions for managing the platform.

```python
# aci/cli/main.py

@app.group()
def mcp():
    """MCP server management commands"""
    pass

@mcp.command()
def upsert_server(server_file: str):
    """Insert or update MCP server from JSON file"""
    with open(server_file) as f:
        data = json.load(f)
    
    # Upsert server
    server = await mcp_server_service.upsert(session, data)
    print(f"✓ Server {server.name} created/updated")

@mcp.command()
def upsert_tools(tools_file: str):
    """Insert or update tools from JSON file"""
    with open(tools_file) as f:
        tools_data = json.load(f)
    
    # Upsert tools
    for tool_data in tools_data:
        tool = await mcp_tool_service.upsert(session, tool_data)
        print(f"✓ Tool {tool.name} created/updated")
```

Usage:

```bash
# Insert MCP server
docker compose exec runner dotenv run python -m aci.cli mcp upsert-server \
  --server-file ./mcp_servers/airtable/server.json

# Insert tools
docker compose exec runner dotenv run python -m aci.cli mcp upsert-tools \
  --tools-file ./mcp_servers/airtable/tools.json
```

## Configuration

### Environment Variables

```bash
# Database
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=secret
POSTGRES_DB=gate22

# Application
ENVIRONMENT=local  # local, staging, production
LOG_LEVEL=INFO
SECRET_KEY=your-secret-key

# OAuth
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_REDIRECT_URI=http://localhost:8000/v1/control-plane/auth/google/callback

# External Services
OPENAI_API_KEY=your-openai-key
SENTRY_DSN=your-sentry-dsn

# CORS
CORS_ORIGINS=http://localhost:3000,https://app.aci.dev
```

### Service Configuration

```python
# aci/control_plane/config.py

from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # App
    APP_TITLE: str = "Gate22 Control Plane"
    APP_ROOT_PATH: str = "/v1/control-plane"
    ENVIRONMENT: Environment = Environment.LOCAL
    
    # Database
    POSTGRES_HOST: str
    POSTGRES_PORT: int = 5432
    POSTGRES_USER: str
    POSTGRES_PASSWORD: str
    POSTGRES_DB: str
    
    # Auth
    JWT_SECRET_KEY: str
    JWT_ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    
    # OAuth
    GOOGLE_CLIENT_ID: str
    GOOGLE_CLIENT_SECRET: str
    GOOGLE_REDIRECT_URI: str
    
    class Config:
        env_file = ".env.local"

settings = Settings()
```

## Testing

### Running Tests

```bash
# Run all tests
docker compose exec runner pytest

# Run with coverage
docker compose exec runner pytest --cov=aci --cov-report=html

# Run specific test file
docker compose exec runner pytest aci/control_plane/tests/test_auth.py

# Run with markers
docker compose exec runner pytest -m "unit"
docker compose exec runner pytest -m "integration"
```

### Test Structure

```python
# aci/control_plane/tests/test_auth.py

import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_google_login(client: AsyncClient, mock_google_oauth):
    """Test Google OAuth login flow"""
    response = await client.post(
        "/v1/control-plane/auth/google",
        json={"code": "test_auth_code"}
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert "refresh_token" in data

@pytest.mark.asyncio
async def test_protected_route_without_auth(client: AsyncClient):
    """Test protected route requires authentication"""
    response = await client.get("/v1/control-plane/users/me")
    assert response.status_code == 401
```

## Code Quality

### Linting & Formatting

```bash
# Run ruff linter
docker compose exec runner ruff check .

# Run ruff formatter
docker compose exec runner ruff format .

# Type checking with mypy
docker compose exec runner mypy aci/
```

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml

repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.6
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

## Deployment

### Docker Containers

Gate22 backend runs as multiple Docker containers:

```yaml
# compose.yml

services:
  control_plane:
    build:
      context: .
      dockerfile: Dockerfile.control_plane
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://...
      - ENVIRONMENT=production
    
  mcp:
    build:
      context: .
      dockerfile: Dockerfile.mcp
    ports:
      - "8001:8001"
    
  virtual_mcp:
    build:
      context: .
      dockerfile: Dockerfile.virtual_mcp
    ports:
      - "8002:8002"
    
  migration:
    build:
      context: .
      dockerfile: Dockerfile.migration
    command: alembic upgrade head
```

### Health Checks

```python
# Health check endpoints

@router.get("/health")
async def health_check():
    """Basic health check"""
    return {"status": "healthy"}

@router.get("/health/db")
async def db_health_check(session: AsyncSession = Depends(get_session)):
    """Database health check"""
    try:
        await session.execute(text("SELECT 1"))
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "database": str(e)}
```

## Performance Considerations

### Database Optimization

- Connection pooling with SQLAlchemy
- Indexes on frequently queried columns
- Vector indexes for tool search (pgvector)
- Query optimization with eager loading

### Async I/O

- All network I/O is async (httpx)
- Background task processing
- Efficient resource utilization

### Caching Strategy

- Tool metadata caching (planned)
- Configuration caching
- Redis integration (roadmap)

## Next Steps

- [Frontend Architecture](../frontend/overview.md)
- [MCP Servers Guide](../mcp-servers/overview.md)
- [API Reference](../api-reference.md)
- [Deployment Guide](../deployment.md)
