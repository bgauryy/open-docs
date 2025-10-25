# OpenCode - Prompt Processing

> **Deep dive into OpenCode's prompt construction, context assembly, and AI interaction system**

---

## Table of Contents

- [Overview](#overview)
- [Prompt Architecture](#prompt-architecture)
- [Context Assembly](#context-assembly)
- [System Prompts](#system-prompts)
- [Tool Integration](#tool-integration)
- [Message Formatting](#message-formatting)
- [Streaming & Response](#streaming--response)
- [Prompt Templates](#prompt-templates)

---

## Overview

The prompt system is OpenCode's most complex module, responsible for assembling all context needed for effective AI interactions.

**Key File**: `session/prompt.ts` (1766 lines, 54KB)

### Responsibilities

1. **Context Assembly** - Gather files, LSP data, history, environment info
2. **System Prompts** - Inject provider-specific and custom instructions
3. **Tool Descriptions** - Format available tools for AI
4. **Message Formatting** - Convert to AI SDK format
5. **Streaming Management** - Handle real-time AI responses
6. **Error Handling** - Retry logic and fallbacks
7. **Token Management** - Stay within model limits

---

## Prompt Architecture

### Core Components

```typescript
export namespace SessionPrompt {
  export const OUTPUT_TOKEN_MAX = 32_000
  const MAX_RETRIES = 10
  
  // Main entry point
  export async function prompt(input: PromptInput): Promise<MessageV2.WithParts>
  
  // Command execution
  export async function command(input: CommandInput): Promise<void>
  
  // Events
  export const Event = {
    Idle: Bus.event("session.idle", ...)
  }
}
```

### Prompt Input Schema

```typescript
export const PromptInput = z.object({
  sessionID: Identifier.schema("session"),     // Target session
  messageID: Identifier.schema("message").optional(),
  
  model: z.object({
    providerID: z.string(),                    // e.g., "anthropic"
    modelID: z.string(),                       // e.g., "claude-3-5-sonnet"
  }).optional(),
  
  agent: z.string().optional(),                // Agent name
  system: z.string().optional(),               // Custom system prompt
  tools: z.record(z.string(), z.boolean()).optional(), // Tool filter
  
  parts: z.array(z.discriminatedUnion("type", [
    MessageV2.TextPart,       // Text content
    MessageV2.FilePart,       // File attachments
    MessageV2.AgentPart,      // Agent switch
  ])),
})
```

### Prompt Flow

```
User Input
    │
    ▼
┌───────────────────┐
│ SessionPrompt     │
│ .prompt()         │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Check if session  │
│ is busy (locked)  │
└────────┬──────────┘
         │ Free
         ▼
┌───────────────────┐
│ Create user       │
│ message & parts   │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Queue or process  │
│ immediately       │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Acquire session   │
│ lock              │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Build context     │
│ - System prompts  │
│ - Files           │
│ - LSP data        │
│ - History         │
│ - Tools           │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Check compaction  │
│ (if overflow)     │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Stream AI request │
│ (with retries)    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Process response  │
│ - Text chunks     │
│ - Tool calls      │
│ - Steps           │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Execute tools     │
│ (if requested)    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Handle result     │
│ - Success         │
│ - Error/retry     │
│ - Update session  │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ Release lock      │
│ Process queue     │
└────────┬──────────┘
         │
         ▼
    Return result
```

---

## Context Assembly

### User Message Creation

```typescript
async function createUserMessage(input: PromptInput) {
  const msg = await Session.updateMessage({
    id: input.messageID ?? Identifier.ascending("message"),
    sessionID: input.sessionID,
    role: "user",
    parentID: findLastAssistantMessage(),
    system: [],
    mode: "build",  // or "chat"
    path: {
      cwd: Instance.directory,
      root: Instance.worktree,
    },
    time: { created: Date.now() },
  })
  
  // Create parts
  for (const partInput of input.parts) {
    const part = await Session.updatePart({
      ...partInput,
      id: partInput.id ?? Identifier.ascending("part"),
      messageID: msg.id,
      sessionID: input.sessionID,
    })
  }
  
  return { info: msg, parts: await Session.getParts(msg.id) }
}
```

### Context Sources

#### 1. **System Prompts**

Multiple layers of system prompts:

```typescript
const systemPrompts = [
  ...SystemPrompt.header(providerID),          // Provider-specific header
  ...SystemPrompt.provider(modelID),           // Model-specific prompt
  ...(await SystemPrompt.environment()),       // Environment info
  ...(await SystemPrompt.custom()),            // AGENTS.md files
  agent.system,                                 // Agent-specific prompt
  input.system,                                 // User override
]
```

#### 2. **Environment Information**

```typescript
export async function environment() {
  return [{
    `Here is some useful information about the environment:`,
    `<env>`,
    ` Working directory: ${Instance.directory}`,
    ` Is directory a git repo: ${project.vcs === "git" ? "yes" : "no"}`,
    ` Platform: ${process.platform}`,
    ` Today's date: ${new Date().toDateString()}`,
    `</env>`,
    `<project>`,
    ` ${project.vcs === "git" ? await Ripgrep.tree({...}) : ""}`,
    `</project>`,
  ].join("\n")}
}
```

#### 3. **Custom Instructions (AGENTS.md)**

**Discovery Order**:
1. Local files (searched upwards from working directory):
   - `AGENTS.md`
   - `CLAUDE.md`
   - `CONTEXT.md`

2. Global files (user's home directory):
   - `~/.opencode/AGENTS.md`
   - `~/.claude/CLAUDE.md`

3. Config instructions:
   - Paths from `config.instructions` array
   - Supports glob patterns
   - Supports `~/` expansion

```typescript
export async function custom() {
  const paths = new Set<string>()
  
  // Find local rule files (stop at first match per file type)
  for (const localRuleFile of LOCAL_RULE_FILES) {
    const matches = await Filesystem.findUp(
      localRuleFile,
      Instance.directory,
      Instance.worktree
    )
    if (matches.length > 0) {
      matches.forEach((path) => paths.add(path))
      break
    }
  }
  
  // Check global rule files
  for (const globalRuleFile of GLOBAL_RULE_FILES) {
    if (await Bun.file(globalRuleFile).exists()) {
      paths.add(globalRuleFile)
      break
    }
  }
  
  // Process config instructions (supports globs)
  if (config.instructions) {
    for (let instruction of config.instructions) {
      // Handle ~/ paths
      if (instruction.startsWith("~/")) {
        instruction = path.join(os.homedir(), instruction.slice(2))
      }
      
      // Glob matching
      const matches = await Filesystem.globUp(
        instruction,
        Instance.directory,
        Instance.worktree
      )
      matches.forEach((path) => paths.add(path))
    }
  }
  
  // Read all found files
  return Promise.all(
    Array.from(paths).map((p) => Bun.file(p).text())
  ).then((result) => result.filter(Boolean))
}
```

#### 4. **Conversation History**

```typescript
// Get filtered message history
const history = await Session.messages(sessionID)
  .then(MessageV2.filterCompacted)  // Skip compacted content
  .then((msgs) => msgs.map(MessageV2.toModelMessage))
```

**Filtering**:
- Skip parts marked as compacted
- Include summaries as context
- Preserve user messages
- Keep tool call results

#### 5. **Tool Descriptions**

```typescript
// Get available tools
const registry = ToolRegistry.create(Instance)

// Convert to AI SDK format
const tools = Object.entries(registry).map(([name, tool]) => ({
  description: tool.description,  // From .txt file
  parameters: tool.schema,         // Zod schema
  execute: tool.execute,
}))
```

#### 6. **MCP Resources**

```typescript
// Include MCP server tools/resources
const mcpTools = await MCP.getTools()
const mcpResources = await MCP.getResources()
```

---

## System Prompts

**File**: `session/system.ts`

### Provider-Specific Prompts

Different providers need different prompting styles:

```typescript
export function provider(modelID: string) {
  if (modelID.includes("gpt-5"))
    return [PROMPT_CODEX]
  if (modelID.includes("gpt-") || modelID.includes("o1") || modelID.includes("o3"))
    return [PROMPT_BEAST]
  if (modelID.includes("gemini-"))
    return [PROMPT_GEMINI]
  if (modelID.includes("claude"))
    return [PROMPT_ANTHROPIC]
  
  // Default for other models
  return [PROMPT_ANTHROPIC_WITHOUT_TODO]
}
```

### Header/Spoofing Prompts

Some providers benefit from identity spoofing:

```typescript
export function header(providerID: string) {
  if (providerID.includes("anthropic"))
    return [PROMPT_ANTHROPIC_SPOOF.trim()]
  return []
}
```

**PROMPT_ANTHROPIC_SPOOF** Example:
```
You are Claude, a large language model trained by Anthropic.
Knowledge cutoff: 2024-04
Current date: 2025-01-15
```

### Specialized Prompts

#### Summarization

```typescript
export function summarize(providerID: string) {
  switch (providerID) {
    case "anthropic":
      return [PROMPT_ANTHROPIC_SPOOF.trim(), PROMPT_SUMMARIZE]
    default:
      return [PROMPT_SUMMARIZE]
  }
}
```

#### Title Generation

```typescript
export function title(providerID: string) {
  switch (providerID) {
    case "anthropic":
      return [PROMPT_ANTHROPIC_SPOOF.trim(), PROMPT_TITLE]
    default:
      return [PROMPT_TITLE]
  }
}
```

---

## Tool Integration

### Tool Registry

```typescript
const registry = ToolRegistry.create(Instance)

// Available tools (16 built-in + MCP tools)
{
  read: ReadTool,
  write: WriteTool,
  edit: EditTool,
  multiedit: MultiEditTool,
  bash: BashTool,
  grep: GrepTool,
  glob: GlobTool,
  ls: LsTool,
  "lsp-diagnostics": LSPDiagnosticsTool,
  "lsp-hover": LSPHoverTool,
  patch: PatchTool,
  task: TaskTool,
  todo: TodoTool,
  webfetch: WebFetchTool,
  // + MCP tools dynamically added
}
```

### Tool Filtering

```typescript
// Allow/disallow specific tools
const filteredTools = input.tools
  ? Object.entries(registry).filter(([name]) => input.tools[name] !== false)
  : Object.entries(registry)
```

### Tool Execution Flow

```
AI requests tool
    │
    ▼
┌──────────────────┐
│ Parse tool call  │
│ - name           │
│ - arguments      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Check permission │
│ (if required)    │
└────────┬─────────┘
         │ Approved
         ▼
┌──────────────────┐
│ Create tool part │
│ (status: active) │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Execute tool     │
│ with timeout     │
└────────┬─────────┘
         │
         ├─Success─▶ Update part (status: completed)
         │
         └─Error───▶ Update part (status: error)
```

---

## Message Formatting

### Converting to AI SDK Format

```typescript
function toModelMessage(msg: MessageV2.WithParts): ModelMessage {
  if (msg.info.role === "user") {
    return {
      role: "user",
      content: msg.parts.map(part => {
        if (part.type === "text") {
          return { type: "text", text: part.text }
        }
        if (part.type === "file") {
          return {
            type: "file",
            data: part.data,
            mimeType: part.mime,
          }
        }
      })
    }
  }
  
  if (msg.info.role === "assistant") {
    return {
      role: "assistant",
      content: msg.parts.map(part => {
        if (part.type === "text") {
          return { type: "text", text: part.text }
        }
        if (part.type === "tool") {
          return {
            type: "tool-call",
            toolCallId: part.id,
            toolName: part.tool,
            args: part.state.input,
          }
        }
      })
    }
  }
}
```

### Complete Prompt Structure

```typescript
{
  // System messages (combined into single system prompt)
  system: [
    ...providerHeader,
    ...providerPrompt,
    ...environmentInfo,
    ...customInstructions,
    agentSystemPrompt,
  ].join("\n\n"),
  
  // Conversation messages
  messages: [
    { role: "user", content: "First message" },
    { role: "assistant", content: "Response..." },
    { role: "user", content: "Second message" },
    // ...
  ],
  
  // Available tools
  tools: {
    read: { description: "...", parameters: {...}, execute: fn },
    write: { description: "...", parameters: {...}, execute: fn },
    // ...
  },
  
  // Generation settings
  maxTokens: OUTPUT_TOKEN_MAX,
  temperature: 0.7,
  // ...
}
```

---

## Streaming & Response

### Streaming Setup

```typescript
const stream = streamText({
  model: provider.language,
  maxRetries: 0,  // Manual retry logic
  abortSignal: lock.signal,
  messages: [...],
  tools: {...},
  onError(error) {
    log.error("stream error", { error })
  },
})
```

### Processing Stream Events

```typescript
for await (const value of stream.fullStream) {
  switch (value.type) {
    case "text-delta":
      // Append text chunk
      part.text += value.text
      await Session.updatePart({ part, delta: value.text })
      break
      
    case "tool-call":
      // Execute tool
      const result = await executeTool(value)
      break
      
    case "step-start":
      // New reasoning step
      await Session.updatePart({
        type: "step-start",
        ...
      })
      break
      
    case "step-finish":
      // Step completed
      await Session.updatePart({
        type: "step-finish",
        ...
      })
      break
      
    case "finish-step":
      // Update token usage/cost
      const usage = Session.getUsage({
        model: modelInfo,
        usage: value.usage,
        metadata: value.providerMetadata,
      })
      msg.cost += usage.cost
      msg.tokens = usage.tokens
      await Session.updateMessage(msg)
      break
      
    case "error":
      throw value.error
  }
}
```

### Retry Logic

```typescript
const MAX_RETRIES = 10

let attempt = 0
while (attempt < MAX_RETRIES) {
  try {
    const stream = streamText({...})
    await processStream(stream)
    break  // Success
  } catch (error) {
    if (!isRetryable(error) || attempt >= MAX_RETRIES - 1) {
      throw error
    }
    
    // Exponential backoff
    const delayMs = SessionRetry.getRetryDelayInMs(error, attempt)
    await SessionRetry.sleep(delayMs, signal)
    
    // Create retry part
    await Session.updatePart({
      type: "retry",
      attempt: attempt + 1,
      error: MessageV2.fromError(error),
    })
    
    attempt++
  }
}
```

---

## Prompt Templates

### Built-in Templates

OpenCode includes 16 tool prompt templates (`.txt` files):

| File | Size | Purpose |
|------|------|---------|
| `bash.txt` | 8.8KB | Shell command execution |
| `todowrite.txt` | 8.8KB | TODO management |
| `task.txt` | 3.6KB | Task tracking |
| `multiedit.txt` | 2.4KB | Multi-file editing |
| `edit.txt` | 1.3KB | Single file editing |
| `read.txt` | 1.1KB | File reading |
| `todoread.txt` | 977B | TODO reading |
| `webfetch.txt` | 848B | Web content fetching |
| `grep.txt` | 672B | Content searching |
| `write.txt` | 623B | File writing |
| `glob.txt` | 545B | File pattern matching |
| `websearch.txt` | 485B | Web searching |
| `ls.txt` | 314B | Directory listing |
| `lsp-diagnostics.txt` | 11B | LSP diagnostics |
| `lsp-hover.txt` | 11B | LSP hover info |
| `patch.txt` | 11B | Patch application |

### Template Loading

```typescript
// Import at compile time
import READ_PROMPT from "./tool/read.txt"
import WRITE_PROMPT from "./tool/write.txt"
// ...

// Use in tool definition
export function ReadTool(instance: Instance) {
  return tool({
    description: READ_PROMPT,  // Loaded from .txt
    parameters: z.object({...}),
    execute: async (args) => {...},
  })
}
```

### Template Format

Tool prompts are plain text descriptions:

**Example** (`read.txt`):
```
Read and display contents of files. Can read one or multiple files simultaneously.

Usage:
- Single file: path="src/auth.ts"
- Multiple files: path=["src/auth.ts", "src/user.ts"]

Returns the file content or list of contents.
```

---

## Queue Management

### Concurrent Request Handling

```typescript
// State per instance
const state = Instance.state(() => {
  const queued = new Map<
    string,
    { messageID: string, callback: (msg: MessageV2.WithParts) => void }[]
  >()
  return { queued }
})

function isBusy(sessionID: string): boolean {
  return SessionLock.isLocked(sessionID)
}

// Queue if busy
if (isBusy(input.sessionID)) {
  return new Promise((resolve) => {
    const queue = state().queued.get(input.sessionID) ?? []
    queue.push({
      messageID: userMsg.info.id,
      callback: resolve,
    })
    state().queued.set(input.sessionID, queue)
  })
}

// Process queue after completion
function processQueue(sessionID: string) {
  const queue = state().queued.get(sessionID)
  if (!queue || queue.length === 0) return
  
  const next = queue.shift()
  if (next) {
    prompt({ sessionID, messageID: next.messageID })
      .then(next.callback)
  }
}
```

---

## Best Practices

### 1. Always Check Compaction

```typescript
// Before streaming, check if compaction needed
const overflow = SessionCompaction.isOverflow({
  tokens: lastMessage.tokens,
  model: modelInfo,
})

if (overflow) {
  await SessionCompaction.run({
    sessionID,
    providerID,
    modelID,
    signal: lock.signal,
  })
}
```

### 2. Handle Abort Signals

```typescript
// Respect abort signals throughout
for await (const chunk of stream) {
  signal.throwIfAborted()
  // Process chunk...
}
```

### 3. Update Progress Incrementally

```typescript
// Stream text deltas
await Session.updatePart({
  part: textPart,
  delta: newChunk,  // Incremental update
})

// Emit events for UI updates
Bus.publish(MessageV2.Event.PartUpdated, {
  part: textPart,
  delta: newChunk,
})
```

### 4. Preserve Context

```typescript
// Don't lose important context
const systemPrompts = [
  ...providerSpecific,
  ...environment,        // ← Critical context
  ...customInstructions, // ← User preferences
  ...agentPrompt,       // ← Behavior guidelines
]
```

---

## Summary

OpenCode's prompt processing system:

- **Assembles rich context** from multiple sources
- **Supports custom instructions** via AGENTS.md
- **Handles streaming** with real-time updates
- **Manages tools** with permission controls
- **Implements retries** with exponential backoff
- **Queues requests** for busy sessions
- **Respects limits** with automatic compaction

The 1766-line `prompt.ts` file orchestrates all these concerns into a cohesive system that enables effective AI-assisted coding.

---

## Next Steps

- **[05-system-prompts.md](./05-system-prompts.md)** - Custom prompts and AGENTS.md
- **[06-tool-system.md](./06-tool-system.md)** - Tool architecture
- **[07-tool-implementations.md](./07-tool-implementations.md)** - Individual tools

