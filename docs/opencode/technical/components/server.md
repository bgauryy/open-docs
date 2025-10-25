# Server

**Path**: `packages/opencode/src/server`
**Type**: API Layer
**File Count**: 3

## Description

HTTP server using Hono framework for remote access.

## Purpose

The server component provides an HTTP API for remote access to OpenCode functionality. It enables web clients, TUI, and other applications to interact with OpenCode over HTTP using RESTful APIs and Server-Sent Events (SSE).

## Key Features

- HTTP/REST API
- Server-Sent Events (SSE) for streaming
- Session management endpoints
- TUI communication endpoints
- CORS support
- Authentication support (optional)
- WebSocket support (future)

## Component Files

- `server.ts` - Main Hono server setup
- `project.ts` - Project-related routes
- `tui.ts` - TUI communication routes

## Dependencies

### Internal Dependencies
- `packages/opencode/src/session` - Session management (5 imports)

### External Dependencies
- `hono` - HTTP framework

## Usage

### Start Server

```bash
# Start on default port (3000)
opencode serve

# Start on custom port
opencode serve --port 8080

# Start with authentication
opencode serve --auth

# Start with custom host
opencode serve --host 0.0.0.0 --port 8080
```

### Programmatic Usage

```typescript
import { Server } from './server';

const server = new Server({
  port: 3000,
  host: 'localhost',
  cors: {
    origin: '*',
    methods: ['GET', 'POST', 'DELETE']
  },
  auth: false
});

await server.start();
console.log('Server running on http://localhost:3000');

// Graceful shutdown
process.on('SIGTERM', async () => {
  await server.stop();
});
```

## API Endpoints

### Session Management

#### Create Session
```http
POST /api/session
Content-Type: application/json

{
  "model": "claude-3-sonnet",
  "tools": ["bash", "edit", "read", "write"]
}

Response: 200 OK
{
  "session_id": "abc123",
  "created_at": "2025-10-25T14:30:00Z",
  "model": "claude-3-sonnet"
}
```

#### Send Message (SSE Stream)
```http
POST /api/session/:id/message
Content-Type: application/json

{
  "content": "help me build a feature"
}

Response: 200 OK (text/event-stream)
event: text
data: {"content": "I can help you with that..."}

event: tool_call
data: {"tool": "bash", "params": {"command": "ls"}}

event: done
data: {"tokens": 1234}
```

#### List Sessions
```http
GET /api/sessions

Response: 200 OK
{
  "sessions": [
    {
      "id": "abc123",
      "created_at": "2025-10-25T14:30:00Z",
      "last_message": "2025-10-25T15:00:00Z",
      "model": "claude-3-sonnet",
      "message_count": 5
    }
  ]
}
```

#### Get Session
```http
GET /api/session/:id

Response: 200 OK
{
  "id": "abc123",
  "created_at": "2025-10-25T14:30:00Z",
  "model": "claude-3-sonnet",
  "messages": [
    {
      "role": "user",
      "content": "help me build a feature",
      "timestamp": "2025-10-25T14:30:00Z"
    },
    {
      "role": "assistant",
      "content": "I can help...",
      "timestamp": "2025-10-25T14:30:05Z"
    }
  ]
}
```

#### Delete Session
```http
DELETE /api/session/:id

Response: 200 OK
{
  "success": true
}
```

### TUI Endpoints

#### Event Stream
```http
GET /tui/events
Accept: text/event-stream

Response: 200 OK (text/event-stream)
event: session_created
data: {"session_id": "abc123"}

event: message
data: {"type": "text", "content": "Hello"}

event: tool_call
data: {"tool": "bash", "command": "ls"}
```

#### Send Input
```http
POST /tui/input
Content-Type: application/json

{
  "type": "keypress",
  "key": "enter"
}

Response: 200 OK
{
  "success": true
}
```

### Health Check

```http
GET /health

Response: 200 OK
{
  "status": "ok",
  "uptime": 3600,
  "version": "1.0.0"
}
```

## Server-Sent Events (SSE)

The server uses SSE for real-time communication:

### Event Types

#### text
```
event: text
data: {"content": "chunk of text"}
```

#### tool_call
```
event: tool_call
data: {"tool": "bash", "params": {"command": "ls"}}
```

#### tool_result
```
event: tool_result
data: {"result": "file1.txt\nfile2.txt"}
```

#### error
```
event: error
data: {"message": "An error occurred"}
```

#### done
```
event: done
data: {"tokens": 1234, "duration": 5000}
```

## Client Examples

