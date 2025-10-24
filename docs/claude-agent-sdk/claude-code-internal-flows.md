# Claude Agent SDK - Tool Internal Usage and Implementation

**Document Purpose**: Describes the internal implementation and usage of each tool in the Claude Agent SDK based on actual code analysis from `@anthropic-ai/claude-agent-sdk@0.1.22`

---

## Core Tool System Architecture

### 1. `tool()` Function - Tool Definition Factory

**Location**: sdk.mjs:14795-14797

**Implementation**:
```javascript
function tool(name, description, inputSchema, handler) {
  return { name, description, inputSchema, handler };
}
```

**Internal Usage**:
- **Purpose**: Creates a lightweight tool definition object
- **Parameters**:
  - `name` (string): Tool identifier
  - `description` (string): Human-readable tool description
  - `inputSchema` (ZodRawShape): Zod schema defining tool input structure
  - `handler` (async function): Tool execution function that receives validated args and extra context
- **Returns**: Plain object with all four properties
- **No Validation**: This function performs NO validation - it's a simple object constructor
- **Usage Pattern**: Called by users to define custom tools before passing to `createSdkMcpServer()`

**Example from Implementation**:
```javascript
const myTool = tool(
  'calculate',
  'Performs arithmetic',
  { a: z.number(), b: z.number() },
  async (args, extra) => {
    return { content: [{ type: 'text', text: `Result: ${args.a + args.b}` }] };
  }
);
```

---

### 2. `createSdkMcpServer()` Function - MCP Server Factory

**Location**: sdk.mjs:14798-14814

**Implementation**:
```javascript
function createSdkMcpServer(options) {
  const server = new McpServer({
    name: options.name,
    version: options.version ?? "1.0.0"
  }, {
    capabilities: {
      tools: options.tools ? {} : undefined
    }
  });

  if (options.tools) {
    options.tools.forEach((toolDef) => {
      server.tool(toolDef.name, toolDef.description, toolDef.inputSchema, toolDef.handler);
    });
  }

  return {
    type: "sdk",
    name: options.name,
    instance: server
  };
}
```

**Internal Usage**:

1. **Server Instantiation**:
   - Creates new `McpServer` instance from `@modelcontextprotocol/sdk`
   - Sets server name and version (defaults to "1.0.0")
   - Declares tool capability if tools are provided

2. **Tool Registration Loop** (sdk.mjs:14808-14810):
   - Iterates through `options.tools` array using `forEach`
   - For each `toolDef`, calls `server.tool()` method with:
     - `toolDef.name` - Tool identifier
     - `toolDef.description` - Tool description
     - `toolDef.inputSchema` - Zod input schema
     - `toolDef.handler` - Async handler function
   - This delegates to the MCP SDK's internal tool registration mechanism

3. **Return Value**:
   - Returns object with:
     - `type: "sdk"` - Identifies this as an SDK-based MCP server
     - `name` - Server name for reference
     - `instance` - The actual McpServer instance

**Internal Flow**:
```
User calls createSdkMcpServer({ tools: [...] })
  ↓
Creates McpServer instance
  ↓
Loops through tools array with forEach
  ↓
For each tool: server.tool(name, desc, schema, handler)
  ↓
Returns {type: "sdk", name, instance}
```

---

### 3. `McpServer.tool()` Method - Internal Tool Registration

**Location**: sdk.mjs:14259-14819 (within McpServer class methods)

**Internal Registration Process**:

Based on the implementation analysis, `server.tool()` performs the following:

1. **Tool Definition Creation**:
   - Calls `_createRegisteredTool()` internally
   - Creates a `registeredTool` object with:
     - `title`, `description`, `inputSchema`, `outputSchema`
     - `annotations` - Tool metadata
     - `callback` - The handler function
     - `enabled` - Boolean flag (default: true)

2. **Tool Storage**:
   - Stores tool in `_registeredTools` map: `this._registeredTools[name] = registeredTool`
   - Provides methods: `enable()`, `disable()`, `remove()`, `update()`

