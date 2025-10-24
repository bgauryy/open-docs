# Claude Agent SDK - Internal Constants & Implementations

**SDK Version**: 0.1.22
**Source**: `cli.js` (9.7MB bundled JavaScript)

---

## Table of Contents

1. [Tool Limits & Constants](#tool-limits--constants)
2. [Agent System Constants](#agent-system-constants)
3. [Timeout & Execution Limits](#timeout--execution-limits)
4. [Extended Thinking (Ultrathink)](#extended-thinking-ultrathink)
5. [Storage & Cache Limits](#storage--cache-limits)
6. [UI & Display Constants](#ui--display-constants)
7. [Error & Retry Configuration](#error--retry-configuration)

---

## Tool Limits & Constants

### Read Tool Limits

```javascript
// From cli.js
const READ_DEFAULT_LINES = 2000;      // BG1 in cli.js
const READ_CHAR_TRUNCATE = 2000;      // eE9 in cli.js - per line
const PDF_MAX_SIZE = 33554432;        // UOA in cli.js (32MB)
```

**Behavior**:
- Default: Returns first 2000 lines
- Line truncation: Each line truncated at 2000 characters
- PDF limit: Maximum 32MB file size
- Truncation is silent (no warning or ellipsis)

**Use Cases Affected**:
- Reading large files (must use offset/limit pagination)
- Minified code (lines > 2000 chars truncated)
- Large PDFs rejected

### Bash Tool Limits

```javascript
const BASH_DEFAULT_TIMEOUT = 120000;   // 2 minutes
const BASH_MAX_TIMEOUT = 600000;       // 10 minutes
const BASH_OUTPUT_TRUNCATE = 30000;    // characters
```

**Behavior**:
- Default timeout: 2 minutes (120,000ms)
- Maximum timeout: 10 minutes (600,000ms)
- Output truncation: 30,000 combined characters (stdout + stderr)
- Truncation is silent (no indicator)

**Workarounds**:
```typescript
// Long-running command
Bash({ 
  command: "npm install", 
  timeout: 600000  // Explicit 10 min
})

// Or use background mode
Bash({ 
  command: "npm test", 
  run_in_background: true 
})

// Or redirect output to file
Bash({ 
  command: "npm test > test-results.txt 2>&1" 
})
// Then: Read({ file_path: "test-results.txt" })
```

### Grep Tool Defaults

```javascript
const GREP_DEFAULT_OUTPUT_MODE = "files_with_matches";
```

**Behavior**:
- Default: Returns only filenames, not content
- Must explicitly set `output_mode: "content"` to see matching lines
- Common source of confusion for new users

**Example**:
```typescript
// Only returns filenames (default)
Grep({ pattern: "function.*authenticate" })

// Returns actual content
Grep({ 
  pattern: "function.*authenticate",
  output_mode: "content"
})
```

### TodoWrite Constraints

```javascript
const TODO_IN_PROGRESS_LIMIT = 1;  // Exactly one task
```

**Behavior**:
- System enforces exactly ONE task with status "in_progress"
- Tool call fails if constraint violated
- Must complete current task before starting next
- No parallel task tracking allowed

---

## Agent System Constants

### Agent Colors

```javascript
// From cli.js line 2614
const AGENT_COLORS = [
  "red",
  "blue", 
  "green",
  "yellow",
  "purple",
  "orange",
  "pink",
  "cyan"
];

const AGENT_COLOR_SUFFIX = "_FOR_SUBAGENTS_ONLY";

// Actual color values used
const AGENT_COLOR_MAP = {
  red: "red_FOR_SUBAGENTS_ONLY",
  blue: "blue_FOR_SUBAGENTS_ONLY",
  green: "green_FOR_SUBAGENTS_ONLY",
  yellow: "yellow_FOR_SUBAGENTS_ONLY",
  purple: "purple_FOR_SUBAGENTS_ONLY",
  orange: "orange_FOR_SUBAGENTS_ONLY",
  pink: "pink_FOR_SUBAGENTS_ONLY",
  cyan: "cyan_FOR_SUBAGENTS_ONLY"
};
```

**Behavior**:
- 8 total colors for subagent visual identification
- Color assignment is deterministic (hash-based on agent type)
- Colors suffixed with "_FOR_SUBAGENTS_ONLY" internally
- Used for UI rendering and agent differentiation
- general-purpose agent doesn't get color assignment

### Built-in Agent Tool Access

```javascript
const AGENT_TOOLS = {
  "Explore": ["Glob", "Grep", "Read", "Bash"],
  "general-purpose": ["*"],  // ALL tools
  "statusline-setup": ["Read", "Edit"],
  "output-style-setup": ["Read", "Write", "Edit", "Glob", "Grep"],
  "security-review": [
    "Bash(git diff:*)", 
    "Bash(git status:*)", 
    "Bash(git log:*)", 
    "Bash(git show:*)", 
    "Bash(git remote show:*)", 
    "Read", 
    "Glob", 
    "Grep", 
    "LS", 
    "Task"
  ]
};
```

**Notes**:
- `*` means all tools available
- Security review agent has restricted bash (git commands only)
- Tool restrictions enforced at invocation time

### Agent Context Defaults

```javascript
const AGENT_CONTEXT_DEFAULTS = {
  forkContext: false,  // Default: isolated context
  isAsync: false,      // Default: synchronous execution
  model: "inherit"     // Default: inherit parent model
};
```

---

## Timeout & Execution Limits

### Hook Timeout

```javascript
const HOOK_DEFAULT_TIMEOUT = 5000;  // 5 seconds
```

**Behavior**:
- Default 5 second timeout for hooks
- Configurable via `asyncTimeout` in AsyncHookJSONOutput
- Hook killed after timeout expires
- No built-in retry mechanism

### Conversation Limits

```javascript
const DEFAULT_MAX_TURNS = Infinity;  // No limit by default
const HISTORY_RETENTION_LIMIT = 100;  // Prompt history stored
```

**Behavior**:
- No default turn limit (can be set via `maxTurns` option)
- Prompt history stores last 100 commands
- Auto-compaction triggers based on token count (not fixed number)

---

## Extended Thinking (Ultrathink)

### Ultrathink Constants

```javascript
// From cli.js line 1497
const THINKING_TOKEN_LIMITS = {
  ULTRATHINK: 31999,
  NONE: 0
};

const ULTRATHINK_PATTERN = /\bultrathink\b/gi;

const ULTRATHINK_TRIGGERS = [
  "ultrathink",
  "think ultra hard",
  "think ultrahard"
];
```

**Detection Function**:
```javascript
function isUltrathinkTrigger(input) {
  let normalized = input.toLowerCase();
  return (
    normalized === "ultrathink" ||
    normalized === "think ultra hard" ||
    normalized === "think ultrahard"
  );
}
```

**Behavior**:
- Maximum 31,999 thinking tokens for extended reasoning
- Triggered by specific keywords in prompt
- Case-insensitive pattern matching
- Applied for single turn only (not persistent)

**Usage**:
```
ultrathink: analyze the security implications of this code
think ultra hard: find the optimal algorithm
```

---

## Storage & Cache Limits

### File System Limits

```javascript
const MAX_CACHED_FILES = 1000;  // Per session (observed)
const PASTED_CONTENT_LIMIT = 1024;  // Chars for history
const SESSION_HISTORY_LIMIT = 100;  // Commands stored
```

**Session Storage Paths**:
```
~/.claude/sessions/<session-id>/
├── transcript.json           # Full conversation
├── checkpoints/              # Auto-checkpoint every 5 messages
│   ├── checkpoint-0.json
│   ├── checkpoint-5.json
│   ├── checkpoint-10.json
│   └── ...
└── file-snapshots/           # File content snapshots
    ├── <hash-1>.txt
    ├── <hash-2>.txt
    └── ...
```

### Cache Configuration

```javascript
const WEBFETCH_CACHE_DURATION = 900000;  // 15 minutes (900,000ms)
```

**Behavior**:
- WebFetch uses 15-minute self-cleaning cache
- Cache key: URL
- Automatic cleanup after expiration
- No manual cache invalidation API

---

## UI & Display Constants

### Prompt History Storage

```javascript
const HISTORY_FILE = "history.jsonl";  // In ~/.claude/
const HISTORY_MAX_ENTRIES = 100;
const HISTORY_TRUNCATION_LENGTH = 1024;  // Pasted content
```

**Format**:
```jsonl
{"display":"npm install","pastedContents":{},"timestamp":1729775232000}
{"display":"Read src/app.ts","pastedContents":{},"timestamp":1729775245000}
```

### Terminal Detection

```javascript
const SUPPORTED_TERMINALS = [
  "vscode",
  "cursor", 
  "windsurf",
  "ghostty",
  "wezterm"
];

const UNSUPPORTED_MULTIPLEXERS = [
  "tmux",
  "screen"
];
```

**Behavior**:
- Terminal detection for feature availability
- Some features disabled in tmux/screen
- Status line setup varies by terminal

---

## Error & Retry Configuration

### Lock File Retry

```javascript
const LOCK_RETRY_CONFIG = {
  retries: 3,
  minTimeout: 50,
  stale: 10000  // 10 seconds
};
```

**Usage**:
- File locking for concurrent access protection
- History file writes use file locks
- Retry 3 times with exponential backoff
- Lock considered stale after 10 seconds

### Model Deprecation Warnings

```javascript
const DEPRECATED_MODELS = {
  "claude-1.3": "November 6th, 2024",
  "claude-1.3-100k": "November 6th, 2024",
  "claude-instant-1.1": "November 6th, 2024",
  "claude-instant-1.1-100k": "November 6th, 2024",
  "claude-instant-1.2": "November 6th, 2024",
  "claude-3-sonnet-20240229": "July 21st, 2025",
  "claude-3-opus-20240229": "January 5th, 2026",
  "claude-2.1": "July 21st, 2025",
  "claude-2.0": "July 21st, 2025",
  "claude-3-5-sonnet-20241022": "October 22, 2025",
  "claude-3-5-sonnet-20240620": "October 22, 2025"
};
```

**Behavior**:
- Warning issued when using deprecated model
- Shows end-of-life date
- Recommendation to migrate to newer model

### Extended Thinking Token Limits by Model

```javascript
const MODEL_THINKING_LIMITS = {
  "claude-opus-4-20250514": 8192,
  "claude-opus-4-0": 8192,
  "claude-4-opus-20250514": 8192,
  "anthropic.claude-opus-4-20250514-v1:0": 8192,
  "claude-opus-4@20250514": 8192,
  "claude-opus-4-1-20250805": 8192,
  "anthropic.claude-opus-4-1-20250805-v1:0": 8192,
  "claude-opus-4-1@20250805": 8192
};
```

---

## Bundle & Package Constants

### Bundle Characteristics

```javascript
const CLI_BUNDLE_SIZE = 9734140;  // 9.7MB
const SDK_MODULE_SIZE = 534036;    // 521.5KB
const YOGA_WASM_SIZE = 88658;      // 86.6KB
```

**Structure**:
- CLI: Single bundled file (no external dependencies)
- SDK: Separate module export for programmatic use
- Yoga: WebAssembly for flexbox layout (UI rendering)

### Optional Dependencies (Platform-Specific)

```javascript
const IMAGE_PROCESSING_DEPS = {
  "@img/sharp-darwin-arm64": "^0.33.5",
  "@img/sharp-darwin-x64": "^0.33.5",
  "@img/sharp-linux-arm": "^0.33.5",
  "@img/sharp-linux-arm64": "^0.33.5",
  "@img/sharp-linux-x64": "^0.33.5",
  "@img/sharp-win32-x64": "^0.33.5"
};
```

**Behavior**:
- Image processing requires platform-specific Sharp binaries
- Optional dependencies (silently degrades if missing)
- Cross-platform builds require correct platform selection

---

## Performance Characteristics

### Typical Execution Times

| Tool | Typical Time | Notes |
|------|-------------|-------|
| Read | 1-50ms | Cached after first read |
| Write | 5-20ms | Atomic operation |
| Edit | 10-30ms | Read + validate + write |
| Glob | 50-500ms | Depends on pattern |
| Grep | 100-2000ms | Very fast (ripgrep) |
| Bash | Variable | Command-dependent |
| Task (sync) | 5-60s | Agent complexity |
| Task (async) | ~100ms | Returns immediately |

### Memory Usage

| Component | Memory | Notes |
|-----------|--------|-------|
| Base process | ~100-150MB | CLI startup |
| Per MCP server (stdio) | ~20-50MB | Process overhead |
| Per MCP server (HTTP/SSE) | ~10-30MB | Connection overhead |
| Session state | ~10-30MB | Message history |
| File cache | ~100MB | Max ~1000 files |
| **Typical total** | **200-500MB** | Normal usage |
| **Heavy usage** | **500MB-1GB** | Many MCP servers |

---

## Implementation Patterns

### Read-Before-Write Enforcement

```javascript
// Internal tracking (simplified)
const sessionReadFiles = new Set();

function enforceReadBeforeWrite(filePath) {
  if (!sessionReadFiles.has(filePath)) {
    throw new Error("File has not been read yet");
  }
}

// On Read tool
function handleReadTool(input) {
  // ... read file
  sessionReadFiles.add(input.file_path);
  return content;
}

// On Write/Edit tool
function handleWriteTool(input) {
  enforceReadBeforeWrite(input.file_path);
  // ... write file
}
```

**Characteristics**:
- Session-scoped (cleared on new session)
- Set-based tracking (fast O(1) lookup)
- Cannot be bypassed
- Resets on session resume

### Edit Uniqueness Check

```javascript
// Internal validation (simplified)
function validateEditUniqueness(fileContent, oldString, replaceAll) {
  const occurrences = countOccurrences(fileContent, oldString);
  
  if (occurrences === 0) {
    throw new Error("old_string not found in file");
  }
  
  if (occurrences > 1 && !replaceAll) {
    throw new Error(
      "old_string appears multiple times. Use replace_all=true " +
      "or make old_string more unique."
    );
  }
  
  return true;
}
```

**Characteristics**:
- Exact string matching (no fuzzy match)
- Whitespace-sensitive
- Count-based validation
- Fails before any modification

---

## Key Observations

### Design Decisions
1. **Conservative Limits**: Default limits favor safety over flexibility
2. **Silent Truncation**: Output truncation happens without warning
3. **Strict Validation**: Read-before-write and uniqueness strictly enforced
4. **Token Efficiency**: Agent color system minimizes token usage
5. **Session Isolation**: State resets prevent cross-session pollution

### Performance Trade-offs
1. **Large Bundle**: Fast startup but slow installation
2. **File Caching**: Fast repeated access but memory usage
3. **Agent Colors**: Visual clarity with 8 colors (sufficient for most uses)
4. **Hook Timeout**: Balance between responsiveness and flexibility

### Security Considerations
1. **Tool Limits**: Prevent resource exhaustion
2. **Timeout Enforcement**: Prevent runaway processes
3. **Read-Before-Write**: Prevent accidental overwrites
4. **Permission Validation**: Multi-level security checks

---

