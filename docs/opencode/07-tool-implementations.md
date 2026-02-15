# OpenCode - Tool Implementations

> **Detailed reference for all built-in tools with code examples and usage patterns**

---

This document covers the 20+ built-in tools that power OpenCode's file operations, code execution, web search, and system interactions.

---

## File Operations

### read - File Reading

**Purpose**: Read file contents with support for offsets, limits, and images.

**Parameters**:
```typescript
{
  filePath: string        // Path to file
  offset?: number        // Line number to start from (0-based)
  limit?: number         // Number of lines to read (default: 2000)
}
```

**Key Features**:
- Line-numbered output for easy reference
- Automatic binary file detection
- Image file support (JPEG, PNG, GIF, WebP, BMP)
- File existence checking with suggestions
- Long line truncation (max 2000 chars/line)
- LSP file tracking integration

**Example Output**:
```
<file>
00001| import { User } from './types'
00002| import { hash } from './crypto'
00003| 
00004| export async function createUser(data: UserInput) {
00005|   const hashed = await hash(data.password)
00006|   return User.create({ ...data, password: hashed })
00007| }
</file>
```

---

### write - File Writing

**Purpose**: Create or overwrite files with content.

**Parameters**:
```typescript
{
  filePath: string    // Path to file
  content: string     // Content to write
}
```

**Key Features**:
- Creates parent directories automatically
- Binary content detection and rejection
- Working directory validation
- File watching trigger

---

### edit - File Editing

**Purpose**: Edit files using search/replace or diff-based patching.

**Parameters**:
```typescript
{
  filePath: string
  oldStr?: string              // Text to find
  newStr?: string              // Replacement text
  replaceAll?: boolean         // Replace all occurrences
  searchPattern?: string       // Regex pattern
  replaceWith?: string         // Replacement pattern
  diff?: string               // Unified diff format
}
```

**Key Features**:
- Precision diff algorithm (`@pierre/precision-diffs`)
- Multiple edit modes: exact match, regex, or diff
- Dry-run mode for validation
- Snapshot creation for revert
- Line-based and fuzzy matching

**Modes**:
1. **Exact Match**: `oldStr` + `newStr`
2. **Regex**: `searchPattern` + `replaceWith`
3. **Diff**: `diff` (unified diff format)

---

### multiedit - Multi-File Editing

**Purpose**: Edit multiple files in a single operation.

**Parameters**:
```typescript
{
  edits: Array<{
    filePath: string
    oldStr: string
    newStr: string
    replaceAll?: boolean
  }>
}
```

**Key Features**:
- Atomic multi-file operations
- Rollback on failure
- Parallel execution
- Progress tracking

---

### patch - Patch Application

**Purpose**: Apply unified diff patches to files.

**Parameters**:
```typescript
{
  patch: string    // Unified diff content
}
```

**Usage**: Apply Git-style patches to files.

---

## Search & Discovery

### grep - Content Search

**Purpose**: Search file contents using patterns.

**Parameters**:
```typescript
{
  pattern: string           // Search pattern (regex)
  caseSensitive?: boolean   // Case sensitivity
  path?: string            // Specific path to search
}
```

**Features**:
- Ripgrep-powered search
- Regex support
- Context lines around matches
- Respects .gitignore

---

### glob - File Pattern Matching

**Purpose**: Find files by glob patterns.

**Parameters**:
```typescript
{
  pattern: string    // Glob pattern (e.g., "**/*.ts")
}
```

**Examples**:
- `**/*.ts` - All TypeScript files
- `src/**/*.test.ts` - Test files in src/
- `*.{js,ts}` - JS or TS files in current dir

---

### ls - Directory Listing

**Purpose**: List directory contents.

**Parameters**:
```typescript
{
  path?: string      // Directory path (default: current)
  recursive?: boolean // Recursive listing
}
```

**Output**: File names, sizes, types, and permissions.

---

## Execution

### bash - Shell Commands

**Purpose**: Execute shell commands.

**Parameters**:
```typescript
{
  command: string         // Shell command
  cwd?: string           // Working directory
  timeout?: number       // Timeout in ms
}
```