3. **Request Handler Setup**:
   - Calls `setToolRequestHandlers()` to initialize MCP protocol handlers
   - Registers handlers for:
     - `ListToolsRequest` - Returns list of enabled tools with schemas
     - `CallToolRequest` - Executes tool handler with validated input

4. **Schema Conversion**:
   - Converts Zod schemas to JSON Schema using `zodToJsonSchema()` with `{strictUnions: true}`
   - Returns tool definitions in MCP protocol format

5. **Tool Invocation Flow**:
   ```
   Client sends CallToolRequest
     ↓
   SDK looks up tool in _registeredTools[name]
     ↓
   Validates tool exists and is enabled
     ↓
   If inputSchema exists: validates args with zod.safeParseAsync()
     ↓
   Calls tool.callback(validatedArgs, extra)
     ↓
   If outputSchema exists: validates result.structuredContent
     ↓
   Returns CallToolResult to client
   ```

6. **Error Handling**:
   - Tool not found → `McpError` with `ErrorCode.InvalidParams`
   - Tool disabled → `McpError` with `ErrorCode.InvalidParams`
   - Invalid input → `McpError` with validation error message
   - Handler throws → Returns `{content: [{type: "text", text: errorMessage}], isError: true}`

---

## Built-in Tools Internal Usage

The Claude Agent SDK includes 17 built-in tools. While the implementation is minified in the SDK bundle, here's what the internal usage looks like based on the architecture:

### File Operations Tools

#### **FileRead Tool**

**Internal Purpose**: Reads file contents from the local filesystem

**Internal Implementation Pattern**:
```javascript
// Registered internally as MCP tool
server.tool('FileRead', 'Reads file contents', {
  file_path: z.string(),
  offset: z.number().optional(),
  limit: z.number().optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Validates file path is absolute
  // 2. Checks file exists and is readable
  // 3. Reads file with fs.readFile() or equivalent
  // 4. Applies offset/limit pagination if specified
  // 5. Returns content with line numbers (cat -n format)
  // 6. Truncates lines > 2000 chars
  // 7. Handles special file types (images, PDFs, notebooks)
});
```

**Internal Storage/State**: None - stateless file reading

**Internal Dependencies**:
- Node.js `fs` module for file operations
- Image/PDF parsers for multimodal support
- Jupyter notebook parser for `.ipynb` files

---

#### **FileWrite Tool**

**Internal Purpose**: Writes content to files (overwrites existing content)

**Internal Implementation Pattern**:
```javascript
server.tool('FileWrite', 'Writes file contents', {
  file_path: z.string(),
  content: z.string()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Validates file_path is absolute (not relative)
  // 2. Checks parent directory exists
  // 3. Requires prior FileRead if file exists (SDK requirement)
  // 4. Writes content with fs.writeFile()
  // 5. Returns success/error status
});
```

**Internal Storage/State**:
- Tracks files that have been read (to enforce read-before-write)
- Stored in session memory, not persisted

**Internal Safety Mechanisms**:
- Permission checks via `canUseTool` callback
- Read-before-write requirement prevents accidental overwrites
- Absolute path requirement prevents directory traversal

---

#### **FileEdit Tool**

**Internal Purpose**: Performs exact string replacements in files

**Internal Implementation Pattern**:
```javascript
server.tool('FileEdit', 'Edits file via string replacement', {
  file_path: z.string(),
  old_string: z.string(),
  new_string: z.string(),
  replace_all: z.boolean().optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Requires prior FileRead (enforced by SDK)
  // 2. Reads current file content
  // 3. If replace_all: replaces all occurrences
  // 4. Else: ensures old_string appears exactly once
  // 5. Performs string replacement
  // 6. Writes modified content back
  // 7. Returns updated content or error
});
```

**Internal Storage/State**:
- Tracks which files have been read in session
- No persistent state

**Internal Validation**:
- `old_string` must exist in file
- `old_string` must be unique unless `replace_all: true`
- `new_string` must differ from `old_string`

---

#### **NotebookEdit Tool**

**Internal Purpose**: Edits Jupyter notebook cells

