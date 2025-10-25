# Codex CLI - Exec Mode Internals (Non-Interactive Implementation)

> **📚 Official User Guide**: For usage instructions, see [Official exec.md](../../context/codex/docs/exec.md)
>
> **🎯 This Document**: Focuses on internal implementation of `codex exec` - the non-interactive automation mode.

---

## Quick Links

- **User Guide**: `/context/codex/docs/exec.md` - How to use `codex exec`
- **This Doc**: Implementation details for developers
- **Related**: [10-implementation.md](./10-implementation.md) - Core execution patterns

---

## Table of Contents
- [Exec Mode Overview](#exec-mode-overview)
- [Architecture](#architecture)
- [CLI Arguments](#cli-arguments)
- [Event System](#event-system)
- [Output Modes](#output-modes)
- [Structured Output](#structured-output)
- [Session Resume](#session-resume)
- [Implementation Details](#implementation-details)

---

## Exec Mode Overview

### What is Exec Mode?

**Exec mode** (`codex exec`) is Codex's **non-interactive automation mode** designed for:
- CI/CD pipelines
- Scripting and automation
- Batch processing
- Programmatic access to Codex

### Key Differences from Interactive Mode

| Feature | Interactive (`codex`) | Exec (`codex exec`) |
|---------|----------------------|---------------------|
| **Input** | Live user interaction | Single prompt string |
| **Output** | Rich TUI | Text/JSON to stdout |
| **Approvals** | Interactive prompts | Auto-handled by policy |
| **Session** | Long-lived | Single-run (or resumed) |
| **Use Case** | Development workflow | Automation/CI/CD |

---

## Architecture

### Module Structure

```
codex-rs/exec/
├── src/
│   ├── main.rs                              # Entry point (1.3KB)
│   ├── lib.rs                               # Core library (15KB)
│   ├── cli.rs                               # CLI argument parsing (3.8KB)
│   ├── exec_events.rs                       # Event type definitions (7.4KB)
│   ├── event_processor.rs                   # Event processor trait (1.3KB)
│   ├── event_processor_with_human_output.rs # Human-readable output (22KB)
│   └── event_processor_with_jsonl_output.rs # JSON Lines output (16KB)
└── tests/
    └── suite/
        └── resume.rs                        # Resume functionality tests
```

### High-Level Flow

```
┌──────────────────────────────────────────────────────────────┐
│                     codex exec "prompt"                       │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│ 1. Parse CLI Arguments (cli.rs)                              │
│    ├─ Prompt                                                 │
│    ├─ Output mode (text/json)                                │
│    ├─ Output schema (structured)                             │
│    ├─ Config overrides                                       │
│    └─ Resume session ID                                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│ 2. Initialize Codex Core                                     │
│    ├─ Load configuration                                     │
│    ├─ Authenticate                                           │
│    ├─ Create conversation manager                            │
│    └─ Set up sandbox/approval policy                         │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│ 3. Start or Resume Session                                   │
│    ├─ New: Create conversation                               │
│    └─ Resume: Load existing conversation                     │
└────────────────────┬─────────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│ 4. Send Prompt & Process Events (lib.rs)                     │
│    ├─ Send user message                                      │
│    ├─ Receive event stream from core                         │
│    └─ Route to event processor                               │
└────────────────────┬─────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐       ┌───────▼────────────┐
│ Human Output   │       │ JSON Lines Output  │
│ (text mode)    │       │ (--json mode)      │
│                │       │                    │
│ • stderr: logs │       │ • stdout: JSONL    │
│ • stdout: final│       │ • events stream    │
└────────────────┘       └────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────────┐
│ 5. Output Final Result                                       │
│    ├─ Text: Last agent message                               │
│    ├─ JSON: Event stream                                     │
│    └─ Structured: JSON matching schema                       │
└──────────────────────────────────────────────────────────────┘
```

---

## CLI Arguments

### Exec CLI Structure

**Location**: `exec/src/cli.rs`

**Actual Structure** (from source):
```rust
#[derive(Parser, Debug)]
#[command(version)]
pub struct Cli {
    /// Action to perform. If omitted, runs a new non-interactive session.
    #[command(subcommand)]
    pub command: Option<Command>,

    /// Optional image(s) to attach to the initial prompt.
    #[arg(short = 'i', long = "image")]
    pub images: Vec<PathBuf>,

    /// The prompt to send to the agent.
    #[arg(value_name = "PROMPT")]
    pub prompt: String,

    /// Allow running Codex outside a Git repository.
    #[arg(long = "skip-git-repo-check", default_value_t = false)]
    pub skip_git_repo_check: bool,

    /// Path to a JSON Schema file describing the model's final response shape.
    #[arg(long = "output-schema", value_name = "FILE")]
    pub output_schema: Option<PathBuf>,

    // ... config overrides inherited from CliConfigOverrides
}

#[derive(Parser, Debug)]
pub enum Command {
    /// Resume a previous exec session
    Resume {
        /// Session ID to resume, or --last for most recent
        session_id: Option<String>,

        #[arg(long = "last")]
        last: bool,

        /// New prompt to continue the session
        #[arg(value_name = "PROMPT")]
        prompt: String,
    },
}
```

### Key Arguments

```bash
# Basic usage
codex exec "prompt text"

# With images
codex exec -i screenshot.png "explain this"

# JSON output mode
codex exec --json "analyze code"

# Structured output with schema
codex exec --output-schema schema.json "extract data"

# Skip git check
codex exec --skip-git-repo-check "run anywhere"

# Resume session
codex exec resume SESSION_ID "continue task"
codex exec resume --last "follow up"

# Config overrides (from CliConfigOverrides)
codex exec --model gpt-5 "prompt"
codex exec --sandbox workspace-write "prompt"
codex exec -a never "prompt"  # Never ask for approval
```

---

## Event System

### Event Types

**Location**: `exec/src/exec_events.rs`

**Event Hierarchy** (from source):
```rust
/// Top-level JSONL events emitted by codex exec
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq, TS)]
#[serde(tag = "type")]
pub enum ThreadEvent {
    /// Emitted when a new thread is started as the first event.
    #[serde(rename = "thread.started")]
    ThreadStarted(ThreadStartedEvent),

    /// Emitted when a turn is started by sending a new prompt to the model.
    #[serde(rename = "turn.started")]
    TurnStarted(TurnStartedEvent),

    /// Emitted when a turn completes successfully.
    #[serde(rename = "turn.completed")]
    TurnCompleted(TurnCompletedEvent),

    /// Emitted when a turn fails.
    #[serde(rename = "turn.failed")]
    TurnFailed(TurnFailedEvent),

    /// Emitted when items are added/updated/completed in the thread.
    #[serde(rename = "item.started")]
    ItemStarted(ItemEvent),

    #[serde(rename = "item.updated")]
    ItemUpdated(ItemEvent),

    #[serde(rename = "item.completed")]
    ItemCompleted(ItemEvent),

    /// Emitted when an unrecoverable error occurs.
    #[serde(rename = "error")]
    Error(ThreadErrorEvent),
}
```

### Item Types

Items represent specific actions the agent is performing:

```rust
pub enum ItemType {
    AgentMessage,       // Assistant text response
    Reasoning,          // Thinking/reasoning summary
    CommandExecution,   // Shell command running
    FileChange,         // File edit/write
    McpToolCall,        // MCP tool invocation
    WebSearch,          // Web search performed
    TodoList,           // Plan/todo list updates
}
```

### Event Flow Example

```json
{"type":"thread.started","thread_id":"0199a213-81c0-7800-8aa1-bbab2a035a53"}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_0","type":"reasoning","text":"Analyzing repository"}}
{"type":"item.started","item":{"id":"item_1","type":"command_execution","command":"ls -la"}}
{"type":"item.completed","item":{"id":"item_1","type":"command_execution","exit_code":0,"output":"..."}}
{"type":"item.completed","item":{"id":"item_2","type":"agent_message","text":"Found 15 files"}}
{"type":"turn.completed","usage":{"input_tokens":1000,"output_tokens":50}}
```

---

## Output Modes

### 1. Human-Readable Output (Default)

**Implementation**: `event_processor_with_human_output.rs` (22KB)

**Behavior**:
- **stderr**: Activity logs, progress, commands
- **stdout**: Final agent message only

**Example**:
```bash
$ codex exec "count files"
# stderr:
Running command: ls | wc -l
Command completed: exit code 0

# stdout:
There are 42 files in this directory.
```

**Use Case**:
- Quick scripts
- Piping final output to other tools
- Clean stdout for automation

**Implementation Details**:
```rust
// Simplified representation
pub struct HumanOutputProcessor {
    stderr: io::Stderr,
    stdout: io::Stdout,
    final_message: Option<String>,
}

impl EventProcessor for HumanOutputProcessor {
    fn process_event(&mut self, event: ThreadEvent) -> Result<()> {
        match event {
            ThreadEvent::ItemCompleted(item) => {
                match item.item_type {
                    ItemType::AgentMessage => {
                        // Save for final output to stdout
                        self.final_message = Some(item.text);
                    }
                    ItemType::CommandExecution => {
                        // Log to stderr
                        writeln!(self.stderr, "Command: {}", item.command)?;
                    }
                    // ... other item types
                }
            }
            // ... other event types
        }
        Ok(())
    }

    fn finalize(&mut self) -> Result<()> {
        // Write final message to stdout
        if let Some(msg) = &self.final_message {
            writeln!(self.stdout, "{}", msg)?;
        }
        Ok(())
    }
}
```

### 2. JSON Lines Output

**Implementation**: `event_processor_with_jsonl_output.rs` (16KB)

**Behavior**:
- **stdout**: JSONL stream (one event per line)
- Real-time event streaming

**Example**:
```bash
$ codex exec --json "count files"
{"type":"thread.started","thread_id":"abc123"}
{"type":"turn.started"}
{"type":"item.started","item":{"id":"i1","type":"command_execution","command":"ls | wc -l"}}
{"type":"item.completed","item":{"id":"i1","status":"completed","exit_code":0}}
{"type":"item.completed","item":{"id":"i2","type":"agent_message","text":"42 files"}}
{"type":"turn.completed","usage":{"input_tokens":100,"output_tokens":10}}
```

**Use Case**:
- Programmatic consumption
- Real-time monitoring
- Detailed logging
- Building on top of Codex

**Implementation Details**:
```rust
pub struct JsonlOutputProcessor {
    stdout: io::Stdout,
}

impl EventProcessor for JsonlOutputProcessor {
    fn process_event(&mut self, event: ThreadEvent) -> Result<()> {
        // Serialize event to JSON
        let json = serde_json::to_string(&event)?;

        // Write to stdout with newline
        writeln!(self.stdout, "{}", json)?;

        // Flush to ensure real-time output
        self.stdout.flush()?;

        Ok(())
    }
}
```

---

## Structured Output

### Overview

**Structured output** forces the agent's final response to match a JSON schema.

**Use Cases**:
- Extracting structured data
- Reliable automation
- Type-safe responses
- Integration with other systems

### JSON Schema

**Requirement**: Must follow [OpenAI strict schema rules](https://platform.openai.com/docs/guides/structured-outputs)

**Example Schema** (`schema.json`):
```json
{
  "type": "object",
  "properties": {
    "project_name": { "type": "string" },
    "programming_languages": {
      "type": "array",
      "items": { "type": "string" }
    },
    "file_count": { "type": "integer" },
    "has_tests": { "type": "boolean" }
  },
  "required": ["project_name", "programming_languages"],
  "additionalProperties": false
}
```

### Usage

```bash
# Create schema file
cat > schema.json <<EOF
{
  "type": "object",
  "properties": {
    "summary": { "type": "string" },
    "issues_found": { "type": "integer" }
  },
  "required": ["summary", "issues_found"],
  "additionalProperties": false
}
EOF

# Run with structured output
codex exec --output-schema schema.json "analyze this codebase"

# Output (stdout):
{"summary":"Well-structured project with good tests","issues_found":3}
```

### Implementation Flow

```
┌────────────────────────────────────────────────────┐
│ 1. User provides JSON Schema                       │
└─────────────────┬──────────────────────────────────┘
                  │
┌─────────────────▼──────────────────────────────────┐
│ 2. Schema validated against strict schema rules    │
│    └─ Error if invalid                             │
└─────────────────┬──────────────────────────────────┘
                  │
┌─────────────────▼──────────────────────────────────┐
│ 3. Schema sent to LLM as response format           │
│    └─ Model constrained to match schema            │
└─────────────────┬──────────────────────────────────┘
                  │
┌─────────────────▼──────────────────────────────────┐
│ 4. Agent performs task normally                    │
│    └─ Reasoning, tool use, etc.                    │
└─────────────────┬──────────────────────────────────┘
                  │
┌─────────────────▼──────────────────────────────────┐
│ 5. Final response extracted and validated          │
│    └─ Guaranteed to match schema                   │
└─────────────────┬──────────────────────────────────┘
                  │
┌─────────────────▼──────────────────────────────────┐
│ 6. JSON written to stdout                          │
└────────────────────────────────────────────────────┘
```

### Combination with `-o` Flag

```bash
# Write structured output to file
codex exec --output-schema schema.json -o output.json "analyze"

# Only JSON in output.json, no other text
cat output.json
{"summary":"...","issues_found":3}
```

---

## Session Resume

### Overview

Exec mode supports **resuming** previous sessions to continue conversations.

### Resume Commands

```bash
# Resume specific session
codex exec resume SESSION_ID "continue task"

# Resume most recent session
codex exec resume --last "follow up question"
```

### Session IDs

**Where to find session IDs**:
1. From first event: `{"type":"thread.started","thread_id":"SESSION_ID"}`
2. From `~/.codex/sessions/` directory
3. From previous exec output

### Implementation

**Location**: `exec/tests/suite/resume.rs` (tests), `exec/src/lib.rs` (implementation)

**Flow**:
```rust
// Simplified representation
pub async fn exec_resume(
    session_id: &str,
    prompt: &str,
    config: &Config,
) -> Result<()> {
    // 1. Load conversation manager
    let mut manager = ConversationManager::load_from_session(session_id)?;

    // 2. Verify session exists
    if !manager.has_conversation(session_id) {
        return Err("Session not found");
    }

    // 3. Resume conversation
    let conversation = manager.get_conversation(session_id)?;

    // 4. Send new prompt
    let events = conversation.send_message(prompt).await?;

    // 5. Process events (same as new session)
    process_events(events, &mut event_processor)?;

    Ok(())
}
```

### Preserved State

When resuming, the following is preserved:
- ✅ **Conversation history**: All previous turns
- ✅ **Context**: Files read, commands run
- ✅ **Configuration**: Original model, settings
- ❌ **Output mode**: Must specify again (--json, --output-schema)
- ❌ **Config overrides**: Must specify again

**Example**:
```bash
# First run
codex exec --model gpt-5 --json "review code"
# Output includes: "thread_id":"abc123"

# Resume (must specify flags again)
codex exec --model gpt-5 --json resume abc123 "fix the issues found"
```

---

## Implementation Details

### Main Entry Point

**Location**: `exec/src/main.rs` (1.3KB)

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // Parse CLI arguments
    let cli = Cli::parse();

    // Delegate to library
    codex_exec::run(cli).await
}
```

### Core Library

**Location**: `exec/src/lib.rs` (15KB)

**Main Function** (simplified):
```rust
pub async fn run(cli: Cli) -> Result<()> {
    // 1. Load configuration
    let config = load_config_with_overrides(cli.config_overrides)?;

    // 2. Determine output mode
    let output_mode = if cli.json {
        OutputMode::Json
    } else {
        OutputMode::Human
    };

    // 3. Create event processor
    let mut processor: Box<dyn EventProcessor> = match output_mode {
        OutputMode::Human => Box::new(HumanOutputProcessor::new()),
        OutputMode::Json => Box::new(JsonlOutputProcessor::new()),
    };

    // 4. Initialize Codex core
    let codex = CodexCore::new(config).await?;

    // 5. Start or resume session
    let events = match cli.command {
        Some(Command::Resume { session_id, last, prompt }) => {
            let id = resolve_session_id(session_id, last)?;
            codex.resume_and_send(id, prompt).await?
        }
        None => {
            codex.start_and_send(cli.prompt, cli.images).await?
        }
    };

    // 6. Process event stream
    for event in events {
        processor.process_event(event)?;
    }

    // 7. Finalize output
    processor.finalize()?;

    Ok(())
}
```

### Event Processing

**Trait Definition** (`event_processor.rs`):
```rust
pub trait EventProcessor {
    /// Process a single event from the stream
    fn process_event(&mut self, event: ThreadEvent) -> Result<()>;

    /// Called when stream completes
    fn finalize(&mut self) -> Result<()>;
}
```

---

## Advanced Usage Patterns

### 1. CI/CD Pipeline

```yaml
# .github/workflows/codex.yml
name: Codex Code Review
on: [pull_request]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Run Codex Review
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          codex exec \
            --sandbox workspace-write \
            -a never \
            "Review this PR for security issues" > review.txt

      - name: Post Review
        run: gh pr comment -F review.txt
```

### 2. Batch Processing

```bash
#!/bin/bash
# Process multiple files
for file in src/**/*.ts; do
  echo "Analyzing $file"
  codex exec "Review $file for TypeScript best practices" -o "${file}.review.txt"
done
```

### 3. JSON Processing

```bash
# Extract structured data
codex exec --json "analyze codebase" | \
  jq '.[] | select(.type == "item.completed") | select(.item.type == "agent_message") | .item.text'
```

### 4. Data Extraction

```bash
# Extract project metadata
codex exec --output-schema schema.json "extract project info" | \
  jq '.project_name'
```

---

## Performance Considerations

### Token Usage

Exec mode has the same token costs as interactive mode:
- Initial prompt
- System prompts
- Tools available
- Conversation history

**Optimization tips**:
- Use `--skip-git-repo-check` to avoid git operations
- Limit scope with sandbox modes
- Use structured output to reduce parsing overhead

### Output Buffering

**Human mode**:
- stderr: Line-buffered (real-time logging)
- stdout: Buffered until finalization (clean output)

**JSON mode**:
- stdout: Line-buffered with explicit flush (real-time streaming)

---

## Testing

### Test Suite

**Location**: `exec/tests/suite/`

**Coverage**:
- Basic execution
- Resume functionality
- JSON output format
- Structured output validation
- Error handling

**Run tests**:
```bash
cd codex-rs/exec
cargo test
```

---

## Comparison with Interactive Mode

| Aspect | Interactive Mode | Exec Mode |
|--------|------------------|-----------|
| **Binary** | `codex` | `codex exec` |
| **Source** | `codex-rs/tui/` | `codex-rs/exec/` |
| **UI** | Rich TUI (ratatui) | Text/JSON output |
| **Input** | Live keyboard | CLI argument |
| **Approvals** | Interactive prompts | Policy-driven |
| **Resume** | Picker UI | CLI command |
| **Output** | Formatted display | stdout stream |
| **Logs** | Log file | stderr |
| **Use Case** | Development | Automation |

---

## Related Documentation

- **Official**: `/context/codex/docs/exec.md` - User guide for exec mode
- **Custom**: [10-implementation.md](./10-implementation.md) - Core implementation patterns
- **Custom**: [09-state-management.md](./09-state-management.md) - Session persistence
- **Custom**: [06-tool-system.md](./06-tool-system.md) - Tool execution

---

## Source Code Reference

**Module**: `codex-rs/exec/`

| File | Size | Purpose |
|------|------|---------|
| `main.rs` | 1.3KB | Entry point |
| `lib.rs` | 15KB | Core orchestration |
| `cli.rs` | 3.8KB | CLI parsing |
| `exec_events.rs` | 7.4KB | Event types |
| `event_processor.rs` | 1.3KB | Processor trait |
| `event_processor_with_human_output.rs` | 22KB | Human-readable output |
| `event_processor_with_jsonl_output.rs` | 16KB | JSON Lines output |

**Total**: ~67KB of exec mode implementation

---

**Last Updated**: October 25, 2025
**Source**: Verified against `codex-rs/exec/` source code
**Note**: Code examples are simplified for clarity. See actual source for complete implementation.
