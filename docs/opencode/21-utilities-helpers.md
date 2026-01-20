# OpenCode - Utilities & Helpers

> **Common utilities, helpers, and shared functionality**

---

## Overview

OpenCode's utility modules provide shared functionality across the codebase:
- **Logging** - Structured logging
- **Error handling** - Error types and formatting
- **Locking** - Concurrency control
- **Context management** - Async context tracking
- **Deferred promises** - Promise utilities
- **Filesystem** - Path and file helpers

**Files**: `packages/opencode/src/util/`

---

## Logging

**File**: `util/log.ts`

### Creating Loggers

```typescript
import { Log } from "./util/log"

const log = Log.create({ service: "session" })

log.info("Session created", { sessionID: "abc123" })
log.error("Failed to load", { error })
log.debug("Internal state", { state })
```

### Log Levels

```typescript
log.debug("Detailed debug info")  // DEBUG=*
log.info("Informational message") // Always shown
log.warn("Warning message")        // Always shown
log.error("Error occurred", { error })  // Always shown
```

### Structured Logging

```typescript
log.info("Tool executed", {
  tool: "read",
  args: { filePath: "auth.ts" },
  duration: 123,
  result: "success"
})

// Output:
// [12:34:56] INFO (session): Tool executed
//   tool: read
//   args: { filePath: "auth.ts" }
//   duration: 123ms
//   result: success
```

### Tagged Loggers

```typescript
const sessionLog = log.clone().tag("session", "abc123")

sessionLog.info("Message sent")
// [12:34:56] INFO (session) [session=abc123]: Message sent
```

---

## Error Handling

**File**: `util/error.ts`

### Named Errors

```typescript
import { NamedError } from "./util/error"

throw new NamedError("SESSION_NOT_FOUND", "Session does not exist", {
  sessionID: "abc123"
})
```

### Error Formatting

```typescript
function formatError(error: unknown): {
  code: string
  message: string
  details?: Record<string, any>
} {
  if (error instanceof NamedError) {
    return {
      code: error.name,
      message: error.message,
      details: error.details
    }
  }
  
  if (error instanceof Error) {
    return {
      code: "UNKNOWN_ERROR",
      message: error.message
    }
  }
  
  return {
    code: "UNKNOWN_ERROR",
    message: String(error)
  }
}
```

### Error Types

```typescript
// Common error codes
export const ErrorCode = {
  SESSION_NOT_FOUND: "SESSION_NOT_FOUND",
  PERMISSION_DENIED: "PERMISSION_DENIED",
  INVALID_INPUT: "INVALID_INPUT",
  TOOL_EXECUTION_FAILED: "TOOL_EXECUTION_FAILED",
  RATE_LIMITED: "RATE_LIMITED",
  INTERNAL_ERROR: "INTERNAL_ERROR",
}
```

---

## Locking

**File**: `util/lock.ts`

### Simple Locks

```typescript
import { Lock } from "./util/lock"

const locks = new Map<string, Lock>()

async function withLock<T>(
  key: string,
  fn: () => Promise<T>
): Promise<T> {
  const lock = locks.get(key) ?? new Lock()
  locks.set(key, lock)
  
  await lock.acquire()
  try {
    return await fn()
  } finally {
    lock.release()
    if (lock.isEmpty()) {
      locks.delete(key)
    }
  }
}

// Usage
await withLock("session_123", async () => {
  // Critical section
  await updateSession()
})
```

### Advanced Locking

```typescript
// With timeout
const acquired = await lock.acquire({ timeout: 5000 })
if (!acquired) {
  throw new Error("Lock timeout")
}

// With abort signal
const controller = new AbortController()
await lock.acquire({ signal: controller.signal })

// Later: cancel
controller.abort()
```

---

## Deferred Promises

**File**: `util/defer.ts`

### Creating Deferred Promises

```typescript
import { defer } from "./util/defer"

const deferred = defer<string>()

// Resolve later
setTimeout(() => {
  deferred.resolve("value")
}, 1000)

// Wait for resolution
const result = await deferred.promise
console.log(result) // "value"
```

### Use Cases

```typescript
// Queue processing
const queue = new Map<string, Deferred<Result>>()

function enqueue(id: string): Promise<Result> {
  const deferred = defer<Result>()
  queue.set(id, deferred)
  return deferred.promise
}

function complete(id: string, result: Result) {
  const deferred = queue.get(id)
  if (deferred) {
    deferred.resolve(result)
    queue.delete(id)
  }
}

// Permission requests
const pending = defer<boolean>()

showPermissionDialog({
  onApprove: () => pending.resolve(true),
  onDeny: () => pending.resolve(false)
})

const approved = await pending.promise
```

---

## Context Management

**File**: `util/context.ts`

### Async Local Storage

