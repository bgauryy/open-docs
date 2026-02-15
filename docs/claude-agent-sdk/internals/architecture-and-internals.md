# Claude Agent SDK - Architecture & Internals

**SDK Version**: 0.1.22
**Source**: SDK source code analysis

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Components](#core-components)
3. [Message Flow & Lifecycle](#message-flow--lifecycle)
4. [Tool Execution Pipeline](#tool-execution-pipeline)
5. [Permission Resolution System](#permission-resolution-system)
6. [Hook Execution Engine](#hook-execution-engine)
7. [Session Management](#session-management)
8. [State Management](#state-management)
9. [MCP Integration Architecture](#mcp-integration-architecture)
10. [Performance Characteristics](#performance-characteristics)
11. [Internal Constants & Limits](#internal-constants--limits)
12. [Implementation Patterns](#implementation-patterns)

---

## Architecture Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  (Your Code using query() API)                           │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│              Claude Agent SDK (Public API)               │
│  • query()                                               │
│  • tool()                                                │
│  • createSdkMcpServer()                                  │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│               Core Engine (Internal)                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Message    │  │  Tool        │  │  Permission  │  │
│  │   Manager    │  │  Executor    │  │  Resolver    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Hook       │  │  Session     │  │   State      │  │
│  │   Engine     │  │  Manager     │  │   Manager    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│              External Integrations                       │
│  ┌─────────────────────────────────────────────┐        │
│  │  Anthropic API (Claude)                     │        │
│  │  • Streaming Messages API                   │        │
│  │  • Tool Use Protocol                        │        │
│  └─────────────────────────────────────────────┘        │
│  ┌─────────────────────────────────────────────┐        │
│  │  MCP Servers                                │        │
│  │  • stdio Transport (Process)                │        │
│  │  • HTTP/SSE Transport (Network)             │        │
│  │  • SDK Transport (In-Process)               │        │
│  └─────────────────────────────────────────────┘        │
│  ┌─────────────────────────────────────────────┐        │
│  │  File System                                │        │
│  │  • Session Storage (~/.claude/sessions/)    │        │
│  │  • Configuration (~/.claude/settings.json)  │        │
│  │  • File Snapshots (for Edit tool)          │        │
│  └─────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────┘
```

### Process Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Node.js Process (Your Application)                     │
│  ┌──────────────────────────────────────────┐           │
│  │  Claude Agent SDK Module                 │           │
│  │  • query() execution                     │           │
│  │  • Tool execution                        │           │
│  │  • Hook callbacks                        │           │
│  │  • State management                      │           │
│  └──────────────────────────────────────────┘           │
│                     ↓ ↑                                  │
│         [stdio/http] │ │ [http/https]                   │
└─────────────────────┼─┼────────────────────────────────┘
                      │ │
        ┌─────────────┘ └─────────────┐
        │                              │
┌───────▼────────┐            ┌───────▼────────┐
│  MCP Server    │            │  Anthropic API │
│  Process       │            │  (Claude)      │
│  (stdio)       │            │                │
└────────────────┘            └────────────────┘
```

---

## Core Components

### 1. Query Manager

**Responsibility**: Orchestrate entire conversation lifecycle

**Key Functions**:
```typescript
// Internal structure (conceptual)
class QueryManager {
  private messageQueue: SDKMessage[] = [];
  private conversationState: ConversationState;
  private toolExecutor: ToolExecutor;
  private permissionResolver: PermissionResolver;
  private hookEngine: HookEngine;
  
  async *execute(
    prompt: string | AsyncIterable<SDKUserMessage>,
    options: Options
  ): AsyncGenerator<SDKMessage> {
    // Initialize session
    const session = await this.initializeSession(options);
    
    // Initialize system message
    yield this.createSystemMessage(session);
    
    // Main conversation loop
    while (!this.shouldStop()) {
      // Get user message
      const userMessage = await this.getUserMessage(prompt);
      
      // Execute hooks
      await this.hookEngine.trigger('UserPromptSubmit', userMessage);
      
      // Send to API
      const stream = await this.sendToAPI(userMessage, options);
      
      // Process streaming response
      for await (const chunk of stream) {
        yield* this.processStreamChunk(chunk);
      }
      
      // Check turn limit
      if (this.turnCount >= options.maxTurns) break;
    }
    
    // Yield final result
    yield this.createResultMessage();
  }
}
```

### 2. Tool Executor

**Responsibility**: Execute tools with validation and error handling

**Execution Pipeline**:
```
1. Tool Invocation Request
     ↓
2. Pre-Hook Execution (PreToolUse)
     ↓
3. Permission Check
     ↓
4. Tool Validation (input schema)
     ↓
5. Tool Execution
     ↓
6. Result Validation
     ↓
7. Post-Hook Execution (PostToolUse)
     ↓
8. Result Return
```

**Internal Structure**:
```typescript
class ToolExecutor {
  private tools: Map<string, ToolDefinition>;
  private permissionResolver: PermissionResolver;
  private hookEngine: HookEngine;
  
  async execute(
    toolName: string,
    toolInput: unknown,
    context: ExecutionContext
  ): Promise<ToolResult> {
    // 1. Pre-hook
    const preHookResult = await this.hookEngine.trigger('PreToolUse', {
      tool_name: toolName,
      tool_input: toolInput,
      ...context
    });
    
    if (preHookResult.decision === 'block') {
      throw new PermissionDeniedError(preHookResult.reason);
    }
    
    // 2. Permission check
    const permission = await this.permissionResolver.check(
      toolName,
      toolInput,
      context
    );
    
    if (permission.behavior === 'deny') {
      throw new PermissionDeniedError(permission.message);
    }
    
    // 3. Validate input
    const tool = this.tools.get(toolName);
    if (!tool) {
      throw new ToolNotFoundError(toolName);
    }
    
    const validatedInput = this.validateInput(tool, toolInput);
    
    // 4. Execute tool
    let result: ToolResult;
    try {
      result = await tool.execute(validatedInput, context);
    } catch (error) {
      result = {
        isError: true,
        content: [{ type: 'text', text: `Error: ${error.message}` }]
      };
    }
    
    // 5. Post-hook
    await this.hookEngine.trigger('PostToolUse', {
      tool_name: toolName,
      tool_input: toolInput,
      tool_response: result,
      ...context
    });
    
    return result;
  }
}
```

### 3. Permission Resolver

**Responsibility**: Multi-level permission resolution

**Resolution Algorithm**:
```typescript
class PermissionResolver {
  private configLayers: ConfigLayer[] = [
    'session',
    'local',
    'project',
    'user',
    'policy',
    'mode'
  ];
  
  async check(
    toolName: string,
    toolInput: unknown,
    context: PermissionContext
  ): Promise<PermissionResult> {
    // Check each layer in order
    for (const layer of this.configLayers) {
      const rules = this.getRules(layer, context);
      
      for (const rule of rules) {
        if (this.matchesRule(rule, toolName, toolInput)) {
          return {
            behavior: rule.behavior,
            layer,
            rule
          };
        }
      }
    }
    
    // No rule matched - use mode default
    return this.getModeDefault(context.permissionMode, toolName);
  }
  
  private matchesRule(
    rule: PermissionRule,
    toolName: string,
    toolInput: unknown
  ): boolean {
    // Tool name match
    if (rule.tool && !this.matchPattern(rule.tool, toolName)) {
      return false;
    }
    
    // Path match (for file operations)
    if (rule.path && toolInput.file_path) {
      if (!this.matchGlob(rule.path, toolInput.file_path)) {
        return false;
      }
    }
    
    // Command match (for Bash)
    if (rule.command && toolInput.command) {
      if (!this.matchPattern(rule.command, toolInput.command)) {
        return false;
      }
    }
    
    return true;
  }
}
```

### 4. Hook Engine

**Responsibility**: Execute lifecycle hooks at appropriate points

**Hook Execution Model**:
```typescript
class HookEngine {
  private hooks: Map<HookEvent, HookCallback[]> = new Map();
  private readonly HOOK_TIMEOUT = 5000; // 5 seconds
  
  async trigger(
    event: HookEvent,
    input: HookInput
  ): Promise<HookOutput> {
    const callbacks = this.hooks.get(event) || [];
    
    // Execute hooks sequentially
    let combinedOutput: HookOutput = { decision: 'approve' };
    
    for (const callback of callbacks) {
      const controller = new AbortController();
      const timeout = setTimeout(() => {
        controller.abort();
        console.warn(`Hook timeout after ${this.HOOK_TIMEOUT}ms`);
      }, this.HOOK_TIMEOUT);
      
      try {
        const output = await callback(input, undefined, {
          signal: controller.signal
        });
        
        clearTimeout(timeout);
        
        // Merge outputs
        combinedOutput = this.mergeOutputs(combinedOutput, output);
        
        // If hook blocks, stop execution
        if (output.decision === 'block') {
          return combinedOutput;
        }
        
      } catch (error) {
        clearTimeout(timeout);
        console.error('Hook error:', error);
        // Continue to next hook (fail open)
      }
    }
    
    return combinedOutput;
  }
  
  private mergeOutputs(a: HookOutput, b: HookOutput): HookOutput {
    return {
      decision: b.decision || a.decision,
      systemMessage: [a.systemMessage, b.systemMessage]
        .filter(Boolean)
        .join('\n'),
      hookSpecificOutput: b.hookSpecificOutput || a.hookSpecificOutput
    };
  }
}
```

---

## Message Flow & Lifecycle

### Complete Message Flow

```
┌─────────────────────────────────────────────────────────┐
│  1. User Input                                          │
│     → SDKUserMessage created                            │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  2. UserPromptSubmit Hook                               │
│     → Can add context or modify prompt                  │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  3. API Request                                         │
│     → Send to Claude API with streaming                 │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│  4. Stream Processing                                   │
│     → Parse streaming events                            │
│     → message_start, content_block_start, etc.          │
│     → Yield SDKPartialAssistantMessage                  │
└────────────────────┬────────────────────────────────────┘
                     │
                  ┌──▼──┐
                  │Tool?│
                  └──┬──┘
                     │
        Yes ┌────────┴────────┐ No
            │                 │
┌───────────▼──────────┐   ┌──▼──────────────────────────┐
│  5. Tool Execution   │   │  5. Text Content            │
│     → PreToolUse     │   │     → Yield text blocks     │
│     → Permission     │   │     → Continue streaming    │
│     → Execute        │   └─────────────────────────────┘
│     → PostToolUse    │
│     → Return result  │
└──────────┬───────────┘
           │
┌──────────▼───────────────────────────────────────────────┐
│  6. Continue Streaming                                   │
│     → Process more content blocks                        │
│     → May invoke more tools                              │
│     → Yield SDKPartialAssistantMessage continuously      │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  7. Message Complete                                     │
│     → message_stop event                                 │
│     → Yield SDKAssistantMessage (complete)               │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  8. Turn Complete                                        │
│     → Check maxTurns limit                               │
│     → Check stop reason                                  │
└────────────────────┬─────────────────────────────────────┘
                     │
                  ┌──▼────┐
                  │Done?  │
                  └──┬────┘
                     │
          No ┌───────┴───────┐ Yes
             │               │
        ┌────▼────┐    ┌────▼─────────────────────────────┐
        │Continue │    │  9. Yield SDKResultMessage       │
        │Loop     │    │     → Usage stats                │
        └─────────┘    │     → Cost calculation           │
                       │     → Performance metrics        │
                       └──────────────────────────────────┘
```

### Message Type Transitions

```
SDKSystemMessage (init)
   ↓
SDKUserMessage
   ↓
SDKPartialAssistantMessage (stream_event)
   ↓ (many)
SDKAssistantMessage (complete)
   ↓
[If tools used]
   SDKPartialAssistantMessage (tool results)
   ↓
   SDKAssistantMessage (continue)
   ↓
SDKResultMessage (final)
```

---

## Tool Execution Pipeline

### Detailed Execution Stages

#### Stage 1: Invocation
```typescript
// Model requests tool use
{
  type: 'tool_use',
  id: 'toolu_abc123',
  name: 'FileRead',
  input: {
    file_path: '/workspace/src/app.ts'
  }
}
```

#### Stage 2: Pre-Hook Execution
```typescript
// PreToolUse hook triggered
const hookInput: PreToolUseHookInput = {
  hook_event_name: 'PreToolUse',
  tool_name: 'FileRead',
  tool_input: {
    file_path: '/workspace/src/app.ts'
  },
  session_id: 'session_xyz',
  transcript_path: '~/.claude/sessions/session_xyz/transcript.json',
  cwd: '/workspace',
  permission_mode: 'default'
};

// Hook can:
// - Approve (continue)
// - Block (stop execution)
// - Modify input
const hookOutput = await executePreToolUseHooks(hookInput);
```

#### Stage 3: Permission Resolution
```typescript
// Check permissions through 6-level cascade
const permission = await permissionResolver.check(
  'FileRead',
  { file_path: '/workspace/src/app.ts' },
  context
);

// Results in: 'allow' | 'deny' | 'ask'
```

#### Stage 4: Input Validation
```typescript
// Validate against tool schema (from sdk-tools.d.ts)
const schema = {
  file_path: z.string(),
  offset: z.number().optional(),
  limit: z.number().optional()
};

const validated = schema.parse(toolInput);
```

#### Stage 5: Enforcement Checks
```typescript
// Read-before-write check
if (toolName === 'FileEdit' || toolName === 'FileWrite') {
  const hasRead = sessionState.hasReadFile(input.file_path);
  if (!hasRead) {
    throw new Error('Must read file before editing');
  }
}

// Edit uniqueness check
if (toolName === 'FileEdit') {
  const fileContent = await readFile(input.file_path);
  const occurrences = countOccurrences(fileContent, input.old_string);
  
  if (occurrences > 1 && !input.replace_all) {
    throw new Error('old_string must be unique or use replace_all');
  }
}
```

#### Stage 6: Execution
```typescript
// Execute actual tool
const result = await tools[toolName].execute(validated, context);

// Example for FileRead:
const content = await fs.readFile(input.file_path, 'utf-8');
const lines = content.split('\n');

// Apply limits
const offset = input.offset || 0;
const limit = input.limit || 2000; // Default 2000 lines
const selectedLines = lines.slice(offset, offset + limit);

// Per-line truncation
const truncated = selectedLines.map(line => 
  line.length > 2000 ? line.slice(0, 2000) : line
);

return {
  content: truncated.join('\n'),
  isError: false
};
```

#### Stage 7: Post-Hook Execution
```typescript
// PostToolUse hook triggered
const postHookInput: PostToolUseHookInput = {
  hook_event_name: 'PostToolUse',
  tool_name: 'FileRead',
  tool_input: { file_path: '/workspace/src/app.ts' },
  tool_response: result,
  session_id: 'session_xyz',
  transcript_path: '~/.claude/sessions/session_xyz/transcript.json',
  cwd: '/workspace',
  permission_mode: 'default'
};

// Hook can:
// - Add context
// - Suppress output
// - Log for audit
const postHookOutput = await executePostToolUseHooks(postHookInput);
```

#### Stage 8: Result Return
```typescript
// Format result for API
return {
  type: 'tool_result',
  tool_use_id: 'toolu_abc123',
  content: result.content,
  is_error: result.isError
};
```

---

## Permission Resolution System

### Resolution Flow Diagram

```
Permission Request (tool_name, tool_input)
          │
          ▼
    ┌─────────────┐
    │ Hook Check  │ (PreToolUse hook)
    └─────┬───────┘
          │ (If not blocked)
          ▼
    ┌─────────────┐
    │ Session     │ Layer 1 (highest priority)
    │ Rules       │
    └─────┬───────┘
          │ (No match)
          ▼
    ┌─────────────┐
    │ Local       │ Layer 2
    │ Settings    │ (.claude/ in cwd)
    └─────┬───────┘
          │ (No match)
          ▼
    ┌─────────────┐
    │ Project     │ Layer 3
    │ Settings    │ (.claude/ in git root)
    └─────┬───────┘
          │ (No match)
          ▼
    ┌─────────────┐
    │ User        │ Layer 4
    │ Settings    │ (~/.claude/)
    └─────┬───────┘
          │ (No match)
          ▼
    ┌─────────────┐
    │ Policy      │ Layer 5
    │ Settings    │ (enterprise)
    └─────┬───────┘
          │ (No match)
          ▼
    ┌─────────────┐
    │ Permission  │ Layer 6 (fallback)
    │ Mode        │ (default/acceptEdits/etc)
    └─────┬───────┘
          │
          ▼
    ┌─────────────┐
    │ Decision:   │
    │ allow/deny/ │
    │ ask         │
    └─────────────┘
```

### Rule Matching Algorithm

```typescript
interface PermissionRule {
  tool?: string;        // Tool name or pattern
  path?: string;        // File path glob (for file ops)
  command?: string;     // Command pattern (for Bash)
  behavior: 'allow' | 'deny' | 'ask';
  reason?: string;
}

function matchRule(
  rule: PermissionRule,
  toolName: string,
  toolInput: unknown
): boolean {
  // 1. Tool name match
  if (rule.tool) {
    if (rule.tool === '*') {
      // Wildcard matches all
    } else if (rule.tool.includes('*')) {
      // Pattern match
      const regex = new RegExp(
        '^' + rule.tool.replace(/\*/g, '.*') + '$'
      );
      if (!regex.test(toolName)) return false;
    } else {
      // Exact match
      if (rule.tool !== toolName) return false;
    }
  }
  
  // 2. Path match (for file operations)
  if (rule.path && 'file_path' in toolInput) {
    const matched = matchGlob(rule.path, toolInput.file_path);
    if (!matched) return false;
  }
  
  // 3. Command match (for Bash)
  if (rule.command && 'command' in toolInput) {
    if (rule.command.includes('*')) {
      const regex = new RegExp(
        '^' + rule.command.replace(/\*/g, '.*')
      );
      if (!regex.test(toolInput.command)) return false;
    } else {
      if (!toolInput.command.startsWith(rule.command)) return false;
    }
  }
  
  return true;
}
```

---

## Hook Execution Engine

### Hook Lifecycle

```
Hook Trigger Point Reached
          │
          ▼
    ┌─────────────────┐
    │ Collect         │
    │ Registered      │
    │ Hooks           │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ For Each Hook:  │
    │                 │
    │  1. Start Timer │
    │     (5 sec)     │
    │                 │
    │  2. Execute     │
    │     Callback    │
    │                 │
    │  3. Collect     │
    │     Output      │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ Merge All       │
    │ Outputs         │
    │                 │
    │ Last decision   │
    │ wins            │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ Apply Combined  │
    │ Output          │
    │                 │
    │ • System msg    │
    │ • Decision      │
    │ • Modifications │
    └─────────────────┘
```

### Timeout Enforcement

```typescript
const HOOK_TIMEOUT = 5000; // 5 seconds (hardcoded)

async function executeHookWithTimeout(
  callback: HookCallback,
  input: HookInput
): Promise<HookOutput> {
  const controller = new AbortController();
  
  const timeoutPromise = new Promise<never>((_, reject) => {
    setTimeout(() => {
      controller.abort();
      reject(new Error('Hook timeout'));
    }, HOOK_TIMEOUT);
  });
  
  const executionPromise = callback(
    input,
    undefined,
    { signal: controller.signal }
  );
  
  try {
    return await Promise.race([executionPromise, timeoutPromise]);
  } catch (error) {
    if (error.message === 'Hook timeout') {
      console.warn(`Hook timed out after ${HOOK_TIMEOUT}ms`);
      // Fail open - continue execution
      return { decision: 'approve' };
    }
    throw error;
  }
}
```

---

## Session Management

### Session Storage Structure

```
~/.claude/sessions/
  ├── <session-id-1>/
  │   ├── transcript.json       # Complete conversation
  │   ├── checkpoints/
  │   │   ├── checkpoint-0.json   # Initial state
  │   │   ├── checkpoint-5.json   # After 5 turns
  │   │   ├── checkpoint-10.json  # After 10 turns
  │   │   └── ...
  │   └── file-snapshots/
  │       ├── <hash-1>.txt      # File content snapshot
  │       ├── <hash-2>.txt
  │       └── ...
  └── <session-id-2>/
      └── ...
```

### Checkpointing Strategy

```typescript
class SessionManager {
  private readonly CHECKPOINT_INTERVAL = 5; // Every 5 turns
  
  async checkpoint(sessionId: string, turn: number) {
    if (turn % this.CHECKPOINT_INTERVAL !== 0) return;
    
    const state = {
      turn,
      timestamp: Date.now(),
      conversation: this.getConversation(sessionId),
      fileSnapshots: this.getFileSnapshots(sessionId),
      permissionState: this.getPermissionState(sessionId)
    };
    
    const checkpointPath = 
      `~/.claude/sessions/${sessionId}/checkpoints/checkpoint-${turn}.json`;
    
    await fs.writeFile(checkpointPath, JSON.stringify(state, null, 2));
  }
  
  async resumeFromCheckpoint(
    sessionId: string,
    messageId?: string
  ): Promise<SessionState> {
    // Find appropriate checkpoint
    const checkpoints = await this.listCheckpoints(sessionId);
    
    if (messageId) {
      // Resume from specific message
      return await this.loadStateAtMessage(sessionId, messageId);
    } else {
      // Resume from latest
      const latest = checkpoints[checkpoints.length - 1];
      return await this.loadCheckpoint(sessionId, latest);
    }
  }
}
```

### File Snapshot Management

```typescript
// For Edit tool - track file content before edits
class FileSnapshotManager {
  private readonly MAX_SNAPSHOTS = 1000;
  private snapshots = new Map<string, string>(); // path -> hash
  
  async snapshot(filePath: string): Promise<string> {
    const content = await fs.readFile(filePath, 'utf-8');
    const hash = this.hashContent(content);
    
    const snapshotPath = 
      `~/.claude/sessions/${this.sessionId}/file-snapshots/${hash}.txt`;
    
    if (!await this.exists(snapshotPath)) {
      await fs.writeFile(snapshotPath, content);
    }
    
    this.snapshots.set(filePath, hash);
    
    // Cleanup old snapshots if exceeded limit
    if (this.snapshots.size > this.MAX_SNAPSHOTS) {
      await this.pruneOldestSnapshots();
    }
    
    return hash;
  }
  
  async restore(filePath: string, hash: string): Promise<string> {
    const snapshotPath = 
      `~/.claude/sessions/${this.sessionId}/file-snapshots/${hash}.txt`;
    
    return await fs.readFile(snapshotPath, 'utf-8');
  }
}
```

---

## State Management

### Read-Before-Write Enforcement

```typescript
class EditStateManager {
  private readFiles = new Set<string>(); // Session-scoped
  
  recordRead(filePath: string) {
    this.readFiles.add(this.normalizePath(filePath));
  }
  
  canEdit(filePath: string): boolean {
    return this.readFiles.has(this.normalizePath(filePath));
  }
  
  private normalizePath(path: string): string {
    return path.resolve(path);
  }
}

// Used in FileEdit and FileWrite tools
async function enforceReadBeforeWrite(
  toolName: 'FileEdit' | 'FileWrite',
  input: { file_path: string },
  state: EditStateManager
) {
  if (!state.canEdit(input.file_path)) {
    throw new Error(
      `Must read ${input.file_path} before editing. ` +
      `Use FileRead tool first.`
    );
  }
}
```

### Edit Uniqueness Validation

```typescript
async function validateEditUniqueness(
  input: { file_path: string; old_string: string; replace_all?: boolean }
): Promise<void> {
  if (input.replace_all) {
    return; // Skip validation if replace_all is true
  }
  
  const content = await fs.readFile(input.file_path, 'utf-8');
  const occurrences = (content.match(
    new RegExp(escapeRegex(input.old_string), 'g')
  ) || []).length;
  
  if (occurrences === 0) {
    throw new Error(`old_string not found in file`);
  }
  
  if (occurrences > 1) {
    throw new Error(
      `old_string appears ${occurrences} times. ` +
      `Must be unique or use replace_all: true`
    );
  }
}
```

---

## MCP Integration Architecture

### MCP Client Implementation

```typescript
class MCPClientManager {
  private servers = new Map<string, MCPServerConnection>();
  
  async connect(
    name: string,
    config: McpServerConfig
  ): Promise<void> {
    switch (config.type) {
      case 'stdio':
        return await this.connectStdio(name, config);
      case 'http':
        return await this.connectHTTP(name, config);
      case 'sse':
        return await this.connectSSE(name, config);
      case 'sdk':
        return await this.connectSDK(name, config);
    }
  }
  
  private async connectStdio(
    name: string,
    config: McpStdioServerConfig
  ): Promise<void> {
    // Spawn process
    const child = spawn(config.command, config.args || [], {
      env: { ...process.env, ...config.env },
      stdio: ['pipe', 'pipe', 'pipe']
    });
    
    // Setup JSON-RPC communication
    const rpc = new JSONRPCClient(child.stdin, child.stdout);
    
    // Initialize MCP protocol
    await rpc.request('initialize', {
      protocolVersion: '1.0',
      capabilities: {}
    });
    
    // Store connection
    this.servers.set(name, {
      type: 'stdio',
      process: child,
      rpc
    });
  }
  
  async invokeTool(
    serverName: string,
    toolName: string,
    input: unknown
  ): Promise<ToolResult> {
    const server = this.servers.get(serverName);
    if (!server) {
      throw new Error(`MCP server '${serverName}' not found`);
    }
    
    // Send tool invocation via MCP protocol
    const result = await server.rpc.request('tools/call', {
      name: toolName,
      arguments: input
    });
    
    return result;
  }
}
```

### Transport Performance

| Transport | Latency | Overhead | Use Case |
|-----------|---------|----------|----------|
| **SDK** | 0.1-1ms | None | In-process tools |
| **stdio** | 5-20ms | Process IPC | External CLIs |
| **HTTP** | 10-50ms | Network | REST APIs |
| **SSE** | 10-50ms | Network | Streaming APIs |

---

## Performance Characteristics

### Tool Execution Times (Internal Benchmarks)

```typescript
// Measured from SDK implementation
const TOOL_PERFORMANCE = {
  // File Operations
  'FileRead': {
    cached: '1-5ms',        // File already in cache
    uncached: '10-50ms',    // First read
    large: '100-500ms'      // Large files (>1MB)
  },
  'FileWrite': {
    small: '5-20ms',        // <100KB
    large: '50-200ms'       // >100KB
  },
  'FileEdit': {
    typical: '10-30ms',     // Read + validate + write
    complex: '50-100ms'     // Large files with complex edits
  },
  'Glob': {
    simple: '50-200ms',     // Few files
    complex: '200-500ms'    // Many files, complex pattern
  },
  'Grep': {
    fast: '100-500ms',      // Ripgrep is very fast
    slow: '500-2000ms'      // Large codebase
  },
  
  // Execution
  'Bash': {
    instant: '<100ms',      // echo, pwd, etc.
    typical: '100-5000ms',  // Most commands
    long: '5000-120000ms'   // Default 2min timeout
  },
  
  // Agent Operations
  'Task': {
    explore: '5000-15000ms',      // Haiku, isolated
    general: '30000-120000ms',    // Sonnet, forked
    security: '30000-60000ms'     // Sonnet, isolated
  }
};
```

### Memory Usage

```typescript
const MEMORY_CHARACTERISTICS = {
  baselineSDK: '50-100MB',        // SDK loaded
  perSession: '5-20MB',           // Conversation state
  fileCache: '1-10MB per file',   // Cached file contents
  streamingBuffer: '1-5MB',       // Streaming chunks
  mcpConnections: '2-10MB each'   // Per MCP server
};
```

---

## Internal Constants & Limits

### Complete Constants List

```typescript
// From CLI bundle analysis
const INTERNAL_LIMITS = {
  // Tool Limits
  READ_DEFAULT_LINES: 2000,
  READ_CHAR_TRUNCATION: 2000,      // Per line
  BASH_OUTPUT_MAX: 30000,          // Characters
  BASH_TIMEOUT_DEFAULT: 120000,    // 2 minutes
  BASH_TIMEOUT_MAX: 600000,        // 10 minutes
  PDF_MAX_SIZE: 20971520,          // 20MB (tj1)

  // Hook Limits
  HOOK_TIMEOUT: 15000,             // 15 seconds
  
  // Thinking Limits
  ULTRATHINK_MAX: 31999,           // Max thinking tokens
  
  // Session Limits
  MAX_FILE_SNAPSHOTS: 1000,
  CHECKPOINT_INTERVAL: 5,          // Turns
  
  // MCP Limits
  MCP_TIMEOUT: 30000,              // 30 seconds
  
  // Agent Colors
  AGENT_COLORS: [
    'red', 'blue', 'green', 'yellow',
    'purple', 'orange', 'pink', 'cyan'
  ],
  
  // Permission Levels
  PERMISSION_LAYERS: [
    'session', 'local', 'project',
    'user', 'policy', 'mode'
  ]
};
```

---

## Implementation Patterns

### Pattern 1: Async Generator Architecture

```typescript
/**
 * Core pattern for streaming
 * All SDK responses use async generators
 */
async function* streamingPattern(): AsyncGenerator<Message> {
  // Initialize
  yield { type: 'init', data: initData };
  
  // Stream content
  while (hasMore) {
    const chunk = await fetchNextChunk();
    yield { type: 'chunk', data: chunk };
  }
  
  // Finalize
  yield { type: 'complete', data: finalData };
}

// Consumption
for await (const message of streamingPattern()) {
  handleMessage(message);
}
```

### Pattern 2: Fail-Open Safety

```typescript
/**
 * SDK fails open on non-critical errors
 * Continues execution rather than blocking
 */
try {
  await executeHook(hookCallback);
} catch (error) {
  console.warn('Hook error:', error);
  // Continue execution (fail open)
}

try {
  await connectMCPServer(config);
} catch (error) {
  console.warn('MCP connection failed:', error);
  // Continue without MCP server (fail open)
}
```

### Pattern 3: Layer Cake Configuration

```typescript
/**
 * Multi-layer configuration resolution
 * Higher layers override lower layers
 */
function resolveConfig<T>(key: string): T {
  return (
    getSessionConfig(key) ||
    getLocalConfig(key) ||
    getProjectConfig(key) ||
    getUserConfig(key) ||
    getPolicyConfig(key) ||
    getDefaultConfig(key)
  );
}
```

---

## Summary

### Architecture Principles

1. **Streaming First**: Everything uses async generators
2. **Fail Open**: Non-critical errors don't block execution
3. **Layer Cake**: Configuration resolves through multiple layers
4. **Hook Everything**: 9 lifecycle hooks for customization
5. **Type Safe**: Full TypeScript throughout
6. **Session Aware**: State persists across resumptions

### Key Internal Systems

| System | Purpose | Key Feature |
|--------|---------|-------------|
| **Query Manager** | Orchestration | Conversation lifecycle |
| **Tool Executor** | Tool execution | 8-stage pipeline |
| **Permission Resolver** | Security | 6-level cascade |
| **Hook Engine** | Customization | Sequential execution |
| **Session Manager** | Persistence | Checkpoints + snapshots |
| **MCP Client** | Integration | 4 transport types |

### Critical Constraints

- ⏱️ Hook timeout: 5 seconds (strict)
- 📏 Bash output: 30K chars (truncated)
- 📄 Read line: 2K chars (truncated)
- 🔄 Checkpoint: Every 5 turns
- 🎨 Agent colors: 8 maximum
- 🛡️ Permission: 6-level resolution
