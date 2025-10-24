# Codex CLI - Architecture & Design

## Table of Contents
- [System Architecture](#system-architecture)
- [Component Relationships](#component-relationships)
- [Data Flow](#data-flow)
- [Technology Stack](#technology-stack)
- [Design Patterns](#design-patterns)

---

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface                          │
│  ┌──────────────────────┐        ┌──────────────────────────┐  │
│  │   Terminal (TUI)     │        │  Non-Interactive (Exec)  │  │
│  │   • ratatui-based    │        │  • Quiet mode            │  │
│  │   • Real-time UI     │        │  • CI/CD mode            │  │
│  └──────────┬───────────┘        └──────────┬───────────────┘  │
└─────────────┼──────────────────────────────┼──────────────────┘
              │                              │
              └──────────────┬───────────────┘
                             ▼
              ┌──────────────────────────────┐
              │      CLI Entry Point         │
              │   (cli/src/main.rs)          │
              │   • Argument parsing         │
              │   • Subcommand routing       │
              └──────────────┬───────────────┘
                             ▼
              ┌──────────────────────────────┐
              │      Core Orchestrator       │
              │   (core/src/codex.rs)        │
              │   • Session management       │
              │   • Event queue              │
              └──────┬───────────────────┬───┘
                     │                   │
          ┌──────────▼────────┐   ┌─────▼──────────┐
          │   Model Client    │   │  Tool Router   │
          │  (client.rs)      │   │  (tools/)      │
          └──────┬────────────┘   └─────┬──────────┘
                 │                      │
        ┌────────▼────────┐    ┌────────▼─────────────┐
        │  LLM Provider   │    │   Tool Handlers      │
        │  • OpenAI       │    │   • shell.rs         │
        │  • Azure        │    │   • apply_patch.rs   │
        │  • Gemini       │    │   • read_file.rs     │
        │  • Ollama       │    │   • grep_files.rs    │
        └─────────────────┘    │   • list_dir.rs      │
                               │   • plan.rs          │
                               │   • mcp.rs           │
                               └──────────────────────┘
```

### Component Layers

#### Layer 1: Entry & Dispatch
- **Node.js Wrapper** (`codex-cli/bin/codex.js`): Platform detection, spawns Rust binary
- **CLI Parser** (`cli/src/main.rs`): Argument parsing with clap, subcommand routing

#### Layer 2: Core Orchestration
- **Codex Core** (`core/src/codex.rs`): Main agent logic, event loop, state management
- **Session Management** (`core/src/state/`): Tracks sessions, turns, and history

#### Layer 3: Integration
- **Model Client** (`core/src/client.rs`): LLM API communication
- **Tool Router** (`core/src/tools/router.rs`): Tool dispatch and execution

#### Layer 4: Execution
- **Tool Handlers** (`core/src/tools/handlers/`): Individual tool implementations
- **Security Layer** (`core/src/sandboxing/`, `core/src/command_safety/`): Sandboxing and validation

---

## Component Relationships

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                          Codex                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Submission Queue                      │    │
│  │  (async_channel::Sender<Submission>)               │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                         │
│  ┌────────────────▼───────────────────────────────────┐    │
│  │              Event Loop                            │    │
│  │  • Process submissions                             │    │
│  │  • Coordinate with ModelClient                     │    │
│  │  • Dispatch tool calls                             │    │
│  │  • Emit events                                     │    │
│  └────────────────┬───────────────────────────────────┘    │
│                   │                                         │
│  ┌────────────────▼───────────────────────────────────┐    │
│  │              Event Queue                           │    │
│  │  (async_channel::Receiver<Event>)                  │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Tool System Architecture

```
Tool Call (from LLM)
      │
      ▼
┌─────────────────┐
│   ToolRouter    │  ← Receives function call
│   • Validates   │
│   • Routes      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ToolRegistry   │  ← Maps tool names to handlers
│  • shell        │
│  • apply_patch  │
│  • read_file    │
│  • ...          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ToolOrchestrator│  ← Manages execution
│  • Security     │
│  • Sandboxing   │
│  • Parallelism  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Tool Handler   │  ← Executes specific tool
│  • Validates    │
│  • Executes     │
│  • Formats      │
└─────────────────┘
```

### State Flow

```
UserInput
    ↓
[Build Prompt]
    ├─ System Prompt (prompt.md)
    ├─ AGENTS.md chain
    ├─ Environment Context
    └─ User Message
    ↓
[ModelClient::stream]
    ↓
ResponseStream
    ├─ Text Deltas
    ├─ Reasoning Deltas
    └─ Function Calls
    ↓
[Process Events]
    ├─ Update UI
    ├─ Execute Tools
    └─ Track Diffs
    ↓
[Iterative Loop]
    ↓
Task Complete
```

---

## Data Flow

### Request Processing Pipeline

```
1. User Submission
   ├─ Input text
   ├─ Attached files/images
   └─ Context items

2. Prompt Assembly
   ├─ Load system prompts
   ├─ Discover AGENTS.md files
   ├─ Inject environment context
   └─ Format conversation history

3. API Request
   ├─ Build request payload
   ├─ Include tool specifications
   └─ Stream to LLM provider

4. Response Processing
   ├─ Parse SSE/streaming events
   ├─ Extract text deltas
   ├─ Detect function calls
   └─ Handle errors/retries

5. Tool Execution
   ├─ Validate tool call
   ├─ Check security policies
   ├─ Request approval if needed
   ├─ Execute in sandbox
   └─ Format result

6. Result Feedback
   ├─ Send tool output to LLM
   └─ Continue iteration

7. Turn Completion
   ├─ Save conversation history
   ├─ Track file diffs
   └─ Emit completion event
```

### Event Flow

```
Codex Event Loop
      │
      ├─→ SessionConfiguredEvent
      │
      ├─→ TaskStartedEvent
      │
      ├─→ AgentMessageDelta (streaming text)
      │   ├─→ AgentReasoningDelta (o1/o3 models)
      │   └─→ AgentReasoningSectionBreak
      │
      ├─→ ItemStartedEvent (tool call)
      │   ├─→ ExecApprovalRequestEvent (if needed)
      │   ├─→ ApplyPatchApprovalRequestEvent (if needed)
      │   └─→ ItemCompletedEvent
      │
      ├─→ TurnDiffEvent (file changes)
      │
      ├─→ TokenCountEvent (usage tracking)
      │
      └─→ TaskCompletedEvent
```

---

## Technology Stack

### Core Technologies

#### Rust Ecosystem

| Crate | Purpose | Location |
|-------|---------|----------|
| `tokio` | Async runtime | Throughout |
| `futures` | Stream abstractions | Core, client |
| `serde` / `serde_json` | Serialization | Everywhere |
| `reqwest` | HTTP client | Model client |
| `clap` | CLI parsing | Entry point |
| `ratatui` | Terminal UI | TUI layer |

#### Key Dependencies

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
futures = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["stream"] }
clap = { version = "4", features = ["derive"] }
ratatui = "0.28"
```

### Architecture Patterns

#### Async/Await Throughout

All I/O operations use `async`/`await`:
- File operations: `tokio::fs`
- Network requests: `reqwest` with streaming
- Inter-task communication: `async_channel`

```rust
// Example: Async file reading
pub async fn read_project_docs(config: &Config) -> io::Result<Option<String>> {
    let file = tokio::fs::File::open(&path).await?;
    let mut reader = tokio::io::BufReader::new(file);
    reader.read_to_end(&mut data).await?;
    // ...
}
```

#### Event-Driven Communication

Queue pair pattern for submissions and events:

```rust
pub struct Codex {
    tx_sub: Sender<Submission>,
    rx_event: Receiver<Event>,
}

// Producer
codex.submit(Op::UserTurn { ... }).await?;

// Consumer
while let Ok(event) = codex.recv_event().await {
    match event {
        Event::AgentMessageDelta { delta } => { /* ... */ }
        Event::ItemCompleted { .. } => { /* ... */ }
    }
}
```

#### Stream Processing

LLM responses are processed as streams:

```rust
let mut stream = model_client.stream(&prompt).await?;
while let Some(event) = stream.next().await {
    match event {
        ResponseEvent::Delta(delta) => { /* ... */ }
        ResponseEvent::FunctionCall(call) => { /* ... */ }
        ResponseEvent::Done => break,
    }
}
```

---

## Design Patterns

### 1. Builder Pattern

Used extensively for constructing complex objects:

```rust
// Tool registry building
let registry = ToolRegistryBuilder::new()
    .register_shell_tool(shell_tool)
    .register_apply_patch_tool(patch_tool)
    .register_read_file_tool(read_tool)
    .build();
```

### 2. Strategy Pattern

Different implementations selected at runtime:

```rust
// Model client strategy
match self.provider.wire_api {
    WireApi::Responses => self.stream_responses(prompt).await,
    WireApi::Chat => self.stream_chat_completions(prompt).await,
}
```

### 3. Registry Pattern

Tool handlers registered by name:

```rust
pub struct ToolRegistry {
    handlers: HashMap<String, Box<dyn ToolHandler>>,
}

impl ToolRegistry {
    pub fn route(&self, tool_name: &str, args: Value) -> Result<ToolResult> {
        let handler = self.handlers.get(tool_name)?;
        handler.execute(args).await
    }
}
```

### 4. Decorator Pattern

Security and sandboxing wrap tool execution:

```rust
// Approval store wraps execution
let result = approval_store.execute_with_approval(
    tool_call,
    || actual_execution(),
).await?;
```

### 5. Observer Pattern

Event emission for UI updates:

```rust
// Emit events for observers (UI, logging, etc.)
tx_event.send(Event::AgentMessageDelta { delta }).await?;
tx_event.send(Event::TokenCount { usage }).await?;
```

### 6. Factory Pattern

Creating tool specifications:

```rust
fn create_shell_tool() -> ToolSpec {
    ToolSpec::Function(ResponsesApiTool {
        name: "shell".to_string(),
        description: "Execute shell commands".to_string(),
        parameters: JsonSchema::Object { /* ... */ },
    })
}
```

---

## Key Architectural Decisions

### Why Event-Driven?

**Benefits:**
- Decouples UI from core logic
- Easy to add observers (logging, telemetry)
- Natural fit for streaming responses
- Testable without UI

### Why Tool-Based Execution?

**Benefits:**
- Clear security boundaries
- Easy to audit what agent can do
- LLM can't execute arbitrary code
- Composable and extensible

**Trade-offs:**
- More latency than direct execution
- Must design tool interfaces carefully

### Why Rust?

**Benefits:**
- Memory safety without GC pauses
- Excellent concurrency primitives
- Fast compile-time error detection
- Native performance

**Trade-offs:**
- Steeper learning curve
- Longer compile times
- More verbose than scripting languages

---

## Related Documentation

- [03-prompt-processing.md](./03-prompt-processing.md) - How prompts flow through the system
- [06-tool-system.md](./06-tool-system.md) - Detailed tool architecture
- [10-implementation.md](./10-implementation.md) - Implementation deep dive