```typescript
import { AsyncLocalStorage } from "async_hooks"

const context = new AsyncLocalStorage<ContextData>()

// Set context
context.run({ sessionID: "abc123" }, async () => {
  // Context available in all async calls
  await processMessage()
})

// Get context anywhere
function getCurrentSession(): string | undefined {
  return context.getStore()?.sessionID
}
```

### Request Context

```typescript
interface RequestContext {
  requestID: string
  sessionID: string
  userID?: string
  startTime: number
}

const requestContext = new AsyncLocalStorage<RequestContext>()

// Middleware
app.use(async (c, next) => {
  await requestContext.run({
    requestID: ulid(),
    sessionID: c.req.header("X-Session-ID"),
    startTime: Date.now()
  }, () => next())
})

// Access in handlers
function logOperation(operation: string) {
  const ctx = requestContext.getStore()
  log.info(operation, {
    requestID: ctx?.requestID,
    sessionID: ctx?.sessionID,
    duration: Date.now() - (ctx?.startTime ?? 0)
  })
}
```

---

## Filesystem Utilities

**File**: `util/filesystem.ts`

### Path Operations

```typescript
import { Filesystem } from "./util/filesystem"

// Find file upwards
const found = await Filesystem.findUp(
  "package.json",
  "/current/dir",
  "/root"
)

// Glob pattern upwards
const configs = await Filesystem.globUp(
  "*.config.{js,ts}",
  "/current/dir",
  "/root"
)

// Check containment
const contains = Filesystem.contains(
  "/workspace",
  "/workspace/src/file.ts"
)
```

### File Utilities

```typescript
// Ensure directory exists
await Filesystem.ensureDir("/path/to/dir")

// Copy file
await Filesystem.copy("/src/file.ts", "/dest/file.ts")

// Move file
await Filesystem.move("/old/path.ts", "/new/path.ts")

// Delete recursively
await Filesystem.remove("/path/to/dir")
```

---

## Lazy Initialization

**File**: `util/lazy.ts`

### Lazy Values

```typescript
import { lazy } from "./util/lazy"

const expensiveValue = lazy(async () => {
  // Computed only once
  return await computeExpensiveValue()
})

// First call: computes value
const value1 = await expensiveValue()

// Second call: returns cached
const value2 = await expensiveValue()

// value1 === value2
```

---

## PTY System

Pseudo-terminal handling for bash execution:

**PTY Creation**:
```typescript
const pty = createPty({
  shell: '/bin/bash',
  cwd: projectPath,
  env: process.env,
  cols: 80,
  rows: 24
})
```

**Signal Handling**:
- `SIGINT` (Ctrl+C) - Interrupt
- `SIGTERM` - Terminate
- `SIGKILL` - Force kill

**Buffer Management**:
- Output buffered up to 100KB
- Older output discarded when limit reached
- Configurable via `OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH`

---

## Shell Detection

Detects user's preferred shell:

```typescript
const shell = detectShell()
// Returns: 'bash', 'zsh', 'fish', 'pwsh', etc.
```

**Detection Order**:
1. `SHELL` environment variable
2. `/etc/passwd` entry
3. System default

**Platform Differences**:
| Platform | Default Shell |
|----------|---------------|
| macOS | zsh |
| Linux | bash |
| Windows | pwsh / cmd |

---

## ID Generation

ULID-based identifier generation:

```typescript
import { generateId } from 'opencode/id'

const id = generateId()
// Returns: "01ARZ3NDEKTSV4RRFFQ69G5FAV"
```

**ULID Format**:
- 26 characters
- Timestamp prefix (sortable)
- Random suffix (collision-resistant)
- URL-safe characters

**Usage**:
- Session IDs
- Message IDs
- Tool call IDs

---

## Format Utilities

Text formatting utilities:

```typescript
import { format } from 'opencode/format'

// Truncate long text
format.truncate(text, 100)

// Format file size
format.fileSize(1024 * 1024)  // "1 MB"

// Format duration
format.duration(65000)  // "1m 5s"

// Format timestamp
format.timestamp(Date.now())  // "2 minutes ago"
```

---

## Storage Layer

Key-value storage abstraction:

```typescript
import { storage } from 'opencode/storage'

// Set value
await storage.set('key', { data: 'value' })

// Get value
const value = await storage.get('key')

// Delete value
await storage.delete('key')

// List keys
const keys = await storage.keys('prefix:*')
```

**Backends**:
- File-based (default): `~/.opencode/data/`
- SQLite (optional): Better for large datasets
- Memory (testing): Non-persistent

---

## Best Practices

**Logging**:
- Use structured logging with context
- Tag loggers for filtering
- Log at appropriate levels
- Don't log sensitive data

**Error Handling**:
- Use NamedError for known errors
- Include relevant context
- Handle errors at boundaries
- Log errors with stack traces

