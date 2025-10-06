# MCP Servers Overview

Gate22 supports two types of MCP servers: **Remote MCP Servers** and **Virtual MCP Servers**. This guide explains both types and shows you how to add new servers to the platform.

## Understanding MCP Server Types

### Remote MCP Servers

Remote MCP servers are external MCP protocol servers that Gate22 proxies and manages.

**Characteristics:**
- Implement the full MCP protocol
- Run independently (external services)
- Gate22 acts as a proxy with authentication and permissions
- Examples: Claude Desktop MCP servers, custom MCP implementations

**Use Cases:**
- Integration with existing MCP servers
- Protocol-native tool providers
- Services that already support MCP

### Virtual MCP Servers

Virtual MCP servers are REST APIs that Gate22 converts into MCP tools.

**Characteristics:**
- Standard REST APIs (not MCP protocol)
- Gate22 translates REST → MCP format
- Dynamic tool generation from API specifications
- Examples: Airtable, Notion, GitHub (via REST API)

**Use Cases:**
- Integration with REST APIs
- Quick API-to-tool conversion
- Services without native MCP support

## Directory Structure

### Remote MCP Servers

```
backend/mcp_servers/
└── {server_name}/
    ├── server.json        # Server metadata and configuration
    └── tools.json         # Tool definitions (optional)
```

### Virtual MCP Servers

```
backend/virtual_mcp_servers/
└── {server_name}/
    ├── server.json        # Server metadata
    └── tools.json         # REST API tool definitions
```

## Server Definition Format

### server.json Structure

#### Remote MCP Server

```json
{
  "name": "EXAMPLE_REMOTE",
  "url": "https://mcp.example.com/sse",
  "transport_type": "streamable_http",
  "description": "Example remote MCP server",
  "logo": "https://example.com/logo.svg",
  "categories": ["Productivity", "Communication"],
  "auth_configs": [
    {
      "type": "oauth2",
      "location": "header",
      "name": "Authorization",
      "prefix": "Bearer",
      "authorize_url": "https://example.com/oauth/authorize",
      "access_token_url": "https://example.com/oauth/token",
      "scopes": ["read", "write"]
    }
  ],
  "server_metadata": {
    "is_virtual_mcp_server": false
  }
}
```

#### Virtual MCP Server

