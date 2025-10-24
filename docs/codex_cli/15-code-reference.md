# Codex CLI - Code Reference

## Table of Contents
- [Critical Files Index](#critical-files-index)
- [Module Documentation](#module-documentation)
- [Key Structs and Traits](#key-structs-and-traits)
- [Important Functions](#important-functions)
- [Cross-References](#cross-references)

---

## Critical Files Index

### Entry Points

| File | Purpose | LOC | Key Functions |
|------|---------|-----|---------------|
| `codex-cli/bin/codex.js` | Node.js wrapper | 100 | Platform detection, binary spawning |
| `cli/src/main.rs` | Rust CLI entry | 300 | Argument parsing, mode routing |
| `core/src/codex.rs` | Core orchestrator | 1200 | Event loop, turn processing |

### Core Modules

| File | Purpose | LOC | Dependencies |
|------|---------|-----|-------------|
| `core/src/client.rs` | Model client | 800 | reqwest, futures |
| `core/src/config.rs` | Configuration | 600 | serde, yaml |
| `core/src/state/mod.rs` | State management | 500 | tokio |
| `core/src/conversation_history.rs` | History tracking | 400 | serde_json |

### Tool System

| File | Purpose | LOC | Key Types |
|------|---------|-----|-----------|
| `core/src/tools/mod.rs` | Tool infrastructure | 200 | ToolRouter, ToolContext |
| `core/src/tools/spec.rs` | Tool specifications | 600 | ToolSpec, JsonSchema |
| `core/src/tools/router.rs` | Tool routing | 300 | ToolRegistry |
| `core/src/tools/handlers/shell.rs` | Shell tool | 400 | ExecOutput |
| `core/src/tools/handlers/apply_patch.rs` | Patch tool | 500 | ApplyPatchAction |

### UI Layer

| File | Purpose | LOC | Framework |
|------|---------|-----|-----------|
| `tui/src/app.rs` | TUI application | 800 | ratatui |
| `tui/src/render.rs` | Rendering logic | 600 | ratatui |
| `tui/src/events.rs` | Event handling | 400 | crossterm |
| `exec/src/lib.rs` | Non-interactive mode | 300 | tokio |

### System Prompts

| File | Purpose | LOC | Usage |
|------|---------|-----|-------|
| `core/prompt.md` | Base system prompt | 311 | Regular mode |
| `core/review_prompt.md` | Review mode | 87 | Code review |
| `core/gpt_5_codex_prompt.md` | GPT-5 specific | 107 | GPT-5 models |

---

## Module Documentation

### codex-core

**Purpose**: Core agent logic and orchestration

**Modules**:
```
core/
├── src/
│   ├── codex.rs              # Main orchestrator
│   ├── client.rs             # LLM client
│   ├── client_common.rs      # Shared client logic
│   ├── config.rs             # Configuration
│   ├── config_loader/        # Config loading
│   ├── state/                # State management
│   ├── tools/                # Tool system
│   ├── custom_prompts.rs     # Custom prompts
│   ├── project_doc.rs        # AGENTS.md loading
│   ├── conversation_history.rs
│   ├── turn_diff_tracker.rs
│   ├── error.rs              # Error types
│   └── ...
```

**Public API**:
```rust
// Main entry point
pub struct Codex;
impl Codex {
    pub async fn spawn(params: CodexParams) -> Result<CodexSpawnOk>;
    pub async fn submit(&self, op: Op) -> Result<()>;
    pub async fn recv_event(&self) -> Result<Event>;
}

// Configuration
pub struct Config;
impl Config {
    pub fn load() -> Result<Config>;
    pub fn load_with_overrides(overrides: &CliConfigOverrides) -> Result<Config>;
}

// Events
pub enum Event {
    SessionConfigured { ... },
    TaskStarted { turn_id: TurnId },
    AgentMessageDelta { delta: String },
    ItemCompleted { ... },
    TaskCompleted { ... },
    // ... more variants
}
```

### codex-cli

**Purpose**: Command-line interface

**Modules**:
```
cli/
├── src/
│   ├── main.rs               # Entry point
│   ├── login.rs              # Login commands
│   ├── completion.rs         # Shell completions
│   ├── sandbox.rs            # Sandbox debug
│   └── ...
```

**Key Structures**:
```rust
#[derive(Parser)]
struct MultitoolCli {
    #[clap(flatten)]
    pub config_overrides: CliConfigOverrides,
    
    #[clap(flatten)]
    feature_toggles: FeatureToggles,
    
    #[clap(subcommand)]
    subcommand: Option<Subcommand>,
}

#[derive(Subcommand)]
enum Subcommand {
    Exec(ExecCli),
    Login(LoginCommand),
    Mcp(McpCli),
    // ...
}
```

### codex-tui

**Purpose**: Terminal user interface

**Modules**:
```
tui/
├── src/
│   ├── app.rs                # App state
│   ├── render.rs             # Rendering
│   ├── events.rs             # Event handling
│   ├── widgets/              # Custom widgets
│   └── ...
```

**Key Types**:
```rust
pub struct App {
    pub state: AppState,
    pub codex: Option<Codex>,
    pub conversation_history: Vec<Turn>,
    pub current_input: String,
    pub mode: Mode,
}

pub enum AppState {
    Initializing,
    Ready,
    Processing,
    AwaitingApproval,
    Error { message: String },
}
```

### codex-exec

**Purpose**: Non-interactive execution

**Key Function**:
```rust
pub async fn run_exec_mode(
    cli: ExecCli,
    config_overrides: &CliConfigOverrides,
) -> ExitCode;
```

### codex-chatgpt

**Purpose**: ChatGPT-specific client

**Features**:
- Chat Completions API support
- Streaming response handling
- Token aggregation

### codex-protocol

**Purpose**: Protocol types and serialization

**Types**:
```rust
pub struct ResponsesApiRequest { ... }
pub struct ChatCompletionsRequest { ... }
pub enum ResponseEvent { ... }
```

---

## Key Structs and Traits

### Codex

**Location**: `core/src/codex.rs`

```rust
pub struct Codex {
    next_id: AtomicU64,
    tx_sub: Sender<Submission>,
    rx_event: Receiver<Event>,
}
```

**Methods**:
- `spawn()` - Create new instance
- `submit()` - Submit operation
- `recv_event()` - Receive event

### SessionState

**Location**: `core/src/state/mod.rs`

```rust
pub struct SessionState {
    pub session_id: SessionId,
    pub conversation_id: ConversationId,
    pub services: SessionServices,
    pub active_turn: Option<ActiveTurn>,
    pub history: ConversationHistory,
}
```

### ModelClient

**Location**: `core/src/client.rs`

```rust
pub struct ModelClient {
    config: Arc<Config>,
    auth_manager: Option<Arc<AuthManager>>,
    client: reqwest::Client,
    provider: ModelProviderInfo,
}
```

**Methods**:
- `stream()` - Stream from model
- `stream_with_task_kind()` - Stream with specific prompt
- `get_model_context_window()` - Get context limit

### ToolRouter

**Location**: `core/src/tools/router.rs`

```rust
pub struct ToolRouter {
    registry: ToolRegistry,
    orchestrator: ToolOrchestrator,
    context: Arc<ToolContext>,
}
```

**Methods**:
- `route()` - Route tool call to handler

### ToolHandler Trait

**Location**: `core/src/tools/mod.rs`

```rust
#[async_trait]
pub trait ToolHandler: Send + Sync {
    async fn execute(&self, args: Value) -> Result<ToolOutput>;
    fn name(&self) -> &str;
    fn description(&self) -> &str;
}
```

### Config

**Location**: `core/src/config.rs`

```rust
pub struct Config {
    pub model: Option<String>,
    pub provider: Option<String>,
    pub providers: HashMap<String, ProviderConfig>,
    pub approval_mode: ApprovalMode,
    pub sandbox_mode: SandboxMode,
    // ... many more fields
}
```

---

## Important Functions

### Core Functions

#### build_prompt

**Location**: `core/src/codex.rs`

```rust
async fn build_prompt(
    state: &SessionState,
    input: &UserInput,
    context: &[TurnContextItem],
) -> Result<Prompt>
```

**Purpose**: Assembles complete prompt from system prompt, AGENTS.md, history, and user input.

#### handle_user_turn

**Location**: `core/src/codex.rs`

```rust
async fn handle_user_turn(
    state: &mut SessionState,
    input: UserInput,
    context: Vec<TurnContextItem>,
    tx_event: &Sender<Event>,
) -> Result<()>
```

**Purpose**: Main turn processing logic - builds prompt, streams response, executes tools.

#### execute_tool

**Location**: `core/src/codex.rs`

```rust
async fn execute_tool(
    state: &SessionState,
    tool_name: &str,
    arguments: &str,
) -> Result<ToolResult>
```

**Purpose**: Parses tool arguments, checks approval, routes to handler.

### Configuration Functions

#### load_config

**Location**: `core/src/config_loader/mod.rs`

```rust
pub fn load_config() -> Result<Config>
```

**Purpose**: Loads configuration from file, environment, and defaults.

#### get_user_instructions

**Location**: `core/src/project_doc.rs`

```rust
pub async fn get_user_instructions(config: &Config) -> Option<String>
```

**Purpose**: Discovers and concatenates AGENTS.md files.

### Client Functions

#### stream_responses

**Location**: `core/src/client.rs`

```rust
async fn stream_responses(
    &self,
    prompt: &Prompt,
    task_kind: TaskKind,
) -> Result<ResponseStream>
```

**Purpose**: Streams from Responses API.

#### stream_chat_completions

**Location**: `chatgpt/src/lib.rs`

```rust
pub async fn stream_chat_completions(
    prompt: &Prompt,
    model_family: &ModelFamily,
    client: &reqwest::Client,
    provider: &ModelProviderInfo,
) -> Result<ResponseStream>
```

**Purpose**: Streams from Chat Completions API.

### Tool Functions

#### format_exec_output_for_model

**Location**: `core/src/tools/mod.rs`

```rust
pub fn format_exec_output_for_model(exec_output: &ExecToolCallOutput) -> String
```

**Purpose**: Formats command output for LLM, truncating if necessary.

### Utility Functions

#### parse_unified_diff

**Location**: `apply-patch/src/lib.rs`

```rust
pub fn parse_unified_diff(patch_text: &str) -> Result<Patch>
```

**Purpose**: Parses unified diff format.

#### apply_hunk

**Location**: `apply-patch/src/lib.rs`

```rust
pub fn apply_hunk(lines: &mut Vec<&str>, hunk: &Hunk) -> Result<()>
```

**Purpose**: Applies single diff hunk to file.

---

## Cross-References

### Flow: User Input → Response

```
User Input
    ↓
main.rs::main()
    ↓
codex.rs::Codex::submit(Op::UserTurn)
    ↓
codex.rs::handle_user_turn()
    ├─→ project_doc.rs::get_user_instructions()
    ├─→ codex.rs::build_prompt()
    └─→ client.rs::ModelClient::stream()
        ├─→ client.rs::stream_responses() [Responses API]
        └─→ chatgpt/lib.rs::stream_chat_completions() [Chat API]
    ↓
codex.rs::process_response_stream()
    ├─→ Event::AgentMessageDelta (text)
    ├─→ Event::FunctionCall (tool)
    │   └─→ router.rs::ToolRouter::route()
    │       └─→ handlers/*::execute()
    └─→ Event::TaskCompleted
    ↓
app.rs::handle_codex_event()
    └─→ render.rs::render_app()
```

### Flow: Tool Execution

```
LLM Function Call
    ↓
codex.rs::execute_tool()
    ↓
router.rs::ToolRouter::route()
    ├─→ Check security (command_safety/)
    ├─→ Check approval (approval_store/)
    └─→ orchestrator.rs::ToolOrchestrator::execute()
        ├─→ Apply sandboxing (seatbelt.rs / landlock.rs)
        └─→ handlers/*::execute()
            ├─→ shell.rs::execute_shell()
            ├─→ apply_patch.rs::apply_patch()
            ├─→ read_file.rs::read_file()
            └─→ ...
    ↓
Tool Result → Back to LLM
```

### Flow: Configuration Loading

```
main.rs
    ↓
config_loader/mod.rs::load_config()
    ├─→ Load defaults
    ├─→ config_loader/file.rs::find_config_file()
    │   └─→ Check ~/.codex/config.{yaml,json}
    ├─→ config_loader/parse.rs::parse_config_file()
    ├─→ config.rs::apply_env_overrides()
    └─→ Apply CLI overrides
    ↓
Config ready
```

---

## File Location Quick Reference

### By Feature

**Authentication**:
- `core/src/auth.rs` - AuthManager
- `cli/src/login.rs` - Login commands

**Configuration**:
- `core/src/config.rs` - Config struct
- `core/src/config_loader/` - Loading logic

**LLM Integration**:
- `core/src/client.rs` - ModelClient
- `core/src/client_common.rs` - Shared logic
- `chatgpt/src/lib.rs` - Chat Completions

**Tool System**:
- `core/src/tools/mod.rs` - Infrastructure
- `core/src/tools/spec.rs` - Specifications
- `core/src/tools/handlers/` - Implementations

**Security**:
- `core/src/sandboxing/` - Sandbox logic
- `core/src/command_safety/` - Command validation
- `core/src/seatbelt.rs` - macOS Seatbelt
- `linux-sandbox/` - Linux Landlock

**UI**:
- `tui/src/app.rs` - TUI app
- `exec/src/lib.rs` - Non-interactive

**State**:
- `core/src/state/` - State management
- `core/src/conversation_history.rs` - History
- `core/src/turn_diff_tracker.rs` - Diffs

**Prompts**:
- `core/prompt.md` - Base prompt
- `core/src/custom_prompts.rs` - Custom prompts
- `core/src/project_doc.rs` - AGENTS.md

---

## Testing

### Test Files

```
tests/
├── tools/
│   ├── shell.test.rs
│   ├── apply_patch.test.rs
│   └── ...
├── security/
│   ├── command_safety.test.rs
│   ├── path_validator.test.rs
│   └── ...
├── integration/
│   ├── end_to_end.test.rs
│   └── ...
└── fixtures/
    └── ...
```

### Running Tests

```bash
# All tests
cargo test

# Specific module
cargo test --package codex-core --lib tools

# With output
cargo test -- --nocapture

# Integration tests
cargo test --test integration
```

---

## Documentation Cross-Reference

### By Topic

**Getting Started**:
- [01-overview.md](./01-overview.md)

**Architecture**:
- [02-architecture.md](./02-architecture.md)
- [10-implementation.md](./10-implementation.md)

**Prompts**:
- [03-prompt-processing.md](./03-prompt-processing.md)
- [05-system-prompts.md](./05-system-prompts.md)

**LLM**:
- [04-llm-integration.md](./04-llm-integration.md)
- [13-authentication.md](./13-authentication.md)

**Tools**:
- [06-tool-system.md](./06-tool-system.md)
- [11-tool-implementations.md](./11-tool-implementations.md)

**Security**:
- [07-security-sandboxing.md](./07-security-sandboxing.md)

**Configuration**:
- [08-configuration.md](./08-configuration.md)

**State**:
- [09-state-management.md](./09-state-management.md)

**UI**:
- [12-ui-layer.md](./12-ui-layer.md)

**MCP**:
- [14-mcp-integration.md](./14-mcp-integration.md)

---

## Conclusion

This code reference provides an index to the Codex CLI codebase. For detailed implementation explanations, refer to the topic-specific documentation files.

**Quick Navigation**:
- Entry points: `codex-cli/bin/codex.js`, `cli/src/main.rs`
- Core logic: `core/src/codex.rs`
- Tools: `core/src/tools/`
- UI: `tui/src/`
- Config: `core/src/config.rs`
- System prompts: `core/*.md`

