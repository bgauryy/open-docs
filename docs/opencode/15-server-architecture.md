# OpenCode - Server Architecture

> **HTTP API server implementation and endpoints**

---

## Overview

OpenCode's server provides HTTP/WebSocket APIs for:
- **Session management** - Create, list, manage sessions
- **Message handling** - Send/receive AI messages
- **Project management** - Multi-project support
- **Provider info** - List models and capabilities
- **File operations** - Browse and manage files

**Files**:
- `server/server.ts` (44KB) - Main server with Hono
- `server/project.ts` - Project management
- `server/tui.ts` - TUI server integration

---

## Starting the Server

### Headless Server

```bash
opencode serve --port 8080 --host 0.0.0.0
```

**Options**:
- `--port` - Port to listen on (default: 3000)
- `--host` - Host to bind to (default: localhost)
- `--cors` - Enable CORS

### TUI Server

```bash
opencode tui
```

Automatically starts server for TUI communication.

---

## HTTP API Reference

### Session Endpoints

#### POST /sessions

Create new session:
```bash
curl -X POST http://localhost:8080/sessions \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "anthropic",
    "model": "claude-3-5-sonnet"
  }'
```

Response:
```json
{
  "id": "session_abc123",
  "created": 1234567890,
  "provider": "anthropic",
  "model": "claude-3-5-sonnet"
}
```

#### GET /sessions

List all sessions:
```bash
curl http://localhost:8080/sessions
```

#### GET /sessions/:id

Get session details:
```bash
curl http://localhost:8080/sessions/session_abc123
```

#### DELETE /sessions/:id

Delete session:
```bash
curl -X DELETE http://localhost:8080/sessions/session_abc123
```

---

### Message Endpoints

#### POST /sessions/:id/messages

Send message to session:
```bash
curl -X POST http://localhost:8080/sessions/session_abc123/messages \
  -H "Content-Type: application/json" \
  -d '{
    "parts": [{
      "type": "text",
      "text": "Add error handling to auth.ts"
    }]
  }'
```

Response (streaming):
```json
{"type":"message","data":{"role":"assistant","content":"I'll add try-catch blocks..."}}
{"type":"tool","data":{"tool":"read","args":{"filePath":"auth.ts"}}}
{"type":"tool-result","data":{"output":"File contents..."}}
{"type":"complete","data":{"tokens":1234,"cost":0.05}}
```

#### GET /sessions/:id/messages

List messages in session:
```bash
curl http://localhost:8080/sessions/session_abc123/messages
```

---

### Provider Endpoints

#### GET /providers

List available providers:
```bash
curl http://localhost:8080/providers
```

Response:
```json
[
  {
    "id": "anthropic",
    "name": "Anthropic",
    "models": ["claude-3-5-sonnet", "claude-3-opus"]
  },
  {
    "id": "openai",
    "name": "OpenAI",
    "models": ["gpt-4o", "gpt-4-turbo"]
  }
]
```

#### GET /models

List all available models:
```bash
curl http://localhost:8080/models
```

---

### Project Endpoints

#### GET /projects

List projects:
```bash
curl http://localhost:8080/projects
```

#### POST /projects

Create/register project:
```bash
curl -X POST http://localhost:8080/projects \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-app",
    "path": "/path/to/project"
  }'
```

#### GET /projects/:id/files

List files in project:
```bash
curl http://localhost:8080/projects/proj_123/files?path=src
```

---

## WebSocket API

### Connection

```javascript
const ws = new WebSocket('ws://localhost:8080/sessions/session_abc123/stream')

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'message',
    content: 'Add logging to functions'
  }))
}

ws.onmessage = (event) => {
  const data = JSON.parse(event.data)
  
  switch (data.type) {
    case 'text-delta':
      console.log('Text:', data.text)
      break
    case 'tool-call':
      console.log('Tool:', data.tool, data.args)
      break
    case 'complete':
      console.log('Done:', data)
      ws.close()
      break
  }
}
```

---

## Server Implementation

### Hono Framework

```typescript
import { Hono } from 'hono'
import { cors } from 'hono/cors'

const app = new Hono()

// Middleware
app.use('*', cors())
app.use('*', logger())

// Routes
app.post('/sessions', async (c) => {
  const body = await c.req.json()
  const session = await Session.create(body)
  return c.json(session)
})

app.get('/sessions', async (c) => {
  const sessions = await Session.list()
  return c.json(sessions)
})

app.post('/sessions/:id/messages', async (c) => {
  const { id } = c.req.param()
  const body = await c.req.json()
  
  // Stream response
  return c.stream(async (stream) => {
    await SessionPrompt.prompt({
      sessionID: id,
      parts: body.parts,
      onChunk: (chunk) => stream.write(JSON.stringify(chunk))
    })
  })
})

// Start server
export default {
  port: 3000,
  fetch: app.fetch,
}
```

---

## Error Handling

### Standard Errors

```json
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session not found",
    "details": {
      "sessionId": "session_abc123"
    }
  }
}
```

### Error Codes

- `SESSION_NOT_FOUND` - Invalid session ID
- `INVALID_REQUEST` - Malformed request
- `PERMISSION_DENIED` - Operation not allowed
- `RATE_LIMITED` - Too many requests
- `INTERNAL_ERROR` - Server error

---

## Authentication

### API Keys (Future)

```bash
curl http://localhost:8080/sessions \
  -H "Authorization: Bearer sk-opencode-..."
```

### Local Only (Current)

Server only listens on localhost by default for security.

---

## CORS Configuration

For web clients:
```bash
opencode serve --cors
```

Or in config:
```json
{
  "server": {
    "cors": {
      "origin": ["http://localhost:3000"],
      "credentials": true
    }
  }
}
```

---

## Client SDKs

### JavaScript/TypeScript

```typescript
import { OpenCodeClient } from '@opencode/client'

const client = new OpenCodeClient('http://localhost:8080')

// Create session
const session = await client.sessions.create({
  provider: 'anthropic',
  model: 'claude-3-5-sonnet'
})

// Send message
for await (const chunk of client.sessions.message(session.id, {
  text: 'Add tests'
})) {
  console.log(chunk)
}
```

### Python

```python
from opencode import Client

client = Client("http://localhost:8080")

# Create session
session = client.sessions.create(
    provider="anthropic",
    model="claude-3-5-sonnet"
)

# Send message
for chunk in client.sessions.message(session.id, text="Add tests"):
    print(chunk)
```

---

## Best Practices

**Performance**:
- Use streaming for long responses
- Implement connection pooling
- Add caching where appropriate
- Monitor latency

**Security**:
- Only expose on localhost (default)
- Use HTTPS in production
- Implement authentication
- Validate all inputs

**Scalability**:
- Run multiple server instances
- Load balance across instances
- Use persistent storage
- Implement session cleanup

---

For implementation, see `packages/opencode/src/server/`.

