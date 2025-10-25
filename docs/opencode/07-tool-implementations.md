# OpenCode - Tool Implementations

> **Detailed reference for all built-in tools with code examples and usage patterns**

---

This document covers the 13 built-in tools that power OpenCode's file operations, code execution, and system interactions.

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