```json
{
  "name": "AIRTABLE",
  "url": "https://mcp.aci.dev/virtual/mcp?server_name=AIRTABLE",
  "transport_type": "streamable_http",
  "description": "Airtable is a cloud-based platform that combines the functionality of a database with the visual interface of a spreadsheet.",
  "logo": "https://raw.githubusercontent.com/aipotheosis-labs/aipolabs-icons/refs/heads/main/apps/airtable.svg",
  "categories": ["Data & Analytics", "Productivity"],
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
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier (uppercase, underscore-separated) |
| `url` | string | Yes | MCP server endpoint URL |
| `transport_type` | string | Yes | Transport protocol (`streamable_http`, `sse`, `websocket`) |
| `description` | string | Yes | Human-readable description |
| `logo` | string | Yes | URL to server logo/icon |
| `categories` | string[] | Yes | Categories for organization |
| `auth_configs` | array | Yes | Supported authentication methods |
| `server_metadata` | object | Yes | Additional metadata |

### Authentication Configuration

#### No Authentication

```json
{
  "type": "no_auth"
}
```

#### API Key

```json
{
  "type": "api_key",
  "location": "header",      // "header", "query", or "cookie"
  "name": "X-API-Key",       // Header/query/cookie name
  "prefix": "Bearer"         // Optional prefix
}
```

#### OAuth2

```json
{
  "type": "oauth2",
  "location": "header",
  "name": "Authorization",
  "prefix": "Bearer",
  "authorize_url": "https://provider.com/oauth/authorize",
  "access_token_url": "https://provider.com/oauth/token",
  "scopes": ["read", "write"],
  "client_registration_url": "https://provider.com/register"  // Optional
}
```

## Tool Definition Format

### tools.json Structure

#### Remote MCP Server Tools

For remote MCP servers, tools are often discovered dynamically. However, you can provide static definitions:

```json
[
  {
    "name": "EXAMPLE__CREATE_TASK",
    "description": "Create a new task in Example",
    "input_schema": {
      "type": "object",
      "properties": {
        "title": {
          "type": "string",
          "description": "Task title"
        },
        "description": {
          "type": "string",
          "description": "Task description"
        },
        "due_date": {
          "type": "string",
          "format": "date",
          "description": "Due date (YYYY-MM-DD)"
        }
      },
      "required": ["title"]
    }
  }
]
```

#### Virtual MCP Server Tools

Virtual MCP servers require explicit tool definitions with REST API details:

```json
[
  {
    "name": "AIRTABLE__LIST_BASES",
    "description": "List all bases in Airtable",
    "tool_metadata": {
      "type": "rest",
      "method": "GET",
      "endpoint": "https://api.airtable.com/v0/meta/bases"
    },
    "input_schema": {
      "type": "object",
      "properties": {
        "header": {
          "type": "object",
          "description": "HTTP headers",
          "properties": {
            "Content-Type": {
              "type": "string",
              "default": "application/json"
            }
          },
          "required": ["Content-Type"]
        }
      }
    }
  },
  {
    "name": "AIRTABLE__CREATE_RECORD",
    "description": "Create a new record in an Airtable table",
    "tool_metadata": {
      "type": "rest",
      "method": "POST",
      "endpoint": "https://api.airtable.com/v0/{baseId}/{tableId}"
    },
    "input_schema": {
      "type": "object",
      "properties": {
        "path": {
          "type": "object",
          "properties": {
            "baseId": {
              "type": "string",
              "description": "Base ID"
            },
            "tableId": {
              "type": "string",
              "description": "Table ID or name"
            }
          },
          "required": ["baseId", "tableId"]
        },
        "body": {
          "type": "object",
          "properties": {
            "fields": {
              "type": "object",
              "description": "Record fields as key-value pairs"
            }
          },
          "required": ["fields"]
        }
      }
    }
  }
]
```

### Virtual MCP Tool Metadata

For Virtual MCP servers, `tool_metadata` specifies the REST API details:

```json
{
  "type": "rest",
  "method": "GET|POST|PUT|PATCH|DELETE",
  "endpoint": "https://api.example.com/path/{param}"
}
```

### Input Schema Sections

Virtual MCP tools support structured input schemas:

```json
{
  "type": "object",
  "properties": {
    "header": {
      "type": "object",
      "description": "HTTP headers",
      "properties": { ... }
    },
    "path": {
      "type": "object",
      "description": "Path parameters (for {param} in endpoint)",
      "properties": { ... }
    },
    "query": {
      "type": "object",
      "description": "Query string parameters",
      "properties": { ... }
    },
    "body": {
      "type": "object",
      "description": "Request body (POST/PUT/PATCH)",
      "properties": { ... }
    }
  }
}
```

## Adding a New Remote MCP Server

### Step 1: Create Directory

```bash
cd backend/mcp_servers
mkdir my_mcp_server
```

### Step 2: Create server.json

```bash
cd my_mcp_server
cat > server.json << 'EOF'
{
  "name": "MY_MCP_SERVER",
  "url": "https://mcp.myservice.com/sse",
  "transport_type": "streamable_http",
  "description": "My custom MCP server for XYZ functionality",
  "logo": "https://myservice.com/logo.svg",
  "categories": ["Custom", "Integration"],
  "auth_configs": [
    {
      "type": "api_key",
      "location": "header",
      "name": "X-API-Key"
    }
  ],
  "server_metadata": {
    "is_virtual_mcp_server": false,
    "custom_field": "optional metadata"
  }
}
EOF
```

### Step 3: (Optional) Create tools.json

If the server doesn't support dynamic tool discovery:

```bash
cat > tools.json << 'EOF'
[
  {
    "name": "MY_MCP_SERVER__MY_TOOL",
    "description": "Description of what this tool does",
    "input_schema": {
      "type": "object",
      "properties": {
        "param1": {
          "type": "string",
          "description": "Parameter description"
        }
      },
      "required": ["param1"]
    }
  }
]
EOF
```

### Step 4: Insert into Database

```bash
cd /path/to/backend

