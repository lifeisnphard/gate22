# Architecture Overview

Gate22 is designed as a modular, scalable system for managing and governing MCP (Model Context Protocol) servers. The architecture follows a clear separation of concerns with distinct services handling different responsibilities.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend Portal                         │
│                    (Next.js 15 + TypeScript)                    │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Agents &   │  │  MCP Server  │  │   Settings   │         │
│  │   Bundles    │  │     Mgmt     │  │  & Members   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ REST API
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Control Plane API                         │
│                         (FastAPI Service)                        │
│                                                                  │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   User & Auth   │  │   MCP Server │  │   Bundles &  │      │
│  │   Management    │  │ Configurations│  │  Permissions │      │
│  └─────────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │
                    ┌─────────┴─────────┐
                    ▼                    ▼
        ┌──────────────────┐  ┌──────────────────┐
        │   MCP Service    │  │ Virtual MCP Svc  │
        │  (Remote MCPs)   │  │  (REST → MCP)    │
        │                  │  │                  │
        │  ┌────────────┐  │  │  ┌────────────┐  │
        │  │  Protocol  │  │  │  │   REST     │  │
        │  │   Proxy    │  │  │  │  Executor  │  │
        │  └────────────┘  │  │  └────────────┘  │
        └──────────────────┘  └──────────────────┘
                    │                    │
                    ▼                    ▼
        ┌──────────────────┐  ┌──────────────────┐
        │  Remote MCP      │  │   External       │
        │  Servers         │  │   REST APIs      │
        └──────────────────┘  └──────────────────┘
                              
                    ┌─────────────────┐
                    │   PostgreSQL    │
                    │  + pgvector     │
                    └─────────────────┘
```

## Core Components

### 1. Frontend Portal (Next.js 15)

The user interface for managing the Gate22 platform.

**Key Features:**
- **Authentication**: Google OAuth, email/password login
- **MCP Server Management**: Browse, configure, and manage MCP servers
- **Bundle Creation**: Create and manage MCP bundles with custom tool selections
- **Permissions**: Configure function-level allow lists
- **Monitoring**: View tool usage and audit logs

**Technology Stack:**
- Next.js 15 with App Router
- TypeScript for type safety
- Tailwind CSS v4 for styling
- Radix UI + shadcn/ui for components
- Zustand + React Query for state management
- React Hook Form + Zod for form validation

### 2. Control Plane API (FastAPI)

The central management and orchestration service.

**Responsibilities:**
- User authentication and authorization
- Organization and team management
- MCP server registration and configuration
- Bundle creation and management
- Permission and access control
- Audit logging and monitoring

**Key Modules:**
- `routes/`: API endpoints for different resources
- `services/`: Business logic and data operations
- `middleware/`: Request/response interceptors
- `access_control.py`: Permission enforcement

**Database Schema:**
- Users, Organizations, Teams
- MCP Servers and Configurations
- Bundles and Tool Permissions
- Connected Accounts (OAuth credentials)
- Audit Logs

### 3. MCP Service (FastAPI)

Hosts and proxies remote MCP servers.

**Responsibilities:**
- MCP protocol implementation
- Connection to remote MCP servers
- Authentication handling for remote MCPs
- Tool discovery and execution
- Request/response transformation

**Key Features:**
- Streamable HTTP transport support
- OAuth2 and API key authentication
- Dynamic tool loading
- Error handling and retry logic

**Protocol Support:**
- Streamable HTTP (primary)
- SSE (Server-Sent Events)
- WebSocket (planned)

### 4. Virtual MCP Service (FastAPI)

Converts REST APIs into MCP servers.

**Responsibilities:**
- REST API integration
- Dynamic tool generation from API specs
- Request/response mapping
- Authentication forwarding
- Error handling and validation

**How It Works:**
1. Admin defines a Virtual MCP server with REST endpoints
2. Tools are created with HTTP method, endpoint, and schema
3. Service translates MCP tool calls to REST API requests
4. Responses are mapped back to MCP format

**Authentication Support:**
- API Key (header, query, cookie)
- OAuth2 (authorization code flow)
- No Auth (public APIs)

## Data Flow

### User Creates a Bundle

```
1. User selects MCP configurations in Frontend
2. Frontend → POST /bundles to Control Plane
3. Control Plane validates permissions
4. Control Plane creates bundle record in DB
5. Control Plane generates unique bundle key
6. Frontend displays MCP endpoint URL
```

### Agent Uses a Bundle

```
1. Agent connects to bundle endpoint
2. MCP Service authenticates request
3. MCP Service loads bundle configuration
4. Agent calls search/execute functions
5. MCP Service routes to appropriate backend:
   - Remote MCP → MCP Service → Remote Server
   - Virtual MCP → Virtual MCP Service → REST API
