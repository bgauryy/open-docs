# Claude Agent SDK - Complete Tool System Documentation

## Overview

The Claude Agent SDK provides **17 built-in tools** that enable AI agents to interact with files, execute commands, search code, fetch web content, and manage workflows. These tools form the foundation of Claude Code's capabilities and can be extended with custom MCP (Model Context Protocol) tools.

**SDK Version**: 0.1.22
**Package**: @anthropic-ai/claude-agent-sdk

---

## Table of Contents

1. [Built-in Tools (17 Total)](#built-in-tools)
2. [Tool Input Schemas](#tool-input-schemas)
3. [Permission System](#permission-system)
4. [MCP Tool Integration](#mcp-tool-integration)
5. [Custom Tool Creation](#custom-tool-creation)
6. [Tool Invocation Patterns](#tool-invocation-patterns)

---

## Built-in Tools

The SDK includes 17 built-in tools organized by category:

### File Operations (4 tools)
1. **FileRead** - Read file contents with pagination support
2. **FileWrite** - Write/overwrite files
3. **FileEdit** - Edit files using string replacement
4. **NotebookEdit** - Edit Jupyter notebook cells

### File Discovery (2 tools)
5. **Glob** - Pattern-based file matching
6. **Grep** - Content search using ripgrep

### Command Execution (3 tools)
7. **Bash** - Execute shell commands
8. **BashOutput** - Retrieve output from background shells
9. **KillShell** - Terminate background shell processes

### Web & Search (2 tools)
10. **WebFetch** - Fetch and process web content
11. **WebSearch** - Search the web with domain filtering

### MCP Integration (2 tools)
12. **McpInput** - Generic MCP tool invocation
13. **ListMcpResources** - List available MCP resources
14. **ReadMcpResource** - Read specific MCP resources

### Task Management (1 tool)
15. **TodoWrite** - Manage task lists with status tracking

### Agent Control (2 tools)
16. **Agent** - Invoke specialized subagents
17. **ExitPlanMode** - Submit plans for user approval

---

## Tool Input Schemas

Complete TypeScript definitions for all tool inputs:

### 1. AgentInput

Invokes a specialized subagent for specific tasks.

```typescript
export interface AgentInput {
  /**
   * A short (3-5 word) description of the task
   */
  description: string;

  /**
   * The task for the agent to perform
   */
  prompt: string;

  /**
   * The type of specialized agent to use for this task
   */
  subagent_type: string;
}
```

**Use Cases**:
- Delegating specialized tasks (e.g., code review, testing)
- Complex multi-step workflows requiring focus
- Domain-specific operations

**Permission Requirements**: None specified (controlled by parent agent)

---

### 2. BashInput

Executes shell commands with timeout and background execution support.

```typescript
export interface BashInput {
  /**
   * The command to execute
   */
  command: string;

  /**
   * Optional timeout in milliseconds (max 600000)
   */
  timeout?: number;

  /**
   * Clear, concise description of what this command does in 5-10 words,
   * in active voice. Examples:
   * - Input: ls → Output: List files in current directory
   * - Input: git status → Output: Show working tree status
   * - Input: npm install → Output: Install package dependencies
   * - Input: mkdir foo → Output: Create directory 'foo'
   */
  description?: string;

  /**
   * Set to true to run this command in the background.
   * Use BashOutput to read the output later.
   */
  run_in_background?: boolean;
}
```

**Use Cases**:
- File system operations (ls, cd, mkdir, rm)
- Git operations (commit, push, pull, diff)
- Build commands (npm, yarn, cargo, make)
- Long-running processes (servers, watchers) with `run_in_background: true`

**Permission Requirements**:
- Command execution permissions
- Write access for file-modifying commands
- Network access for download commands

**Constraints**:
- Maximum timeout: 600,000ms (10 minutes)
- Default timeout: 120,000ms (2 minutes)

---

### 3. BashOutputInput

Retrieves output from background shell processes.

```typescript
export interface BashOutputInput {
  /**
   * The ID of the background shell to retrieve output from
   */
  bash_id: string;

  /**
   * Optional regular expression to filter the output lines.
   * Only lines matching this regex will be included in the result.
   * Any lines that do not match will no longer be available to read.
   */
  filter?: string;
}
```

**Use Cases**:
- Monitoring long-running processes
- Checking server logs
- Polling build/test output

**Permission Requirements**: None (read-only operation)

**Notes**:
- Returns only new output since last check
- Filtered lines are consumed and not available in future reads

---

### 4. ExitPlanModeInput

Submits a plan to the user for approval when in planning mode.

```typescript
export interface ExitPlanModeInput {
  /**
   * The plan you came up with, that you want to run by the user
   * for approval. Supports markdown. The plan should be pretty concise.
   */
  plan: string;
}
```

**Use Cases**:
- Complex multi-step operations requiring user approval
- Risk-sensitive changes (deletions, migrations)
- Budget-conscious workflows

**Permission Requirements**: Permission mode must be set to `plan`

**Notes**:
- Markdown formatting supported
- Plan should be concise and actionable
- User can approve, modify, or reject

---

### 5. FileEditInput

Performs exact string replacements in files.

```typescript
export interface FileEditInput {
  /**
   * The absolute path to the file to modify
   */
  file_path: string;

  /**
   * The text to replace
   */
  old_string: string;

  /**
   * The text to replace it with (must be different from old_string)
   */
  new_string: string;

  /**
   * Replace all occurrences of old_string (default false)
   */
  replace_all?: boolean;
}
```

**Use Cases**:
- Precise code edits
- Refactoring (variable/function renames)
- Bug fixes with known patterns

**Permission Requirements**: Write access to file

**Constraints**:
- `old_string` must exist in file
- `old_string` must be unique (unless `replace_all: true`)
- `new_string` must differ from `old_string`
- File must be read before editing (SDK requirement)

**Best Practices**:
- Include context in `old_string` for uniqueness
- Preserve indentation from source
- Use `replace_all: true` for global renames

---

### 6. FileReadInput

Reads file contents with optional pagination.

```typescript
export interface FileReadInput {
  /**
   * The absolute path to the file to read
   */
  file_path: string;

  /**
   * The line number to start reading from.
   * Only provide if the file is too large to read at once.
   */
  offset?: number;

  /**
   * The number of lines to read.
   * Only provide if the file is too large to read at once.
   */
  limit?: number;
}
```

**Use Cases**:
- Reading source code
- Analyzing configuration files
- Viewing logs
- Reading images (multimodal support)
- Reading PDFs (page-by-page)
- Reading Jupyter notebooks

**Permission Requirements**: Read access to file

**Features**:
- Default: 2000 lines from start
- Lines >2000 chars are truncated
- Results use `cat -n` format (line numbers)
- Multimodal: reads images (PNG, JPG, etc.)
- PDF support: extracts text and visuals
- Jupyter notebook support: includes cell outputs

**Best Practices**:
- Use `offset` and `limit` for large files
- Read before editing (required by SDK)

---

### 7. FileWriteInput

Writes content to files, overwriting existing content.

```typescript
export interface FileWriteInput {
  /**
   * The absolute path to the file to write (must be absolute, not relative)
   */
  file_path: string;

  /**
   * The content to write to the file
   */
  content: string;
}
```

**Use Cases**:
- Creating new files
- Overwriting existing files
- Generating code/config files

**Permission Requirements**: Write access to directory

**Constraints**:
- Must read existing files before overwriting (SDK requirement)
- Path must be absolute
- Overwrites all existing content

**Best Practices**:
- Prefer FileEdit for existing files
- Only create new files when necessary
- Avoid creating documentation files proactively

---

### 8. GlobInput

Pattern-based file matching using glob syntax.

```typescript
export interface GlobInput {
  /**
   * The glob pattern to match files against
   */
  pattern: string;

  /**
   * The directory to search in. If not specified, the current working
   * directory will be used. IMPORTANT: Omit this field to use the default
   * directory. DO NOT enter "undefined" or "null" - simply omit it for
   * the default behavior. Must be a valid directory path if provided.
   */
  path?: string;
}
```

**Use Cases**:
- Finding files by pattern (e.g., `**/*.ts`, `src/**/*.test.js`)
- Discovering project structure
- Batch operations on file sets

**Permission Requirements**: Read access to directory

**Patterns**:
- `*` - matches any characters except `/`
- `**` - matches any characters including `/`
- `?` - matches single character
- `[abc]` - matches any character in brackets
- `{a,b}` - matches either pattern

**Best Practices**:
- Results sorted by modification time
- Use specific patterns to reduce results
- Combine with Grep for content filtering

---

### 9. GrepInput

Content search using ripgrep with advanced filtering.

```typescript
export interface GrepInput {
  /**
   * The regular expression pattern to search for in file contents
   */
  pattern: string;

  /**
   * File or directory to search in (rg PATH).
   * Defaults to current working directory.
   */
  path?: string;

  /**
   * Glob pattern to filter files (e.g. "*.js", "*.{ts,tsx}")
   * Maps to rg --glob
   */
  glob?: string;

  /**
   * Output mode: "content" shows matching lines (supports -A/-B/-C context,
   * -n line numbers, head_limit), "files_with_matches" shows file paths
   * (supports head_limit), "count" shows match counts (supports head_limit).
   * Defaults to "files_with_matches".
   */
  output_mode?: "content" | "files_with_matches" | "count";

  /**
   * Number of lines to show before each match (rg -B).
   * Requires output_mode: "content", ignored otherwise.
   */
  "-B"?: number;

  /**
   * Number of lines to show after each match (rg -A).
   * Requires output_mode: "content", ignored otherwise.
   */
  "-A"?: number;

  /**
   * Number of lines to show before and after each match (rg -C).
   * Requires output_mode: "content", ignored otherwise.
   */
  "-C"?: number;

  /**
   * Show line numbers in output (rg -n).
   * Requires output_mode: "content", ignored otherwise.
   */
  "-n"?: boolean;

  /**
   * Case insensitive search (rg -i)
   */
  "-i"?: boolean;

  /**
   * File type to search (rg --type).
   * Common types: js, py, rust, go, java, etc.
   * More efficient than include for standard file types.
   */
  type?: string;

  /**
   * Limit output to first N lines/entries, equivalent to "| head -N".
   * Works across all output modes: content (limits output lines),
   * files_with_matches (limits file paths), count (limits count entries).
   * When unspecified, shows all results from ripgrep.
   */
  head_limit?: number;

  /**
   * Enable multiline mode where . matches newlines and patterns can
   * span lines (rg -U --multiline-dotall). Default: false.
   */
  multiline?: boolean;
}
```

**Use Cases**:
- Finding function/class definitions
- Searching for TODO comments
- Analyzing API usage
- Code pattern analysis

**Permission Requirements**: Read access to search path

**Output Modes**:
1. `files_with_matches` (default) - List matching files only
2. `content` - Show matching lines with optional context
3. `count` - Show match counts per file

**Best Practices**:
- Start with `files_with_matches` to locate files
- Use `content` with context (`-C: 3`) for details
- Apply `type` filter for language-specific searches
- Use `head_limit` to prevent overwhelming output

---

### 10. KillShellInput

Terminates background shell processes.

```typescript
export interface KillShellInput {
  /**
   * The ID of the background shell to kill
   */
  shell_id: string;
}
```

**Use Cases**:
- Stopping runaway processes
- Cleaning up after testing
- Terminating development servers

**Permission Requirements**: Process termination permissions

**Notes**:
- Use shell ID from BashInput's run_in_background
- Cannot be undone

---

### 11. ListMcpResourcesInput

Lists available MCP resources from configured servers.

```typescript
export interface ListMcpResourcesInput {
  /**
   * Optional server name to filter resources by
   */
  server?: string;
}
```

**Use Cases**:
- Discovering available MCP capabilities
- Debugging MCP server connections
- Resource enumeration

**Permission Requirements**: MCP server access

**Notes**:
- Returns resources from all servers if `server` omitted
- Each resource includes server name in response

---

### 12. McpInput

Generic MCP tool invocation (flexible schema).

```typescript
export interface McpInput {
  [k: string]: unknown;
}
```

**Use Cases**:
- Invoking custom MCP tools
- Accessing MCP server capabilities
- Integration with external services

**Permission Requirements**: Depends on specific MCP tool

**Notes**:
- Schema is intentionally flexible
- Actual parameters defined by MCP server
- Tool name typically included in input

---

### 13. NotebookEditInput

Edits Jupyter notebook cells with insert/replace/delete modes.

```typescript
export interface NotebookEditInput {
  /**
   * The absolute path to the Jupyter notebook file to edit
   * (must be absolute, not relative)
   */
  notebook_path: string;

  /**
   * The ID of the cell to edit. When inserting a new cell, the new cell
   * will be inserted after the cell with this ID, or at the beginning
   * if not specified.
   */
  cell_id?: string;

  /**
   * The new source for the cell
   */
  new_source: string;

  /**
   * The type of the cell (code or markdown). If not specified, it defaults
   * to the current cell type. If using edit_mode=insert, this is required.
   */
  cell_type?: "code" | "markdown";

  /**
   * The type of edit to make (replace, insert, delete).
   * Defaults to replace.
   */
  edit_mode?: "replace" | "insert" | "delete";
}
```

**Use Cases**:
- Modifying data analysis notebooks
- Creating tutorial notebooks
- Fixing notebook code/documentation

**Permission Requirements**: Write access to notebook file

**Edit Modes**:
- `replace` (default) - Replace cell content
- `insert` - Add new cell after specified cell_id
- `delete` - Remove cell (cell_id required)

**Best Practices**:
- Read notebook before editing
- Specify `cell_type` when inserting
- Use cell_id for precise targeting

---

### 14. ReadMcpResourceInput

Reads specific resources from MCP servers.

```typescript
export interface ReadMcpResourceInput {
  /**
   * The MCP server name
   */
  server: string;

  /**
   * The resource URI to read
   */
  uri: string;
}
```

**Use Cases**:
- Fetching MCP resource content
- Reading external data sources
- Integration with MCP services

**Permission Requirements**: Read access to MCP server

**Notes**:
- URI format depends on MCP server
- Server must be configured in SDK options

---

### 15. TodoWriteInput

Manages task lists with status tracking.

```typescript
export interface TodoWriteInput {
  /**
   * The updated todo list
   */
  todos: {
    content: string;
    status: "pending" | "in_progress" | "completed";
    activeForm: string;
  }[];
}
```

**Use Cases**:
- Tracking multi-step workflows
- Progress visualization
- Complex task management

**Permission Requirements**: None (in-memory)

**Task States**:
- `pending` - Not yet started
- `in_progress` - Currently working (limit: 1 at a time)
- `completed` - Finished successfully

**Best Practices**:
- Use for 3+ step tasks
- Maintain exactly 1 `in_progress` task
- Complete tasks immediately after finishing
- Remove stale tasks

---

### 16. WebFetchInput

Fetches and processes web content using AI.

```typescript
export interface WebFetchInput {
  /**
   * The URL to fetch content from
   */
  url: string;

  /**
   * The prompt to run on the fetched content
   */
  prompt: string;
}
```

**Use Cases**:
- Extracting documentation
- Analyzing web pages
- Gathering research data

**Permission Requirements**: Network access

**Features**:
- Converts HTML to markdown
- AI-powered content extraction
- 15-minute cache for repeated requests
- Auto-upgrades HTTP to HTTPS

**Constraints**:
- Read-only (no modifications)
- Results may be summarized for large pages

---

### 17. WebSearchInput

Searches the web with domain filtering.

```typescript
export interface WebSearchInput {
  /**
   * The search query to use
   */
  query: string;

  /**
   * Only include search results from these domains
   */
  allowed_domains?: string[];

  /**
   * Never include search results from these domains
   */
  blocked_domains?: string[];
}
```

**Use Cases**:
- Finding current information
- Research beyond knowledge cutoff
- Discovering documentation/tutorials

**Permission Requirements**: Network access

**Features**:
- Domain filtering (allowlist/blocklist)
- US-only availability
- Returns formatted search blocks

**Best Practices**:
- Account for current date in queries
- Use domain filters for trusted sources
- Query should be 2+ characters

---

## Permission System

The SDK implements a sophisticated permission system to control tool usage and ensure user safety.

### Permission Modes

```typescript
export type PermissionMode =
  | 'default'              // Standard prompting
  | 'acceptEdits'          // Auto-approve edits
  | 'bypassPermissions'    // No prompts
  | 'plan';                // Planning mode
```

### Permission Behaviors

```typescript
export type PermissionBehavior =
  | 'allow'   // Execute without prompting
  | 'deny'    // Block execution
  | 'ask';    // Prompt user for approval
```

### CanUseTool Callback

Custom permission logic via callback:

```typescript
export type CanUseTool = (
  toolName: string,
  input: Record<string, unknown>,
  options: {
    /** Signaled if the operation should be aborted */
    signal: AbortSignal;

    /**
     * Suggestions for updating permissions so that the user will not be
     * prompted again for this tool during this session.
     * Typically used for 'always allow' functionality.
     */
    suggestions?: PermissionUpdate[];
  }
) => Promise<PermissionResult>;
```

### Permission Results

```typescript
export type PermissionResult =
  | {
      behavior: 'allow';
      /** Updated tool input to use, if any changes needed */
      updatedInput: Record<string, unknown>;
      /** Permission updates to apply (e.g., 'always allow') */
      updatedPermissions?: PermissionUpdate[];
    }
  | {
      behavior: 'deny';
      /** Reason for denial or guidance for the model */
      message: string;
      /** If true, interrupt execution completely */
      interrupt?: boolean;
    };
```

### Permission Updates

```typescript
export type PermissionUpdate =
  | { type: 'addRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'replaceRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'removeRules'; rules: PermissionRuleValue[]; behavior: PermissionBehavior; destination: PermissionUpdateDestination }
  | { type: 'setMode'; mode: PermissionMode; destination: PermissionUpdateDestination }
  | { type: 'addDirectories'; directories: string[]; destination: PermissionUpdateDestination }
  | { type: 'removeDirectories'; directories: string[]; destination: PermissionUpdateDestination };

export type PermissionUpdateDestination =
  | 'userSettings'      // Global user config
  | 'projectSettings'   // Project-specific config
  | 'localSettings'     // Local workspace config
  | 'session';          // Current session only
```

### Permission Rules

```typescript
export type PermissionRuleValue = {
  toolName: string;
  ruleContent?: string;  // Optional pattern/condition
};
```

### Tool-Specific Permission Requirements

| Tool | Permission Type | Notes |
|------|----------------|-------|
| **FileRead** | Read | Checks file access |
| **FileWrite** | Write | Checks directory access |
| **FileEdit** | Write | Requires prior read |
| **NotebookEdit** | Write | Notebook-specific |
| **Bash** | Execute | Command-dependent |
| **WebFetch** | Network | URL validation |
| **WebSearch** | Network | US-only restriction |
| **Agent** | None | Inherited from parent |
| **Glob** | Read | Directory access |
| **Grep** | Read | Search path access |
| **TodoWrite** | None | In-memory only |
| **BashOutput** | None | Read-only |
| **KillShell** | Execute | Process termination |
| **MCP Tools** | Varies | Server-dependent |

---

## MCP Tool Integration

The SDK supports custom tools via the Model Context Protocol (MCP).

### MCP Server Configuration

```typescript
export type McpServerConfig =
  | McpStdioServerConfig      // External process
  | McpSSEServerConfig        // Server-Sent Events
  | McpHttpServerConfig       // HTTP transport
  | McpSdkServerConfigWithInstance;  // In-process SDK

// Stdio transport (most common)
export type McpStdioServerConfig = {
  type?: 'stdio';
  command: string;
  args?: string[];
  env?: Record<string, string>;
};

// SSE transport
export type McpSSEServerConfig = {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
};

// HTTP transport
export type McpHttpServerConfig = {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
};

// SDK transport (in-process)
export type McpSdkServerConfig = {
  type: 'sdk';
  name: string;
};

export type McpSdkServerConfigWithInstance = McpSdkServerConfig & {
  instance: McpServer;
};
```

### Configuring MCP Servers

Add MCP servers to SDK options:

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const agent = query({
  prompt: "Your task here",
  options: {
    mcpServers: {
      // External stdio server
      'my-external-tool': {
        type: 'stdio',
        command: 'node',
        args: ['./mcp-server.js'],
        env: { API_KEY: process.env.API_KEY }
      },

      // SSE server
      'remote-service': {
        type: 'sse',
        url: 'https://api.example.com/mcp',
        headers: { 'Authorization': 'Bearer token' }
      },

      // HTTP server
      'http-service': {
        type: 'http',
        url: 'https://api.example.com/mcp',
        headers: { 'X-API-Key': 'secret' }
      },

      // SDK server (in-process)
      'custom-tools': createSdkMcpServer({
        name: 'custom-tools',
        version: '1.0.0',
        tools: [/* ... */]
      })
    }
  }
});
```

### MCP Server Status

Check server connection status:

```typescript
export type McpServerStatus = {
  name: string;
  status: 'connected' | 'failed' | 'needs-auth' | 'pending';
  serverInfo?: {
    name: string;
    version: string;
  };
};

// Query server status
const status = await agent.mcpServerStatus();
```

---

## Custom Tool Creation

Create custom tools that run in-process using the SDK.

### Tool Definition

```typescript
type SdkMcpToolDefinition<Schema extends ZodRawShape = ZodRawShape> = {
  name: string;
  description: string;
  inputSchema: Schema;
  handler: (
    args: z.infer<ZodObject<Schema>>,
    extra: unknown
  ) => Promise<CallToolResult>;
};
```

### Creating a Tool

Use the `tool()` helper function:

```typescript
import { tool, createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

// Define the tool
const greetTool = tool(
  'greet',
  'Greets a person by name',
  {
    name: z.string().describe('The name of the person to greet'),
    formal: z.boolean().optional().describe('Use formal greeting')
  },
  async (args, extra) => {
    const greeting = args.formal
      ? `Good day, ${args.name}!`
      : `Hey ${args.name}!`;

    return {
      content: [{
        type: 'text',
        text: greeting
      }]
    };
  }
);
```

### Implementation (from sdk.mjs)

```javascript
function tool(name, description, inputSchema, handler) {
  return {
    name,
    description,
    inputSchema,
    handler
  };
}
```

### Creating an MCP Server

Bundle tools into an MCP server:

```typescript
import { createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk';

const customServer = createSdkMcpServer({
  name: 'my-custom-tools',
  version: '1.0.0',
  tools: [
    greetTool,
    // ... more tools
  ]
});
```

### Implementation (from sdk.mjs)

```javascript
function createSdkMcpServer(options) {
  const server = new McpServer(
    {
      name: options.name,
      version: options.version ?? "1.0.0"
    },
    {
      capabilities: {
        tools: options.tools ? {} : undefined
      }
    }
  );

  if (options.tools) {
    options.tools.forEach((toolDef) => {
      server.tool(
        toolDef.name,
        toolDef.description,
        toolDef.inputSchema,
        toolDef.handler
      );
    });
  }

  return {
    type: "sdk",
    name: options.name,
    instance: server
  };
}
```

### Complete Example

```typescript
import {
  query,
  tool,
  createSdkMcpServer
} from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

// 1. Define custom tools
const calculateTool = tool(
  'calculate',
  'Performs basic arithmetic calculations',
  {
    operation: z.enum(['add', 'subtract', 'multiply', 'divide'])
      .describe('The arithmetic operation to perform'),
    a: z.number().describe('First operand'),
    b: z.number().describe('Second operand')
  },
  async (args) => {
    let result: number;
    switch (args.operation) {
      case 'add': result = args.a + args.b; break;
      case 'subtract': result = args.a - args.b; break;
      case 'multiply': result = args.a * args.b; break;
      case 'divide':
        if (args.b === 0) {
          return {
            content: [{ type: 'text', text: 'Error: Division by zero' }],
            isError: true
          };
        }
        result = args.a / args.b;
        break;
    }

    return {
      content: [{
        type: 'text',
        text: `Result: ${result}`
      }]
    };
  }
);

const weatherTool = tool(
  'get_weather',
  'Gets weather information for a city',
  {
    city: z.string().describe('City name'),
    units: z.enum(['celsius', 'fahrenheit']).optional()
      .describe('Temperature units')
  },
  async (args) => {
    // In real implementation, call weather API
    return {
      content: [{
        type: 'text',
        text: `Weather in ${args.city}: Sunny, 72°${args.units === 'celsius' ? 'C' : 'F'}`
      }]
    };
  }
);

// 2. Create MCP server with tools
const customTools = createSdkMcpServer({
  name: 'my-custom-tools',
  version: '1.0.0',
  tools: [calculateTool, weatherTool]
});

// 3. Use in agent
const agent = query({
  prompt: "What's 42 + 58? Also check the weather in San Francisco.",
  options: {
    mcpServers: {
      'custom': customTools
    }
  }
});

// 4. Process results
for await (const message of agent) {
  console.log(message);
}
```

### Tool Handler Return Type

```typescript
import { CallToolResult } from '@modelcontextprotocol/sdk/types.js';

// Return type for tool handlers
type CallToolResult = {
  content: Array<{
    type: 'text' | 'image' | 'resource';
    text?: string;
    data?: string;
    mimeType?: string;
  }>;
  isError?: boolean;
};
```

### Best Practices for Custom Tools

1. **Schema Design**:
   - Use descriptive field names
   - Include `.describe()` for all fields
   - Mark optional fields with `.optional()`
   - Validate inputs thoroughly

2. **Error Handling**:
   - Return `{ isError: true }` for errors
   - Provide helpful error messages
   - Handle edge cases gracefully

3. **Performance**:
   - Keep tools focused and single-purpose
   - Use async/await for I/O operations
   - Consider timeout constraints

4. **Documentation**:
   - Write clear tool descriptions
   - Document expected behavior
   - Include usage examples

5. **Testing**:
   - Test with various inputs
   - Handle invalid data gracefully
   - Verify error conditions

---

## Tool Invocation Patterns

### Basic Tool Usage

Tools are automatically available to the agent based on configuration:

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const agent = query({
  prompt: "Read the README.md file and summarize it",
  options: {
    cwd: '/path/to/project'
  }
});

for await (const message of agent) {
  if (message.type === 'assistant') {
    // Agent may use FileRead tool automatically
    console.log(message);
  }
}
```

### Controlling Tool Access

Restrict or allow specific tools:

```typescript
const agent = query({
  prompt: "Analyze this codebase",
  options: {
    // Only allow read-only tools
    allowedTools: [
      'FileRead',
      'Glob',
      'Grep',
      'Bash'  // Still need to control via canUseTool
    ],

    // Or block dangerous tools
    disallowedTools: [
      'FileWrite',
      'FileEdit',
      'KillShell'
    ]
  }
});
```

### Custom Permission Logic

Implement fine-grained control with `canUseTool`:

```typescript
const agent = query({
  prompt: "Refactor the codebase",
  options: {
    canUseTool: async (toolName, input, { signal, suggestions }) => {
      // Example: Require approval for file writes
      if (toolName === 'FileWrite') {
        const filePath = input.file_path as string;

        // Auto-allow writes to specific directories
        if (filePath.startsWith('/tmp/')) {
          return {
            behavior: 'allow',
            updatedInput: input
          };
        }

        // Deny writes to sensitive files
        if (filePath.includes('.env') || filePath.includes('secrets')) {
          return {
            behavior: 'deny',
            message: 'Cannot write to sensitive files',
            interrupt: true
          };
        }

        // Prompt for other files
        const userApproval = await askUser(
          `Allow write to ${filePath}?`,
          ['Yes', 'No', 'Always Allow']
        );

        if (userApproval === 'Always Allow') {
          return {
            behavior: 'allow',
            updatedInput: input,
            updatedPermissions: suggestions  // Apply 'always allow'
          };
        }

        return {
          behavior: userApproval === 'Yes' ? 'allow' : 'deny',
          updatedInput: input,
          message: userApproval === 'No'
            ? 'User declined file write'
            : undefined
        };
      }

      // Example: Validate bash commands
      if (toolName === 'Bash') {
        const command = input.command as string;

        // Block dangerous commands
        const dangerousPatterns = [
          /rm\s+-rf\s+\//,
          /sudo\s+/,
          /chmod\s+777/
        ];

        for (const pattern of dangerousPatterns) {
          if (pattern.test(command)) {
            return {
              behavior: 'deny',
              message: `Dangerous command blocked: ${command}`,
              interrupt: false
            };
          }
        }
      }

      // Default: allow
      return {
        behavior: 'allow',
        updatedInput: input
      };
    }
  }
});
```

### Monitoring Tool Usage

Track tool invocations with hooks:

```typescript
const agent = query({
  prompt: "Build and test the project",
  options: {
    hooks: {
      PreToolUse: [{
        hooks: [async (input, toolUseID, { signal }) => {
          const hookInput = input as PreToolUseHookInput;
          console.log(`About to use: ${hookInput.tool_name}`);
          console.log(`Input:`, hookInput.tool_input);

          return { continue: true };
        }]
      }],

      PostToolUse: [{
        hooks: [async (input, toolUseID, { signal }) => {
          const hookInput = input as PostToolUseHookInput;
          console.log(`Completed: ${hookInput.tool_name}`);
          console.log(`Result:`, hookInput.tool_response);

          return { continue: true };
        }]
      }]
    }
  }
});
```

### Handling Tool Errors

```typescript
for await (const message of agent) {
  if (message.type === 'result') {
    if (message.is_error) {
      console.error('Agent encountered an error');
      console.error('Subtype:', message.subtype);

      // Check for permission denials
      if (message.permission_denials.length > 0) {
        console.error('Permission denials:', message.permission_denials);
      }
    } else {
      console.log('Success!');
      console.log('Result:', message.result);
      console.log('Cost:', message.total_cost_usd);
      console.log('Usage:', message.usage);
    }
  }
}
```

### Dynamic Tool Control

Change tool behavior during execution:

```typescript
const agent = query({
  prompt: "Long-running task",
  options: {}
});

// Start processing
(async () => {
  for await (const message of agent) {
    console.log(message);

    // After some condition, change permission mode
    if (message.type === 'assistant' && needsApproval(message)) {
      await agent.setPermissionMode('plan');
    }

    // Or change the model
    if (message.type === 'result' && message.is_error) {
      await agent.setModel('claude-opus-4-20250514');
    }
  }
})();
```

---

## Additional SDK Types

### Message Types

```typescript
export type SDKMessage =
  | SDKAssistantMessage      // Claude's responses
  | SDKUserMessage           // User inputs
  | SDKUserMessageReplay     // Replayed messages
  | SDKResultMessage         // Final results
  | SDKSystemMessage         // System info
  | SDKPartialAssistantMessage  // Streaming chunks
  | SDKCompactBoundaryMessage   // Compaction markers
  | SDKHookResponseMessage;     // Hook outputs

export type SDKUserMessage = {
  type: 'user';
  message: APIUserMessage;
  parent_tool_use_id: string | null;
  isSynthetic?: boolean;  // Not from user directly
  uuid?: UUID;
  session_id: string;
};

export type SDKAssistantMessage = {
  type: 'assistant';
  message: APIAssistantMessage;
  parent_tool_use_id: string | null;
  uuid: UUID;
  session_id: string;
};

export type SDKResultMessage = {
  type: 'result';
  subtype: 'success' | 'error_max_turns' | 'error_during_execution';
  duration_ms: number;
  duration_api_ms: number;
  is_error: boolean;
  num_turns: number;
  total_cost_usd: number;
  usage: NonNullableUsage;
  modelUsage: { [modelName: string]: ModelUsage };
  permission_denials: SDKPermissionDenial[];
  result?: string;  // Only on success
  uuid: UUID;
  session_id: string;
};
```

### Query Options

```typescript
export type Options = {
  abortController?: AbortController;
  additionalDirectories?: string[];
  allowedTools?: string[];
  appendSystemPrompt?: string;
  canUseTool?: CanUseTool;
  continue?: boolean;
  customSystemPrompt?: string;
  cwd?: string;
  disallowedTools?: string[];
  env?: { [envVar: string]: string | undefined };
  executable?: 'bun' | 'deno' | 'node';
  executableArgs?: string[];
  extraArgs?: Record<string, string | null>;
  fallbackModel?: string;
  forkSession?: boolean;
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;
  includePartialMessages?: boolean;
  maxThinkingTokens?: number;
  maxTurns?: number;
  mcpServers?: Record<string, McpServerConfig>;
  model?: string;
  pathToClaudeCodeExecutable?: string;
  permissionMode?: PermissionMode;
  permissionPromptToolName?: string;
  resume?: string;
  resumeSessionAt?: string;
  stderr?: (data: string) => void;
  strictMcpConfig?: boolean;
};
```

### Hook Events

```typescript
export const HOOK_EVENTS = [
  "PreToolUse",        // Before tool execution
  "PostToolUse",       // After tool execution
  "Notification",      // System notifications
  "UserPromptSubmit",  // User submits prompt
  "SessionStart",      // Session begins
  "SessionEnd",        // Session ends
  "Stop",             // Agent stops
  "SubagentStop",     // Subagent stops
  "PreCompact"        // Before context compaction
] as const;

export type HookEvent = typeof HOOK_EVENTS[number];
```

---

## Resources

- **Official Documentation**: https://docs.claude.com/en/api/agent-sdk/overview
- **Migration Guide**: https://docs.claude.com/en/docs/claude-code/sdk/migration-guide
- **GitHub Repository**: https://github.com/anthropics/claude-agent-sdk-typescript
- **Discord Community**: https://anthropic.com/discord
- **Data Usage Policies**: https://docs.anthropic.com/en/docs/claude-code/data-usage

---

## Summary

The Claude Agent SDK provides:

- **17 built-in tools** covering files, commands, web, search, and task management
- **Complete TypeScript schemas** with detailed parameter documentation
- **Flexible permission system** with multiple modes and custom logic
- **MCP integration** supporting stdio, SSE, HTTP, and in-process servers
- **Custom tool creation** using simple `tool()` and `createSdkMcpServer()` APIs
- **Rich type system** for safe and efficient agent development

This tool system enables powerful autonomous agents while maintaining user control and safety through comprehensive permission management.