**Locking**:
- Always release locks
- Use try/finally
- Set timeouts
- Avoid deadlocks

**Performance**:
- Use lazy initialization
- Cache expensive operations
- Implement timeouts
- Monitor resource usage

---

For implementation, see `packages/opencode/src/util/`.



---

# Enhanced UI & Utilities Documentation

---

## TUI Keyboard Shortcuts

### Navigation

| Shortcut | Action |
|----------|--------|
| `↑` / `k` | Move up |
| `↓` / `j` | Move down |
| `←` / `h` | Move left / collapse |
| `→` / `l` | Move right / expand |
| `PgUp` | Page up |
| `PgDn` | Page down |
| `Home` | Go to top |
| `End` | Go to bottom |
| `Tab` | Next panel |
| `Shift+Tab` | Previous panel |

### Editing

| Shortcut | Action |
|----------|--------|
| `Enter` | Submit message |
| `Shift+Enter` | New line in message |
| `Ctrl+C` | Cancel current operation |
| `Ctrl+D` | Exit OpenCode |
| `Ctrl+L` | Clear screen |
| `Ctrl+U` | Clear input line |
| `Ctrl+W` | Delete word backward |
| `Ctrl+K` | Delete to end of line |

### Session Management

| Shortcut | Action |
|----------|--------|
| `Ctrl+N` | New session |
| `Ctrl+O` | Open session picker |
| `Ctrl+S` | Share session |
| `Ctrl+E` | Export session |
| `Ctrl+Z` | Undo (restore snapshot) |
| `Ctrl+Y` | Redo |

### Tool Interaction

| Shortcut | Action |
|----------|--------|
| `y` | Approve tool execution |
| `n` | Deny tool execution |
| `a` | Always approve this tool |
| `e` | Edit tool parameters |
| `?` | Show tool help |

### View Controls

| Shortcut | Action |
|----------|--------|
| `Ctrl+\\` | Toggle sidebar |
| `Ctrl+/` | Toggle help |
| `Ctrl+P` | Command palette |
| `F1` | Help |
| `F2` | Settings |
| `F3` | Sessions |
| `F4` | Files |

---

## Desktop Application Features

### Unique Features (vs TUI)

| Feature | Desktop | TUI |
|---------|---------|-----|
| File drag & drop | Yes | No |
| Image preview | Yes | Limited |
| Clickable links | Yes | Limited |
| Multiple windows | Yes | No |
| System tray | Yes | No |
| Notifications | Native | Terminal |
| Clipboard images | Yes | No |
| Themes | Full | ANSI colors |

### SolidJS Architecture

The desktop app uses SolidJS for reactive UI:

```
packages/desktop/
├── src/
│   ├── App.tsx          # Main app component
│   ├── components/      # UI components
│   │   ├── Chat.tsx
│   │   ├── Sidebar.tsx
│   │   ├── ToolPanel.tsx
│   │   └── Settings.tsx
│   ├── stores/          # State management
│   │   ├── session.ts
│   │   ├── settings.ts
│   │   └── theme.ts
│   └── utils/           # Utilities
└── electron/            # Electron main process
```

### Native Integration

- **File System**: Native file dialogs
- **Clipboard**: System clipboard access
- **Notifications**: OS notifications
- **Auto-update**: Electron auto-updater
- **Menu**: Native application menu

### Settings

```json
{
  "desktop": {
    "theme": "system",  // "light", "dark", "system"
    "fontSize": 14,
    "fontFamily": "JetBrains Mono",
    "minimizeToTray": true,
    "startMinimized": false,
    "showNotifications": true
  }
}
```

---

## Server HTTP API

### REST Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Health check |
| `GET` | `/api/sessions` | List sessions |
| `POST` | `/api/sessions` | Create session |
| `GET` | `/api/sessions/:id` | Get session |
| `DELETE` | `/api/sessions/:id` | Delete session |
| `POST` | `/api/sessions/:id/messages` | Send message |
| `GET` | `/api/sessions/:id/messages` | Get messages |
| `GET` | `/api/models` | List available models |
| `GET` | `/api/tools` | List available tools |

### WebSocket Events

Connect to `/ws` for real-time updates:

```typescript
const ws = new WebSocket('ws://localhost:8080/ws')

ws.onmessage = (event) => {
  const message = JSON.parse(event.data)
  switch (message.type) {
    case 'message':
      // New message received
      break
    case 'tool_call':
      // Tool execution started
      break
    case 'tool_result':
      // Tool execution completed
      break
    case 'stream':
      // Streaming chunk
      break
  }
}
```

### Authentication

When `OPENCODE_SERVER_USERNAME` and `OPENCODE_SERVER_PASSWORD` are set:

```bash
curl -u admin:password http://localhost:8080/api/sessions
```

Or via header:
```bash
curl -H "Authorization: Basic YWRtaW46cGFzc3dvcmQ=" http://localhost:8080/api/sessions
```

