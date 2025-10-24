# Claude Agent SDK - Hooks System Complete Reference

**SDK Version**: 0.1.22
**Extraction Date**: 2025-10-24
**Source**: `sdkTypes.d.ts`, analyzed from SDK source

---

## Table of Contents

1. [Overview](#overview)
2. [Hook Events (All 9)](#hook-events-all-9)
3. [Hook Input Structures](#hook-input-structures)
4. [Hook Output Structures](#hook-output-structures)
5. [Hook Execution Lifecycle](#hook-execution-lifecycle)
6. [Advanced Hook Patterns](#advanced-hook-patterns)
7. [Real-World Examples](#real-world-examples)
8. [Gotchas & Best Practices](#gotchas--best-practices)

---

## Overview

The Claude Agent SDK provides a sophisticated hook system with **9 different events** that allow you to intercept, modify, or block operations at critical points in the agent lifecycle.

### Hook System Architecture

```
User Action → Hook Trigger → Hook Callbacks → Decision → Continue/Block/Modify
```

### Hook Types

**Synchronous Hooks** (immediate response):
- PreToolUse
- PostToolUse  
- UserPromptSubmit
- Notification

**Lifecycle Hooks** (session events):
- SessionStart
- SessionEnd
- Stop
- SubagentStop
- PreCompact

---

## Hook Events (All 9)

### 1. PreToolUse

**Triggered**: Before any tool is executed

**Purpose**: 
- Validate tool inputs
- Block dangerous operations
- Modify tool inputs before execution
- Override permission decisions
- Add logging/auditing

**Power Level**: ⚠️ **HIGHEST** - Can completely block operations

**Input Structure**:
```typescript
{
  hook_event_name: 'PreToolUse';
  tool_name: string;
  tool_input: unknown;
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Output Capabilities**:
```typescript
{
  decision?: 'approve' | 'block';  // Override permission system
  systemMessage?: string;           // Add message to conversation
  hookSpecificOutput?: {
    hookEventName: 'PreToolUse';
    permissionDecision?: 'allow' | 'deny' | 'ask';
    permissionDecisionReason?: string;
    updatedInput?: Record<string, unknown>;  // Modify tool input
  }
}
```

**Use Cases**:
- Block `git push --force` to main branch
- Sanitize file paths before operations
- Log all tool usage for audit trail
- Enforce company security policies
- Transform inputs (e.g., relative → absolute paths)

---

### 2. PostToolUse

**Triggered**: After tool execution completes

**Purpose**:
- Add context based on tool results
- Log tool outputs
- Trigger downstream actions
- Analyze results for patterns
- Suppress tool output from model

**Power Level**: 🔵 Medium - Can affect conversation context

**Input Structure**:
```typescript
{
  hook_event_name: 'PostToolUse';
  tool_name: string;
  tool_input: unknown;
  tool_response: unknown;          // Tool execution result
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Output Capabilities**:
```typescript
{
  suppressOutput?: boolean;         // Hide output from model
  systemMessage?: string;           // Add context message
  hookSpecificOutput?: {
    hookEventName: 'PostToolUse';
    additionalContext?: string;    // Add to conversation
  }
}
```

**Use Cases**:
- Add warnings for detected issues in output
- Suppress verbose output from commands
- Extract and format important data
- Trigger notifications based on results
- Log performance metrics

---

### 3. Notification

**Triggered**: When system sends notifications

**Purpose**:
- Intercept system notifications
- Send to external systems (Slack, email, etc.)
- Filter or transform notifications
- Add custom notification routing

**Power Level**: 🟢 Low - Informational only

**Input Structure**:
```typescript
{
  hook_event_name: 'Notification';
  message: string;
  title?: string;
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Use Cases**:
- Send important notifications to Slack
- Filter out noisy notifications
- Add timestamps and metadata
- Route different notification levels differently
- Create notification history log

---

### 4. UserPromptSubmit

**Triggered**: Before user prompt sent to model

**Purpose**:
- Add context to user prompts
- Modify user input
- Block inappropriate prompts
- Inject system instructions
- Add metadata or timestamps

**Power Level**: ⚠️ **HIGH** - Can modify all user input

**Input Structure**:
```typescript
{
  hook_event_name: 'UserPromptSubmit';
  prompt: string;
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Output Capabilities**:
```typescript
{
  continue?: boolean;                // Stop execution if false
  preventContinuation?: boolean;     // Alternative stop flag
  stopReason?: string;               // Reason for stopping
  systemMessage?: string;            // Add system message
  hookSpecificOutput?: {
    hookEventName: 'UserPromptSubmit';
    additionalContext?: string;      // Inject context
  }
}
```

**Use Cases**:
- Add current date/time to prompts
- Inject project-specific context
- Block prompts containing secrets
- Add user identity information
- Transform query format

---

### 5. SessionStart

**Triggered**: When session begins or resumes

**Purpose**:
- Initialize session resources
- Load custom configurations
- Add session context
- Setup monitoring/logging
- Validate environment

**Power Level**: 🟢 Setup - Initialization phase

**Input Structure**:
```typescript
{
  hook_event_name: 'SessionStart';
  source: 'startup' | 'resume' | 'clear' | 'compact';
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Output Capabilities**:
```typescript
{
  systemMessage?: string;           // Add startup message
  hookSpecificOutput?: {
    hookEventName: 'SessionStart';
    additionalContext?: string;     // Add session context
  }
}
```

**Use Cases**:
- Load project-specific settings
- Display welcome message with instructions
- Initialize external service connections
- Log session start for analytics
- Validate API keys and credentials

---

### 6. SessionEnd

**Triggered**: When session terminates

**Purpose**:
- Cleanup resources
- Save session metrics
- Generate session summary
- Close external connections
- Archive session data

**Power Level**: 🟢 Cleanup - Teardown phase

**Input Structure**:
```typescript
{
  hook_event_name: 'SessionEnd';
  reason: ExitReason;               // Why session ended
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Exit Reasons**: 
```typescript
const EXIT_REASONS: string[];  // Defined in SDK
// Examples: "user_exit", "error", "max_turns", "completion"
```

**Use Cases**:
- Generate session summary report
- Upload logs to analytics service
- Clean up temporary files
- Save session metrics
- Send completion notification

---

### 7. Stop

**Triggered**: When user interrupts execution (Ctrl+C)

**Purpose**:
- Handle graceful shutdown
- Save work in progress
- Cancel long-running operations
- Cleanup partial state
- Log interruption reason

**Power Level**: 🔴 Critical - Emergency handling

**Input Structure**:
```typescript
{
  hook_event_name: 'Stop';
  stop_hook_active: boolean;        // Is stop hook running?
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Use Cases**:
- Save current work before exit
- Cancel running background processes
- Clean up temporary resources
- Log interruption for debugging
- Notify external systems of abort

---

### 8. SubagentStop

**Triggered**: When subagent terminates

**Purpose**:
- Monitor subagent completion
- Collect subagent metrics
- Log subagent results
- Handle subagent errors
- Track agent performance

**Power Level**: 🟢 Monitoring - Observability

**Input Structure**:
```typescript
{
  hook_event_name: 'SubagentStop';
  stop_hook_active: boolean;
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Use Cases**:
- Log agent execution time
- Track agent success/failure rates
- Monitor token usage per agent
- Debug agent behavior
- Collect agent performance metrics

---

### 9. PreCompact

**Triggered**: Before conversation context compression

**Purpose**:
- Archive full context before compression
- Customize compression behavior
- Save important information
- Generate pre-compression summary
- Control what gets compressed

**Power Level**: ⚠️ **HIGH** - Last chance before data loss

**Input Structure**:
```typescript
{
  hook_event_name: 'PreCompact';
  trigger: 'manual' | 'auto';       // Manual (/compact) or automatic
  custom_instructions: string | null;
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode?: string;
}
```

**Use Cases**:
- Save full transcript before compression
- Extract key decisions before loss
- Generate summary for reference
- Notify user of compaction
- Apply custom compression rules

---

## Hook Input Structures

### Base Hook Input

All hooks receive these base fields:

```typescript
type BaseHookInput = {
  session_id: string;           // Current session UUID
  transcript_path: string;      // Path to transcript file
  cwd: string;                  // Current working directory
  permission_mode?: string;     // Current permission mode
};
```

### Hook Callback Signature

```typescript
type HookCallback = (
  input: HookInput,              // Event-specific input
  toolUseID: string | undefined, // Tool use ID (if applicable)
  options: {
    signal: AbortSignal;         // Cancellation signal
  }
) => Promise<HookJSONOutput>;
```

### Hook Matcher Structure

```typescript
interface HookCallbackMatcher {
  matcher?: string;              // Tool name pattern (undefined = all)
  hooks: HookCallback[];         // Array of callbacks to execute
}
```

**Matcher Examples**:
```typescript
// Match all tools
{ matcher: undefined, hooks: [callback] }

// Match specific tool
{ matcher: "Bash", hooks: [callback] }

// Match tool pattern (if supported)
{ matcher: "File*", hooks: [callback] }  // FileRead, FileWrite, FileEdit
```

---

## Hook Output Structures

### Synchronous Hook Output

```typescript
type SyncHookJSONOutput = {
  // Flow Control
  continue?: boolean;            // Continue execution (default: true)
  suppressOutput?: boolean;      // Hide output from model
  stopReason?: string;           // Reason for stopping
  
  // Decision Override
  decision?: 'approve' | 'block'; // Override permission decision
  
  // Context Injection
  systemMessage?: string;        // Add message to conversation
  reason?: string;               // Reason for decision
  
  // Hook-Specific Output
  hookSpecificOutput?: 
    | PreToolUseHookOutput
    | PostToolUseHookOutput
    | UserPromptSubmitHookOutput
    | SessionStartHookOutput;
};
```

### Async Hook Output

```typescript
type AsyncHookJSONOutput = {
  async: true;                   // Mark as async
  asyncTimeout?: number;         // Optional timeout (ms)
};
```

**Behavior**:
- Async hooks return immediately
- No output expected
- Used for fire-and-forget operations (logging, notifications)

### Hook-Specific Outputs

#### PreToolUse Specific

```typescript
{
  hookEventName: 'PreToolUse';
  permissionDecision?: 'allow' | 'deny' | 'ask';
  permissionDecisionReason?: string;
  updatedInput?: Record<string, unknown>;  // Modified tool input
}
```

#### PostToolUse Specific

```typescript
{
  hookEventName: 'PostToolUse';
  additionalContext?: string;    // Context added to conversation
}
```

#### UserPromptSubmit Specific

```typescript
{
  hookEventName: 'UserPromptSubmit';
  additionalContext?: string;    // Context injected into prompt
}
```

#### SessionStart Specific

```typescript
{
  hookEventName: 'SessionStart';
  additionalContext?: string;    // Session initialization context
}
```

---

## Hook Execution Lifecycle

### Execution Order

```
1. Hook Trigger → Event detected
2. Hook Matcher → Find matching callbacks
3. Hook Execution → Run callbacks in order
4. Output Merging → Combine multiple hook outputs
5. Decision Application → Apply continue/block/modify decisions
6. Operation Proceeds → Or blocked if decision = 'block'
```

### Multiple Hook Handling

When multiple hooks match:

```typescript
const hooks: HookCallbackMatcher[] = [
  { matcher: "Bash", hooks: [logHook, validateHook] },
  { matcher: undefined, hooks: [globalHook] }
];

// Execution order:
// 1. globalHook (matches all)
// 2. logHook (matches "Bash")
// 3. validateHook (matches "Bash")
```

**Output Merging**:
- All hooks executed sequentially
- Last hook's decision wins
- systemMessages concatenated
- First `continue: false` stops execution

### Timeout Handling

```javascript
const HOOK_DEFAULT_TIMEOUT = 5000;  // 5 seconds
```

**Behavior**:
- Hook killed after timeout
- Warning logged
- Operation continues (fail-open)
- No retry mechanism

**Recommendation**: Keep hooks fast (<1 second)

---

## Advanced Hook Patterns

### Pattern 1: Security Gate

**Use Case**: Block dangerous operations

```typescript
const securityGate: HookCallback = async (input) => {
  if (input.hook_event_name === 'PreToolUse') {
    if (input.tool_name === 'Bash') {
      const command = input.tool_input.command;
      
      // Block force pushes to main
      if (command.includes('git push --force') && 
          command.includes('main')) {
        return {
          decision: 'block',
          systemMessage: 'Blocked: Force push to main is not allowed',
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'deny',
            permissionDecisionReason: 'Force push to main branch prohibited'
          }
        };
      }
    }
  }
  
  return { decision: 'approve' };
};
```

### Pattern 2: Input Sanitization

**Use Case**: Modify tool inputs before execution

```typescript
const pathSanitizer: HookCallback = async (input) => {
  if (input.hook_event_name === 'PreToolUse') {
    if (input.tool_name === 'FileRead' || 
        input.tool_name === 'FileWrite' ||
        input.tool_name === 'FileEdit') {
      
      // Convert relative to absolute paths
      const filePath = input.tool_input.file_path;
      if (!filePath.startsWith('/')) {
        const absolutePath = path.join(input.cwd, filePath);
        
        return {
          decision: 'approve',
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'allow',
            updatedInput: {
              ...input.tool_input,
              file_path: absolutePath
            }
          }
        };
      }
    }
  }
  
  return { decision: 'approve' };
};
```

### Pattern 3: Audit Logging

**Use Case**: Log all tool usage

```typescript
const auditLogger: HookCallback = async (input, toolUseID) => {
  // PreToolUse: Log intent
  if (input.hook_event_name === 'PreToolUse') {
    await logToDatabase({
      event: 'tool_start',
      tool: input.tool_name,
      input: input.tool_input,
      toolUseID,
      timestamp: new Date(),
      session: input.session_id
    });
  }
  
  // PostToolUse: Log result
  if (input.hook_event_name === 'PostToolUse') {
    await logToDatabase({
      event: 'tool_complete',
      tool: input.tool_name,
      result: input.tool_response,
      toolUseID,
      timestamp: new Date(),
      session: input.session_id
    });
  }
  
  return { decision: 'approve' };
};
```

### Pattern 4: Context Injection

**Use Case**: Add dynamic context to prompts

```typescript
const contextInjector: HookCallback = async (input) => {
  if (input.hook_event_name === 'UserPromptSubmit') {
    // Add current date, time, and project info
    const context = [
      `Current time: ${new Date().toISOString()}`,
      `Project: ${getProjectName(input.cwd)}`,
      `Git branch: ${await getGitBranch(input.cwd)}`
    ].join('\n');
    
    return {
      hookSpecificOutput: {
        hookEventName: 'UserPromptSubmit',
        additionalContext: context
      }
    };
  }
  
  return {};
};
```

### Pattern 5: Async Notification

**Use Case**: Send notifications without blocking

```typescript
const slackNotifier: HookCallback = async (input) => {
  // Return immediately, send notification in background
  if (input.hook_event_name === 'Notification') {
    // Fire and forget
    sendToSlack(input.message, input.title).catch(console.error);
    
    return {
      async: true
    };
  }
  
  return {};
};
```

---

## Real-World Examples

### Example 1: Enterprise Security Compliance

```typescript
const complianceHooks: Partial<Record<HookEvent, HookCallbackMatcher[]>> = {
  PreToolUse: [{
    hooks: [async (input) => {
      if (input.tool_name === 'Bash') {
        const command = input.tool_input.command;
        
        // Block production database access
        if (command.includes('psql') && 
            command.includes('production')) {
          await notifySecurityTeam({
            event: 'blocked_production_access',
            user: process.env.USER,
            command
          });
          
          return {
            decision: 'block',
            systemMessage: 'Production database access requires approval'
          };
        }
      }
      
      return { decision: 'approve' };
    }]
  }],
  
  SessionEnd: [{
    hooks: [async (input) => {
      // Generate compliance report
      await generateComplianceReport(input.session_id);
      return {};
    }]
  }]
};
```

### Example 2: Development Workflow Enhancement

```typescript
const devWorkflowHooks: Partial<Record<HookEvent, HookCallbackMatcher[]>> = {
  PostToolUse: [{
    matcher: 'Bash',
    hooks: [async (input) => {
      const command = input.tool_input.command;
      const output = input.tool_response;
      
      // Auto-open PR when tests pass
      if (command.includes('npm test') && 
          output.includes('All tests passed')) {
        return {
          systemMessage: '✅ Tests passed! You can create a PR with /create-pr'
        };
      }
      
      return {};
    }]
  }],
  
  UserPromptSubmit: [{
    hooks: [async (input) => {
      // Inject recent git changes as context
      const changes = await getRecentChanges(input.cwd);
      
      return {
        hookSpecificOutput: {
          hookEventName: 'UserPromptSubmit',
          additionalContext: `Recent changes:\n${changes}`
        }
      };
    }]
  }]
};
```

---

## Gotchas & Best Practices

### Gotchas

1. **Hook Timeout (5 seconds)**:
   - Hooks killed after 5 seconds
   - No warning, silent failure
   - Keep hooks fast

2. **Hook Matcher Syntax**:
   - Not fully documented
   - Pattern matching behavior unclear
   - Use exact tool names

3. **Output Merging**:
   - Multiple hooks executed sequentially
   - Last decision wins
   - Can lead to unexpected behavior

4. **Async Hooks**:
   - No output expected
   - No error handling
   - Fire-and-forget only

5. **Permission Override**:
   - Hooks can bypass all permissions
   - Security risk if misused
   - Use carefully

### Best Practices

1. **Keep Hooks Fast** (<1 second):
   ```typescript
   // ❌ Bad: Slow hook
   const slowHook = async () => {
     await complexDatabaseQuery();  // 3 seconds
   };
   
   // ✅ Good: Fast hook with async followup
   const fastHook = async () => {
     asyncDatabaseLog().catch(console.error);
     return {};
   };
   ```

2. **Handle Errors Gracefully**:
   ```typescript
   const robustHook = async (input) => {
     try {
       await riskyOperation();
       return { decision: 'approve' };
     } catch (error) {
       console.error('Hook error:', error);
       // Fail open: allow operation to proceed
       return { decision: 'approve' };
     }
   };
   ```

3. **Use Specific Matchers**:
   ```typescript
   // ✅ Good: Specific matcher
   { matcher: "Bash", hooks: [bashValidator] }
   
   // ⚠️ Okay: Global matcher
   { matcher: undefined, hooks: [globalLogger] }
   ```

4. **Provide Clear Messages**:
   ```typescript
   return {
     decision: 'block',
     systemMessage: 'Operation blocked: Reason XYZ. Try ABC instead.',
     hookSpecificOutput: {
       hookEventName: 'PreToolUse',
       permissionDecisionReason: 'Detailed technical reason for logs'
     }
   };
   ```

5. **Test Hook Performance**:
   ```typescript
   const timedHook = async (input) => {
     const start = Date.now();
     const result = await yourHookLogic(input);
     const duration = Date.now() - start;
     
     if (duration > 1000) {
       console.warn(`Hook took ${duration}ms - optimize!`);
     }
     
     return result;
   };
   ```

---

## Summary

### Hook Power Ranking

1. **PreToolUse** - ⚠️ HIGHEST (can block/modify operations)
2. **UserPromptSubmit** - ⚠️ HIGH (can modify all input)
3. **PreCompact** - ⚠️ HIGH (last chance before data loss)
4. **PostToolUse** - 🔵 Medium (can affect context)
5. **Stop** - 🔴 Critical (emergency handling)
6. **SessionStart/End** - 🟢 Setup/Cleanup
7. **SubagentStop** - 🟢 Monitoring
8. **Notification** - 🟢 Informational

### When to Use Each Hook

| Hook | Primary Use Cases |
|------|-------------------|
| PreToolUse | Security, validation, input transformation |
| PostToolUse | Logging, context addition, result analysis |
| UserPromptSubmit | Context injection, prompt modification |
| SessionStart | Initialization, setup, welcome messages |
| SessionEnd | Cleanup, reporting, metrics |
| Stop | Graceful shutdown, save work |
| SubagentStop | Agent monitoring, performance tracking |
| PreCompact | Archive, custom compression |
| Notification | External system integration |
