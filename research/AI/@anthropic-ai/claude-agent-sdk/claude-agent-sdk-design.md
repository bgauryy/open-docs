# Claude Agent SDK - Design Architecture Documentation

**Package:** `@anthropic-ai/claude-agent-sdk`  
**Version:** 0.1.22  
**License:** SEE LICENSE IN README.md  
**Node:** >= 18.0.0  

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Core Components](#core-components)
4. [Type System](#type-system)
5. [Tool System](#tool-system)
6. [Hook System](#hook-system)
7. [MCP Integration](#mcp-integration)
8. [Message Flow](#message-flow)
9. [Permission System](#permission-system)
10. [Agent System](#agent-system)
11. [Bundled Dependencies](#bundled-dependencies)
12. [Design Patterns](#design-patterns)
13. [API Surface](#api-surface)

---

## Overview

The Claude Agent SDK is a TypeScript/JavaScript SDK that enables programmatic creation of AI agents with Claude Code's capabilities. It provides a comprehensive framework for building autonomous agents that can understand codebases, edit files, run commands, and execute complex workflows.

### Key Features

- **Programmatic Agent Control**: Create and control Claude-powered agents via API
- **Tool System**: Rich set of built-in tools (bash, file operations, web search, etc.)
- **Hook System**: Lifecycle hooks for monitoring and controlling agent behavior
- **MCP Integration**: Full Model Context Protocol support for custom tools and resources
- **Permission Management**: Granular permission control with multiple modes
- **Streaming Support**: Async generator-based streaming for real-time interaction
- **Sub-agent Support**: Hierarchical agent execution with specialized agent types
- **Session Management**: Resume, fork, and manage conversation sessions

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     User Application                         │
│  (Uses claude-agent-sdk to create agents)                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  SDK Layer (sdk.mjs)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ query()      │  │ tool()       │  │ createSdk    │      │
│  │ function     │  │ function     │  │ McpServer()  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                  CLI Process (cli.js)                        │
│  - Main execution engine (9.3MB bundled)                    │
│  - Tool execution                                            │
│  - Message handling                                          │
│  - Anthropic API integration                                 │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Claude API (Anthropic Backend)                  │
└─────────────────────────────────────────────────────────────┘
```

### Process Model

The SDK uses a **subprocess architecture**:

1. **Parent Process**: User application with SDK
2. **Child Process**: CLI executable (`cli.js`) spawned as subprocess
3. **Communication**: STDIO-based JSON streaming between parent and child
4. **MCP Servers**: Additional processes for custom tools (optional)

### File Structure

```
@anthropic-ai/claude-agent-sdk/
├── cli.js                 # Main CLI executable (9.3MB) - bundled implementation
├── sdk.mjs                # SDK implementation (521KB) - main API
├── sdk.d.ts               # Main SDK TypeScript definitions
├── sdk-tools.d.ts         # Tool input schemas (7.1KB)
├── sdkTypes.d.ts          # Comprehensive type definitions (15KB)
├── yoga.wasm              # Layout engine for UI rendering (86KB)
├── LICENSE.md             # License information
├── README.md              # Documentation
├── package.json           # Package manifest
└── vendor/                # Bundled vendor dependencies
    ├── claude-code-jetbrains-plugin/  # JetBrains IDE integration
    │   └── lib/           # Kotlin/Ktor JARs (35 files, ~12MB)
    └── ripgrep/           # Fast search tool
        ├── arm64-darwin/  # macOS ARM binaries
        ├── arm64-linux/   # Linux ARM binaries
        ├── x64-darwin/    # macOS x64 binaries
        ├── x64-linux/     # Linux x64 binaries
        └── x64-win32/     # Windows x64 binaries
```

---

## Core Components

### 1. Query Function

The main entry point for creating agent sessions.

```typescript
function query({
  prompt: string | AsyncIterable<SDKUserMessage>,
  options?: Options
}): Query
```

**Returns**: `Query` object (AsyncGenerator) that yields `SDKMessage` events

**Key Features**:
- Accepts string prompts or streaming message iterables
- Returns async generator for message streaming
- Supports control requests (interrupt, setPermissionMode, etc.)
- Session management (resume, fork)

### 2. Query Interface

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

### 3. Options Configuration

Comprehensive configuration object for agent behavior:

```typescript
type Options = {
  // Execution Control
  abortController?: AbortController;
  maxTurns?: number;
  maxThinkingTokens?: number;
  continue?: boolean;
  
  // Tools & Permissions
  allowedTools?: string[];
  disallowedTools?: string[];
  canUseTool?: CanUseTool;
  permissionMode?: PermissionMode;
  permissionPromptToolName?: string;
  
  // Environment
  cwd?: string;
  env?: Record<string, string | undefined>;
  additionalDirectories?: string[];
  
  // System Prompts
  customSystemPrompt?: string;
  appendSystemPrompt?: string;
  
  // Hooks
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;
  
  // MCP Servers
  mcpServers?: Record<string, McpServerConfig>;
  strictMcpConfig?: boolean;
  
  // Session Management
  resume?: string;
  forkSession?: boolean;
  resumeSessionAt?: string;
  
  // Models
  model?: string;
  fallbackModel?: string;
  
  // Streaming
  includePartialMessages?: boolean;
  stderr?: (data: string) => void;
  
  // Advanced
  executable?: 'bun' | 'deno' | 'node';
  executableArgs?: string[];
  extraArgs?: Record<string, string | null>;
  pathToClaudeCodeExecutable?: string;
};
```

---

## Type System

### Message Types

The SDK uses a rich message type system for communication:

#### 1. SDKUserMessage
User input messages from the application or user.

```typescript
type SDKUserMessage = {
  type: 'user';
  uuid?: UUID;
  session_id: string;
  message: APIUserMessage;
  parent_tool_use_id: string | null;
  isSynthetic?: boolean;  // Generated by system, not user
};
```

#### 2. SDKAssistantMessage
Claude's responses including text and tool uses.

```typescript
type SDKAssistantMessage = {
  type: 'assistant';
  uuid: UUID;
  session_id: string;
  message: APIAssistantMessage;
  parent_tool_use_id: string | null;
};
```

#### 3. SDKSystemMessage
System initialization and status messages.

```typescript
type SDKSystemMessage = {
  type: 'system';
  subtype: 'init';
  uuid: UUID;
  session_id: string;
  agents?: string[];
  apiKeySource: ApiKeySource;
  claude_code_version: string;
  cwd: string;
  tools: string[];
  mcp_servers: Array<{name: string; status: string}>;
  model: string;
  permissionMode: PermissionMode;
  slash_commands: string[];
  output_style: string;
};
```

#### 4. SDKResultMessage
Final execution results with usage statistics.

```typescript
type SDKResultMessage = {
  type: 'result';
  subtype: 'success' | 'error_max_turns' | 'error_during_execution';
  uuid: UUID;
  session_id: string;
  duration_ms: number;
  duration_api_ms: number;
  is_error: boolean;
  num_turns: number;
  result?: string;  // Only for success
  total_cost_usd: number;
  usage: NonNullableUsage;
  modelUsage: Record<string, ModelUsage>;
  permission_denials: SDKPermissionDenial[];
};
```

#### 5. SDKPartialAssistantMessage
Streaming events for real-time updates.

```typescript
type SDKPartialAssistantMessage = {
  type: 'stream_event';
  uuid: UUID;
  session_id: string;
  event: RawMessageStreamEvent;
  parent_tool_use_id: string | null;
};
```

#### 6. SDKCompactBoundaryMessage
Marks conversation compaction points.

```typescript
type SDKCompactBoundaryMessage = {
  type: 'system';
  subtype: 'compact_boundary';
  uuid: UUID;
  session_id: string;
  compact_metadata: {
    trigger: 'manual' | 'auto';
    pre_tokens: number;
  };
};
```

### Usage Tracking Types

```typescript
type ModelUsage = {
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
  webSearchRequests: number;
  costUSD: number;
  contextWindow: number;
};
```

---

## Tool System

### Built-in Tools

The SDK provides 15 built-in tools for agent operations:

#### 1. **Agent** - Sub-agent delegation
- **Purpose**: Delegate tasks to specialized sub-agents
- **Input**: `{ description: string, prompt: string, subagent_type: string }`

#### 2. **Bash** - Command execution
- **Purpose**: Execute shell commands
- **Input**: `{ command: string, timeout?: number, description?: string, run_in_background?: boolean }`

#### 3. **BashOutput** - Background output retrieval
- **Purpose**: Get output from background bash processes
- **Input**: `{ bash_id: string, filter?: string }`

#### 4. **ExitPlanMode** - Plan approval
- **Purpose**: Present plans to users for approval
- **Input**: `{ plan: string }`

#### 5. **FileEdit** - File modification
- **Purpose**: Search and replace in files
- **Input**: `{ file_path: string, old_string: string, new_string: string, replace_all?: boolean }`

#### 6. **FileRead** - File reading
- **Purpose**: Read file contents with optional pagination
- **Input**: `{ file_path: string, offset?: number, limit?: number }`

#### 7. **FileWrite** - File creation/overwrite
- **Purpose**: Write content to files
- **Input**: `{ file_path: string, content: string }`

#### 8. **Glob** - File pattern matching
- **Purpose**: Find files matching patterns
- **Input**: `{ pattern: string, path?: string }`

#### 9. **Grep** - Content search
- **Purpose**: Search file contents with regex
- **Input**: Multiple options including pattern, path, glob, output_mode, context lines, etc.

#### 10. **KillShell** - Process termination
- **Purpose**: Kill background shell processes
- **Input**: `{ shell_id: string }`

#### 11. **ListMcpResources** - MCP resource discovery
- **Purpose**: List available MCP resources
- **Input**: `{ server?: string }`

#### 12. **Mcp** - Custom MCP tool invocation
- **Purpose**: Call custom MCP tools
- **Input**: Dynamic based on tool schema

#### 13. **NotebookEdit** - Jupyter notebook editing
- **Purpose**: Edit Jupyter notebook cells
- **Input**: `{ notebook_path: string, cell_id?: string, new_source: string, cell_type?: 'code'|'markdown', edit_mode?: 'replace'|'insert'|'delete' }`

#### 14. **ReadMcpResource** - MCP resource fetching
- **Purpose**: Fetch MCP resource contents
- **Input**: `{ server: string, uri: string }`

#### 15. **TodoWrite** - Task management
- **Purpose**: Update TODO list
- **Input**: `{ todos: Array<{content: string, status: string, activeForm: string}> }`

#### 16. **WebFetch** - Web content retrieval
- **Purpose**: Fetch and analyze web content
- **Input**: `{ url: string, prompt: string }`

#### 17. **WebSearch** - Web searching
- **Purpose**: Search the web with domain filtering
- **Input**: `{ query: string, allowed_domains?: string[], blocked_domains?: string[] }`

### Tool Definition API

For creating custom tools via MCP:

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
const myTool = tool(
  'greet',
  'Greet a user by name',
  { name: z.string() },
  async (args) => ({
    content: [{ type: 'text', text: `Hello, ${args.name}!` }]
  })
);
```

---

## Hook System

Hooks provide lifecycle event monitoring and control.

### Hook Events

```typescript
const HOOK_EVENTS = [
  'PreToolUse',        // Before tool execution
  'PostToolUse',       // After tool execution
  'Notification',      // System notifications
  'UserPromptSubmit',  // User message submission
  'SessionStart',      // Session initialization
  'SessionEnd',        // Session termination
  'Stop',              // Agent stopped
  'SubagentStop',      // Sub-agent stopped
  'PreCompact'         // Before conversation compaction
] as const;
```

### Hook Input Types

Each hook receives specific input:

#### PreToolUseHookInput
```typescript
type PreToolUseHookInput = {
  hook_event_name: 'PreToolUse';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  tool_name: string;
  tool_input: unknown;
};
```

#### PostToolUseHookInput
```typescript
type PostToolUseHookInput = {
  hook_event_name: 'PostToolUse';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;
};
```

### Hook Output Types

Hooks can return:

#### Async Hook (long-running)
```typescript
type AsyncHookJSONOutput = {
  async: true;
  asyncTimeout?: number;
};
```

#### Sync Hook (immediate)
```typescript
type SyncHookJSONOutput = {
  continue?: boolean;
  suppressOutput?: boolean;
  stopReason?: string;
  decision?: 'approve' | 'block';
  systemMessage?: string;
  reason?: string;
  hookSpecificOutput?: {
    hookEventName: 'PreToolUse';
    permissionDecision?: 'allow' | 'deny' | 'ask';
    permissionDecisionReason?: string;
    updatedInput?: Record<string, unknown>;
  } | {
    hookEventName: 'UserPromptSubmit' | 'SessionStart' | 'PostToolUse';
    additionalContext?: string;
  };
};
```

### Hook Registration

```typescript
const options: Options = {
  hooks: {
    PreToolUse: [{
      matcher: 'bash',  // Optional: match specific tool
      hooks: [
        async (input, toolUseID, { signal }) => {
          console.log('About to execute:', input.tool_name);
          return { continue: true };
        }
      ]
    }]
  }
};
```

---

## MCP Integration

### MCP Server Configurations

Four transport types supported:

#### 1. STDIO (Process-based)
```typescript
type McpStdioServerConfig = {
  type?: 'stdio';
  command: string;
  args?: string[];
  env?: Record<string, string>;
};
```

#### 2. SSE (Server-Sent Events)
```typescript
type McpSSEServerConfig = {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
};
```

#### 3. HTTP
```typescript
type McpHttpServerConfig = {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
};
```

#### 4. SDK (In-process)
```typescript
type McpSdkServerConfig = {
  type: 'sdk';
  name: string;
  instance: McpServer;
};
```

### Creating SDK MCP Servers

```typescript
function createSdkMcpServer(options: {
  name: string;
  version?: string;
  tools?: Array<SdkMcpToolDefinition<any>>;
}): McpSdkServerConfigWithInstance
```

**Example**:
```typescript
const customServer = createSdkMcpServer({
  name: 'my-tools',
  version: '1.0.0',
  tools: [
    tool('echo', 'Echo a message', { text: z.string() }, async (args) => ({
      content: [{ type: 'text', text: args.text }]
    }))
  ]
});

const q = query({
  prompt: 'Hello',
  options: {
    mcpServers: {
      'my-tools': customServer
    }
  }
});
```

---

## Message Flow

### Typical Execution Flow

```
1. User Application
   └─> query({ prompt, options })
        └─> Spawns CLI process (cli.js)
             └─> Sends init message
                  ↓
2. CLI Process
   └─> Initializes environment
   └─> Loads tools
   └─> Connects to MCP servers
   └─> Sends SDKSystemMessage (init)
        ↓
3. Streaming Loop
   ├─> User sends prompt
   │    └─> SDKUserMessage
   │         ↓
   ├─> Claude generates response
   │    └─> SDKPartialAssistantMessage (streaming)
   │    └─> SDKAssistantMessage (complete)
   │         ↓
   ├─> Tool execution requested
   │    └─> PreToolUse hook (optional)
   │    └─> Permission check
   │    └─> Tool execution
   │    └─> PostToolUse hook (optional)
   │    └─> Result back to Claude
   │         ↓
   └─> Repeat until complete
        ↓
4. Completion
   └─> SDKResultMessage (with usage stats)
```

### Control Request Flow

```
User Application                CLI Process
     │                               │
     ├─> interrupt()                 │
     │      └─────────────────────>  │
     │                              Stop current turn
     │                               │
     ├─> setPermissionMode('plan')   │
     │      └─────────────────────>  │
     │                              Update permission mode
     │                               │
     ├─> setModel('opus')            │
     │      └─────────────────────>  │
     │                              Switch model
     │                               │
     └─> accountInfo()               │
            └─────────────────────>  │
            <─────────────────────   │
              Return account info
```

---

## Permission System

### Permission Modes

```typescript
type PermissionMode = 
  | 'default'            // Ask for permission on sensitive operations
  | 'acceptEdits'        // Auto-accept file edits, ask for others
  | 'bypassPermissions'  // Execute all tools without asking
  | 'plan';              // Planning mode, no execution
```

### Permission Behaviors

```typescript
type PermissionBehavior = 'allow' | 'deny' | 'ask';
```

### Permission Rules

```typescript
type PermissionRuleValue = {
  toolName: string;
  ruleContent?: string;  // Tool-specific rule (e.g., file path pattern)
};
```

### Permission Updates

Six update types:

1. **addRules**: Add permission rules
2. **replaceRules**: Replace existing rules
3. **removeRules**: Remove specific rules
4. **setMode**: Change permission mode
5. **addDirectories**: Add allowed directories
6. **removeDirectories**: Remove allowed directories

### Custom Permission Callback

```typescript
type CanUseTool = (
  toolName: string,
  input: Record<string, unknown>,
  options: {
    signal: AbortSignal;
    suggestions?: PermissionUpdate[];
  }
) => Promise<PermissionResult>;

type PermissionResult = {
  behavior: 'allow';
  updatedInput: Record<string, unknown>;
  updatedPermissions?: PermissionUpdate[];
} | {
  behavior: 'deny';
  message: string;
  interrupt?: boolean;
};
```

**Example**:
```typescript
const canUseTool: CanUseTool = async (toolName, input, { suggestions }) => {
  if (toolName === 'bash') {
    const command = input.command as string;
    if (command.includes('rm -rf')) {
      return {
        behavior: 'deny',
        message: 'Dangerous command blocked',
        interrupt: false
      };
    }
  }
  return {
    behavior: 'allow',
    updatedInput: input
  };
};
```

---

## Agent System

### Agent Definitions

Custom agents with specialized behavior:

```typescript
type AgentDefinition = {
  description: string;
  tools?: string[];  // Restrict to specific tools
  prompt: string;    // Agent-specific system prompt
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
```

### System Prompt Configuration

Two configuration styles:

#### 1. Custom System Prompt
```typescript
type Options = {
  systemPrompt?: string | {
    type: 'preset';
    preset: 'claude_code';
    append?: string;
  };
};
```

#### 2. Legacy (deprecated)
```typescript
type Options = {
  customSystemPrompt?: string;
  appendSystemPrompt?: string;
};
```

### Sub-agent Execution

Agents can delegate to specialized sub-agents:

```typescript
// Parent agent delegates to sub-agent
const parentQuery = query({
  prompt: 'Analyze and fix the bug',
  options: {
    agents: {
      'debugger': {
        description: 'Specialized debugging agent',
        tools: ['bash', 'file_read', 'grep'],
        prompt: 'You are an expert debugger. Focus on finding root causes.',
        model: 'opus'
      }
    }
  }
});
```

Sub-agents:
- Have their own tool restrictions
- Can use different models
- Receive specialized system prompts
- Report back to parent agent

---

## Bundled Dependencies

### 1. Ripgrep

Fast search tool bundled for all platforms:

- **Purpose**: High-performance content searching (used by Grep tool)
- **Platforms**: macOS (ARM/x64), Linux (ARM/x64), Windows (x64)
- **Size**: ~24MB total (4-6MB per platform)
- **Components**:
  - `rg` binary: Command-line executable
  - `ripgrep.node`: Native Node.js binding

### 2. Claude Code JetBrains Plugin

Integration for JetBrains IDEs:

- **Purpose**: IDE integration for Claude Code
- **Technology**: Kotlin with Ktor framework
- **Size**: ~12MB (35 JAR files)
- **Components**:
  - Core plugin JAR
  - Kotlin standard library
  - Ktor HTTP client/server
  - Kotlin coroutines
  - Serialization libraries

### 3. Yoga Layout Engine

Facebook's Yoga layout engine:

- **Purpose**: Flexbox layout calculations for UI rendering
- **Format**: WebAssembly (WASM)
- **Size**: 86KB
- **Use Case**: Terminal UI layout

---

## Design Patterns

### 1. Process Isolation

**Pattern**: Parent-child process architecture
- **Benefit**: Isolation, stability, easier termination
- **Trade-off**: IPC overhead, larger memory footprint

### 2. Streaming Architecture

**Pattern**: AsyncGenerator for message streaming
- **Benefit**: Real-time updates, low memory usage
- **Implementation**: `for await (const message of query) { ... }`

### 3. Event-Driven Hooks

**Pattern**: Observer pattern for lifecycle events
- **Benefit**: Extensibility without modifying core
- **Use Cases**: Logging, analytics, custom permission logic

### 4. Type-Safe Tool System

**Pattern**: Zod schemas for runtime validation
- **Benefit**: Runtime safety + TypeScript types
- **Implementation**: JSON Schema → TypeScript types

### 5. MCP Protocol Abstraction

**Pattern**: Multiple transport implementations behind unified interface
- **Benefit**: Flexibility (STDIO, HTTP, SSE, in-process)
- **Use Cases**: Different deployment scenarios

### 6. Capability-Based Security

**Pattern**: Explicit permission system with multiple modes
- **Benefit**: Fine-grained control over agent actions
- **Layers**: Mode → Rules → Callbacks

### 7. Session Continuity

**Pattern**: Stateful sessions with resume/fork capability
- **Benefit**: Long-running workflows, recovery from interruptions
- **Storage**: Transcript files with message history

---

## API Surface

### Main Exports (sdk.d.ts)

```typescript
// Primary function
export function query(params: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;

// Agent definitions
export type AgentDefinition = { ... };
export type SettingSource = 'user' | 'project' | 'local';
export type Options = { ... };

// All types from sdkTypes.js
export type {
  // Usage & Tracking
  NonNullableUsage,
  ModelUsage,
  ApiKeySource,
  ConfigScope,
  
  // MCP
  McpStdioServerConfig,
  McpSSEServerConfig,
  McpHttpServerConfig,
  McpSdkServerConfig,
  McpSdkServerConfigWithInstance,
  McpServerConfig,
  McpServerConfigForProcessTransport,
  
  // Permissions
  PermissionBehavior,
  PermissionUpdate,
  PermissionResult,
  PermissionRuleValue,
  CanUseTool,
  PermissionMode,
  
  // Hooks
  HookEvent,
  HookCallback,
  HookCallbackMatcher,
  BaseHookInput,
  PreToolUseHookInput,
  PostToolUseHookInput,
  NotificationHookInput,
  UserPromptSubmitHookInput,
  SessionStartHookInput,
  StopHookInput,
  SubagentStopHookInput,
  PreCompactHookInput,
  SessionEndHookInput,
  HookInput,
  AsyncHookJSONOutput,
  SyncHookJSONOutput,
  HookJSONOutput,
  ExitReason,
  
  // Messages
  SDKMessageBase,
  SDKUserMessage,
  SDKUserMessageReplay,
  SDKAssistantMessage,
  SDKPermissionDenial,
  SDKResultMessage,
  SDKSystemMessage,
  SDKPartialAssistantMessage,
  SDKCompactBoundaryMessage,
  SDKMessage,
  
  // Metadata
  SlashCommand,
  ModelInfo,
  McpServerStatus,
  
  // Query
  Query,
};

// Constants & utilities
export {
  HOOK_EVENTS,
  EXIT_REASONS,
  tool,
  createSdkMcpServer,
  AbortError,
};
```

### Tool Schemas (sdk-tools.d.ts)

Auto-generated TypeScript definitions for all tool inputs from JSON schemas.

```typescript
export type ToolInputSchemas =
  | AgentInput
  | BashInput
  | BashOutputInput
  | ExitPlanModeInput
  | FileEditInput
  | FileReadInput
  | FileWriteInput
  | GlobInput
  | GrepInput
  | KillShellInput
  | ListMcpResourcesInput
  | McpInput
  | NotebookEditInput
  | ReadMcpResourceInput
  | TodoWriteInput
  | WebFetchInput
  | WebSearchInput;

// Each input type has full interface definition
export interface BashInput {
  command: string;
  timeout?: number;
  description?: string;
  run_in_background?: boolean;
}

// ... (all other tool inputs)
```

---

## Key Design Decisions

### 1. Why Subprocess Architecture?

**Decision**: Run CLI as separate process instead of in-process
**Rationale**:
- **Isolation**: Crashes in CLI don't affect user application
- **Termination**: Can kill subprocess cleanly
- **Resource Management**: Easier to track/limit resources
- **Compatibility**: CLI can use native modules without affecting parent

### 2. Why AsyncGenerator for Query?

**Decision**: Return AsyncGenerator instead of Promise<Result>
**Rationale**:
- **Streaming**: Real-time updates during execution
- **Memory Efficient**: Don't buffer entire conversation
- **Cancellable**: Can stop iteration anytime
- **Flexible**: Can filter/transform messages in flight

### 3. Why Bundle Ripgrep?

**Decision**: Include ripgrep binaries for all platforms
**Rationale**:
- **Reliability**: Guaranteed to have fast search
- **Consistency**: Same behavior across platforms
- **Performance**: Critical for Grep tool performance
- **User Experience**: No external dependencies required

### 4. Why Hooks vs Middleware?

**Decision**: Event-based hooks instead of middleware chain
**Rationale**:
- **Simplicity**: Easier mental model
- **Flexibility**: Can have multiple hooks per event
- **Async Support**: Natural async handling
- **Matching**: Can filter by tool name

### 5. Why Multiple Permission Modes?

**Decision**: Four permission modes instead of boolean
**Rationale**:
- **Flexibility**: Different use cases need different safety levels
- **UX**: 'acceptEdits' reduces prompts while staying safe
- **Development**: 'bypassPermissions' for testing/automation
- **Planning**: 'plan' mode for cost-effective design phase

### 6. Why MCP Integration?

**Decision**: Full MCP support with multiple transports
**Rationale**:
- **Extensibility**: Users can add custom tools without forking
- **Standard**: MCP is emerging standard for AI tools
- **Ecosystem**: Leverage existing MCP servers
- **Flexibility**: Different transports for different scenarios

### 7. Why Include JetBrains Plugin?

**Decision**: Bundle JetBrains plugin JARs
**Rationale**:
- **IDE Integration**: Native IDE support
- **Distribution**: Single package for all capabilities
- **Version Lock**: Ensure compatibility between SDK and plugin

---

## Usage Patterns

### Basic Usage

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const q = query({
  prompt: 'Create a hello world script',
  options: {
    cwd: '/path/to/project'
  }
});

for await (const message of q) {
  if (message.type === 'assistant') {
    console.log('Assistant:', message.message);
  } else if (message.type === 'result') {
    console.log('Done! Cost: $', message.total_cost_usd);
    console.log('Tokens:', message.usage);
  }
}
```

### Advanced Usage with Hooks

```typescript
import { query, HOOK_EVENTS } from '@anthropic-ai/claude-agent-sdk';

const q = query({
  prompt: 'Analyze the codebase',
  options: {
    hooks: {
      PreToolUse: [{
        hooks: [async (input) => {
          console.log(`Tool: ${input.tool_name}`);
          // Log all tool uses
          return { continue: true };
        }]
      }],
      PostToolUse: [{
        hooks: [async (input) => {
          console.log(`Result: ${input.tool_response}`);
          return {};
        }]
      }]
    },
    maxTurns: 20,
    permissionMode: 'acceptEdits'
  }
});

for await (const message of q) {
  // Handle messages
}
```

### Custom MCP Tools

```typescript
import { query, tool, createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

const myServer = createSdkMcpServer({
  name: 'custom-tools',
  version: '1.0.0',
  tools: [
    tool(
      'database_query',
      'Query the database',
      { sql: z.string() },
      async (args) => {
        const result = await db.query(args.sql);
        return {
          content: [{ 
            type: 'text', 
            text: JSON.stringify(result) 
          }]
        };
      }
    )
  ]
});

const q = query({
  prompt: 'Get all users from database',
  options: {
    mcpServers: {
      'custom-tools': myServer
    }
  }
});
```

### Session Management

```typescript
// Start session
const q1 = query({
  prompt: 'Start working on feature X',
  options: { cwd: '/project' }
});

let sessionId: string;
for await (const message of q1) {
  if (message.type === 'system' && message.subtype === 'init') {
    sessionId = message.session_id;
  }
}

// Resume session later
const q2 = query({
  prompt: 'Continue working on feature X',
  options: {
    cwd: '/project',
    resume: sessionId
  }
});
```

### Permission Control

```typescript
import { query, type CanUseTool } from '@anthropic-ai/claude-agent-sdk';

const canUseTool: CanUseTool = async (toolName, input, { suggestions }) => {
  // Custom permission logic
  if (toolName === 'file_write') {
    const path = input.file_path as string;
    if (path.includes('config')) {
      // Block config file writes
      return {
        behavior: 'deny',
        message: 'Cannot modify config files',
        interrupt: false
      };
    }
  }
  
  // Allow with suggestions
  return {
    behavior: 'allow',
    updatedInput: input,
    updatedPermissions: suggestions
  };
};

const q = query({
  prompt: 'Update the application',
  options: {
    canUseTool,
    permissionMode: 'default'
  }
});
```

---

## Performance Considerations

### Token Usage

- **Cost Tracking**: Every message includes cost and token usage
- **Model Selection**: Choose appropriate model (haiku/sonnet/opus)
- **Context Management**: Automatic compaction for long conversations
- **Cache Utilization**: Prompt caching for repeated context

### Memory Management

- **Streaming**: Messages not buffered, processed as they arrive
- **Partial Messages**: Optional via `includePartialMessages`
- **Session Storage**: Transcript files can grow large

### Process Overhead

- **Startup Time**: CLI process spawn adds latency (~500ms-2s)
- **IPC Cost**: JSON serialization for every message
- **Mitigation**: Reuse sessions with `resume` option

---

## Security Considerations

### 1. Permission System
- Always use appropriate `permissionMode` for your use case
- Implement custom `canUseTool` for sensitive operations
- Review permission rules before auto-allowing

### 2. Tool Restrictions
- Use `allowedTools` to limit agent capabilities
- Use `disallowedTools` to block specific tools
- Configure MCP servers with `strictMcpConfig: true`

### 3. File System Access
- Limit with `additionalDirectories`
- Review all `file_write` operations
- Be cautious with `bash` tool

### 4. Environment Variables
- Be careful passing sensitive env vars via `env` option
- Subprocess inherits parent environment by default

### 5. MCP Servers
- Validate MCP server sources
- Use HTTPS for remote MCP servers
- Implement authentication for MCP endpoints

---

## Migration Notes

### From Claude Code SDK

The package was renamed from `@anthropic-ai/claude-code-sdk` to `@anthropic-ai/claude-agent-sdk`.

**Breaking Changes**:
1. Package name changed
2. Some option names may have changed
3. Refer to migration guide: https://docs.claude.com/en/docs/claude-code/sdk/migration-guide

---

## Conclusion

The Claude Agent SDK provides a comprehensive, type-safe framework for building AI agents with Claude Code's capabilities. Its architecture emphasizes:

- **Safety**: Multiple permission layers and isolated execution
- **Flexibility**: Extensive configuration and extensibility via MCP
- **Developer Experience**: Type safety, streaming, hooks
- **Performance**: Efficient tools (ripgrep), cost tracking, session management
- **Reliability**: Process isolation, error handling, session recovery

The SDK is production-ready for building autonomous AI agents that can understand codebases, execute commands, edit files, and complete complex workflows programmatically.

---

## References

- **Official Documentation**: https://docs.claude.com/en/api/agent-sdk/overview
- **GitHub Repository**: https://github.com/anthropics/claude-agent-sdk-typescript
- **Discord Community**: https://anthropic.com/discord
- **Migration Guide**: https://docs.claude.com/en/docs/claude-code/sdk/migration-guide
- **Data Usage Policies**: https://docs.anthropic.com/en/docs/claude-code/data-usage