### Rate Limits

Default rate limits (configurable):
- 100 requests/minute per IP
- 10 concurrent connections per IP
- 1MB max request body

---

## File System Operations

### File Watcher

The file watcher monitors project files for changes:

**Watch Events**:
| Event | Description |
|-------|-------------|
| `create` | File created |
| `modify` | File modified |
| `delete` | File deleted |
| `rename` | File renamed |

**Debouncing**:
- Changes debounced by 100ms default
- Prevents rapid-fire events during saves

**Ignore Patterns**:
Default ignores:
- `node_modules/`
- `.git/`
- `dist/`, `build/`
- `*.log`
- OS files (`.DS_Store`, `Thumbs.db`)

**Configuration**:
```json
{
  "fileWatcher": {
    "enabled": true,
    "debounceMs": 100,
    "ignore": ["custom-ignore/**"]
  }
}
```

**Performance**:
- Uses OS-native watching (fsevents, inotify)
- Scales to large projects (10K+ files)
- Minimal CPU usage

---

## Utilities & Helpers

### PTY System

Pseudo-terminal handling for bash execution:

**PTY Creation**:
```typescript
const pty = createPty({
  shell: '/bin/bash',
  cwd: projectPath,
  env: process.env,
  cols: 80,
  rows: 24
})
```

**Signal Handling**:
- `SIGINT` (Ctrl+C) - Interrupt
- `SIGTERM` - Terminate
- `SIGKILL` - Force kill

**Buffer Management**:
- Output buffered up to 100KB
- Older output discarded when limit reached
- Configurable via `OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH`

### Shell Detection

Detects user's preferred shell:

```typescript
const shell = detectShell()
// Returns: 'bash', 'zsh', 'fish', 'pwsh', etc.
```

**Detection Order**:
1. `SHELL` environment variable
2. `/etc/passwd` entry
3. System default

**Platform Differences**:
| Platform | Default Shell |
|----------|---------------|
| macOS | zsh |
| Linux | bash |
| Windows | pwsh / cmd |

### ID Generation

ULID-based identifier generation:

```typescript
import { generateId } from 'opencode/id'

const id = generateId()
// Returns: "01ARZ3NDEKTSV4RRFFQ69G5FAV"
```

**ULID Format**:
- 26 characters
- Timestamp prefix (sortable)
- Random suffix (collision-resistant)
- URL-safe characters

**Usage**:
- Session IDs
- Message IDs
- Tool call IDs

### Format Utilities

Text formatting utilities:

```typescript
import { format } from 'opencode/format'

// Truncate long text
format.truncate(text, 100)

// Format file size
format.fileSize(1024 * 1024)  // "1 MB"

// Format duration
format.duration(65000)  // "1m 5s"

// Format timestamp
format.timestamp(Date.now())  // "2 minutes ago"
```

### Storage Layer

Key-value storage abstraction:

```typescript
import { storage } from 'opencode/storage'

// Set value
await storage.set('key', { data: 'value' })

// Get value
const value = await storage.get('key')

// Delete value
await storage.delete('key')

// List keys
const keys = await storage.keys('prefix:*')
```

**Backends**:
- File-based (default): `~/.opencode/data/`
- SQLite (optional): Better for large datasets
- Memory (testing): Non-persistent

---

## Logging & Error Handling

### Log Levels

| Level | Description | When to Use |
|-------|-------------|-------------|
| `debug` | Detailed debugging | Development |
| `info` | General information | Normal operation |
| `warn` | Warning messages | Potential issues |
| `error` | Error messages | Failures |
| `fatal` | Critical errors | Unrecoverable |

### Log Format

```
2026-01-20T12:00:00.000Z [INFO] Session created session=abc123 project=/path
```

**Fields**:
- Timestamp (ISO 8601)
- Level
- Message
- Structured key=value pairs

### Error Types

```typescript
// API errors
class APIError extends Error {
  code: number
  message: string
}

// Network errors
class NetworkError extends Error {
  cause: Error
}

// Tool errors
class ToolError extends Error {
  tool: string
  parameters: object
}

// Validation errors
class ValidationError extends Error {
  field: string
  expected: string
  actual: string
}
```

### Error Recovery

| Error Type | Recovery Strategy |
|------------|-------------------|
| Network | Retry with backoff |
| API rate limit | Wait and retry |
| Tool failure | Report to LLM |
| Validation | Return helpful message |
| Unhandled | Log and recover state |

---

## Related Documentation

- [02-cli-reference.md](./02-cli-reference.md) - CLI commands
- [15-server-architecture.md](./15-server-architecture.md) - Server details
- [17-tui-implementation.md](./17-tui-implementation.md) - TUI architecture
- [18-desktop-application.md](./18-desktop-application.md) - Desktop app