**Security Features**:
- Permission system integration
- Timeout enforcement
- Output capture (stdout + stderr)
- Exit code tracking
- Dangerous command detection

**Output Includes**:
- Standard output
- Standard error
- Exit code
- Execution time

---

## Language Server Protocol

### lsp-diagnostics - Get Diagnostics

**Purpose**: Get LSP diagnostics (errors/warnings) for a file.

**Parameters**:
```typescript
{
  filePath: string    // File to check
}
```

**Output**: List of diagnostics with severity, line, column, and message.

---

### lsp-hover - Get Hover Info

**Purpose**: Get type information and documentation at a position.

**Parameters**:
```typescript
{
  filePath: string
  line: number
  column: number
}
```

**Output**: Type information, documentation, and references.

---

## Task Management

### task - Task Tracking

**Purpose**: Create and manage development tasks.

**Parameters**:
```typescript
{
  action: "create" | "list" | "complete" | "delete"
  taskId?: string
  title?: string
  description?: string
}
```

**Features**:
- Task persistence across sessions
- Priority and status tracking
- Due dates
- Task dependencies

---

### todo - TODO Management

**Write**:
```typescript
{
  operation: "write"
  items: Array<{
    content: string
    status: "pending" | "in_progress" | "completed"
  }>
}
```

**Read**:
```typescript
{
  operation: "read"
}
```

**Features**:
- Structured TODO tracking
- Status management
- Progress visualization

---

## Web Access

### webfetch - Fetch Web Content

**Purpose**: Fetch and convert web pages to markdown.

**Parameters**:
```typescript
{
  url: string    // URL to fetch
}
```

**Features**:
- HTML to Markdown conversion (turndown)
- Content extraction
- Image handling
- Error recovery

---

### websearch - Web Search (v1.0+)

**Purpose**: Search the web using Exa AI-powered search API.

**Parameters**:
```typescript
{
  query: string                    // Search query
  numResults?: number              // Number of results (default: 8)
  livecrawl?: "fallback" | "preferred"  // Live crawl mode
  type?: "auto" | "fast" | "deep" // Search type
  contextMaxCharacters?: number    // Max chars for context (default: 10000)
}
```

**Features**:
- AI-powered web search via Exa
- Live crawling support for fresh content
- LLM-optimized context output
- Multiple search modes (auto, fast, deep)
- Permission system integration (`websearch` permission)

**Search Types**:
- `auto` - Balanced search (default)
- `fast` - Quick results, less comprehensive
- `deep` - Comprehensive search, more thorough

**Live Crawl Modes**:
- `fallback` - Use cached content, live crawl as backup
- `preferred` - Prioritize live crawling for fresh data

---

### codesearch - Code Context Search (v1.0+)

**Purpose**: Search for code documentation, API references, and library examples.

**Parameters**:
```typescript
{
  query: string       // Search query for APIs, libraries, SDKs
  tokensNum?: number  // Tokens to return (1000-50000, default: 5000)
}
```

**Features**:
- Exa-powered code documentation search
- Returns LLM-optimized code context
- Supports library/framework queries
- Configurable context size

**Example Queries**:
- "React useState hook examples"
- "Python pandas dataframe filtering"
- "Express.js middleware"
- "Next.js partial prerendering configuration"

**Usage**:
```typescript
// Find React hooks documentation
{ query: "React useEffect cleanup function", tokensNum: 3000 }

// Comprehensive API reference
{ query: "AWS S3 SDK JavaScript multipart upload", tokensNum: 10000 }
```

---

## Plan Mode Tools (v1.0+)

### plan_enter - Enter Plan Mode

**Purpose**: Switch from build agent to plan agent for research and planning.

**Parameters**: None

**Behavior**:
1. Prompts user to confirm mode switch
2. Creates new message switching to plan agent
3. Plan saved to `.opencode/plans/session-<id>.md`

**Use Case**: When you need to research and plan before making changes.

---

### plan_exit - Exit Plan Mode

**Purpose**: Switch from plan agent back to build agent for implementation.

**Parameters**: None

**Behavior**:
1. Prompts user to confirm mode switch
2. Creates new message switching to build agent
3. Instructs agent to execute the plan

**Use Case**: When planning is complete and ready to implement.

---