# Insert server
docker compose exec runner dotenv run python -m aci.cli mcp upsert-server \
  --server-file ./mcp_servers/my_mcp_server/server.json

# Insert tools (if you created tools.json)
docker compose exec runner dotenv run python -m aci.cli mcp upsert-tools \
  --tools-file ./mcp_servers/my_mcp_server/tools.json
```

## Adding a New Virtual MCP Server

### Step 1: Create Directory

```bash
cd backend/virtual_mcp_servers
mkdir my_api_service
```

### Step 2: Create server.json

```bash
cd my_api_service
cat > server.json << 'EOF'
{
  "name": "MY_API_SERVICE",
  "description": "My REST API service converted to MCP"
}
EOF
```

Note: For virtual servers, only `name` and `description` are in the server.json. The full configuration is generated automatically.

### Step 3: Create tools.json

Define REST API endpoints as MCP tools:

```bash
cat > tools.json << 'EOF'
[
  {
    "name": "MY_API_SERVICE__GET_ITEMS",
    "description": "Get list of items from My API Service",
    "tool_metadata": {
      "type": "rest",
      "method": "GET",
      "endpoint": "https://api.myservice.com/v1/items"
    },
    "input_schema": {
      "type": "object",
      "properties": {
        "header": {
          "type": "object",
          "properties": {
            "Content-Type": {
              "type": "string",
              "default": "application/json"
            }
          },
          "required": ["Content-Type"]
        },
        "query": {
          "type": "object",
          "properties": {
            "limit": {
              "type": "integer",
              "description": "Number of items to return",
              "default": 10
            },
            "offset": {
              "type": "integer",
              "description": "Number of items to skip",
              "default": 0
            }
          }
        }
      }
    }
  },
  {
    "name": "MY_API_SERVICE__CREATE_ITEM",
    "description": "Create a new item in My API Service",
    "tool_metadata": {
      "type": "rest",
      "method": "POST",
      "endpoint": "https://api.myservice.com/v1/items"
    },
    "input_schema": {
      "type": "object",
      "properties": {
        "header": {
          "type": "object",
          "properties": {
            "Content-Type": {
              "type": "string",
              "default": "application/json"
            }
          },
          "required": ["Content-Type"]
        },
        "body": {
          "type": "object",
          "properties": {
            "name": {
              "type": "string",
              "description": "Item name"
            },
            "description": {
              "type": "string",
              "description": "Item description"
            },
            "price": {
              "type": "number",
              "description": "Item price"
            }
          },
          "required": ["name", "price"]
        }
      }
    }
  }
]
EOF
```

### Step 4: Insert into Database

```bash
cd /path/to/backend

# Insert server
docker compose exec runner dotenv run python -m aci.cli virtual-mcp upsert-server \
  --server-file ./virtual_mcp_servers/my_api_service/server.json

# Insert tools
docker compose exec runner dotenv run python -m aci.cli virtual-mcp upsert-tools \
  --tools-file ./virtual_mcp_servers/my_api_service/tools.json
```

## Tool Naming Convention

Follow this naming pattern for consistency:

```
{SERVER_NAME}__{TOOL_ACTION}
```

Examples:
- `AIRTABLE__LIST_BASES`
- `AIRTABLE__CREATE_RECORD`
- `GITHUB__CREATE_ISSUE`
- `NOTION__GET_PAGE`

## Categories

Use consistent categories for better organization:

- **Productivity**: Task management, notes, calendars
- **Communication**: Email, chat, messaging
- **Data & Analytics**: Databases, BI tools, data platforms
- **Development**: Code repositories, CI/CD, IDEs
- **Document Management**: PDFs, documents, file storage
- **CRM**: Customer relationship management
- **Marketing**: Marketing automation, email campaigns
- **Finance**: Invoicing, payments, accounting
- **Custom**: Internal or specialized tools

## Advanced: Dynamic Tool Discovery

For remote MCP servers that support dynamic tool discovery, Gate22 can automatically fetch available tools:

```python
# This happens automatically when tools are not pre-defined

