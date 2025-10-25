# Codex CLI - MCP Integration (Implementation Details)

> **📚 Official User Guide**: For MCP setup and usage, see [Official advanced.md](../../context/codex/docs/advanced.md)
>
> **🎯 This Document**: Focuses on internal MCP implementation, connection management, and protocol details for developers.

---

## Quick Links

- **User Guide**: `/context/codex/docs/advanced.md#model-context-protocol` - How to configure MCP servers
- **This Doc**: MCP implementation details for developers
- **Related**: [18-mcp-development.md](./18-mcp-development.md) - Building custom MCP servers

---

## Table of Contents
- [Model Context Protocol Overview](#model-context-protocol-overview)
- [MCP Connection Manager](#mcp-connection-manager)
- [Server Configuration](#server-configuration)
- [Tool and Resource Discovery](#tool-and-resource-discovery)
- [Custom MCP Servers](#custom-mcp-servers)

---

## Model Context Protocol Overview

### What is MCP?

**Model Context Protocol (MCP)** is a standard protocol that allows AI applications to connect to external data sources and tools. It provides:

- **Tools**: Functions the AI can call
- **Resources**: Data the AI can read
- **Prompts**: Pre-defined prompt templates

### MCP Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Codex CLI                             │
│  ┌───────────────────────────────────────────────────┐  │
│  │           MCP Connection Manager                  │  │
│  │  • Manages multiple MCP server connections        │  │
│  │  • Discovers tools and resources                  │  │
│  │  • Routes requests to appropriate servers         │  │
│  └──────────┬────────────────────┬───────────────────┘  │
│             │                    │                       │
│  ┌──────────▼────────┐  ┌───────▼──────────┐           │
│  │  MCP Connection 1 │  │  MCP Connection 2│           │
│  │  (filesystem)     │  │  (database)      │           │
│  └──────────┬────────┘  └───────┬──────────┘           │
└─────────────┼────────────────────┼──────────────────────┘
              │                    │
              │ stdio/SSE          │ stdio/SSE
              │                    │
┌─────────────▼────────┐  ┌───────▼──────────┐
│   MCP Server         │  │   MCP Server     │
│   (Node.js/Python)   │  │   (Node.js)      │
│                      │  │                  │
│   Tools:             │  │   Tools:         │
│   • read_file        │  │   • query        │
│   • write_file       │  │   • execute      │
│                      │  │                  │
│   Resources:         │  │   Resources:     │
│   • file://...       │  │   • schema://... │
└──────────────────────┘  └──────────────────┘
```

---

## MCP Connection Manager

### McpConnectionManager Structure

**Location**: `core/src/mcp/connection_manager.rs`

```rust
pub struct McpConnectionManager {
    connections: Arc<RwLock<HashMap<String, McpConnection>>>,
    config: Arc<Config>,
}

pub struct McpConnection {
    pub name: String,
    pub transport: McpTransport,
    pub tools: Vec<McpTool>,
    pub resources: Vec<McpResource>,
    pub prompts: Vec<McpPrompt>,
}

pub enum McpTransport {
    Stdio {
        child: Child,
        stdin: ChildStdin,
        stdout: BufReader<ChildStdout>,
    },
    Sse {
        url: String,
        client: reqwest::Client,
    },
}
```

### Initialization

```rust
impl McpConnectionManager {
    pub async fn new(config: Arc<Config>) -> Result<Self> {
        let mut connections = HashMap::new();
        
        // Initialize connections from config
        for (name, server_config) in &config.mcp_servers {
            let connection = Self::connect_to_server(name, server_config).await?;
            connections.insert(name.clone(), connection);
        }
        
        Ok(Self {
            connections: Arc::new(RwLock::new(connections)),
            config,
        })
    }
    
    async fn connect_to_server(
        name: &str,
        config: &McpServerConfig,
    ) -> Result<McpConnection> {
        // 1. Spawn server process
        let mut child = tokio::process::Command::new(&config.command)
            .args(&config.args.unwrap_or_default())
            .envs(config.env.clone().unwrap_or_default())
            .stdin(Stdio::piped())
            .stdout(Stdio::piped())
            .stderr(Stdio::inherit())
            .spawn()?;
        
        let stdin = child.stdin.take().unwrap();
        let stdout = BufReader::new(child.stdout.take().unwrap());
        
        let transport = McpTransport::Stdio { child, stdin, stdout };
        
        // 2. Initialize protocol
        let mut connection = McpConnection {
            name: name.to_string(),
            transport,
            tools: Vec::new(),
            resources: Vec::new(),
            prompts: Vec::new(),
        };
        
        connection.initialize().await?;
        
        // 3. Discover capabilities
        connection.discover_tools().await?;
        connection.discover_resources().await?;
        connection.discover_prompts().await?;
        
        Ok(connection)
    }
}
```

### Connection Management

```rust
impl McpConnectionManager {
    pub async fn get_connection(&self, name: &str) -> Result<Arc<McpConnection>> {
        let connections = self.connections.read().await;
        
        connections.get(name)
            .cloned()
            .ok_or_else(|| CodexErr::McpError(format!("Server '{}' not found", name)))
    }
    
    pub async fn list_servers(&self) -> Vec<String> {
        let connections = self.connections.read().await;
        connections.keys().cloned().collect()
    }
    
    pub async fn shutdown_all(&self) -> Result<()> {
        let mut connections = self.connections.write().await;
        
        for (name, connection) in connections.drain() {
            connection.shutdown().await?;
        }
        
        Ok(())
    }
}
```

---

## Server Configuration

### Config Format

**Location**: `~/.codex/config.yaml`

```yaml
mcpServers:
  # Filesystem server
  filesystem:
    command: npx
    args:
      - "-y"
      - "@modelcontextprotocol/server-filesystem"
      - "/path/to/allowed/directory"
    env:
      DEBUG: "mcp:*"
  
  # GitHub server
  github:
    command: npx
    args:
      - "-y"
      - "@modelcontextprotocol/server-github"
    env:
      GITHUB_TOKEN: ${GITHUB_TOKEN}
  
  # Database server
  postgres:
    command: npx
    args:
      - "-y"
      - "@modelcontextprotocol/server-postgres"
    env:
      DATABASE_URL: postgresql://localhost/mydb
  
  # Custom Python server
  custom:
    command: python
    args:
      - "/path/to/my_mcp_server.py"
```

### McpServerConfig Structure

```rust
#[derive(Deserialize, Clone)]
pub struct McpServerConfig {
    pub command: String,
    pub args: Option<Vec<String>>,
    pub env: Option<HashMap<String, String>>,
    pub transport: Option<McpTransportType>,
}

#[derive(Deserialize, Clone)]
pub enum McpTransportType {
    Stdio,
    Sse { url: String },
}
```

---

## Tool and Resource Discovery

### Tool Discovery

```rust
impl McpConnection {
    pub async fn discover_tools(&mut self) -> Result<()> {
        // Send ListTools request
        let request = McpRequest::ListTools;
        let response = self.send_request(request).await?;
        
        match response {
            McpResponse::ListTools { tools } => {
                self.tools = tools;
                Ok(())
            }
            _ => Err(CodexErr::McpError("Unexpected response".into())),
        }
    }
    
    pub async fn call_tool(
        &self,
        tool_name: &str,
        arguments: Value,
    ) -> Result<McpToolResult> {
        let request = McpRequest::CallTool {
            name: tool_name.to_string(),
            arguments: Some(arguments),
        };
        
        let response = self.send_request(request).await?;
        
        match response {
            McpResponse::CallToolResult { content, is_error } => {
                Ok(McpToolResult { content, is_error })
            }
            _ => Err(CodexErr::McpError("Unexpected response".into())),
        }
    }
}
```

### Resource Discovery

```rust
impl McpConnection {
    pub async fn discover_resources(&mut self) -> Result<()> {
        let request = McpRequest::ListResources;
        let response = self.send_request(request).await?;
        
        match response {
            McpResponse::ListResources { resources } => {
                self.resources = resources;
                Ok(())
            }
            _ => Err(CodexErr::McpError("Unexpected response".into())),
        }
    }
    
    pub async fn read_resource(&self, uri: &str) -> Result<Vec<McpResourceContent>> {
        let request = McpRequest::ReadResource {
            uri: uri.to_string(),
        };
        
        let response = self.send_request(request).await?;
        
        match response {
            McpResponse::ReadResource { contents } => {
                Ok(contents)
            }
            _ => Err(CodexErr::McpError("Unexpected response".into())),
        }
    }
}
```

### Protocol Implementation

```rust
// MCP protocol types
#[derive(Serialize, Deserialize)]
#[serde(tag = "method", content = "params")]
pub enum McpRequest {
    Initialize {
        protocol_version: String,
        client_info: ClientInfo,
    },
    ListTools,
    CallTool {
        name: String,
        arguments: Option<Value>,
    },
    ListResources,
    ReadResource {
        uri: String,
    },
    ListPrompts,
    GetPrompt {
        name: String,
        arguments: Option<HashMap<String, String>>,
    },
}

#[derive(Serialize, Deserialize)]
#[serde(tag = "result", content = "data")]
pub enum McpResponse {
    Initialize {
        protocol_version: String,
        server_info: ServerInfo,
        capabilities: ServerCapabilities,
    },
    ListTools {
        tools: Vec<McpTool>,
    },
    CallToolResult {
        content: Vec<McpContent>,
        is_error: Option<bool>,
    },
    ListResources {
        resources: Vec<McpResource>,
    },
    ReadResource {
        contents: Vec<McpResourceContent>,
    },
    // ...
}
```

---

## Tool Integration

### Registering MCP Tools

**Location**: `core/src/tools/mcp_tools.rs`

```rust
pub fn register_mcp_tools(
    registry: &mut ToolRegistry,
    mcp_manager: Arc<McpConnectionManager>,
) -> Result<()> {
    // Get all MCP tools from all servers
    let all_tools = discover_all_mcp_tools(&mcp_manager).await?;
    
    for (server_name, tool) in all_tools {
        // Create wrapper tool
        let mcp_tool_handler = McpToolHandler {
            server_name: server_name.clone(),
            tool_name: tool.name.clone(),
            schema: tool.input_schema.clone(),
            mcp_manager: mcp_manager.clone(),
        };
        
        // Register with prefixed name
        let prefixed_name = format!("mcp_{}_{}", server_name, tool.name);
        registry.register(prefixed_name, Box::new(mcp_tool_handler));
    }
    
    Ok(())
}

struct McpToolHandler {
    server_name: String,
    tool_name: String,
    schema: JsonSchema,
    mcp_manager: Arc<McpConnectionManager>,
}

#[async_trait]
impl ToolHandler for McpToolHandler {
    async fn execute(&self, args: Value) -> Result<ToolOutput> {
        let connection = self.mcp_manager.get_connection(&self.server_name).await?;
        
        let result = connection.call_tool(&self.tool_name, args).await?;
        
        Ok(ToolOutput {
            success: !result.is_error.unwrap_or(false),
            content: format_mcp_content(&result.content),
        })
    }
}
```

### MCP Tool Schema

```rust
fn convert_mcp_tool_to_spec(tool: &McpTool) -> ToolSpec {
    ToolSpec {
        name: tool.name.clone(),
        description: tool.description.clone().unwrap_or_default(),
        parameters: convert_json_schema(&tool.input_schema),
        strict: false,
    }
}
```

---

## Custom MCP Servers

### Creating a Custom Server (Python)

```python
# my_mcp_server.py
import asyncio
import json
import sys
from mcp import MCPServer, Tool, Resource

server = MCPServer("my-server", "1.0.0")

@server.tool("custom_search")
async def custom_search(query: str, limit: int = 10):
    """Search my custom data source"""
    results = await search_database(query, limit)
    return {"results": results}

@server.resource("data://config")
async def get_config():
    """Get application configuration"""
    return {"content": json.dumps(load_config())}

async def main():
    # Stdio transport
    async with server.stdio() as (read_stream, write_stream):
        await server.run(read_stream, write_stream)

if __name__ == "__main__":
    asyncio.run(main())
```

### Creating a Custom Server (Node.js)

```javascript
// my-mcp-server.js
import { MCPServer } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new MCPServer({
  name: "my-server",
  version: "1.0.0",
});

// Register tool
server.tool(
  "custom_search",
  "Search my custom data source",
  {
    query: { type: "string", description: "Search query" },
    limit: { type: "number", description: "Max results", default: 10 },
  },
  async ({ query, limit }) => {
    const results = await searchDatabase(query, limit);
    return {
      content: [{ type: "text", text: JSON.stringify(results) }],
    };
  }
);

// Register resource
server.resource(
  "data://config",
  "Application configuration",
  async () => {
    const config = loadConfig();
    return {
      contents: [{ type: "text", text: JSON.stringify(config) }],
    };
  }
);

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Registering Custom Server

```yaml
# ~/.codex/config.yaml
mcpServers:
  my-server:
    command: node
    args:
      - "/path/to/my-mcp-server.js"
    env:
      DATABASE_URL: "..."
```

---

## MCP Protocol Details

### Message Format

All messages are JSON-RPC 2.0 format over stdio:

```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "/path/to/file.txt"
    }
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "File contents here..."
      }
    ]
  }
}
```

### Content Types

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum McpContent {
    Text {
        text: String,
    },
    Image {
        data: String,  // Base64
        mime_type: String,
    },
    Resource {
        uri: String,
        mime_type: Option<String>,
    },
}
```

---

## Error Handling

### MCP Errors

```rust
#[derive(Debug)]
pub enum McpError {
    ConnectionFailed(String),
    ServerNotFound(String),
    ToolNotFound { server: String, tool: String },
    ResourceNotFound { server: String, uri: String },
    ProtocolError(String),
    Timeout,
}

impl McpConnection {
    async fn handle_error(&self, error: McpError) -> Result<()> {
        match error {
            McpError::Timeout => {
                // Attempt reconnection
                self.reconnect().await?;
            }
            McpError::ConnectionFailed(_) => {
                // Mark connection as failed
                self.mark_failed();
            }
            _ => {
                // Log error
                error!("MCP error: {:?}", error);
            }
        }
        
        Ok(())
    }
}
```

---

## Best Practices

### For Users

1. **Start Simple**: Begin with official MCP servers
2. **Test Servers**: Verify servers work before adding to config
3. **Manage Permissions**: Be aware of what each server can access
4. **Monitor Resources**: Some servers may be resource-intensive

### For Server Developers

1. **Follow Protocol**: Implement all required MCP methods
2. **Error Handling**: Return meaningful error messages
3. **Security**: Validate all inputs, restrict access
4. **Documentation**: Document tools and resources clearly
5. **Testing**: Test with actual MCP clients

---

## Related Documentation

- [06-tool-system.md](./06-tool-system.md) - Tool system architecture
- [08-configuration.md](./08-configuration.md) - Configuration options
- [11-tool-implementations.md](./11-tool-implementations.md) - Tool implementations

