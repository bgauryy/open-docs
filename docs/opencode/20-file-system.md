# OpenCode - File System

> **File operations, watching, and search utilities**

---

## Overview

OpenCode's file system module provides:
- **File operations** - Read, write, watch
- **Search** - Ripgrep integration
- **Fuzzy finding** - FZF integration
- **Ignore patterns** - .gitignore support
- **Time tracking** - File access tracking

**Files**:
- `file/index.ts` - Core file operations
- `file/ripgrep.ts` - Search integration
- `file/fzf.ts` - Fuzzy finder
- `file/ignore.ts` - Ignore patterns
- `file/watcher.ts` - File watching
- `file/time.ts` - Access time tracking

---

## File Operations

### Basic Operations

```typescript
import { Filesystem } from "./util/filesystem"

// Read file
const content = await Bun.file("path/to/file.ts").text()

// Write file
await Bun.write("path/to/file.ts", content)

// Check existence
const exists = await Bun.file("path/to/file.ts").exists()

// Get file info
const stat = await Bun.file("path/to/file.ts").stat()
```

### Path Operations

```typescript
// Find file upwards
const found = await Filesystem.findUp(
  "package.json",
  "/current/dir",
  "/workspace/root"
)
// Returns: ["/workspace/package.json"]

// Glob pattern upwards
const configs = await Filesystem.globUp(
  "*.config.{js,ts}",
  "/current/dir",
  "/workspace/root"
)
// Returns: ["/workspace/vite.config.ts", ...]

// Check if path contains another
const contains = Filesystem.contains("/workspace", "/workspace/src/file.ts")
// Returns: true
```

---

## Ripgrep Integration

**File**: `file/ripgrep.ts`

### Search Content

```typescript
import { Ripgrep } from "./file/ripgrep"

// Search for pattern
const results = await Ripgrep.search({
  pattern: "export function",
  cwd: "/project",
  limit: 100,
})

// Results:
// [
//   { file: "src/auth.ts", line: 23, text: "export function login() {" },
//   { file: "src/user.ts", line: 45, text: "export function getUser() {" },
// ]
```

### Project Tree

```typescript
// Generate file tree
const tree = await Ripgrep.tree({
  cwd: "/project",
  limit: 200,
})

// Output:
// src/
//   auth.ts
//   user.ts
//   db.ts
// tests/
//   auth.test.ts
```

---

## FZF Integration

**File**: `file/fzf.ts`

### Fuzzy File Finder

```typescript
import { FZF } from "./file/fzf"

// Find files
const selected = await FZF.find({
  cwd: "/project",
  prompt: "Select file:",
  preview: true,
})

// Returns selected file path
```

---

## Ignore Patterns

**File**: `file/ignore.ts`

### .gitignore Support

```typescript
import { Ignore } from "./file/ignore"

// Load .gitignore
const ig = await Ignore.load("/project")

// Check if ignored
const ignored = ig.ignores("node_modules/package/file.js")
// Returns: true

// Filter file list
const files = ["src/auth.ts", "node_modules/lib.js", "dist/bundle.js"]
const filtered = files.filter(f => !ig.ignores(f))
// Returns: ["src/auth.ts"]
```

### Custom Patterns

```typescript
const ig = Ignore.create()
ig.add("*.log")
ig.add("dist/")
ig.add("node_modules/")

const shouldInclude = !ig.ignores("src/file.ts")
```

---

## File Watching

**File**: `file/watcher.ts`

### Watch for Changes

```typescript
import { Watcher } from "./file/watcher"

const watcher = await Watcher.create({
  paths: ["/project/src"],
  ignore: ["node_modules", ".git"],
  onChange: (event) => {
    console.log(`${event.type}: ${event.path}`)
  }
})

// Stop watching
watcher.close()
```

### Events

```typescript
type WatchEvent =
  | { type: "add", path: string }
  | { type: "change", path: string }
  | { type: "unlink", path: string }
```

---

## Time Tracking

**File**: `file/time.ts`

### Track File Access

```typescript
import { FileTime } from "./file/time"

// Mark file as read
FileTime.read("session_123", "/project/src/auth.ts")

// Mark file as edited
FileTime.edit("session_123", "/project/src/auth.ts")

// Get recently accessed files
const recent = FileTime.recent("session_123", 10)
// Returns: ["/project/src/auth.ts", ...]

// Get file stats
const stats = FileTime.stats("session_123")
// Returns: { reads: 45, edits: 12, files: 23 }
```

### Use Cases

- Show recently edited files
- Track file access patterns
- Generate activity reports
- Suggest relevant files

---

## Best Practices

**File Operations**:
- Always check file existence
- Handle errors gracefully
- Use absolute paths
- Respect .gitignore

**Search**:
- Limit result count
- Use specific patterns
- Filter by file type
- Cache results when possible

**Watching**:
- Watch specific directories
- Debounce rapid changes
- Clean up watchers
- Handle errors

**Performance**:
- Use ripgrep for large searches
- Stream large files
- Implement pagination
- Cache frequently accessed files

---

For implementation, see `packages/opencode/src/file/`.

