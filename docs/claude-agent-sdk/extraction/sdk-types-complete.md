# Claude Agent SDK - Complete Type System Extraction

**SDK Version**: 0.1.22
**Extraction Date**: 2025-10-24
**Source Files**: `sdkTypes.d.ts`, `sdk.d.ts`, `sdk-tools.d.ts`

---

## Table of Contents

1. [Core SDK Exports](#core-sdk-exports)
2. [Query Interface](#query-interface)
3. [Message Types](#message-types)
4. [Hook System Types](#hook-system-types)
5. [Permission System Types](#permission-system-types)
6. [MCP Integration Types](#mcp-integration-types)
7. [Agent Definition Types](#agent-definition-types)
8. [Tool Input Schemas](#tool-input-schemas)
9. [Usage and Metrics Types](#usage-and-metrics-types)

---

## Core SDK Exports

### Main Query Function

```typescript
export declare function query(_params: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;
```

### SDK Utility Functions

```typescript
export declare const HOOK_EVENTS: readonly [
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

export declare const EXIT_REASONS: string[];

export declare function tool<Schema extends ZodRawShape>(
  name: string,
  description: string,
  inputSchema: Schema,
  handler: (
    args: z.infer<ZodObject<Schema>>,
    extra: unknown
  ) => Promise<CallToolResult>
): SdkMcpToolDefinition<Schema>;

export declare function createSdkMcpServer(
  options: CreateSdkMcpServerOptions
): McpSdkServerConfigWithInstance;

export declare class AbortError extends Error {}
```

---

## Query Interface

### AsyncGenerator Interface

```typescript
export interface Query extends AsyncGenerator<SDKMessage, void> {
  // Control Requests (only supported when streaming)
  interrupt(): Promise<void>;
  setPermissionMode(mode: PermissionMode): Promise<void>;
  setModel(model?: string): Promise<void>;
  setMaxThinkingTokens(maxThinkingTokens: number | null): Promise<void>;
  
  // Metadata Queries
  supportedCommands(): Promise<SlashCommand[]>;
  supportedModels(): Promise<ModelInfo[]>;
  mcpServerStatus(): Promise<McpServerStatus[]>;
  accountInfo(): Promise<AccountInfo>;
}
```

### Query Options

```typescript
export type Options = {
  // Control
  abortController?: AbortController;
  continue?: boolean;
  maxThinkingTokens?: number;
  maxTurns?: number;
  
  // Working Directory
  cwd?: string;
  additionalDirectories?: string[];
  
  // Model Configuration
  model?: string;
  fallbackModel?: string;
  customSystemPrompt?: string;
  appendSystemPrompt?: string;
  
  // Tools
  allowedTools?: string[];
  disallowedTools?: string[];
  
  // Permissions
  canUseTool?: CanUseTool;
  permissionMode?: PermissionMode;
  permissionPromptToolName?: string;
  
  // Hooks
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;
  
  // MCP Servers
  mcpServers?: Record<string, McpServerConfig>;
  strictMcpConfig?: boolean;
  
  // Advanced
  executable?: 'bun' | 'deno' | 'node';
  executableArgs?: string[];
  env?: { [envVar: string]: string | undefined };
  extraArgs?: Record<string, string | null>;
  pathToClaudeCodeExecutable?: string;
  
  // Session Management
  resume?: string;
  forkSession?: boolean;
  resumeSessionAt?: string;
  
  // Debugging
  stderr?: (data: string) => void;
  includePartialMessages?: boolean;
};
```

### Agent SDK Specific Options

```typescript
export type Options = Omit<
  BaseOptions,
  'customSystemPrompt' | 'appendSystemPrompt'
> & {
  agents?: Record<string, AgentDefinition>;
  settingSources?: SettingSource[];
  systemPrompt?:
    | string
    | {
        type: 'preset';
        preset: 'claude_code';
        append?: string;
      };
};

export type SettingSource = 'user' | 'project' | 'local';
```

---

## Message Types

### Base Message Type

```typescript
export type SDKMessageBase = {
  uuid: UUID;
  session_id: string;
};
```

### User Messages

```typescript
type SDKUserMessageContent = {
  type: 'user';
  message: APIUserMessage; // from @anthropic-ai/sdk
  parent_tool_use_id: string | null;
  isSynthetic?: boolean; // True if system-generated
};

export type SDKUserMessage = SDKUserMessageContent & {
  uuid?: UUID;
  session_id: string;
};

export type SDKUserMessageReplay = SDKMessageBase & SDKUserMessageContent & {
  isReplay: true; // Prevents duplicate messages in array
};
```

### Assistant Messages

```typescript
export type SDKAssistantMessage = SDKMessageBase & {
  type: 'assistant';
  message: APIAssistantMessage; // from @anthropic-ai/sdk
  parent_tool_use_id: string | null;
};

export type SDKPartialAssistantMessage = SDKMessageBase & {
  type: 'stream_event';
  event: RawMessageStreamEvent; // from @anthropic-ai/sdk
  parent_tool_use_id: string | null;
};
```

### System Messages

```typescript
export type SDKSystemMessage = SDKMessageBase & {
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

export type SDKCompactBoundaryMessage = SDKMessageBase & {
  type: 'system';
  subtype: 'compact_boundary';
  compact_metadata: {
    trigger: 'manual' | 'auto';
    pre_tokens: number;
  };
};

export type SDKHookResponseMessage = SDKMessageBase & {
  type: 'system';
  subtype: 'hook_response';
  hook_name: string;
  hook_event: string;
  stdout: string;
  stderr: string;
  exit_code?: number;
};
```

### Result Messages

```typescript
export type SDKPermissionDenial = {
  tool_name: string;
  tool_use_id: string;
  tool_input: Record<string, unknown>;
};

export type SDKResultMessage =
  | (SDKMessageBase & {
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
    })
  | (SDKMessageBase & {
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
    });
```

### Union Type

```typescript
export type SDKMessage =
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

## Hook System Types

### Hook Events

```typescript
export declare const HOOK_EVENTS: readonly [
  "PreToolUse",        // Before tool execution
  "PostToolUse",       // After tool execution
  "Notification",      // System notifications
  "UserPromptSubmit",  // Before sending to model
  "SessionStart",      // Session initialization
  "SessionEnd",        // Session termination
  "Stop",              // User interruption (Ctrl+C)
  "SubagentStop",      // Subagent termination
  "PreCompact"         // Before context compression
];

export type HookEvent = (typeof HOOK_EVENTS)[number];
```

### Hook Callbacks

```typescript
export type HookCallback = (
  input: HookInput,
  toolUseID: string | undefined,
  options: {
    signal: AbortSignal;
  }
) => Promise<HookJSONOutput>;

export interface HookCallbackMatcher {
  matcher?: string; // Tool name pattern or undefined for all
  hooks: HookCallback[];
}
```

### Hook Input Types

```typescript
export type BaseHookInput = {
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
};

export type PreToolUseHookInput = BaseHookInput & {
  hook_event_name: 'PreToolUse';
  tool_name: string;
  tool_input: unknown;
};

export type PostToolUseHookInput = BaseHookInput & {
  hook_event_name: 'PostToolUse';
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;
};

export type NotificationHookInput = BaseHookInput & {
  hook_event_name: 'Notification';
  message: string;
  title?: string;
};

export type UserPromptSubmitHookInput = BaseHookInput & {
  hook_event_name: 'UserPromptSubmit';
  prompt: string;
};

export type SessionStartHookInput = BaseHookInput & {
  hook_event_name: 'SessionStart';
  source: 'startup' | 'resume' | 'clear' | 'compact';
};

export type StopHookInput = BaseHookInput & {
  hook_event_name: 'Stop';
  stop_hook_active: boolean;
};

export type SubagentStopHookInput = BaseHookInput & {
  hook_event_name: 'SubagentStop';
  stop_hook_active: boolean;
};

export type PreCompactHookInput = BaseHookInput & {
  hook_event_name: 'PreCompact';
  trigger: 'manual' | 'auto';
  custom_instructions: string | null;
};

export declare const EXIT_REASONS: string[];
export type ExitReason = (typeof EXIT_REASONS)[number];

export type SessionEndHookInput = BaseHookInput & {
  hook_event_name: 'SessionEnd';
  reason: ExitReason;
};

export type HookInput =
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

```typescript
export type AsyncHookJSONOutput = {
  async: true;
  asyncTimeout?: number;
};

export type SyncHookJSONOutput = {
  // Control flow
  continue?: boolean;
  suppressOutput?: boolean;
  stopReason?: string;
  decision?: 'approve' | 'block';
  systemMessage?: string;
  reason?: string;
  
  // Hook-specific output
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

export type HookJSONOutput = AsyncHookJSONOutput | SyncHookJSONOutput;
```

---

## Permission System Types

### Permission Modes

```typescript
export type PermissionMode =
  | 'default'              // Ask for each operation
  | 'acceptEdits'          // Auto-approve Edit tool
  | 'bypassPermissions'    // Approve everything (DANGEROUS)
  | 'plan';                // Planning mode, no execution
```

### Permission Behaviors

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask';
```

### Permission Rules

```typescript
export type PermissionRuleValue = {
  toolName: string;
  ruleContent?: string; // Pattern (glob or regex)
};
```

### Permission Updates

```typescript
type PermissionUpdateDestination =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'session';

export type PermissionUpdate =
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

### Permission Results

```typescript
export type PermissionResult =
  | {
      behavior: 'allow';
      updatedInput: Record<string, unknown>;
      updatedPermissions?: PermissionUpdate[];
    }
  | {
      behavior: 'deny';
      message: string;
      interrupt?: boolean;
    };
```

### CanUseTool Callback

```typescript
export type CanUseTool = (
  toolName: string,
  input: Record<string, unknown>,
  options: {
    signal: AbortSignal;
    suggestions?: PermissionUpdate[];
  }
) => Promise<PermissionResult>;
```

---

## MCP Integration Types

### MCP Server Configurations

```typescript
export type McpStdioServerConfig = {
  type?: 'stdio';
  command: string;
  args?: string[];
  env?: Record<string, string>;
};

export type McpSSEServerConfig = {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
};

export type McpHttpServerConfig = {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
};

export type McpSdkServerConfig = {
  type: 'sdk';
  name: string;
};

export type McpSdkServerConfigWithInstance = McpSdkServerConfig & {
  instance: McpServer; // from @modelcontextprotocol/sdk
};

export type McpServerConfig =
  | McpStdioServerConfig
  | McpSSEServerConfig
  | McpHttpServerConfig
  | McpSdkServerConfigWithInstance;

export type McpServerConfigForProcessTransport =
  | McpStdioServerConfig
  | McpSSEServerConfig
  | McpHttpServerConfig
  | McpSdkServerConfig;
```

### MCP Server Status

```typescript
export type McpServerStatus = {
  name: string;
  status: 'connected' | 'failed' | 'needs-auth' | 'pending';
  serverInfo?: {
    name: string;
    version: string;
  };
};
```

### MCP Tool Definition

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

type CreateSdkMcpServerOptions = {
  name: string;
  version?: string;
  tools?: Array<SdkMcpToolDefinition<any>>;
};
```

---

## Agent Definition Types

### Agent Definition

```typescript
export type AgentDefinition = {
  description: string;
  tools?: string[];
  prompt: string;
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
```

---

## Tool Input Schemas

### Complete Tool Input Union

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
```

### Individual Tool Schemas

```typescript
export interface AgentInput {
  description: string;
  prompt: string;
  subagent_type: string;
}

export interface BashInput {
  command: string;
  timeout?: number;
  description?: string;
  run_in_background?: boolean;
}

export interface BashOutputInput {
  bash_id: string;
  filter?: string;
}

export interface ExitPlanModeInput {
  plan: string;
}

export interface FileEditInput {
  file_path: string;
  old_string: string;
  new_string: string;
  replace_all?: boolean;
}

export interface FileReadInput {
  file_path: string;
  offset?: number;
  limit?: number;
}

export interface FileWriteInput {
  file_path: string;
  content: string;
}

export interface GlobInput {
  pattern: string;
  path?: string;
}

export interface GrepInput {
  pattern: string;
  path?: string;
  glob?: string;
  output_mode?: 'content' | 'files_with_matches' | 'count';
  '-B'?: number;
  '-A'?: number;
  '-C'?: number;
  '-n'?: boolean;
  '-i'?: boolean;
  type?: string;
  head_limit?: number;
  multiline?: boolean;
}

export interface KillShellInput {
  shell_id: string;
}

export interface ListMcpResourcesInput {
  server?: string;
}

export interface McpInput {
  [k: string]: unknown;
}

export interface NotebookEditInput {
  notebook_path: string;
  cell_id?: string;
  new_source: string;
  cell_type?: 'code' | 'markdown';
  edit_mode?: 'replace' | 'insert' | 'delete';
}

export interface ReadMcpResourceInput {
  server: string;
  uri: string;
}

export interface TodoWriteInput {
  todos: {
    content: string;
    status: 'pending' | 'in_progress' | 'completed';
    activeForm: string;
  }[];
}

export interface WebFetchInput {
  url: string;
  prompt: string;
}

export interface WebSearchInput {
  query: string;
  allowed_domains?: string[];
  blocked_domains?: string[];
}
```

---

## Usage and Metrics Types

### Usage Types

```typescript
export type NonNullableUsage = {
  [K in keyof Usage]: NonNullable<Usage[K]>;
};

export type ModelUsage = {
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
  webSearchRequests: number;
  costUSD: number;
  contextWindow: number;
};
```

### Configuration Types

```typescript
export type ApiKeySource = 'user' | 'project' | 'org' | 'temporary';
export type ConfigScope = 'local' | 'user' | 'project';
```

### Model Information

```typescript
export type ModelInfo = {
  value: string;
  displayName: string;
  description: string;
};

export type SlashCommand = {
  name: string;
  description: string;
  argumentHint: string;
};

export type AccountInfo = {
  email?: string;
  organization?: string;
  subscriptionType?: string;
  tokenSource?: string;
  apiKeySource?: string;
};
```

---

## Type Relationships Diagram

```
Query (AsyncGenerator)
├── Options
│   ├── Hooks (HookEvent -> HookCallbackMatcher[])
│   ├── Permissions (PermissionMode, CanUseTool)
│   ├── MCP Servers (McpServerConfig)
│   └── Agent Definitions
│
├── Yields SDKMessage
│   ├── SDKUserMessage
│   ├── SDKAssistantMessage
│   ├── SDKPartialAssistantMessage (streaming)
│   ├── SDKSystemMessage
│   ├── SDKCompactBoundaryMessage
│   ├── SDKHookResponseMessage
│   └── SDKResultMessage
│       ├── ModelUsage
│       └── SDKPermissionDenial[]
│
└── Control Methods
    ├── interrupt()
    ├── setPermissionMode()
    ├── setModel()
    ├── setMaxThinkingTokens()
    ├── supportedCommands()
    ├── supportedModels()
    ├── mcpServerStatus()
    └── accountInfo()
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
