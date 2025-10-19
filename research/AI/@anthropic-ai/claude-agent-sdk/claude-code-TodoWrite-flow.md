# TodoWrite Storage Mechanism - Under the Hood

## 🎯 TL;DR - How Tasks Are Stored

**Tasks are stored IN-MEMORY ONLY** in the CLI process, organized by agent ID.

- **Storage Type:** In-memory JavaScript object (not persisted to disk)
- **Storage Location:** `AppState.todos[agentId]`
- **Lifetime:** Exists only during the CLI process lifetime
- **Scope:** Per-agent (each agent has its own todo list)

---

## 📋 Complete Implementation

### TodoWrite Call Function (cli.js:907)

```javascript
async*call({todos:A},B){
  // 1. Get current todos for this agent
  let Z=(await B.getAppState()).todos[B.agentId]??[],

  // 2. Clear todos if all are completed, otherwise keep new list
  G=A.every((Y)=>Y.status==="completed")?[]:A;

  // 3. Update app state with new todos
  B.setAppState((Y)=>({
    ...Y,
    todos:{
      ...Y.todos,
      [B.agentId]:G
    }
  })),

  // 4. Return result with old and new todos
  yield{
    type:"result",
    data:{
      oldTodos:Z,
      newTodos:A
    }
  }
}
```

### Deconstructed Step-by-Step

#### Step 1: Retrieve Current Todos
```javascript
let Z = (await B.getAppState()).todos[B.agentId] ?? []
```

**What happens:**
- `B.getAppState()` - Async call to get the current application state
- `.todos` - Access the todos object in app state
- `[B.agentId]` - Get todos for this specific agent
- `?? []` - Default to empty array if none exist

**State Structure:**
```typescript
AppState = {
  todos: {
    "agent-uuid-1": [
      { content: "Task 1", status: "pending", activeForm: "..." },
      { content: "Task 2", status: "completed", activeForm: "..." }
    ],
    "agent-uuid-2": [
      { content: "Task 3", status: "in_progress", activeForm: "..." }
    ],
    "main-thread-id": [
      // Main agent's todos
    ]
  },
  // ... other state properties
}
```

#### Step 2: Determine New Todos
```javascript
G = A.every((Y) => Y.status === "completed") ? [] : A;
```

**Logic:**
- If **all tasks are completed** → Clear the list (empty array)
- Otherwise → Keep the new todo list

**Rationale:** Automatically clean up when all work is done.

#### Step 3: Update App State
```javascript
B.setAppState((Y) => ({
  ...Y,
  todos: {
    ...Y.todos,
    [B.agentId]: G
  }
}))
```

**What happens:**
- Spreads existing state `...Y`
- Updates the todos object:
  - Spreads existing todos `...Y.todos`
  - Overwrites this agent's todos `[B.agentId]: G`
- Immutable update pattern (creates new object)

**Result:**
```javascript
// Before
{ todos: { "agent-1": [old tasks] } }

// After
{ todos: { "agent-1": [new tasks] } }
```

#### Step 4: Yield Result
```javascript
yield {
  type: "result",
  data: {
    oldTodos: Z,
    newTodos: A
  }
}
```

Returns both old and new todos for reference.

---

## 🏗️ AppState Architecture

### What is AppState?

`AppState` is the CLI's **in-memory state management system**. It's similar to Redux or React Context but runs in the CLI process.

### AppState Structure (Inferred)

```typescript
interface AppState {
  // Todo lists keyed by agent ID
  todos: {
    [agentId: string]: Todo[]
  };

  // Permission context
  toolPermissionContext: {
    mode: PermissionMode;
    // ... other permission state
  };

  // File read cache
  // (from other parts of code)

  // Other runtime state...
}

interface Todo {
  content: string;              // Task description
  status: "pending" | "in_progress" | "completed";
  activeForm: string;           // Present tense form ("Running tests")
}
```

### AppState Management

**Get State:**
```javascript
const state = await B.getAppState();
```

**Set State:**
```javascript
B.setAppState((currentState) => ({
  ...currentState,
  todos: { ...newTodos }
}));
```

The `B` parameter in the `call()` function is the **tool execution context** provided by the CLI runtime.

---

## 🔑 Key Findings

### 1. **In-Memory Only**

❌ **NOT persisted to:**
- Session files (`.claude-sessions/`)
- Disk/database
- Cloud storage

✅ **Stored in:**
- CLI process memory
- JavaScript heap
- Lost when CLI process exits

### 2. **Per-Agent Isolation**

Each agent has its own todo list:

```javascript
// Main agent
todos["main-thread-id"] = [...]

// Subagent 1
todos["code-reviewer-uuid"] = [...]

// Subagent 2
todos["test-writer-uuid"] = [...]
```

**Why?**
- Prevents todo conflicts between agents
- Each agent tracks its own work
- Parent agent doesn't see child todos
- Clean separation of concerns

### 3. **Auto-Cleanup on Completion**

When all todos are marked `completed`, the list is automatically cleared:

```javascript
// All completed → clear list
todos = [
  { status: "completed", ... },
  { status: "completed", ... }
]
// Result: todos = []

// Some pending → keep list
todos = [
  { status: "completed", ... },
  { status: "pending", ... }
]
// Result: todos = [both tasks kept]
```

### 4. **Immutable Updates**

State updates use immutable patterns:

```javascript
// Spreads to create new objects
B.setAppState((Y) => ({
  ...Y,                    // Copy existing state
  todos: {
    ...Y.todos,            // Copy existing todos
    [B.agentId]: G         // Update this agent only
  }
}))
```

This prevents accidental mutations and state corruption.

---

## 🔄 Lifecycle

### Session Start
```
CLI Process Starts
    ↓
AppState Initialized
    ↓
todos = {}  (empty object)
```

### During Execution
```
Agent calls TodoWrite
    ↓
getAppState() → current todos
    ↓
setAppState() → updated todos
    ↓
Store in memory (todos[agentId])
```

### Session End
```
CLI Process Exits
    ↓
AppState Destroyed
    ↓
All todos LOST (not persisted)
```

---

## 🚫 What's NOT Stored

**Todos are NOT:**
- Written to session files
- Persisted between sessions
- Saved to `.claude-sessions/{uuid}/`
- Stored in transcripts
- Accessible across processes
- Recoverable after crash

**Todos ARE:**
- Ephemeral (exist during session only)
- In-memory only
- Per-agent isolated
- Automatically cleaned on completion

---

## 💡 Implications

### For Users

1. **Todos don't persist** - If CLI crashes, todos are lost
2. **Can't resume todos** - New session = new todo list
3. **Not in transcript** - Todos aren't saved in conversation history
4. **Per-agent scope** - Each agent tracks separately

### For Developers

1. **Simple state management** - No database, no files
2. **Fast access** - Memory reads/writes are instant
3. **No cleanup needed** - Memory freed on process exit
4. **Thread-safe** - Single process, single event loop

---

## 🔍 Why This Design?

### Advantages ✅

1. **Performance**: Instant reads/writes (no I/O)
2. **Simplicity**: No persistence logic needed
3. **Clean**: Auto-cleanup on process exit
4. **Isolation**: Each agent has separate state
5. **No corruption**: Can't have stale state across sessions

### Trade-offs ⚠️

1. **No persistence**: Lost on crash/exit
2. **No recovery**: Can't resume todos
3. **Memory usage**: Large todo lists use RAM
4. **No sharing**: Can't share todos between processes

---

## 📊 Comparison with Other Data

| Data Type | Storage | Persistence | Scope |
|-----------|---------|-------------|-------|
| **Todos** | In-memory | ❌ No | Per-agent |
| **Messages** | Session files | ✅ Yes | Per-session |
| **Transcripts** | `.jsonl` files | ✅ Yes | Per-session |
| **Permissions** | Session files | ✅ Yes | Per-session |
| **File Cache** | In-memory | ❌ No | Per-session |
| **MCP State** | In-memory | ❌ No | Per-server |

---

## 🛠️ How to Verify This

### Test 1: Check Session Files
```bash
# Start a session, create todos
echo "Create todos and exit"

# Check session directory
ls .claude-sessions/{uuid}/
# Result: No todos.json or similar file
```

### Test 2: Process Memory
```bash
# During session with todos
ps aux | grep claude-agent-sdk

# Check memory usage increases with todos
# But no disk I/O for todo storage
```

### Test 3: Code Inspection
```bash
# Search for todo persistence
grep -r "writeTodos\|saveTodos\|persistTodos" cli.js
# Result: No matches (not persisted)
```

---

## 📝 Summary

### Storage Mechanism

```
TodoWrite tool called
    ↓
B.getAppState()
    ↓
Reads from: AppState.todos[agentId]
    ↓
B.setAppState()
    ↓
Writes to: AppState.todos[agentId]
    ↓
Stored in: JavaScript heap memory (CLI process)
    ↓
Lifetime: Until CLI process exits
    ↓
Persistence: NONE
```

### Data Structure

```javascript
// In-memory only
AppState = {
  todos: {
    [agentId]: [
      {
        content: string,
        status: "pending" | "in_progress" | "completed",
        activeForm: string
      }
    ]
  }
}
```

### Key Characteristics

- ⚡ **Fast**: In-memory operations
- 🔒 **Isolated**: Per-agent separation
- 🧹 **Auto-cleanup**: Cleared on completion
- ❌ **Ephemeral**: Not persisted
- 🎯 **Simple**: No database complexity

---

## 🔮 Future Possibilities

If Anthropic wanted to add todo persistence, they could:

1. **Add to session state**:
   ```javascript
   // Write todos to .claude-sessions/{uuid}/todos.json
   await writeJSON(sessionPath + '/todos.json', todos)
   ```

2. **Include in transcripts**:
   ```javascript
   // Add todos to transcript.jsonl
   { type: 'system', subtype: 'todos', data: todos }
   ```

3. **Use SQLite**:
   ```javascript
   // Persistent database for todos
   db.run('INSERT INTO todos ...')
   ```

But currently, **none of this exists**. Todos are purely in-memory.
