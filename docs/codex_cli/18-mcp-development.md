# MCP Server Development Guide

Complete guide to building Model Context Protocol (MCP) servers for Codex, from simple tools to production-ready integrations.

---

## Overview

The Model Context Protocol (MCP) is an open standard that allows Codex to connect to external tools, data sources, and services. MCP servers expose:

- **Tools**: Functions the AI can call
- **Resources**: Data the AI can read
- **Prompts**: Reusable templates

**Official MCP Spec:** https://modelcontextprotocol.io/

---

## Quick Start: Hello World MCP Server

### 1. Create a Simple STDIO Server (Node.js)

**File:** `hello-mcp-server.js`

```javascript
#!/usr/bin/env node

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  {
    name: "hello-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Register tool
server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "greet",
        description: "Greet a person by name",
        inputSchema: {
          type: "object",
          properties: {
            name: {
              type: "string",
              description: "Name of person to greet",
            },
          },
          required: ["name"],
        },
      },
    ],
  };
});

// Handle tool calls
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "greet") {
    const personName = args.name;
    return {
      content: [
        {
          type: "text",
          text: `Hello, ${personName}! Welcome to MCP.`,
        },
      ],
    };
  }

  throw new Error(`Unknown tool: ${name}`);
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
}

main().catch((error) => {
  console.error("Server error:", error);
  process.exit(1);
});
```

### 2. Make Executable

```bash
chmod +x hello-mcp-server.js
```

### 3. Configure in Codex

Add to `~/.codex/config.toml`:

```toml
[mcp_servers.hello]
command = "node"
args = ["./hello-mcp-server.js"]
```

### 4. Test in Codex

```bash
codex

# In TUI:
# "Use the greet tool to say hello to Alice"
```

The model will call your `greet` tool with `{"name": "Alice"}`.

---

## MCP Protocol Fundamentals

### Transport Types

#### STDIO Transport

**Use case:** Local servers, simple integrations

```javascript
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Codex config:**
```toml
[mcp_servers.my_server]
command = "node"
args = ["server.js"]
```

#### Streamable HTTP Transport

**Use case:** Remote servers, OAuth-enabled services

```javascript
import { HttpServerTransport } from "@modelcontextprotocol/sdk/server/http.js";

const transport = new HttpServerTransport({
  port: 3000,
  path: "/mcp",
});
await server.connect(transport);
```

**Codex config:**
```toml
[mcp_servers.my_server]
url = "http://localhost:3000/mcp"

# Optional OAuth
experimental_use_rmcp_client = true  # In root config
bearer_token_env_var = "MY_SERVER_TOKEN"
```

---

## Tool Development

### Tool Definition

Tools are defined with JSON Schema:

```javascript
server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "search_docs",
        description: "Search technical documentation",
        inputSchema: {
          type: "object",
          properties: {
            query: {
              type: "string",
              description: "Search query",
            },
            limit: {
              type: "integer",
              description: "Max results",
              default: 10,
            },
          },
          required: ["query"],
        },
      },
    ],
  };
});
```

### Tool Execution

```javascript
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "search_docs") {
    const results = await searchDocumentation(args.query, args.limit || 10);

    return {
      content: [
        {
          type: "text",
          text: JSON.stringify(results, null, 2),
        },
      ],
    };
  }

  throw new Error(`Unknown tool: ${name}`);
});
```

### Error Handling

Always handle errors gracefully:

```javascript
server.setRequestHandler("tools/call", async (request) => {
  try {
    const { name, arguments: args } = request.params;

    if (name === "risky_operation") {
      // Validate inputs
      if (!args.required_param) {
        throw new Error("Missing required_param");
      }

      // Perform operation
      const result = await performOperation(args);

      return {
        content: [{ type: "text", text: result }],
      };
    }

    throw new Error(`Unknown tool: ${name}`);
  } catch (error) {
    // MCP error format
    return {
      content: [
        {
          type: "text",
          text: `Error: ${error.message}`,
        },
      ],
      isError: true,
    };
  }
});
```

---

## Resource Development

Resources provide read-only data to the AI.

### Resource List

```javascript
server.setRequestHandler("resources/list", async () => {
  return {
    resources: [
      {
        uri: "file:///docs/api-reference.md",
        name: "API Reference",
        description: "Complete API documentation",
        mimeType: "text/markdown",
      },
      {
        uri: "db://users/stats",
        name: "User Statistics",
        description: "Current user stats",
        mimeType: "application/json",
      },
    ],
  };
});
```

### Resource Read

```javascript
server.setRequestHandler("resources/read", async (request) => {
  const { uri } = request.params;

  if (uri === "file:///docs/api-reference.md") {
    const content = await fs.readFile("./docs/api-reference.md", "utf-8");
    return {
      contents: [
        {
          uri,
          mimeType: "text/markdown",
          text: content,
        },
      ],
    };
  }

  if (uri === "db://users/stats") {
    const stats = await getUserStats();
    return {
      contents: [
        {
          uri,
          mimeType: "application/json",
          text: JSON.stringify(stats),
        },
      ],
    };
  }

  throw new Error(`Unknown resource: ${uri}`);
});
```

---

## Advanced Features

### OAuth Integration

For servers requiring authentication:

```javascript
import { OAuthProvider } from "@modelcontextprotocol/sdk/server/oauth.js";

