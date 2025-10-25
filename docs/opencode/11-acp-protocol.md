# OpenCode - ACP Protocol

> **Agent Client Protocol implementation for IDE integration**

---

## Overview

**Agent Client Protocol (ACP)** is a JSON-RPC protocol for IDEs to control coding agents. OpenCode implements ACP for integration with:
- **Zed** - Built-in ACP support
- **VS Code** - Via ACP extension
- **Other IDEs** - Any editor supporting JSON-RPC

**Files**:
- `acp/README.md` - Protocol documentation
- `acp/agent.ts` (17KB) - ACP agent implementation
- `acp/session.ts` - Session mapping
- `acp/client.ts` - Client capabilities

**Spec**: https://agentclientprotocol.com

---

## Architecture

### ACP Command

```bash
opencode acp
```

**What it does**:
1. Starts JSON-RPC server on stdin/stdout
2. Listens for IDE commands
3. Maps to OpenCode sessions
4. Returns responses in ACP format

### Protocol Flow

```
IDE (Zed/VS Code)
    │
    ├─ Initialize ─▶ opencode acp
    │                    │
    │                    ▼
    │              Create Agent
    │                    │
    ├─ CreateSession ──▶ │
    │   ◀─ SessionID ────┤
    │                    │
    ├─ SendMessage ────▶ │
    │                    ├─ Process
    │                    ├─ Stream Response
    │   ◀─ Messages ─────┤
    │                    │
    └─ CloseSession ───▶ │
                         ▼
                    Cleanup
```

---

## ACP Methods

### Initialize

**Request**:
```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "agentName": "opencode",
    "version": "1.0"
  },
  "id": 1
}
```

**Response**:
```json
{
  "jsonrpc": "2.0",
  "result": {
    "capabilities": {
      "tools": true,
      "streaming": true,
      "multimodal": true
    }
  },
  "id": 1
}
```

### Create Session

**Request**:
```json
{
  "method": "sessions/create",
  "params": {
    "workspaceDirectory": "/path/to/project"
  }
}
```

**Response**:
```json
{
  "result": {
    "sessionId": "session_abc123"
  }
}
```

### Send Message

**Request**:
```json
{
  "method": "sessions/message",
  "params": {
    "sessionId": "session_abc123",
    "message": {
      "role": "user",
      "content": "Add error handling to auth.ts"
    }
  }
}
```

**Response** (streaming):
```json
{
  "result": {
    "messages": [
      {
        "role": "assistant",
        "content": "I'll add try-catch blocks..."
      },
      {
        "type": "tool_use",
        "name": "read",
        "input": { "filePath": "auth.ts" }
      },
      {
        "type": "tool_result",
        "content": "File contents..."
      }
    ]
  }
}
```

---

## IDE Integration

### Zed Editor

**Configuration** (`~/.config/zed/settings.json`):
```json
{
  "agents": {
    "opencode": {
      "command": "opencode",
      "args": ["acp"],
      "enabled": true
    }
  }
}
```

**Usage**:
1. Open project in Zed
2. Start assistant panel
3. Select "opencode" agent
4. Chat naturally

### VS Code

**Installation**:
```bash
code --install-extension opencode.acp-client
```

**Configuration** (`.vscode/settings.json`):
```json
{
  "opencode.acp.enabled": true,
  "opencode.acp.command": "opencode acp"
}
```

---

## Session Mapping

ACP sessions map to OpenCode sessions:

```typescript
// ACP session ID → OpenCode session ID
const sessionMap = new Map<string, string>()

// On ACP create
const acpID = generateACPSessionID()
const opencodeID = await Session.create({...})
sessionMap.set(acpID, opencodeID)

// On ACP message
const opencodeID = sessionMap.get(acpID)
await SessionPrompt.prompt({
  sessionID: opencodeID,
  parts: [{ type: "text", text: message.content }]
})
```

---

## Current Limitations

From `acp/README.md`:

**Not Yet Implemented**:
- ❌ Streaming responses (buffered for now)
- ❌ Tool call progress reporting
- ❌ Real-time status updates
- ❌ Cancellation support

**Implemented**:
- ✅ Session management
- ✅ Message handling
- ✅ Tool execution
- ✅ Error handling
- ✅ Multi-session support

---

## Future Enhancements

**Planned Features**:
1. **Streaming** - Real-time response chunks
2. **Progress** - Tool execution status
3. **Cancellation** - Abort in-progress requests
4. **Context** - File/selection context from IDE
5. **Edit Apply** - Direct code application

---

## Development

### Testing ACP

```bash
# Start ACP server
opencode acp

# Send test message (in another terminal)
echo '{"jsonrpc":"2.0","method":"initialize","id":1}' | opencode acp
```

### Debugging

```bash
# Enable debug logging
DEBUG=acp:* opencode acp
```

---

## Best Practices

**For IDE Developers**:
- Implement all required methods
- Handle streaming responses
- Show tool execution progress
- Implement cancellation
- Cache session IDs

**For Users**:
- Use latest OpenCode version
- Configure per-project if needed
- Report issues with logs

---

For implementation, see `packages/opencode/src/acp/` and spec at https://agentclientprotocol.com.

