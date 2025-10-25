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

