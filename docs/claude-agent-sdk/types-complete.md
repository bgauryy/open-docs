# Claude Agent SDK - Complete Type System Documentation

**SDK Version**: 0.1.22
**Package**: @anthropic-ai/claude-agent-sdk

---

## Table of Contents

1. [Core API Types](#core-api-types)
2. [Message Type System](#message-type-system)
3. [Hook System](#hook-system)
4. [Permission System](#permission-system)
5. [MCP Server Configuration](#mcp-server-configuration)
6. [Tool Input Schemas](#tool-input-schemas)
7. [Agent System](#agent-system)
8. [Model & Usage Types](#model--usage-types)
9. [Query Interface](#query-interface)
10. [Error Types](#error-types)
11. [Type Relationships & Dependencies](#type-relationships--dependencies)
12. [Key Observations](#key-observations)
13. [Usage Examples](#usage-examples)
14. [Summary](#summary)

---

## Core API Types

### Main Query Function

```typescript
function query({
  prompt,
  options,
}: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;
```

### Options Type

The main configuration object for SDK queries:

```typescript
type Options = {
  // Control Flow
  abortController?: AbortController;
  continue?: boolean;
  maxTurns?: number;
  maxThinkingTokens?: number;

  // Session Management
  resume?: string;
  forkSession?: boolean;
  resumeSessionAt?: string;

  // Model Configuration
  model?: string;
  fallbackModel?: string;

  // System Prompts
  customSystemPrompt?: string;
  appendSystemPrompt?: string;

  // Tool Configuration
  allowedTools?: string[];
  disallowedTools?: string[];

  // Permissions
  permissionMode?: PermissionMode;
  canUseTool?: CanUseTool;
  permissionPromptToolName?: string;

  // Environment
  cwd?: string;
  env?: { [envVar: string]: string | undefined };
  executable?: 'bun' | 'deno' | 'node';
  executableArgs?: string[];

  // MCP Servers
  mcpServers?: Record<string, McpServerConfig>;
  strictMcpConfig?: boolean;

  // Hooks
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;

  // Additional
  additionalDirectories?: string[];
  extraArgs?: Record<string, string | null>;
  includePartialMessages?: boolean;
  pathToClaudeCodeExecutable?: string;
  stderr?: (data: string) => void;
};
```

### Enhanced Options (SDK API)

```typescript
type Options = Omit<BaseOptions, 'customSystemPrompt' | 'appendSystemPrompt'> & {
  agents?: Record<string, AgentDefinition>;
  settingSources?: SettingSource[];
  systemPrompt?: string | {
    type: 'preset';
    preset: 'claude_code';
    append?: string;
  };
};
```

---

## Message Type System

### Base Message Type

```typescript
type SDKMessageBase = {
  uuid: UUID;
  session_id: string;
};
```

### User Messages

```typescript
type SDKUserMessage = {
  uuid?: UUID;
  session_id: string;
  type: 'user';
  message: APIUserMessage; // from @anthropic-ai/sdk
  parent_tool_use_id: string | null;
  isSynthetic?: boolean; // True if system-generated, not from user
};
```

```typescript
type SDKUserMessageReplay = SDKMessageBase & {
  type: 'user';
  message: APIUserMessage;
  parent_tool_use_id: string | null;
  isSynthetic?: boolean;
  isReplay: true; // Prevents duplicate messages
};
```

### Assistant Messages

```typescript
type SDKAssistantMessage = SDKMessageBase & {
  type: 'assistant';
  message: APIAssistantMessage; // from @anthropic-ai/sdk
  parent_tool_use_id: string | null;
};
```

### Partial Assistant Messages (Streaming)

```typescript
type SDKPartialAssistantMessage = SDKMessageBase & {
  type: 'stream_event';
  event: RawMessageStreamEvent; // from @anthropic-ai/sdk
  parent_tool_use_id: string | null;
};
```

### Result Messages

Success result:

```typescript
type SDKResultMessage = SDKMessageBase & {
  type: 'result';
  subtype: 'success';
  duration_ms: number;
  duration_api_ms: number;
  is_error: boolean;
  num_turns: number;
  result: string;
  total_cost_usd: number;
  usage: NonNullableUsage;
  modelUsage: { [modelName: string]: ModelUsage };
  permission_denials: SDKPermissionDenial[];
};
```

Error result:

```typescript
type SDKResultMessage = SDKMessageBase & {
  type: 'result';
  subtype: 'error_max_turns' | 'error_during_execution';
  duration_ms: number;
  duration_api_ms: number;
  is_error: boolean;
  num_turns: number;
  total_cost_usd: number;
  usage: NonNullableUsage;
  modelUsage: { [modelName: string]: ModelUsage };
  permission_denials: SDKPermissionDenial[];
};
```

### System Messages

Init message:

```typescript
type SDKSystemMessage = SDKMessageBase & {
  type: 'system';
  subtype: 'init';
  agents?: string[];
  apiKeySource: ApiKeySource;
  claude_code_version: string;
  cwd: string;
  tools: string[];
  mcp_servers: { name: string; status: string }[];
  model: string;
  permissionMode: PermissionMode;
  slash_commands: string[];
  output_style: string;
};
```

Compact boundary message:

```typescript
type SDKCompactBoundaryMessage = SDKMessageBase & {
  type: 'system';
  subtype: 'compact_boundary';
  compact_metadata: {
    trigger: 'manual' | 'auto';
    pre_tokens: number;
  };
};
```

Hook response message:

```typescript
type SDKHookResponseMessage = SDKMessageBase & {
  type: 'system';
  subtype: 'hook_response';
  hook_name: string;
  hook_event: string;
  stdout: string;
  stderr: string;
  exit_code?: number;
};
```

### Permission Denial

```typescript
type SDKPermissionDenial = {
  tool_name: string;
  tool_use_id: string;
  tool_input: Record<string, unknown>;
};
```

### Union Type

```typescript
type SDKMessage =
  | SDKAssistantMessage
  | SDKUserMessage
  | SDKUserMessageReplay
  | SDKResultMessage
  | SDKSystemMessage
  | SDKPartialAssistantMessage
  | SDKCompactBoundaryMessage
  | SDKHookResponseMessage;
```

---

## Hook System

### Hook Events

```typescript
const HOOK_EVENTS = [
  "PreToolUse",
  "PostToolUse",
  "Notification",
  "UserPromptSubmit",
  "SessionStart",
  "SessionEnd",
  "Stop",
  "SubagentStop",
  "PreCompact"
] as const;

type HookEvent = typeof HOOK_EVENTS[number];
```

### Hook Input Types

Base hook input:

```typescript
type BaseHookInput = {
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
};
```

#### PreToolUse Hook

```typescript
type PreToolUseHookInput = BaseHookInput & {
  hook_event_name: 'PreToolUse';
  tool_name: string;
  tool_input: unknown;
};
```

#### PostToolUse Hook

```typescript
type PostToolUseHookInput = BaseHookInput & {
  hook_event_name: 'PostToolUse';
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;
};
```

#### Notification Hook

```typescript
type NotificationHookInput = BaseHookInput & {
  hook_event_name: 'Notification';
  message: string;
  title?: string;
};
```

#### UserPromptSubmit Hook

```typescript
type UserPromptSubmitHookInput = BaseHookInput & {
  hook_event_name: 'UserPromptSubmit';
  prompt: string;
};
```

#### SessionStart Hook

```typescript
type SessionStartHookInput = BaseHookInput & {
  hook_event_name: 'SessionStart';
  source: 'startup' | 'resume' | 'clear' | 'compact';
};
```

#### SessionEnd Hook

```typescript
type SessionEndHookInput = BaseHookInput & {
  hook_event_name: 'SessionEnd';
  reason: ExitReason;
};
```

#### Stop Hook

```typescript
type StopHookInput = BaseHookInput & {
  hook_event_name: 'Stop';
  stop_hook_active: boolean;
};
```

#### SubagentStop Hook

```typescript
type SubagentStopHookInput = BaseHookInput & {
  hook_event_name: 'SubagentStop';
  stop_hook_active: boolean;
};
```

#### PreCompact Hook

```typescript
type PreCompactHookInput = BaseHookInput & {
  hook_event_name: 'PreCompact';
  trigger: 'manual' | 'auto';
  custom_instructions: string | null;
};
```

### Hook Input Union

```typescript
type HookInput =
  | PreToolUseHookInput
  | PostToolUseHookInput
  | NotificationHookInput
  | UserPromptSubmitHookInput
  | SessionStartHookInput
  | SessionEndHookInput
  | StopHookInput
  | SubagentStopHookInput
  | PreCompactHookInput;
```

### Hook Output Types

Async hook output:

```typescript
type AsyncHookJSONOutput = {
  async: true;
  asyncTimeout?: number;
};
```

Sync hook output:

```typescript
type SyncHookJSONOutput = {
  continue?: boolean;
  suppressOutput?: boolean;
  stopReason?: string;
  decision?: 'approve' | 'block';
  systemMessage?: string;
  reason?: string;
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
};
```

Union:

```typescript
type HookJSONOutput = AsyncHookJSONOutput | SyncHookJSONOutput;
```

### Hook Callback Types

```typescript
type HookCallback = (
  input: HookInput,
  toolUseID: string | undefined,
  options: { signal: AbortSignal }
) => Promise<HookJSONOutput>;
```

```typescript
interface HookCallbackMatcher {
  matcher?: string;
  hooks: HookCallback[];
}
```

---

## Permission System

### Permission Modes

```typescript
type PermissionMode =
  | 'default'
  | 'acceptEdits'
  | 'bypassPermissions'
  | 'plan';
```

### Permission Behaviors

```typescript
type PermissionBehavior = 'allow' | 'deny' | 'ask';
```

### Permission Updates

```typescript
type PermissionUpdateDestination =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'session';
```

```typescript
type PermissionUpdate =
  | {
      type: 'addRules';
      rules: PermissionRuleValue[];
      behavior: PermissionBehavior;
      destination: PermissionUpdateDestination;
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

### Permission Rule Value

```typescript
type PermissionRuleValue = {
  toolName: string;
  ruleContent?: string;
};
```

### Permission Result

Allow result:

```typescript
type PermissionResult = {
  behavior: 'allow';
  updatedInput: Record<string, unknown>;
  updatedPermissions?: PermissionUpdate[];
};
```

Deny result:

```typescript
type PermissionResult = {
  behavior: 'deny';
  message: string;
  interrupt?: boolean;
};
```

### CanUseTool Callback

```typescript
type CanUseTool = (
  toolName: string,
  input: Record<string, unknown>,
  options: {
    signal: AbortSignal;
    suggestions?: PermissionUpdate[];
  }
) => Promise<PermissionResult>;
```

---

## MCP Server Configuration

### Server Config Types

Stdio server:

```typescript
type McpStdioServerConfig = {
  type?: 'stdio';
  command: string;
  args?: string[];
  env?: Record<string, string>;
};
```

SSE server:

```typescript
type McpSSEServerConfig = {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
};
```

HTTP server:

```typescript
type McpHttpServerConfig = {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
};
```

SDK server:

```typescript
type McpSdkServerConfig = {
  type: 'sdk';
  name: string;
};
```

SDK server with instance:

```typescript
type McpSdkServerConfigWithInstance = McpSdkServerConfig & {
  instance: McpServer; // from @modelcontextprotocol/sdk
};
```

### Union Types

```typescript
type McpServerConfig =
  | McpStdioServerConfig
  | McpSSEServerConfig
  | McpHttpServerConfig
  | McpSdkServerConfigWithInstance;
```

```typescript
type McpServerConfigForProcessTransport =
  | McpStdioServerConfig
  | McpSSEServerConfig
  | McpHttpServerConfig
  | McpSdkServerConfig;
```

### MCP Server Status

```typescript
type McpServerStatus = {
  name: string;
  status: 'connected' | 'failed' | 'needs-auth' | 'pending';
  serverInfo?: {
    name: string;
    version: string;
  };
};
```

### Creating SDK MCP Servers

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

```typescript
function tool<Schema extends ZodRawShape>(
  name: string,
  description: string,
  inputSchema: Schema,
  handler: (
    args: z.infer<ZodObject<Schema>>,
    extra: unknown
  ) => Promise<CallToolResult>
): SdkMcpToolDefinition<Schema>;
```

```typescript
type CreateSdkMcpServerOptions = {
  name: string;
  version?: string;
  tools?: Array<SdkMcpToolDefinition<any>>;
};
```

```typescript
function createSdkMcpServer(
  options: CreateSdkMcpServerOptions
): McpSdkServerConfigWithInstance;
```

---

## Tool Input Schemas

All tool input schemas are auto-generated from JSON Schema. The main union type:

```typescript
type ToolInputSchemas =
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
```

### Agent Tool

```typescript
interface AgentInput {
  description: string;       // 3-5 word description
  prompt: string;            // Task for the agent
  subagent_type: string;     // Type of specialized agent
}
```

### Bash Tool

```typescript
interface BashInput {
  command: string;           // Command to execute
  timeout?: number;          // Max 600000ms
  description?: string;      // 5-10 word description
  run_in_background?: boolean; // Run in background
}
```

### BashOutput Tool

```typescript
interface BashOutputInput {
  bash_id: string;           // Background shell ID
  filter?: string;           // Regex filter for output
}
```

### ExitPlanMode Tool

```typescript
interface ExitPlanModeInput {
  plan: string;              // Markdown plan for user approval
}
```

### FileEdit Tool

```typescript
interface FileEditInput {
  file_path: string;         // Absolute path
  old_string: string;        // Text to replace
  new_string: string;        // Replacement text
  replace_all?: boolean;     // Replace all occurrences
}
```

### FileRead Tool

```typescript
interface FileReadInput {
  file_path: string;         // Absolute path
  offset?: number;           // Start line number
  limit?: number;            // Number of lines
}
```

### FileWrite Tool

```typescript
interface FileWriteInput {
  file_path: string;         // Absolute path (required)
  content: string;           // File content
}
```

### Glob Tool

```typescript
interface GlobInput {
  pattern: string;           // Glob pattern (e.g., "**/*.ts")
  path?: string;             // Directory to search (default: cwd)
}
```

### Grep Tool

```typescript
interface GrepInput {
  pattern: string;           // Regex pattern
  path?: string;             // File or directory (default: cwd)
  glob?: string;             // File filter (e.g., "*.js")
  output_mode?: 'content' | 'files_with_matches' | 'count';
  '-B'?: number;             // Lines before match
  '-A'?: number;             // Lines after match
  '-C'?: number;             // Lines before and after
  '-n'?: boolean;            // Show line numbers
  '-i'?: boolean;            // Case insensitive
  type?: string;             // File type (js, py, etc.)
  head_limit?: number;       // Limit output lines
  multiline?: boolean;       // Multiline mode
}
```

### KillShell Tool

```typescript
interface KillShellInput {
  shell_id: string;          // Shell ID to kill
}
```

### ListMcpResources Tool

```typescript
interface ListMcpResourcesInput {
  server?: string;           // Filter by server name
}
```

### Mcp Tool

```typescript
interface McpInput {
  [k: string]: unknown;      // Generic MCP tool input
}
```

### NotebookEdit Tool

```typescript
interface NotebookEditInput {
  notebook_path: string;     // Absolute path to .ipynb
  cell_id?: string;          // Cell to edit/insert after
  new_source: string;        // New cell content
  cell_type?: 'code' | 'markdown';
  edit_mode?: 'replace' | 'insert' | 'delete';
}
```

### ReadMcpResource Tool

```typescript
interface ReadMcpResourceInput {
  server: string;            // MCP server name
  uri: string;               // Resource URI
}
```

### TodoWrite Tool

```typescript
interface TodoWriteInput {
  todos: {
    content: string;
    status: 'pending' | 'in_progress' | 'completed';
    activeForm: string;
  }[];
}
```

### WebFetch Tool

```typescript
interface WebFetchInput {
  url: string;               // URL to fetch
  prompt: string;            // Prompt for content analysis
}
```

### WebSearch Tool

```typescript
interface WebSearchInput {
  query: string;             // Search query
  allowed_domains?: string[]; // Whitelist domains
  blocked_domains?: string[]; // Blacklist domains
}
```

---

## Agent System

### Agent Definition

```typescript
type AgentDefinition = {
  description: string;
  tools?: string[];
  prompt: string;
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
```

### Setting Sources

```typescript
type SettingSource = 'user' | 'project' | 'local';
```

---

## Model & Usage Types

### Model Usage

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

### Non-Nullable Usage

```typescript
type NonNullableUsage = {
  [K in keyof Usage]: NonNullable<Usage[K]>;
};
```

Where `Usage` comes from `@anthropic-ai/sdk`:

```typescript
// From @anthropic-ai/sdk
type BetaUsage = {
  input_tokens?: number;
  output_tokens?: number;
  cache_creation_input_tokens?: number;
  cache_read_input_tokens?: number;
};
```

### API Key Source

```typescript
type ApiKeySource = 'user' | 'project' | 'org' | 'temporary';
```

### Config Scope

```typescript
type ConfigScope = 'local' | 'user' | 'project';
```

### Model Info

```typescript
type ModelInfo = {
  value: string;
  displayName: string;
  description: string;
};
```

---

## Query Interface

The `Query` interface extends `AsyncGenerator` and provides control methods:

```typescript
interface Query extends AsyncGenerator<SDKMessage, void> {
  // Control Requests (only supported with streaming I/O)

  interrupt(): Promise<void>;

  setPermissionMode(mode: PermissionMode): Promise<void>;

  setModel(model?: string): Promise<void>;

  /**
   * Set max thinking tokens. Use null to clear limit.
   */
  setMaxThinkingTokens(maxThinkingTokens: number | null): Promise<void>;

  supportedCommands(): Promise<SlashCommand[]>;

  supportedModels(): Promise<ModelInfo[]>;

  mcpServerStatus(): Promise<McpServerStatus[]>;

  accountInfo(): Promise<AccountInfo>;
}
```

### Slash Command

```typescript
type SlashCommand = {
  name: string;
  description: string;
  argumentHint: string;
};
```

### Account Info

```typescript
type AccountInfo = {
  email?: string;
  organization?: string;
  subscriptionType?: string;
  tokenSource?: string;
  apiKeySource?: string;
};
```

---

## Error Types

### Exit Reasons

```typescript
const EXIT_REASONS: string[];

type ExitReason = typeof EXIT_REASONS[number];
```

The SDK defines multiple exit reasons used in `SessionEndHookInput`.

### Abort Error

```typescript
class AbortError extends Error {}
```

---

## Type Relationships & Dependencies

### External Dependencies

The SDK imports types from:

1. **@anthropic-ai/sdk**:
   - `MessageParam` (aliased as `APIUserMessage`)
   - `BetaMessage` (aliased as `APIAssistantMessage`)
   - `BetaUsage` (aliased as `Usage`)
   - `BetaRawMessageStreamEvent` (aliased as `RawMessageStreamEvent`)

2. **@modelcontextprotocol/sdk**:
   - `CallToolResult` (from `/types.js`)
   - `McpServer` (from `/server/mcp.js`)

3. **crypto**:
   - `UUID`

4. **zod**:
   - `z`
   - `ZodRawShape`
   - `ZodObject`

### Type Hierarchy

```
Query (Interface)
├── AsyncGenerator<SDKMessage, void>
└── Control Methods
    ├── interrupt()
    ├── setPermissionMode()
    ├── setModel()
    ├── setMaxThinkingTokens()
    ├── supportedCommands()
    ├── supportedModels()
    ├── mcpServerStatus()
    └── accountInfo()

SDKMessage (Union)
├── SDKAssistantMessage
├── SDKUserMessage
├── SDKUserMessageReplay
├── SDKResultMessage
│   ├── subtype: 'success'
│   └── subtype: 'error_max_turns' | 'error_during_execution'
├── SDKSystemMessage
│   ├── subtype: 'init'
│   ├── subtype: 'compact_boundary'
│   └── subtype: 'hook_response'
└── SDKPartialAssistantMessage

HookInput (Union)
├── PreToolUseHookInput
├── PostToolUseHookInput
├── NotificationHookInput
├── UserPromptSubmitHookInput
├── SessionStartHookInput
├── SessionEndHookInput
├── StopHookInput
├── SubagentStopHookInput
└── PreCompactHookInput

PermissionUpdate (Union)
├── type: 'addRules'
├── type: 'replaceRules'
├── type: 'removeRules'
├── type: 'setMode'
├── type: 'addDirectories'
└── type: 'removeDirectories'

McpServerConfig (Union)
├── McpStdioServerConfig
├── McpSSEServerConfig
├── McpHttpServerConfig
└── McpSdkServerConfigWithInstance

ToolInputSchemas (Union)
├── AgentInput
├── BashInput
├── BashOutputInput
├── ExitPlanModeInput
├── FileEditInput
├── FileReadInput
├── FileWriteInput
├── GlobInput
├── GrepInput
├── KillShellInput
├── ListMcpResourcesInput
├── McpInput
├── NotebookEditInput
├── ReadMcpResourceInput
├── TodoWriteInput
├── WebFetchInput
└── WebSearchInput
```

---

## Key Observations

### Type Safety Features
1. All tool inputs are strongly typed via auto-generated interfaces
2. Union types ensure exhaustive pattern matching
3. Discriminated unions used for message types (via `type` and `subtype` fields)
4. AsyncGenerator typing enables proper TypeScript support for streaming

### SDK Design Patterns
1. **Async Generator Pattern**: Main query returns AsyncGenerator for streaming
2. **Discriminated Unions**: All message types use type/subtype discriminators
3. **Hook System**: Functional callback pattern with type-safe inputs/outputs
4. **Permission System**: Multi-level update mechanism with destination targeting
5. **MCP Integration**: Multiple transport types unified through union types

### Notable Type Features
1. **Optional Chaining Safety**: Many fields optional to allow progressive disclosure
2. **Synthetic Messages**: `isSynthetic` flag distinguishes system-generated messages
3. **Replay Detection**: `isReplay` prevents duplicate message processing
4. **Token Tracking**: Detailed usage breakdown including cache metrics
5. **Tool Whitelisting**: String array for tool names (no blacklist support)

---

## Usage Examples

### Basic Query

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const session = query({
  prompt: "Help me analyze this codebase",
  options: {
    model: 'claude-sonnet-4',
    maxTurns: 10,
    cwd: '/path/to/project',
  }
});

for await (const message of session) {
  console.log(message.type, message);
}
```

### With Permissions

```typescript
const session = query({
  prompt: "Edit some files",
  options: {
    permissionMode: 'default',
    canUseTool: async (toolName, input, { signal, suggestions }) => {
      // Custom permission logic
      if (toolName === 'FileEdit') {
        // Ask user for approval
        const approved = await askUser(input);
        if (approved) {
          return {
            behavior: 'allow',
            updatedInput: input,
            updatedPermissions: suggestions, // Apply suggestions
          };
        } else {
          return {
            behavior: 'deny',
            message: 'User denied file edit',
            interrupt: true,
          };
        }
      }
      return {
        behavior: 'allow',
        updatedInput: input,
      };
    },
  }
});
```

### With Hooks

```typescript
const session = query({
  prompt: "Run some tasks",
  options: {
    hooks: {
      PreToolUse: [{
        matcher: 'Bash',
        hooks: [async (input, toolUseID, { signal }) => {
          const hookInput = input as PreToolUseHookInput;
          console.log('About to run:', hookInput.tool_input);

          return {
            continue: true,
            decision: 'approve',
          };
        }],
      }],
      PostToolUse: [{
        hooks: [async (input, toolUseID, { signal }) => {
          const hookInput = input as PostToolUseHookInput;
          console.log('Tool finished:', hookInput.tool_response);

          return {
            continue: true,
          };
        }],
      }],
    },
  }
});
```

### With MCP Servers

```typescript
const session = query({
  prompt: "Use custom tools",
  options: {
    mcpServers: {
      'my-server': {
        type: 'stdio',
        command: 'node',
        args: ['./my-mcp-server.js'],
        env: {
          API_KEY: process.env.API_KEY,
        },
      },
      'remote-server': {
        type: 'sse',
        url: 'https://mcp.example.com',
        headers: {
          'Authorization': 'Bearer token',
        },
      },
    },
  }
});
```

### With Custom SDK MCP Server

```typescript
import { createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

const customServer = createSdkMcpServer({
  name: 'my-custom-tools',
  version: '1.0.0',
  tools: [
    tool(
      'customTool',
      'Does something custom',
      {
        input: z.string(),
      },
      async (args, extra) => {
        return {
          content: [{ type: 'text', text: `Processed: ${args.input}` }],
        };
      }
    ),
  ],
});

const session = query({
  prompt: "Use my custom tool",
  options: {
    mcpServers: {
      'custom': customServer,
    },
  }
});
```

### With Agents

```typescript
const session = query({
  prompt: "@researcher find information about TypeScript",
  options: {
    agents: {
      researcher: {
        description: 'Research specialist',
        prompt: 'You are a research specialist. Focus on finding accurate information.',
        tools: ['WebSearch', 'WebFetch'],
        model: 'sonnet',
      },
      coder: {
        description: 'Coding specialist',
        prompt: 'You are a coding specialist. Focus on writing quality code.',
        tools: ['FileEdit', 'FileWrite', 'Bash'],
        model: 'sonnet',
      },
    },
  }
});
```

### Control Methods

```typescript
const session = query({
  prompt: "Long running task",
  options: {
    maxTurns: 100,
  }
});

// Run in background and control
(async () => {
  for await (const message of session) {
    console.log(message);

    // Dynamically change model
    if (message.type === 'assistant') {
      await session.setModel('claude-opus-4');
    }

    // Check account info
    const account = await session.accountInfo();
    console.log('Account:', account);

    // Interrupt if needed
    if (shouldStop()) {
      await session.interrupt();
    }
  }
})();
```

---

## Summary

The Claude Agent SDK provides a comprehensive type system for building AI agents with:

1. **17 Tool Input Schemas** for various operations (file, bash, web, MCP, etc.)
2. **8 Message Types** for communication (user, assistant, system, result, etc.)
3. **9 Hook Events** for lifecycle management
4. **6 Permission Update Types** for fine-grained control
5. **4 MCP Server Config Types** for extensibility
6. **Streaming Query Interface** with control methods
7. **Complete Type Safety** with TypeScript

All types are well-documented, use discriminated unions for type safety, and provide comprehensive coverage of the SDK's capabilities.
