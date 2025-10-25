# API Reference

This document provides a complete reference for all APIs exposed by OpenCode, organized by component.

## CLI Commands

### RunCommand

**File**: `packages/opencode/src/cli/cmd/run.ts:31`

Start an interactive coding session with AI.

**Signature**:
```bash
opencode run [message..] [--model MODEL] [--continue] [--session ID]
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `message` | `string[]` | Initial message to send to AI |
| `--model` | `string` | AI model to use |
| `--continue` | `boolean` | Continue last session |
| `--session` | `string` | Resume specific session by ID |

**Examples**:
```bash
# Start new session
opencode run "help me build a REST API"

# Continue last session
opencode run --continue "add authentication"

# Use specific model
opencode run --model claude-3-opus "refactor this code"

# Resume specific session
opencode run --session abc123 "continue working"
```

---

### ServeCommand

**File**: `packages/opencode/src/cli/cmd/serve.ts:4`

Start HTTP server for remote access.

**Signature**:
```bash
opencode serve [--port PORT]
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `--port` | `number` | Port to listen on |

**Examples**:
```bash
# Start on default port (3000)
opencode serve

# Start on custom port
opencode serve --port 8080
```

---

### AcpCommand

**File**: `packages/opencode/src/cli/cmd/acp.ts:16`

Start Agent Client Protocol server.

**Signature**:
```bash
opencode acp
```

**Examples**:
```bash
# Start ACP server
opencode acp
```

---

### McpCommand

**File**: `packages/opencode/src/cli/cmd/mcp.ts:7`

Manage Model Context Protocol integrations.

**Signature**:
```bash
opencode mcp [add|list]
```

**Examples**:
```bash
# Add MCP server
opencode mcp add

# List configured servers
opencode mcp list
```

---

### TuiCommand

**File**: `packages/opencode/src/cli/cmd/tui.ts:29`

Launch terminal UI.

**Signature**:
```bash
opencode tui
```

**Examples**:
```bash
# Launch TUI
opencode tui
```

---

### AuthCommand

**File**: `packages/opencode/src/cli/cmd/auth.ts:13`

Manage authentication.

**Signature**:
```bash
opencode auth [login|logout|list]
```

**Examples**:
```bash
# Login
opencode auth login

# Logout
opencode auth logout

# List auth providers
opencode auth list
```

---

### AgentCommand

**File**: `packages/opencode/src/cli/cmd/agent.ts:136`

Run predefined agent workflows.

**Signature**:
```bash
opencode agent [name]
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | Agent workflow name |

**Examples**:
```bash
# Run agent
opencode agent my-workflow
```

---

## Tools

### BashTool

**File**: `packages/opencode/src/tool/bash.ts:46`

Execute shell commands.

**Signature**:
```typescript
Tool.define("bash", {command: string, timeout?: number})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `command` | `string` | Shell command to execute |
| `timeout` | `number` | Command timeout in milliseconds |

**Returns**: `{stdout: string, stderr: string, exitCode: number}`

**Example**:
```typescript
const result = await Tool.execute('bash', {
  command: 'npm test',
  timeout: 30000
});
console.log(result.stdout);
```

---

### EditTool

**File**: `packages/opencode/src/tool/edit.ts:20`

Edit files with exact string replacement.

**Signature**:
```typescript
Tool.define("edit", {file_path: string, old_string: string, new_string: string})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_path` | `string` | Path to file to edit |
| `old_string` | `string` | String to find and replace |
| `new_string` | `string` | Replacement string |

**Returns**: `{success: boolean, message: string}`

**Example**:
```typescript
const result = await Tool.execute('edit', {
  file_path: 'src/api.ts',
  old_string: 'const API_URL = "http://localhost"',
  new_string: 'const API_URL = "https://api.example.com"'
});
```

---

### ReadTool

**File**: `packages/opencode/src/tool/read.ts:16`

Read file contents.

**Signature**:
```typescript
Tool.define("read", {file_path: string, offset?: number, limit?: number})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_path` | `string` | Path to file to read |
| `offset` | `number` | Line offset to start reading |
| `limit` | `number` | Number of lines to read |

**Returns**: `string`

**Example**:
```typescript
// Read entire file
const content = await Tool.execute('read', {
  file_path: 'src/index.ts'
});

// Read specific lines
const partial = await Tool.execute('read', {
  file_path: 'src/index.ts',
  offset: 10,
  limit: 20
});
```

---

### WriteTool

**File**: `packages/opencode/src/tool/write.ts:14`

