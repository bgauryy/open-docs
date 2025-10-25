# Tool System Architecture

This guide provides a deep dive into Codex's tool system architecture, including how tools are registered, invoked, approved, and extended.

## Overview

Codex uses a comprehensive tool system that allows the AI model to interact with the filesystem, shell, web, and external services. The tool system is built on:

- **Tool Registration**: Tools are defined in `core/src/tools/spec.rs`
- **Tool Orchestration**: Request handling in `core/src/tools/orchestrator.rs`
- **Tool Handlers**: Individual tool implementations in `core/src/tools/handlers/`
- **Approval Flow**: User confirmation system for risky operations
- **MCP Integration**: External tool support via Model Context Protocol

---

## Core Tool Types

### Built-in Tools

Codex provides the following built-in tools (as found in the codebase):

1. **`local_shell`** - Execute shell commands
   - Handler: `core/src/tools/handlers/shell.rs`
   - Approval: Required for untrusted/networked commands
   - Sandboxing: Uses Seatbelt (macOS) or Landlock (Linux)

2. **`read_file`** - Read file contents
   - Handler: Built into tool orchestrator
   - Permissions: Respects sandbox boundaries

3. **`apply_patch`** - Apply unified diff patches
   - Handler: `core/src/tools/handlers/apply_patch.rs`
   - Approval: Required for file modifications

4. **`list_dir`** - List directory contents
   - Handler: Built into tool orchestrator
   - Permissions: Read-only, respects sandbox

5. **`grep_files`** - Search file contents
   - Handler: Built into tool orchestrator
   - Uses: ripgrep under the hood

6. **`web_search`** - Web search capability
   - Opt-in: Requires `tools.web_search = true` in config
   - Provider: First-party OpenAI web search

7. **`view_image`** - Attach images to requests
   - Opt-in: Requires `tools.view_image = true` in config
   - Use case: Screenshots, diagrams, error messages

8. **MCP Tools** - External tools from MCP servers
   - Dynamic: Loaded from configured MCP servers
   - Handler: `core/src/mcp_tool_call.rs`

---

## Tool Registration Flow

### 1. Tool Definition

Tools are defined using JSON Schema in `core/src/tools/spec.rs`:

```rust
pub struct ToolSpec {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,  // JSON Schema
}
```

### 2. Tool Discovery

When a session starts:

1. **Load built-in tools** from static definitions
2. **Connect to MCP servers** configured in `config.toml`
3. **Query MCP tools** via `tools/list` MCP request
4. **Merge tool lists** and send to model in initial request

### 3. Tool Availability

Tools are filtered based on:
- **Configuration**: `tools.web_search`, `tools.view_image`, etc.
- **MCP Server Status**: Only tools from connected servers
- **Approval Policy**: Some tools hidden in restrictive modes
- **Sandbox Mode**: Tools respect filesystem boundaries

---

## Tool Invocation Flow

### Request Flow

```
User Message
    ↓
Model Request (with tool definitions)
    ↓
Model Response (with tool_calls)
    ↓
Tool Approval Check
    ↓
Tool Execution
    ↓
Tool Result Collection
    ↓
Next Model Request (with tool results)
```

### Detailed Steps

1. **Model invokes tool** by returning `tool_calls` in response
2. **Approval check** runs (see Approval System below)
3. **Tool handler** executes the tool
4. **Result captured** (stdout, stderr, exit code, file changes)
5. **Result formatted** as JSON for next model request
6. **Loop continues** until model sends final message

---

## Approval System

### Approval Policies

Located in `core/src/config.rs`:

```toml
approval_policy = "untrusted"  # Ask before untrusted commands
approval_policy = "on-failure"  # Ask when sandboxed command fails
approval_policy = "on-request"  # Model decides when to ask
approval_policy = "never"       # Never ask (automation mode)
```

### Approval Decision Flow

Implementation in `common/src/approval_presets.rs`:

```
Tool Call Received
    ↓
Is approval_policy = "never"? → Execute immediately
    ↓
Is command trusted? (whitelist check)
    ↓
  Yes → Execute
    ↓
  No → Does policy require approval?
    ↓
    Yes → Show approval UI
        ↓
        User Decision:
        - Approve Once
        - Approve for Session
        - Deny
        - Abort Conversation
    ↓
Execute or Skip based on decision
```

### Trusted Command Whitelist

The system maintains a hardcoded list of "safe" commands in `core/src/command_safety/`:

- Read-only commands: `ls`, `cat`, `grep`, `find`, `git log`
- Build commands: `npm install`, `cargo build`, `pytest`
- Version checks: `node --version`, `python --version`

Commands **not** on the whitelist require approval when `approval_policy = "untrusted"`.

---

## Tool Handlers

### Shell Command Handler

**Location:** `core/src/tools/handlers/shell.rs`

Key features:
- Sandbox integration (Seatbelt/Landlock)
- Environment variable filtering
- Working directory management
- Timeout handling (default: 2 minutes, configurable)
- Output capture (stdout/stderr)

Example invocation:
```json
{
  "name": "local_shell",
  "arguments": {
    "command": "npm test",
    "timeout_ms": 120000
  }
}
```

### Apply Patch Handler

**Location:** `core/src/tools/handlers/apply_patch.rs`

Key features:
- Unified diff format support
- Dry-run mode (check before applying)
- Conflict detection
- Atomic file updates
- Permission preservation