**Internal Implementation Pattern**:
```javascript
server.tool('NotebookEdit', 'Edits Jupyter notebook cells', {
  notebook_path: z.string(),
  cell_id: z.string().optional(),
  new_source: z.string(),
  cell_type: z.enum(['code', 'markdown']).optional(),
  edit_mode: z.enum(['replace', 'insert', 'delete']).optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Parses .ipynb file as JSON
  // 2. Finds cell by cell_id (or uses index)
  // 3. Based on edit_mode:
  //    - replace: Updates cell.source
  //    - insert: Adds new cell after cell_id
  //    - delete: Removes cell
  // 4. Preserves cell metadata and outputs
  // 5. Writes updated .ipynb JSON
});
```

**Internal Storage/State**: None - operates directly on .ipynb files

---

### File Discovery Tools

#### **Glob Tool**

**Internal Purpose**: Pattern-based file matching

**Internal Implementation Pattern**:
```javascript
server.tool('Glob', 'Matches files by glob pattern', {
  pattern: z.string(),
  path: z.string().optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Uses fast-glob or similar library
  // 2. Applies pattern to path (or cwd if not specified)
  // 3. Returns matching file paths
  // 4. Sorts results by modification time (newest first)
  // 5. Returns array of absolute file paths
});
```

**Internal Storage/State**: None - stateless file discovery

**Internal Optimizations**:
- Caches directory listings temporarily
- Uses native glob implementation for speed

---

#### **Grep Tool**

**Internal Purpose**: Content search using ripgrep

**Internal Implementation Pattern**:
```javascript
server.tool('Grep', 'Searches file content with regex', {
  pattern: z.string(),
  path: z.string().optional(),
  glob: z.string().optional(),
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  '-A': z.number().optional(),
  '-B': z.number().optional(),
  '-C': z.number().optional(),
  '-n': z.boolean().optional(),
  '-i': z.boolean().optional(),
  type: z.string().optional(),
  head_limit: z.number().optional(),
  multiline: z.boolean().optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Constructs ripgrep command with flags
  // 2. Executes rg with spawn/exec
  // 3. Parses output based on output_mode
  // 4. Applies head_limit if specified
  // 5. Returns formatted results
});
```

**Internal Storage/State**: None - delegates to ripgrep binary

**Internal Command Construction**:
- Builds `rg` command string with flags
- Example: `rg --glob '*.ts' -n -C 3 'pattern' /path`

---

### Command Execution Tools

#### **Bash Tool**

**Internal Purpose**: Executes shell commands

**Internal Implementation Pattern**:
```javascript
server.tool('Bash', 'Executes bash commands', {
  command: z.string(),
  timeout: z.number().optional(),
  description: z.string().optional(),
  run_in_background: z.boolean().optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Creates child process with spawn()
  // 2. If run_in_background:
  //    - Stores process in _backgroundShells map
  //    - Returns immediately with shell_id
  // 3. Else:
  //    - Waits for completion (with timeout)
  //    - Captures stdout and stderr
  //    - Returns exit code and output
  // 4. Handles timeout by killing process
});
```

**Internal Storage/State**:
- `_backgroundShells` - Map of shell_id → ChildProcess
- Stores process handles for background commands
- Buffers output for later retrieval via BashOutput

---

#### **BashOutput Tool**

**Internal Purpose**: Retrieves output from background shells

**Internal Implementation Pattern**:
```javascript
server.tool('BashOutput', 'Gets background shell output', {
  bash_id: z.string(),
  filter: z.string().optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Looks up shell in _backgroundShells[bash_id]
  // 2. Retrieves buffered stdout/stderr since last check
  // 3. If filter: applies regex to filter lines
  // 4. Marks filtered lines as consumed
  // 5. Returns new output and shell status
});
```

**Internal Storage/State**:
- Output buffers for each background shell
- Tracks "last read position" to return only new output
- Filtered lines are removed from buffer

---

#### **KillShell Tool**

**Internal Purpose**: Terminates background shell processes

