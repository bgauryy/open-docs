# Claude Agent SDK: Hook System and Permission System

**Complete Reference Documentation**

**SDK Version**: 0.1.22
**Package**: @anthropic-ai/claude-agent-sdk

---

## Table of Contents

1. [Hook System](#hook-system)
   - [Overview](#hook-system-overview)
   - [All 9 Hook Events](#all-9-hook-events)
   - [Hook Input Schemas](#hook-input-schemas)
   - [Hook Output Schemas](#hook-output-schemas)
   - [Hook Execution Flow](#hook-execution-flow)
   - [Hook Matcher Patterns](#hook-matcher-patterns)
   - [Hook Callback Interface](#hook-callback-interface)
   - [Hook-Specific Output](#hook-specific-output)
   - [Hook Examples](#hook-examples)

2. [Permission System](#permission-system)
   - [Overview](#permission-system-overview)
   - [All 4 Permission Modes](#all-4-permission-modes)
   - [Permission Rules](#permission-rules)
   - [Permission Update Types](#permission-update-types)
   - [Permission Result](#permission-result)
   - [Custom Permission Callbacks](#custom-permission-callbacks)
   - [Permission Patterns](#permission-patterns)
   - [Permission Examples](#permission-examples)

---

## Hook System

### Hook System Overview

The Claude Agent SDK provides a comprehensive hook system that allows you to intercept and modify agent behavior at critical points during execution. Hooks are asynchronous callbacks that receive event data and can control the agent's flow.

### All 9 Hook Events

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

export type HookEvent = (typeof HOOK_EVENTS)[number];
```

#### Hook Event Descriptions

| Event | Triggered When | Use Cases |
|-------|----------------|-----------|
| **PreToolUse** | Before a tool is executed | Permission checks, input validation, tool blocking |
| **PostToolUse** | After a tool completes execution | Logging, result modification, error handling |
| **Notification** | When the agent sends a notification | Custom notification handling, filtering |
| **UserPromptSubmit** | When user submits a prompt | Prompt preprocessing, context injection |
| **SessionStart** | When a session starts or resumes | Initialization, session setup |
| **SessionEnd** | When a session ends | Cleanup, final logging, analytics |
| **Stop** | When the agent is stopped | Graceful shutdown, state saving |
| **SubagentStop** | When a subagent is stopped | Subagent cleanup |
| **PreCompact** | Before conversation history compaction | Custom compaction logic, history archival |

---

### Hook Input Schemas

#### Base Hook Input

All hook inputs extend this base interface:

```typescript
export type BaseHookInput = {
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
};
```

#### 1. PreToolUse Hook Input

```typescript
export type PreToolUseHookInput = BaseHookInput & {
  hook_event_name: 'PreToolUse';
  tool_name: string;
  tool_input: unknown;
};
```

**Fields:**
- `tool_name`: Name of the tool about to be executed
- `tool_input`: Input parameters for the tool

**Use Case:** Intercept tool calls before execution to validate, modify, or block them.

---

#### 2. PostToolUse Hook Input

```typescript
export type PostToolUseHookInput = BaseHookInput & {
  hook_event_name: 'PostToolUse';
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;
};
```

**Fields:**
- `tool_name`: Name of the executed tool
- `tool_input`: Input parameters that were used
- `tool_response`: Response returned by the tool

**Use Case:** Log tool usage, modify responses, or add context based on results.

---

#### 3. Notification Hook Input

```typescript
export type NotificationHookInput = BaseHookInput & {
  hook_event_name: 'Notification';
  message: string;
  title?: string;
};
```

**Fields:**
- `message`: Notification message content
- `title`: Optional notification title

**Use Case:** Custom notification handling, filtering, or routing.

---

#### 4. UserPromptSubmit Hook Input

```typescript
export type UserPromptSubmitHookInput = BaseHookInput & {
  hook_event_name: 'UserPromptSubmit';
  prompt: string;
};
```

**Fields:**
- `prompt`: The user's submitted prompt text

**Use Case:** Preprocess prompts, inject context, or validate user input.

---

#### 5. SessionStart Hook Input

```typescript
export type SessionStartHookInput = BaseHookInput & {
  hook_event_name: 'SessionStart';
  source: 'startup' | 'resume' | 'clear' | 'compact';
};
```

**Fields:**
- `source`: How the session started
  - `startup`: New session
  - `resume`: Resumed from saved state
  - `clear`: Started after clearing history
  - `compact`: Started after compaction

**Use Case:** Session initialization, loading custom state, or setup tasks.

---

#### 6. SessionEnd Hook Input

```typescript
export declare const EXIT_REASONS: string[];
export type ExitReason = (typeof EXIT_REASONS)[number];

export type SessionEndHookInput = BaseHookInput & {
  hook_event_name: 'SessionEnd';
  reason: ExitReason;
};
```

**Fields:**
- `reason`: Why the session ended (from EXIT_REASONS constant)

**Use Case:** Cleanup, analytics, saving session data.

---

#### 7. Stop Hook Input

```typescript
export type StopHookInput = BaseHookInput & {
  hook_event_name: 'Stop';
  stop_hook_active: boolean;
};
```

**Fields:**
- `stop_hook_active`: Whether stop hooks are currently active

**Use Case:** Graceful shutdown, state persistence.

---

#### 8. SubagentStop Hook Input

```typescript
export type SubagentStopHookInput = BaseHookInput & {
  hook_event_name: 'SubagentStop';
  stop_hook_active: boolean;
};
```

**Fields:**
- `stop_hook_active`: Whether stop hooks are currently active

**Use Case:** Subagent cleanup and resource management.

---

#### 9. PreCompact Hook Input

```typescript
export type PreCompactHookInput = BaseHookInput & {
  hook_event_name: 'PreCompact';
  trigger: 'manual' | 'auto';
  custom_instructions: string | null;
};
```

**Fields:**
- `trigger`: How compaction was triggered
  - `manual`: User-initiated
  - `auto`: Automatically triggered
- `custom_instructions`: Optional custom compaction instructions

**Use Case:** Custom history management, archival before compaction.

---

#### Union Type: HookInput

```typescript
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

---

### Hook Output Schemas

Hooks can return either **synchronous** or **asynchronous** output.

#### Async Hook Output

```typescript
export type AsyncHookJSONOutput = {
  async: true;
  asyncTimeout?: number; // Optional timeout in milliseconds
};
```

**Use Case:** When the hook needs to perform long-running operations.

---

#### Sync Hook Output

```typescript
export type SyncHookJSONOutput = {
  continue?: boolean;           // Whether to continue execution
  suppressOutput?: boolean;     // Suppress output from being shown
  stopReason?: string;          // Reason for stopping (if continue=false)
  decision?: 'approve' | 'block'; // Approval decision
  systemMessage?: string;       // Message to add to conversation
  reason?: string;              // Reason for the decision

  // Hook-specific output (see next section)
  hookSpecificOutput?: {
    hookEventName: 'PreToolUse';
    permissionDecision?: 'allow' | 'deny' | 'ask';
    permissionDecisionReason?: string;
    updatedInput?: Record<string, unknown>;
  } | {
    hookEventName: 'UserPromptSubmit';
    additionalContext?: string;
  } | {
    hookEventName: 'SessionStart';
    additionalContext?: string;
  } | {
    hookEventName: 'PostToolUse';
    additionalContext?: string;
  };
};
```

---

#### Union Type: HookJSONOutput

```typescript
export type HookJSONOutput = AsyncHookJSONOutput | SyncHookJSONOutput;
```

---

### Hook Execution Flow

```
1. Agent triggers event (e.g., about to use a tool)
   ↓
2. SDK looks for registered hooks for this event type
   ↓
3. For each matching hook callback:
   - Check if matcher pattern matches (if specified)
   - Execute hook callback with input data
   ↓
4. Hook returns HookJSONOutput
   ↓
5. SDK processes output:
   - If async=true: Wait for async operation
   - If decision='block': Stop execution
   - If hookSpecificOutput provided: Apply modifications
   - If continue=false: Stop agent execution
   ↓
6. Continue with modified behavior
```

---

### Hook Matcher Patterns

```typescript
export interface HookCallbackMatcher {
  matcher?: string;  // Optional pattern to match against
  hooks: HookCallback[];
}
```

**Registration in Options:**

```typescript
hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;
```

**Pattern Matching:**
- `matcher` is an optional string pattern
- If provided, the hook only executes when the pattern matches
- If omitted, the hook executes for all events of that type
- Multiple matchers can be registered for the same event

**Example:**

```typescript
{
  hooks: {
    PreToolUse: [
      {
        matcher: 'Bash',
        hooks: [bashToolValidator]
      },
      {
        matcher: 'Edit',
        hooks: [editToolValidator]
      },
      {
        // No matcher - runs for all PreToolUse events
        hooks: [globalToolLogger]
      }
    ]
  }
}
```

---

### Hook Callback Interface

```typescript
export type HookCallback = (
  input: HookInput,
  toolUseID: string | undefined,
  options: {
    signal: AbortSignal;
  }
) => Promise<HookJSONOutput>;
```

**Parameters:**
- `input`: Hook-specific input data (discriminated union based on `hook_event_name`)
- `toolUseID`: ID of the tool use (if applicable, undefined otherwise)
- `options.signal`: AbortSignal for cancellation support

**Returns:** Promise resolving to HookJSONOutput

---

### Hook-Specific Output

Different hooks support specific output fields:

#### PreToolUse Hook-Specific Output

```typescript
{
  hookEventName: 'PreToolUse';
  permissionDecision?: 'allow' | 'deny' | 'ask';
  permissionDecisionReason?: string;
  updatedInput?: Record<string, unknown>;
}
```

**Fields:**
- `permissionDecision`: Override permission system decision
- `permissionDecisionReason`: Explanation for the decision
- `updatedInput`: Modified tool input to use instead of original

**Use Case:** Custom permission logic, input sanitization, parameter validation.

---

#### UserPromptSubmit Hook-Specific Output

```typescript
{
  hookEventName: 'UserPromptSubmit';
  additionalContext?: string;
}
```

**Fields:**
- `additionalContext`: Additional context to inject into the conversation

**Use Case:** Add context based on user's prompt, inject relevant information.

---

#### SessionStart Hook-Specific Output

```typescript
{
  hookEventName: 'SessionStart';
  additionalContext?: string;
}
```

**Fields:**
- `additionalContext`: Context to add at session start

**Use Case:** Initialize session with custom context, load user preferences.

---

#### PostToolUse Hook-Specific Output

```typescript
{
  hookEventName: 'PostToolUse';
  additionalContext?: string;
}
```

**Fields:**
- `additionalContext`: Context to add based on tool results

**Use Case:** Provide guidance based on tool output, error handling instructions.

---

### Hook Examples

#### Example 1: PreToolUse - Block Dangerous Commands

```typescript
import { query, HookCallback, PreToolUseHookInput } from '@anthropic-ai/claude-agent-sdk';

const dangerousCommandBlocker: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'PreToolUse') {
    const hookInput = input as PreToolUseHookInput;

    if (hookInput.tool_name === 'Bash') {
      const command = hookInput.tool_input as { command?: string };
      const dangerousPatterns = [/rm\s+-rf\s+\//, /mkfs/, /dd\s+if=/];

      if (command.command && dangerousPatterns.some(p => p.test(command.command!))) {
        return {
          decision: 'block',
          stopReason: 'Dangerous command blocked',
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'deny',
            permissionDecisionReason: 'Command contains potentially dangerous operations'
          }
        };
      }
    }
  }

  return { continue: true };
};

const session = query({
  prompt: 'Delete all temporary files',
  options: {
    hooks: {
      PreToolUse: [{
        matcher: 'Bash',
        hooks: [dangerousCommandBlocker]
      }]
    }
  }
});
```

---

#### Example 2: PostToolUse - Logging and Analytics

```typescript
const toolUsageLogger: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'PostToolUse') {
    const hookInput = input as PostToolUseHookInput;

    // Log to analytics
    await logToolUsage({
      sessionId: hookInput.session_id,
      toolName: hookInput.tool_name,
      toolUseId: toolUseID,
      timestamp: new Date().toISOString(),
      success: !hookInput.tool_response?.error
    });

    // Continue normally
    return { continue: true };
  }

  return { continue: true };
};

const session = query({
  prompt: 'Analyze the codebase',
  options: {
    hooks: {
      PostToolUse: [{
        hooks: [toolUsageLogger]
      }]
    }
  }
});
```

---

#### Example 3: UserPromptSubmit - Context Injection

```typescript
const contextInjector: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'UserPromptSubmit') {
    const hookInput = input as UserPromptSubmitHookInput;

    // Add project-specific context
    const projectContext = await loadProjectContext(hookInput.cwd);

    return {
      continue: true,
      hookSpecificOutput: {
        hookEventName: 'UserPromptSubmit',
        additionalContext: `Project Context:\n${projectContext}\n\nUser's request: ${hookInput.prompt}`
      }
    };
  }

  return { continue: true };
};
```

---

#### Example 4: SessionStart - Initialization

```typescript
const sessionInitializer: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'SessionStart') {
    const hookInput = input as SessionStartHookInput;

    if (hookInput.source === 'startup') {
      // Load user preferences
      const preferences = await loadUserPreferences();

      return {
        continue: true,
        hookSpecificOutput: {
          hookEventName: 'SessionStart',
          additionalContext: `User preferences loaded: ${JSON.stringify(preferences)}`
        }
      };
    }
  }

  return { continue: true };
};
```

---

#### Example 5: PreCompact - History Archival

```typescript
const historyArchiver: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'PreCompact') {
    const hookInput = input as PreCompactHookInput;

    // Archive conversation history before compaction
    await archiveConversation({
      sessionId: hookInput.session_id,
      transcriptPath: hookInput.transcript_path,
      trigger: hookInput.trigger,
      timestamp: new Date().toISOString()
    });

    return {
      continue: true,
      systemMessage: 'Conversation history archived successfully'
    };
  }

  return { continue: true };
};
```

---

#### Example 6: Multiple Hooks with Matchers

```typescript
const bashValidator: HookCallback = async (input, toolUseID, { signal }) => {
  // Bash-specific validation logic
  return { continue: true };
};

const editValidator: HookCallback = async (input, toolUseID, { signal }) => {
  // Edit-specific validation logic
  return { continue: true };
};

const globalLogger: HookCallback = async (input, toolUseID, { signal }) => {
  // Log all tool uses
  return { continue: true };
};

const session = query({
  prompt: 'Refactor the authentication module',
  options: {
    hooks: {
      PreToolUse: [
        { matcher: 'Bash', hooks: [bashValidator] },
        { matcher: 'Edit', hooks: [editValidator] },
        { hooks: [globalLogger] } // No matcher - runs for all
      ]
    }
  }
});
```

---

## Permission System

### Permission System Overview

The Claude Agent SDK includes a sophisticated permission system that controls tool usage and file access. It supports multiple modes, custom callbacks, and granular permission rules.

---

### All 4 Permission Modes

```typescript
export type PermissionMode =
  | 'default'
  | 'acceptEdits'
  | 'bypassPermissions'
  | 'plan';
```

#### Mode Descriptions

| Mode | Behavior | Use Case |
|------|----------|----------|
| **default** | Standard permission checking. User is prompted for tool usage that requires permission. | Interactive development, code review |
| **acceptEdits** | Auto-approve edit operations (Edit, Write tools). User still prompted for other tools. | Code generation, refactoring tasks |
| **bypassPermissions** | Skip all permission checks. All tools are automatically approved. | Automated scripts, trusted environments |
| **plan** | Planning mode. Agent plans actions but doesn't execute them. | Strategy development, architecture planning |

---

#### Mode Details

##### 1. default Mode

```typescript
permissionMode: 'default'
```

**Behavior:**
- User is prompted before tool execution
- Permission rules are fully enforced
- Most restrictive mode for safety

**Example Use Case:**
```typescript
const session = query({
  prompt: 'Update the database schema',
  options: {
    permissionMode: 'default' // Safe, requires user approval
  }
});
```

---

##### 2. acceptEdits Mode

```typescript
permissionMode: 'acceptEdits'
```

**Behavior:**
- Automatically approves Edit and Write operations
- User still prompted for Bash, Delete, and other tools
- Useful for code generation workflows

**Example Use Case:**
```typescript
const session = query({
  prompt: 'Generate a REST API with CRUD operations',
  options: {
    permissionMode: 'acceptEdits' // Auto-approve file edits
  }
});
```

---

##### 3. bypassPermissions Mode

```typescript
permissionMode: 'bypassPermissions'
```

**Behavior:**
- All tools are automatically approved
- No user prompts
- Use with caution in trusted environments

**Example Use Case:**
```typescript
const session = query({
  prompt: 'Run the full test suite and fix any failures',
  options: {
    permissionMode: 'bypassPermissions' // Fully automated
  }
});
```

---

##### 4. plan Mode

```typescript
permissionMode: 'plan'
```

**Behavior:**
- Agent generates a plan but doesn't execute
- No actual tool usage occurs
- Safe exploration of what the agent would do

**Example Use Case:**
```typescript
const session = query({
  prompt: 'How would you refactor this codebase?',
  options: {
    permissionMode: 'plan' // Just plan, don't execute
  }
});
```

---

### Permission Rules

#### Permission Rule Value

```typescript
export type PermissionRuleValue = {
  toolName: string;
  ruleContent?: string;
};
```

**Fields:**
- `toolName`: The tool this rule applies to (e.g., 'Bash', 'Edit', 'Read')
- `ruleContent`: Optional rule-specific content or pattern

**Example:**
```typescript
{
  toolName: 'Bash',
  ruleContent: 'git commit.*' // Only allow git commit commands
}
```

---

#### Permission Behavior

```typescript
export type PermissionBehavior = 'allow' | 'deny' | 'ask';
```

**Values:**
- `allow`: Automatically permit the tool usage
- `deny`: Automatically block the tool usage
- `ask`: Prompt the user for permission

---

### Permission Update Types

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

---

#### Update Type Details

##### 1. addRules

Add new permission rules without removing existing ones.

```typescript
{
  type: 'addRules',
  rules: [
    { toolName: 'Bash', ruleContent: 'npm test' },
    { toolName: 'Edit' }
  ],
  behavior: 'allow',
  destination: 'session'
}
```

**Use Case:** Grant additional permissions during a session.

---

##### 2. replaceRules

Replace all existing rules with new ones.

```typescript
{
  type: 'replaceRules',
  rules: [
    { toolName: 'Read' },
    { toolName: 'Bash', ruleContent: 'git.*' }
  ],
  behavior: 'allow',
  destination: 'projectSettings'
}
```

**Use Case:** Reset permissions to a known state.

---

##### 3. removeRules

Remove specific permission rules.

```typescript
{
  type: 'removeRules',
  rules: [
    { toolName: 'Bash', ruleContent: 'rm.*' }
  ],
  behavior: 'deny',
  destination: 'session'
}
```

**Use Case:** Revoke previously granted permissions.

---

##### 4. setMode

Change the permission mode.

```typescript
{
  type: 'setMode',
  mode: 'acceptEdits',
  destination: 'session'
}
```

**Use Case:** Switch between permission modes during execution.

---

##### 5. addDirectories

Add directories to the allowed directories list.

```typescript
{
  type: 'addDirectories',
  directories: ['/home/user/project/src', '/home/user/project/tests'],
  destination: 'projectSettings'
}
```

**Use Case:** Grant access to additional directories.

---

##### 6. removeDirectories

Remove directories from the allowed directories list.

```typescript
{
  type: 'removeDirectories',
  directories: ['/tmp'],
  destination: 'session'
}
```

**Use Case:** Revoke access to specific directories.

---

### Permission Update Destinations

| Destination | Scope | Persistence |
|-------------|-------|-------------|
| **session** | Current session only | Lost when session ends |
| **localSettings** | Current project/directory | Stored in local settings file |
| **projectSettings** | Project-wide | Stored in project settings |
| **userSettings** | User-wide (all projects) | Stored in user settings |

---

### Permission Result

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

---

#### Allow Result

```typescript
{
  behavior: 'allow',
  updatedInput: { /* potentially modified tool input */ },
  updatedPermissions: [
    {
      type: 'addRules',
      rules: [{ toolName: 'Bash', ruleContent: 'git.*' }],
      behavior: 'allow',
      destination: 'session'
    }
  ]
}
```

**Fields:**
- `behavior`: Must be 'allow'
- `updatedInput`: Tool input to use (can be modified from original)
- `updatedPermissions`: Optional permission updates to apply (e.g., for "always allow")

**Use Case:** Approve tool usage and optionally update permissions.

---

#### Deny Result

```typescript
{
  behavior: 'deny',
  message: 'Cannot execute this command in production environment',
  interrupt: true
}
```

**Fields:**
- `behavior`: Must be 'deny'
- `message`: Explanation for denial or guidance for the model
- `interrupt`: If true, stop execution entirely. If false/undefined, model can try alternative approach.

**Use Case:** Block tool usage with explanation.

---

### Custom Permission Callbacks

#### CanUseTool Callback

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

**Parameters:**
- `toolName`: Name of the tool requesting permission
- `input`: Tool input parameters
- `options.signal`: AbortSignal for cancellation
- `options.suggestions`: Suggested permission updates for "always allow" flow

**Returns:** Promise resolving to PermissionResult

---

### Permission Patterns

#### Pattern 1: Environment-Based Permissions

```typescript
const canUseTool: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  const isProduction = process.env.NODE_ENV === 'production';

  if (isProduction && toolName === 'Bash') {
    const command = input.command as string;

    // Block destructive commands in production
    if (command.includes('rm') || command.includes('drop')) {
      return {
        behavior: 'deny',
        message: 'Destructive commands are not allowed in production',
        interrupt: true
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

#### Pattern 2: Role-Based Permissions

```typescript
const canUseTool: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  const userRole = await getCurrentUserRole();

  const rolePermissions = {
    admin: ['Bash', 'Edit', 'Write', 'Delete'],
    developer: ['Edit', 'Write', 'Read', 'Bash'],
    viewer: ['Read']
  };

  const allowedTools = rolePermissions[userRole] || [];

  if (!allowedTools.includes(toolName)) {
    return {
      behavior: 'deny',
      message: `${userRole} role does not have permission to use ${toolName}`,
      interrupt: false
    };
  }

  return {
    behavior: 'allow',
    updatedInput: input
  };
};
```

---

#### Pattern 3: Content-Based Validation

```typescript
const canUseTool: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  if (toolName === 'Edit' || toolName === 'Write') {
    const filePath = input.file_path as string;

    // Block editing of critical configuration files
    const criticalFiles = ['.env', 'package.json', 'tsconfig.json'];

    if (criticalFiles.some(file => filePath.endsWith(file))) {
      return {
        behavior: 'deny',
        message: `Cannot modify critical file: ${filePath}. Please review changes manually.`,
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

#### Pattern 4: Interactive Approval with Persistence

```typescript
const canUseTool: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  // Show interactive prompt to user
  const userResponse = await promptUser({
    tool: toolName,
    input: input,
    options: [
      'Allow once',
      'Always allow',
      'Deny',
      'Deny and don\'t ask again'
    ]
  });

  switch (userResponse.choice) {
    case 'Allow once':
      return {
        behavior: 'allow',
        updatedInput: input
      };

    case 'Always allow':
      return {
        behavior: 'allow',
        updatedInput: input,
        updatedPermissions: suggestions // Use SDK's suggestions
      };

    case 'Deny':
      return {
        behavior: 'deny',
        message: 'User denied this operation',
        interrupt: false
      };

    case 'Deny and don\'t ask again':
      return {
        behavior: 'deny',
        message: 'User denied this operation',
        interrupt: false,
        updatedPermissions: [{
          type: 'addRules',
          rules: [{ toolName, ruleContent: JSON.stringify(input) }],
          behavior: 'deny',
          destination: 'session'
        }]
      };
  }
};
```

---

### Permission Examples

#### Example 1: Custom Permission Handler

```typescript
import { query, CanUseTool } from '@anthropic-ai/claude-agent-sdk';

const customPermissionHandler: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  console.log(`Permission requested for ${toolName}`);
  console.log('Input:', input);

  // Auto-approve read operations
  if (toolName === 'Read') {
    return {
      behavior: 'allow',
      updatedInput: input
    };
  }

  // Custom logic for Bash commands
  if (toolName === 'Bash') {
    const command = input.command as string;

    // Allow safe git operations
    if (command.startsWith('git status') || command.startsWith('git log')) {
      return {
        behavior: 'allow',
        updatedInput: input,
        updatedPermissions: [{
          type: 'addRules',
          rules: [{ toolName: 'Bash', ruleContent: 'git (status|log).*' }],
          behavior: 'allow',
          destination: 'session'
        }]
      };
    }

    // Block destructive operations
    if (command.includes('rm -rf')) {
      return {
        behavior: 'deny',
        message: 'Destructive rm -rf commands are not allowed',
        interrupt: true
      };
    }
  }

  // Default: ask user
  return {
    behavior: 'allow',
    updatedInput: input
  };
};

const session = query({
  prompt: 'Check git status and clean up old files',
  options: {
    canUseTool: customPermissionHandler
  }
});
```

---

#### Example 2: Directory-Based Permissions

```typescript
const directoryBasedPermissions: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  const allowedDirectories = [
    '/home/user/projects/myapp/src',
    '/home/user/projects/myapp/tests'
  ];

  if (toolName === 'Edit' || toolName === 'Write') {
    const filePath = input.file_path as string;

    const isAllowed = allowedDirectories.some(dir => filePath.startsWith(dir));

    if (!isAllowed) {
      return {
        behavior: 'deny',
        message: `Cannot modify files outside allowed directories: ${allowedDirectories.join(', ')}`,
        interrupt: false
      };
    }
  }

  return {
    behavior: 'allow',
    updatedInput: input
  };
};

const session = query({
  prompt: 'Refactor the authentication code',
  options: {
    canUseTool: directoryBasedPermissions,
    additionalDirectories: ['/home/user/projects/myapp/src']
  }
});
```

---

#### Example 3: Logging with Permission Tracking

```typescript
const loggingPermissionHandler: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  // Log permission request
  await logPermissionRequest({
    timestamp: new Date().toISOString(),
    toolName,
    input,
    suggestions
  });

  // Implement custom logic
  const result = await customPermissionLogic(toolName, input);

  // Log decision
  await logPermissionDecision({
    timestamp: new Date().toISOString(),
    toolName,
    decision: result.behavior
  });

  return result;
};
```

---

#### Example 4: Multi-Stage Approval

```typescript
const multiStageApproval: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  // Stage 1: Automated checks
  const automatedCheck = await runAutomatedSecurityChecks(toolName, input);

  if (automatedCheck.failed) {
    return {
      behavior: 'deny',
      message: `Automated security check failed: ${automatedCheck.reason}`,
      interrupt: true
    };
  }

  // Stage 2: Risk assessment
  const riskLevel = assessRisk(toolName, input);

  if (riskLevel === 'high') {
    // Stage 3: Human approval required
    const approved = await requestHumanApproval({
      toolName,
      input,
      riskLevel
    });

    if (!approved) {
      return {
        behavior: 'deny',
        message: 'Human reviewer denied this operation',
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

#### Example 5: Permission Mode Switching

```typescript
const session = query({
  prompt: 'Develop a new feature',
  options: {
    permissionMode: 'default' // Start with default
  }
});

// During execution, switch to acceptEdits for code generation phase
for await (const message of session) {
  if (message.type === 'assistant') {
    // Detect when we're in code generation phase
    const content = JSON.stringify(message.message);

    if (content.includes('I will now generate the code')) {
      await session.setPermissionMode('acceptEdits');
      console.log('Switched to acceptEdits mode for code generation');
    }
  }
}
```

---

## Integration Examples

### Complete Example: Hooks + Permissions

```typescript
import {
  query,
  HookCallback,
  CanUseTool,
  PreToolUseHookInput,
  PostToolUseHookInput
} from '@anthropic-ai/claude-agent-sdk';

// Hook: Log all tool usage
const toolLogger: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'PreToolUse') {
    console.log(`[PRE] Tool: ${input.tool_name}, ID: ${toolUseID}`);
  } else if (input.hook_event_name === 'PostToolUse') {
    console.log(`[POST] Tool: ${input.tool_name}, ID: ${toolUseID}`);
  }
  return { continue: true };
};

// Hook: Block specific patterns
const dangerBlocker: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'PreToolUse') {
    const hookInput = input as PreToolUseHookInput;

    if (hookInput.tool_name === 'Bash') {
      const command = hookInput.tool_input as { command?: string };

      if (command.command?.includes('sudo')) {
        return {
          decision: 'block',
          stopReason: 'sudo commands are not allowed',
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'deny',
            permissionDecisionReason: 'Security policy: no sudo'
          }
        };
      }
    }
  }

  return { continue: true };
};

// Permission: Custom approval logic
const customPermissions: CanUseTool = async (toolName, input, { signal, suggestions }) => {
  // Auto-approve safe operations
  const safeTools = ['Read', 'Glob', 'Grep'];
  if (safeTools.includes(toolName)) {
    return {
      behavior: 'allow',
      updatedInput: input
    };
  }

  // Custom approval for other tools
  console.log(`Requesting permission for ${toolName}`);

  return {
    behavior: 'allow',
    updatedInput: input
  };
};

// Create session with both hooks and permissions
const session = query({
  prompt: 'Analyze the codebase and suggest improvements',
  options: {
    permissionMode: 'default',
    canUseTool: customPermissions,
    hooks: {
      PreToolUse: [
        { hooks: [toolLogger, dangerBlocker] }
      ],
      PostToolUse: [
        { hooks: [toolLogger] }
      ]
    }
  }
});

// Process messages
for await (const message of session) {
  if (message.type === 'assistant') {
    console.log('Assistant:', message.message);
  } else if (message.type === 'result') {
    console.log('Result:', message);
  }
}
```

---

## SDK Message Types Related to Hooks and Permissions

### Hook Response Message

```typescript
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

### Permission Denial Tracking

```typescript
export type SDKPermissionDenial = {
  tool_name: string;
  tool_use_id: string;
  tool_input: Record<string, unknown>;
};
```

**Included in Result Messages:**

```typescript
export type SDKResultMessage = SDKMessageBase & {
  // ... other fields
  permission_denials: SDKPermissionDenial[];
};
```

---

## Advanced Patterns

### Pattern: Conditional Hook Execution

```typescript
const conditionalHook: HookCallback = async (input, toolUseID, { signal }) => {
  // Only run during business hours
  const hour = new Date().getHours();
  const isBusinessHours = hour >= 9 && hour <= 17;

  if (!isBusinessHours && input.hook_event_name === 'PreToolUse') {
    return {
      decision: 'block',
      stopReason: 'Operations only allowed during business hours (9 AM - 5 PM)',
      hookSpecificOutput: {
        hookEventName: 'PreToolUse',
        permissionDecision: 'deny',
        permissionDecisionReason: 'Outside business hours'
      }
    };
  }

  return { continue: true };
};
```

---

### Pattern: Permission Caching

```typescript
class PermissionCache {
  private cache = new Map<string, PermissionResult>();

  createCanUseTool(): CanUseTool {
    return async (toolName, input, { signal, suggestions }) => {
      const cacheKey = `${toolName}:${JSON.stringify(input)}`;

      // Check cache
      if (this.cache.has(cacheKey)) {
        return this.cache.get(cacheKey)!;
      }

      // Compute permission
      const result = await this.computePermission(toolName, input);

      // Cache result
      this.cache.set(cacheKey, result);

      return result;
    };
  }

  private async computePermission(
    toolName: string,
    input: Record<string, unknown>
  ): Promise<PermissionResult> {
    // Your permission logic here
    return {
      behavior: 'allow',
      updatedInput: input
    };
  }
}

const permissionCache = new PermissionCache();

const session = query({
  prompt: 'Process all files',
  options: {
    canUseTool: permissionCache.createCanUseTool()
  }
});
```

---

### Pattern: Async Hook Processing

```typescript
const asyncHook: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name === 'PreToolUse') {
    // Return immediately, process asynchronously
    return {
      async: true,
      asyncTimeout: 30000 // 30 seconds timeout
    };
  }

  return { continue: true };
};
```

---

## Summary

### Hook System Quick Reference

| Hook Event | When Triggered | Key Fields | Output Options |
|------------|----------------|------------|----------------|
| PreToolUse | Before tool execution | tool_name, tool_input | permissionDecision, updatedInput |
| PostToolUse | After tool execution | tool_name, tool_response | additionalContext |
| Notification | On notification | message, title | - |
| UserPromptSubmit | On user prompt | prompt | additionalContext |
| SessionStart | Session starts | source | additionalContext |
| SessionEnd | Session ends | reason | - |
| Stop | Agent stops | stop_hook_active | - |
| SubagentStop | Subagent stops | stop_hook_active | - |
| PreCompact | Before compaction | trigger, custom_instructions | - |

---

### Permission System Quick Reference

| Mode | Auto-Approve | Use Case |
|------|--------------|----------|
| default | None | Interactive, safe development |
| acceptEdits | Edits only | Code generation |
| bypassPermissions | All tools | Automation, trusted environments |
| plan | Nothing (planning only) | Strategy, exploration |

---

### Permission Update Types

| Type | Purpose | Example |
|------|---------|---------|
| addRules | Add permissions | Allow git commands |
| replaceRules | Reset permissions | Start fresh |
| removeRules | Revoke permissions | Remove dangerous pattern |
| setMode | Change mode | Switch to acceptEdits |
| addDirectories | Grant directory access | Add src/ folder |
| removeDirectories | Revoke directory access | Remove tmp/ folder |

---

## Additional Resources

- **Official Documentation:** https://docs.claude.com/en/api/agent-sdk/overview
- **Migration Guide:** https://docs.claude.com/en/docs/claude-code/sdk/migration-guide
- **GitHub Issues:** https://github.com/anthropics/claude-agent-sdk-typescript/issues
- **Discord Community:** https://anthropic.com/discord

---

**Document Version:** 1.0
**SDK Version:** @anthropic-ai/claude-agent-sdk@1.x
