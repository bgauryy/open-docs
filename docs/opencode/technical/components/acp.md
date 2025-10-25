# ACP (Agent Client Protocol)

**Path**: `packages/opencode/src/acp`
**Type**: Protocol Layer
**File Count**: 4

## Description

Agent Client Protocol implementation for standardized agent-to-agent communication.

## Purpose

The ACP component provides a protocol-level abstraction that enables OpenCode to communicate with external agent clients using a standardized protocol. This allows other applications, tools, and agents to integrate with OpenCode programmatically.

## Key Features

- JSON-RPC 2.0 based protocol
- Session management via protocol
- Tool execution through protocol
- Streaming responses
- Multiple transport options (stdio, TCP, Unix socket)

## Component Files

- `agent.ts` - Main ACP agent implementation
- `session.ts` - ACP session manager
- `protocol.ts` - Protocol message handlers
- `transport.ts` - Transport layer (stdio/socket)

## Dependencies

### Internal Dependencies
- `packages/opencode/src/session` - Session management (3 imports)

### External Dependencies
- `@agentclientprotocol/sdk` - ACP protocol implementation

## Usage

### Start ACP Server

```bash
# Start with stdio transport (default)
opencode acp

# Start with TCP socket
opencode acp --transport tcp --port 9000
```

### Connect from Client

```typescript
import { ACPClient } from '@agentclientprotocol/sdk';

const client = new ACPClient({
  command: 'opencode',
  args: ['acp']
});

await client.connect();

const session = await client.createSession({
  model: 'claude-3-sonnet'
});

for await (const chunk of client.sendMessage(session.sessionId, {
  content: 'help me build a feature'
})) {
  console.log(chunk.content);
}
```

## Protocol Messages

### Initialize
```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocolVersion": "1.0",
    "clientInfo": {"name": "client", "version": "1.0.0"}
  },
  "id": 1
}
```

### Create Session
```json
{
  "jsonrpc": "2.0",
  "method": "session/create",
  "params": {"model": "claude-3-sonnet"},
  "id": 2
}
```

### Send Message
```json
{
  "jsonrpc": "2.0",
  "method": "session/message",
  "params": {
    "sessionId": "abc123",
    "content": "help me build a feature"
  },
  "id": 3
}
```

## Related Documentation

- [Agent Client Protocol Flow](../flows/acp-agent-flow.md)
- [Session Management Component](./session.md)