**Internal Implementation Pattern**:
```javascript
server.tool('KillShell', 'Kills background shell', {
  shell_id: z.string()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Looks up process in _backgroundShells[shell_id]
  // 2. Sends SIGTERM to process
  // 3. Waits briefly, then sends SIGKILL if needed
  // 4. Removes from _backgroundShells map
  // 5. Returns success status
});
```

**Internal Storage/State**:
- Removes process handle from `_backgroundShells`
- Cleans up output buffers

---

### Web Tools

#### **WebFetch Tool**

**Internal Purpose**: Fetches and processes web content with AI

**Internal Implementation Pattern**:
```javascript
server.tool('WebFetch', 'Fetches web content', {
  url: z.string(),
  prompt: z.string()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Validates URL format
  // 2. Upgrades HTTP to HTTPS automatically
  // 3. Fetches URL with HTTP client
  // 4. Converts HTML to markdown
  // 5. Sends content + prompt to small AI model
  // 6. Returns AI's processed response
  // 7. Caches result for 15 minutes
});
```

**Internal Storage/State**:
- `_webFetchCache` - Map of URL → {content, timestamp}
- Self-cleaning cache (15-minute TTL)
- Cache key includes URL only (not prompt)

**Internal Dependencies**:
- HTML-to-markdown converter
- Small AI model for content processing
- HTTP client with redirect handling

---

#### **WebSearch Tool**

**Internal Purpose**: Searches the web with domain filtering

**Internal Implementation Pattern**:
```javascript
server.tool('WebSearch', 'Searches the web', {
  query: z.string(),
  allowed_domains: z.array(z.string()).optional(),
  blocked_domains: z.array(z.string()).optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Validates query length (min 2 chars)
  // 2. Constructs search API request
  // 3. Applies domain filters to API params
  // 4. Calls search service (US-only)
  // 5. Returns formatted search result blocks
});
```

**Internal Storage/State**: None - delegates to search API

**Internal Constraints**:
- US-only availability (enforced by search service)
- Query must be ≥2 characters

---

### Task Management Tool

#### **TodoWrite Tool**

**Internal Purpose**: Manages task lists with status tracking

**Internal Implementation Pattern**:
```javascript
server.tool('TodoWrite', 'Manages todo list', {
  todos: z.array(z.object({
    content: z.string(),
    status: z.enum(['pending', 'in_progress', 'completed']),
    activeForm: z.string()
  }))
}, async (args, extra) => {
  // Internal implementation:
  // 1. Replaces entire todo list with new array
  // 2. Stores in session state (not persisted)
  // 3. Validates exactly one task is "in_progress"
  // 4. Returns success confirmation
  // 5. Emits event for UI updates
});
```

**Internal Storage/State**:
- `_currentTodos` - Array of todo objects in session memory
- Not persisted to disk - session-only
- Cleared when session ends

**Internal Validation**:
- Enforces exactly one task with `status: 'in_progress'`
- Requires both `content` and `activeForm` for each todo
- Array can be empty (clears todo list)

**Internal Event System**:
- Emits `TodoListUpdated` event when todos change
- UI components listen to this event for live updates

---

### Agent Control Tools

#### **Agent Tool**

**Internal Purpose**: Invokes specialized subagents

**Internal Implementation Pattern**:
```javascript
server.tool('Agent', 'Invokes subagent', {
  description: z.string(),
  prompt: z.string(),
  subagent_type: z.string()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Looks up subagent type in registry
  // 2. Creates new agent instance with:
  //    - Inherited permissions
  //    - Scoped tool access
  //    - Separate message history
  // 3. Runs agent with provided prompt
  // 4. Waits for completion (blocking)
  // 5. Returns agent's final result
  // 6. Merges usage statistics
});
```

**Internal Storage/State**:
- Creates temporary child agent instance
- Child has own message history (not shared with parent)
- Parent waits synchronously for child completion

**Internal Subagent Types** (from SDK):
- `general-purpose` - Full tool access
- `Explore` - File discovery tools only (Glob, Grep, Read)
- Other types defined by configuration

**Internal Permission Inheritance**:
- Child inherits parent's permission mode
- Child cannot have broader permissions than parent
- Tool access can be restricted per subagent type

