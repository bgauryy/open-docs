# Claude Agent SDK: Hook System and Permission System

**Complete Reference Documentation**

**SDK Version**: 0.1.22
**Package**: @anthropic-ai/claude-agent-sdk

---

## Table of Contents

1. [Hook System](#hook-system)
   - [Overview](#hook-system-overview)
   - [All 15 Hook Events](#all-15-hook-events)
   - [Hook Input Schemas](#hook-input-schemas)
   - [Hook Output Schemas](#hook-output-schemas)
   - [Hook Execution Flow](#hook-execution-flow)
   - [Hook Matcher Patterns](#hook-matcher-patterns)
   - [Hook Callback Interface](#hook-callback-interface)
   - [Hook-Specific Output](#hook-specific-output)
   - [Hook Examples](#hook-examples)

2. [Permission System](#permission-system)
   - [Overview](#permission-system-overview)
   - [All 6 Permission Modes](#all-6-permission-modes)
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

### All 15 Hook Events

```typescript
export declare const HOOK_EVENTS: readonly [
  "PreToolUse",
  "PostToolUse",
  "PostToolUseFailure",
  "Notification",
  "UserPromptSubmit",
  "SessionStart",
  "SessionEnd",
  "Stop",
  "SubagentStart",
  "SubagentStop",
  "PreCompact",
  "PermissionRequest",
  "Setup",
  "TeammateIdle",
  "TaskCompleted"
];

export type HookEvent = (typeof HOOK_EVENTS)[number];
```

#### Hook Event Descriptions

| Event | Triggered When | Use Cases |
|-------|----------------|-----------|
| **PreToolUse** | Before a tool is executed | Permission checks, input validation, tool blocking |
| **PostToolUse** | After a tool completes successfully | Logging, result modification, error handling |
| **PostToolUseFailure** | After a tool execution fails | Error logging, failure handling, retry logic |
| **Notification** | When the agent sends a notification | Custom notification handling, filtering |
| **UserPromptSubmit** | When user submits a prompt | Prompt preprocessing, context injection |
| **SessionStart** | When a session starts or resumes | Initialization, session setup |
| **SessionEnd** | When a session ends | Cleanup, final logging, analytics |
| **Stop** | When the agent is stopped | Graceful shutdown, state saving |
| **SubagentStart** | When a subagent is started | Subagent initialization, tracking |
| **SubagentStop** | When a subagent is stopped | Subagent cleanup |
| **PreCompact** | Before conversation history compaction | Custom compaction logic, history archival |
| **PermissionRequest** | When permission is requested | Custom permission logic, audit logging |
| **Setup** | During system setup/initialization | System initialization, configuration |
| **TeammateIdle** | When a teammate becomes idle | Task reassignment, notification |
| **TaskCompleted** | When a task is completed | Task tracking, metrics, notifications |

#### Event Timing and Runtime Impact (v2.1.42)

This section describes how Claude Code v2.1.42 actually *uses* each hook event: what it matches on, where it runs, and what parts of hook output can affect behavior.

##### Tool lifecycle hooks

- **PreToolUse**: Runs *before* a tool is executed and before final permission gating. It can:
  - override/force a permission outcome (`allow` / `deny` / `ask`)
  - modify tool input (`updatedInput`)
  - inject additional context into the transcript/UI
  - stop continuation (`continue: false`) in integrations that honor it
- **PermissionRequest**: Runs when the runtime needs an allow/deny decision in a context where prompts are avoided or delegated. It can:
  - `allow`/`deny` a specific tool request
  - optionally apply permission updates (`updatedPermissions`)
  - optionally abort execution on deny (`interrupt: true`)
- **PostToolUse**: Runs after a tool completes successfully. It can:
  - inject additional context
  - stop continuation (`continue: false`) for integrations that honor it
  - optionally replace MCP tool output (`updatedMCPToolOutput`) for MCP tools only
- **PostToolUseFailure**: Runs after a tool fails. In v2.1.42 it is primarily for:
  - logging/auditing
  - injecting recovery context (it does not rewrite the tool result)

`tool_use_id` correlates PreToolUse / PostToolUse / PostToolUseFailure for the same tool call.

##### Prompt, session, and system lifecycle hooks

- **UserPromptSubmit**: Runs after user input is captured and before the main model call. It can:
  - block the prompt via a “blocking” hook outcome (e.g., command hook exit code `2`)
  - stop continuation (`continue: false`)
  - inject additional context (commonly used to prepend policy/project context)
- **Setup**: Runs during runtime initialization and maintenance flows (match query is `trigger`). Commonly used for environment checks and initialization-time policies.
- **SessionStart**: Runs when the session starts/resumes/clears/after compaction (match query is `source`). Often used to attach session-wide context.
- **SessionEnd**: Runs when a session ends; commonly executed “outside-REPL” for cleanup/logging (it does not participate in tool permission gating).
- **Stop / SubagentStop**: Runs when stopping an agent or subagent. Some integrations honor `continue: false` to prevent continuation.
- **SubagentStart**: Runs when a subagent is started (match query is `agent_type`); commonly used for audit/constraints.
- **Notification**: Runs when a notification is emitted; commonly executed “outside-REPL” for side effects (it does not block/modify agent execution).

##### Compaction hooks

- **PreCompact**: Runs before compaction. In v2.1.42, successful PreCompact command hook stdout may be used as **new compaction instructions** (concatenated across hooks).

##### Team/task hooks

- **TeammateIdle / TaskCompleted**: Emitted for team/task orchestration signals. In v2.1.42 there is **no match query**, so `matcher` does not filter; all configured hooks for the event will run.

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
  tool_use_id: string;
};
```

**Fields:**
- `tool_name`: Name of the tool about to be executed
- `tool_input`: Input parameters for the tool
- `tool_use_id`: Unique ID for this tool use (correlates Pre/Post/Failure events)

**Use Case:** Intercept tool calls before execution to validate, modify, or block them.

---

#### 2. PostToolUse Hook Input

```typescript
export type PostToolUseHookInput = BaseHookInput & {
  hook_event_name: 'PostToolUse';
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;
  tool_use_id: string;
};
```

**Fields:**
- `tool_name`: Name of the executed tool
- `tool_input`: Input parameters that were used
- `tool_response`: Response returned by the tool
- `tool_use_id`: Unique ID for this tool use (correlates Pre/Post/Failure events)

**Use Case:** Log tool usage, modify responses, or add context based on results.

---

#### 3. Notification Hook Input

```typescript
export type NotificationHookInput = BaseHookInput & {
  hook_event_name: 'Notification';
  message: string;
  title?: string;
  notification_type: string;
};
```

**Fields:**
- `message`: Notification message content
- `title`: Optional notification title
- `notification_type`: Notification type/category string

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
  agent_type?: string;
  model?: string;
};
```

**Fields:**
- `source`: How the session started
  - `startup`: New session
  - `resume`: Resumed from saved state
  - `clear`: Started after clearing history
  - `compact`: Started after compaction
- `agent_type`: Optional agent type for the session (when applicable)
- `model`: Optional model identifier for the session (when applicable)

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
  agent_id: string;
  agent_transcript_path: string;
  agent_type: string;
};
```

**Fields:**
- `stop_hook_active`: Whether stop hooks are currently active
- `agent_id`: Subagent ID
- `agent_transcript_path`: Transcript path for the subagent session
- `agent_type`: Subagent type

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

**Runtime behavior (v2.1.42):**
- PreCompact hooks run before history compaction.
- In the Claude Code runtime, PreCompact is commonly executed in an “outside-REPL” context where hook stdout is treated as **plain text**.
- Any **non-empty stdout** from successful PreCompact hook commands may be concatenated to form new compaction instructions.

---

#### 10. PermissionRequest Hook Input

```typescript
export type PermissionRequestHookInput = BaseHookInput & {
  hook_event_name: 'PermissionRequest';
  tool_name: string;
  tool_input: unknown;
  permission_suggestions?: unknown[]; // PermissionUpdate[] (see Permission System section)
};
```

**Fields:**
- `tool_name`: Tool being requested
- `tool_input`: Proposed tool input
- `permission_suggestions`: Optional suggested permission updates (for “remember this decision” UX)

**Use Case:** Centralize auditing / policy for permission prompts, or enforce additional constraints before a prompt is shown.

---

#### 11. PostToolUseFailure Hook Input

```typescript
export type PostToolUseFailureHookInput = BaseHookInput & {
  hook_event_name: 'PostToolUseFailure';
  tool_name: string;
  tool_input: unknown;
  tool_use_id: string;
  error: string;
  is_interrupt?: boolean;
};
```

**Fields:**
- `tool_name`: Tool that failed
- `tool_input`: Tool input that was used
- `tool_use_id`: Unique tool-use ID (correlates with Pre/Post events)
- `error`: Error message/string
- `is_interrupt`: Optional flag indicating an interrupt-style failure

**Use Case:** Error logging, failure handling, custom retries, or surfacing richer context when a tool fails.

---

#### 12. Setup Hook Input

```typescript
export type SetupHookInput = BaseHookInput & {
  hook_event_name: 'Setup';
  trigger: 'init' | 'maintenance';
};
```

**Fields:**
- `trigger`: Why setup hooks are running (`init` vs `maintenance`)

**Use Case:** Initialization-time policies and environment checks.

---

#### 13. SubagentStart Hook Input

```typescript
export type SubagentStartHookInput = BaseHookInput & {
  hook_event_name: 'SubagentStart';
  agent_id: string;
  agent_type: string;
};
```

**Fields:**
- `agent_id`: Subagent ID
- `agent_type`: Subagent type

**Use Case:** Track subagent lifecycle and attach additional context/logging.

---

#### 14. TeammateIdle Hook Input

```typescript
export type TeammateIdleHookInput = BaseHookInput & {
  hook_event_name: 'TeammateIdle';
  teammate_name: string;
  team_name: string;
};
```

**Fields:**
- `teammate_name`: Teammate identifier/name
- `team_name`: Team name

**Use Case:** Team orchestration signals (reassignment, notifications) when a teammate becomes idle.

---

#### 15. TaskCompleted Hook Input

```typescript
export type TaskCompletedHookInput = BaseHookInput & {
  hook_event_name: 'TaskCompleted';
  task_id: string;
  task_subject: string;
  task_description?: string;
  teammate_name?: string;
  team_name?: string;
};
```

**Fields:**
- `task_id`: Completed task ID
- `task_subject`: Task title/subject
- `task_description`: Optional task description
- `teammate_name`: Optional teammate (team mode)
- `team_name`: Optional team name

**Use Case:** Task tracking, metrics, and post-completion automation.

---

#### Union Type: HookInput

```typescript
export type HookInput =
  | PreToolUseHookInput
  | PostToolUseHookInput
  | PostToolUseFailureHookInput
  | NotificationHookInput
  | UserPromptSubmitHookInput
  | SessionStartHookInput
  | SessionEndHookInput
  | StopHookInput
  | SubagentStartHookInput
  | SubagentStopHookInput
  | PreCompactHookInput
  | PermissionRequestHookInput
  | SetupHookInput
  | TeammateIdleHookInput
  | TaskCompletedHookInput;
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

**Use Case:** Indicates that a hook is running out-of-band and the runtime should not expect immediate, structured control output.

**Notes (v2.1.42 runtime):**
- There are two related “async” concepts:
  1. **Config-level async command hooks** (hook definition has `async: true`): the runtime backgrounds the process and later emits **async hook response attachments** by scanning the hook’s stdout for a JSON line.
  2. **Output-level `{ async: true }`** (hook returns/prints `AsyncHookJSONOutput`): treated as “no synchronous control output” for that hook result.
- If `asyncTimeout` is omitted, the runtime uses a default of **15000ms**.

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
    additionalContext?: string;
  } | {
    hookEventName: 'UserPromptSubmit';
    additionalContext?: string;
  } | {
    hookEventName: 'SessionStart';
    additionalContext?: string;
  } | {
    hookEventName: 'Setup';
    additionalContext?: string;
  } | {
    hookEventName: 'SubagentStart';
    additionalContext?: string;
  } | {
    hookEventName: 'PostToolUse';
    additionalContext?: string;
    updatedMCPToolOutput?: unknown;
  } | {
    hookEventName: 'PostToolUseFailure';
    additionalContext?: string;
  } | {
    hookEventName: 'Notification';
    additionalContext?: string;
  } | {
    hookEventName: 'PermissionRequest';
    decision: {
      behavior: 'allow';
      updatedInput?: Record<string, unknown>;
      updatedPermissions?: unknown[]; // PermissionUpdate[] (see Permission System section)
    } | {
      behavior: 'deny';
      message?: string;
      interrupt?: boolean;
    };
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
1. Runtime constructs a HookInput payload (BaseHookInput + event-specific fields)
   ↓
2. Runtime selects hooks registered for the event type
   - Optional matcher filters are applied using an event-specific “match query” string
   - Hooks are de-duplicated by their command/prompt identity
   ↓
3. Hooks execute in a deterministic type order:
   command → prompt → agent → callback → function
   ↓
4. Command hooks may print JSON to stdout; callback hooks return objects directly
   ↓
5. Runtime aggregates results across hooks:
   - permission decisions (deny > ask > allow)
   - updated tool input (only applied when allowed/asked)
   - additional context snippets
   - stop/prevent-continuation signals (event-dependent)
   ↓
6. Event-specific integration applies the parts it supports
```

#### Output processing semantics (v2.1.42)

When a hook prints JSON (command hooks) or returns an object (callback hooks), the runtime interprets it with the following semantics:

- `continue: false` stops the current flow (and may set a user-visible `stopReason`).
- `systemMessage` injects a system-level message into the transcript/UI.
- `suppressOutput: true` hides hook stdout from the transcript (useful for noisy hooks).
- `decision: "approve" | "block"` is a legacy mechanism used by some hook types to translate into allow/deny behavior. For **PreToolUse**, prefer `hookSpecificOutput.permissionDecision`.
- `hookSpecificOutput` must include a matching `hookEventName`. The runtime may reject hook output if the event name is incorrect.
- For `hookEventName: "PermissionRequest"`, the hook can return a structured allow/deny `decision`, optionally including:
  - `updatedInput` (tool input to use if allowed)
  - `updatedPermissions` (permission updates to apply if allowed)

Tool hooks correlate across phases using `tool_use_id` (present on PreToolUse/PostToolUse/PostToolUseFailure payloads).

#### Command hook exit code semantics (v2.1.42)

When running a **command hook** (a hook that executes a shell command), the runtime treats exit codes as:
- `0`: success
- `2`: **blocking** (produces a blocking error and is treated as a “hard stop” for some events)
- anything else: non-blocking error (reported, but typically does not stop the session)

#### Matcher “match query” by event (v2.1.42)

Matchers are evaluated against a per-event query string:

| Event | Match Query Used For `matcher` | Notes |
|---|---|---|
| PreToolUse / PostToolUse / PostToolUseFailure / PermissionRequest | `tool_name` | Matcher is typically a tool name (e.g. `Bash`, `Edit`). |
| SessionStart | `source` | One of `startup` / `resume` / `clear` / `compact`. |
| Setup | `trigger` | `init` or `maintenance`. |
| PreCompact | `trigger` | `manual` or `auto`. |
| Notification | `notification_type` | Free-form notification type string. |
| SessionEnd | `reason` | A value from `EXIT_REASONS`. |
| SubagentStart / SubagentStop | `agent_type` | Allows targeting a specific agent type. |
| UserPromptSubmit / Stop / TeammateIdle / TaskCompleted | *(none)* | No match query is provided; `matcher` does not filter (all configured hooks run). |

#### Outside-REPL hook execution (v2.1.42)

Some events are commonly executed in an “outside-REPL” context where hooks do **not** participate in permission gating or flow control:
- `Notification` hooks are executed for side effects (their output is not used to block/modify behavior).
- `SessionEnd` hooks are executed for cleanup/logging (failures may be written to stderr).
- `PreCompact` hooks are executed to generate compaction instructions; see the PreCompact section above.

#### Async command hook responses (v2.1.42)

When a **command hook definition** is configured as `async: true`, the runtime backgrounds the hook process and later emits an attachment of type `async_hook_response` containing:
- the hook identity (`processId`, `hookName`, `hookEvent`, optional `toolName`)
- raw `stdout`/`stderr`/`exitCode`
- an extracted `response` object (the runtime scans stdout for a JSON line and takes the first JSON object that does **not** include an `async` field)

Async hook responses are informational; they arrive after the triggering action and should not be relied on to gate the original tool execution.

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
  additionalContext?: string;
}
```

**Fields:**
- `permissionDecision`: Override permission system decision
- `permissionDecisionReason`: Explanation for the decision
- `updatedInput`: Modified tool input to use instead of original
- `additionalContext`: Optional context text to inject for this hook result

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
  updatedMCPToolOutput?: unknown;
}
```

**Fields:**
- `additionalContext`: Context to add based on tool results
- `updatedMCPToolOutput`: Optional replacement/patch output for MCP tools

**Use Case:** Provide guidance based on tool output, error handling instructions.

---

#### Setup Hook-Specific Output

```typescript
{
  hookEventName: 'Setup';
  additionalContext?: string;
}
```

**Use Case:** Attach initialization-time context (policies, environment notes) to the session.

---

#### SubagentStart Hook-Specific Output

```typescript
{
  hookEventName: 'SubagentStart';
  additionalContext?: string;
}
```

**Use Case:** Add context when spawning subagents (tracking, audit, constraints).

---

#### PostToolUseFailure Hook-Specific Output

```typescript
{
  hookEventName: 'PostToolUseFailure';
  additionalContext?: string;
}
```

**Use Case:** Provide context and recovery instructions when a tool fails.

**Runtime note (v2.1.42):** PostToolUseFailure runs after the tool has already failed, so it is not a retry mechanism by itself. It is primarily used to surface diagnostics and recovery context.

---

#### Notification Hook-Specific Output

```typescript
{
  hookEventName: 'Notification';
  additionalContext?: string;
}
```

**Use Case:** Attach additional context for notification handling/routing.

**Runtime note (v2.1.42):** In the Claude Code CLI runtime, Notification hooks are typically executed outside the main REPL hook pipeline. Treat them as side-effect hooks (logging, routing, integrations) rather than a way to inject context that affects tool gating.

---

#### PermissionRequest Hook-Specific Output

```typescript
{
  hookEventName: 'PermissionRequest';
  decision: {
    behavior: 'allow';
    updatedInput?: Record<string, unknown>;
    updatedPermissions?: PermissionUpdate[];
  } | {
    behavior: 'deny';
    message?: string;
    interrupt?: boolean;
  };
}
```

**Use Case:** Implement policy-driven permission decisions and optionally persist allow/deny rules.

**Runtime behavior (v2.1.42):**
- Returning `behavior: "allow"` can replace tool input (`updatedInput`) and may apply permission updates (`updatedPermissions`).
- Returning `behavior: "deny"` can optionally abort the current execution (`interrupt: true`) in contexts that honor interrupts.

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

#### Example 7: PermissionRequest - Deny and Interrupt in Headless Contexts

```typescript
import { HookCallback, PermissionRequestHookInput } from '@anthropic-ai/claude-agent-sdk';

const denyBashInHeadlessMode: HookCallback = async (input) => {
  if (input.hook_event_name !== 'PermissionRequest') return { continue: true };

  const hookInput = input as PermissionRequestHookInput;

  if (hookInput.tool_name === 'Bash') {
    return {
      continue: true,
      hookSpecificOutput: {
        hookEventName: 'PermissionRequest',
        decision: {
          behavior: 'deny',
          message: 'Bash is not allowed in this execution context',
          interrupt: true
        }
      }
    };
  }

  return { continue: true };
};
```

---

#### Example 8: PostToolUseFailure - Add Recovery Context

```typescript
import { HookCallback, PostToolUseFailureHookInput } from '@anthropic-ai/claude-agent-sdk';

const addFailureRecoveryHints: HookCallback = async (input) => {
  if (input.hook_event_name !== 'PostToolUseFailure') return { continue: true };

  const hookInput = input as PostToolUseFailureHookInput;

  return {
    continue: true,
    hookSpecificOutput: {
      hookEventName: 'PostToolUseFailure',
      additionalContext: `Tool ${hookInput.tool_name} failed. Error: ${hookInput.error}\n\nSuggested next step: re-check inputs and permissions.`
    }
  };
};
```

---

## Permission System

### Permission System Overview

The Claude Agent SDK includes a sophisticated permission system that controls tool usage and file access. It supports multiple modes, custom callbacks, and granular permission rules.

#### Permission evaluation lifecycle (v2.1.42 runtime)

At a high level, the Claude Code runtime evaluates tool permissions in this order:

1. **PreToolUse hooks** run first and may:
   - force a permission outcome (`allow` / `deny` / `ask`)
   - rewrite tool input (`updatedInput`)
2. The runtime evaluates **permission rules** and the tool’s own `checkPermissions` behavior.
3. If the result is `ask`:
   - In `dontAsk` mode, the runtime converts `ask` → **deny** (no prompt).
   - In headless/async contexts where prompts are avoided, the runtime may run **PermissionRequest hooks** to obtain an allow/deny decision (and optionally apply `updatedPermissions`).
   - In interactive contexts, the runtime can present a permission prompt to the user.
4. If the mode is `bypassPermissions` (and bypass is available), the runtime may allow without prompting.

---

### All 6 Permission Modes

```typescript
export type PermissionMode =
  | 'default'
  | 'acceptEdits'
  | 'bypassPermissions'
  | 'delegate'
  | 'dontAsk'
  | 'plan';
```

#### Mode Descriptions

| Mode | Behavior | Use Case |
|------|----------|----------|
| **default** | Standard permission checking. User is prompted for tool usage that requires permission. | Interactive development, code review |
| **acceptEdits** | Auto-approve edit operations (Edit, Write tools). User still prompted for other tools. | Code generation, refactoring tasks |
| **bypassPermissions** | Allow tool execution without prompting (only when bypass mode is available). | Trusted/controlled environments |
| **delegate** | Restrict available tools to a collaboration-focused subset and delegate approvals externally. | Enterprise teammate workflows |
| **dontAsk** | Never prompt. If a tool would require approval, deny it instead of asking. | Headless/async contexts where prompts are unavailable |
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
- Tool executions are allowed without prompting **only when bypass mode is available**.
- Availability can be disabled by policy / feature gate and may be forced back to `default`.
- This is intentionally “dangerous”: use only in trusted environments.

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

##### 4. delegate Mode
```typescript
permissionMode: 'delegate'
```

**Behavior:**
- The runtime may restrict available tools to a limited set intended for team/task orchestration.
- Approval decisions are expected to be handled by an external system or callback, not by interactive prompts.
- This mode is typically used for enterprise teammate workflows.

**Delegate tool allowlist (v2.1.42 runtime)**:
- Team tools: `TeamCreate`, `TeamDelete`, `SendMessage`
- Task tools: `TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`
- Agent orchestration: `Task` (subagent tool)

**Example Use Case:**
```typescript
const session = query({
  prompt: 'Update production database',
  options: {
    permissionMode: 'delegate'
  }
});
```

Note: Delegated approvals are typically implemented via the `PermissionRequest` hook event and/or an MCP tool used for prompting in print mode (see `permissionPromptToolName` / `--permission-prompt-tool`).

---

##### 5. dontAsk Mode
```typescript
permissionMode: 'dontAsk'
```

**Behavior:**
- No permission prompts are shown.
- If a tool would otherwise return `ask`, the runtime converts that into a `deny` with a mode-based decision reason.
- This mode is designed for contexts where prompting is impossible or undesirable (e.g., background/async agents).

**Example Use Case:**
```typescript
// Used internally by claude-code-guide agent
{
  agentType: "claude-code-guide",
  permissionMode: "dontAsk",  // Avoid prompts; will deny if a prompt would be required
  tools: ["Glob", "Grep", "Read", "WebFetch", "WebSearch"]
}
```

---

##### 6. plan Mode

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
