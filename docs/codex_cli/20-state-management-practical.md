# State Management Internals

This guide explains how Codex manages conversation state, sessions, history, and persistence across the TUI, CLI, and exec modes.

## Overview

Codex maintains state across multiple layers:

- **Session State**: Active conversation in `~/.codex/sessions/`
- **History**: Message log in `~/.codex/history.jsonl`
- **Conversation Manager**: In-memory state tracking (`core/src/conversation_manager.rs`)
- **Turn State**: Individual request/response cycles (`core/src/state/turn.rs`)
- **Transcript**: Message sequence for model context

---

## Session Storage

### Session Directory Structure

```
~/.codex/sessions/
├── <uuid-1>/
│   ├── metadata.json       # Session info (created, model, cwd)
│   ├── transcript.json     # Full conversation history
│   └── state.json          # Current state snapshot
├── <uuid-2>/
│   └── ...
└── active -> <uuid-current>  # Symlink to active session
```

### Session Metadata

**File:** `~/.codex/sessions/<uuid>/metadata.json`

```json
{
  "id": "01933e84-1234-7890-abcd-ef0123456789",
  "created_at": "2025-10-25T10:30:00Z",
  "last_updated": "2025-10-25T11:45:00Z",
  "model": "gpt-5-codex",
  "cwd": "/Users/you/project",
  "approval_policy": "on-request",
  "sandbox_mode": "workspace-write"
}
```

### Transcript Format

**File:** `~/.codex/sessions/<uuid>/transcript.json`

```json
{
  "messages": [
    {
      "role": "user",
      "content": "Refactor the auth module"
    },
    {
      "role": "assistant",
      "content": "I'll help refactor the auth module...",
      "tool_calls": [
        {
          "id": "call_abc123",
          "type": "function",
          "function": {
            "name": "read_file",
            "arguments": "{\"path\": \"src/auth.rs\"}"
          }
        }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_abc123",
      "content": "// file contents..."
    }
  ]
}
```

### State Snapshot

**File:** `~/.codex/sessions/<uuid>/state.json`

Stores current execution state:
- Pending tool approvals
- MCP server connections
- Compact status (has conversation been compressed?)
- Last turn ID

---

## History Persistence

### history.jsonl Format

**File:** `~/.codex/history.jsonl`

One JSON object per line (JSONL):

```jsonl
{"timestamp":"2025-10-25T10:30:00Z","session_id":"01933e84...","role":"user","content":"Fix the build"}
{"timestamp":"2025-10-25T10:30:15Z","session_id":"01933e84...","role":"assistant","content":"I'll check the build errors..."}
{"timestamp":"2025-10-25T10:30:45Z","session_id":"01933e84...","role":"tool","tool":"local_shell","output":"npm test passed"}
```

### History Configuration

**In `config.toml`:**

```toml
[history]
persistence = "save-all"  # or "none" to disable
```

Permissions: `0600` (user read/write only) for security

---

## Conversation Manager

**Location:** `core/src/conversation_manager.rs`

The conversation manager is the in-memory state keeper for active sessions.

### Responsibilities

1. **Message Sequencing**: Maintain correct order of user/assistant/tool messages
2. **Context Window**: Track token usage and trigger compaction
3. **Turn Management**: Coordinate request → response → tool → result cycles
4. **Fork Support**: Enable conversation branching (backtracking)
5. **Compaction**: Trigger transcript compression when approaching context limit

### Key Methods

```rust
impl ConversationManager {
    // Add a new message to the transcript
    pub fn add_message(&mut self, message: Message);

    // Get messages for next API request
    pub fn get_messages_for_request(&self) -> Vec<Message>;

    // Fork from a specific turn
    pub fn fork_from_turn(&mut self, turn_id: u64) -> Result<()>;

    // Compress older messages
    pub fn compact(&mut self) -> Result<()>;
}
```

---

## Turn State

**Location:** `core/src/state/turn.rs`