async def discover_tools(server_url: str, auth: MCPAuthManager):
    """Discover tools from remote MCP server"""
    async with mcp.ClientSession(server_url, auth=auth) as session:
        # List tools from remote server
        result = await session.list_tools()
        
        # Store in database
        for tool in result.tools:
            await mcp_tool_service.upsert(session, {
                "name": tool.name,
                "description": tool.description,
                "input_schema": tool.inputSchema,
                "mcp_server_id": server.id,
            })
```

## Example: Complete Airtable Virtual MCP Server

This is a real example from the Gate22 codebase:

**server.json:**
```json
{
  "name": "AIRTABLE",
  "description": "Airtable is a cloud-based platform that combines the functionality of a database with the visual interface of a spreadsheet. It allows users to organize and track information, collaborate with team members, and create custom applications."
}
```

**tools.json (excerpt):**
```json
[
  {
    "name": "AIRTABLE__LIST_BASES",
    "description": "List all bases in Airtable",
    "tool_metadata": {
      "type": "rest",
      "method": "GET",
      "endpoint": "https://api.airtable.com/v0/meta/bases"
    },
    "input_schema": {
      "type": "object",
      "properties": {
        "header": {
          "type": "object",
          "description": "Headers for the http request",
          "properties": {
            "Content-Type": {
              "type": "string",
              "description": "Content-Type header",
              "default": "application/json"
            }
          },
          "required": ["Content-Type"]
        }
      }
    }
  }
]
```

## Bulk Import Script

For adding multiple servers at once:

```bash
#!/bin/bash

# Import all MCP servers
for server_dir in ./mcp_servers/*/; do
  server_file="${server_dir}server.json"
  tools_file="${server_dir}tools.json"
  
  if [ -f "$server_file" ]; then
    docker compose exec runner dotenv run python -m aci.cli mcp upsert-server \
      --server-file "$server_file"
    
    if [ -f "$tools_file" ]; then
      docker compose exec runner dotenv run python -m aci.cli mcp upsert-tools \
        --tools-file "$tools_file"
    fi
  fi
done

# Import all Virtual MCP servers
for server_dir in ./virtual_mcp_servers/*/; do
  server_file="${server_dir}server.json"
  tools_file="${server_dir}tools.json"
  
  if [ -f "$server_file" ]; then
    docker compose exec runner dotenv run python -m aci.cli virtual-mcp upsert-server \
      --server-file "$server_file"
    
    if [ -f "$tools_file" ]; then
      docker compose exec runner dotenv run python -m aci.cli virtual-mcp upsert-tools \
        --tools-file "$tools_file"
    fi
  fi
done
```

## Testing Your MCP Server

### 1. Verify Server in Database

```bash
docker compose exec runner dotenv run python -m aci.cli mcp list-servers
```

### 2. Test in Frontend

1. Log into the Gate22 portal
2. Navigate to "MCP Servers"
3. Find your server in the list
4. Click to view details

### 3. Create Configuration

1. Click "Add Configuration"
2. Set up authentication
3. Select allowed tools
4. Save configuration

### 4. Test in Bundle

1. Create a new bundle
2. Add your MCP configuration
3. Copy the bundle endpoint
4. Test with an MCP client

## Common Issues & Solutions

### Issue: Server Not Appearing

**Solution:** Check that:
- `server.json` is valid JSON
- Server name is unique
- Import command completed successfully

### Issue: Tools Not Loading

**Solution:** 
- For Remote MCP: Verify server URL is accessible
- For Virtual MCP: Check tool metadata format
- Verify authentication configuration

### Issue: Authentication Failing

**Solution:**
- Verify auth_config matches server requirements
- Check credential configuration in frontend
- Review authentication logs

## Next Steps

- [Backend Architecture](../backend/overview.md)
- [Authentication & Authorization](../architecture/authentication.md)
- [API Reference](../api-reference.md)