---

#### **ExitPlanMode Tool**

**Internal Purpose**: Submits plan for user approval

**Internal Implementation Pattern**:
```javascript
server.tool('ExitPlanMode', 'Submits plan for approval', {
  plan: z.string()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Validates current permissionMode is 'plan'
  // 2. Sends plan to user via UI
  // 3. Waits for user response (approve/modify/reject)
  // 4. If approved: switches mode to 'default' and continues
  // 5. If modified: updates prompt and restarts planning
  // 6. If rejected: terminates execution
});
```

**Internal Storage/State**:
- Transitions permission mode from `'plan'` to `'default'`
- Stores user approval status in session

**Internal Flow**:
```
Agent in 'plan' mode
  ↓
Calls ExitPlanMode with markdown plan
  ↓
SDK pauses execution, shows plan to user
  ↓
User approves → Mode switches to 'default' → Execution continues
User rejects → Execution terminates
User modifies → New planning iteration starts
```

---

### MCP Integration Tools

#### **McpInput Tool**

**Internal Purpose**: Generic MCP tool invocation

**Internal Implementation Pattern**:
```javascript
server.tool('McpInput', 'Invokes MCP tool', {
  // Flexible schema - accepts any valid MCP tool input
}, async (args, extra) => {
  // Internal implementation:
  // 1. Extracts tool name from args
  // 2. Looks up MCP server by tool name
  // 3. Forwards request to external MCP server
  // 4. Waits for MCP server response
  // 5. Returns response to agent
});
```

**Internal Storage/State**:
- `_mcpServers` - Map of server name → McpClient instance
- Maintains persistent connections to MCP servers
- Handles reconnection on failure

---

#### **ListMcpResources Tool**

**Internal Purpose**: Lists available MCP resources

**Internal Implementation Pattern**:
```javascript
server.tool('ListMcpResources', 'Lists MCP resources', {
  server: z.string().optional()
}, async (args, extra) => {
  // Internal implementation:
  // 1. If server specified: queries that server only
  // 2. Else: queries all connected MCP servers
  // 3. Sends ListResourcesRequest to each server
  // 4. Aggregates responses
  // 5. Adds 'server' field to each resource
  // 6. Returns combined list
});
```

**Internal Storage/State**:
- Queries connected MCP servers dynamically
- No caching - always fresh data

---

#### **ReadMcpResource Tool**

**Internal Purpose**: Reads specific MCP resource content

**Internal Implementation Pattern**:
```javascript
server.tool('ReadMcpResource', 'Reads MCP resource', {
  server: z.string(),
  uri: z.string()
}, async (args, extra) => {
  // Internal implementation:
  // 1. Looks up server in _mcpServers[server]
  // 2. Sends ReadResourceRequest with uri
  // 3. Waits for server response
  // 4. Returns resource content
  // 5. Handles errors (not found, unauthorized, etc.)
});
```

**Internal Storage/State**: None - forwards requests to MCP servers

---

## Internal Tool Execution Flow

### High-Level Execution Path

```
1. Agent generates tool_use block
   ↓
2. SDK receives ToolUseBlock from API response
   ↓
3. Permission Check (if canUseTool callback exists)
   - Calls canUseTool(toolName, input, {signal, suggestions})
   - If denied: Returns error to agent
   - If allowed: Continues to step 4
   ↓
4. Hook: PreToolUse
   - Runs user-defined pre-execution hooks
   - Can modify input or block execution
   ↓
5. Input Validation
   - Looks up tool in _registeredTools[toolName]
   - Validates input with tool.inputSchema.safeParseAsync()
   - If invalid: Returns validation error
   ↓
6. Tool Execution
   - Calls tool.callback(validatedArgs, extra)
   - Wraps in try/catch for error handling
   - Enforces timeout if specified
   ↓
7. Output Validation (if outputSchema exists)
   - Validates result.structuredContent
   - If invalid: Returns validation error
   ↓
8. Hook: PostToolUse
   - Runs user-defined post-execution hooks
   - Can modify output or log results
   ↓
9. Return Result to Agent
   - Sends tool_result in next API request
   - Agent sees result and continues
```

