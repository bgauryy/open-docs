# Claude Agent SDK - Comprehensive Guide

**Package**: `@anthropic-ai/claude-agent-sdk`
**Version**: 0.1.22
**Repository**: https://github.com/anthropics/claude-agent-sdk-typescript
**Documentation**: https://docs.claude.com/en/api/agent-sdk/overview

---

## Table of Contents

1. [Overview](#overview)
2. [Getting Started](#getting-started)
3. [Core API](#core-api)
4. [Built-in Tools](#built-in-tools)
5. [Agent System](#agent-system)
6. [Tool System](#tool-system)
7. [Hook System](#hook-system)
8. [Permission System](#permission-system)
9. [MCP Integration](#mcp-integration)
10. [Message Types](#message-types)
11. [Session Management](#session-management)
12. [Configuration Options](#configuration-options)
13. [Advanced Features](#advanced-features)
14. [Complete Examples](#complete-examples)
15. [Implementation Details](#implementation-details)

---

## Overview

The Claude Agent SDK enables programmatic creation of AI agents with Claude Code's capabilities. Build autonomous agents that can:
- Understand and navigate codebases
- Read, write, and edit files
- Execute shell commands
- Search code and files
- Integrate with MCP servers
- Handle complex multi-step workflows

### Actual Exports

```javascript
export {
  tool,
  query,
  createSdkMcpServer
};
```

**Source Files** (compiled from TypeScript):
- `../src/entrypoints/agentSdk.ts` - Main query function
- `../src/services/mcp/createSdkMcpServer.ts` - MCP server creation

### Core Capabilities

1. **Query Interface**: Async generator pattern for streaming interaction
2. **Agent System**: Define specialized sub-agents with custom prompts and tool access
3. **17+ Built-in Tools**: Complete file, shell, search, and web capabilities
4. **Hook System**: 9 event types for intercepting and modifying behavior
5. **Permission System**: 4 modes with fine-grained control and runtime updates
6. **MCP Integration**: 4 transport types + custom in-process tools
7. **Session Management**: Resume, fork, and manage long-running sessions
8. **Usage Tracking**: Per-model token and cost tracking
9. **Runtime Control**: Change models, permissions, and parameters during execution
10. **Error Handling**: Comprehensive error types and recovery

---

## Getting Started

### Installation

```bash
npm install @anthropic-ai/claude-agent-sdk
```

### Basic Usage

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const session = query({
  prompt: "Analyze this codebase and create a summary",
  options: {
    cwd: '/path/to/project',
    model: 'claude-sonnet-4',
    maxTurns: 50
  }
});

// Iterate through messages
for await (const message of session) {
  if (message.type === 'assistant') {
    console.log(message.message.content);
  } else if (message.type === 'result') {
    console.log('Usage:', message.usage);
    console.log('Cost:', message.total_cost_usd);
  }
}
```

### Streaming Input

```typescript
async function* generatePrompts() {
  yield {
    type: 'user',
    message: { role: 'user', content: 'First task' },
    session_id: 'session-123',
    parent_tool_use_id: null
  };

  // Wait for some condition...
  await someAsyncOperation();

  yield {
    type: 'user',
    message: { role: 'user', content: 'Next task' },
    session_id: 'session-123',
    parent_tool_use_id: null
  };
}

const session = query({
  prompt: generatePrompts(),
  options: { /* ... */ }
});
```

---

## Core API

### Primary Function: `query()`

The main entry point for creating and interacting with Claude agents.

```typescript
function query(params: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query
```

**Parameters**:
- `prompt`: Either a string or async iterable of messages
- `options`: Configuration object (see [Configuration Options](#configuration-options))

**Returns**: `Query` object (AsyncGenerator)

### Query Interface

The `Query` object extends `AsyncGenerator` and provides control methods:

```typescript
interface Query extends AsyncGenerator<SDKMessage, void> {
  // Control Methods
  interrupt(): Promise<void>;
  setPermissionMode(mode: PermissionMode): Promise<void>;
  setModel(model?: string): Promise<void>;
  setMaxThinkingTokens(maxThinkingTokens: number | null): Promise<void>;
  
  // Information Methods
  supportedCommands(): Promise<SlashCommand[]>;
  supportedModels(): Promise<ModelInfo[]>;
  mcpServerStatus(): Promise<McpServerStatus[]>;
  accountInfo(): Promise<AccountInfo>;
}
```

#### Control Methods

**1. `interrupt()` - Stop current turn**
```typescript
const session = query({ prompt: "Long running task", options });

// Interrupt from another context
setTimeout(() => {
  session.interrupt();
}, 5000);

for await (const message of session) {
  // Processing...
}
```

**2. `setPermissionMode(mode)` - Change permission mode**
```typescript
const session = query({
  prompt: "Task requiring permission changes",
  options: { permissionMode: 'default' }
});

for await (const message of session) {
  if (message.type === 'assistant') {
    // After reviewing first response, bypass permissions
    await session.setPermissionMode('bypassPermissions');
  }
}
```

**3. `setModel(model)` - Switch model**
```typescript
const session = query({
  prompt: "Multi-stage task",
  options: { model: 'claude-sonnet-4' }
});

for await (const message of session) {
  if (needsMoreCapability) {
    // Switch to more capable model
    await session.setModel('claude-opus-4');
  }
}
```

**4. `setMaxThinkingTokens(tokens)` - Limit thinking tokens**
```typescript
const session = query({ prompt: "Complex reasoning task", options });

// Limit thinking tokens for cost control
await session.setMaxThinkingTokens(5000);

// Or remove limit
await session.setMaxThinkingTokens(null);
```

#### Information Methods

**1. `supportedCommands()` - Get slash commands**
```typescript
const commands = await session.supportedCommands();
/*
Returns:
[
  { name: 'help', description: 'Show help', argumentHint: '' },
  { name: 'clear', description: 'Clear session', argumentHint: '' },
  // ...
]
*/
```

**2. `supportedModels()` - Get available models**
```typescript
const models = await session.supportedModels();
/*
Returns:
[
  {
    value: 'claude-sonnet-4',
    displayName: 'Claude Sonnet 4',
    description: 'Latest Sonnet model'
  },
  // ...
]
*/
```

**3. `mcpServerStatus()` - Get MCP server status**
```typescript
const status = await session.mcpServerStatus();
/*
Returns:
[
  {
    name: 'filesystem',
    status: 'connected',
    serverInfo: { name: 'fs-server', version: '1.0.0' }
  },
  {
    name: 'database',
    status: 'failed'
  }
]
*/
```

**4. `accountInfo()` - Get account information**
```typescript
const info = await session.accountInfo();
/*
Returns:
{
  email: 'user@example.com',
  organization: 'My Org',
  subscriptionType: 'pro',
  tokenSource: 'user',
  apiKeySource: 'user'
}
*/
```

### Tool Creation API

Create custom MCP tools for the SDK.

```typescript
function tool<Schema extends ZodRawShape>(
  name: string,
  description: string,
  inputSchema: Schema,
  handler: (args: z.infer<ZodObject<Schema>>, extra: unknown) => Promise<CallToolResult>
): SdkMcpToolDefinition<Schema>
```

**Example**:
```typescript
import { tool } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

const greetTool = tool(
  'greet',
  'Greet a user by name',
  { name: z.string() },
  async (args) => ({
    content: [{ type: 'text', text: `Hello, ${args.name}!` }]
  })
);
```

### MCP Server Creation

Create in-process MCP servers.

```typescript
function createSdkMcpServer(options: {
  name: string;
  version?: string;
  tools?: Array<SdkMcpToolDefinition<any>>;
}): McpSdkServerConfigWithInstance
```

**Example**:
```typescript
import { createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk';

const myServer = createSdkMcpServer({
  name: 'my-tools',
  version: '1.0.0',
  tools: [greetTool, otherTool]
});

const session = query({
  prompt: 'Hello',
  options: {
    mcpServers: {
      'my-tools': myServer
    }
  }
});
```

### Constants & Errors

```typescript
// Hook Events
export const HOOK_EVENTS: readonly [
  "PreToolUse",
  "PostToolUse", 
  "Notification",
  "UserPromptSubmit",
  "SessionStart",
  "SessionEnd",
  "Stop",
  "SubagentStop",
  "PreCompact"
];

// Exit Reasons
export const EXIT_REASONS: string[];

// Error Classes
export class AbortError extends Error {
  // Thrown when operation is aborted
}
```

---

## Built-in Tools

The SDK provides 17 built-in tools for agent operations:

### 1. Task (Agent) - Sub-agent Delegation

Invoke sub-agents for specialized tasks.

**Schema**:
```typescript
interface AgentInput {
  description: string;        // Short task description (3-5 words)
  prompt: string;            // Detailed task for the agent
  subagent_type: string;     // Agent name from agents config
}
```

**Example**:
```typescript
{
  "tool": "agent",
  "input": {
    "description": "Debug authentication issue",
    "prompt": "Find why users can't log in",
    "subagent_type": "debugger"
  }
}
```

### 2. Bash - Command Execution

Execute shell commands.

**Schema**:
```typescript
interface BashInput {
  command: string;
  description?: string;      // What the command does
  timeout?: number;          // Max 600000ms (10 min)
  run_in_background?: boolean;  // For long-running processes
}
```

**Example**:
```typescript
{
  "tool": "bash",
  "input": {
    "command": "npm install",
    "description": "Install dependencies"
  }
}
```

### 3. BashOutput - Background Output Retrieval

Get output from background bash processes.

**Schema**:
```typescript
interface BashOutputInput {
  bash_id: string;
  filter?: string; // Regex to filter output lines
}
```

### 4. KillShell - Process Termination

Kill background shell processes.

**Schema**:
```typescript
interface KillShellInput {
  shell_id: string;
}
```

### 5. Read - File Reading

Read file contents with optional pagination.

**Schema**:
```typescript
interface FileReadInput {
  file_path: string;         // Absolute path
  offset?: number;           // Start line
  limit?: number;            // Number of lines
}
```

### 6. Write - File Creation/Overwrite

Write content to files.

**Schema**:
```typescript
interface FileWriteInput {
  file_path: string;         // Absolute path
  content: string;
}
```

### 7. Edit - File Modification

Search and replace in files.

**Schema**:
```typescript
interface FileEditInput {
  file_path: string;
  old_string: string;        // Must be unique unless replace_all
  new_string: string;
  replace_all?: boolean;     // Replace all occurrences
}
```

**Example**:
```typescript
{
  "tool": "file_edit",
  "input": {
    "file_path": "/path/to/file.ts",
    "old_string": "const x = 1;",
    "new_string": "const x = 2;",
    "replace_all": false
  }
}
```

### 8. Glob - File Pattern Matching

Find files by pattern.

**Schema**:
```typescript
interface GlobInput {
  pattern: string;           // e.g., "**/*.ts"
  path?: string;             // Search directory (default: cwd)
}
```

**Example**:
```typescript
{
  "tool": "glob",
  "input": {
    "pattern": "src/**/*.ts",
    "path": "/project"
  }
}
```

### 9. Grep - Content Search

Search file contents with regex.

**Schema**:
```typescript
interface GrepInput {
  pattern: string;           // Regex pattern
  path?: string;             // Directory to search
  glob?: string;             // File pattern filter
  output_mode?: 'content' | 'files_with_matches' | 'count';
  '-B'?: number;             // Lines before match
  '-A'?: number;             // Lines after match
  '-C'?: number;             // Lines before and after
  '-n'?: boolean;            // Show line numbers
  '-i'?: boolean;            // Case insensitive
  type?: string;             // File type (js, py, etc.)
  head_limit?: number;       // Limit output lines
  multiline?: boolean;       // Enable multiline matching
}
```

**Example**:
```typescript
{
  "tool": "grep",
  "input": {
    "pattern": "function.*getData",
    "path": "./src",
    "output_mode": "content",
    "-C": 3,
    "-n": true,
    "type": "ts"
  }
}
```

### 10. TodoWrite - Task Management

Manage task lists.

**Schema**:
```typescript
interface TodoWriteInput {
  todos: {
    content: string;
    status: 'pending' | 'in_progress' | 'completed';
    activeForm: string;      // Present continuous form
  }[];
}
```

### 11. NotebookEdit - Jupyter Notebook Editing

Edit Jupyter notebooks.

**Schema**:
```typescript
interface NotebookEditInput {
  notebook_path: string;
  cell_id?: string;          // Cell to edit (or position for insert)
  new_source: string;
  cell_type?: 'code' | 'markdown';
  edit_mode?: 'replace' | 'insert' | 'delete';
}
```

### 12. WebFetch - Web Content Retrieval

Fetch and analyze web content.

**Schema**:
```typescript
interface WebFetchInput {
  url: string;
  prompt: string;            // What to extract from the page
}
```

### 13. WebSearch - Web Searching

Search the web.

**Schema**:
```typescript
interface WebSearchInput {
  query: string;
  allowed_domains?: string[];
  blocked_domains?: string[];
}
```

### 14. ListMcpResources - MCP Resource Discovery

List resources from MCP servers.

**Schema**:
```typescript
interface ListMcpResourcesInput {
  server?: string;           // Optional server name filter
}
```

### 15. ReadMcpResource - MCP Resource Fetching

Read a specific MCP resource.

**Schema**:
```typescript
interface ReadMcpResourceInput {
  server: string;
  uri: string;
}
```

### 16. Mcp - Custom MCP Tool Invocation

Generic MCP tool invocation.

**Schema**:
```typescript
interface McpInput {
  [key: string]: unknown;    // Tool-specific parameters
}
```

### 17. ExitPlanMode - Plan Approval

Exit planning mode and begin execution.

**Schema**:
```typescript
interface ExitPlanModeInput {
  plan: string;              // The plan (markdown supported)
}
```

---

## Agent System

### Agent Definition

Define specialized sub-agents with custom capabilities:

```typescript
interface AgentDefinition {
  description: string;       // Human-readable description
  tools?: string[];          // Optional array of allowed tool names
  prompt: string;            // Custom system prompt
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
}
```

### Configuring Agents

```typescript
const options = {
  agents: {
    // Code reviewer agent
    'code-reviewer': {
      description: 'Reviews code for quality, security, and best practices',
      tools: ['Read', 'Grep', 'Glob'],  // Restrict to read-only tools
      prompt: `You are a senior code reviewer. Analyze code for:
        - Security vulnerabilities
        - Performance issues
        - Code quality and maintainability
        - Best practices compliance
        Provide specific, actionable feedback.`,
      model: 'sonnet'  // Can use different model than parent
    },

    // Test generator agent
    'test-writer': {
      description: 'Generates comprehensive test suites',
      tools: ['Read', 'Write', 'Bash'],
      prompt: `You are a testing expert. Create thorough test suites with:
        - Unit tests for all functions
        - Integration tests for workflows
        - Edge case coverage
        - Clear test descriptions`,
      model: 'opus'  // More capable model for complex task
    },

    // Documentation agent
    'doc-writer': {
      description: 'Creates and updates documentation',
      prompt: 'Generate clear, comprehensive documentation',
      model: 'inherit'  // Use same model as parent
      // tools not specified = inherit parent's tools
    }
  }
};

const session = query({
  prompt: "Review all code and generate tests",
  options
});
```

### Agent Features

- **Description**: Human-readable description of agent's purpose
- **Tools**: Optional array of allowed tool names (restricts access)
- **Prompt**: Custom system prompt for specialized behavior
- **Model**: 'sonnet' | 'opus' | 'haiku' | 'inherit'
- **Invocation**: Agents are invoked via the Task tool with `subagent_type` parameter

### Agent Tool Access

```typescript
// Agent with full tool access (inherits from parent)
agents: {
  'full-access': {
    description: 'Has all tools',
    prompt: 'Do anything needed'
    // No tools restriction
  }
}

// Agent with restricted tools
agents: {
  'read-only': {
    description: 'Can only read and search',
    tools: ['Read', 'Glob', 'Grep'],
    prompt: 'Analyze code without modifications'
  }
}
```

### System Prompt Configuration

Two approaches for system prompts:

**Approach 1: Preset with Append**
```typescript
type Options = {
  systemPrompt?: {
    type: 'preset';
    preset: 'claude_code';
    append?: string;
  };
};
```

**Approach 2: Custom**
```typescript
type Options = {
  systemPrompt?: string;
};
```

**Example**:
```typescript
const options1 = {
  systemPrompt: {
    type: 'preset',
    preset: 'claude_code',
    append: 'Focus on code quality and maintainability.'
  }
};

const options2 = {
  systemPrompt: 'You are a helpful coding assistant.'
};
```

---

## Tool System

### Custom MCP Tools

Create custom tools that run in-process:

```typescript
import { tool, createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

// Define a custom tool
const customTool = tool(
  'calculate_metrics',
  'Calculates code metrics for a file',
  {
    file_path: z.string().describe('Path to the file'),
    metrics: z.array(z.enum(['complexity', 'coverage', 'maintainability']))
  },
  async (args, extra) => {
    // Tool implementation
    const result = await calculateMetrics(args.file_path, args.metrics);
    return {
      content: [{ type: 'text', text: JSON.stringify(result) }]
    };
  }
);

// Create MCP server with custom tools
const mcpServer = createSdkMcpServer({
  name: 'metrics-server',
  version: '1.0.0',
  tools: [customTool]
});

// Use in query
const session = query({
  prompt: "Analyze code metrics",
  options: {
    mcpServers: {
      'metrics': mcpServer
    }
  }
});
```

---

## Hook System

### Available Hooks

Hooks allow you to intercept and modify agent behavior at key points:

```typescript
const HOOK_EVENTS = [
  'PreToolUse',         // Before a tool is executed
  'PostToolUse',        // After a tool completes
  'Notification',       // When a notification is sent
  'UserPromptSubmit',   // When user submits a prompt
  'SessionStart',       // When a session starts
  'SessionEnd',         // When a session ends
  'Stop',               // When agent is stopped
  'SubagentStop',       // When a sub-agent stops
  'PreCompact'          // Before context compaction
] as const;

type HookEvent = typeof HOOK_EVENTS[number];
```

### Hook Definition

```typescript
interface HookCallbackMatcher {
  matcher?: string;          // Optional pattern to match tool names
  hooks: HookCallback[];     // Array of hook functions
}

type HookCallback = (
  input: HookInput,
  toolUseID: string | undefined,
  options: { signal: AbortSignal }
) => Promise<HookJSONOutput>;
```

### Hook Input Types

#### PreToolUse
```typescript
interface PreToolUseHookInput {
  hook_event_name: 'PreToolUse';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  tool_name: string;
  tool_input: unknown;
}
```

#### PostToolUse
```typescript
interface PostToolUseHookInput {
  hook_event_name: 'PostToolUse';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;
}
```

#### UserPromptSubmit
```typescript
interface UserPromptSubmitHookInput {
  hook_event_name: 'UserPromptSubmit';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  prompt: string;
}
```

#### SessionStart
```typescript
interface SessionStartHookInput {
  hook_event_name: 'SessionStart';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  source: 'startup' | 'resume' | 'clear' | 'compact';
}
```

#### SessionEnd
```typescript
interface SessionEndHookInput {
  hook_event_name: 'SessionEnd';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  reason: ExitReason;
}
```

#### Notification
```typescript
interface NotificationHookInput {
  hook_event_name: 'Notification';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  message: string;
  title?: string;
}
```

#### PreCompact
```typescript
interface PreCompactHookInput {
  hook_event_name: 'PreCompact';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  trigger: 'manual' | 'auto';
  custom_instructions: string | null;
}
```

#### Stop / SubagentStop
```typescript
interface StopHookInput {
  hook_event_name: 'Stop' | 'SubagentStop';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  stop_hook_active: boolean;
}
```

### Hook Output Types

#### Synchronous Hook Output
```typescript
interface SyncHookJSONOutput {
  continue?: boolean;              // Continue execution
  suppressOutput?: boolean;         // Hide hook output from user
  stopReason?: string;             // Why execution stopped
  decision?: 'approve' | 'block';  // Permission decision
  systemMessage?: string;          // Message to show user
  reason?: string;                 // Reason for decision

  // Hook-specific outputs
  hookSpecificOutput?:
    | {
        hookEventName: 'PreToolUse';
        permissionDecision?: 'allow' | 'deny' | 'ask';
        permissionDecisionReason?: string;
        updatedInput?: Record<string, unknown>;
      }
    | {
        hookEventName: 'UserPromptSubmit';
        additionalContext?: string;
      }
    | {
        hookEventName: 'SessionStart';
        additionalContext?: string;
      }
    | {
        hookEventName: 'PostToolUse';
        additionalContext?: string;
      };
}
```

#### Asynchronous Hook Output
```typescript
interface AsyncHookJSONOutput {
  async: true;
  asyncTimeout?: number;    // Timeout in milliseconds
}
```

### Hook Usage Example

```typescript
const options = {
  hooks: {
    // Pre-tool use hook with matcher
    PreToolUse: [{
      matcher: 'Bash',  // Only match Bash tool
      hooks: [
        async (input, toolUseID, { signal }) => {
          const bashInput = input as PreToolUseHookInput;

          // Log the command
          console.log('Executing:', bashInput.tool_input);

          // Modify dangerous commands
          if (typeof bashInput.tool_input === 'object' &&
              'command' in bashInput.tool_input) {
            const cmd = bashInput.tool_input.command;

            if (cmd.includes('rm -rf /')) {
              return {
                decision: 'block',
                systemMessage: 'Dangerous command blocked!',
                reason: 'Command would delete root directory',
                hookSpecificOutput: {
                  hookEventName: 'PreToolUse',
                  permissionDecision: 'deny',
                  permissionDecisionReason: 'Safety violation'
                }
              };
            }
          }

          return { continue: true };
        }
      ]
    }],

    // Post-tool use hook (no matcher = all tools)
    PostToolUse: [{
      hooks: [
        async (input) => {
          const postInput = input as PostToolUseHookInput;
          console.log(`Tool ${postInput.tool_name} completed`);

          // Add context based on tool result
          return {
            continue: true,
            hookSpecificOutput: {
              hookEventName: 'PostToolUse',
              additionalContext: 'Tool executed successfully'
            }
          };
        }
      ]
    }],

    // Session lifecycle hooks
    SessionStart: [{
      hooks: [
        async (input) => {
          const startInput = input as SessionStartHookInput;
          console.log(`Session started from ${startInput.source}`);
          return {
            continue: true,
            hookSpecificOutput: {
              hookEventName: 'SessionStart',
              additionalContext: 'Welcome back!'
            }
          };
        }
      ]
    }],

    SessionEnd: [{
      hooks: [
        async (input) => {
          const endInput = input as SessionEndHookInput;
          console.log(`Session ended: ${endInput.reason}`);
          // Cleanup, save state, etc.
          return { continue: true };
        }
      ]
    }],

    // User interaction hooks
    UserPromptSubmit: [{
      hooks: [
        async (input) => {
          const promptInput = input as UserPromptSubmitHookInput;
          // Validate or augment user prompts
          return {
            continue: true,
            hookSpecificOutput: {
              hookEventName: 'UserPromptSubmit',
              additionalContext: 'Additional context about the project'
            }
          };
        }
      ]
    }]
  }
};
```

---

## Permission System

### Permission Modes

```typescript
type PermissionMode =
  | 'default'            // Ask for each tool use
  | 'acceptEdits'        // Auto-accept file edits
  | 'bypassPermissions'  // Skip all permission checks
  | 'plan';              // Planning mode (no execution)
```

### Permission Behavior

```typescript
type PermissionBehavior = 'allow' | 'deny' | 'ask';
```

### Permission Rules

```typescript
interface PermissionRuleValue {
  toolName: string;
  ruleContent?: string;    // Optional pattern to match
}
```

### Permission Updates

```typescript
type PermissionUpdate =
  | {
      type: 'addRules';
      rules: PermissionRuleValue[];
      behavior: PermissionBehavior;
      destination: 'userSettings' | 'projectSettings' | 'localSettings' | 'session';
    }
  | {
      type: 'replaceRules';
      rules: PermissionRuleValue[];
      behavior: PermissionBehavior;
      destination: PermissionUpdateDestination;
    }
  | {
      type: 'removeRules';
      rules: PermissionRuleValue[];
      behavior: PermissionBehavior;
      destination: PermissionUpdateDestination;
    }
  | {
      type: 'setMode';
      mode: PermissionMode;
      destination: PermissionUpdateDestination;
    }
  | {
      type: 'addDirectories';
      directories: string[];
      destination: PermissionUpdateDestination;
    }
  | {
      type: 'removeDirectories';
      directories: string[];
      destination: PermissionUpdateDestination;
    };
```

### Permission Results

```typescript
type PermissionResult =
  | {
      behavior: 'allow';
      updatedInput: Record<string, unknown>;
      updatedPermissions?: PermissionUpdate[];
    }
  | {
      behavior: 'deny';
      message: string;          // Reason or guidance
      interrupt?: boolean;      // Stop execution
    };
```

### Custom Permission Handler

```typescript
type CanUseTool = (
  toolName: string,
  input: Record<string, unknown>,
  options: {
    signal: AbortSignal;
    suggestions?: PermissionUpdate[];  // Suggested permission updates
  }
) => Promise<PermissionResult>;

// Usage
const options = {
  canUseTool: async (toolName, input, { signal, suggestions }) => {
    // Custom permission logic
    if (toolName === 'Bash') {
      const command = input.command as string;

      // Allow safe commands
      if (command.startsWith('git ') || command.startsWith('npm ')) {
        return {
          behavior: 'allow',
          updatedInput: input,
          // Optionally apply suggestions for "always allow"
          updatedPermissions: suggestions
        };
      }

      // Block dangerous commands
      if (command.includes('rm -rf')) {
        return {
          behavior: 'deny',
          message: 'Dangerous delete command blocked',
          interrupt: true
        };
      }
    }

    // Default: allow with original input
    return {
      behavior: 'allow',
      updatedInput: input
    };
  }
};
```

### Runtime Permission Control

```typescript
const session = query({ prompt: "...", options: { permissionMode: 'default' } });

// Change permission mode during execution
await session.setPermissionMode('bypassPermissions');

// Continue iteration
for await (const message of session) {
  // ...
}
```

---

## MCP Integration

### MCP Server Types

#### 1. Stdio Transport
```typescript
interface McpStdioServerConfig {
  type?: 'stdio';
  command: string;
  args?: string[];
  env?: Record<string, string>;
}

// Example
const mcpServers = {
  'filesystem': {
    type: 'stdio',
    command: 'npx',
    args: ['-y', '@modelcontextprotocol/server-filesystem', '/path/to/allowed'],
    env: { DEBUG: 'true' }
  }
};
```

#### 2. SSE Transport
```typescript
interface McpSSEServerConfig {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
}

// Example
const mcpServers = {
  'remote-api': {
    type: 'sse',
    url: 'https://api.example.com/mcp',
    headers: { 'Authorization': 'Bearer token' }
  }
};
```

#### 3. HTTP Transport
```typescript
interface McpHttpServerConfig {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
}
```

#### 4. SDK Transport (In-Process)
```typescript
interface McpSdkServerConfigWithInstance {
  type: 'sdk';
  name: string;
  instance: McpServer;
}

// Create custom tools
const customServer = createSdkMcpServer({
  name: 'my-tools',
  version: '1.0.0',
  tools: [
    tool('my_tool', 'Description', { /* schema */ }, async (args) => {
      // Implementation
      return { content: [{ type: 'text', text: 'Result' }] };
    })
  ]
});

const options = {
  mcpServers: {
    'custom': customServer
  }
};
```

### MCP Server Configuration

```typescript
const options = {
  mcpServers: {
    // Filesystem server
    'fs': {
      type: 'stdio',
      command: 'npx',
      args: ['-y', '@modelcontextprotocol/server-filesystem', './data']
    },

    // Database server
    'db': {
      type: 'stdio',
      command: 'uvx',
      args: ['mcp-server-postgres', 'postgresql://localhost/mydb']
    },

    // Remote API server
    'api': {
      type: 'sse',
      url: 'https://api.example.com/mcp',
      headers: { 'X-API-Key': process.env.API_KEY }
    },

    // Custom in-process tools
    'custom': createSdkMcpServer({
      name: 'custom-tools',
      tools: [myCustomTool]
    })
  },

  strictMcpConfig: true  // Fail if MCP server connection fails
};
```

### MCP Server Status

```typescript
const session = query({ prompt: "...", options });

// Get MCP server status
const status = await session.mcpServerStatus();

/*
Returns:
[
  {
    name: 'fs',
    status: 'connected',
    serverInfo: { name: 'filesystem', version: '1.0.0' }
  },
  {
    name: 'db',
    status: 'failed'
  },
  {
    name: 'custom',
    status: 'connected',
    serverInfo: { name: 'custom-tools', version: '1.0.0' }
  }
]
*/
```

---

## Message Types

### Message Type Hierarchy

```typescript
type SDKMessage =
  | SDKUserMessage
  | SDKUserMessageReplay
  | SDKAssistantMessage
  | SDKPartialAssistantMessage
  | SDKResultMessage
  | SDKSystemMessage
  | SDKCompactBoundaryMessage
  | SDKHookResponseMessage;
```

### User Messages

```typescript
interface SDKUserMessage {
  type: 'user';
  uuid?: UUID;
  session_id: string;
  message: APIUserMessage;
  parent_tool_use_id: string | null;
  isSynthetic?: boolean;      // Not from actual user input
}

interface SDKUserMessageReplay {
  type: 'user';
  uuid: UUID;
  session_id: string;
  message: APIUserMessage;
  parent_tool_use_id: string | null;
  isReplay: true;             // Acknowledgment of existing message
}
```

### Assistant Messages

```typescript
interface SDKAssistantMessage {
  type: 'assistant';
  uuid: UUID;
  session_id: string;
  message: APIAssistantMessage;  // Contains content blocks and tool uses
  parent_tool_use_id: string | null;
}

interface SDKPartialAssistantMessage {
  type: 'stream_event';
  uuid: UUID;
  session_id: string;
  event: RawMessageStreamEvent;  // Streaming event from API
  parent_tool_use_id: string | null;
}
```

### Result Messages

```typescript
type SDKResultMessage =
  | {
      type: 'result';
      subtype: 'success';
      uuid: UUID;
      session_id: string;
      duration_ms: number;
      duration_api_ms: number;
      is_error: false;
      num_turns: number;
      result: string;
      total_cost_usd: number;
      usage: NonNullableUsage;
      modelUsage: Record<string, ModelUsage>;
      permission_denials: SDKPermissionDenial[];
    }
  | {
      type: 'result';
      subtype: 'error_max_turns' | 'error_during_execution';
      uuid: UUID;
      session_id: string;
      duration_ms: number;
      duration_api_ms: number;
      is_error: true;
      num_turns: number;
      total_cost_usd: number;
      usage: NonNullableUsage;
      modelUsage: Record<string, ModelUsage>;
      permission_denials: SDKPermissionDenial[];
    };

interface SDKPermissionDenial {
  tool_name: string;
  tool_use_id: string;
  tool_input: Record<string, unknown>;
}
```

### System Messages

```typescript
interface SDKSystemMessage {
  type: 'system';
  subtype: 'init';
  uuid: UUID;
  session_id: string;
  agents?: string[];
  apiKeySource: ApiKeySource;
  claude_code_version: string;
  cwd: string;
  tools: string[];
  mcp_servers: { name: string; status: string; }[];
  model: string;
  permissionMode: PermissionMode;
  slash_commands: string[];
  output_style: string;
}

interface SDKCompactBoundaryMessage {
  type: 'system';
  subtype: 'compact_boundary';
  uuid: UUID;
  session_id: string;
  compact_metadata: {
    trigger: 'manual' | 'auto';
    pre_tokens: number;
  };
}

interface SDKHookResponseMessage {
  type: 'system';
  subtype: 'hook_response';
  uuid: UUID;
  session_id: string;
  hook_name: string;
  hook_event: string;
  stdout: string;
  stderr: string;
  exit_code?: number;
}
```

### Usage Tracking

```typescript
interface NonNullableUsage {
  input_tokens: number;
  output_tokens: number;
  cache_creation_input_tokens: number;
  cache_read_input_tokens: number;
}

interface ModelUsage {
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
  webSearchRequests: number;
  costUSD: number;
  contextWindow: number;
}
```

---

## Session Management

### Resume Sessions
```typescript
// Start a session
const session1 = query({
  prompt: "Start working on feature X",
  options: { cwd: '/project' }
});

// Get session ID from first message
let sessionId: string;
for await (const message of session1) {
  if (message.session_id) {
    sessionId = message.session_id;
    break;
  }
}

// Later, resume the session
const session2 = query({
  prompt: "Continue with feature X",
  options: {
    resume: sessionId,
    cwd: '/project'
  }
});
```

### Fork Sessions
```typescript
// Resume but fork to new session ID
const session = query({
  prompt: "Try alternative approach",
  options: {
    resume: originalSessionId,
    forkSession: true  // Creates new session with copied history
  }
});
```

### Resume from Specific Message
```typescript
// Resume from a specific assistant message
const session = query({
  prompt: "Continue from checkpoint",
  options: {
    resume: sessionId,
    resumeSessionAt: messageId  // message.id from SDKAssistantMessage
  }
});
```

### Exit Reasons

```typescript
const EXIT_REASONS = [
  'user_exit',
  'max_turns',
  'error',
  'abort',
  'complete',
  // ... more reasons
] as const;

// In SessionEnd hook
hooks: {
  SessionEnd: [{
    hooks: [
      async (input) => {
        const endInput = input as SessionEndHookInput;

        switch (endInput.reason) {
          case 'max_turns':
            console.log('Hit maximum turns');
            break;
          case 'complete':
            console.log('Task completed successfully');
            break;
          case 'user_exit':
            console.log('User requested exit');
            break;
          case 'error':
            console.log('Error occurred');
            break;
        }

        return { continue: true };
      }
    ]
  }]
}
```

---

## Configuration Options

### Complete Options Interface

```typescript
interface Options {
  // Execution Control
  abortController?: AbortController;
  maxTurns?: number;
  maxThinkingTokens?: number;
  continue?: boolean;

  // Model Configuration
  model?: string;
  fallbackModel?: string;

  // Session Management
  resume?: string;                    // Session ID to resume
  resumeSessionAt?: string;           // Resume from specific message ID
  forkSession?: boolean;              // Fork resumed session to new ID

  // System Prompts
  systemPrompt?: string | {
    type: 'preset';
    preset: 'claude_code';
    append?: string;
  };

  // Agent Configuration
  agents?: Record<string, AgentDefinition>;
  settingSources?: ('user' | 'project' | 'local')[];

  // Working Directory & Environment
  cwd?: string;
  env?: Record<string, string | undefined>;
  additionalDirectories?: string[];

  // Tool Configuration
  allowedTools?: string[];
  disallowedTools?: string[];

  // Permission System
  permissionMode?: PermissionMode;
  canUseTool?: CanUseTool;
  permissionPromptToolName?: string;

  // Hooks
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;

  // MCP Configuration
  mcpServers?: Record<string, McpServerConfig>;
  strictMcpConfig?: boolean;

  // Runtime Configuration
  executable?: 'bun' | 'deno' | 'node';
  executableArgs?: string[];
  pathToClaudeCodeExecutable?: string;

  // Output Control
  includePartialMessages?: boolean;
  stderr?: (data: string) => void;

  // Advanced
  extraArgs?: Record<string, string | null>;
}
```

### Example Configurations

#### Basic Configuration
```typescript
const session = query({
  prompt: "Analyze the codebase",
  options: {
    cwd: '/path/to/project',
    model: 'claude-sonnet-4',
    maxTurns: 25
  }
});
```

#### Advanced Configuration
```typescript
const session = query({
  prompt: "Comprehensive analysis and refactoring",
  options: {
    cwd: process.cwd(),
    model: 'claude-sonnet-4',
    fallbackModel: 'claude-opus-4',
    maxTurns: 100,
    maxThinkingTokens: 10000,

    // Custom system prompt
    systemPrompt: {
      type: 'preset',
      preset: 'claude_code',
      append: 'Focus on code quality and maintainability.'
    },

    // Specialized agents
    agents: {
      'refactor': {
        description: 'Refactors code for better quality',
        tools: ['Read', 'Write', 'Edit', 'Bash'],
        prompt: 'Apply SOLID principles and clean code practices',
        model: 'opus'
      }
    },

    // Permission configuration
    permissionMode: 'acceptEdits',
    additionalDirectories: ['/path/to/libs'],

    // Tool restrictions
    allowedTools: ['Read', 'Write', 'Edit', 'Bash', 'Glob', 'Grep'],

    // MCP servers
    mcpServers: {
      'db': {
        type: 'stdio',
        command: 'uvx',
        args: ['mcp-server-postgres', process.env.DATABASE_URL!]
      }
    },

    // Hooks
    hooks: {
      PreToolUse: [{
        matcher: 'Bash',
        hooks: [
          async (input) => {
            console.log('Executing:', input);
            return { continue: true };
          }
        ]
      }]
    },

    // Environment
    env: {
      NODE_ENV: 'development',
      DEBUG: 'true'
    },

    // Output
    includePartialMessages: true,
    stderr: (data) => console.error('ERROR:', data)
  }
});
```

---

## Advanced Features

### 1. Context Compaction

Handle automatic context window management:

```typescript
const options = {
  hooks: {
    PreCompact: [{
      hooks: [
        async (input) => {
          const compactInput = input as PreCompactHookInput;
          console.log(`Compacting ${compactInput.pre_tokens} tokens`);
          console.log(`Trigger: ${compactInput.trigger}`);

          // Optionally interrupt compaction
          if (compactInput.trigger === 'auto' && shouldKeepFullContext) {
            return {
              continue: false,
              stopReason: 'Keeping full context for this operation'
            };
          }

          return { continue: true };
        }
      ]
    }]
  }
};
```

### 2. Streaming Input & Interactive Sessions

```typescript
async function* interactiveSession() {
  // Initial prompt
  yield {
    type: 'user' as const,
    message: { role: 'user' as const, content: 'Start analysis' },
    session_id: 'interactive-session',
    parent_tool_use_id: null
  };

  // Wait for user input
  const nextPrompt = await getUserInput();

  yield {
    type: 'user' as const,
    message: { role: 'user' as const, content: nextPrompt },
    session_id: 'interactive-session',
    parent_tool_use_id: null
  };

  // Can continue yielding based on conditions...
}

const session = query({
  prompt: interactiveSession(),
  options: { /* ... */ }
});

for await (const message of session) {
  if (message.type === 'assistant') {
    // Show assistant response to user
    displayToUser(message.message.content);
  }
}
```

### 3. Error Handling

```typescript
import { AbortError } from '@anthropic-ai/claude-agent-sdk';

const abortController = new AbortController();

const session = query({
  prompt: "Long task",
  options: {
    abortController,
    maxTurns: 50
  }
});

try {
  for await (const message of session) {
    if (message.type === 'result') {
      if (message.subtype === 'error_max_turns') {
        console.error('Hit max turns limit');
      } else if (message.subtype === 'error_during_execution') {
        console.error('Execution error occurred');
      } else {
        console.log('Success:', message.result);
      }
    }
  }
} catch (error) {
  if (error instanceof AbortError) {
    console.log('Task was aborted');
  } else {
    console.error('Unexpected error:', error);
  }
}
```

### 4. Permission Denial Tracking

Track which tools were denied during execution:

```typescript
for await (const message of session) {
  if (message.type === 'result') {
    if (message.permission_denials.length > 0) {
      console.log('Tools denied:');
      message.permission_denials.forEach(denial => {
        console.log(`- ${denial.tool_name}`);
        console.log(`  Input:`, denial.tool_input);
      });
    }
  }
}
```

### 5. Multi-Model Usage Tracking

Track usage per model when using multiple models:

```typescript
for await (const message of session) {
  if (message.type === 'result') {
    console.log('Total cost:', message.total_cost_usd);
    console.log('\nPer-model usage:');

    Object.entries(message.modelUsage).forEach(([model, usage]) => {
      console.log(`\n${model}:`);
      console.log(`  Input tokens: ${usage.inputTokens}`);
      console.log(`  Output tokens: ${usage.outputTokens}`);
      console.log(`  Cache hits: ${usage.cacheReadInputTokens}`);
      console.log(`  Cost: $${usage.costUSD}`);
      console.log(`  Context window: ${usage.contextWindow}`);
    });
  }
}
```

### 6. Custom Executables

Run with different JavaScript runtimes:

```typescript
const options = {
  executable: 'bun',  // or 'deno' or 'node'
  executableArgs: ['--inspect'],
  pathToClaudeCodeExecutable: '/custom/path/to/claude-code'
};
```

### 7. Setting Sources

Control which configuration sources are used:

```typescript
const options = {
  // Only use user and project settings, not local
  settingSources: ['user', 'project']
};
```

---

## Complete Examples

### Example 1: Basic File Analysis

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const session = query({
  prompt: 'List all TypeScript files',
  options: {
    cwd: '/project'
  }
});

for await (const message of session) {
  if (message.type === 'assistant') {
    console.log('Assistant:', message.message);
  } else if (message.type === 'result') {
    console.log('Cost: $', message.total_cost_usd);
  }
}
```

### Example 2: Custom MCP Tools

```typescript
import { query, tool, createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

const dbTool = tool(
  'query_database',
  'Query the database',
  { sql: z.string() },
  async (args) => {
    const result = await db.query(args.sql);
    return {
      content: [{ type: 'text', text: JSON.stringify(result) }]
    };
  }
);

const server = createSdkMcpServer({
  name: 'db-tools',
  tools: [dbTool]
});

const session = query({
  prompt: 'Get all users',
  options: {
    mcpServers: { 'db-tools': server }
  }
});
```

### Example 3: With Hooks and Permissions

```typescript
const canUseTool: CanUseTool = async (toolName, input) => {
  if (toolName === 'file_write') {
    const path = input.file_path as string;
    if (path.includes('/sensitive/')) {
      return {
        behavior: 'deny',
        message: 'Cannot write to sensitive directory'
      };
    }
  }
  return { behavior: 'allow', updatedInput: input };
};

const session = query({
  prompt: 'Refactor the code',
  options: {
    canUseTool,
    permissionMode: 'default',
    hooks: {
      PreToolUse: [{
        hooks: [async (input) => {
          console.log(`Tool: ${input.tool_name}`);
          return { continue: true };
        }]
      }],
      PostToolUse: [{
        hooks: [async (input) => {
          console.log(`Result: ${input.tool_response}`);
          return {};
        }]
      }]
    }
  }
});
```

### Example 4: Advanced Agent System

```typescript
import {
  query,
  createSdkMcpServer,
  tool,
  type SDKMessage,
  type PreToolUseHookInput,
  type PostToolUseHookInput
} from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

// Create custom MCP tools
const metricsServer = createSdkMcpServer({
  name: 'code-metrics',
  version: '1.0.0',
  tools: [
    tool(
      'analyze_complexity',
      'Analyzes code complexity metrics',
      {
        path: z.string(),
        metrics: z.array(z.enum(['cyclomatic', 'cognitive', 'halstead']))
      },
      async (args) => {
        const results = await analyzeCodeMetrics(args.path, args.metrics);
        return {
          content: [{ type: 'text', text: JSON.stringify(results, null, 2) }]
        };
      }
    )
  ]
});

// Configure session with all features
const session = query({
  prompt: "Perform comprehensive code review and refactoring",
  options: {
    cwd: '/path/to/project',
    model: 'claude-sonnet-4',
    fallbackModel: 'claude-opus-4',
    maxTurns: 100,
    maxThinkingTokens: 20000,

    // System prompt
    systemPrompt: {
      type: 'preset',
      preset: 'claude_code',
      append: `Focus on:
        - Code quality and maintainability
        - Performance optimization
        - Security best practices
        - Test coverage`
    },

    // Define specialized agents
    agents: {
      'security-auditor': {
        description: 'Performs security audit of code',
        tools: ['Read', 'Grep', 'Glob'],
        prompt: `You are a security expert. Analyze code for:
          - SQL injection vulnerabilities
          - XSS vulnerabilities
          - Authentication/authorization issues
          - Sensitive data exposure
          - Dependency vulnerabilities
          Provide specific line numbers and remediation steps.`,
        model: 'opus'
      },

      'performance-optimizer': {
        description: 'Optimizes code performance',
        tools: ['Read', 'Edit', 'Bash', 'analyze_complexity'],
        prompt: `You are a performance expert. Optimize for:
          - Algorithm efficiency
          - Memory usage
          - Database query optimization
          - Caching opportunities
          Measure before and after with benchmarks.`,
        model: 'sonnet'
      },

      'test-engineer': {
        description: 'Creates comprehensive test suites',
        tools: ['Read', 'Write', 'Bash'],
        prompt: `Create thorough tests with:
          - Unit tests for all functions
          - Integration tests for workflows
          - Edge cases and error conditions
          - >80% code coverage
          Run tests and fix failures.`,
        model: 'sonnet'
      }
    },

    // Permission configuration
    permissionMode: 'acceptEdits',
    additionalDirectories: ['/shared/libs'],

    // Tool restrictions
    allowedTools: [
      'Read', 'Write', 'Edit', 'Bash',
      'Glob', 'Grep', 'Task',
      'analyze_complexity'
    ],

    // MCP servers
    mcpServers: {
      'metrics': metricsServer,
      'database': {
        type: 'stdio',
        command: 'uvx',
        args: ['mcp-server-postgres', process.env.DATABASE_URL!]
      }
    },
    strictMcpConfig: false,

    // Hooks for monitoring and control
    hooks: {
      PreToolUse: [{
        matcher: 'Bash',
        hooks: [
          async (input, toolUseID, { signal }) => {
            const bashInput = input as PreToolUseHookInput;
            console.log('[BASH]', bashInput.tool_input);

            // Safety checks
            if (typeof bashInput.tool_input === 'object' &&
                'command' in bashInput.tool_input) {
              const cmd = bashInput.tool_input.command as string;

              if (cmd.includes('rm -rf') || cmd.includes('format')) {
                return {
                  decision: 'block',
                  systemMessage: 'Dangerous command blocked',
                  hookSpecificOutput: {
                    hookEventName: 'PreToolUse',
                    permissionDecision: 'deny'
                  }
                };
              }
            }

            return { continue: true };
          }
        ]
      }],

      PostToolUse: [{
        hooks: [
          async (input) => {
            const postInput = input as PostToolUseHookInput;
            console.log(`[COMPLETE] ${postInput.tool_name}`);

            // Log tool results for audit
            logToolUse(postInput.tool_name, postInput.tool_input, postInput.tool_response);

            return { continue: true };
          }
        ]
      }],

      SessionEnd: [{
        hooks: [
          async (input) => {
            console.log('[SESSION END]', input);
            // Cleanup, generate reports, etc.
            return { continue: true };
          }
        ]
      }]
    },

    // Environment
    env: {
      NODE_ENV: 'development',
      DEBUG: 'true'
    },

    // Include streaming updates
    includePartialMessages: true,

    // Custom error handler
    stderr: (data) => console.error('[ERROR]', data)
  }
});

// Process messages
let totalCost = 0;
const toolsUsed = new Set<string>();

try {
  for await (const message of session) {
    switch (message.type) {
      case 'system':
        if (message.subtype === 'init') {
          console.log('Session initialized');
          console.log('Available agents:', message.agents);
          console.log('MCP servers:', message.mcp_servers);
        }
        break;

      case 'assistant':
        console.log('\n[ASSISTANT]');
        message.message.content.forEach(block => {
          if (block.type === 'text') {
            console.log(block.text);
          } else if (block.type === 'tool_use') {
            toolsUsed.add(block.name);
            console.log(`Using tool: ${block.name}`);
          }
        });
        break;

      case 'stream_event':
        // Handle streaming updates
        if (message.event.type === 'content_block_delta') {
          process.stdout.write(message.event.delta.text || '');
        }
        break;

      case 'result':
        console.log('\n[RESULT]');
        console.log('Status:', message.subtype);
        console.log('Turns:', message.num_turns);
        console.log('Duration:', message.duration_ms, 'ms');
        console.log('Cost: $', message.total_cost_usd);

        if (!message.is_error) {
          console.log('Result:', message.result);
        }

        if (message.permission_denials.length > 0) {
          console.log('\nPermission denials:');
          message.permission_denials.forEach(d => {
            console.log(`- ${d.tool_name}`);
          });
        }

        console.log('\nPer-model usage:');
        Object.entries(message.modelUsage).forEach(([model, usage]) => {
          console.log(`${model}:`, {
            input: usage.inputTokens,
            output: usage.outputTokens,
            cost: usage.costUSD
          });
        });

        totalCost = message.total_cost_usd;
        break;
    }
  }

  console.log('\n=== Session Complete ===');
  console.log('Total cost:', totalCost);
  console.log('Tools used:', Array.from(toolsUsed));

} catch (error) {
  console.error('Session error:', error);
}

// Helper functions
function analyzeCodeMetrics(path: string, metrics: string[]) {
  // Implementation
  return { complexity: 5, maintainability: 85 };
}

function logToolUse(tool: string, input: unknown, response: unknown) {
  // Log to file/database for audit trail
}
```

---

## Implementation Details

### Core Classes

#### Query Class
```javascript
class Query {
  transport;           // ProcessTransport instance
  isSingleUserTurn;   // Boolean flag
  canUseTool;         // Permission callback
  hooks;              // Hook callbacks
  abortController;    // AbortController for cancellation
}
```

#### ProcessTransport Class
Handles communication with the Claude Code CLI process via stdio:
```javascript
class ProcessTransport {
  options;
  child;              // Child process
  childStdin;         // Stdin stream
  childStdout;        // Stdout stream
  ready;              // Boolean ready state
  abortController;
  exitError;
  exitListeners;
  processExitHandler;
  abortHandler;
}
```

#### AbortError Class
```javascript
class AbortError extends Error {}
```

Used when operations are aborted via the abort controller.

### Internal CLI Flags

The SDK passes these flags to the Claude Code CLI:

```javascript
const cliFlags = [
  "--output-format", "stream-json",
  "--verbose",
  "--input-format", "stream-json",
  "--system-prompt",           // Custom system prompt
  "--append-system-prompt",    // Append to default prompt
  "--max-thinking-tokens",     // Limit thinking tokens
  "--max-turns",               // Maximum conversation turns
  "--model",                   // Model selection
  "--debug-to-stderr",         // Debug output
  "--permission-prompt-tool",  // Permission handler
  "--continue",                // Continue conversation
  "--resume",                  // Resume session ID
  "--allowedTools",            // Comma-separated tool list
  "--disallowedTools",         // Comma-separated tool list
  "--mcp-config",              // JSON MCP config
  "--agents",                  // JSON agents config
  "--setting-sources",         // Comma-separated sources
  "--strict-mcp-config",       // Fail on MCP errors
  "--permission-mode",         // Permission mode
  "--fallback-model",          // Fallback model
  "--include-partial-messages",// Include streaming events
  "--add-dir",                 // Additional directories
  "--fork-session",            // Fork resumed session
  "--resume-session-at"        // Resume from message ID
];
```

### Actual Function Implementations

#### tool()
```javascript
function tool(name, description, inputSchema, handler) {
  return { name, description, inputSchema, handler };
}
```

Simple factory function that creates a tool definition object.

#### createSdkMcpServer()
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

Creates an MCP server instance and registers all provided tools.

#### query()
```javascript
function query({ prompt, options }) {
  const { systemPrompt, settingSources, ...rest } = options ?? {};

  let customSystemPrompt;
  let appendSystemPrompt;

  if (systemPrompt === undefined) {
    customSystemPrompt = "";
  } else if (typeof systemPrompt === "string") {
    customSystemPrompt = systemPrompt;
  } else if (systemPrompt.type === "preset") {
    appendSystemPrompt = systemPrompt.append;
  }

  let pathToClaudeCodeExecutable = rest.pathToClaudeCodeExecutable;
  if (!pathToClaudeCodeExecutable) {
    const filename = fileURLToPath(import.meta.url);
    const dirname2 = join3(filename, "..");
    pathToClaudeCodeExecutable = join3(dirname2, "cli.js");
  }

  process.env.CLAUDE_AGENT_SDK_VERSION = "0.1.22";

  return createSharedQuery({
    prompt,
    options: {
      ...rest,
      pathToClaudeCodeExecutable,
      customSystemPrompt,
      appendSystemPrompt,
      settingSources: settingSources ?? []
    }
  });
}
```

Main entry point that:
1. Processes system prompt options
2. Resolves path to Claude Code CLI executable
3. Sets SDK version environment variable
4. Creates and returns a Query instance via `createSharedQuery()`

### Runtime Executable Detection

```javascript
function isRunningWithBun() {
  return process.versions.bun !== undefined;
}

// Used to select default executable
executable: isRunningWithBun() ? "bun" : "node"
```

The SDK automatically detects if running under Bun and adjusts accordingly.

### Permission Prompt Tool Modes

From the code:
```javascript
if (canUseTool) {
  if (permissionPromptToolName) {
    throw new Error(
      "canUseTool callback cannot be used with permissionPromptToolName. " +
      "Please use one or the other."
    );
  }
  args.push("--permission-prompt-tool", "stdio");
} else if (permissionPromptToolName) {
  args.push("--permission-prompt-tool", permissionPromptToolName);
}
```

You can either use:
- `canUseTool` callback → Uses stdio communication
- `permissionPromptToolName` → Uses a named tool

But NOT both at the same time.

### MCP Configuration Validation

```javascript
if (mcpServers && Object.keys(mcpServers).length > 0) {
  args.push("--mcp-config", JSON.stringify({ mcpServers }));
}
```

MCP servers are passed as JSON-stringified configuration.

### Model Validation

```javascript
if (fallbackModel) {
  if (model && fallbackModel === model) {
    throw new Error(
      "Fallback model cannot be the same as the main model. " +
      "Please specify a different model for fallbackModel option."
    );
  }
  args.push("--fallback-model", fallbackModel);
}
```

The SDK enforces that fallback model must be different from the main model.

### Additional Implementation Classes

Found in the bundled code:
- **SdkControlServerTransport**: Handles SDK control commands
- **McpServer**: MCP protocol server implementation
- **McpError**: MCP-specific errors
- **ParseInputLazyPath**: Lazy path parsing
- **ParseStatus**: Parse status tracking
- **Completable**: Async completion handling
- **Stream**: Streaming utilities

### Zod Integration

The SDK uses Zod for schema validation (all Zod classes bundled):
- ZodObject, ZodString, ZodNumber, ZodBoolean
- ZodArray, ZodTuple, ZodRecord, ZodMap, ZodSet
- ZodEnum, ZodNativeEnum, ZodLiteral
- ZodUnion, ZodDiscriminatedUnion, ZodIntersection
- ZodOptional, ZodNullable, ZodDefault
- ZodPromise, ZodFunction
- ZodEffects, ZodPipeline
- And more...

---

## Summary

The Claude Agent SDK provides a comprehensive platform for building autonomous AI agents with:

### Core Capabilities
- **Query Interface**: Async generator pattern for streaming interaction
- **Agent System**: Define specialized sub-agents with custom prompts and tool access
- **17+ Built-in Tools**: Complete file, shell, search, and web capabilities
- **Hook System**: 9 event types for intercepting and modifying behavior
- **Permission System**: 4 modes with fine-grained control and runtime updates
- **MCP Integration**: 4 transport types + custom in-process tools
- **Session Management**: Resume, fork, and manage long-running sessions
- **Usage Tracking**: Per-model token and cost tracking
- **Runtime Control**: Change models, permissions, and parameters during execution
- **Error Handling**: Comprehensive error types and recovery

### Key Features
- **Streaming**: Real-time message streaming with partial updates
- **Multi-Model**: Use different models for different agents
- **Custom Tools**: Create in-process MCP tools with Zod schemas
- **Hooks**: Intercept tool use, user input, and session lifecycle
- **Permissions**: Custom handlers and rule-based control
- **Context Management**: Automatic compaction with hooks
- **Type Safety**: Full TypeScript definitions
- **Flexibility**: Extensive configuration options

### Use Cases
- Autonomous code analysis and refactoring
- Multi-step workflows with specialized agents
- Interactive development assistants
- Code review and security auditing
- Test generation and quality assurance
- Documentation generation
- Custom development tools with MCP integration

---

*Documentation generated from comprehensive source code analysis of @anthropic-ai/claude-agent-sdk v0.1.22*

*Validated against actual minified implementation in `sdk.mjs` and TypeScript definitions*

