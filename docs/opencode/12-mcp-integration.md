# OpenCode - MCP Integration

> **Model Context Protocol support for extending OpenCode with external tools and resources**

---

## Overview

**Model Context Protocol (MCP)** enables OpenCode to integrate with external servers that provide:
- **Tools** - Additional capabilities (databases, APIs, services)
- **Resources** - Data sources and context
- **Prompts** - Reusable prompt templates

**Spec**: https://modelcontextprotocol.io

**Files**:
- `mcp/index.ts` - MCP client implementation
- Configuration in `.opencode/config.json`

---

## MCP Architecture

```
OpenCode
    │
    ├─ MCP Client
    │     │
    │     ├─ Server 1 (Database)
    │     │     ├─ Tools: query, insert
    │     │     └─ Resources: schema
    │     │
    │     ├─ Server 2 (GitHub)
    │     │     ├─ Tools: create_issue
    │     │     └─ Resources: repos
    │     │
    │     └─ Server 3 (Slack)
    │           ├─ Tools: send_message
    │           └─ Resources: channels
    │
    └─ Tool Registry
          └─ Combines built-in + MCP tools
```

---

## Configuration

**.opencode/config.json**:
```json
{
  "mcp": {
    "servers": {
      "database": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-postgres"],
        "env": {
          "DATABASE_URL": "postgresql://..."
        }
      },
      "github": {
        "command": "mcp-server-github",
        "env": {
          "GITHUB_TOKEN": "${GITHUB_TOKEN}"
        }
      },
      "custom": {
        "command": "node",
        "args": ["./my-mcp-server.js"]
      }
    }
  }
}
```

---

## MCP Commands

### List Servers

```bash
opencode mcp list
```

Output:
```
Configured MCP Servers:

database (@modelcontextprotocol/server-postgres)
  Tools: query, insert, update
  Resources: schema
  Status: Running

github (mcp-server-github)
  Tools: create_issue, list_repos
  Resources: repositories
  Status: Running
```

### Add Server

```bash
opencode mcp add @modelcontextprotocol/server-postgres
```

Automatically adds to config with prompts for configuration.

### Test Server

```bash
opencode mcp test database
```

Validates connection and lists capabilities.

---

## Using MCP Tools

MCP tools are automatically available to AI:

**User**: "Query the database for active users"

**AI**:
```json
{
  "tool": "mcp_database_query",
  "parameters": {
    "sql": "SELECT * FROM users WHERE active = true"
  }
}
```

**Result**:
```
Found 42 active users:
- alice@example.com
- bob@example.com
...
```

---

## Creating MCP Servers

### TypeScript Example

```typescript
import { MCPServer } from "@modelcontextprotocol/sdk"

const server = new MCPServer({
  name: "my-service",
  version: "1.0.0"
})

// Register tool
server.addTool({
  name: "fetch_data",
  description: "Fetch data from API",
  parameters: {
    type: "object",
    properties: {
      endpoint: { type: "string" }
    }
  },
  async execute({ endpoint }) {
    const response = await fetch(`https://api.example.com/${endpoint}`)
    return await response.json()
  }
})

// Start server
server.listen()
```

### Python Example

```python
from mcp import Server, Tool

server = Server("my-service", "1.0.0")

@server.tool(
    name="process_data",
    description="Process data with Python"
)
async def process_data(data: str) -> str:
    # Process data
    return f"Processed: {data}"

if __name__ == "__main__":
    server.run()
```

---

## Available MCP Servers

**Official Servers**:
- `@modelcontextprotocol/server-postgres` - PostgreSQL
- `@modelcontextprotocol/server-sqlite` - SQLite
- `@modelcontextprotocol/server-filesystem` - File system access
- `@modelcontextprotocol/server-github` - GitHub integration
- `@modelcontextprotocol/server-slack` - Slack integration

**Community Servers**: See https://github.com/modelcontextprotocol/servers

---

## Best Practices

**Server Selection**:
- Use official servers when available
- Test thoroughly before production
- Monitor performance and errors
- Document custom servers

**Security**:
- Never commit API keys to config
- Use environment variables
- Validate all server inputs
- Limit server permissions

**Performance**:
- Keep servers lightweight
- Implement timeouts
- Cache when possible
- Monitor resource usage

---

For implementation, see `packages/opencode/src/mcp/` and spec at https://modelcontextprotocol.io.

