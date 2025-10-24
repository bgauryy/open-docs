# Gemini CLI - Tools Reference

**Generated:** 2024-10-24
**Purpose:** Complete catalog of all built-in tools, their parameters, behaviors, and gotchas

---

## Table of Contents

1. [Tool System Overview](#tool-system-overview)
2. [Built-in Core Tools](#built-in-core-tools)
3. [Tool Parameters & Schemas](#tool-parameters--schemas)
4. [Tool Execution Flow](#tool-execution-flow)
5. [Tool Confirmation System](#tool-confirmation-system)
6. [MCP Tools](#mcp-tools)
7. [Discovered Tools](#discovered-tools)
8. [Tool Gotchas & Limitations](#tool-gotchas--limitations)

---

## Tool System Overview

### Architecture

Gemini CLI uses a **declarative tool system** that separates parameter validation from execution:

```typescript
Interface Hierarchy:
┌─────────────────────────────────────┐
│  DeclarativeTool<TParams, TResult>  │
│  - name, displayName, description   │
│  - schema (FunctionDeclaration)     │
│  - build(params) → ToolInvocation   │
└──────────────┬──────────────────────┘
               │ extends
┌──────────────▼──────────────────────┐
│  BaseDeclarativeTool                │
│  - validateToolParams()             │
│  - createInvocation()               │
└──────────────┬──────────────────────┘
               │ extends
┌──────────────▼──────────────────────┐
│  Concrete Tool Implementation       │
│  (e.g., GlobTool, ReadFileTool)    │
└─────────────────────────────────────┘
```

### Tool Kinds

Tools are categorized by their side effects:

```typescript
enum Kind {
  Read = 'read',        // No side effects
  Edit = 'edit',        // Modifies files
  Delete = 'delete',    // Removes files
  Move = 'move',        // Moves/renames files
  Search = 'search',    // Searches content
  Execute = 'execute',  // Runs commands
  Think = 'think',      // Internal processing
  Fetch = 'fetch',      // Network requests
  Other = 'other'       // Misc operations
}

// Mutator kinds require confirmation
const MUTATOR_KINDS = [Kind.Edit, Kind.Delete, Kind.Move, Kind.Execute];
```

---

## Built-in Core Tools

### 1. `glob` - File Pattern Matching

**Tool Name:** `glob`  
**Kind:** `Search`  
**Display Name:** "Glob"  
**Purpose:** Find files matching glob patterns

#### Parameters

```typescript
interface GlobToolParams {
  pattern: string;              // Glob pattern (e.g., "**/*.ts")
  path?: string;                // Search directory (optional)
  case_sensitive?: boolean;     // Case-sensitive matching (default: false)
  respect_git_ignore?: boolean; // Respect .gitignore (default: true)
  respect_gemini_ignore?: boolean; // Respect .geminiignore (default: true)
}
```

#### Behavior

- **Sorting:** Results sorted by modification time (newest first), then alphabetically
- **Recency Threshold:** Files modified within 24 hours listed first
- **Filtering:** Respects .gitignore/.geminiignore by default
- **Workspace Boundary:** Only searches within workspace directories

#### Example Output

```
Found 15 file(s) matching "**/*.ts" within /path/to/project:
/path/to/project/src/index.ts
/path/to/project/src/tools/glob.ts
...
```

#### Gotchas

- Pattern is NOT escaped by default - use literal strings for special chars
- If path exists as literal file, pattern is auto-escaped
- Cross-workspace search supported if no path specified
- Empty results include ignored file count

---

### 2. `search_file_content` - Content Search (Grep)

**Tool Name:** `search_file_content`  
**Kind:** `Search`  
**Display Name:** "Search File Content"  
**Purpose:** Search for patterns in file contents

#### Parameters

```typescript
interface GrepToolParams {
  pattern: string;    // Regex pattern to search
  path?: string;      // Directory to search (optional)
  include?: string;   // File pattern to include (e.g., "*.js")
}
```

#### Strategy Priority

Grep tool uses multiple strategies with fallback:

1. **git grep** (preferred if in git repo)
   - Flags: `--untracked -n -E --ignore-case`
   - Respects .gitignore automatically
   
2. **system grep** (fallback)
   - Flags: `-r -n -H -E -I`
   - Excludes common directories (node_modules, .git, etc.)
   
3. **JavaScript fallback** (last resort)
   - Pure Node.js implementation
   - Uses fast-glob for file discovery

#### Example Output

```
Found 3 matches for pattern "export.*Tool" in path ".":
---
File: src/tools/glob.ts
L15: export interface GlobPath {
L67: export function sortFileEntries(
L94: export interface GlobToolParams {
---
```

#### Gotchas

- Pattern is treated as **extended regex** by default
- Case-insensitive by default
- Binary files are skipped (grep -I flag)
- Permission denied errors are silently ignored
- Falls back silently if git/grep unavailable

---

### 3. `list_directory` - Directory Listing (LS)

**Tool Name:** `list_directory`  
**Kind:** `Read`  
**Display Name:** "List Directory"  
**Purpose:** List files and directories

#### Parameters

```typescript
interface LSToolParams {
  path: string;                  // Absolute path to list
  ignore?: string[];             // Glob patterns to ignore
  file_filtering_options?: {
    respect_git_ignore?: boolean;
    respect_gemini_ignore?: boolean;
  };
}
```

#### Behavior

- **Sorting:** Directories first, then files (alphabetically)
- **Filtering:** Respects ignore patterns + .gitignore/.geminiignore
- **Format:** `[DIR] name` for directories, `name` for files
- **Stats:** Shows ignored count if any

#### Example Output

```
Directory listing for /path/to/project/src:
[DIR] commands
[DIR] config
[DIR] tools
index.ts
types.ts

(5 ignored)
```

#### Gotchas

- Path **must be absolute** (enforced)
- Returns error if path is not a directory
- Empty directories show "Directory is empty"
- Ignore patterns use glob syntax with regex conversion

---

### 4. `read_file` - Read Single File

**Tool Name:** `read_file`  
**Kind:** `Read`  
**Display Name:** "Read File"  
**Purpose:** Read file contents with optional line ranges

#### Parameters

```typescript
interface ReadFileToolParams {
  absolute_path: string;          // Absolute file path (REQUIRED)
  offset?: number;                // Starting line number (0-indexed, optional)
  limit?: number;                 // Number of lines to read (optional)
  // Note: Line numbers are always enabled (handled internally)
}
```

#### Behavior

- **Encoding Detection:** Auto-detects file encoding
- **Binary Detection:** Returns error for binary files
- **Line Numbering:** Format: `   1|content` (6-char padding)
- **Full File:** Returns entire file if no range specified
- **Range Validation:** Clamps to valid range, no error

#### Example Output

```
File: src/index.ts (150 lines)

     1|/**
     2| * @license
     3| * Copyright 2025 Google LLC
     4| */
     5|
     6|import { GlobTool } from './tools/glob.js';
```

#### Gotchas

- **MUST use `absolute_path` parameter** - relative paths not supported
- `offset` is 0-indexed (first line is 0, not 1)
- Binary files detected via heuristic (first 4096 bytes)
- Very large files (>1MB) should use `offset` and `limit` for partial reads
- No automatic truncation - returns full range if no limit specified

---

### 5. `read_many_files` - Batch File Reading

**Tool Name:** `read_many_files`  
**Kind:** `Read`  
**Display Name:** "Read Many Files"  
**Purpose:** Read multiple files efficiently

#### Parameters

```typescript
interface ReadManyFilesParams {
  paths: string[];                    // Array of file/directory paths (REQUIRED)
  include?: string[];                 // Glob patterns to include (e.g., ["*.ts", "*.js"])
  exclude?: string[];                 // Glob patterns to exclude (e.g., ["*.test.ts"])
  recursive?: boolean;                // Recursively search directories (default: false)
  useDefaultExcludes?: boolean;       // Use default exclusions (node_modules, etc.)
  file_filtering_options?: {          // Advanced filtering options
    respect_git_ignore?: boolean;     // Respect .gitignore (default: true)
    respect_gemini_ignore?: boolean;  // Respect .geminiignore (default: true)
  };
  // Note: Line numbers always enabled, no per-file line ranges
}
```

#### Behavior

- **Parallel Reading:** Reads files concurrently
- **Individual Errors:** Failed files don't block others
- **Directory Support:** Can accept directories with `recursive: true`
- **Pattern Matching:** Uses `include`/`exclude` for glob-based filtering
- **Format:** Separates files with headers

#### Example Output

```
Read 3 file(s):

===== File: src/tools/glob.ts (363 lines) =====
     1|import { glob } from 'glob';
     2|import path from 'node:path';
...

===== File: src/tools/grep.ts (690 lines) =====
     1|import { spawn } from 'node:child_process';
...
```

#### Gotchas

- **Use `paths` parameter** (not `file_paths`)
- No per-file line ranges supported (reads entire files)
- Max files not enforced (use responsibly)
- Failures are individual, not fatal
- Directory paths require `recursive: true` to traverse
- `file_filtering_options` applies to all paths

---

### 6. `replace` - File Editing

**Tool Name:** `replace`  
**Kind:** `Edit`  
**Display Name:** "Replace"  
**Purpose:** Edit file contents using search-and-replace

#### Parameters

```typescript
interface ReplaceToolParams {
  file_path: string;        // Absolute path to file
  old_string: string;       // Exact string to find
  new_string: string;       // Replacement string
  require_final_newline?: boolean; // Ensure final newline (default: true)
}
```

#### Behavior

- **Exact Match:** old_string must match **exactly** (whitespace matters)
- **Single Replacement:** Only replaces **first occurrence**
- **Create if Missing:** Creates file if doesn't exist
- **Diff Generation:** Shows unified diff in output
- **Confirmation:** Requires user confirmation (unless auto-accept)

#### Diff Stats

```typescript
interface DiffStat {
  model_added_lines: number;    // Lines added by model
  model_removed_lines: number;  // Lines removed by model
  model_added_chars: number;    // Chars added by model
  model_removed_chars: number;  // Chars removed by model
  user_added_lines: number;     // User edits after proposal
  user_removed_lines: number;   // User edits after proposal
  user_added_chars: number;     // User chars added
  user_removed_chars: number;   // User chars removed
}
```

#### Gotchas

- **Whitespace sensitive** - tabs vs spaces matter
- **Only first match** - not global replace
- **Must be exact** - no fuzzy matching
- Creates file if missing (may surprise users)
- Requires `require_final_newline` for consistent formatting

---

### 7. `write_file` - File Creation

**Tool Name:** `write_file`  
**Kind:** `Edit`  
**Display Name:** "Write File"  
**Purpose:** Create or overwrite files

#### Parameters

```typescript
interface WriteFileParams {
  file_path: string;              // Absolute path to write
  content: string;                // File contents
  require_final_newline?: boolean; // Ensure final newline (default: true)
}
```

#### Behavior

- **Overwrites:** Replaces existing files without warning
- **Creates Dirs:** Auto-creates parent directories
- **Confirmation:** Requires user confirmation
- **Diff:** Shows diff if file exists

#### Example Output

```
Created file: src/new-tool.ts (150 lines)
```

#### Gotchas

- **No backup** - overwrites existing files immediately after confirmation
- **Creates directories** - may create unexpected directory structure
- Parent directory creation is recursive
- No size limit enforced

---

### 8. `run_shell_command` - Shell Execution

**Tool Name:** `run_shell_command`  
**Kind:** `Execute`  
**Display Name:** "Run Shell Command"  
**Purpose:** Execute shell commands

#### Parameters

```typescript
interface ShellCommandParams {
  command: string;        // Command to execute (REQUIRED)
  description?: string;   // Description of what the command does (optional)
  directory?: string;     // Working directory for command execution (optional)
}
```

#### Execution Methods

1. **PTY (Preferred):**
   - Uses `@lydell/node-pty` or `node-pty`
   - Full terminal emulation
   - ANSI color support
   - Interactive command support

2. **child_process (Fallback):**
   - When PTY unavailable
   - No interactive support
   - Limited ANSI support

#### Shell Configuration

**Platform-specific:**
- **Unix/Linux:** `/bin/bash -c` or `$SHELL -c`
- **macOS:** `/bin/zsh -c` or `$SHELL -c`
- **Windows:** `cmd.exe /C` or `powershell.exe -Command`

#### Environment Variables

```typescript
{
  ...process.env,
  GEMINI_CLI: '1',           // Indicates running in Gemini CLI
  TERM: 'xterm-256color',    // Terminal type
  PAGER: 'cat'               // Pager command (configurable)
}
```

#### Resource Limits

- **Buffer Size:** 16MB max output
- **Timeout:** 30 seconds default (configurable)
- **Process Management:** Full process group kill on abort

#### Example Output

```
$ npm test

> test
> vitest run

✓ tests/glob.test.ts (5)
✓ tests/grep.test.ts (8)

Test Files  2 passed (2)
     Tests  13 passed (13)
```

#### Gotchas

- **Interactive commands** may hang in child_process mode
- **Background processes** should use `&` suffix
- **Environment** is inherited but PAGER is overridden
- **Binary output** detected and shown as byte count
- **Working directory** is project root (CWD)
- **Confirmation required** for modifying commands
- Command must be **explained** to user before execution

---

### 9. `google_web_search` - Web Search

**Tool Name:** `google_web_search`  
**Kind:** `Fetch`  
**Display Name:** "Google Web Search"  
**Purpose:** Search the web via Google

#### Parameters

```typescript
interface WebSearchParams {
  query: string;      // Search query
  num_results?: number; // Number of results (default: 10, max: 20)
}
```

#### Behavior

- Uses Google Search API (requires API key)
- Returns titles, URLs, and snippets
- Markdown formatted output
- Respects rate limits

#### Example Output

```
Web search results for "typescript decorators":

1. **TypeScript: Documentation - Decorators**
   https://www.typescriptlang.org/docs/handbook/decorators.html
   Decorators provide a way to add both annotations and a meta-programming syntax...

2. **Understanding TypeScript Decorators**
   https://example.com/ts-decorators
   A comprehensive guide to using decorators in TypeScript...
```

#### Gotchas

- Requires `GOOGLE_API_KEY` and `GOOGLE_CSE_ID` environment variables
- Rate limited by Google API
- Results may be cached
- No automatic retry on failure

---

### 10. `web_fetch` - HTTP Fetching

**Tool Name:** `web_fetch`  
**Kind:** `Fetch`  
**Display Name:** "Web Fetch"  
**Purpose:** Fetch and parse web page content

#### Parameters

```typescript
interface WebFetchParams {
  url: string;           // URL to fetch
  html_to_markdown?: boolean; // Convert HTML to markdown (default: true)
}
```

#### Behavior

- **Fetches:** HTTP/HTTPS URLs
- **Converts:** HTML → Markdown (optional)
- **Extracts:** Main content, strips navigation/ads
- **Timeout:** 30 seconds
- **Size Limit:** 10MB response

#### Example Output

```
Fetched: https://example.com/article

# Article Title

This is the main content extracted from the page...
```

#### Gotchas

- Large pages may be truncated
- JavaScript-rendered content not executed
- Some websites may block the user agent
- Respects robots.txt (should, but not enforced)

---

### 11. `save_memory` - Memory Storage

**Tool Name:** `save_memory`  
**Kind:** `Other`  
**Display Name:** "Save Memory"  
**Purpose:** Store user preferences and facts

#### Parameters

```typescript
interface SaveMemoryParams {
  fact: string;    // Memory fact to save (REQUIRED)
}
```

#### Behavior

- **Storage:** `~/.config/gemini-cli/memory.txt`
- **Format:** Timestamped entries
- **Append-only:** Never overwrites
- **Use Case:** User preferences, facts, coding style

#### Example

```typescript
// Model calls
save_memory({ fact: "User prefers single quotes in TypeScript" });

// Stored as
[2025-10-24 10:30:00] User prefers single quotes in TypeScript
```

#### Gotchas

- Not searchable directly (manual file inspection)
- No deletion mechanism
- Appends to existing memory
- Injected into system prompt on next session

---

### 12. `write_todos` - TODO Management (Experimental)

**Tool Name:** `write_todos`  
**Kind:** `Think`  
**Display Name:** "Write TODOs"  
**Purpose:** Manage task lists

**Status:** Experimental (disabled by default)

#### Parameters

```typescript
interface WriteTodosParams {
  merge: boolean;     // Merge with existing TODOs
  todos: Todo[];      // Array of TODO items
}

interface Todo {
  description: string;
  status: 'pending' | 'in_progress' | 'completed' | 'cancelled';
  id: string;
}
```

#### Behavior

- **Storage:** In-memory during session
- **Display:** Rendered in UI
- **Merge:** Can update existing TODOs by ID

#### Gotchas

- **Experimental** - may be removed
- Disabled by default (`useWriteTodos: false`)
- Not persisted across sessions
- UI-only feature

---

### 13. `smart-edit` - AI-Powered Editing (Experimental)

**Tool Name:** `smart-edit` (replaces `replace` when enabled)  
**Kind:** `Edit`  
**Display Name:** "Smart Edit"  
**Purpose:** Intelligent file editing

**Status:** Experimental

#### Parameters

```typescript
interface SmartEditParams {
  file_path: string;
  old_string: string;
  new_string: string;
  // ... similar to replace
}
```

#### Behavior

- Uses fuzzy matching for old_string
- Attempts to infer correct location
- Falls back to exact match

#### Gotchas

- **Experimental** - may produce unexpected results
- Enabled via `useSmartEdit: true` setting
- May edit wrong location if context is ambiguous
- More permissive than `replace` tool

---

## Tool Parameters & Schemas

### Common Parameter Patterns

#### Absolute Paths

Most file-related tools require **absolute paths**:

```typescript
// ✅ Correct
read_file({ file_path: "/Users/me/project/src/index.ts" })

// ❌ Wrong
read_file({ file_path: "src/index.ts" })
```

**Why:** Eliminates ambiguity about working directory

#### Optional Parameters

Parameters with `?` are optional and have defaults:

```typescript
interface ExampleParams {
  required: string;      // Must provide
  optional?: boolean;    // Has default value
}
```

#### Schema Validation

All parameters are validated via JSON Schema:

```typescript
const schema = {
  type: "object",
  properties: {
    pattern: { type: "string" },
    path: { type: "string" }
  },
  required: ["pattern"],
  additionalProperties: false
};
```

---

## Tool Execution Flow

### Complete Flow Diagram

```
1. Model Requests Tool Call
   ↓
2. Tool Registry Lookup
   ├─ Found → Continue
   └─ Not Found → Error
   ↓
3. Parameter Validation (JSON Schema)
   ├─ Valid → Continue
   └─ Invalid → Return error to model
   ↓
4. Build Tool Invocation
   ↓
5. Check Confirmation Requirements
   ├─ Auto-Accept (safe tools) → Skip to execute
   ├─ Message Bus Policy
   │  ├─ ALLOW → Skip to execute
   │  ├─ DENY → Cancel
   │  └─ ASK_USER → Continue
   └─ User Confirmation
       ├─ Approved → Execute
       ├─ Modify → Rebuild invocation → Execute
       └─ Cancelled → Return error
   ↓
6. Execute Tool
   ├─ Success → Format result
   └─ Error → Format error
   ↓
7. Return to Model (LLM history)
```

### Confirmation Outcomes

```typescript
enum ToolConfirmationOutcome {
  ProceedOnce = 'proceed_once',          // This time only
  ProceedAlways = 'proceed_always',       // Always for all tools
  ProceedAlwaysServer = 'proceed_always_server', // For MCP server
  ProceedAlwaysTool = 'proceed_always_tool',     // For specific tool
  ModifyWithEditor = 'modify_with_editor',       // Edit first
  Cancel = 'cancel'                              // Abort
}
```

---

## Tool Confirmation System

### Auto-Accept (Safe Tools)

Tools that are automatically approved when `autoAccept: true`:

- `glob` (Read only)
- `search_file_content` (Read only)
- `list_directory` (Read only)
- `read_file` (Read only)
- `read_many_files` (Read only)

### Confirmation Details Types

#### Edit Confirmation

```typescript
interface ToolEditConfirmationDetails {
  type: 'edit';
  title: string;
  fileName: string;
  filePath: string;
  fileDiff: string;              // Unified diff
  originalContent: string | null;
  newContent: string;
  isModifying?: boolean;          // Can modify before accept
  ideConfirmation?: Promise<DiffUpdateResult>; // IDE integration
}
```

#### Execute Confirmation

```typescript
interface ToolExecuteConfirmationDetails {
  type: 'exec';
  title: string;
  command: string;      // Full command
  rootCommand: string;  // Base command (e.g., "npm")
}
```

#### MCP Tool Confirmation

```typescript
interface ToolMcpConfirmationDetails {
  type: 'mcp';
  title: string;
  serverName: string;
  toolName: string;
  toolDisplayName: string;
}
```

### Message Bus Integration (Experimental)

When enabled (`enableMessageBusIntegration: true`), tool confirmations go through a policy engine:

```typescript
Tool Call → Message Bus Publish (TOOL_CONFIRMATION_REQUEST)
  ↓
Policy Engine Evaluates
  ├─ ALLOW → Auto-approve
  ├─ DENY → Auto-reject
  └─ ASK_USER → Show UI confirmation
  ↓
Response published (TOOL_CONFIRMATION_RESPONSE)
```

---

## MCP Tools

### Discovery Process

MCP tools are discovered from configured MCP servers:

```yaml
# .gemini/config.yaml
mcpServers:
  my-server:
    command: "node"
    args: ["./server.js"]
    # or
    url: "https://mcp-server.example.com"
    oauth:
      enabled: true
```

### Tool Registration

```typescript
// From MCP server
{
  "name": "custom_tool",
  "description": "Does something custom",
  "parameters": {
    "type": "object",
    "properties": {
      "input": { "type": "string" }
    }
  }
}
```

### Execution

MCP tools are wrapped in `DiscoveredMCPTool`:

```typescript
class DiscoveredMCPTool {
  constructor(
    mcpCallableTool,      // MCP SDK tool wrapper
    serverName,           // Server identifier
    toolName,             // Tool name
    description,          // Tool description
    parameterSchema,      // JSON schema
    trust,                // Trust level
    messageBus,           // Confirmation bus
    config,               // Runtime config
    extensionId           // Optional extension ID
  )
}
```

### Trust Levels

- `trust: true` - Auto-approve tool calls from this server
- `trust: false` (default) - Require confirmation

### Tool Filtering

Include/exclude specific tools from an MCP server:

```yaml
mcpServers:
  my-server:
    command: "node server.js"
    includeTools: ["tool1", "tool2"]  # Only these
    # or
    excludeTools: ["dangerous_tool"]   # All except these
```

---

## Discovered Tools

### Custom Tool Discovery

Define a command to discover custom tools:

```yaml
tools:
  discoveryCommand: "node discover-tools.js"
  callCommand: "node call-tool.js"
```

**Discovery Output Format:**

```json
[
  {
    "name": "my_tool",
    "description": "Does something",
    "function_declarations": [
      {
        "name": "my_tool",
        "parameters": { /* JSON schema */ }
      }
    ]
  }
]
```

**Tool Execution:**

When tool is called, the `callCommand` is executed:

```bash
$ node call-tool.js my_tool
# Receives params via stdin as JSON
# Returns result via stdout as JSON
```

---

## Tool Gotchas & Limitations

### Common Issues

#### 1. **Relative Paths Not Supported**

```typescript
// ❌ Will fail
read_file({ file_path: "src/index.ts" })

// ✅ Must use absolute
read_file({ file_path: "/absolute/path/to/project/src/index.ts" })
```

**Why:** Eliminates ambiguity, ensures security boundary checks work

---

#### 2. **Whitespace Sensitivity in `replace`**

```typescript
// ❌ Won't match if actual file has tabs
replace({
  old_string: "  function foo() {",  // 2 spaces
  // ...
})

// ✅ Match exact whitespace
replace({
  old_string: "\tfunction foo() {",  // tab
  // ...
})
```

**Solution:** Read file first to see exact whitespace

---

#### 3. **First Match Only in `replace`**

```typescript
// Only replaces FIRST occurrence
replace({
  old_string: "TODO",
  new_string: "DONE"
})
// If file has multiple "TODO", only first is replaced
```

**Solution:** Use multiple replace calls or smart-edit

---

#### 4. **Binary File Detection**

Files are detected as binary by checking first 4096 bytes:

```typescript
// Heuristic in isBinary()
if (hasNullByte || hasUncommonControlChars) {
  return true; // Treated as binary
}
```

**Issue:** Some text files with unusual encoding may be misdetected

---

#### 5. **Case-Insensitive grep**

The `search_file_content` tool is **case-insensitive by default**:

```typescript
// Will match "TODO", "Todo", "todo"
search_file_content({ pattern: "todo" })
```

**Solution:** Use regex case-sensitive patterns if needed: `[T]odo`

---

#### 6. **glob Pattern Escaping**

Glob patterns are not escaped by default:

```typescript
// ❌ Will treat [] as character class
glob({ pattern: "file[1].ts" })

// ✅ Escape special chars or use exact filename
```

**Exception:** If pattern matches existing file path, it's auto-escaped

---

#### 7. **Shell Command Hangs**

Interactive commands may hang in child_process mode:

```typescript
// ❌ May hang
run_shell_command({ command: "ssh user@host" })

// ✅ Use non-interactive flags
run_shell_command({ command: "ssh -o BatchMode=yes user@host" })
```

**Solution:** Enable PTY mode or use non-interactive command variations

---

#### 8. **Memory Not Searchable**

The `save_memory` tool appends to a file but doesn't provide search:

```typescript
// ❌ Can't search memory directly
// ✅ Memory is injected into system prompt automatically
```

**Workaround:** Read `~/.config/gemini-cli/memory.txt` manually

---

#### 9. **Tool Output Truncation**

Tool output is truncated to prevent token overflow:

```typescript
// Default limits
DEFAULT_TRUNCATE_TOOL_OUTPUT_THRESHOLD = 50000; // chars
DEFAULT_TRUNCATE_TOOL_OUTPUT_LINES = 100;       // lines
```

**Settings:**

```yaml
tools:
  enableToolOutputTruncation: true
  truncateToolOutputThreshold: 50000
  truncateToolOutputLines: 100
```

---

#### 10. **MCP Tool Timeout**

MCP tools have a 10-minute default timeout:

```yaml
mcpServers:
  my-server:
    timeout: 600000  # 10 minutes in milliseconds
```

**Issue:** Long-running tools may timeout

**Solution:** Increase timeout for specific servers

---

### Performance Considerations

#### Parallel Tool Execution

Independent tools run concurrently:

```typescript
// These run in PARALLEL
model_calls: [
  { name: "read_file", args: { file_path: "/path/a.ts" } },
  { name: "read_file", args: { file_path: "/path/b.ts" } },
  { name: "read_file", args: { file_path: "/path/c.ts" } }
]
```

**Benefit:** 3x faster than sequential

---

#### Read Many Files Optimization

Use `read_many_files` instead of multiple `read_file` calls:

```typescript
// ❌ Slower (3 separate calls)
read_file({ file_path: "a.ts" })
read_file({ file_path: "b.ts" })
read_file({ file_path: "c.ts" })

// ✅ Faster (1 batch call)
read_many_files({ 
  file_paths: ["a.ts", "b.ts", "c.ts"]
})
```

---

#### Grep Strategy Performance

1. **git grep** - Fastest (if available)
2. **system grep** - Fast (if available)
3. **JavaScript** - Slow (fallback)

**Optimization:** Ensure git is installed for best grep performance

---

### Security Considerations

#### Path Traversal Protection

All file paths are validated:

```typescript
// ❌ Blocked
read_file({ file_path: "/etc/passwd" })
// Error: Path is not within workspace

// ❌ Blocked
read_file({ file_path: "/project/../../../etc/passwd" })
// Error: Path traversal detected
```

---

#### Sensitive File Filtering

Auto-blocked patterns:

- `.env*`
- `*credentials*`
- `*secret*`
- `*.key`
- `*.pem`
- `node_modules/`
- `.git/`

**Override:** Not recommended, but possible via `ignorePathFilter`

---

This comprehensive tools reference covers all built-in tools, their parameters, behaviors, and common gotchas. Use this as a complete reference for understanding how Gemini CLI tools work under the hood.