const oauthProvider = new OAuthProvider({
  authorizationUrl: "https://example.com/oauth/authorize",
  tokenUrl: "https://example.com/oauth/token",
  clientId: process.env.CLIENT_ID,
  clientSecret: process.env.CLIENT_SECRET,
  redirectUri: "http://localhost:3000/oauth/callback",
});

server.setOAuthProvider(oauthProvider);
```

**Codex setup:**

```toml
# In config.toml
experimental_use_rmcp_client = true

[mcp_servers.my_oauth_server]
url = "https://example.com/mcp"
```

**Usage:**

```bash
codex mcp login my_oauth_server
# Opens browser for OAuth flow
```

### Dynamic Tools

Generate tools dynamically based on environment:

```javascript
server.setRequestHandler("tools/list", async () => {
  const apiKey = process.env.API_KEY;
  const tools = [];

  // Base tools
  tools.push({
    name: "public_search",
    description: "Search public data",
    inputSchema: { /* ... */ },
  });

  // Premium tools (only if API key present)
  if (apiKey) {
    tools.push({
      name: "premium_search",
      description: "Search premium data",
      inputSchema: { /* ... */ },
    });
  }

  return { tools };
});
```

### Streaming Responses

For long-running operations:

```javascript
server.setRequestHandler("tools/call", async (request, { sendProgress }) => {
  const { name, arguments: args } = request.params;

  if (name === "long_running_task") {
    // Send progress updates
    await sendProgress({ progress: 0.25, message: "Processing..." });
    await doWork();

    await sendProgress({ progress: 0.5, message: "Halfway there..." });
    await doMoreWork();

    await sendProgress({ progress: 1.0, message: "Complete!" });

    return {
      content: [{ type: "text", text: "Task completed" }],
    };
  }
});
```

---

## Testing MCP Servers

### 1. MCP Inspector

The official MCP testing tool:

```bash
npx @modelcontextprotocol/inspector node hello-mcp-server.js
```

**Features:**
- Interactive tool testing
- Request/response logging
- Schema validation
- Timeout testing

### 2. Unit Tests

Test your handlers directly:

```javascript
import { describe, it, expect } from "vitest";
import { createMockServer } from "./test-utils.js";

describe("greet tool", () => {
  it("should greet by name", async () => {
    const server = createMockServer();

    const response = await server.callTool("greet", {
      name: "Alice",
    });

    expect(response.content[0].text).toBe("Hello, Alice! Welcome to MCP.");
  });

  it("should require name parameter", async () => {
    const server = createMockServer();

    await expect(server.callTool("greet", {})).rejects.toThrow();
  });
});
```

### 3. Integration Tests with Codex

Test in actual Codex environment:

```bash
# Add server to test config
cat > test-config.toml << EOF
[mcp_servers.test_server]
command = "node"
args = ["./hello-mcp-server.js"]
EOF

# Run Codex with test config
CODEX_HOME=./test-codex-home codex exec \
  --config "$(cat test-config.toml)" \
  "Use the greet tool to say hello to Bob"
