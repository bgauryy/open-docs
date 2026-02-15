# Claude Agent SDK - Internal Constants & Implementations

**SDK Version**: 0.1.22
**Source**: `@anthropic-ai/claude-code/cli.js` (bundled JavaScript)

---

## Table of Contents

1. [Tool Limits & Constants](#tool-limits--constants)
2. [Agent System Constants](#agent-system-constants)
3. [Timeout & Execution Limits](#timeout--execution-limits)
4. [Extended Thinking (Ultrathink)](#extended-thinking-ultrathink)
5. [Storage & Cache Limits](#storage--cache-limits)
6. [UI & Display Constants](#ui--display-constants)
7. [Error & Retry Configuration](#error--retry-configuration)
8. [Bundle & Package Constants](#bundle--package-constants)
9. [Performance Characteristics](#performance-characteristics)
10. [Implementation Patterns](#implementation-patterns)
11. [Key Observations](#key-observations)

---

## Tool Limits & Constants

### Read Tool Limits

```javascript
// From cli.js
const READ_DEFAULT_LINES = 2000;      // XS1 in v2.1.42
const READ_CHAR_TRUNCATE = 2000;      // yN5 in v2.1.42
const PDF_MAX_SIZE = 20971520;        // tj1 in v2.1.42 (20MB)
```

**Quick Reference (v2.1.42)**:
- READ_DEFAULT_LINES: Search for `= 2000;` with "lines" in context
  - Variable name: `XS1`
- READ_CHAR_TRUNCATE: Search for `= 2000;` with "char" or "truncate" in context
  - Variable name: `yN5`
- PDF_MAX_SIZE: Search for `20971520` or `var tj1 =`
  - Variable name: `tj1`
  - Value: 20971520 (20MB, not 32MB as previously documented)
  - Where to look: `@anthropic-ai/claude-code/cli.js` (search: `var tj1 = 20971520`)

**Behavior**:
- Default: Returns first 2000 lines
- Line truncation: Each line truncated at 2000 characters
- PDF limit: Maximum **20MB** file size (corrected from 32MB)
- Truncation is silent (no warning or ellipsis)

**Use Cases Affected**:
- Reading large files (must use offset/limit pagination)
- Minified code (lines > 2000 chars truncated)
- Large PDFs > 20MB rejected

### Bash Tool Limits

```javascript
const BASH_DEFAULT_TIMEOUT = 120000;   // wJY in v2.1.42 - 2 minutes
const BASH_MAX_TIMEOUT = 600000;       // TP in v2.1.42 - 10 minutes
const BASH_OUTPUT_TRUNCATE = 30000;    // EMY/lNY/etc. in v2.1.42 - characters
```

**Quick Reference (v2.1.42)**:
- BASH_DEFAULT_TIMEOUT: Search for `= 120000;`
  - Variable name: `wJY`
- BASH_MAX_TIMEOUT: Search for `= 600000;`
  - Variable name: `TP`
- BASH_OUTPUT_TRUNCATE: Search for `= 30000;` (multiple matches, check context)
  - Possible variable names: `EMY`, `lNY`, `yTY`, etc.

**Behavior**:
- Default timeout: 2 minutes (120,000ms)
- Maximum timeout: 10 minutes (600,000ms)
- Output truncation: Defaults to **30,000 characters**, configurable via `BASH_MAX_OUTPUT_LENGTH` (clamped to a max of **150,000**)
- When output is truncated, an explicit marker is appended (e.g. `... [N lines truncated] ...`)

**Configurability (env vars)**
- `BASH_DEFAULT_TIMEOUT_MS`: sets the default timeout (must be a positive integer; defaults to `120000`)
- `BASH_MAX_TIMEOUT_MS`: sets the maximum allowed timeout; if set, it is forced to be at least the default timeout; if not set, defaults to `max(600000, BASH_DEFAULT_TIMEOUT_MS)`
- `BASH_MAX_OUTPUT_LENGTH`: sets the truncation threshold; effective range is `30000..150000`

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

**Evidence / anchors**
- Timeout defaults and env overrides: `@anthropic-ai/claude-code/cli.js` (search: `BASH_DEFAULT_TIMEOUT_MS`, `BASH_MAX_TIMEOUT_MS`)
- Output truncation threshold: `@anthropic-ai/claude-code/cli.js` (search: `BASH_MAX_OUTPUT_LENGTH`)
- Truncation marker behavior: `@anthropic-ai/claude-code/cli.js` (search: `... [${z} lines truncated] ...`)

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

### Task Tool Output Truncation

Task/subagent outputs are truncated before being displayed if they exceed a configurable character budget.

```javascript
const TASK_MAX_OUTPUT_LENGTH_DEFAULT = 32000;
const TASK_MAX_OUTPUT_LENGTH_MAX = 160000;
// Env override: TASK_MAX_OUTPUT_LENGTH
```

**Behavior**:
- If output exceeds the limit, the displayed output includes a prefix like `[Truncated. Full output: <path>]` and then the tail of the output.
- This truncation is separate from model output token limits; it is a local display/storage safeguard.

**Evidence / anchors**
- `@anthropic-ai/claude-code/cli.js` (search: `TASK_MAX_OUTPUT_LENGTH`, `[Truncated. Full output:`)

### MCP Output Truncation

MCP tool outputs are truncated based on a token budget.

```javascript
const MAX_MCP_OUTPUT_TOKENS_DEFAULT = 25000;
// Env override: MAX_MCP_OUTPUT_TOKENS
```

**Behavior**:
- When output exceeds the limit, a truncation notice is appended (e.g. `[OUTPUT TRUNCATED - exceeded <N> token limit]`).
- Token accounting approximates characters as `tokens * 4` for some internal thresholds.

**Evidence / anchors**
- `@anthropic-ai/claude-code/cli.js` (search: `MAX_MCP_OUTPUT_TOKENS`, `[OUTPUT TRUNCATED`)

---

## Agent System Constants

### Agent Colors

```javascript
// From CLI bundle
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
const HOOK_DEFAULT_TIMEOUT = 15000;  // 15 seconds (corrected from 5 seconds)
```

**Source (v2.1.42):** `@anthropic-ai/claude-code/cli.js` (search: `asyncTimeout || 15000`)

**Behavior**:
- Default **15 second** timeout for hooks (corrected from 5 seconds)
- Configurable via `asyncTimeout` in hook configuration
- Hook killed after timeout expires
- No built-in retry mechanism

### MCP Timeouts

```javascript
const MCP_TIMEOUT_DEFAULT = 30000;        // Env override: MCP_TIMEOUT
const MCP_TOOL_TIMEOUT_DEFAULT = 100000000; // Env override: MCP_TOOL_TIMEOUT (effectively infinite)
```

**Behavior**:
- `MCP_TIMEOUT` is used as a default timeout value for MCP connection/operations in parts of the MCP runtime.
- `MCP_TOOL_TIMEOUT` is used as the default timeout for `mcp-cli` tool calls/reads when no explicit timeout is provided.

**Evidence / anchors**
- `@anthropic-ai/claude-code/cli.js` (search: `MCP_TIMEOUT`)
- `@anthropic-ai/claude-code/cli.js` (search: `MCP_TOOL_TIMEOUT`, `TIMEOUT_100000000`)

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
// Variable names in v2.1.42
const THINKING_TOKEN_LIMITS = {
  ULTRATHINK: 31999,  // SBq in v2.1.42
  NONE: 0
};

const ULTRATHINK_PATTERN = /\bultrathink\b/gi;

const ULTRATHINK_TRIGGERS = [
  "ultrathink",
  "think ultra hard",
  "think ultrahard"
];
```

**Quick Reference (v2.1.42)**:
- ULTRATHINK_MAX: Search for `= 31999;`
  - Variable name: `SBq`
- ULTRATHINK_PATTERN: Search for `/\\bultrathink\\b/gi` or string "ultrathink"
- Trigger keywords: Search for "think ultra hard" or "think ultrahard"

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

**Deprecated in v2.1.42**:
> **Important**: Ultrathink no longer has any effect. The thinking budget is now set to maximum by default for all requests.
>
> The CLI will show a deprecation warning if ultrathink keywords are used:
> "Ultrathink no longer does anything. Thinking budget is now max by default."

**Current Behavior** (v2.1.42):
- All requests automatically use maximum thinking budget (31,999 tokens)
- Ultrathink keywords are detected but ignored
- A deprecation warning is displayed
- No user action needed - extended thinking is always enabled

**Overrides / switches**
- `MAX_THINKING_TOKENS`: if set, overrides the thinking token budget used by the runtime.
- `alwaysThinkingEnabled` setting: when explicitly set to `false`, disables “always thinking” behavior in the runtime’s thinking enablement checks.

**Evidence / anchors**
- Ultrathink deprecation notification: `@anthropic-ai/claude-code/cli.js` (search: `Ultrathink no longer does anything`)
- Thinking budget override + always-thinking check: `@anthropic-ai/claude-code/cli.js` (search: `MAX_THINKING_TOKENS`, `alwaysThinkingEnabled`)

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

### File Checkpointing (Rewind)

File checkpointing is the mechanism behind “rewind code” features (saving file snapshots that can be restored later).

```javascript
// Env switch (global):
//   CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING=1  -> disables checkpointing
//
// Runtime setting key:
//   fileCheckpointingEnabled (global setting)
```

**Behavior**:
- When disabled, Claude Code does not record file checkpoints for rewind/restore workflows.
- The feature is exposed as a global setting (`fileCheckpointingEnabled`) and can also be force-disabled via the environment variable.

**Evidence / anchors**
- Settings catalog: `@anthropic-ai/claude-code/cli.js` (search: `fileCheckpointingEnabled`)
- Env gating in UI/runtime: `@anthropic-ai/claude-code/cli.js` (search: `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING`)

### Cache Configuration

```javascript
const WEBFETCH_CACHE_TTL_MS = 900000;        // 15 minutes
const WEBFETCH_CACHE_MAX_SIZE_BYTES = 52428800; // 50MB total
```

**Behavior**:
- WebFetch caches fetched URL results in an in-memory cache keyed by the input URL.
- Cache entries expire after 15 minutes, and the cache is size-bounded (50MB total, measured by `Buffer.byteLength(content)`).
- `http://` URLs are automatically rewritten to `https://` before fetching.
- `text/html` responses are converted to Markdown before being cached/returned.
- By default, WebFetch runs a domain preflight allow/block check; this can be skipped via `skipWebFetchPreflight` (managed setting).

**Evidence / anchors**
- WebFetch cache TTL + max size + URL normalization + HTML conversion: `@anthropic-ai/claude-code/cli.js` (search: `ttl: TIMEOUT_900000`, `maxSize: TIMEOUT_52428800`, `protocol === \"http:\"`, `turndown`)
- Preflight skip setting: `@anthropic-ai/claude-code/cli.js` (search: `skipWebFetchPreflight`)

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

### HTTP Retry Policy (Network Requests)

```javascript
const HTTP_MAX_RETRIES_DEFAULT = 3;
const HTTP_RETRY_DELAY_BASE_MS = 1000;
const HTTP_MAX_RETRY_DELAY_MS = 64000;
const RETRY_AFTER_HEADERS = ["retry-after-ms", "x-ms-retry-after-ms", "Retry-After"];
```

**Behavior**:
- Default max retries is 3.
- For throttling responses (notably HTTP 429 and 503), the client respects common retry-after headers.
- For other retryable conditions (timeouts, connection errors, certain HTTP statuses like 408 and 5xx), an exponential backoff is applied with a base delay and a capped maximum delay.

**Evidence / anchors**
- Retry defaults and strategies: `@anthropic-ai/claude-code/cli.js` (search: `var yh1 = 3;`, `var dB5 = 1000;`, `var cB5 = 64000;`, `pB5 = [\"retry-after-ms\"`)
- Retry policy orchestration: `@anthropic-ai/claude-code/cli.js` (search: `Retry ${_}: Attempting to send request`)

### Rate Limit Handling (HTTP 429)

When the API returns HTTP 429, Claude Code constructs a user-facing message and may include additional guidance when unified rate limit headers are present.

**Notable headers**
- `anthropic-ratelimit-unified-reset`
- `anthropic-ratelimit-unified-representative-claim`
- `anthropic-ratelimit-unified-overage-status`
- `anthropic-ratelimit-unified-overage-reset`
- `anthropic-ratelimit-unified-overage-disabled-reason`

**Evidence / anchors**
- `@anthropic-ai/claude-code/cli.js` (search: `status === 429`, `anthropic-ratelimit-unified-`)

### Model Output Token Maximum

Claude Code caps assistant output tokens and provides an environment variable override:

```javascript
// Env override: CLAUDE_CODE_MAX_OUTPUT_TOKENS
```

**Behavior**:
- If a response exceeds the configured maximum, Claude Code emits a message instructing you to set `CLAUDE_CODE_MAX_OUTPUT_TOKENS` to configure the behavior.
- The effective default and upper bound depend on the active model/provider and are clamped internally.

**Evidence / anchors**
- `@anthropic-ai/claude-code/cli.js` (search: `CLAUDE_CODE_MAX_OUTPUT_TOKENS`, `response exceeded`)

### Telemetry and “Nonessential Traffic” Switches

```javascript
const OTEL_METRICS_EXPORT_INTERVAL_MS = 300000; // 5 minutes
const OTEL_BATCH_DELAY_MS = 5000;
// Primary enable: CLAUDE_CODE_ENABLE_TELEMETRY
```

**Behavior**:
- OpenTelemetry exporters are enabled when `CLAUDE_CODE_ENABLE_TELEMETRY` is set (and other conditions permit).
- In some environments/providers, telemetry may be disabled regardless of the enable flag (e.g. when using certain non-first-party providers or when `DISABLE_TELEMETRY` is set).
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` disables various background/optional network behaviors (e.g. auto-updates, product feedback).

**Evidence / anchors**
- Telemetry enable flag + export interval: `@anthropic-ai/claude-code/cli.js` (search: `CLAUDE_CODE_ENABLE_TELEMETRY`, `exportIntervalMillis: 300000`)
- “Nonessential traffic” gating: `@anthropic-ai/claude-code/cli.js` (search: `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`)
- Telemetry disable conditions: `@anthropic-ai/claude-code/cli.js` (search: `DISABLE_TELEMETRY`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`)

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