Write content to file.

**Signature**:
```typescript
Tool.define("write", {file_path: string, content: string})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `file_path` | `string` | Path to file to write |
| `content` | `string` | Content to write |

**Returns**: `{success: boolean}`

**Example**:
```typescript
const result = await Tool.execute('write', {
  file_path: 'src/config.json',
  content: JSON.stringify({ api: 'https://api.example.com' }, null, 2)
});
```

---

### GrepTool

**File**: `packages/opencode/src/tool/grep.ts:8`

Search for patterns in files using ripgrep.

**Signature**:
```typescript
Tool.define("grep", {pattern: string, path?: string, type?: string})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `pattern` | `string` | Regex pattern to search |
| `path` | `string` | Directory to search in |
| `type` | `string` | File type filter |

**Returns**: `string[]`

**Example**:
```typescript
// Search in current directory
const matches = await Tool.execute('grep', {
  pattern: 'function.*export'
});

// Search in specific directory with type filter
const tsMatches = await Tool.execute('grep', {
  pattern: 'interface',
  path: './src',
  type: 'ts'
});
```

---

### GlobTool

**File**: `packages/opencode/src/tool/glob.ts:8`

Find files matching glob pattern.

**Signature**:
```typescript
Tool.define("glob", {pattern: string, path?: string})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `pattern` | `string` | Glob pattern |
| `path` | `string` | Directory to search in |

**Returns**: `string[]`

**Example**:
```typescript
// Find all TypeScript files
const tsFiles = await Tool.execute('glob', {
  pattern: '**/*.ts'
});

// Find files in specific directory
const srcFiles = await Tool.execute('glob', {
  pattern: '*.ts',
  path: './src'
});
```

---

### LspDiagnosticTool

**File**: `packages/opencode/src/tool/lsp-diagnostics.ts:8`

Get language server diagnostics.

**Signature**:
```typescript
Tool.define("lsp_diagnostics", {uri?: string})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `uri` | `string` | File URI to get diagnostics for |

**Returns**: `Diagnostic[]`

**Example**:
```typescript
// Get all diagnostics
const diagnostics = await Tool.execute('lsp_diagnostics', {});

// Get diagnostics for specific file
const fileDiagnostics = await Tool.execute('lsp_diagnostics', {
  uri: 'file:///path/to/file.ts'
});
```

---

### TodoWriteTool

**File**: `packages/opencode/src/tool/todo.ts:6`

Write todo list for task tracking.

**Signature**:
```typescript
Tool.define("todowrite", {todos: Todo[]})
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `todos` | `Todo[]` | Array of todo items with status |

**Returns**: `{success: boolean}`

**Example**:
```typescript
const result = await Tool.execute('todowrite', {
  todos: [
    {
      content: 'Implement authentication',
      status: 'in_progress',
      activeForm: 'Implementing authentication'
    },
    {
      content: 'Write tests',
      status: 'pending',
      activeForm: 'Writing tests'
    }
  ]
});
```

---

## Session Management

### Session.create

**File**: `packages/opencode/src/session/index.ts`

Create new AI session.

**Signature**:
```typescript
async Session.create(opts: SessionOptions): Promise<Session>
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `opts` | `SessionOptions` | Session configuration |

**Returns**: `Promise<Session>`

**Example**:
```typescript
const session = await Session.create({
  model: 'claude-3-sonnet',
  tools: ['bash', 'edit', 'read', 'write']
});
```

---

### Session.resume

**File**: `packages/opencode/src/session/index.ts`

Resume existing session.

**Signature**:
```typescript
async Session.resume(sessionId: string): Promise<Session>
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `sessionId` | `string` | Session ID to resume |

**Returns**: `Promise<Session>`

**Example**:
```typescript
const session = await Session.resume('abc123');
```

---

### Session.message

**File**: `packages/opencode/src/session/index.ts`

Send message and stream response.

**Signature**:
```typescript
async Session.message(content: string): AsyncGenerator<MessageChunk>
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `content` | `string` | User message |

**Returns**: `AsyncGenerator<MessageChunk>`

**Example**:
```typescript
const response = session.message('Help me build a feature');
for await (const chunk of response) {
  if (chunk.type === 'text') {
    process.stdout.write(chunk.content);
  }
}
```

---

## Provider Integration

### Provider.stream

**File**: `packages/opencode/src/provider/provider.ts`

Stream AI responses from provider.