```

---

## Production Best Practices

### 1. Error Handling

```javascript
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  try {
    // Validate required parameters
    if (name === "my_tool" && !args.required_field) {
      return {
        content: [{
          type: "text",
          text: "Error: required_field is missing"
        }],
        isError: true
      };
    }

    // Execute with timeout
    const result = await Promise.race([
      executeTool(name, args),
      timeout(30000, "Tool execution timed out")
    ]);

    return result;
  } catch (error) {
    // Log for debugging
    console.error(`Tool ${name} failed:`, error);

    // Return user-friendly error
    return {
      content: [{
        type: "text",
        text: `Tool execution failed: ${error.message}`
      }],
      isError: true
    };
  }
});
```

### 2. Timeout Handling

```javascript
function timeout(ms, message) {
  return new Promise((_, reject) => {
    setTimeout(() => reject(new Error(message)), ms);
  });
}

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  const result = await Promise.race([
    executeTool(name, args),
    timeout(60000, "Execution timed out after 60s"),
  ]);

  return result;
});
```

Configure in Codex:

```toml
[mcp_servers.my_server]
command = "node"
args = ["server.js"]
tool_timeout_sec = 60  # Match server timeout
```

### 3. Logging

```javascript
import winston from "winston";

const logger = winston.createLogger({
  level: "info",
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: "mcp-server.log" }),
  ],
});

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  logger.info("Tool called", { name, args });

  try {
    const result = await executeTool(name, args);
    logger.info("Tool succeeded", { name, result });
    return result;
  } catch (error) {
    logger.error("Tool failed", { name, error: error.message });
    throw error;
  }
});
```

### 4. Rate Limiting

```javascript
import rateLimit from "express-rate-limit";

const limiter = new Map();

function checkRateLimit(toolName) {
  const now = Date.now();
  const key = toolName;

  if (!limiter.has(key)) {
    limiter.set(key, { count: 0, resetAt: now + 60000 });
  }

  const limit = limiter.get(key);

  if (now > limit.resetAt) {
    limit.count = 0;
    limit.resetAt = now + 60000;
  }

  if (limit.count >= 10) {
    throw new Error("Rate limit exceeded: 10 calls per minute");
  }

  limit.count++;
}

server.setRequestHandler("tools/call", async (request) => {
  const { name } = request.params;

  checkRateLimit(name);

  // Execute tool...
});
```

### 5. Input Validation

```javascript
import Ajv from "ajv";

const ajv = new Ajv();

const schemas = {
  search_docs: {
    type: "object",
    properties: {
      query: { type: "string", minLength: 1, maxLength: 500 },
      limit: { type: "integer", minimum: 1, maximum: 100 },
    },
    required: ["query"],
  },
};

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  // Validate against schema
  const schema = schemas[name];
  if (schema && !ajv.validate(schema, args)) {
    throw new Error(`Invalid arguments: ${ajv.errorsText()}`);
  }

  // Execute tool...
});
```

---

## Deployment

### Local Development

```bash
# Use absolute path in config
[mcp_servers.dev_server]
command = "node"
args = ["/absolute/path/to/server.js"]
cwd = "/absolute/path/to/project"
```

### Docker Container

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production

COPY . .

CMD ["node", "server.js"]
```

**Codex config:**

```toml
[mcp_servers.docker_server]
command = "docker"
args = ["run", "--rm", "-i", "my-mcp-server:latest"]
```

### Remote Server (Streamable HTTP)

```javascript
import express from "express";
import { HttpServerTransport } from "@modelcontextprotocol/sdk/server/http.js";

const app = express();

const transport = new HttpServerTransport({
  app,
  path: "/mcp",
  port: 3000,
});

await server.connect(transport);

app.listen(3000, () => {
  console.log("MCP server listening on :3000/mcp");
});
```

**Codex config:**

```toml
[mcp_servers.remote_server]
url = "https://my-server.example.com/mcp"
bearer_token_env_var = "MCP_AUTH_TOKEN"
```

---

## Debugging

### Enable MCP Logging in Codex

```bash
export RUST_LOG=codex_core=debug,rmcp_client=debug
codex
```

Logs go to `~/.codex/log/codex-tui.log`:

```bash
tail -F ~/.codex/log/codex-tui.log | grep mcp
```

### MCP Server Logging

Log to stderr (appears in Codex logs):

```javascript
console.error("MCP Server: Tool called:", name);
```

Or to a file:

```javascript
import fs from "fs";

const log = fs.createWriteStream("mcp-server.log", { flags: "a" });

function logInfo(message, data) {
  log.write(`${new Date().toISOString()} INFO ${message} ${JSON.stringify(data)}\n`);
}

server.setRequestHandler("tools/call", async (request) => {
  logInfo("Tool called", request.params);
  // ...
});
```