Example invocation:
```json
{
  "name": "apply_patch",
  "arguments": {
    "patch": "--- a/file.txt\n+++ b/file.txt\n@@ -1 +1 @@\n-old\n+new"
  }
}
```

### MCP Tool Handler

**Location:** `core/src/mcp_tool_call.rs`

Key features:
- Dynamic tool routing to correct MCP server
- Timeout handling (configurable per server)
- Error translation (MCP errors → Codex errors)
- Result serialization

---

## Extending the Tool System

### Option 1: Create an MCP Server

**Recommended approach** for custom tools.

1. Implement MCP protocol (STDIO or Streamable HTTP)
2. Define tools via `tools/list` response
3. Handle `tools/call` requests
4. Configure in `config.toml`:

```toml
[mcp_servers.my_tool]
command = "node"
args = ["./my-mcp-server.js"]
```

See [MCP Server Development Guide](./mcp-development.md) for details.

### Option 2: Modify Built-in Tools (Source Code)

**For advanced users only.**

1. Add tool spec in `core/src/tools/spec.rs`
2. Implement handler in `core/src/tools/handlers/my_tool.rs`
3. Register in orchestrator (`core/src/tools/orchestrator.rs`)
4. Rebuild Codex from source

**Example (pseudo-code):**
```rust
// In tools/spec.rs
pub fn my_custom_tool_spec() -> ToolSpec {
    ToolSpec {
        name: "my_custom_tool".into(),
        description: "Does something custom".into(),
        input_schema: json!({
            "type": "object",
            "properties": {
                "param": { "type": "string" }
            },
            "required": ["param"]
        })
    }
}

// In tools/handlers/my_tool.rs
pub async fn handle_my_tool(params: MyToolParams) -> Result<ToolResult> {
    // Implementation
}
```

---

## Tool Result Format

Tools return results in this structure:

```rust
pub struct ToolResult {
    pub output: String,         // Human-readable output
    pub success: bool,          // Did tool succeed?
    pub metadata: Option<Value> // Additional data (exit code, etc.)
}
```

For shell commands:
```json
{
  "output": "test output\n",
  "success": true,
  "metadata": {
    "exit_code": 0,
    "duration_ms": 1234
  }
}
```

---

## Tool Parallelism

Codex supports **parallel tool execution** when the model requests multiple independent tools.

**Implementation:** `core/src/tools/orchestrator.rs`

Example scenario:
- Model requests: `read_file("a.txt")` and `read_file("b.txt")` simultaneously
- Orchestrator executes both in parallel using `tokio::join!`
- Results collected and sent together in next request

**Limitations:**
- Only read-only tools are parallelized by default
- Write operations execute sequentially to prevent conflicts
- MCP tools execute sequentially (server-dependent)

---

## Tool Debugging

### Enable Verbose Logging

```bash
RUST_LOG=codex_core=debug codex
```

Check logs at `~/.codex/log/codex-tui.log`:

```bash
tail -F ~/.codex/log/codex-tui.log
```

### Common Issues

**Tool not appearing in model's tool list:**
- Check if tool is enabled in config (`tools.web_search = true`)
- Verify MCP server is connected (`/status` in TUI)
- Check MCP server logs for errors

**Tool execution fails:**
- Check sandbox mode (tool may need `--sandbox danger-full-access`)
- Verify file paths are absolute
- Check permissions on target files/directories

**Approval prompt not showing:**
- Verify `approval_policy` setting
- Check if command is on trusted whitelist
- Ensure not using `codex exec` (never prompts)

---

## Best Practices

### For Users

1. **Use restrictive approval policies by default** (`untrusted` or `on-request`)
2. **Review tool calls** before approving, especially shell commands
3. **Enable sandbox** to limit filesystem access (`workspace-write` or `read-only`)
4. **Use MCP servers** for custom functionality instead of modifying source

### For MCP Server Developers

1. **Keep tools focused** - one tool does one thing well
2. **Validate inputs** - check all parameters before execution
3. **Return structured output** - use JSON when possible
4. **Handle timeouts** - implement graceful cancellation
5. **Log errors** - help users debug integration issues

### For Codex Developers

1. **Add tests** for new tools (`core/tests/suite/tools.rs`)
2. **Document parameters** with clear JSON Schema
3. **Consider security** - what can go wrong with this tool?
4. **Profile performance** - avoid blocking the orchestrator thread
5. **Handle errors gracefully** - don't crash on malformed input

---

## Tool Performance

### Benchmarks (Typical Values)

- **`read_file`**: 1-5ms for small files (<100KB)
- **`list_dir`**: 5-20ms for directories with <1000 entries
- **`grep_files`**: 10-100ms depending on search scope
- **`local_shell`**: Varies by command (200ms-120s timeout)
- **MCP tools**: Varies by server (configurable timeout)

### Optimization Tips

1. **Batch operations** - Read multiple files in parallel
2. **Limit search scope** - Use specific paths for `grep_files`
3. **Cache results** - Avoid re-running expensive commands
4. **Use streaming** - For large outputs, consider chunking
5. **Tune timeouts** - Set `tool_timeout_sec` per MCP server

---

## References

- [Sandbox & Approvals](./sandbox.md) - Security model for tools
- [Config Reference](./config.md) - Tool configuration options
- [MCP Integration](./advanced.md#model-context-protocol) - External tools
- [MCP Development Guide](./mcp-development.md) - Building custom tools