**Signature**:
```typescript
async Provider.stream(opts: StreamOptions): AsyncGenerator<TextChunk>
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `opts` | `StreamOptions` | Streaming configuration with model, messages, tools |

**Returns**: `AsyncGenerator<TextChunk>`

**Example**:
```typescript
const stream = Provider.stream({
  model: 'claude-3-sonnet',
  messages: [
    { role: 'user', content: 'Hello' }
  ],
  tools: ['bash', 'edit', 'read']
});

for await (const chunk of stream) {
  console.log(chunk.content);
}
```

---

## File Operations

### File.ripgrep

**File**: `packages/opencode/src/file/ripgrep.ts`

Search files using ripgrep.

**Signature**:
```typescript
async File.ripgrep(pattern: string, opts?: RipgrepOptions): Promise<Match[]>
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `pattern` | `string` | Search pattern |
| `opts` | `RipgrepOptions` | Search options |

**Returns**: `Promise<Match[]>`

**Example**:
```typescript
const matches = await File.ripgrep('TODO', {
  path: './src',
  type: 'ts'
});

for (const match of matches) {
  console.log(`${match.file}:${match.line}: ${match.text}`);
}
```

---

### File.fzf

**File**: `packages/opencode/src/file/fzf.ts`

Fuzzy find files.

**Signature**:
```typescript
async File.fzf(query: string): Promise<string[]>
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | `string` | Fuzzy search query |

**Returns**: `Promise<string[]>`

**Example**:
```typescript
const files = await File.fzf('index');
console.log('Matching files:', files);
```

---

### File.watch

**File**: `packages/opencode/src/file/watcher.ts`

Watch file/directory for changes.

**Signature**:
```typescript
File.watch(path: string, callback: (event) => void): Watcher
```

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | `string` | Path to watch |
| `callback` | `function` | Change callback |

**Returns**: `Watcher`

**Example**:
```typescript
const watcher = File.watch('./src', (event) => {
  console.log(`File ${event.type}: ${event.path}`);
});

// Stop watching
watcher.close();
```

---

## Type Definitions

### SessionOptions

```typescript
interface SessionOptions {
  model: string;              // AI model name
  tools?: string[];           // Available tools
  systemPrompt?: string;      // Custom system prompt
  temperature?: number;       // Temperature (0-1)
  maxTokens?: number;        // Max response tokens
}
```

### MessageChunk

```typescript
interface MessageChunk {
  type: 'text' | 'tool_call' | 'tool_result';
  content?: string;
  tool?: string;
  params?: Record<string, any>;
  result?: any;
}
```

### StreamOptions

```typescript
interface StreamOptions {
  model: string;
  messages: Message[];
  tools?: Tool[];
  temperature?: number;
  maxTokens?: number;
}
```

### RipgrepOptions

```typescript
interface RipgrepOptions {
  path?: string;            // Search path
  type?: string;            // File type filter
  caseInsensitive?: boolean;
  fixed?: boolean;          // Fixed string search
  maxMatches?: number;      // Max matches per file
}
```

### Match

```typescript
interface Match {
  file: string;            // File path
  line: number;            // Line number
  column: number;          // Column number
  text: string;            // Matching line text
}
```

### Diagnostic

```typescript
interface Diagnostic {
  uri: string;             // File URI
  range: {
    start: { line: number; character: number };
    end: { line: number; character: number };
  };
  severity: 'error' | 'warning' | 'info' | 'hint';
  message: string;
  source?: string;
}
```

### Todo

```typescript
interface Todo {
  content: string;         // Todo description
  status: 'pending' | 'in_progress' | 'completed';
  activeForm: string;      // Present continuous form
}
```

---

## Error Handling

All async APIs may throw errors. Always wrap in try-catch:

```typescript
try {
  const result = await Tool.execute('bash', {
    command: 'npm test'
  });
} catch (error) {
  if (error.code === 'TIMEOUT') {
    console.error('Command timed out');
  } else if (error.code === 'EXECUTION_ERROR') {
    console.error('Command failed:', error.message);
  }
}
```

### Common Error Codes

- `TIMEOUT` - Operation timed out
- `EXECUTION_ERROR` - Execution failed
- `VALIDATION_ERROR` - Invalid parameters
- `FILE_NOT_FOUND` - File doesn't exist
- `PERMISSION_ERROR` - Insufficient permissions
- `PROTOCOL_ERROR` - Protocol violation

---

## Rate Limits

When using AI providers, be aware of rate limits:

- **Anthropic**: Varies by tier
- **OpenAI**: Varies by tier
- **Local models**: No limits

The session manager automatically handles rate limiting and retries.