6. Results returned to agent
7. Audit log created in Control Plane
```

### Tool Discovery Flow

```
1. Agent requests tool list
2. MCP Service queries Control Plane for bundle
3. Control Plane returns allowed tools
4. MCP Service filters by permissions
5. Tool metadata returned to agent
```

## Security Architecture

### Authentication Layers

1. **User Authentication** (Frontend → Control Plane)
   - JWT tokens with refresh mechanism
   - Google OAuth integration
   - Session management

2. **Bundle Authentication** (Agent → MCP Service)
   - Bundle key validation
   - User identity verification
   - Permission checks

3. **Backend Authentication** (MCP → External Services)
   - OAuth2 with PKCE
   - API key management
   - Credential encryption

### Authorization Model

**Role-Based Access Control (RBAC):**
- Organization roles: Admin, Member
- Team roles: Admin, Member
- Resource-level permissions

**Function-Level Permissions:**
- Allow list per MCP configuration
- Bundle-specific tool filtering
- Dynamic permission evaluation

### Data Protection

- **Encryption at Rest**: Sensitive credentials encrypted in database
- **Encryption in Transit**: TLS for all external communications
- **Secret Management**: Environment-based configuration
- **Audit Logging**: Comprehensive activity tracking

## Scalability Considerations

### Horizontal Scaling

- **Stateless Services**: All services are stateless and can be scaled horizontally
- **Database Connection Pooling**: Efficient database resource usage
- **Load Balancing**: Support for multiple instances behind load balancer

### Performance Optimizations

- **Connection Reuse**: HTTP connection pooling for external services
- **Caching**: Tool metadata caching (planned)
- **Async Processing**: Non-blocking I/O for all network operations
- **Vector Search**: pgvector for efficient tool search

### Resource Management

- **Rate Limiting**: Per-user and per-bundle rate limits (planned)
- **Quotas**: Usage quotas and budget controls (roadmap)
- **Circuit Breakers**: Fault tolerance for external services

## Technology Stack

### Backend

- **Framework**: FastAPI (Python 3.12+)
- **Database**: PostgreSQL 15+ with pgvector
- **ORM**: SQLAlchemy 2.0
- **Migrations**: Alembic
- **Authentication**: JWT, OAuth2
- **HTTP Client**: httpx (async)
- **Validation**: Pydantic v2
- **Testing**: pytest
- **Code Quality**: ruff, mypy

### Frontend

- **Framework**: Next.js 15
- **Language**: TypeScript 5
- **Styling**: Tailwind CSS v4
- **UI Components**: Radix UI + shadcn/ui
- **State**: Zustand + React Query
- **Forms**: React Hook Form + Zod
- **Testing**: Vitest + Testing Library

### Infrastructure

- **Container**: Docker + Docker Compose
- **CI/CD**: GitHub Actions
- **Monitoring**: Sentry (error tracking)
- **Logging**: Structured JSON logging

## Next Steps

- [Backend Architecture Deep Dive](../backend/overview.md)
- [Frontend Architecture Deep Dive](../frontend/overview.md)
- [MCP Server Implementation](../mcp-servers/overview.md)
- [Authentication & Authorization](authentication.md)
