# Authentication & Authorization

Gate22 implements a comprehensive security model with multiple layers of authentication and fine-grained authorization controls.

## Authentication Flow

### User Authentication (Frontend)

Gate22 supports multiple authentication methods for users accessing the management portal.

#### 1. Google OAuth

```
┌─────────┐                                    ┌────────────┐
│ Browser │                                    │   Google   │
└────┬────┘                                    └─────┬──────┘
     │                                                │
     │ 1. Click "Sign in with Google"                │
     ├──────────────────────────────────────────────►│
     │                                                │
     │ 2. Google OAuth authorization                 │
     │◄──────────────────────────────────────────────┤
     │                                                │
     │ 3. Authorization code                         │
     ├──────────────┐                                │
     │              │                                │
     │              ▼                                │
     │     ┌──────────────┐                          │
     │     │Control Plane │                          │
     │     └──────┬───────┘                          │
     │            │                                  │
     │            │ 4. Exchange code for tokens      │
     │            ├─────────────────────────────────►│
     │            │                                  │
     │            │ 5. ID token + access token       │
     │            │◄─────────────────────────────────┤
     │            │                                  │
     │            │ 6. Create/update user            │
     │            ├──────────►[Database]             │
     │            │                                  │
     │ 7. JWT tokens (access + refresh)              │
     │◄───────────┤                                  │
     │            │                                  │
```

**Implementation Details:**

```python
# backend/aci/control_plane/routes/auth.py

@router.post("/google")
async def google_login(
    request: Request,
    code: str,
    session: AsyncSession = Depends(get_session),
):
    # Exchange authorization code for tokens
    google_tokens = await exchange_google_code(code)
    
    # Get user info from Google
    user_info = await get_google_user_info(google_tokens.access_token)
    
    # Create or update user in database
    user = await user_service.get_or_create_google_user(
        session=session,
        email=user_info.email,
        name=user_info.name,
        email_verified=user_info.email_verified,
    )
    
    # Generate JWT tokens
    access_token = create_access_token(user.id)
    refresh_token = await create_refresh_token(session, user.id)
    
    return {"access_token": access_token, "refresh_token": refresh_token}
```

#### 2. Email/Password Authentication

```
┌─────────┐                           ┌────────────┐
│ Browser │                           │Control Plane│
└────┬────┘                           └─────┬──────┘
     │                                      │
     │ 1. POST /auth/login                  │
     │    {email, password}                 │
     ├─────────────────────────────────────►│
     │                                      │
     │                                      │ 2. Verify password hash
     │                                      ├─────────►[Database]
     │                                      │
     │                                      │ 3. Generate JWT tokens
     │                                      │
     │ 4. JWT tokens (access + refresh)     │
     │◄─────────────────────────────────────┤
     │                                      │
```

**Password Security:**
- Passwords hashed with bcrypt (cost factor: 12)
- Passwords never stored in plaintext
- Minimum password requirements enforced

### Bundle Authentication (MCP Clients)

When AI agents or IDEs connect to a bundle endpoint, they use bundle key authentication.

```
┌──────────┐                          ┌─────────────┐
│   Agent  │                          │ MCP Service │
└─────┬────┘                          └──────┬──────┘
      │                                      │
      │ 1. Connect with bundle key           │
      │    Authorization: Bearer <bundle_key>│
      ├─────────────────────────────────────►│
      │                                      │
      │                                      │ 2. Validate bundle key
      │                                      ├────────►[Control Plane]
      │                                      │
      │                                      │ 3. Load bundle config
      │                                      │◄────────[Control Plane]
      │                                      │
      │ 4. MCP connection established        │
      │◄─────────────────────────────────────┤
      │                                      │
```

### Service-to-Service Authentication

For backend authentication to external MCP servers and APIs, Gate22 supports multiple methods.

#### OAuth2 with PKCE