## Batch Tool - Parallel Tool Execution

**Purpose**: Execute up to 25 tools in parallel for efficient multi-operation workflows.

**Parameters**:
```typescript
{
  tool_calls: Array<{
    tool: string       // Tool name to execute
    parameters: object // Tool-specific parameters
  }>
}
```

**Key Features**:
- Execute up to 25 tools simultaneously
- Automatic progress tracking per tool call
- Partial success handling (some tools can fail while others succeed)
- Session part updates for each individual call

**Disallowed Tools** (cannot be batched):
- `batch` - Prevents recursive batching
- Other tools with interactive requirements

**Error Handling**:
- Each tool execution is independent
- Failed tools don't block successful ones
- Metadata output includes success/failure counts per tool
- Detailed error messages for each failed tool

**Example Usage**:
```typescript
{
  tool_calls: [
    { tool: "read", parameters: { filePath: "src/index.ts" } },
    { tool: "read", parameters: { filePath: "src/utils.ts" } },
    { tool: "grep", parameters: { pattern: "TODO", path: "src/" } }
  ]
}
```

**Output Metadata**:
```typescript
{
  total: 3,
  successful: 3,
  failed: 0,
  results: [
    { tool: "read", status: "success", output: "..." },
    { tool: "read", status: "success", output: "..." },
    { tool: "grep", status: "success", output: "..." }
  ]
}
```

---

## Question Tool - Structured User Questions

**Purpose**: Ask structured questions to the user during tool execution with multiple choice support.

**Parameters**:
```typescript
{
  questions: Array<{
    question: string      // Complete question text
    header: string        // Short label (max 30 characters)
    options: Array<{
      label: string       // Option text (1-5 words)
      description: string // Option explanation
    }>
    multiple?: boolean    // Allow multiple selections (default: false)
  }>
}
```

**Key Features**:
- Structured question format for consistent UX
- Multiple choice with descriptions
- Single or multiple selection modes
- Custom answer support via text input
- Integration with Question system for async responses

**Question Types**:
1. **Single Choice**: User selects one option
2. **Multiple Choice**: User can select multiple options (`multiple: true`)
3. **Custom Input**: When no options provided, accepts free-form text

**Example - Single Choice**:
```typescript
{
  questions: [{
    question: "Which testing framework should we use for this project?",
    header: "Test Framework",
    options: [
      { label: "Jest", description: "Popular, great for React projects" },
      { label: "Vitest", description: "Fast, Vite-native testing" },
      { label: "Mocha", description: "Flexible, callback-based" }
    ]
  }]
}
```

**Example - Multiple Choice**:
```typescript
{
  questions: [{
    question: "Which linters should we configure?",
    header: "Linter Selection",
    options: [
      { label: "ESLint", description: "JavaScript/TypeScript linting" },
      { label: "Prettier", description: "Code formatting" },
      { label: "Stylelint", description: "CSS/SCSS linting" }
    ],
    multiple: true
  }]
}
```

---

## Question System Internal API

The Question system (`src/question/`) provides the internal infrastructure for user interactions.

**Core Methods**:

### Question.ask()
Prompts the user with a question and waits for response.
```typescript
const response = await Question.ask({
  question: "Continue with deployment?",
  options: [
    { label: "Yes", description: "Deploy to production" },
    { label: "No", description: "Cancel deployment" }
  ]
})
```

### Question.reply()
Handles user response to a pending question.
```typescript
Question.reply(questionId, selectedOptions)
```

### Question.reject()
Dismisses a pending question without answer.
```typescript
Question.reject(questionId)
```

### Question.RejectedError
Error class thrown when a question is rejected/dismissed.
```typescript
try {
  const answer = await Question.ask(...)
} catch (error) {
  if (error instanceof Question.RejectedError) {
    // User dismissed the question
  }
}
```

**Events**:
- `Asked` - Fired when a question is displayed
- `Replied` - Fired when user provides answer
- `Rejected` - Fired when question is dismissed

**TUI Integration**:
Questions render as interactive prompts in the terminal UI with keyboard navigation for option selection.

---

## Tool Permission Matrix

