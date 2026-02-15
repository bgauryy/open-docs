# Claude Agent SDK - MCP Integration Complete Reference

**SDK Version**: 0.1.22
**Claude Code Runtime**: v2.1.42
**Primary Sources**:
- SDK types: `sdkTypes.d.ts`
- Claude Code runtime (v2.1.42): `@anthropic-ai/claude-code/cli.js` (distributed bundle)

Note: MCP behavior at runtime is determined by the Claude Code executable you run (see `comprehensive-guide.md` for `pathToClaudeCodeExecutable`).

---

## Table of Contents

1. [Overview](#overview)
2. [Claude Code MCP CLI (mcp-cli)](#claude-code-mcp-cli-mcp-cli)
3. [Claude Code MCP Management (claude mcp and /mcp)](#claude-code-mcp-management-claude-mcp-and-mcp)
4. [MCP Server Types](#mcp-server-types)
5. [Transport Mechanisms](#transport-mechanisms)
6. [Server Configuration](#server-configuration)
7. [SDK MCP Server Creation](#sdk-mcp-server-creation)
8. [MCP Tools Integration](#mcp-tools-integration)
9. [MCP Resources](#mcp-resources)
10. [Server Lifecycle](#server-lifecycle)
11. [Real-World Examples](#real-world-examples)
12. [Gotchas & Best Practices](#gotchas--best-practices)

---

## Overview

The Model Context Protocol (MCP) allows Claude Agent SDK to integrate with external tools and servers. The SDK supports **4 transport types** and provides both external server integration and in-process SDK server creation.

### Key Concepts

```
Claude Agent SDK
     ↓
MCP Client (built-in)
     ↓
     ├─► stdio Transport → External process
     ├─► SSE Transport → HTTP Server-Sent Events
     ├─► HTTP Transport → Standard HTTP
     └─► SDK Transport → In-process (same runtime)
```

**Benefits**:
- **External Tool Integration**: Connect to any MCP server
- **In-Process Tools**: Zero IPC overhead with SDK transport
- **Protocol Standardization**: Consistent tool interface
- **Resource Management**: Access external resources

---

## Claude Code MCP CLI (`mcp-cli`)

Claude Code v2.1.42 includes a CLI surface for inspecting and invoking MCP servers and tools. In Claude Code, this is typically accessed as an entry-mode command:

```bash
claude --mcp-cli <command> [args]
```

Commands (v2.1.42):

1. `servers` — List connected MCP servers  
   - `claude --mcp-cli servers [--json]`

2. `tools [server]` — List available tools (optionally filter by server)  
   - `claude --mcp-cli tools`  
   - `claude --mcp-cli tools filesystem`

3. `info <server>/<tool>` — Show tool description and input schema  
   - `claude --mcp-cli info my-server/my-tool`

4. `call <server>/<tool> <args>` — Invoke a tool with JSON args (or `-` to read JSON from stdin)  
   - `claude --mcp-cli call my-server/my-tool '{\"key\":\"value\"}'`  
   - `echo '{\"key\":\"value\"}' | claude --mcp-cli call my-server/my-tool -`
   - Options: `--json`, `--timeout <ms>`, `--debug`

5. `grep <pattern>` — Regex search tool names/descriptions  
   - `claude --mcp-cli grep postgres`
   - Options: `--json`, `--ignore-case` (default: true)

6. `resources [server]` — List MCP resources  
   - `claude --mcp-cli resources`  
   - `claude --mcp-cli resources filesystem`

7. `read <resource> [uri]` — Read a resource  
   - `claude --mcp-cli read my-server/my-resource-name`  
   - `claude --mcp-cli read my-server file:///path/to/file`
   - Options: `--json`, `--timeout <ms>`, `--debug`

Timeout notes:
- `--timeout` defaults to `MCP_TOOL_TIMEOUT` (and is effectively “very large” by default in v2.1.42).
- Separate from runtime MCP request timeouts used inside an interactive session (`MCP_TIMEOUT`).

---

## Claude Code MCP Management (`claude mcp` and `/mcp`)

Claude Code has two complementary CLI surfaces for MCP:

- `claude mcp ...` configures MCP servers (persisted to disk).
- `claude --mcp-cli ...` inspects and invokes MCP tools/resources for a running session (reads the session’s MCP state/endpoint).

Inside an interactive session, use `/mcp` to manage connections and authentication prompts (e.g., connect/disconnect a server or complete OAuth).

### `claude mcp` subcommands (v2.1.42)

- `claude mcp list` — list configured servers and check health.
- `claude mcp get <name>` — show the resolved config and status for one server.
- `claude mcp add <name> <commandOrUrl> [args...]` — add a server (stdio/http/sse) with options for `--scope`, `--transport`, headers, env, and OAuth.
- `claude mcp add-json <name> <json>` — add a server using a JSON string (stdio/http/sse).
- `claude mcp add-from-claude-desktop` — import servers from Claude Desktop (Mac and WSL only).
- `claude mcp remove <name> [-s <scope>]` — remove a server (or disambiguate when the same name exists in multiple scopes).
- `claude mcp reset-project-choices` — reset per-user approvals/rejections for project-scoped `.mcp.json` servers in this project.
- `claude mcp serve` — start the Claude Code MCP server (used for certain integrations).

### Scopes and where they live

In Claude Code v2.1.42, “project-scoped MCP servers” are stored in `.mcp.json` (not in `.claude/settings.json`).

| Scope | Where it is stored | Intended use |
|---|---|---|
| `local` | `.claude/settings.local.json` | Private to you in this working directory |
| `user` | `~/.claude/settings.json` | Personal defaults across projects |
| `project` | `.mcp.json` | Shared in the repo, but requires per-user approval |

---

## MCP Server Types

### Type Hierarchy (Extracted from Source)

```typescript
// From sdkTypes.d.ts
export type McpServerConfig = 
  | McpStdioServerConfig      // External process (stdio)
  | McpSSEServerConfig        // HTTP Server-Sent Events
  | McpHttpServerConfig       // Standard HTTP
  | McpSdkServerConfigWithInstance;  // In-process SDK

export type McpServerConfigForProcessTransport = 
  | McpStdioServerConfig 
  | McpSSEServerConfig 
  | McpHttpServerConfig 
  | McpSdkServerConfig;
```

### Comparison Matrix

| Transport | IPC | Latency | Use Case | Process Boundary |
|-----------|-----|---------|----------|------------------|
| **stdio** | stdin/stdout | ~5-20ms | External CLIs | Separate process |
| **SSE** | HTTP stream | ~10-50ms | Remote servers | Network/remote |
| **HTTP** | HTTP req/res | ~10-50ms | REST APIs | Network/remote |
| **SDK** | Direct call | ~0.1-1ms | Custom tools | Same process |

---

## Transport Mechanisms

### 1. stdio Transport

**Definition** (from source):
```typescript
export type McpStdioServerConfig = {
  type?: 'stdio';
  command: string;
  args?: string[];
  env?: Record<string, string>;
};
```

**Characteristics**:
- **Process**: Spawns external process
- **Communication**: stdin/stdout pipes
- **Overhead**: ~5-20ms per call (process communication)
- **Use Case**: CLI tools, npm packages, external executables

**Configuration Example**:
```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "env": {
        "NODE_ENV": "production"
      }
    }
  }
}
```

**Programmatic Configuration**:
```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const result = await query({
  prompt: "List files in /workspace",
  options: {
    mcpServers: {
      'filesystem': {
        type: 'stdio',
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-filesystem', '/workspace']
      }
    }
  }
});
```

**Process Lifecycle**:
```
1. SDK spawns process: `npx -y @modelcontextprotocol/...`
2. Process starts, initializes MCP server
3. SDK communicates via stdin/stdout
4. Process kept alive for session duration
5. Process terminated on session end
```

---

### 2. SSE Transport (Server-Sent Events)

**Definition** (from source):
```typescript
export type McpSSEServerConfig = {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
};
```

**Characteristics**:
- **Protocol**: HTTP with SSE for streaming
- **Communication**: Long-polling, server push
- **Overhead**: ~10-50ms (network latency)
- **Use Case**: Remote servers, cloud services

**Configuration Example**:
```json
{
  "mcpServers": {
    "remote-api": {
      "type": "sse",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_TOKEN}",
        "X-Custom-Header": "value"
      }
    }
  }
}
```

**Environment Variable Substitution**:
```json
{
  "headers": {
    "Authorization": "Bearer ${API_TOKEN}"
  }
}
```
- `${VAR_NAME}` → Substituted from environment
- Supports `${VAR_NAME}` and `${VAR_NAME:-default}` (default used when the env var is missing)

---

### 3. HTTP Transport

**Definition** (from source):
```typescript
export type McpHttpServerConfig = {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
};
```

**Characteristics**:
- **Protocol**: Standard HTTP request/response
- **Communication**: Synchronous HTTP calls
- **Overhead**: ~10-50ms (network latency)
- **Use Case**: REST APIs, standard web services

**Configuration Example**:
```json
{
  "mcpServers": {
    "weather-api": {
      "type": "http",
      "url": "https://weather.example.com/mcp",
      "headers": {
        "API-Key": "${WEATHER_API_KEY}",
        "Content-Type": "application/json"
      }
    }
  }
}
```

**HTTP vs SSE**:
- **HTTP**: Request → Response (one-shot)
- **SSE**: Long-lived connection with server push

---

### 4. SDK Transport (In-Process)

**Definition** (from source):
```typescript
export type McpSdkServerConfig = {
  type: 'sdk';
  name: string;
};

export type McpSdkServerConfigWithInstance = McpSdkServerConfig & {
  instance: McpServer;
};
```

**Characteristics**:
- **Process**: Same process as SDK
- **Communication**: Direct function calls (no IPC)
- **Overhead**: ~0.1-1ms (native function call)
- **Use Case**: Custom tools, high-performance, zero latency

**Creation Function** (from source):
```typescript
export declare function createSdkMcpServer(
  options: CreateSdkMcpServerOptions
): McpSdkServerConfigWithInstance;

type CreateSdkMcpServerOptions = {
  name: string;
  version?: string;
  tools?: Array<SdkMcpToolDefinition<any>>;
};
```

**Example**:
```typescript
import { createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

// Define custom tool
const customTool = tool(
  'calculate',
  'Perform mathematical calculations',
  {
    expression: z.string().describe('Mathematical expression to evaluate')
  },
  async (args) => {
    try {
      const result = eval(args.expression); // ⚠️ Example only - don't use eval in production!
      return {
        content: [{ type: 'text', text: String(result) }],
        isError: false
      };
    } catch (error) {
      return {
        content: [{ type: 'text', text: `Error: ${error.message}` }],
        isError: true
      };
    }
  }
);

// Create SDK MCP server
const calculatorServer = createSdkMcpServer({
  name: 'calculator',
  version: '1.0.0',
  tools: [customTool]
});

// Use in query
const result = await query({
  prompt: "Calculate 42 * 1337",
  options: {
    mcpServers: {
      'calculator': calculatorServer
    }
  }
});
```

**Performance Comparison**:
```
stdio transport:  ~15ms per tool call
SSE transport:    ~30ms per tool call
HTTP transport:   ~25ms per tool call
SDK transport:    ~0.5ms per tool call  ← 30-60x faster!
```

---

## Server Configuration

### Configuration Sources (Claude Code runtime)

Claude Code v2.1.42 can load MCP servers from several sources:

```
1. CLI flags: --mcp-config (dynamic), optionally with --strict-mcp-config
2. Local settings: .claude/settings.local.json
3. User settings: ~/.claude/settings.json
4. Project config: .mcp.json (shared, requires approval)
5. Plugins: enabled plugins can contribute MCP servers (and MCP bundles)
6. Enterprise: managed-mcp.json (exclusive control when present)
```

Notes:
- `.claude/settings.json` is a project settings file, but MCP servers are not read from it in v2.1.42 (use `.mcp.json` instead for project-shared servers).
- When an enterprise MCP config is present, Claude Code disallows `--strict-mcp-config` and blocks most dynamic MCP configuration.

### File / flag schema

#### Local + user settings

`~/.claude/settings.json` (user scope) and `.claude/settings.local.json` (local scope) can include:

```typescript
{
  "mcpServers": {
    "<server-name>": McpServerConfig
  }
}
```

#### Project-shared `.mcp.json`

`.mcp.json` must include a top-level `mcpServers` map:

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    }
  }
}
```

Servers from `.mcp.json` are gated by an approval step (per user, per project). Until approved, they remain pending and will not be used.

#### CLI: `--mcp-config`

Each `--mcp-config` argument is either:
- a JSON string containing `{ "mcpServers": { ... } }`, or
- a file path to a JSON file containing `{ "mcpServers": { ... } }`.

Claude Code merges multiple `--mcp-config` arguments left-to-right (later entries override earlier ones) and marks these servers as `scope: "dynamic"`.

### Precedence (when enterprise MCP config is not active)

At runtime, Claude Code merges MCP servers roughly in this order (later sources override earlier ones when names collide):

1. Plugin-provided MCP servers
2. User-scoped MCP servers (`~/.claude/settings.json`)
3. Approved project MCP servers (`.mcp.json`)
4. Local MCP servers (`.claude/settings.local.json`)

After merging, enterprise allow/deny policy filters may still block servers globally.

### Complete example (user-scoped)

**File**: `~/.claude/settings.json`

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"],
      "env": {
        "LOG_LEVEL": "info"
      }
    },
    
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    
    "database": {
      "type": "http",
      "url": "https://db-proxy.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${DB_TOKEN}"
      }
    }
  }
}
```

Note: The SDK “in-process” transport (`type: "sdk"`) is programmatic-only (it requires an in-memory server instance). It is not something Claude Code can load from JSON files.

---

## SDK MCP Server Creation

### Tool Definition Interface (from source)

```typescript
type SdkMcpToolDefinition<Schema extends ZodRawShape = ZodRawShape> = {
  name: string;
  description: string;
  inputSchema: Schema;
  handler: (
    args: z.infer<ZodObject<Schema>>,
    extra: unknown
  ) => Promise<CallToolResult>;
};
```

### Tool Creation Function (from source)

```typescript
export declare function tool<Schema extends ZodRawShape>(
  name: string,
  description: string,
  inputSchema: Schema,
  handler: (
    args: z.infer<ZodObject<Schema>>,
    extra: unknown
  ) => Promise<CallToolResult>
): SdkMcpToolDefinition<Schema>;
```

### Complete Example: Custom Tool Suite

```typescript
import { createSdkMcpServer, tool, query } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

// Tool 1: Date/Time
const dateTimeTool = tool(
  'get_datetime',
  'Get current date and time in specified timezone',
  {
    timezone: z.string()
      .optional()
      .describe('IANA timezone (e.g., America/New_York)')
  },
  async (args) => {
    const tz = args.timezone || 'UTC';
    const now = new Date().toLocaleString('en-US', { timeZone: tz });
    
    return {
      content: [{ 
        type: 'text', 
        text: `Current time in ${tz}: ${now}` 
      }],
      isError: false
    };
  }
);

// Tool 2: UUID Generator
const uuidTool = tool(
  'generate_uuid',
  'Generate a random UUID',
  {},
  async () => {
    const uuid = crypto.randomUUID();
    
    return {
      content: [{ 
        type: 'text', 
        text: uuid 
      }],
      isError: false
    };
  }
);

// Tool 3: Hash String
const hashTool = tool(
  'hash_string',
  'Hash a string using specified algorithm',
  {
    text: z.string().describe('Text to hash'),
    algorithm: z.enum(['sha256', 'sha512', 'md5'])
      .optional()
      .describe('Hash algorithm')
  },
  async (args) => {
    const crypto = await import('crypto');
    const algo = args.algorithm || 'sha256';
    const hash = crypto.createHash(algo).update(args.text).digest('hex');
    
    return {
      content: [{ 
        type: 'text', 
        text: `${algo}: ${hash}` 
      }],
      isError: false
    };
  }
);

// Create MCP server with all tools
const utilsServer = createSdkMcpServer({
  name: 'utils',
  version: '1.0.0',
  tools: [dateTimeTool, uuidTool, hashTool]
});

// Use in conversation
const result = await query({
  prompt: "What's the current time in Tokyo and generate a UUID",
  options: {
    mcpServers: {
      'utils': utilsServer
    }
  }
});
```

---

## MCP Tools Integration

### Tool Discovery

MCP tools are automatically discovered and made available:

```
1. SDK connects to MCP server
2. Server advertises available tools (via MCP protocol)
3. SDK registers tools with AI model
4. Model can invoke tools by name
```

### Tool Input Schema (from source)

```typescript
export interface McpInput {
  [k: string]: unknown;
}
```

**Note**: MCP tool inputs are dynamic (any valid JSON object).

### Tool Invocation Flow

```
1. Model decides to use MCP tool
2. SDK checks permissions
3. SDK sends tool request to MCP server
4. MCP server executes tool
5. Server returns result
6. SDK formats result for model
7. Model continues with result
```

---

## MCP Resources

### Resource Types (from source)

```typescript
// List all resources from MCP servers
export interface ListMcpResourcesInput {
  /** Optional server name to filter resources by */
  server?: string;
}

// Read specific resource
export interface ReadMcpResourceInput {
  /** The MCP server name */
  server: string;
  /** The resource URI to read */
  uri: string;
}
```

### Resource Access

**List Resources**:
```typescript
// List all resources from all servers
ListMcpResources({})

// List resources from specific server
ListMcpResources({ server: "filesystem" })
```

**Read Resource**:
```typescript
// Read specific resource
ReadMcpResource({
  server: "github",
  uri: "github://repo/owner/name/issues/123"
})
```

### Resource URIs

Resource URIs follow server-specific patterns:

```
filesystem://path/to/file
github://repo/owner/name/issues/123
database://table/users/query
```

---

## Server Lifecycle

### Connection States

```typescript
export type McpServerStatus = {
  name: string;
  status: 'connected' | 'failed' | 'needs-auth' | 'pending';
  serverInfo?: {
    name: string;
    version: string;
  };
};
```

### Lifecycle Stages

```
1. Configuration → Server config loaded from settings
2. Initialization → SDK attempts to connect
3. Connected → Server ready (tools/resources available)
4. Failed → Connection error (check logs)
5. needs-auth → Authentication required
6. pending → Connection in progress
```

### Checking Server Status

```typescript
const query = await query({
  prompt: "Hello",
  options: { mcpServers: { /* config */ } }
});

// Check server status
const status = await query.mcpServerStatus();
console.log(status);
// [
//   { name: 'filesystem', status: 'connected', serverInfo: { name: '...', version: '1.0.0' } },
//   { name: 'github', status: 'failed' },
//   { name: 'custom', status: 'connected' }
// ]
```

### Health Monitoring

```typescript
// Periodic health check
setInterval(async () => {
  const status = await query.mcpServerStatus();
  
  const failed = status.filter(s => s.status === 'failed');
  if (failed.length > 0) {
    console.error('Failed servers:', failed.map(s => s.name));
  }
}, 30000); // Every 30 seconds
```

---

## Real-World Examples

### Example 1: Database Integration

```typescript
import { createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';
import { Pool } from 'pg';

// Database connection pool
const pool = new Pool({
  connectionString: process.env.DATABASE_URL
});

// SQL Query Tool
const queryTool = tool(
  'sql_query',
  'Execute SQL query (read-only)',
  {
    query: z.string().describe('SQL SELECT query'),
    limit: z.number().optional().describe('Row limit (max 100)')
  },
  async (args) => {
    // Safety: Only allow SELECT
    if (!args.query.trim().toLowerCase().startsWith('select')) {
      return {
        content: [{ type: 'text', text: 'Error: Only SELECT queries allowed' }],
        isError: true
      };
    }
    
    try {
      const limit = Math.min(args.limit || 50, 100);
      const result = await pool.query(`${args.query} LIMIT ${limit}`);
      
      return {
        content: [{
          type: 'text',
          text: JSON.stringify(result.rows, null, 2)
        }],
        isError: false
      };
    } catch (error) {
      return {
        content: [{ type: 'text', text: `SQL Error: ${error.message}` }],
        isError: true
      };
    }
  }
);

// Create database MCP server
const dbServer = createSdkMcpServer({
  name: 'database',
  version: '1.0.0',
  tools: [queryTool]
});

// Use in query
const result = await query({
  prompt: "Show me the 10 most recent users",
  options: {
    mcpServers: { 'database': dbServer }
  }
});
```

---

### Example 2: External API Integration (HTTP)

```json
{
  "mcpServers": {
    "weather": {
      "type": "http",
      "url": "https://weather-mcp.example.com",
      "headers": {
        "API-Key": "${WEATHER_API_KEY}"
      }
    },
    
    "stock-market": {
      "type": "http",
      "url": "https://stocks-mcp.example.com",
      "headers": {
        "Authorization": "Bearer ${STOCK_API_TOKEN}"
      }
    }
  }
}
```

**Usage**:
```bash
> What's the weather in Tokyo and the current price of AAPL stock?

# Agent automatically uses both MCP servers
# 1. weather MCP server → Get Tokyo weather
# 2. stock-market MCP server → Get AAPL price
# 3. Combines and presents results
```

---

### Example 3: Multi-Server Workflow

```typescript
const result = await query({
  prompt: `
    1. List files in /workspace (filesystem server)
    2. Check GitHub issues (github server)
    3. Calculate total open issues (custom calculator)
  `,
  options: {
    mcpServers: {
      'filesystem': {
        type: 'stdio',
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-filesystem', '/workspace']
      },
      'github': {
        type: 'stdio',
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-github'],
        env: {
          'GITHUB_TOKEN': process.env.GITHUB_TOKEN
        }
      },
      'calculator': createSdkMcpServer({
        name: 'calculator',
        tools: [/* calculator tools */]
      })
    }
  }
});
```

---

## Gotchas & Best Practices

### Gotchas

1. **Timeouts depend on which surface you’re using**:
   - `claude --mcp-cli call/read` supports `--timeout <ms>` (default: `MCP_TOOL_TIMEOUT`).
   - In-session MCP requests and connection behavior are runtime-defined and can be controlled via environment variables (see `extraction/cli-internal-constants.md` for the authoritative list).

2. **Environment Variable Substitution**:
   ```json
   { "env": { "TOKEN": "${GITHUB_TOKEN}" } }
   ```
   - Missing variables typically produce warnings during config load (not hard failures).
   - `${VAR}` and `${VAR:-default}` are supported in MCP configs that expand vars.

3. **SDK Server Instance Reuse**:
   ```typescript
   // ❌ Wrong: Creating new instance each time
   await query({ options: { mcpServers: {
     'custom': createSdkMcpServer({ /* ... */ })
   }}});
   
   // ✅ Correct: Reuse instance
   const customServer = createSdkMcpServer({ /* ... */ });
   await query({ options: { mcpServers: {
     'custom': customServer
   }}});
   ```

4. **Strict MCP Config Mode**:
   ```typescript
   // Default: use MCP servers from all configured sources
   { strictMcpConfig: false }
   
   // Strict: only use MCP servers provided via --mcp-config
   { strictMcpConfig: true }
   ```
   `strictMcpConfig` is primarily for reproducibility (avoid picking up user/local/project/plugin MCP servers implicitly).

5. **MCP Tool Name Conflicts**:
   ```
   Claude Code namespaces MCP tools (e.g. `mcp__<server>__<tool>`), so they do not collide with built-in tool names like `Read` or `Edit`.
   ```

### Best Practices

**1. Use SDK Transport for Performance**:
```typescript
// ✅ Best: SDK transport (0.5ms)
const server = createSdkMcpServer({ /* ... */ });

// 🟡 Okay: stdio transport (15ms)
{ type: 'stdio', command: '...' }

// 🔴 Slow: HTTP transport over network (50ms+)
{ type: 'http', url: 'https://...' }
```

**2. Environment Variables for Secrets**:
```json
{
  "mcpServers": {
    "api": {
      "env": {
        "API_KEY": "${SECRET_KEY}"  // ✅ Never commit actual keys
      }
    }
  }
}
```

**3. Error Handling in Custom Tools**:
```typescript
tool('my_tool', 'Description', schema, async (args) => {
  try {
    // ... tool logic
    return { content: [{ type: 'text', text: result }], isError: false };
  } catch (error) {
    return {
      content: [{ type: 'text', text: `Error: ${error.message}` }],
      isError: true  // ✅ Mark as error
    };
  }
});
```

**4. Monitor Server Health**:
```typescript
// Check status periodically
const status = await query.mcpServerStatus();
const failed = status.filter(s => s.status === 'failed');

if (failed.length > 0) {
  console.error('Reconnecting failed servers...');
  // Implement reconnection logic
}
```

**5. Use strictMcpConfig in Production**:
```typescript
// Development: lenient
{ strictMcpConfig: false }

// Production: strict
{ strictMcpConfig: true }  // Prefer deterministic MCP source set
```

---

## Summary

### MCP Transport Comparison

| Feature | stdio | SSE | HTTP | SDK |
|---------|-------|-----|------|-----|
| **Latency** | ~15ms | ~30ms | ~25ms | ~0.5ms |
| **Process** | External | Remote | Remote | Same |
| **Use Case** | CLI tools | Cloud services | REST APIs | Custom tools |
| **Overhead** | Medium | High | High | Minimal |
| **Best For** | npm packages | Real-time | Standard APIs | Performance |

### Configuration Checklist

- [ ] Choose appropriate transport type
- [ ] Add server via `claude mcp add` (or project-shared `.mcp.json` / `--mcp-config`)
- [ ] Set environment variables for secrets
- [ ] Test server connection (`claude mcp list`, `/mcp`, or `query.mcpServerStatus()`)
- [ ] Monitor server health
- [ ] Handle errors gracefully
- [ ] Use SDK transport for performance-critical tools
- [ ] Consider strict MCP config for reproducible runs

### Key Takeaways

- ✅ **4 transport types**: stdio, SSE, HTTP, SDK
- ✅ **SDK transport**: 30-60x faster (in-process)
- ✅ **stdio transport**: Best for npm packages and CLIs
- ✅ **HTTP/SSE**: For remote servers and cloud services
- ✅ **Custom tools**: Use `createSdkMcpServer` + `tool` functions
- ⚠️ **Environment variables**: Use `${VAR_NAME}` syntax
- ⚠️ **Project-shared servers**: `.mcp.json` servers require per-user approval before they are used
