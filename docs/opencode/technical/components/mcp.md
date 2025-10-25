# MCP (Model Context Protocol)

**Path**: `packages/opencode/src/mcp`
**Type**: Protocol Layer
**File Count**: 1

## Description

Model Context Protocol integration for extending capabilities with external tools.

## Purpose

The MCP component enables OpenCode to connect to Model Context Protocol servers, extending its capabilities with external tools, resources, and context providers. This allows integration with specialized tools and services.

## Key Features

- MCP client implementation
- Tool discovery from MCP servers
- Resource access
- Dynamic tool registration
- Multiple server support
- Protocol version negotiation

## Component Files

- `index.ts` - MCP client implementation and integration

## External Dependencies

- `@modelcontextprotocol/sdk` - MCP protocol implementation

## Usage

### Configure MCP Servers

Create `.opencode/mcp.json`:

```json
{
  "servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "database.db"]
    }
  }
}
```

### Connect to MCP Server

```typescript
import { MCP } from './mcp';

// Connect to MCP server
const mcp = await MCP.connect({
  server: 'filesystem',
  command: 'npx',
  args: ['-y', '@modelcontextprotocol/server-filesystem', '/workspace']
});

// List available tools
const tools = await mcp.listTools();
console.log('Available tools:', tools.map(t => t.name));

// List resources
const resources = await mcp.listResources();
console.log('Available resources:', resources.map(r => r.uri));

// Call a tool
const result = await mcp.callTool('read_file', {
  path: '/workspace/file.txt'
});
console.log('File content:', result.content);

// Read a resource
const resource = await mcp.readResource('file:///workspace/config.json');
console.log('Resource:', resource);

// Disconnect
await mcp.disconnect();
```

### CLI Management

```bash
# Add MCP server interactively
opencode mcp add

# List configured servers
opencode mcp list

# Remove MCP server
opencode mcp remove filesystem

# Test MCP server connection
opencode mcp test filesystem
```

## Supported MCP Servers

### Official MCP Servers

- **@modelcontextprotocol/server-filesystem** - File system operations
- **@modelcontextprotocol/server-github** - GitHub API integration
- **@modelcontextprotocol/server-sqlite** - SQLite database access
- **@modelcontextprotocol/server-postgres** - PostgreSQL database
- **@modelcontextprotocol/server-memory** - Persistent memory

### Custom MCP Servers

You can use any MCP-compliant server:

```javascript
// custom-server.js
import { MCPServer } from '@modelcontextprotocol/sdk';

const server = new MCPServer({
  name: 'custom-server',
  version: '1.0.0'
});

server.registerTool({
  name: 'custom_operation',
  description: 'Perform custom operation',
  inputSchema: {
    type: 'object',
    properties: {
      param: { type: 'string', description: 'Parameter' }
    },
    required: ['param']
  },
  handler: async (params) => {
    return { result: 'success', data: params };
  }
});

server.listen();
```

## Tool Discovery

When connecting to an MCP server, tools are automatically discovered:

```typescript
const mcp = await MCP.connect({ server: 'filesystem', ... });
const tools = await mcp.listTools();

// Tools are formatted as:
[
  {
    name: 'read_file',
    description: 'Read contents of a file',
    inputSchema: {
      type: 'object',
      properties: {
        path: { type: 'string', description: 'File path' }
      },
      required: ['path']
    }
  },
  {
    name: 'write_file',
    description: 'Write content to a file',
    inputSchema: {
      type: 'object',
      properties: {
        path: { type: 'string' },
        content: { type: 'string' }
      },
      required: ['path', 'content']
    }
  }
]
```

## Resource Access

MCP servers can provide resources (file-like data):

```typescript
// List available resources
const resources = await mcp.listResources();

// Read a resource
const resource = await mcp.readResource('file:///workspace/data.json');
console.log('Content:', resource.contents);

// Resources support templates
const template = await mcp.readResource('template://greeting?name=World');
```

## Integration with Session

MCP tools are automatically registered with sessions:

```typescript
import { Session } from './session';
import { MCP } from './mcp';

// Create session
const session = await Session.create({
  model: 'claude-3-sonnet'
});

// Connect MCP servers
const filesystem = await MCP.connect({ server: 'filesystem', ... });
const github = await MCP.connect({ server: 'github', ... });

// Register MCP tools (automatically done)
await session.registerMCPTools(filesystem);
await session.registerMCPTools(github);

// Now AI can use MCP tools
const response = await session.message(
  'Read the README and check GitHub issues'
);
// AI can call: mcp_filesystem_read_file, mcp_github_list_issues
```

## Tool Namespacing

MCP tools are prefixed with `mcp_{server}_` to avoid conflicts:

```typescript
// filesystem server tools
mcp_filesystem_read_file
mcp_filesystem_write_file
mcp_filesystem_list_directory

// github server tools
mcp_github_list_issues
mcp_github_create_pr
mcp_github_get_repo
```

## Protocol Features

### Server Capabilities

MCP servers can provide:
- **Tools** - Executable functions
- **Resources** - File-like data sources
- **Prompts** - Reusable prompt templates
- **Sampling** - Text generation requests

### Client Capabilities

OpenCode supports:
- Tool execution
- Resource reading
- Prompt expansion
- Progress notifications
- Cancellation

## Error Handling

```typescript
try {
  const result = await mcp.callTool('read_file', {
    path: '/nonexistent.txt'
  });
} catch (error) {
  if (error.code === 'TOOL_ERROR') {
    console.error('Tool execution failed:', error.message);
  } else if (error.code === 'SERVER_DISCONNECTED') {
    console.error('MCP server disconnected');
    // Attempt reconnection
    await mcp.reconnect();
  } else if (error.code === 'TIMEOUT') {
    console.error('Request timed out');
  }
}
```

## Security Considerations

- MCP servers run in separate processes
- Sandboxed execution environment
- Configuration validation
- Environment variable protection
- Resource access control
- Per-server permission model

## Performance Considerations

- Connection pooling for multiple servers
- Lazy server initialization
- Tool result caching
- Efficient IPC communication
- Graceful degradation on failure

## Advanced Usage

### Conditional Tool Registration

```typescript
// Only register MCP tools if server is available
if (await MCP.isAvailable('filesystem')) {
  const mcp = await MCP.connect({ server: 'filesystem', ... });
  await session.registerMCPTools(mcp);
}
```

### Dynamic Server Discovery

```typescript
// Discover all configured MCP servers
const servers = await MCP.listConfigured();

// Connect to all servers
const connections = await Promise.all(
  servers.map(server => MCP.connect(server))
);

// Register all tools
for (const mcp of connections) {
  await session.registerMCPTools(mcp);
}
```

### Server Health Monitoring

```typescript
// Monitor server health
setInterval(async () => {
  const health = await mcp.healthCheck();
  if (!health.ok) {
    console.warn('MCP server unhealthy:', health.error);
    await mcp.reconnect();
  }
}, 30000);
```

## Related Documentation

- [MCP Integration Flow](../flows/mcp-integration-flow.md)
- [MCP Command API](../api-reference.md#mcpcommand)
- [Session Management](./session.md)