```python
# backend/aci/common/mcp_auth_manager.py

class MCPAuthManager(httpx.Auth):
    """Handles authentication for MCP server requests"""
    
    async def async_auth_flow(
        self, request: httpx.Request
    ) -> AsyncGenerator[httpx.Request, httpx.Response]:
        if isinstance(self.auth_config.root, OAuth2Config):
            # Add OAuth2 access token to request
            if self.mcp_server_metadata.is_virtual_mcp_server:
                # Virtual MCP: use special header
                request.headers[VIRTUAL_MCP_AUTH_TOKEN_HEADER] = (
                    self._construct_auth_token_for_virtual_mcp(
                        self.auth_config.root.location,
                        self.auth_config.root.name,
                        self.auth_config.root.prefix,
                        self.auth_credentials.root.access_token,
                    )
                )
            else:
                # Remote MCP: standard OAuth2 header
                request.headers["Authorization"] = (
                    f"Bearer {self.auth_credentials.root.access_token}"
                )
        
        yield request
```

## Authorization Model

### Role-Based Access Control (RBAC)

Gate22 implements RBAC at multiple levels:

#### Organization Roles

```python
class OrganizationRole(str, Enum):
    ADMIN = "admin"    # Full organization control
    MEMBER = "member"  # Standard user access
```

**Admin Permissions:**
- Create/modify MCP server configurations
- Manage organization members and teams
- Set credential modes (org-shared vs per-user)
- Configure function-level allow lists
- View all audit logs

**Member Permissions:**
- View accessible MCP configurations
- Create personal bundles
- Configure personal credentials
- View own audit logs

#### Team Roles

```python
class TeamRole(str, Enum):
    ADMIN = "admin"   # Team management
    MEMBER = "member" # Team access
```

Teams enable isolation and collaboration within organizations.

### Function-Level Permissions

Each MCP configuration can define an allow list of permitted functions.

```python
# Database model
class MCPServerConfiguration(Base):
    __tablename__ = "mcp_server_configurations"
    
    # ... other fields ...
    
    allowed_tools: Mapped[list[str] | None] = mapped_column(
        ARRAY(String(MAX_STRING_LENGTH)), 
        nullable=True
    )
```

**Permission Evaluation:**

```python
async def filter_tools_by_permissions(
    bundle: MCPServerBundle,
    tools: list[MCPTool],
) -> list[MCPTool]:
    """Filter tools based on bundle permissions"""
    allowed_tools = set()
    
    for config in bundle.configurations:
        if config.allowed_tools:
            # Specific allow list
            allowed_tools.update(config.allowed_tools)
        else:
            # No restrictions - all tools allowed
            allowed_tools.update(tool.name for tool in tools)
    
    return [tool for tool in tools if tool.name in allowed_tools]
```

### Credential Modes

Admins can configure how credentials are managed per MCP configuration:

#### 1. Organization-Shared Credentials

```python
class ConnectedAccountOwnership(str, Enum):
    ORGANIZATION = "organization"
```

- Single set of credentials for entire organization
- Managed by organization admins
- Used for internal/trusted services
- No per-user OAuth flow needed

**Use Case:** Internal APIs, shared service accounts

#### 2. Per-User Credentials

```python
class ConnectedAccountOwnership(str, Enum):
    USER = "user"
```

- Each user provides their own credentials
- OAuth flow per user
- Better audit trail
- Supports external services requiring user consent

**Use Case:** External SaaS APIs (GitHub, Notion, etc.)

### Access Control Implementation

```python
# backend/aci/control_plane/access_control.py

async def verify_bundle_access(
    session: AsyncSession,
    user_id: UUID,
    bundle_id: UUID,
) -> bool:
    """Verify user has access to bundle"""
    
    # Check if user owns the bundle
    bundle = await bundle_service.get_by_id(session, bundle_id)
    if bundle.user_id == user_id:
        return True
    
    # Check if bundle is shared with user's team/org (future)
    # ...
    
    return False

async def verify_configuration_access(
    session: AsyncSession,
    user_id: UUID,
    configuration_id: UUID,
) -> bool:
    """Verify user has access to MCP configuration"""
    
    # Get user's organization membership
    membership = await get_organization_membership(session, user_id)
    
    # Get configuration
    config = await configuration_service.get_by_id(session, configuration_id)
    
    # Verify same organization
    if config.organization_id != membership.organization_id:
        return False
    
    # Check visibility rules
    if config.is_private and config.created_by_id != user_id:
        # Private config - only creator can access
        return False
    
    return True
```

## Token Management

### JWT Token Structure

