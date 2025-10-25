# OpenCode - Session Management

> **Complete guide to OpenCode's session lifecycle, state management, and persistence**

---

## Table of Contents

- [Overview](#overview)
- [Session Structure](#session-structure)
- [Session Lifecycle](#session-lifecycle)
- [Message Management](#message-management)
- [Locking Mechanism](#locking-mechanism)
- [Session Compaction](#session-compaction)
- [Session Operations](#session-operations)
- [Storage Layer](#storage-layer)
- [Events](#events)

---

## Overview

Sessions are the core abstraction in OpenCode for managing conversations with AI models. Each session represents:

- A conversation thread with context and history
- A workspace tied to a specific directory
- A collection of messages and their parts
- Persistent state across restarts
- Optional sharing and collaboration

**Key Files**:
- `session/index.ts` - Core session management (430 lines)
- `session/message-v2.ts` - Message handling (18KB)
- `session/lock.ts` - Concurrency control (95 lines)
- `session/compaction.ts` - History management (326 lines)
- `session/revert.ts` - Undo functionality
- `session/summary.ts` - Session summarization
- `session/todo.ts` - Task tracking

---

## Session Structure

### Session Info Schema

```typescript
export const Info = z.object({
  id: Identifier.schema("session"),           // Unique session ID
  projectID: z.string(),                      // Parent project
  directory: z.string(),                      // Working directory
  parentID: Identifier.schema("session").optional(), // Fork parent
  
  summary: z.object({
    diffs: Snapshot.FileDiff.array(),
  }).optional(),
  
  share: z.object({
    url: z.string(),                          // Share URL if shared
  }).optional(),
  
  title: z.string(),                          // Display title
  version: z.string(),                        // OpenCode version
  
  time: z.object({
    created: z.number(),                      // Creation timestamp
    updated: z.number(),                      // Last update
    compacting: z.number().optional(),        // Compaction in progress
  }),
  
  revert: z.object({
    messageID: z.string(),
    partID: z.string().optional(),
    snapshot: z.string().optional(),
    diff: z.string().optional(),
  }).optional(),                              // Revert state
})
```

### Session Types

#### **Root Session**
- No `parentID`
- Represents main conversation thread
- Can be forked into child sessions
- Auto-titled: `"New session - <timestamp>"`

```typescript
const rootSession: Session.Info = {
  id: "sess_01234567",
  projectID: "proj_abcdefgh",
  directory: "/Users/me/project",
  title: "New session - 2025-01-15T10:30:00.000Z",
  version: "0.15.17",
  time: {
    created: 1736936400000,
    updated: 1736936400000,
  }
}
```

#### **Child Session (Fork)**
- Has `parentID` pointing to parent session
- Inherits conversation context up to fork point
- Independent from parent after creation
- Auto-titled: `"Child session - <timestamp>"`

```typescript
const childSession: Session.Info = {
  id: "sess_89abcdef",
  projectID: "proj_abcdefgh",
  parentID: "sess_01234567",  // Fork from this session
  directory: "/Users/me/project",
  title: "Child session - 2025-01-15T11:00:00.000Z",
  version: "0.15.17",
  time: {
    created: 1736938200000,
    updated: 1736938200000,
  }
}
```

---

## Session Lifecycle

### 1. Creation

#### Root Session Creation

```typescript
// Create new root session
const session = await Session.create({
  title: "Implement auth system",  // Optional custom title
})

// Result
{
  id: "sess_01234567",
  projectID: "proj_current",
  directory: "/current/directory",
  title: "Implement auth system",
  version: "0.15.17",
  time: {
    created: 1736936400000,
    updated: 1736936400000,
  }
}
```

**Default Behavior**:
1. Generates descending session ID (newer = smaller)
2. Associates with current project
3. Records creation timestamp
4. Auto-shares if configured
5. Emits `Session.Event.Updated`

#### Forking a Session

```typescript
// Fork from specific message
const forked = await Session.fork({
  sessionID: "sess_01234567",
  messageID: "msg_12345678",  // Optional: fork from this point
})

// Result: New session with copied history up to messageID
```

**Fork Process**:
1. Create new child session with `parentID`
2. Copy all messages before `messageID` (or all if not specified)
3. Copy all parts for each message
4. Generate new IDs for everything
5. Return independent session

**Use Cases**:
- Explore alternative approaches
- Try different models on same context
- Branch conversation without losing original
- A/B test solutions

### 2. Initialization

After creation, sessions can be initialized with a welcome prompt:

```typescript
await Session.initialize({
  sessionID: "sess_01234567",
  providerID: "anthropic",
  modelID: "claude-3-5-sonnet-20241022",
  messageID: "msg_initial",
})
```

**Initialization Prompt** (`session/prompt/initialize.txt`):
```
You are an AI coding assistant in OpenCode.
Working directory: ${path}
...
```

### 3. Updates

#### Touch (Update Timestamp)

```typescript
// Mark session as active
await Session.touch("sess_01234567")
```

Updates `time.updated` to current timestamp.

#### Custom Updates

```typescript
// Update any session field
await Session.update("sess_01234567", (draft) => {
  draft.title = "New title"
  draft.summary = { diffs: [...] }
  // draft.time.updated automatically set
})
```

**Update Pattern**:
- Uses Immer-like draft editing
- Automatically updates `time.updated`
- Emits `Session.Event.Updated`
- Persists to storage

### 4. Sharing

#### Share Session

```typescript
// Enable sharing and get URL
const share = await Session.share("sess_01234567")

// Result
{
  url: "https://opencode.ai/s/01234567",
  secret: "secret_abc123..."
}
```

**Sharing Process**:
1. Check if sharing is enabled in config
2. Create share record with secret
3. Upload session info to share service
4. Upload all messages and parts
5. Return shareable URL
6. Update session with share info

**Auto-Share**:
```typescript
// Via flag
OPENCODE_AUTO_SHARE=1 opencode

// Via config
{
  "share": "auto"  // or "disabled"
}
```

#### Unshare Session

```typescript
// Revoke sharing
await Session.unshare("sess_01234567")
```

Removes share record and clears `share` from session info.

### 5. Deletion

```typescript
// Delete session (and all children)
await Session.remove("sess_01234567")
```

**Deletion Process**:
1. Get session info
2. Recursively delete all child sessions
3. Unshare if shared (ignore errors)
4. Delete all messages
5. Delete all parts for each message
6. Delete session record
7. Emit `Session.Event.Deleted`

**Cascade Delete**:
- Automatically deletes all child sessions
- Ensures no orphaned data
- Safe even if some steps fail

---

## Message Management

### Message Structure

```typescript
// Message info
interface MessageV2.Info {
  id: string                    // Unique message ID
  sessionID: string             // Parent session
  parentID?: string             // Previous message in thread
  role: "user" | "assistant"    // Message author
  system: string[]              // System prompts used
  mode: "build" | "chat"        // Conversation mode
  path: { cwd: string, root: string }
  
  // Assistant-only fields
  modelID?: string
  providerID?: string
  cost?: number
  tokens?: TokenUsage
  error?: ErrorInfo
  summary?: boolean             // True if compaction summary
  time?: { created: number, completed?: number }
}

// Message part (content chunk)
type MessageV2.Part = 
  | TextPart          // Text content
  | ReasoningPart     // Reasoning tokens
  | ToolPart          // Tool execution
  | RetryPart         // Retry attempt
  | StepStartPart     // Step begin
  | StepFinishPart    // Step end
```

### Message Operations

#### Create Message

```typescript
const msg = await Session.updateMessage({
  id: Identifier.ascending("message"),
  sessionID: "sess_01234567",
  role: "user",
  parentID: "msg_previous",  // Optional: link to previous
  system: ["You are an AI assistant"],
  mode: "build",
  path: { cwd: "/project", root: "/project" },
  time: { created: Date.now() },
})
```

#### Get Messages

```typescript
// Get all messages for session
const messages = await Session.messages("sess_01234567")

// Result: Array<{ info: MessageV2.Info, parts: MessageV2.Part[] }>
// Sorted by ID (ascending = chronological)
```

#### Get Single Message

```typescript
// Get specific message with parts
const message = await Session.getMessage({
  sessionID: "sess_01234567",
  messageID: "msg_12345678",
})

// Result: { info: MessageV2.Info, parts: MessageV2.Part[] }
```

#### Update Message

```typescript
// Update message (e.g., add cost/tokens)
await Session.updateMessage({
  ...msg,
  cost: 0.0012,
  tokens: { input: 1000, output: 500, ... },
})
```

Emits `MessageV2.Event.Updated`.

#### Delete Message

```typescript
await Session.removeMessage({
  sessionID: "sess_01234567",
  messageID: "msg_12345678",
})
```

Emits `MessageV2.Event.Removed`.

### Parts Management

#### Create/Update Part

```typescript
// Text part with delta streaming
await Session.updatePart({
  part: {
    type: "text",
    id: "part_123",
    sessionID: "sess_01234567",
    messageID: "msg_12345678",
    text: "Current text",
    time: { start: Date.now() },
  },
  delta: "new chunk",  // Streamed addition
})
```

**Part Types**:

| Type | Purpose | Key Fields |
|------|---------|------------|
| `text` | AI response text | `text`, `time` |
| `reasoning` | Reasoning tokens | `text`, `time` |
| `tool` | Tool execution | `tool`, `state`, `input`, `output` |
| `retry` | Retry attempt | `attempt`, `error` |
| `step-start` | Step begin marker | - |
| `step-finish` | Step end marker | - |

#### Get Parts

```typescript
// Get all parts for message
const parts = await Session.getParts("msg_12345678")

// Result: Array<MessageV2.Part> sorted by ID
```

Emits `MessageV2.Event.PartUpdated` on changes.

---

## Locking Mechanism

**File**: `session/lock.ts`

Sessions use an **advisory locking** system to prevent concurrent modifications.

### Lock Design

```typescript
class SessionLock {
  private locks: Map<string, LockState>
  
  acquire(sessionID: string): DisposableLock
  abort(sessionID: string): boolean
  isLocked(sessionID: string): boolean
  assertUnlocked(sessionID: string): void
}

type LockState = {
  controller: AbortController  // For cancellation
  created: number              // Lock timestamp
}
```

### Acquiring Locks

```typescript
// Automatic disposal with 'using'
await using lock = SessionLock.acquire({ sessionID: "sess_01234567" })

// Lock automatically released when scope exits
```

**Lock Features**:
- **AbortSignal** for cancellable operations
- **Symbol.dispose** for automatic cleanup
- **Timestamp** for debugging stuck locks
- **Force abort** on instance shutdown

### Lock Lifecycle

```
┌─────────────┐
│   Request   │
│   Lock      │
└──────┬──────┘
       │
       ▼
┌─────────────┐    Already    ┌──────────────┐
│  Check if   │───Locked─────▶│ Throw        │
│  Locked     │                │ LockedError  │
└──────┬──────┘                └──────────────┘
       │ Free
       ▼
┌─────────────┐
│  Acquire    │
│  Lock       │
└──────┬──────┘
       │
       ▼
┌─────────────┐    Operation  
│  Hold Lock  │    Complete
│  During Op  │               
└──────┬──────┘               
       │                      
       ▼                      
┌─────────────┐    Automatic
│  Release    │◀───(using)
│  Lock       │
└─────────────┘
```

### Error Handling

```typescript
try {
  await using lock = SessionLock.acquire({ sessionID })
  // Do work...
} catch (error) {
  if (SessionLock.LockedError.isInstance(error)) {
    console.log("Session is busy:", error.data.sessionID)
  }
}
```

### Manual Abort

```typescript
// Abort a locked session (force unlock)
const aborted = SessionLock.abort("sess_01234567")

if (aborted) {
  console.log("Session unlocked and aborted")
}
```

**Use Cases**:
- User cancels operation
- Timeout handling
- Cleanup on shutdown
- Debug stuck sessions

---

## Session Compaction

**File**: `session/compaction.ts`

As conversations grow, sessions need compaction to stay within model context limits.

### Why Compaction?

**Problem**: Long conversations exceed model context windows.

**Solution**: Two-phase compaction:
1. **Pruning** - Remove old tool outputs
2. **Summarization** - Create conversation summary

### Overflow Detection

```typescript
function isOverflow(input: {
  tokens: TokenUsage
  model: Model
}): boolean {
  const context = input.model.limit.context
  if (context === 0) return false
  
  const count = input.tokens.input + 
                input.tokens.cache.read + 
                input.tokens.output
  
  const output = Math.min(
    input.model.limit.output,
    SessionPrompt.OUTPUT_TOKEN_MAX
  )
  
  const usable = context - output
  return count > usable
}
```

**Triggers**:
- Token count exceeds usable context
- Before sending prompt to model
- Automatic if not disabled

### Phase 1: Pruning

**Goal**: Remove old tool outputs to free tokens.

```typescript
export const PRUNE_MINIMUM = 20_000   // Min tokens to prune
export const PRUNE_PROTECT = 40_000   // Stop pruning after this

await SessionCompaction.prune({ sessionID })
```

**Pruning Algorithm**:
1. Iterate messages **backwards** from most recent
2. Skip last 2 conversation turns (protect recent context)
3. Stop at first message with `summary: true`
4. Mark tool outputs as `compacted`
5. Continue until `PRUNE_PROTECT` tokens saved

**What Gets Pruned**:
- ✅ Tool outputs (bash stdout, file contents, etc.)
- ✅ Old parts older than summary
- ❌ User messages (always kept)
- ❌ Assistant text (always kept)
- ❌ Recent 2 turns (protected)

**Pruned Tool Part**:
```typescript
{
  type: "tool",
  tool: "bash",
  state: {
    status: "completed",
    output: "(pruned)",  // Original output removed
    time: {
      start: 1736936400000,
      completed: 1736936500000,
      compacted: 1736940000000,  // Pruning timestamp
    }
  }
}
```

### Phase 2: Summarization

**Goal**: Create AI-generated summary of conversation.

```typescript
await SessionCompaction.run({
  sessionID: "sess_01234567",
  providerID: "anthropic",
  modelID: "claude-3-5-sonnet-20241022",
  signal: abortSignal,  // Optional
})
```

**Summarization Process**:

```
1. Lock session
   ↓
2. Filter to non-compacted messages
   ↓
3. Build summary prompt
   ↓
4. Stream AI response
   ↓
5. Create summary message (summary: true)
   ↓
6. Mark as compaction point
   ↓
7. Emit Compacted event
   ↓
8. Unlock session
```

**Summary Prompt**:
```typescript
const messages = [
  ...systemPrompts,
  ...conversationHistory,
  {
    role: "user",
    content: "Provide a detailed but concise summary of our conversation above. Focus on information that would be helpful for continuing the conversation, including what we did, what we're doing, which files we're working on, and what we're going to do next."
  }
]
```

**Summary Message**:
```typescript
{
  id: "msg_summary",
  sessionID: "sess_01234567",
  role: "assistant",
  summary: true,  // ← Marks as compaction point
  system: ["Summarize conversation..."],
  time: {
    created: 1736940000000,
    completed: 1736940100000,
  },
  parts: [{
    type: "text",
    text: "We've been implementing authentication for the user service. We added JWT token generation, validation middleware, and updated the login endpoint. Currently working on the logout flow and planning to add refresh token support next. Key files: auth.ts, middleware.ts, user.service.ts",
    time: { start: 1736940000000, end: 1736940100000 }
  }]
}
```

### Compaction Retry Logic

**Max Retries**: 10 attempts

```typescript
const MAX_RETRIES = 10

let stream = doStream()
let result = await process(stream, { count: 0, max: MAX_RETRIES })

if (result.shouldRetry) {
  for (let retry = 1; retry < MAX_RETRIES; retry++) {
    const delayMs = SessionRetry.getRetryDelayInMs(error, retry)
    await SessionRetry.sleep(delayMs, signal)
    
    stream = doStream()
    result = await process(stream, { count: retry, max: MAX_RETRIES })
    
    if (!result.shouldRetry) break
  }
}
```

**Retry Conditions**:
- API rate limit errors
- Temporary network failures
- Transient provider issues

**Exponential Backoff**:
- Attempt 1: 1s
- Attempt 2: 2s
- Attempt 3: 4s
- Attempt N: min(2^N, 60)s

### Disabling Compaction

```bash
# Disable automatic compaction
OPENCODE_DISABLE_AUTOCOMPACT=1 opencode

# Disable pruning only
OPENCODE_DISABLE_PRUNE=1 opencode
```

---

## Session Operations

### List Sessions

```typescript
// Async iterator over all sessions in project
for await (const session of Session.list()) {
  console.log(session.title, session.time.updated)
}
```

Returns sessions in storage order (newest first).

### Get Session

```typescript
// Get specific session
const session = await Session.get("sess_01234567")
```

Throws `Storage.NotFoundError` if not found.

### Get Children

```typescript
// Get all child sessions (forks)
const children = await Session.children("sess_01234567")

// Result: Array<Session.Info> for all forks
```

### List All Data

```typescript
// Get complete session data
const session = await Session.get("sess_01234567")
const messages = await Session.messages("sess_01234567")
const children = await Session.children("sess_01234567")

// Complete session state
const complete = {
  session,
  messages,
  children,
}
```

---

## Storage Layer

Sessions are stored in the filesystem under `~/.opencode/data/`:

```
~/.opencode/data/
└── projects/
    └── {projectID}/
        ├── info.json
        └── sessions/
            ├── {sessionID}/
            │   ├── info.json               # Session.Info
            │   ├── messages/
            │   │   └── {messageID}.json    # MessageV2.Info
            │   └── snapshots/
            │       └── {snapshotID}/
            ├── {messageID}/
            │   └── parts/
            │       └── {partID}.json       # MessageV2.Part
            └── ...
```

### Storage Paths

```typescript
// Session info
["session", projectID, sessionID]
// → projects/{projectID}/sessions/{sessionID}/info.json

// Message info
["message", sessionID, messageID]
// → messages/{sessionID}/{messageID}.json

// Part
["part", messageID, partID]
// → parts/{messageID}/{partID}.json

// Share
["share", sessionID]
// → shares/{sessionID}.json
```

### Storage Operations

```typescript
// Write
await Storage.write(path, data)

// Read
const data = await Storage.read<T>(path)

// Update
const updated = await Storage.update<T>(path, (draft) => {
  draft.field = newValue
})

// Remove
await Storage.remove(path)

// List
const items = await Storage.list(pathPrefix)
```

---

## Events

Sessions emit events via the event bus:

### Session Events

```typescript
Session.Event.Updated
// Emitted: Session created/modified
// Data: { info: Session.Info }

Session.Event.Deleted
// Emitted: Session removed
// Data: { info: Session.Info }

Session.Event.Error
// Emitted: Error during session operation
// Data: { sessionID?: string, error: ErrorInfo }
```

### Message Events

```typescript
MessageV2.Event.Updated
// Emitted: Message created/modified
// Data: { info: MessageV2.Info }

MessageV2.Event.Removed
// Emitted: Message deleted
// Data: { sessionID: string, messageID: string }

MessageV2.Event.PartUpdated
// Emitted: Part created/modified/streamed
// Data: { part: MessageV2.Part, delta?: string }
```

### Compaction Events

```typescript
SessionCompaction.Event.Compacted
// Emitted: Session successfully compacted
// Data: { sessionID: string }
```

### Subscribing to Events

```typescript
// Subscribe
const unsubscribe = Bus.subscribe(
  Session.Event.Updated,
  (event) => {
    console.log("Session updated:", event.info.id)
  }
)

// Unsubscribe
unsubscribe()
```

---

## Best Practices

### 1. Always Use Locks

```typescript
// ✅ Good
await using lock = SessionLock.acquire({ sessionID })
// Automatic cleanup

// ❌ Bad
// No lock - race conditions possible
```

### 2. Handle Lock Errors

```typescript
try {
  await using lock = SessionLock.acquire({ sessionID })
  await doWork()
} catch (error) {
  if (SessionLock.LockedError.isInstance(error)) {
    // Session busy, retry or inform user
  }
}
```

### 3. Use Async Iteration

```typescript
// ✅ Good - properly closes iterator
for await (const session of Session.list()) {
  if (condition) break  // Iterator auto-closed
}

// ❌ Bad - iterator may leak
const iterator = Session.list()
for await (const session of iterator) {
  // No early return/break cleanup
}
```

### 4. Respect Compaction

```typescript
// Let compaction run automatically
// Only disable if you have good reason

// Check overflow before prompting
if (SessionCompaction.isOverflow({ tokens, model })) {
  await SessionCompaction.run({ sessionID, providerID, modelID })
}
```

### 5. Clean Up Forked Sessions

```typescript
// Delete unneeded forks
const children = await Session.children(parentID)
for (const child of children) {
  if (shouldDelete(child)) {
    await Session.remove(child.id)
  }
}
```

---

## Summary

OpenCode's session management provides:

- **Persistent conversations** with full history
- **Forking** for exploring alternatives
- **Locking** for safe concurrent access
- **Compaction** for long-running conversations
- **Sharing** for collaboration
- **Event-driven** updates for reactive UIs
- **File-based storage** for simplicity and transparency

Sessions are the foundation of OpenCode's conversation model, enabling powerful workflows while maintaining data integrity and user control.

---

## Next Steps

- **[04-prompt-processing.md](./04-prompt-processing.md)** - Prompt system deep dive
- **[05-system-prompts.md](./05-system-prompts.md)** - Custom prompts and agents
- **[09-state-management.md](./09-state-management.md)** - State persistence