| Tool | Permission Required | Category |
|------|---------------------|----------|
| `read` | None | File Read |
| `write` | `write` | File Write |
| `edit` | `write` | File Write |
| `multiedit` | `write` | File Write |
| `patch` | `write` | File Write |
| `grep` | None | Search |
| `glob` | None | Search |
| `ls` | None | Search |
| `bash` | `bash` or `bash:*` | Execution |
| `lsp-diagnostics` | None | LSP |
| `lsp-hover` | None | LSP |
| `task` | None | Task |
| `todo` | None | Task |
| `webfetch` | `webfetch` | Web |
| `websearch` | `websearch` | Web |
| `codesearch` | `codesearch` | Web |
| `plan_enter` | None | Mode |
| `plan_exit` | None | Mode |
| `batch` | Inherits from batched tools | Meta |
| `question` | None | Interactive |

---

## Edit Tool - Precision Diff Algorithm

The edit tool uses `@pierre/precision-diffs` for intelligent file modifications.

**Algorithm Features**:
- **Exact Match**: Direct string replacement when `oldStr` uniquely identifies target
- **Fuzzy Matching**: Handles minor whitespace differences
- **Line-based Matching**: Falls back to line-by-line comparison
- **Context-aware**: Uses surrounding context for ambiguous matches

**Matching Priority**:
1. Exact string match (fastest)
2. Normalized whitespace match
3. Line-based fuzzy match
4. Contextual line matching

**Validation**:
- Ensures `oldStr` exists in file before edit
- Warns on multiple matches (ambiguous)
- Creates snapshot for revert capability

**Error Recovery**:
- Automatic rollback on failed edits
- Detailed error messages for debugging
- Suggests alternatives for failed matches

---

## Websearch Tool - Exa Integration Details

**API Integration Details**:
- Uses Exa AI-powered search API
- Requires `OPENCODE_ENABLE_EXA` flag or experimental mode
- API key configured via provider settings

**Search Types**:
| Type | Description | Use Case |
|------|-------------|----------|
| `auto` | Balanced search | General queries |
| `fast` | Quick results | Time-sensitive lookups |
| `deep` | Comprehensive | Research, thorough search |

**Live Crawl Modes**:
- `fallback`: Use cached content, crawl if stale
- `preferred`: Always attempt fresh crawl

**Rate Limits**:
- Subject to Exa API rate limits
- Implements automatic retry with backoff
- Caches results to minimize API calls

**Error Handling**:
- Network errors trigger retry
- Invalid queries return helpful messages
- Timeout after 30 seconds

---

## Bash Tool - Extended Documentation

**Complete Parameters**:
```typescript
{
  command: string         // Shell command to execute
  cwd?: string           // Working directory (default: project root)
  timeout?: number       // Timeout in milliseconds (default: 30000)
  background?: boolean   // Run in background (experimental)
}
```

**Security Features**:
- Permission system integration (`bash` permission required)
- Dangerous command detection and warning
- Timeout enforcement to prevent runaway processes
- Output capture with size limits
- Exit code tracking for error detection

**Dangerous Commands** (trigger warnings):
- `rm -rf /` or similar destructive patterns
- `sudo` commands
- Commands modifying system files
- Git force push operations

**Output Limits**:
- Default max output: 100KB
- Configurable via `OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH`
- Long outputs are truncated with indicator

**Timeout Configuration**:
- Default: 30 seconds (30000ms)
- Configurable via `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS`
- Per-call override via `timeout` parameter

---

## Tool Usage Patterns

### Reading Multiple Files

```typescript
// Parallel reads
{
  recipient_name: "multi_tool_use.parallel",
  parameters: {
    tool_uses: [
      { recipient_name: "functions.read", parameters: { filePath: "a.ts" } },
      { recipient_name: "functions.read", parameters: { filePath: "b.ts" } },
      { recipient_name: "functions.read", parameters: { filePath: "c.ts" } }
    ]
  }
}
```

### Search and Edit Pattern

```
1. grep: Find occurrences
2. read: Verify context
3. edit: Make changes
4. bash: Run tests
```

### Refactoring Workflow

```
1. glob: Find all affected files
2. multiedit: Update all files
3. lsp-diagnostics: Check for errors
4. bash: Run type checker
```

---

## Best Practices