A "turn" represents one complete request/response cycle:

```
User Message
    ↓
[Turn Start]
    ↓
API Request
    ↓
API Response (with tool_calls)
    ↓
Tool Execution
    ↓
Tool Results
    ↓
Next API Request (with tool results)
    ↓
Final Assistant Message
    ↓
[Turn End]
```

### Turn Metadata

```rust
pub struct Turn {
    pub id: u64,
    pub started_at: SystemTime,
    pub completed_at: Option<SystemTime>,
    pub input_tokens: u64,
    pub output_tokens: u64,
    pub cached_tokens: Option<u64>,
    pub tool_calls: Vec<ToolCall>,
    pub status: TurnStatus,  // InProgress, Completed, Failed
}
```

---

## Resume Flow

### TUI Resume

**Command:** `codex resume` or `codex resume --last`

**Implementation:** `tui/src/resume_picker.rs`

1. **List sessions** from `~/.codex/sessions/`
2. **Show picker UI** with session metadata (last message, timestamp)
3. **User selects** session
4. **Load transcript** from `transcript.json`
5. **Restore state** (model, cwd, approval policy)
6. **Continue conversation** from last message

### Exec Resume

**Command:** `codex exec resume <session_id>` or `codex exec resume --last`

**Implementation:** `exec/tests/suite/resume.rs`

1. **Locate session** in `~/.codex/sessions/`
2. **Load transcript** silently
3. **Apply new message** from command line
4. **Stream response** with same config as original session

---

## Fork (Backtrack) Flow

**TUI Feature:** Press `Esc` twice to edit a previous message

**Implementation:** `tui/src/app_backtrack.rs`

### Steps

1. **User enters backtrack mode** (Esc in empty composer)
2. **Navigate to previous user message** (Esc repeatedly)
3. **Select message** to edit (Enter)
4. **Transcript forked** from that point:
   - Messages after selected point are **discarded**
   - Conversation ID **remains same** (not a new session)
   - User message **pre-filled** in composer for editing
5. **User edits and resubmits**
6. **New branch** continues from fork point

### Fork Visualization

```
Original:
  User: "Add tests"
    → Assistant: "I'll add unit tests..."
      → User: "Actually, add integration tests"

Fork from second user message:
  User: "Add tests"
    → Assistant: "I'll add unit tests..."
    [FORK HERE]
      → User: "Add both unit and integration tests"  (edited)
        → Assistant: "I'll add both types..." (new response)
```

---

## Compaction

**Location:** `core/src/codex/compact.rs`

When the conversation approaches the model's context window limit, Codex automatically compresses older messages.

### Compaction Strategy

1. **Detect limit** (e.g., 90% of `model_context_window`)
2. **Select old messages** to compress (typically first 50% of transcript)
3. **Send compression request** to model:
   ```
   "Summarize the following conversation in 2-3 sentences..."
   [old messages]
   ```
4. **Replace old messages** with summary
5. **Continue conversation** with reduced context

### Configuration

```toml
# In config.toml
auto_compact_token_limit = 100000  # Trigger at 100K tokens
```

### Compaction Tests

See `core/tests/suite/compact.rs` for behavior:
- Preserves recent messages (last 10-20 turns)
- Maintains system prompt
- Keeps important context (AGENTS.md, etc.)
- Can be resumed after compaction

---

## Archive Flow

**Purpose:** Export and delete old conversations

**Implementation:** `app-server/tests/suite/archive_conversation.rs`

### Steps

1. **Select session** to archive
2. **Export transcript** to external format (JSON, Markdown)
3. **Remove from sessions directory**
4. **Update index** (remove from resume picker)

### Export Formats

**JSON:**
```json
{
  "id": "01933e84...",
  "created_at": "2025-10-25T10:30:00Z",
  "model": "gpt-5-codex",
  "messages": [...]
}
```

