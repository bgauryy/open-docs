# Codex CLI - User Prompt Processing

## Table of Contents
- [Prompt Lifecycle](#prompt-lifecycle)
- [Three-Tier Prompt System](#three-tier-prompt-system)
- [Prompt Assembly](#prompt-assembly)
- [Context Injection](#context-injection)
- [Request Flow](#request-flow)

---

## Prompt Lifecycle

### Complete Processing Pipeline

```
User Input: "fix lint errors"
      │
      ▼
┌─────────────────────────────────────┐
│  1. CLI Entry & Parsing             │
│     cli/src/main.rs                 │
│     • Parse arguments               │
│     • Extract user message          │
│     • Identify mode (TUI/Exec)      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  2. Session Initialization          │
│     core/src/codex.rs               │
│     • Load configuration            │
│     • Initialize ModelClient        │
│     • Setup ToolRouter              │
│     • Initialize state              │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  3. Prompt Assembly                 │
│     core/src/client_common.rs       │
│     ┌─────────────────────────────┐ │
│     │ System Prompt               │ │
│     │  • Base: prompt.md          │ │
│     │  • Task-specific variants   │ │
│     └─────────────────────────────┘ │
│     ┌─────────────────────────────┐ │
│     │ User Instructions           │ │
│     │  • AGENTS.md chain          │ │
│     │  • Custom instructions      │ │
│     └─────────────────────────────┘ │
│     ┌─────────────────────────────┐ │
│     │ Environment Context         │ │
│     │  • OS/shell info            │ │
│     │  • Git status               │ │
│     │  • Working directory        │ │
│     └─────────────────────────────┘ │
│     ┌─────────────────────────────┐ │
│     │ Conversation History        │ │
│     │  • Previous turns           │ │
│     │  • Tool results             │ │
│     └─────────────────────────────┘ │
│     ┌─────────────────────────────┐ │
│     │ Current User Message        │ │
│     │  • Text input               │ │
│     │  • Attached files           │ │
│     │  • Images (if any)          │ │
│     └─────────────────────────────┘ │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  4. Tool Specification              │
│     core/src/tools/spec.rs          │
│     • Include available tools       │
│     • JSON schemas for each         │
│     • Model-specific filtering      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  5. API Request                     │
│     core/src/client.rs              │
│     • Format for wire protocol      │
│     • Add auth headers              │
│     • Stream to LLM endpoint        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  6. Response Processing             │
│     • Parse SSE events              │
│     • Extract deltas                │
│     • Detect function calls         │
│     • Handle tool execution         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  7. Iteration                       │
│     • Send tool results back        │
│     • Continue until complete       │
└─────────────────────────────────────┘
```

---

## Three-Tier Prompt System

### Tier 1: Base System Prompts

**Location**: `codex-rs/core/*.md`

#### Primary Prompts

1. **`prompt.md`** (11KB, 311 lines)
   - Main coding agent instructions
   - Defines personality and communication style
   - AGENTS.md specification
   - Planning guidelines
   - Task execution philosophy
   - Validation and testing approach
   - Final answer formatting rules

2. **`review_prompt.md`** (2.4KB)
   - Code review mode guidelines
   - Finding severity levels (P0-P3)
   - Output format requirements
   - What constitutes a bug

3. **`gpt_5_codex_prompt.md`** (3.7KB)
   - GPT-5 specific variations
   - Editing constraints
   - Plan tool usage
   - Sandboxing awareness

#### Key Sections in prompt.md

```markdown
# How you work

## Personality
Your default personality and tone is concise, direct, and friendly.

## Responsiveness
Before making tool calls, send a brief preamble...

## Planning
You have access to an `update_plan` tool...

## Task execution
You are a coding agent. Please keep going until the query is 
completely resolved...

## Sandbox and approvals
The Codex CLI harness supports several different sandboxing...

## Validating your work
If the codebase has tests or the ability to build or run...

## Presenting your work
Your final message should read naturally, like an update from 
a concise teammate...
```

**Loading**: Hardcoded in Rust, selected based on task type:

```rust
// core/src/client_common.rs
pub fn get_system_prompt(task_kind: TaskKind) -> &'static str {
    match task_kind {
        TaskKind::Regular => include_str!("../prompt.md"),
        TaskKind::Review => include_str!("../review_prompt.md"),
        TaskKind::Compact => include_str!("../prompt.md"), // + compaction
    }
}
```

### Tier 2: Project-Level Instructions

**Location**: Discovered dynamically from filesystem

#### AGENTS.md Discovery

**Implementation**: `core/src/project_doc.rs`

**Search Algorithm**:
1. Start from current working directory (cwd)
2. Walk up to git repository root (detect via `.git`)
3. Collect `AGENTS.md` files from root → cwd
4. Concatenate in order (root first, cwd last)

```rust
pub async fn get_user_instructions(config: &Config) -> Option<String> {
    // Discovers AGENTS.md files
    let paths = discover_project_doc_paths(config)?;
    
    // Loads and concatenates
    let mut parts: Vec<String> = Vec::new();
    for path in paths {
        let text = tokio::fs::read_to_string(&path).await?;
        parts.push(text);
    }
    
    // Merge with config instructions
    let project_doc = parts.join("\n\n");
    let instructions = format!(
        "{}\n\n--- project-doc ---\n\n{}",
        config.user_instructions?,
        project_doc
    );
}
```

#### Priority Order

1. **`AGENTS.override.md`** - Local override (not in git)
2. **`AGENTS.md`** - Standard project documentation
3. **Fallback filenames** - Configurable via `project_doc_fallback_filenames`

#### Scope Rules

From `prompt.md`:
```markdown
## AGENTS.md spec
- The scope of an AGENTS.md file is the entire directory tree rooted 
  at the folder that contains it.
- For every file you touch in the final patch, you must obey 
  instructions in any AGENTS.md file whose scope includes that file.
- More-deeply-nested AGENTS.md files take precedence in the case of 
  conflicting instructions.
- Direct system/developer/user instructions take precedence over 
  AGENTS.md instructions.
```

#### Example AGENTS.md

```markdown
# Project: MyApp

## Code Style
- Use 2-space indentation
- Prefer functional components in React
- Always use TypeScript strict mode

## Testing
- Write tests for all public APIs
- Use Jest for unit tests
- Place tests next to source files

## Build
- Run `npm run build` to compile
- Output goes to `dist/`
```

**Size Limit**: Configurable via `project_doc_max_bytes` (default: reasonable limit)

### Tier 3: Custom User Prompts

**Location**: `~/.codex/prompts/*.md`

**Implementation**: `core/src/custom_prompts.rs`

#### Discovery

```rust
pub async fn discover_prompts_in(dir: &Path) -> Vec<CustomPrompt> {
    // Scan directory for .md files
    let mut entries = fs::read_dir(dir).await?;
    while let Some(entry) = entries.next_entry().await? {
        if entry.path().extension() == Some("md") {
            // Parse frontmatter
            let (description, argument_hint, body) = parse_frontmatter(&content);
            prompts.push(CustomPrompt { name, path, content: body, ... });
        }
    }
    prompts.sort_by(|a, b| a.name.cmp(&b.name));
    prompts
}
```

#### Frontmatter Format

```markdown
---
description: "Quick performance review"
argument-hint: "[file_path]"
---

Review this file for performance issues:
- O(n²) algorithms
- Unnecessary allocations
- Missing caching

File: $1
```

**Usage**: `/perf src/app.rs`

**Variable Substitution**:
- `$1`, `$2`, etc. - Positional arguments
- `$ARGUMENTS` - All arguments concatenated

---

## Prompt Assembly

### Building the Final Prompt

**Location**: `core/src/codex.rs`, method: `build_prompt()`

```rust
async fn build_prompt(
    &self,
    user_input: &UserInput,
    history: &ConversationHistory,
) -> Result<Prompt> {
    let mut messages = Vec::new();
    
    // 1. System message
    let system_prompt = get_system_prompt(self.task_kind);
    let user_instructions = get_user_instructions(&self.config).await;
    let combined_system = format!(
        "{}\n\n{}",
        system_prompt,
        user_instructions.unwrap_or_default()
    );
    messages.push(Message::System { content: combined_system });
    
    // 2. Environment context (developer message)
    let env_context = build_environment_context(&self.config).await;
    messages.push(Message::Developer { content: env_context });
    
    // 3. Conversation history
    for turn in history.turns() {
        messages.extend(turn.messages());
    }
    
    // 4. Current user message
    messages.push(Message::User {
        content: user_input.text.clone(),
        attachments: user_input.files.clone(),
    });
    
    Ok(Prompt { messages, tools: self.tool_specs.clone() })
}
```

---

## Context Injection

### Environment Context

**Generated by**: `core/src/environment_context.rs`

```rust
pub struct EnvironmentContext {
    pub os_info: String,           // "macOS 14.0"
    pub shell: String,             // "/bin/zsh"
    pub cwd: PathBuf,              // Current directory
    pub git_info: Option<GitInfo>, // Branch, status
    pub approval_mode: ApprovalMode,
    pub sandbox_mode: SandboxMode,
    pub network_access: NetworkAccess,
}

fn format_context(ctx: &EnvironmentContext) -> String {
    format!(
        r#"
# Environment

- **OS**: {os}
- **Shell**: {shell}
- **Working Directory**: {cwd}
- **Git Branch**: {branch}
- **Approval Mode**: {approval}
- **Sandbox**: {sandbox}
- **Network**: {network}

{git_status}
"#,
        os = ctx.os_info,
        shell = ctx.shell,
        cwd = ctx.cwd.display(),
        branch = ctx.git_info.branch,
        approval = ctx.approval_mode,
        sandbox = ctx.sandbox_mode,
        network = ctx.network_access,
        git_status = format_git_status(ctx.git_info),
    )
}
```

### Git Context

**Generated by**: `core/src/git_info.rs`

Automatically includes:
- Current branch name
- Uncommitted changes count
- Untracked files count
- Remote tracking branch
- Recent commit info

```rust
pub struct GitInfo {
    pub root: PathBuf,
    pub branch: String,
    pub dirty: bool,
    pub untracked_count: usize,
    pub modified_count: usize,
    pub remote: Option<String>,
}
```

### Attached Files

Users can attach files/images:

```rust
pub struct UserInput {
    pub text: String,
    pub files: Vec<AttachedFile>,
}

pub struct AttachedFile {
    pub path: PathBuf,
    pub content_type: String,
    pub data: Vec<u8>,
}
```

Files are included as separate content items in the message.

---

## Request Flow

### From Prompt to API

**Step 1: Format for Wire Protocol**

```rust
// core/src/client.rs
impl ModelClient {
    pub async fn stream(&self, prompt: &Prompt) -> Result<ResponseStream> {
        match self.provider.wire_api {
            WireApi::Responses => self.stream_responses(prompt).await,
            WireApi::Chat => self.stream_chat_completions(prompt).await,
        }
    }
}
```

**Step 2: Build Request Body**

For Responses API:
```rust
let request = ResponsesApiRequest {
    model: self.config.model_family.model_name.clone(),
    messages: convert_messages(&prompt.messages),
    tools: Some(prompt.tools.clone()),
    stream: true,
    reasoning_effort: self.effort,
    reasoning_summary: self.summary,
};
```

**Step 3: Send Streaming Request**

```rust
let response = self.client
    .post(&url)
    .header("Authorization", format!("Bearer {}", api_key))
    .header("Content-Type", "application/json")
    .json(&request)
    .send()
    .await?;

let stream = response
    .bytes_stream()
    .eventsource()
    .map(parse_sse_event);
```

### Response Processing

**Event Types**:

```rust
pub enum ResponseEvent {
    ResponseCreated { id: String },
    Delta { delta: String },
    ReasoningDelta { summary: String },
    FunctionCall { name: String, args: Value },
    Completed { id: String },
    Error { code: String, message: String },
}
```

**Processing Loop**:

```rust
while let Some(event) = stream.next().await {
    match event? {
        ResponseEvent::Delta { delta } => {
            // Stream text to UI
            tx_event.send(Event::AgentMessageDelta { delta }).await?;
        }
        ResponseEvent::FunctionCall { name, args } => {
            // Execute tool
            let result = tool_router.route(&name, args).await?;
            // Send result back to model
            self.send_tool_result(result).await?;
        }
        ResponseEvent::Completed { .. } => break,
    }
}
```

---

## Optimization Strategies

### History Compaction

When approaching token limits:

```rust
// core/src/codex/compact.rs
pub async fn build_compacted_history(
    original_history: &[Message],
    model_client: &ModelClient,
) -> Result<Vec<Message>> {
    // 1. Keep system prompt
    // 2. Summarize old turns
    // 3. Keep recent turns verbatim
    // 4. Always keep tool results
}
```

### Token Management

```rust
// core/src/token_data.rs
pub struct TokenUsage {
    pub input_tokens: i64,
    pub output_tokens: i64,
    pub cached_tokens: i64,
}

impl ModelClient {
    pub fn get_model_context_window(&self) -> Option<i64> {
        let pct = self.config.effective_context_window_percent;
        self.config.model_context_window.map(|w| w * pct / 100)
    }
}
```

---

## Related Documentation

- [04-llm-integration.md](./04-llm-integration.md) - LLM API details
- [05-system-prompts.md](./05-system-prompts.md) - Prompt customization
- [09-state-management.md](./09-state-management.md) - History management