**File Operations**:
- Always `read` before `edit` to understand context
- Use `multiedit` for related changes across files
- Check `lsp-diagnostics` after edits

**Search**:
- Use `glob` for file discovery
- Use `grep` for content search
- Combine for targeted operations

**Execution**:
- Use `bash` judiciously (permissions required)
- Set reasonable timeouts
- Check exit codes

**Web**:
- Use `webfetch` for documentation/references
- Cache results when possible

---

For detailed implementation code, see the source files in `packages/opencode/src/tool/`.



---

# Enhanced Tool Documentation

---

## Batch Tool - Parallel Tool Execution

**Purpose**: Execute up to 25 tools in parallel for efficient multi-operation workflows.

**Parameters**:
```typescript
{
  tool_calls: Array<{
    tool: string       // Tool name to execute
    parameters: object // Tool-specific parameters
  }>
}
```

**Key Features**:
- Execute up to 25 tools simultaneously
- Automatic progress tracking per tool call
- Partial success handling (some tools can fail while others succeed)
- Session part updates for each individual call

**Disallowed Tools** (cannot be batched):
- `batch` - Prevents recursive batching
- Other tools with interactive requirements

**Error Handling**:
- Each tool execution is independent
- Failed tools don't block successful ones
- Metadata output includes success/failure counts per tool
- Detailed error messages for each failed tool

**Example Usage**:
```typescript
{
  tool_calls: [
    { tool: "read", parameters: { filePath: "src/index.ts" } },
    { tool: "read", parameters: { filePath: "src/utils.ts" } },
    { tool: "grep", parameters: { pattern: "TODO", path: "src/" } }
  ]
}
```

**Output Metadata**:
```typescript
{
  total: 3,
  successful: 3,
  failed: 0,
  results: [
    { tool: "read", status: "success", output: "..." },
    { tool: "read", status: "success", output: "..." },
    { tool: "grep", status: "success", output: "..." }
  ]
}
```

---

## Question Tool - Structured User Questions

**Purpose**: Ask structured questions to the user during tool execution with multiple choice support.

**Parameters**:
```typescript
{
  questions: Array<{
    question: string      // Complete question text
    header: string        // Short label (max 30 characters)
    options: Array<{
      label: string       // Option text (1-5 words)
      description: string // Option explanation
    }>
    multiple?: boolean    // Allow multiple selections (default: false)
  }>
}
```

**Key Features**:
- Structured question format for consistent UX
- Multiple choice with descriptions
- Single or multiple selection modes
- Custom answer support via text input
- Integration with Question system for async responses

**Question Types**:
1. **Single Choice**: User selects one option
2. **Multiple Choice**: User can select multiple options (`multiple: true`)
3. **Custom Input**: When no options provided, accepts free-form text

**Example - Single Choice**:
```typescript
{
  questions: [{
    question: "Which testing framework should we use for this project?",
    header: "Test Framework",
    options: [
      { label: "Jest", description: "Popular, great for React projects" },
      { label: "Vitest", description: "Fast, Vite-native testing" },
      { label: "Mocha", description: "Flexible, callback-based" }
    ]
  }]
}
```

**Example - Multiple Choice**:
```typescript
{
  questions: [{
    question: "Which linters should we configure?",
    header: "Linter Selection",
    options: [
      { label: "ESLint", description: "JavaScript/TypeScript linting" },
      { label: "Prettier", description: "Code formatting" },
      { label: "Stylelint", description: "CSS/SCSS linting" }
    ],
    multiple: true
  }]
}
```

---

## Question System Internal API

The Question system (`src/question/`) provides the internal infrastructure for user interactions.

**Core Methods**:

### Question.ask()
Prompts the user with a question and waits for response.
```typescript
const response = await Question.ask({
  question: "Continue with deployment?",
  options: [
    { label: "Yes", description: "Deploy to production" },
    { label: "No", description: "Cancel deployment" }
  ]
})
```

### Question.reply()
Handles user response to a pending question.
```typescript
Question.reply(questionId, selectedOptions)
```

### Question.reject()
Dismisses a pending question without answer.
```typescript
Question.reject(questionId)
```