**Markdown:**
```markdown
# Codex Conversation - Oct 25, 2025

## User
Fix the build errors

## Assistant
I'll check the build errors...

[...]
```

---

## State Synchronization

### TUI ↔ Core

The TUI maintains UI state separately from the core conversation state:

- **Core state** (`core/src/conversation_manager.rs`): Transcript, tokens, messages
- **UI state** (`tui/src/chatwidget.rs`): Scroll position, selected message, backtrack mode

Synchronization happens via:
- **Events**: Core sends updates to TUI (new message, tool result)
- **Commands**: TUI sends actions to core (user message, approval decision)

### Exec ↔ Core

Exec mode uses the same core state but without UI:

- **No UI state**: Runs headless
- **Event streaming**: `exec/src/exec_events.rs` emits JSONL events
- **No interruption**: Runs to completion (unless timeout)

---

## Concurrency and Locking

### Thread Safety

Codex uses Rust's ownership system to ensure thread safety:

- **Tokio runtime**: Async tasks for API requests, tool execution
- **No shared mutable state**: Each session owns its conversation manager
- **Message passing**: Channels for TUI ↔ core communication

### File Locking

Session files use OS-level locking to prevent concurrent writes:

```rust
// Open with exclusive lock
let file = OpenOptions::new()
    .write(true)
    .create(true)
    .open("transcript.json")?;
file.lock_exclusive()?;
```

---

## State Recovery

### Crash Recovery

If Codex crashes mid-session:

1. **Partial transcript** may be saved
2. **Resume session** to continue from last saved message
3. **Pending tool calls** are discarded (must be re-requested by model)

### Corruption Handling

If session files are corrupted:

1. **Try to load** metadata first
2. **Fallback** to transcript recovery
3. **Worst case**: Start new session (old session preserved in `~/.codex/sessions/<uuid>`)

---

## Performance Considerations

### Session Limits

**Typical limits:**
- Max sessions: Unlimited (practical limit: disk space)
- Session file size: 1-50 MB (depending on transcript length)
- Resume picker: Shows last 100 sessions (configurable)

### Token Tracking

Codex tracks token usage per turn:

```rust
pub struct TokenUsage {
    pub input_tokens: u64,
    pub output_tokens: u64,
    pub cached_tokens: Option<u64>,  // Prompt caching
}
```

Used for:
- Compaction triggers
- Rate limit warnings
- Cost estimation

---

## Debugging State Issues

### Inspect Session

```bash
# View session metadata
cat ~/.codex/sessions/<uuid>/metadata.json | jq

# View transcript
cat ~/.codex/sessions/<uuid>/transcript.json | jq '.messages[-5:]'

# View state snapshot
cat ~/.codex/sessions/<uuid>/state.json | jq
```

### Common Issues

**Session not appearing in resume picker:**
- Check `~/.codex/sessions/` directory exists
- Verify `metadata.json` is valid JSON
- Check file permissions (should be `0600`)

**Fork/backtrack not working:**
- Ensure conversation has multiple user messages
- Check UI state (must be in empty composer)
- Verify no pending tool approvals

**Compaction failures:**
- Check model supports compression prompts
- Verify token counts are accurate
- Increase `auto_compact_token_limit` if too aggressive

---

## Best Practices

### For Users

1. **Use resume** for long-running tasks across days
2. **Fork conversations** to try different approaches
3. **Archive old sessions** to keep `~/.codex/sessions/` clean
4. **Monitor token usage** via `/status` in TUI

### For Developers

1. **Always save state** after significant events
2. **Test resume/fork** in integration tests
3. **Handle corruption** gracefully (don't crash)
4. **Profile I/O** for large transcripts
5. **Use async I/O** for session file operations

---

## References

- [Config Reference](./config.md) - History persistence settings
- [Getting Started](./getting-started.md) - Resume and fork usage
- [Exec Reference](./exec.md) - Non-interactive resume
- [Tool System](./tool-system.md) - Tool result storage in transcript
