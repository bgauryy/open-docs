# OpenCode - Tool System Architecture

> **Complete guide to OpenCode's tool system - how AI agents take actions through tools**

---

## Table of Contents

- [Overview](#overview)
- [Tool Interface](#tool-interface)
- [Tool Registry](#tool-registry)
- [Tool Execution Flow](#tool-execution-flow)
- [Permission System](#permission-system)
- [Custom Tools](#custom-tools)
- [Plugin Tools](#plugin-tools)

---

## Overview

Tools are how OpenCode's AI agents interact with the codebase and system. Each tool represents a specific capability like reading files, running commands, or searching content.

**Core Concepts**:
- **Tool Definition** - Declares what a tool does and its parameters
- **Tool Registry** - Manages available tools
- **Tool Execution** - Runs tools with parameters
- **Permissions** - Controls what tools can do
- **Extensibility** - Custom and plugin tools

---

## Tool Interface

**File**: `tool/tool.ts`

### Tool.Info Structure

```typescript
export namespace Tool {
  export interface Info<
    Parameters extends z.ZodType = z.ZodType,
    M extends Metadata = Metadata
  > {
    id: string
    
    init: () => Promise<{
      description: string            // Prompt description for AI
      parameters: Parameters          // Zod schema for validation
      execute(
        args: z.infer<Parameters>,
        ctx: Context
      ): Promise<{
        title: string                // Short summary for UI
        metadata: M                  // Structured metadata
        output: string               // Text result for AI
        attachments?: MessageV2.FilePart[]  // File attachments
      }>
    }>
  }
}
```

### Tool Context

Every tool execution receives a context object:

```typescript
export type Context<M extends Metadata = Metadata> = {
  sessionID: string          // Current session
  messageID: string          // Message being processed
  agent: string              // Agent name
  abort: AbortSignal         // For cancellation
  callID?: string            // Unique call identifier
  extra?: Record<string, any>  // Additional data
  
  // Update metadata during execution
  metadata(input: {
    title?: string
    metadata?: M
  }): void
}
```

### Defining a Tool

Use `Tool.define()` helper:

```typescript
export const MyTool = Tool.define("mytool", {
  description: "Description shown to AI",
  
  parameters: z.object({
    input: z.string().describe("Input parameter"),
    optional: z.number().optional(),
  }),
  
  async execute(args, ctx) {
    // Tool logic here
    const result = await doWork(args.input)
    
    return {
      title: "Work completed",
      output: result,
      metadata: { /* structured data */ },
    }
  }
})
```

### Async Tool Definition

Tools can have async initialization:

```typescript
export const DynamicTool = Tool.define("dynamic", async () => {
  // Load config or resources
  const config = await loadConfig()
  
  return {
    description: `Tool with ${config.feature}`,
    parameters: z.object({...}),
    execute: async (args, ctx) => {...}
  }
})
```

---

## Tool Registry

**File**: `tool/registry.ts`

### Built-in Tools

```typescript
export namespace ToolRegistry {
  async function all(): Promise<Tool.Info[]> {
    const custom = await state().then((x) => x.custom)
    
    return [
      InvalidTool,         // Invalid tool placeholder
      BashTool,           // Shell command execution
      EditTool,           // File editing with diffs
      WebFetchTool,       // Web content fetching
      GlobTool,           // File pattern matching
      GrepTool,           // Content searching
      ListTool,           // Directory listing
      PatchTool,          // Patch application
      ReadTool,           // File reading
      WriteTool,          // File writing
      TodoWriteTool,      // TODO writing
      TodoReadTool,       // TODO reading
      TaskTool,           // Task management
      ...custom,          // Custom tools
    ]
  }
}
```

### Registration

Tools are discovered and registered from multiple sources:

#### 1. **Built-in Tools**

Hardcoded in `registry.ts`:
```typescript
import { ReadTool } from "./read"
import { WriteTool } from "./write"
// ...

return [ReadTool, WriteTool, ...]
```

#### 2. **Custom Tools from Config Directories**

```typescript
const glob = new Bun.Glob("tool/*.{js,ts}")

for (const dir of await Config.directories()) {
  for await (const match of glob.scan({ cwd: dir })) {
    const mod = await import(match)
    // Register exported tools
  }
}
```

Locations:
- `.opencode/tool/*.ts`
- `~/.opencode/tool/*.ts`

#### 3. **Plugin Tools**

```typescript
const plugins = await Plugin.list()

for (const plugin of plugins) {
  for (const [id, def] of Object.entries(plugin.tool ?? {})) {
    custom.push(fromPlugin(id, def))
  }
}
```

### Tool Filtering

Tools can be enabled/disabled based on permissions:

```typescript
export async function enabled(
  providerID: string,
  modelID: string,
  agent: Agent.Info
): Promise<Record<string, boolean>> {
  const result: Record<string, boolean> = {}
  
  // Disable patch by default
  result["patch"] = false
  
  // Disable edit tools if agent denies edit
  if (agent.permission.edit === "deny") {
    result["edit"] = false
    result["patch"] = false
    result["write"] = false
  }
  
  // Disable bash if agent denies all commands
  if (agent.permission.bash["*"] === "deny" && 
      Object.keys(agent.permission.bash).length === 1) {
    result["bash"] = false
  }
  
  // Disable webfetch if agent denies
  if (agent.permission.webfetch === "deny") {
    result["webfetch"] = false
  }
  
  return result
}
```

---

## Tool Execution Flow

### Complete Flow

```
AI requests tool
    │
    ▼
┌──────────────────┐
│ Parse tool call  │
│ from AI response │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Validate args    │
│ (Zod schema)     │
└────────┬─────────┘
         │ Valid
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
         ├─Success─┐
         │         ▼
         │    ┌──────────────┐
         │    │ Update part  │
         │    │ (completed)  │
         │    └──────────────┘
         │
         └─Error───┐
                   ▼
              ┌──────────────┐
              │ Update part  │
              │ (error)      │
              └──────────────┘
```

### Execution Context

```typescript
// Create context
const ctx: Tool.Context = {
  sessionID: msg.sessionID,
  messageID: msg.id,
  agent: agentName,
  abort: lock.signal,
  callID: toolCall.id,
  extra: {
    providerID,
    modelID,
    bypassCwdCheck: false,
  },
  metadata: (input) => {
    // Update tool part metadata
    part.state.title = input.title ?? part.state.title
    part.state.metadata = {
      ...part.state.metadata,
      ...input.metadata,
    }
  }
}

// Execute tool
const result = await tool.execute(args, ctx)
```

### Tool Part States

```typescript
type ToolPartState = 
  | { status: "active", title?: string }           // Executing
  | { status: "completed", title: string, output: string, metadata: any }  // Done
  | { status: "error", error: ErrorInfo }          // Failed
```

### Example Execution

```typescript
// AI calls tool
{
  type: "tool-call",
  toolCallId: "call_abc123",
  toolName: "read",
  args: {
    filePath: "src/auth.ts",
    offset: 0,
    limit: 100
  }
}

// Create tool part
const part = await Session.updatePart({
  type: "tool",
  id: "part_xyz789",
  sessionID,
  messageID,
  tool: "read",
  state: {
    status: "active",
    input: { filePath: "src/auth.ts", offset: 0, limit: 100 }
  }
})

// Execute
const result = await ReadTool.init().then(def =>
  def.execute(
    { filePath: "src/auth.ts", offset: 0, limit: 100 },
    ctx
  )
)

// Update with result
await Session.updatePart({
  ...part,
  state: {
    status: "completed",
    title: result.title,
    output: result.output,
    metadata: result.metadata,
    input: part.state.input,
    time: {
      start: part.state.time.start,
      completed: Date.now()
    }
  }
})
```

---

## Permission System

**File**: `permission/index.ts`

### Permission Types

```typescript
type PermissionLevel = "allow" | "deny" | "ask"

interface Permissions {
  edit: PermissionLevel                 // File editing
  bash: Record<string, PermissionLevel> // Shell commands
  webfetch: PermissionLevel            // Web access
}
```

### Checking Permissions

```typescript
// Before executing tool
if (requiresPermission(tool)) {
  const approved = await Permission.check({
    sessionID,
    type: getPermissionType(tool),
    details: getPermissionDetails(args)
  })
  
  if (!approved) {
    throw new Error("Permission denied")
  }
}
```

### Agent Permissions

Agents can define default permissions:

```markdown
# Agent: cautious

## Permissions
- edit: ask
- bash: deny
- webfetch: ask
```

Translates to:
```typescript
{
  permission: {
    edit: "ask",
    bash: { "*": "deny" },
    webfetch: "ask"
  }
}
```

### Permission Workflow

```
Tool requires permission
    │
    ▼
┌──────────────────┐
│ Check agent      │
│ permission level │
└────────┬─────────┘
         │
         ├─"allow"──▶ Execute immediately
         │
         ├─"deny"───▶ Reject with error
         │
         └─"ask"────┐
                    ▼
           ┌──────────────────┐
           │ Create permission│
           │ request part     │
           └────────┬─────────┘
                    │
                    ▼
           ┌──────────────────┐
           │ Wait for user    │
           │ approval/denial  │
           └────────┬─────────┘
                    │
                    ├─Approved──▶ Execute tool
                    │
                    └─Denied────▶ Return error
```

---

## Custom Tools

### Creating Custom Tools

**Location**: `.opencode/tool/mytool.ts`

```typescript
import { Tool } from "opencode"
import z from "zod"

export const MyCustomTool = Tool.define("mycustom", {
  description: "My custom tool that does X",
  
  parameters: z.object({
    input: z.string().describe("Input data"),
    option: z.enum(["a", "b", "c"]).optional()
  }),
  
  async execute(args, ctx) {
    // Access session context
    console.log("Session:", ctx.sessionID)
    
    // Check for cancellation
    ctx.abort.throwIfAborted()
    
    // Do work
    const result = processInput(args.input, args.option)
    
    // Update metadata during execution
    ctx.metadata({
      title: "Processing...",
      metadata: { progress: 0.5 }
    })
    
    // Return result
    return {
      title: "Completed successfully",
      output: result,
      metadata: {
        inputLength: args.input.length,
        processingTime: 123
      }
    }
  }
})

// Default export also supported
export default MyCustomTool
```

### Tool with File Attachments

```typescript
export const ScreenshotTool = Tool.define("screenshot", {
  description: "Take a screenshot",
  parameters: z.object({
    url: z.string()
  }),
  
  async execute(args, ctx) {
    const imageData = await takeScreenshot(args.url)
    
    return {
      title: "Screenshot captured",
      output: `Screenshot of ${args.url}`,
      metadata: { url: args.url },
      attachments: [{
        id: Identifier.ascending("part"),
        sessionID: ctx.sessionID,
        messageID: ctx.messageID,
        type: "file",
        mime: "image/png",
        url: `data:image/png;base64,${imageData}`,
      }]
    }
  }
})
```

### Async Initialization

```typescript
export const DatabaseTool = Tool.define("database", async () => {
  // Connect to database
  const db = await connectDatabase()
  
  return {
    description: "Query database",
    parameters: z.object({
      query: z.string()
    }),
    async execute(args, ctx) {
      const results = await db.query(args.query)
      return {
        title: `${results.length} results`,
        output: JSON.stringify(results, null, 2),
        metadata: { count: results.length }
      }
    }
  }
})
```

---

## Plugin Tools

**File**: `@opencode-ai/plugin`

### Plugin Tool Definition

```typescript
// my-plugin.ts
export const tool = {
  mytool: {
    description: "Tool from plugin",
    args: {
      input: { type: "string", description: "Input" }
    },
    execute: async (args, ctx) => {
      return `Processed: ${args.input}`
    }
  }
}
```

### Loading Plugin Tools

```typescript
// Automatic loading
const plugins = await Plugin.list()

for (const plugin of plugins) {
  for (const [id, def] of Object.entries(plugin.tool ?? {})) {
    // Convert to Tool.Info format
    const toolInfo = {
      id,
      init: async () => ({
        parameters: z.object(def.args),
        description: def.description,
        execute: async (args, ctx) => {
          const result = await def.execute(args, ctx)
          return {
            title: "",
            output: result,
            metadata: {},
          }
        }
      })
    }
    
    await ToolRegistry.register(toolInfo)
  }
}
```

---

## Best Practices

### 1. Descriptive Tool Names

**Good**:
```typescript
Tool.define("read-file", ...)
Tool.define("run-tests", ...)
Tool.define("search-codebase", ...)
```

**Bad**:
```typescript
Tool.define("tool1", ...)
Tool.define("do-it", ...)
Tool.define("x", ...)
```

### 2. Clear Descriptions

**Good**:
```
Read and display contents of one or multiple files. 
Returns file content with line numbers. 
Supports offset and limit for large files.
```

**Bad**:
```
Reads files.
```

### 3. Validate Parameters

```typescript
parameters: z.object({
  path: z.string()
    .describe("File path relative to working directory")
    .refine(
      (p) => !p.startsWith("/"),
      "Path must be relative"
    ),
  count: z.number()
    .min(1)
    .max(1000)
    .describe("Number of lines to read")
})
```

### 4. Handle Cancellation

```typescript
async execute(args, ctx) {
  for (const item of items) {
    // Check abort signal
    ctx.abort.throwIfAborted()
    
    await processItem(item)
  }
}
```

### 5. Provide Structured Metadata

```typescript
return {
  title: "Found 5 matches",
  output: matchDetails,
  metadata: {
    count: 5,
    files: ["a.ts", "b.ts"],
    duration: 123
  }
}
```

### 6. Update Progress

```typescript
async execute(args, ctx) {
  ctx.metadata({ title: "Starting..." })
  
  // Do work
  await step1()
  ctx.metadata({ title: "Step 1 complete" })
  
  await step2()
  ctx.metadata({ title: "Step 2 complete" })
  
  return { ... }
}
```

---

## Summary

OpenCode's tool system provides:

- **Flexible interface** for defining tools
- **Type-safe parameters** with Zod validation
- **Registry system** for discovery and management
- **Permission controls** for security
- **Extensibility** through custom and plugin tools
- **Rich context** for tool execution
- **Metadata support** for structured results

Tools are the bridge between AI intent and system actions, enabling powerful automated workflows while maintaining safety through permissions.

---

## Next Steps

- **[07-tool-implementations.md](./07-tool-implementations.md)** - Individual tool details
- **[14-security-permissions.md](./14-security-permissions.md)** - Permission system
- **[23-sdks-integrations.md](./23-sdks-integrations.md)** - Plugin development