### Question.RejectedError
Error class thrown when a question is rejected/dismissed.
```typescript
try {
  const answer = await Question.ask(...)
} catch (error) {
  if (error instanceof Question.RejectedError) {
    // User dismissed the question
  }
}
```

**Events**:
- `Asked` - Fired when a question is displayed
- `Replied` - Fired when user provides answer
- `Rejected` - Fired when question is dismissed

**TUI Integration**:
Questions render as interactive prompts in the terminal UI with keyboard navigation for option selection.

---

## Tool Permission Matrix

| Tool | Permission Required | Category |
|------|---------------------|----------|
| `read` | None | File Read |
| `write` | `write` | File Write |
| `edit` | `write` | File Write |
| `multiedit` | `write` | File Write |
| `patch` | `write` | File Write |
| `grep` | None | Search |
| `glob` | None | Search |
| `ls` | None | Search |
| `bash` | `bash` or `bash:*` | Execution |
| `lsp-diagnostics` | None | LSP |
| `lsp-hover` | None | LSP |
| `task` | None | Task |
| `todo` | None | Task |
| `webfetch` | `webfetch` | Web |
| `websearch` | `websearch` | Web |
| `codesearch` | `codesearch` | Web |
| `plan_enter` | None | Mode |
| `plan_exit` | None | Mode |
| `batch` | Inherits from batched tools | Meta |
| `question` | None | Interactive |

---

## Bash Tool - Extended Documentation

**Complete Parameters**:
```typescript
{
  command: string         // Shell command to execute
  cwd?: string           // Working directory (default: project root)
  timeout?: number       // Timeout in milliseconds (default: 30000)
  background?: boolean   // Run in background (experimental)
}
```

**Security Features**:
- Permission system integration (`bash` permission required)
- Dangerous command detection and warning
- Timeout enforcement to prevent runaway processes
- Output capture with size limits
- Exit code tracking for error detection

**Dangerous Commands** (trigger warnings):
- `rm -rf /` or similar destructive patterns
- `sudo` commands
- Commands modifying system files
- Git force push operations

**Output Limits**:
- Default max output: 100KB
- Configurable via `OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH`
- Long outputs are truncated with indicator

**Timeout Configuration**:
- Default: 30 seconds (30000ms)
- Configurable via `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS`
- Per-call override via `timeout` parameter

---

## Edit Tool - Precision Diff Algorithm

The edit tool uses `@pierre/precision-diffs` for intelligent file modifications.

**Algorithm Features**:
- **Exact Match**: Direct string replacement when `oldStr` uniquely identifies target
- **Fuzzy Matching**: Handles minor whitespace differences
- **Line-based Matching**: Falls back to line-by-line comparison
- **Context-aware**: Uses surrounding context for ambiguous matches

**Matching Priority**:
1. Exact string match (fastest)
2. Normalized whitespace match
3. Line-based fuzzy match
4. Contextual line matching

**Validation**:
- Ensures `oldStr` exists in file before edit
- Warns on multiple matches (ambiguous)
- Creates snapshot for revert capability

**Error Recovery**:
- Automatic rollback on failed edits
- Detailed error messages for debugging
- Suggests alternatives for failed matches

---

## Websearch Tool - Exa Integration

**API Integration Details**:
- Uses Exa AI-powered search API
- Requires `OPENCODE_ENABLE_EXA` flag or experimental mode
- API key configured via provider settings

**Search Types**:
| Type | Description | Use Case |
|------|-------------|----------|
| `auto` | Balanced search | General queries |
| `fast` | Quick results | Time-sensitive lookups |
| `deep` | Comprehensive | Research, thorough search |

**Live Crawl Modes**:
- `fallback`: Use cached content, crawl if stale
- `preferred`: Always attempt fresh crawl

**Rate Limits**:
- Subject to Exa API rate limits
- Implements automatic retry with backoff
- Caches results to minimize API calls

**Error Handling**:
- Network errors trigger retry
- Invalid queries return helpful messages
- Timeout after 30 seconds

---

## Related Documentation

- [06-tool-system.md](./06-tool-system.md) - Tool architecture
- [14-security-permissions.md](./14-security-permissions.md) - Permission system
- [13-configuration.md](./13-configuration.md) - Configuration options
