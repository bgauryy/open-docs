# OpenCode - Resource & Memory Management

> **OS-level resource management, memory optimization, and cleanup patterns**

---

## Table of Contents

- [Overview](#overview)
- [State Management](#state-management)
- [Memory Optimization](#memory-optimization)
- [Process Lifecycle](#process-lifecycle)
- [Resource Cleanup](#resource-cleanup)
- [Best Practices](#best-practices)

---

## Overview

OpenCode implements comprehensive resource management to ensure efficient memory usage and proper cleanup:

- **State Disposal** - Automatic cleanup of per-instance state
- **Session Compaction** - Token-based memory optimization
- **Process Management** - Child process lifecycle handling
- **Symbol.dispose** - Modern JavaScript resource management
- **LSP Cleanup** - Language server process termination

**Key Files**:
- `project/state.ts` - State disposal pattern
- `session/lock.ts` - Lock disposal with Symbol.dispose
- `session/compaction.ts` - Memory compaction
- `lsp/client.ts` - LSP process cleanup
- `bun/index.ts` - Process spawning

---

## State Management

**File**: `project/state.ts`

### State Disposal Pattern

OpenCode uses a disposal pattern for per-instance state cleanup:

```typescript
export namespace State {
  interface Entry {
    state: any
    dispose?: (state: any) => Promise<void>
  }
  
  const entries = new Map<string, Map<any, Entry>>()
  
  export function create<S>(
    root: () => string,
    init: () => S,
    dispose?: (state: Awaited<S>) => Promise<void>
  ) {
    return () => {
      const key = root()
      let collection = entries.get(key)
      
      if (!collection) {
        collection = new Map<string, Entry>()
        entries.set(key, collection)
      }
      
      const exists = collection.get(init)
      if (exists) return exists.state as S
      
      const state = init()
      collection.set(init, {
        state,
        dispose,
      })
      
      return state
    }
  }
  
  // Cleanup all state for a project
  export async function dispose(key: string) {
    for (const [_, entry] of entries.get(key)?.entries() ?? []) {
      if (!entry.dispose) continue
      await entry.dispose(await entry.state)
    }
  }
}
```

### Usage Example

```typescript
// Create state with cleanup
const myState = Instance.state(
  () => {
    return {
      connections: new Map(),
      cache: new Map(),
    }
  },
  async (state) => {
    // Cleanup on instance disposal
    state.connections.clear()
    state.cache.clear()
    console.log("State cleaned up")
  }
)

// Access state
const state = await myState()
state.connections.set(id, connection)

// Later: cleanup when project closes
await State.dispose(projectKey)
```

### Automatic Cleanup

When OpenCode changes directories or closes a project:

```typescript
// In project/instance.ts
export async function switchDirectory(newDir: string) {
  // Dispose old state
  await State.dispose(currentProjectKey)
  
  // Initialize new instance
  Instance.directory = newDir
}
```

---

## Memory Optimization

**File**: `session/compaction.ts`

### Token Overflow Detection

OpenCode monitors token usage to prevent context overflow:

```typescript
export namespace SessionCompaction {
  export function isOverflow(input: {
    tokens: MessageV2.Assistant["tokens"]
    model: ModelsDev.Model
  }) {
    if (Flag.OPENCODE_DISABLE_AUTOCOMPACT) return false
    
    const context = input.model.limit.context
    if (context === 0) return false
    
    // Check if token usage exceeds 80% of context limit
    const threshold = context * 0.8
    return input.tokens.total > threshold
  }
}
```

### Compaction Strategies

When token overflow is detected:

#### Strategy 1: Prune Old Tool Outputs

```typescript
// Remove old tool execution results while keeping:
// - User messages (always)
// - Recent tool calls
// - Tool call metadata
async function pruneOldToolOutputs(session: Session) {
  const messages = await Session.messages(session.id)
  
  for (const msg of messages) {
    if (msg.role === "assistant") {
      for (const part of msg.parts) {
        if (part.type === "tool" && part.age > THRESHOLD) {
          // Mark as compacted
          part.state.compacted = true
          // Keep only summary
          part.state.output = `[Compacted: ${part.tool}]`
        }
      }
    }
  }
}
```

#### Strategy 2: Summarize Old Conversation

```typescript
// If still over limit, summarize old messages
async function summarizeOldConversation(session: Session) {
  const messages = await Session.messages(session.id)
  
  // Group old messages (before recent N)
  const oldMessages = messages.slice(0, -10)
  
  // Send to AI for summarization
  const summary = await generateText({
    model: provider.language,
    system: PROMPT_SUMMARIZE,
    prompt: formatMessages(oldMessages),
  })
  
  // Replace old messages with summary
  await Session.addSummaryMessage({
    sessionID: session.id,
    content: summary.text,
    replacedMessages: oldMessages.map(m => m.id),
  })
}
```

### Automatic Compaction

```typescript
// Before each prompt processing
if (SessionCompaction.isOverflow({ tokens, model })) {
  await SessionCompaction.run({
    sessionID,
    providerID,
    modelID,
    signal: lock.signal,
  })
}
```

**Result**: Maintains conversation continuity while staying within model limits.

---

## Process Lifecycle

**File**: `lsp/client.ts`, `bun/index.ts`

### LSP Server Management

```typescript
// LSP client shutdown
export function shutdown(input: { server: LSPServer }) {
  return {
    async dispose() {
      log.info("shutting down")
      
      // Close JSON-RPC connection
      connection.end()
      connection.dispose()
      
      // Kill LSP server process
      input.server.process.kill()
      
      log.info("shutdown")
    }
  }
}
```

### Bun Process Spawning

```typescript
export namespace BunProc {
  export async function run(
    cmd: string[],
    options?: Bun.SpawnOptions.OptionsObject
  ) {
    const result = Bun.spawn([which(), ...cmd], {
      ...options,
      stdout: "pipe",
      stderr: "pipe",
      env: {
        ...process.env,
        ...options?.env,
        BUN_BE_BUN: "1",
      },
    })
    
    // Wait for completion
    const code = await result.exited
    
    // Read output
    const stdout = result.stdout 
      ? await readableStreamToText(result.stdout)
      : undefined
    const stderr = result.stderr
      ? await readableStreamToText(result.stderr)
      : undefined
    
    if (code !== 0) {
      throw new Error(`Command failed with exit code ${result.exitCode}`)
    }
    
    return result
  }
}
```

### Process Cleanup on Exit

```typescript
// In index.ts
const cancel = new AbortController()

process.on("unhandledRejection", (e) => {
  Log.Default.error("rejection", { e })
})

process.on("uncaughtException", (e) => {
  Log.Default.error("exception", { e })
})

// On exit
try {
  await cli.parse()
} catch (e) {
  Log.Default.error("fatal", data)
  process.exitCode = 1
}

// Abort all ongoing operations
cancel.abort()
```

---

## Resource Cleanup

### Symbol.dispose Pattern

**File**: `session/lock.ts`

Modern JavaScript disposal using `Symbol.dispose`:

```typescript
export function acquire(input: { sessionID: string }) {
  const controller = new AbortController()
  
  state().locks.set(input.sessionID, {
    controller,
    created: Date.now(),
  })
  
  return {
    signal: controller.signal,
    
    abort() {
      controller.abort()
      unset({ sessionID: input.sessionID, controller })
    },
    
    // Automatic cleanup with using/await using
    async [Symbol.dispose]() {
      const removed = unset({
        sessionID: input.sessionID,
        controller
      })
      
      if (removed) {
        log.info("unlocked", { sessionID: input.sessionID })
      }
    },
  }
}
```

### Using with Modern JavaScript

```typescript
// Automatic cleanup with 'using'
{
  using lock = SessionLock.acquire({ sessionID })
  
  // Lock held during block execution
  await processSession(sessionID)
  
  // Lock automatically released at end of block
}

// Or with async
{
  await using lock = SessionLock.acquire({ sessionID })
  await asyncWork()
  // Automatic cleanup after await
}
```

### Force Cleanup on Instance Change

```typescript
// In project/state.ts cleanup callback
const state = Instance.state(
  () => ({
    locks: new Map<string, LockState>()
  }),
  async (current) => {
    // Force abort all locks
    for (const [sessionID, lock] of current.locks) {
      log.info("force abort", { sessionID })
      lock.controller.abort()
    }
    current.locks.clear()
  }
)
```

---

## Best Practices

### 1. Always Cleanup Resources

**Good**:
```typescript
const state = Instance.state(
  () => ({ cache: new Map() }),
  async (state) => {
    state.cache.clear()  // Cleanup
  }
)
```

**Bad**:
```typescript
const state = Instance.state(
  () => ({ cache: new Map() })
  // No cleanup - memory leak!
)
```

### 2. Use Symbol.dispose

**Good**:
```typescript
{
  using resource = acquireResource()
  await useResource(resource)
  // Automatic cleanup
}
```

**Bad**:
```typescript
const resource = acquireResource()
try {
  await useResource(resource)
} finally {
  resource.cleanup()  // Manual, error-prone
}
```

### 3. Monitor Token Usage

```typescript
// Check before operations
const session = await Session.get(sessionID)
if (SessionCompaction.isOverflow({ tokens: session.tokens, model })) {
  await SessionCompaction.run({ sessionID })
}
```

### 4. Handle Process Termination

```typescript
// Clean shutdown
process.on("SIGINT", async () => {
  // Cleanup resources
  await State.dispose(projectKey)
  
  // Kill child processes
  for (const proc of childProcesses) {
    proc.kill()
  }
  
  process.exit(0)
})
```

### 5. Avoid Memory Leaks

**Common Issues**:
- ❌ Event listeners not removed
- ❌ Timers not cleared
- ❌ Large objects in closures
- ❌ Circular references

**Solutions**:
- ✅ Use AbortController for cleanup
- ✅ Clear maps and caches
- ✅ Remove event listeners
- ✅ Use WeakMap for references

---

## Memory Management Strategies

### Per-Instance Isolation

```typescript
// Each project instance has isolated state
export namespace Instance {
  export let directory: string
  export let worktree: string
  
  export function state<T>(
    init: () => T,
    cleanup?: (state: T) => Promise<void>
  ) {
    // State isolated per directory
    // Auto-cleanup on directory change
  }
}
```

### Cache Management

```typescript
// LRU-style cache with limits
const cache = new Map<string, CachedValue>()

function set(key: string, value: any) {
  // Remove oldest if over limit
  if (cache.size >= MAX_SIZE) {
    const firstKey = cache.keys().next().value
    cache.delete(firstKey)
  }
  
  cache.set(key, {
    value,
    timestamp: Date.now()
  })
}

// Periodic cleanup
setInterval(() => {
  const now = Date.now()
  for (const [key, cached] of cache) {
    if (now - cached.timestamp > TTL) {
      cache.delete(key)
    }
  }
}, CLEANUP_INTERVAL)
```

### Stream Processing

```typescript
// Avoid loading entire files into memory
for await (const chunk of stream) {
  // Process incrementally
  await processChunk(chunk)
}

// Instead of:
const entire = await file.text()  // Large memory allocation
```

---

## Monitoring & Debugging

### Memory Usage Tracking

```typescript
// Track memory in development
if (process.env.DEBUG) {
  const usage = process.memoryUsage()
  console.log({
    heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)}MB`,
    external: `${Math.round(usage.external / 1024 / 1024)}MB`,
  })
}
```

### Resource Leak Detection

```typescript
// Track active resources
const activeResources = new Set<string>()

function acquire(id: string) {
  activeResources.add(id)
  
  return {
    [Symbol.dispose]() {
      activeResources.delete(id)
    }
  }
}

// Check for leaks on shutdown
process.on("exit", () => {
  if (activeResources.size > 0) {
    console.warn(`Resource leak detected: ${activeResources.size} resources`)
    console.warn(Array.from(activeResources))
  }
})
```

---

## Summary

OpenCode's resource management provides:

- **Automatic cleanup** via State.dispose pattern
- **Memory optimization** through session compaction
- **Process management** with proper termination
- **Modern disposal** using Symbol.dispose
- **Per-instance isolation** preventing cross-project leaks
- **Token monitoring** to stay within limits

These patterns ensure OpenCode runs efficiently even during long sessions with large codebases, while properly cleaning up resources when switching projects or shutting down.

---

## Next Steps

- **[03-session-management.md](./03-session-management.md)** - Session lifecycle
- **[04-prompt-processing.md](./04-prompt-processing.md)** - Context management
- **[21-utilities-helpers.md](./21-utilities-helpers.md)** - Utility patterns