### Common Issues

**Server not connecting:**
- Check command path is absolute or in `$PATH`
- Verify node/python version matches requirements
- Check `startup_timeout_sec` (default: 10s)

**Tools not appearing:**
- Verify `tools/list` returns valid JSON
- Check inputSchema is valid JSON Schema
- Ensure `capabilities.tools = {}` in server initialization

**Tool execution fails:**
- Check `tool_timeout_sec` (default: 60s)
- Verify `tools/call` handler returns correct format
- Test with MCP Inspector first

---

## Example: Real-World MCP Server

Complete example: A server that queries SQLite databases.

**File:** `sqlite-mcp-server.js`

```javascript
#!/usr/bin/env node

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import Database from "better-sqlite3";
import { readFileSync } from "fs";

const server = new Server(
  {
    name: "sqlite-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
      resources: {},
    },
  }
);

// Database connections
const databases = new Map();

function getDatabase(path) {
  if (!databases.has(path)) {
    databases.set(path, new Database(path, { readonly: true }));
  }
  return databases.get(path);
}

// List available databases as resources
server.setRequestHandler("resources/list", async () => {
  const dbPath = process.env.DATABASE_PATH || "./app.db";

  return {
    resources: [
      {
        uri: `sqlite://${dbPath}`,
        name: "Application Database",
        description: "Main application SQLite database",
        mimeType: "application/x-sqlite3",
      },
    ],
  };
});

// Read schema as resource
server.setRequestHandler("resources/read", async (request) => {
  const { uri } = request.params;

  if (uri.startsWith("sqlite://")) {
    const dbPath = uri.replace("sqlite://", "");
    const db = getDatabase(dbPath);

    const schema = db
      .prepare(
        "SELECT sql FROM sqlite_master WHERE type='table' ORDER BY name"
      )
      .all()
      .map((row) => row.sql)
      .join(";\n\n");

    return {
      contents: [
        {
          uri,
          mimeType: "text/plain",
          text: schema,
        },
      ],
    };
  }

  throw new Error(`Unknown resource: ${uri}`);
});

// SQL query tool
server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "query_database",
        description: "Execute read-only SQL query on SQLite database",
        inputSchema: {
          type: "object",
          properties: {
            query: {
              type: "string",
              description: "SQL SELECT query",
            },
            database: {
              type: "string",
              description: "Database path (optional)",
              default: process.env.DATABASE_PATH || "./app.db",
            },
          },
          required: ["query"],
        },
      },
    ],
  };
});

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "query_database") {
    const dbPath = args.database || process.env.DATABASE_PATH || "./app.db";

    // Security: Only allow SELECT
    if (!args.query.trim().toUpperCase().startsWith("SELECT")) {
      throw new Error("Only SELECT queries are allowed");
    }

    try {
      const db = getDatabase(dbPath);
      const rows = db.prepare(args.query).all();

      return {
        content: [
          {
            type: "text",
            text: JSON.stringify(rows, null, 2),
          },
        ],
      };
    } catch (error) {
      return {
        content: [
          {
            type: "text",
            text: `SQL Error: ${error.message}`,
          },
        ],
        isError: true,
      };
    }
  }

  throw new Error(`Unknown tool: ${name}`);
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);

  // Cleanup on exit
  process.on("SIGTERM", () => {
    databases.forEach((db) => db.close());
    process.exit(0);
  });
}

main().catch((error) => {
  console.error("Server error:", error);
  process.exit(1);
});
```

**Usage:**

```toml
[mcp_servers.sqlite]
command = "node"
args = ["./sqlite-mcp-server.js"]
env = { DATABASE_PATH = "/path/to/database.db" }
```

```bash
codex

# In TUI:
# "Query the database for all users created in the last 7 days"
```

---

## Resources

- **MCP Specification**: https://modelcontextprotocol.io/
- **MCP SDK (TypeScript)**: https://github.com/modelcontextprotocol/typescript-sdk
- **MCP SDK (Python)**: https://github.com/modelcontextprotocol/python-sdk
- **Example Servers**: https://github.com/modelcontextprotocol/servers
- **Codex Config Docs**: [config.md#mcp-integration](./config.md#mcp-integration)