### JavaScript/TypeScript Client

```typescript
// Create session
const response = await fetch('http://localhost:3000/api/session', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    model: 'claude-3-sonnet'
  })
});
const { session_id } = await response.json();

// Send message and stream response
const eventSource = new EventSource(
  `http://localhost:3000/api/session/${session_id}/message`
);

eventSource.addEventListener('text', (event) => {
  const data = JSON.parse(event.data);
  console.log(data.content);
});

eventSource.addEventListener('done', (event) => {
  console.log('Response complete');
  eventSource.close();
});

eventSource.addEventListener('error', (error) => {
  console.error('Connection error:', error);
  eventSource.close();
});
```

### Python Client

```python
import requests
import json
import sseclient

# Create session
response = requests.post('http://localhost:3000/api/session', json={
    'model': 'claude-3-sonnet'
})
session_id = response.json()['session_id']

# Send message and stream response
response = requests.post(
    f'http://localhost:3000/api/session/{session_id}/message',
    json={'content': 'help me build a feature'},
    stream=True
)

client = sseclient.SSEClient(response)
for event in client.events():
    if event.event == 'text':
        data = json.loads(event.data)
        print(data['content'], end='', flush=True)
    elif event.event == 'done':
        break
```

### cURL Examples

```bash
# Create session
SESSION_ID=$(curl -X POST http://localhost:3000/api/session \
  -H "Content-Type: application/json" \
  -d '{"model": "claude-3-sonnet"}' | jq -r '.session_id')

# Send message
curl -X POST http://localhost:3000/api/session/$SESSION_ID/message \
  -H "Content-Type: application/json" \
  -d '{"content": "help me build a feature"}' \
  --no-buffer

# List sessions
curl http://localhost:3000/api/sessions

# Delete session
curl -X DELETE http://localhost:3000/api/session/$SESSION_ID
```

## CORS Configuration

```typescript
const server = new Server({
  cors: {
    origin: ['http://localhost:3000', 'https://app.example.com'],
    methods: ['GET', 'POST', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true
  }
});
```

## Authentication

### Token-Based Auth

```typescript
const server = new Server({
  auth: {
    enabled: true,
    tokens: ['secret-token-1', 'secret-token-2']
  }
});

// Clients must provide Authorization header
fetch('http://localhost:3000/api/session', {
  headers: {
    'Authorization': 'Bearer secret-token-1'
  }
});
```

### Custom Auth Handler

```typescript
const server = new Server({
  auth: {
    enabled: true,
    handler: async (token) => {
      // Custom token validation
      const user = await validateToken(token);
      return user !== null;
    }
  }
});
```

## Error Responses

```json
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session not found: abc123",
    "status": 404
  }
}
```

### Error Codes

- `SESSION_NOT_FOUND` (404)
- `INVALID_REQUEST` (400)
- `UNAUTHORIZED` (401)
- `RATE_LIMIT_EXCEEDED` (429)
- `INTERNAL_ERROR` (500)

## Rate Limiting

```typescript
const server = new Server({
  rateLimit: {
    maxRequests: 100,      // Per minute
    maxSessions: 10,       // Per IP
    windowMs: 60000        // 1 minute
  }
});
```

## Middleware

```typescript
import { Server } from './server';

const server = new Server({ port: 3000 });

// Add custom middleware
server.use(async (c, next) => {
  console.log(`${c.req.method} ${c.req.url}`);
  await next();
});

// Add logging
server.use(async (c, next) => {
  const start = Date.now();
  await next();
  const duration = Date.now() - start;
  console.log(`Request took ${duration}ms`);
});
```

## WebSocket Support (Future)

Planned WebSocket support for bidirectional communication:

```typescript
// Future API
const ws = new WebSocket('ws://localhost:3000/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'create_session',
    model: 'claude-3-sonnet'
  }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(data);
};
```

## Performance Considerations

- Connection pooling
- Session caching
- SSE connection management
- Non-blocking I/O
- Graceful shutdown
- Request timeout handling

## Monitoring

```typescript
const server = new Server({ port: 3000 });

// Track metrics
server.on('request', (req) => {
  metrics.increment('requests.total');
  metrics.timing('requests.duration', req.duration);
});

server.on('session_created', () => {
  metrics.increment('sessions.created');
});

server.on('error', (error) => {
  metrics.increment('errors.total');
  console.error(error);
});
```

## Related Documentation

- [Server Mode Flow](../flows/serve-command-flow.md)
- [Serve Command API](../api-reference.md#servecommand)
- [Session Management](./session.md)