**Access Token:**
```json
{
  "sub": "user_id",
  "exp": 1640000000,
  "iat": 1639999000,
  "type": "access"
}
```

- **Lifetime:** 15 minutes
- **Purpose:** API authentication
- **Storage:** Memory (frontend state)

**Refresh Token:**
```json
{
  "sub": "user_id",
  "exp": 1640000000,
  "iat": 1639999000,
  "type": "refresh",
  "jti": "unique_token_id"
}
```

- **Lifetime:** 7 days
- **Purpose:** Access token renewal
- **Storage:** HttpOnly cookie or secure storage

### Token Refresh Flow

```python
# backend/aci/control_plane/routes/auth.py

@router.post("/refresh")
async def refresh_token(
    refresh_token: str,
    session: AsyncSession = Depends(get_session),
):
    # Verify refresh token
    payload = verify_jwt_token(refresh_token, token_type="refresh")
    
    # Check if token is revoked
    token_record = await get_refresh_token(session, payload["jti"])
    if not token_record or token_record.revoked:
        raise HTTPException(401, "Invalid refresh token")
    
    # Generate new access token
    access_token = create_access_token(payload["sub"])
    
    return {"access_token": access_token}
```

## Audit Logging

All authenticated actions are logged for compliance and security.

```python
# Audit log structure
class AuditLog(Base):
    __tablename__ = "audit_logs"
    
    id: Mapped[UUID]
    user_id: Mapped[UUID]
    organization_id: Mapped[UUID]
    action: Mapped[str]  # e.g., "tool_execute", "bundle_create"
    resource_type: Mapped[str]
    resource_id: Mapped[str]
    metadata: Mapped[dict]  # Additional context
    ip_address: Mapped[str]
    user_agent: Mapped[str]
    created_at: Mapped[datetime]
```

**Logged Events:**
- User authentication (login/logout)
- MCP configuration changes
- Bundle creation/modification
- Tool execution
- Permission changes
- Credential updates

## Security Best Practices

### For Administrators

1. **Use Strong Passwords**: Minimum 12 characters with complexity requirements
2. **Enable MFA**: Multi-factor authentication (roadmap)
3. **Rotate Credentials**: Regular credential rotation for service accounts
4. **Audit Regularly**: Review audit logs for suspicious activity
5. **Principle of Least Privilege**: Grant minimal necessary permissions

### For Developers

1. **Secure Bundle Keys**: Treat bundle keys as secrets
2. **Use Environment Variables**: Never hardcode credentials
3. **Validate Inputs**: Always validate and sanitize inputs
4. **Handle Tokens Securely**: Store tokens in secure storage
5. **Implement Timeouts**: Set reasonable timeouts for operations

### For Deployments

1. **Use TLS**: Always use HTTPS in production
2. **Secure Database**: Use strong database passwords and encryption
3. **Network Isolation**: Isolate backend services from public internet
4. **Secret Management**: Use proper secret management (AWS Secrets Manager, etc.)
5. **Monitor & Alert**: Set up monitoring for security events

## API Authentication Examples

### Frontend API Call

```typescript
// frontend/src/lib/api-client.ts

export async function fetcherWithAuth<T>(token: string) {
  return async (url: string, options?: RequestInit): Promise<T> => {
    const response = await fetch(url, {
      ...options,
      headers: {
        ...options?.headers,
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    });
    
    if (!response.ok) {
      if (response.status === 401) {
        // Token expired - trigger refresh
        await refreshAccessToken();
      }
      throw new Error(`API error: ${response.status}`);
    }
    
    return response.json();
  };
}
```

### MCP Client Connection

```python
# Example: Connecting to a bundle with authentication

import httpx
from mcp import ClientSession

async def connect_to_bundle():
    bundle_key = os.getenv("GATE22_BUNDLE_KEY")
    
    async with httpx.AsyncClient() as client:
        # Connect with bundle key
        response = await client.get(
            "https://mcp.aci.dev/bundle/xyz",
            headers={"Authorization": f"Bearer {bundle_key}"}
        )
        
        # Use MCP protocol
        async with ClientSession(response.url) as session:
            tools = await session.list_tools()
            result = await session.call_tool("TOOL_NAME", arguments={})
```

## Next Steps

- [API Reference](../api-reference.md)
- [Deployment Guide](../deployment.md)
