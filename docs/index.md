# Gate22 Documentation

Welcome to the comprehensive documentation for **Gate22**, an open-source MCP (Model Context Protocol) Gateway and Control Plane for AI governance.

![Gate22 Architecture](https://raw.githubusercontent.com/lifeisnphard/gate22/main/frontend/public/gate22-architecture-light-bg.svg)

## What is Gate22?

Gate22 is built for engineering organizations and teams (Platform/Infra/DevEx, Security, Data/Analytics Eng). It allows you to govern which tools agents can use, what they can do, and how it's audited—across agentic IDEs like Cursor, or other agents and AI tools.

**Key Capabilities:**

- **Function Allow-List Permissioning**: Set precise function-level permissions per MCP configuration
- **Flexible Credential Modes**: Admin-set credential modes (org-shared or per-user)
- **User-Created Bundles**: Create private bundles that expose multiple MCPs through a single unified endpoint
- **Context Window Optimization**: Bundle condenses any number of MCPs and tools into just two functions (search and execute)
- **MCP Tool Management**: Refresh and review tool changes with diff views before deployment
- **Audit & Compliance**: Track and audit all tool usage across your organization

## Architecture Overview

Gate22 consists of three main components:

1. **Control Plane** - Core operations and management API (FastAPI)
2. **MCP Service** - Remote MCP server hosting and proxying (FastAPI)
3. **Virtual MCP Service** - Virtual MCP servers for REST API integration (FastAPI)
4. **Frontend Portal** - Management UI (Next.js 15 + TypeScript)

## Quick Links

- [Getting Started](getting-started.md) - Set up Gate22 locally
- [Architecture](architecture/overview.md) - Deep dive into the system architecture
- [Backend Guide](backend/overview.md) - Backend development and structure
- [Frontend Guide](frontend/overview.md) - Frontend development and structure
- [MCP Servers](mcp-servers/overview.md) - Understanding and adding MCP servers
- [API Reference](api-reference.md) - API documentation

## Use Cases

### Platform & Infrastructure Teams
Roll out agentic IDEs or internal AI agents at organizational scale with proper governance and security controls.

### Security & GRC
Implement least-privilege execution and comprehensive auditability for agent tool-use across your organization.

### Data & Analytics Engineering
Provide governed access to internal tools and BI systems with minimal setup and maximum security.

## Community & Support

- **Cloud Version**: [gate22.aci.dev](https://gate22.aci.dev/)
- **Discord**: [Join our community](https://discord.com/invite/UU2XAnfHJh)
- **GitHub**: [lifeisnphard/gate22](https://github.com/lifeisnphard/gate22)
- **Twitter/X**: [@AipoLabs](https://x.com/AipoLabs)

## License

Gate22 is licensed under the Apache License 2.0. See the [LICENSE](https://github.com/lifeisnphard/gate22/blob/main/LICENSE) file for details.