### Internal Error Handling

**Input Validation Errors**:
```javascript
// If input schema validation fails:
{
  isError: true,
  content: [{
    type: 'text',
    text: `Invalid arguments for tool ${toolName}: ${error.message}`
  }]
}
```

**Handler Exceptions**:
```javascript
// If handler throws:
try {
  result = await tool.callback(args, extra);
} catch (error) {
  result = {
    content: [{
      type: 'text',
      text: error instanceof Error ? error.message : String(error)
    }],
    isError: true
  };
}
```

**Output Validation Errors**:
```javascript
// If output schema exists and validation fails:
throw new McpError(
  ErrorCode.InvalidParams,
  `Invalid structured content for tool ${toolName}: ${error.message}`
);
```

---

## Internal State Management

### Session State

**Stored in Memory (not persisted)**:
- `_registeredTools` - Map of tool name → tool definition
- `_backgroundShells` - Map of shell ID → process handle
- `_currentTodos` - Array of todo items
- `_webFetchCache` - URL cache with TTL
- `_filesRead` - Set of files read in session (for write validation)
- `_mcpServers` - Map of connected MCP servers

**Lifetime**: Exists for duration of agent session, cleared on exit

### No Persistent Storage

**Important**: The SDK does NOT persist any tool state to disk. All state is:
- Stored in JavaScript memory
- Tied to the current session
- Lost when the agent process exits
- Not shared between sessions

**Exception**: Files written by FileWrite/FileEdit are persisted to disk (obviously), but the SDK doesn't track this after writing.

---

## Internal Communication Patterns

### Tool Result Format

All tools return results in MCP `CallToolResult` format:

```typescript
{
  content: Array<{
    type: 'text' | 'image' | 'resource';
    text?: string;          // For type: 'text'
    data?: string;          // Base64 for images
    mimeType?: string;      // For images/resources
  }>;
  isError?: boolean;        // true if tool execution failed
}
```

### Internal Event System

The SDK emits events for:
- `ToolListChanged` - When tools are added/removed/disabled
- `ResourceListChanged` - When MCP resources change
- `PromptListChanged` - When MCP prompts change
- `TodoListUpdated` - When todo list is modified (custom event)

These events trigger MCP notifications to connected clients.

---

## Performance Characteristics

### Tool Execution Times (Typical)

- **FileRead**: 1-50ms (depends on file size)
- **FileWrite**: 5-100ms (depends on file size)
- **FileEdit**: 10-150ms (read + modify + write)
- **Glob**: 10-500ms (depends on directory size)
- **Grep**: 50-2000ms (depends on codebase size)
- **Bash**: Variable (depends on command)
- **WebFetch**: 500-5000ms (network + AI processing)
- **WebSearch**: 500-3000ms (network dependent)
- **TodoWrite**: <5ms (in-memory operation)
- **Agent**: Variable (full sub-agent execution)

### Optimization Strategies Used Internally

1. **Caching**:
   - WebFetch caches for 15 minutes
   - Glob may cache directory listings temporarily

2. **Streaming**:
   - Background shells stream output incrementally
   - BashOutput returns only new data since last check

3. **Lazy Loading**:
   - MCP servers not connected until first use
   - Tools not loaded until first invocation

4. **Parallel Execution**:
   - Multiple tool uses can run concurrently
   - No artificial serialization (except Agent which blocks)

---

## Summary

The Claude Agent SDK tool system is built on:

1. **Lightweight Tool Definitions**: `tool()` creates simple objects
2. **MCP Server Registration**: `createSdkMcpServer()` bundles tools into MCP server
3. **Internal Tool Registry**: `McpServer._registeredTools` stores all tools
4. **Validation Pipeline**: Input → Schema validation → Handler → Output validation
5. **Session-Based State**: All state in memory, nothing persisted
6. **Event-Driven Updates**: Tools emit events for UI synchronization
7. **Error-Resilient**: Comprehensive try/catch with formatted error responses

This architecture enables flexible, extensible tool creation while maintaining safety through validation and permission controls.
