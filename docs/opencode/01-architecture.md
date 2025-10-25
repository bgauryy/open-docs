# OpenCode - System Architecture

> **Deep dive into OpenCode's client/server architecture, component design, and architectural patterns**

---

## Table of Contents

- [Architectural Overview](#architectural-overview)
- [Monorepo Structure](#monorepo-structure)
- [Core Components](#core-components)
- [Client/Server Model](#clientserver-model)
- [Data Flow](#data-flow)
- [Design Patterns](#design-patterns)
- [Technology Decisions](#technology-decisions)

---

## Architectural Overview

### High-Level Architecture

OpenCode follows a **client/server architecture** that separates concerns between:

1. **Server** - Handles AI interactions, file operations, sessions, and state
2. **Clients** - Interface with users (TUI, desktop app, web console, IDE integrations)

```
┌─────────────────────────────────────────────────────────────┐
│                         Clients                              │
├──────────────┬──────────────┬──────────────┬────────────────┤
│     TUI      │  Desktop App │  Web Console │  IDE (via ACP) │
│   (Terminal) │   (SolidJS)  │   (Astro)    │  (Zed, VSCode) │
└──────┬───────┴──────┬───────┴──────┬───────┴────────┬───────┘
       │              │              │                │
       └──────────────┴──────────────┴────────────────┘
                            │
                    HTTP/WebSocket/JSONRPC
                            │
       ┌────────────────────┴────────────────────┐
       │              OpenCode Server            │
       │  ┌──────────────────────────────────┐  │
       │  │     HTTP Server (Hono)           │  │
       │  ├──────────────────────────────────┤  │
       │  │  Session    │  Project  │  Auth  │  │
       │  │  Manager    │  Manager  │  Layer │  │
       │  ├──────────────────────────────────┤  │
       │  │    Prompt   │   Tool    │   LSP  │  │
       │  │   System    │  Registry │ Client │  │
       │  ├──────────────────────────────────┤  │
       │  │   Provider System (AI SDK)       │  │
       │  │  Anthropic│OpenAI│Google│Bedrock │  │
       │  ├──────────────────────────────────┤  │
       │  │    Storage  │   File    │  Config│  │
       │  │    Layer    │  System   │ Manager│  │
       │  └──────────────────────────────────┘  │
       └─────────────────────────────────────────┘
                            │
       ┌────────────────────┴────────────────────┐
       │         External Services               │
       ├──────────────┬──────────────┬───────────┤
       │ AI Providers │ LSP Servers  │   MCP     │
       │ (Claude,GPT) │ (ts,py,rust) │  Servers  │
       └──────────────┴──────────────┴───────────┘
```

### Key Architectural Principles

1. **Separation of Concerns** - Server handles logic, clients handle presentation
2. **Protocol-First** - Standard protocols (HTTP, JSON-RPC, ACP, MCP) for integration
3. **Provider Agnostic** - Abstract AI providers behind unified interface
4. **Multi-Project** - Support multiple simultaneous projects with isolation
5. **Extensible** - Plugin system, MCP servers, custom tools
6. **Type-Safe** - TypeScript throughout with Zod validation

---

## Monorepo Structure

OpenCode is organized as a **Bun-powered monorepo** using workspaces:

```
opencode/
├── packages/
│   ├── opencode/          # Core CLI & Server (main package)
│   ├── desktop/           # Desktop app (SolidJS + Tauri potential)
│   ├── console/           # Web console & management
│   │   ├── app/          # Frontend (SolidJS)
│   │   ├── core/         # Business logic + Drizzle ORM
│   │   ├── function/     # Serverless functions
│   │   ├── mail/         # Email templates (React Email)
│   │   └── resource/     # Cloud resources (Cloudflare/Node)
│   ├── tui/              # Terminal UI (Go - being deprecated)
│   ├── web/              # Documentation site (Astro/Starlight)
│   ├── ui/               # Shared UI components
│   ├── sdk/              # Client SDKs
│   │   ├── go/          # Go SDK
│   │   └── js/          # JavaScript/TypeScript SDK
│   ├── plugin/           # Plugin SDK (@opencode-ai/plugin)
│   ├── slack/            # Slack integration bot
│   ├── function/         # Shared cloud functions
│   ├── identity/         # Identity/auth services
│   └── script/           # Build & utility scripts
├── infra/                # SST infrastructure definitions
│   ├── app.ts           # Application infrastructure
│   ├── console.ts       # Console infrastructure
│   ├── desktop.ts       # Desktop infrastructure
│   └── stage.ts         # Stage management
├── github/               # GitHub Action integration
├── specs/                # API specifications
│   └── project.md       # Project API spec
├── sst.config.ts         # SST configuration
├── package.json          # Root package with workspaces
├── bun.lock              # Bun lockfile
└── turbo.json            # Turborepo config
```

### Package Relationships

```
┌─────────────────────────────────────────────────────────┐
│                   @opencode-ai/sdk                      │
│              (Shared TypeScript SDK)                    │
└────────────────────────┬────────────────────────────────┘
                         │ used by
        ┌────────────────┼────────────────┐
        │                │                │
   ┌────▼────┐      ┌────▼────┐     ┌────▼────┐
   │opencode │      │ desktop │     │ console │
   │  (CLI)  │      │  (GUI)  │     │  (Web)  │
   └─────────┘      └─────────┘     └─────────┘
        │                │                │
        └────────────────┼────────────────┘
                         │
                    ┌────▼────┐
                    │@opencode│
                    │-ai/     │
                    │plugin   │
                    └─────────┘
```

---

## Core Components

### 1. Server (`packages/opencode/src/server/`)

**File**: `server/server.ts` (44KB, 1589 lines)

The HTTP server built on **Hono** framework providing:

#### Key Responsibilities:
- RESTful API for projects, sessions, messages
- Server-Sent Events (SSE) streaming
- WebSocket support for real-time updates
- Error handling and standardized responses
- CORS configuration for cross-origin access

#### Server Initialization:
```typescript
// From server/server.ts
import { Hono } from "hono"
import { cors } from "hono/cors"
import { stream, streamSSE } from "hono/streaming"

export namespace Server {
  const app = new Hono()
  
  export const App = lazy(() => app
    .onError((err, c) => {
      // Centralized error handling
      if (err instanceof NamedError) {
        let status: ContentfulStatusCode
        if (err instanceof Storage.NotFoundError) status = 404
        else if (err instanceof Provider.ModelNotFoundError) status = 400
        else status = 500
        return c.json(err.toObject(), { status })
      }
      // ...
    })
    .use(async (c, next) => {
      // Middleware chain
    })
  )
}
```

#### API Structure:
- **Projects**: CRUD operations for projects
- **Sessions**: Session lifecycle management
- **Messages**: Chat message handling
- **Files**: File operations and status
- **Permissions**: Permission approval workflow
- **Provider**: Model and config queries
- **TUI Routes**: Special routes for TUI client

### 2. Session Management (`packages/opencode/src/session/`)

**Files**:
- `session/index.ts` - Session lifecycle (430 lines)
- `session/message-v2.ts` - Message handling (18KB)
- `session/prompt.ts` - Prompt construction (54KB, 1766 lines)
- `session/compaction.ts` - History compaction
- `session/lock.ts` - Concurrency control
- `session/revert.ts` - Undo functionality
- `session/summary.ts` - Session summarization
- `session/todo.ts` - Task tracking

#### Session Info Schema:
```typescript
export const Info = z.object({
  id: Identifier.schema("session"),
  projectID: z.string(),
  directory: z.string(),
  parentID: Identifier.schema("session").optional(),
  summary: z.object({
    diffs: Snapshot.FileDiff.array(),
  }).optional(),
  share: z.object({
    url: z.string(),
  }).optional(),
  title: z.string(),
  version: z.string(),
  time: z.object({
    created: z.number(),
    updated: z.number(),
    compacting: z.number().optional(),
  }),
  revert: z.object({
    messageID: z.string(),
    partID: z.string().optional(),
    snapshot: z.string().optional(),
    diff: z.string().optional(),
  }).optional(),
})
```

#### Session Events:
- `session.updated` - Session state changed
- `session.deleted` - Session removed
- `session.error` - Error occurred in session

#### Session Lifecycle:
```
┌─────────────┐
│   Create    │
│   Session   │
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│ Initialize  │────▶│   Lock      │
│  Prompt     │     │  Session    │
└──────┬──────┘     └─────────────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│   Process   │────▶│   Stream    │
│  Messages   │     │  Response   │
└──────┬──────┘     └─────────────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│   Compact   │────▶│   Unlock    │
│  (Optional) │     │  Session    │
└─────────────┘     └─────────────┘
```

### 3. Prompt System (`packages/opencode/src/session/prompt.ts`)

**The largest and most complex module** (54KB, 1766 lines)

#### Responsibilities:
- Assemble context from multiple sources
- Inject system prompts and agent guidelines
- Format messages for AI providers
- Include tool descriptions
- Add file context and LSP diagnostics
- Manage token limits
- Handle streaming responses

#### Key Classes:
- `SessionPrompt` - Main prompt orchestrator
- `PromptContext` - Context accumulator
- `SystemPromptBuilder` - System prompt construction

#### Context Sources:
1. **System Prompts** - Agent behavior guidelines
2. **AGENTS.md Files** - Project-specific instructions
3. **File Content** - Relevant code files
4. **LSP Data** - Diagnostics, types, symbols
5. **Tool Descriptions** - Available tools and their schemas
6. **Message History** - Conversation context
7. **Snapshots** - File state snapshots

### 4. Tool System (`packages/opencode/src/tool/`)

**Registry-based architecture** with pluggable tools

#### Tool Registry (`tool/registry.ts`):
```typescript
export namespace ToolRegistry {
  export function create(input: Instance) {
    const tools = {
      read: ReadTool(input),
      write: WriteTool(input),
      edit: EditTool(input),
      multiedit: MultiEditTool(input),
      bash: BashTool(input),
      grep: GrepTool(input),
      glob: GlobTool(input),
      ls: LsTool(input),
      "lsp-diagnostics": LSPDiagnosticsTool(input),
      "lsp-hover": LSPHoverTool(input),
      patch: PatchTool(input),
      task: TaskTool(input),
      todo: TodoTool(input),
      webfetch: WebFetchTool(input),
    }
    
    return tools
  }
}
```

#### Built-in Tools:

| Tool | Purpose | File Size |
|------|---------|-----------|
| `read` | Read file content | 5.7KB |
| `write` | Write file content | 2.5KB |
| `edit` | Edit file with diffs | 19.2KB |
| `multiedit` | Multi-file editing | 1.5KB |
| `bash` | Execute shell commands | 8.0KB |
| `grep` | Search file content | 3.2KB |
| `glob` | Find files by pattern | 1.9KB |
| `ls` | List directory | 2.8KB |
| `lsp-diagnostics` | Get LSP errors | 894B |
| `lsp-hover` | Get type info | 982B |
| `patch` | Apply patches | 6.7KB |
| `task` | Task management | 3.4KB |
| `todo` | TODO tracking | 1.1KB |
| `webfetch` | Fetch web content | 5.5KB |

Each tool has:
- TypeScript implementation (`.ts`)
- Prompt description (`.txt`)
- Zod schema for parameters
- Permission requirements

### 5. LSP Integration (`packages/opencode/src/lsp/`)

**Files**:
- `lsp/index.ts` - Main LSP orchestrator (7.6KB)
- `lsp/server.ts` - Server lifecycle (30KB)
- `lsp/client.ts` - Client communication (6.1KB)
- `lsp/language.ts` - Language detection (2.2KB)

#### LSP Architecture:
```
┌──────────────────────────────────────┐
│       OpenCode LSP Manager           │
├──────────────────────────────────────┤
│  ┌────────────────────────────────┐  │
│  │   Language Server Clients      │  │
│  ├─────────┬─────────┬────────────┤  │
│  │TypeScript│ Python  │   Rust    │  │
│  │  (tsls)  │ (pyright│  (ra)     │  │
│  └─────────┴─────────┴────────────┘  │
│              │                        │
│  ┌───────────▼─────────────────────┐ │
│  │    JSON-RPC Communication       │ │
│  └─────────────────────────────────┘ │
└──────────────────────────────────────┘
                │
    ┌───────────┴───────────┐
    │                       │
┌───▼────┐         ┌────────▼─────┐
│Language│         │   Language    │
│Server  │         │   Server      │
│(tsls)  │         │   (pyright)   │
└────────┘         └───────────────┘
```

#### Features:
- **Diagnostics** - Real-time error detection
- **Hover** - Type information on demand
- **Document Symbols** - Code structure analysis
- **Workspace Symbols** - Project-wide symbol search
- **Multi-Language** - Automatic language detection

### 6. Provider System (`packages/opencode/src/provider/`)

**Files**:
- `provider/provider.ts` - Provider interface (20KB)
- `provider/models.ts` - Model registry
- `provider/transform.ts` - Response transformation

#### Provider Hierarchy:
```
┌─────────────────────────────────────────┐
│        Provider Interface               │
│  (Unified AI provider abstraction)      │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┼────────────┬───────────────┐
    │            │            │               │
┌───▼───┐   ┌───▼───┐   ┌────▼────┐   ┌─────▼─────┐
│Anthrop│   │OpenAI │   │ Google  │   │  Bedrock  │
│  ic   │   │       │   │ Vertex  │   │           │
└───────┘   └───────┘   └─────────┘   └───────────┘
```

#### Model Registry:
- Model capability detection
- Token limit management
- Cost calculation
- Streaming support
- Vision/multimodal support

### 7. Configuration System (`packages/opencode/src/config/`)

**File**: `config/config.ts` (28KB, 757 lines)

#### Configuration Hierarchy:
```
Global Config (~/.opencode/config.json)
         │
         ├─> Project Config (.opencode/config.json)
         │
         └─> Environment Variables
                   │
                   └─> CLI Flags (highest priority)
```

#### Config Schema:
- **Provider settings** - API keys, endpoints
- **Model preferences** - Default models, fallbacks
- **LSP configuration** - Language server paths
- **MCP servers** - External MCP server configs
- **File ignore patterns** - .gitignore-style patterns
- **Security settings** - Sandbox, permissions

### 8. Project Management (`packages/opencode/src/project/`)

**Files**:
- `project/instance.ts` - Project instance
- `project/project.ts` - Project lifecycle
- `project/state.ts` - Project state
- `project/bootstrap.ts` - Initialization

#### Multi-Project Support:
```
┌──────────────────────────────────────────┐
│        OpenCode Server                   │
├──────────────────────────────────────────┤
│  Project A           Project B           │
│  ┌──────────────┐   ┌──────────────┐    │
│  │ Sessions 1,2 │   │ Sessions 3,4 │    │
│  │              │   │              │    │
│  │ Directory:   │   │ Directory:   │    │
│  │ /app/frontend│   │ /app/backend │    │
│  └──────────────┘   └──────────────┘    │
└──────────────────────────────────────────┘
```

Each project maintains:
- Isolated sessions
- Separate configuration
- Independent LSP servers
- Distinct file contexts

### 9. Storage Layer (`packages/opencode/src/storage/`)

**File-based storage** using filesystem:

```
~/.opencode/
├── data/
│   ├── projects/
│   │   ├── {projectID}/
│   │   │   ├── sessions/
│   │   │   │   ├── {sessionID}/
│   │   │   │   │   ├── info.json
│   │   │   │   │   ├── messages/
│   │   │   │   │   │   ├── {messageID}.json
│   │   │   │   │   └── snapshots/
│   │   │   │   │       ├── {snapshotID}/
│   │   │   │   └── ...
│   │   │   └── info.json
│   │   └── ...
│   └── shares/
│       └── {shareID}.json
├── logs/
│   └── opencode.log
└── config.json
```

### 10. ACP Protocol (`packages/opencode/src/acp/`)

**Agent Client Protocol** for IDE integration

**Files**:
- `acp/agent.ts` - Agent implementation (17KB)
- `acp/session.ts` - ACP session mapping
- `acp/client.ts` - Client capabilities
- `acp/server.ts` - ACP server startup
- `acp/README.md` - Protocol documentation

#### ACP Session Manager:
```typescript
export class ACPSessionManager {
  private sessions: Map<string, ACPSessionState>
  
  // Map ACP sessions to OpenCode sessions
  create(cwd: string, mcpServers?: any): ACPSessionState
  load(sessionId: string): ACPSessionState | undefined
  close(sessionId: string): void
}
```

---

## Client/Server Model

### Communication Protocols

#### 1. **HTTP REST API**
Primary protocol for programmatic access:

```
GET    /project                          # List projects
POST   /project/init                     # Create project
GET    /project/:id/session              # List sessions
POST   /project/:id/session              # Create session
POST   /project/:id/session/:sid/message # Send message
GET    /project/:id/session/:sid/message # Get messages
POST   /project/:id/session/:sid/compact # Compact session
```

#### 2. **Server-Sent Events (SSE)**
For streaming AI responses:

```typescript
GET /project/:id/session/:sid/message/:mid/stream
```

#### 3. **JSON-RPC (ACP)**
For IDE agent integration:

```typescript
// Initialize
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{...}}

// Create session
{"jsonrpc":"2.0","id":2,"method":"session/new","params":{...}}

// Send prompt
{"jsonrpc":"2.0","id":3,"method":"session/prompt","params":{...}}
```

### Client Types

#### **TUI Client** (Go, being replaced)
- Full-screen terminal interface
- Direct WebSocket connection
- Real-time streaming
- Keyboard-driven navigation

#### **Desktop Client** (SolidJS)
- Native-feeling GUI
- HTTP + SSE communication
- Rich UI components
- File tree, diff viewer, chat

#### **Web Console** (Astro)
- Browser-based access
- Authentication and authorization
- Project management dashboard
- Team collaboration features

#### **IDE Integration** (ACP)
- JSON-RPC over stdio
- Editor-native experience
- Context from IDE
- Bidirectional communication

---

## Data Flow

### Message Processing Flow

```
User Input (Client)
       │
       ▼
┌──────────────────┐
│  HTTP Request    │
│  POST /message   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Server Router   │
│   (Hono)         │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Session Lock    │
│  (Acquire)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ SessionPrompt    │
│ .prompt()        │
└────────┬─────────┘
         │
         ├──────────┐
         │          ▼
         │    ┌──────────────┐
         │    │ Build Context│
         │    │ - System     │
         │    │ - Files      │
         │    │ - LSP        │
         │    │ - History    │
         │    └──────┬───────┘
         │           │
         ▼           ▼
   ┌─────────────────────┐
   │   Provider.call()   │
   │   (AI Request)      │
   └───────────┬─────────┘
               │
               ▼
   ┌─────────────────────┐
   │  Stream Response    │
   │  (SSE or JSON-RPC)  │
   └───────────┬─────────┘
               │
               ▼
   ┌─────────────────────┐
   │   Tool Execution    │
   │   (if requested)    │
   └───────────┬─────────┘
               │
               ▼
   ┌─────────────────────┐
   │  Store Message      │
   │  (Storage Layer)    │
   └───────────┬─────────┘
               │
               ▼
   ┌─────────────────────┐
   │  Session Unlock     │
   └───────────┬─────────┘
               │
               ▼
        Response (Client)
```

### File Edit Flow

```
AI suggests edit
       │
       ▼
┌──────────────────┐
│   Parse Tool     │
│   Call (edit)    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Permission      │
│  Check           │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Apply Diff      │
│  (@pierre/       │
│   precision-diffs│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Create Snapshot │
│  (for revert)    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Write File      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Trigger LSP     │
│  Re-analysis     │
└────────┬─────────┘
         │
         ▼
   Return Success
```

---

## Design Patterns

### 1. **Namespace Pattern**
Core modules are organized as TypeScript namespaces:

```typescript
export namespace Session {
  export const Info = z.object({...})
  export type Info = z.output<typeof Info>
  
  export const Event = {
    Updated: Bus.event("session.updated", ...),
    Deleted: Bus.event("session.deleted", ...),
  }
  
  export function create(input) {...}
  export function load(id) {...}
}
```

Benefits:
- Clear module boundaries
- Grouping related functionality
- No class overhead
- Tree-shakeable exports

### 2. **Event Bus Pattern**
Decoupled event-driven architecture:

```typescript
// Define event
export const Event = {
  Updated: Bus.event("session.updated", schema)
}

// Publish
Bus.publish(Session.Event.Updated, { info })

// Subscribe
Bus.subscribe(Session.Event.Updated, (event) => {
  console.log("Session updated:", event.info)
})
```

### 3. **Provider Pattern**
Abstract AI providers behind unified interface:

```typescript
interface Provider {
  call(messages, tools): Promise<Response>
  stream(messages, tools): AsyncGenerator<Chunk>
}

// Different implementations
AnthropicProvider implements Provider
OpenAIProvider implements Provider
LocalProvider implements Provider
```

### 4. **Repository Pattern**
Storage abstraction:

```typescript
export namespace Storage {
  export function saveSession(info): Promise<void>
  export function loadSession(id): Promise<Info>
  export function deleteSession(id): Promise<void>
}
```

### 5. **Builder Pattern**
Complex object construction:

```typescript
class PromptBuilder {
  private context: PromptContext = {}
  
  addSystem(prompt: string): this { ... }
  addFiles(files: File[]): this { ... }
  addHistory(messages: Message[]): this { ... }
  addTools(tools: Tool[]): this { ... }
  
  build(): Prompt { ... }
}
```

### 6. **Lazy Initialization**
Defer expensive operations:

```typescript
import { lazy } from "../util/lazy"

export const App = lazy(() => {
  // Expensive initialization only happens on first access
  const app = new Hono()
  // ... setup routes
  return app
})
```

### 7. **Middleware Chain**
Request processing pipeline:

```typescript
app
  .use(cors())
  .use(async (c, next) => {
    // Logging middleware
    await next()
  })
  .use(async (c, next) => {
    // Auth middleware
    await next()
  })
```

---

## Technology Decisions

### Why Bun over Node.js?
- **Faster** - 3x faster startup, better performance
- **Modern** - Built-in TypeScript, ESM first
- **Better DX** - Simplified tooling, no build step for dev
- **Native APIs** - `Bun.file()`, `Bun.write()` are faster

### Why Hono over Express?
- **Modern** - Web standard APIs (Request/Response)
- **Fast** - Optimized routing, minimal overhead
- **Type-safe** - Better TypeScript support
- **Edge-ready** - Runs on Cloudflare Workers, Deno, Bun

### Why SolidJS over React?
- **Performance** - No virtual DOM, true reactivity
- **Simpler** - Less boilerplate, more intuitive
- **Smaller** - Smaller bundle sizes
- **Modern** - JSX without React overhead

### Why Zod over alternatives?
- **Type inference** - Generate TypeScript types from schemas
- **Runtime validation** - Catch errors at boundaries
- **Composable** - Build complex schemas from simple ones
- **Error messages** - Detailed validation errors

### Why tree-sitter over regex?
- **Accurate** - Full syntax tree parsing
- **Incremental** - Update parse tree on edits
- **Multi-language** - Same API for all languages
- **Fast** - Written in C, highly optimized

### Why SST for infrastructure?
- **Type-safe** - Infrastructure as TypeScript code
- **Multi-provider** - Cloudflare, AWS, etc.
- **Live development** - Hot reload infrastructure
- **Modern** - Built for serverless and edge

---

## Summary

OpenCode's architecture is designed for:

- **Flexibility** - Client/server split enables multiple interfaces
- **Extensibility** - Protocols, plugins, MCP servers
- **Performance** - Bun runtime, efficient algorithms
- **Type Safety** - TypeScript + Zod throughout
- **Modularity** - Clear component boundaries
- **Scalability** - Multi-project, multi-session support

The architecture supports the core mission: providing a powerful, open-source, provider-agnostic AI coding agent that developers can trust, extend, and run anywhere.

---

## Next Steps

- **[02-cli-reference.md](./02-cli-reference.md)** - Explore CLI commands
- **[03-session-management.md](./03-session-management.md)** - Deep dive into sessions
- **[15-server-architecture.md](./15-server-architecture.md)** - Server API details

