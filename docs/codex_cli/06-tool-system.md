# Codex CLI - Tool System

## Table of Contents
- [Tool Architecture](#tool-architecture)
- [Available Tools](#available-tools)
- [Tool Specification Format](#tool-specification-format)
- [Tool Execution Flow](#tool-execution-flow)
- [Adding New Tools](#adding-new-tools)

---

## Tool Architecture

### Component Overview

```
┌──────────────────────────────────────────────────────────┐
│                   Tool System Layers                     │
└──────────────────────────────────────────────────────────┘

Layer 1: Router
┌──────────────────┐
│   ToolRouter     │  Receives function calls from LLM
│   • Validates    │  Routes to appropriate handler
│   • Routes       │
└────────┬─────────┘
         │
Layer 2: Registry
┌────────▼─────────┐
│  ToolRegistry    │  Maps tool names to implementations
│  • shell         │
│  • apply_patch   │
│  • read_file     │
│  • ...           │
└────────┬─────────┘
         │
Layer 3: Orchestration
┌────────▼─────────┐
│ ToolOrchestrator │  Manages execution context
│  • Security      │  Handles sandboxing
│  • Approval      │  Tracks state
│  • Parallelism   │
└────────┬─────────┘
         │
Layer 4: Handlers
┌────────▼─────────┐
│   Tool Handler   │  Implements specific tool
│   • shell.rs     │
│   • apply_patch  │
│   • read_file    │
│   • ...          │
└──────────────────┘
```

### Core Components

#### 1. ToolRouter

**Location**: `core/src/tools/router.rs`

```rust
pub struct ToolRouter {
    registry: ToolRegistry,
    orchestrator: ToolOrchestrator,
    context: Arc<ToolContext>,
}

impl ToolRouter {
    pub async fn route(
        &self,
        tool_name: &str,
        arguments: Value,
    ) -> Result<ToolResult> {
        // 1. Validate tool exists
        let handler = self.registry.get(tool_name)?;
        
        // 2. Check security policy
        self.context.check_approval_needed(tool_name, &arguments)?;
        
        // 3. Execute through orchestrator
        self.orchestrator.execute(handler, arguments).await
    }
}
```

#### 2. ToolRegistry

**Location**: `core/src/tools/registry.rs`

```rust
pub struct ToolRegistry {
    handlers: HashMap<String, Box<dyn ToolHandler>>,
}

pub trait ToolHandler: Send + Sync {
    async fn execute(&self, args: Value) -> Result<ToolOutput>;
    fn name(&self) -> &str;
    fn description(&self) -> &str;
}

impl ToolRegistry {
    pub fn register<T: ToolHandler + 'static>(
        &mut self,
        handler: T,
    ) {
        self.handlers.insert(
            handler.name().to_string(),
            Box::new(handler)
        );
    }
}
```

#### 3. ToolOrchestrator

**Location**: `core/src/tools/orchestrator.rs`

```rust
pub struct ToolOrchestrator {
    approval_store: ApprovalStore,
    sandbox_config: SandboxConfig,
    runtime: ToolCallRuntime,
}

impl ToolOrchestrator {
    pub async fn execute(
        &self,
        handler: &dyn ToolHandler,
        args: Value,
    ) -> Result<ToolOutput> {
        // 1. Request approval if needed
        if self.needs_approval(handler.name())? {
            self.approval_store.request_approval(handler.name()).await?;
        }
        
        // 2. Apply sandboxing
        let sandboxed = self.sandbox_config.wrap_execution(
            || handler.execute(args)
        );
        
        // 3. Execute with timeout
        timeout(Duration::from_secs(30), sandboxed).await?
    }
}
```

#### 4. Tool Context

**Location**: `core/src/tools/context.rs`

```rust
pub struct ToolContext {
    pub cwd: PathBuf,
    pub config: Arc<Config>,
    pub diff_tracker: SharedTurnDiffTracker,
    pub env_vars: HashMap<String, String>,
}

impl ToolContext {
    pub fn check_approval_needed(
        &self,
        tool_name: &str,
        args: &Value,
    ) -> Result<bool> {
        match self.config.approval_mode {
            ApprovalMode::Untrusted => {
                !is_safe_command(tool_name, args)
            }
            ApprovalMode::OnRequest => {
                args.get("with_escalated_permissions")
                    .and_then(|v| v.as_bool())
                    .unwrap_or(false)
            }
            ApprovalMode::OnFailure => false,
            ApprovalMode::Never => false,
        }
    }
}
```

---

## Available Tools

### Tool Inventory

| Tool | File | Purpose | Key Features |
|------|------|---------|--------------|
| `shell` | shell.rs | Execute commands | PTY, streaming, sandboxed |
| `apply_patch` | apply_patch.rs | Modify files | Unified diff, freeform/JSON |
| `read_file` | read_file.rs | Read file content | Chunking, encoding detection |
| `list_dir` | list_dir.rs | List directory | Filtering, sorting, metadata |
| `grep_files` | grep_files.rs | Search files | Regex, ripgrep integration |
| `update_plan` | plan.rs | Manage task plans | Status tracking, UI updates |
| `mcp_call_tool` | mcp.rs | Call MCP tools | External tool integration |
| `mcp_read_resource` | mcp_resource.rs | Read MCP resources | External data access |
| `view_image` | view_image.rs | View images | Base64 encoding, metadata |
| `exec_command` | unified_exec.rs | Unified execution | PTY sessions, stdin control |
| `write_stdin` | unified_exec.rs | Write to session | Interactive command control |

### Tool Details

#### 1. shell

**Purpose**: Execute shell commands in PTY

**Schema**:
```json
{
  "name": "shell",
  "description": "Execute a shell command",
  "parameters": {
    "type": "object",
    "properties": {
      "command": {
        "type": "array",
        "items": {"type": "string"},
        "description": "Command and arguments"
      },
      "workdir": {
        "type": "string",
        "description": "Working directory"
      }
    },
    "required": ["command"]
  }
}
```

**Implementation**: `core/src/tools/handlers/shell.rs`

```rust
pub async fn execute_shell(
    command: &[String],
    workdir: Option<PathBuf>,
    ctx: &ToolContext,
) -> Result<ToolOutput> {
    // 1. Validate command safety
    if is_dangerous_command(command) {
        return Err("Command blocked for safety");
    }
    
    // 2. Apply sandboxing
    let sandboxed_cmd = apply_sandbox(command, &ctx.config)?;
    
    // 3. Execute in PTY
    let output = exec_in_pty(
        &sandboxed_cmd,
        workdir.unwrap_or(ctx.cwd.clone())
    ).await?;
    
    // 4. Format output
    Ok(ToolOutput {
        stdout: output.stdout,
        stderr: output.stderr,
        exit_code: output.exit_code,
    })
}
```

#### 2. apply_patch

**Purpose**: Apply code changes using unified diff format

**Schemas**: Two variants

**Freeform** (string argument):
```json
{
  "name": "apply_patch",
  "description": "Apply a unified diff patch",
  "parameters": {
    "type": "object",
    "properties": {
      "patch": {
        "type": "string",
        "description": "Unified diff format patch"
      }
    }
  }
}
```

**JSON** (structured):
```json
{
  "name": "apply_patch",
  "parameters": {
    "type": "object",
    "properties": {
      "file_path": {"type": "string"},
      "original_content": {"type": "string"},
      "updated_content": {"type": "string"}
    }
  }
}
```

**Implementation**: `core/src/tools/handlers/apply_patch.rs`

```rust
pub async fn apply_patch(
    patch_text: &str,
    ctx: &ToolContext,
) -> Result<Vec<ApplyPatchAction>> {
    // 1. Parse patch
    let patch = parse_unified_diff(patch_text)?;
    
    // 2. Validate files are in workspace
    for file in &patch.files {
        ctx.validate_path(&file.path)?;
    }
    
    // 3. Apply changes
    let mut actions = Vec::new();
    for hunk in patch.hunks {
        let action = apply_hunk(&hunk, ctx).await?;
        actions.push(action);
        
        // Track in diff tracker
        ctx.diff_tracker.record_change(&hunk);
    }
    
    Ok(actions)
}
```

#### 3. read_file

**Purpose**: Read file content with smart chunking

**Schema**:
```json
{
  "name": "read_file",
  "parameters": {
    "type": "object",
    "properties": {
      "path": {"type": "string"},
      "start_line": {"type": "number"},
      "end_line": {"type": "number"},
      "max_lines": {"type": "number"}
    },
    "required": ["path"]
  }
}
```

**Implementation**: `core/src/tools/handlers/read_file.rs`

```rust
pub async fn read_file(
    path: &Path,
    start_line: Option<usize>,
    end_line: Option<usize>,
    max_lines: Option<usize>,
    ctx: &ToolContext,
) -> Result<String> {
    // 1. Validate path
    let full_path = ctx.cwd.join(path);
    ctx.validate_path(&full_path)?;
    
    // 2. Read file
    let content = tokio::fs::read_to_string(&full_path).await?;
    
    // 3. Apply line range
    let lines: Vec<&str> = content.lines().collect();
    let start = start_line.unwrap_or(1).saturating_sub(1);
    let end = end_line.unwrap_or(lines.len()).min(lines.len());
    let max = max_lines.unwrap_or(250).min(250); // Cap at 250 lines
    
    let selected = &lines[start..end.min(start + max)];
    
    // 4. Format with line numbers
    let numbered: Vec<String> = selected
        .iter()
        .enumerate()
        .map(|(i, line)| format!("{:4}| {}", start + i + 1, line))
        .collect();
    
    Ok(numbered.join("\n"))
}
```

#### 4. list_dir

**Purpose**: List directory contents with filtering

**Schema**:
```json
{
  "name": "list_dir",
  "parameters": {
    "type": "object",
    "properties": {
      "path": {"type": "string"},
      "recursive": {"type": "boolean"},
      "max_depth": {"type": "number"},
      "pattern": {"type": "string"}
    },
    "required": ["path"]
  }
}
```

#### 5. grep_files

**Purpose**: Search files with regex patterns

**Schema**:
```json
{
  "name": "grep_files",
  "parameters": {
    "type": "object",
    "properties": {
      "pattern": {"type": "string"},
      "path": {"type": "string"},
      "file_pattern": {"type": "string"},
      "case_sensitive": {"type": "boolean"},
      "max_results": {"type": "number"}
    },
    "required": ["pattern", "path"]
  }
}
```

**Implementation**: Uses `ripgrep` (rg) for performance

```rust
pub async fn grep_files(
    pattern: &str,
    path: &Path,
    options: GrepOptions,
    ctx: &ToolContext,
) -> Result<Vec<GrepMatch>> {
    let mut cmd = Command::new("rg");
    cmd.arg("--json")
       .arg("--max-count").arg(options.max_results.to_string())
       .arg(pattern)
       .arg(path);
    
    if !options.case_sensitive {
        cmd.arg("-i");
    }
    
    if let Some(file_pat) = options.file_pattern {
        cmd.arg("--glob").arg(file_pat);
    }
    
    let output = cmd.output().await?;
    parse_rg_json(&output.stdout)
}
```

#### 6. update_plan

**Purpose**: Create and update task plans

**Schema**:
```json
{
  "name": "update_plan",
  "parameters": {
    "type": "object",
    "properties": {
      "steps": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "id": {"type": "string"},
            "description": {"type": "string"},
            "status": {
              "type": "string",
              "enum": ["pending", "in_progress", "completed"]
            }
          }
        }
      },
      "explanation": {"type": "string"}
    },
    "required": ["steps"]
  }
}
```

**Implementation**: `core/src/tools/handlers/plan.rs`

```rust
pub async fn update_plan(
    steps: Vec<PlanStep>,
    explanation: Option<String>,
    ctx: &ToolContext,
) -> Result<()> {
    // 1. Validate plan
    validate_plan_structure(&steps)?;
    
    // 2. Update state
    ctx.plan_state.update(steps.clone()).await?;
    
    // 3. Emit event for UI
    ctx.tx_event.send(Event::PlanUpdated {
        steps,
        explanation,
    }).await?;
    
    Ok(())
}
```

---

## Tool Specification Format

### JSON Schema Structure

**Location**: `core/src/tools/spec.rs`

```rust
pub enum JsonSchema {
    String { description: Option<String> },
    Number { description: Option<String> },
    Boolean { description: Option<String> },
    Array {
        items: Box<JsonSchema>,
        description: Option<String>,
    },
    Object {
        properties: BTreeMap<String, JsonSchema>,
        required: Option<Vec<String>>,
        additional_properties: Option<AdditionalProperties>,
    },
}

pub struct ToolSpec {
    pub name: String,
    pub description: String,
    pub parameters: JsonSchema,
    pub strict: bool,  // Structured outputs mode
}
```

### Creating Tool Specifications

```rust
fn create_read_file_tool() -> ToolSpec {
    let mut properties = BTreeMap::new();
    
    properties.insert(
        "path".to_string(),
        JsonSchema::String {
            description: Some("Path to file to read".to_string()),
        },
    );
    
    properties.insert(
        "start_line".to_string(),
        JsonSchema::Number {
            description: Some("Starting line number (1-indexed)".to_string()),
        },
    );
    
    properties.insert(
        "end_line".to_string(),
        JsonSchema::Number {
            description: Some("Ending line number (inclusive)".to_string()),
        },
    );
    
    ToolSpec {
        name: "read_file".to_string(),
        description: "Read contents of a file".to_string(),
        parameters: JsonSchema::Object {
            properties,
            required: Some(vec!["path".to_string()]),
            additional_properties: Some(false.into()),
        },
        strict: false,
    }
}
```

### Tool Configuration

**Model-Specific Filtering**:

```rust
// core/src/tools/spec.rs
pub struct ToolsConfig {
    pub shell_type: ConfigShellToolType,
    pub apply_patch_tool_type: Option<ApplyPatchToolType>,
    pub web_search_request: bool,
    pub include_view_image_tool: bool,
}

impl ToolsConfig {
    pub fn new(params: &ToolsConfigParams) -> Self {
        let model_family = params.model_family;
        let features = params.features;
        
        Self {
            shell_type: if features.enabled(Feature::StreamableShell) {
                ConfigShellToolType::Streamable
            } else if model_family.uses_local_shell_tool {
                ConfigShellToolType::Local
            } else {
                ConfigShellToolType::Default
            },
            apply_patch_tool_type: model_family.apply_patch_tool_type,
            web_search_request: features.enabled(Feature::WebSearchRequest),
            include_view_image_tool: features.enabled(Feature::ViewImageTool),
        }
    }
}
```

---

## Tool Execution Flow

### Complete Execution Pipeline

```
1. LLM Returns Function Call
   {
     "name": "shell",
     "arguments": "{\"command\":[\"ls\",\"-la\"]}"
   }
   │
   ▼
2. Parse Arguments
   ├─ Validate JSON
   └─ Extract parameters
   │
   ▼
3. Security Checks
   ├─ Command safety validation
   ├─ Path validation
   └─ Approval policy check
   │
   ▼
4. Request Approval (if needed)
   ├─ Emit ApprovalRequestEvent
   ├─ Wait for user response
   └─ Record decision
   │
   ▼
5. Apply Sandboxing
   ├─ Seatbelt (macOS)
   ├─ Landlock (Linux)
   └─ Directory restrictions
   │
   ▼
6. Execute Tool
   ├─ Call handler
   ├─ Stream output (if applicable)
   └─ Capture result
   │
   ▼
7. Format Output
   ├─ Truncate if needed (10KB max)
   ├─ Add metadata (exit code, duration)
   └─ Format for model
   │
   ▼
8. Send to LLM
   {
     "role": "tool",
     "content": "...",
     "tool_call_id": "..."
   }
   │
   ▼
9. Continue Iteration
```

### Error Handling

```rust
pub enum ToolError {
    NotFound(String),
    InvalidArguments(String),
    ApprovalDenied,
    SandboxViolation(String),
    ExecutionFailed {
        exit_code: i32,
        stderr: String,
    },
    Timeout,
}

impl ToolRouter {
    async fn handle_error(&self, error: ToolError) -> ToolOutput {
        match error {
            ToolError::ApprovalDenied => {
                ToolOutput::error("User denied approval")
            }
            ToolError::SandboxViolation(msg) => {
                ToolOutput::error(format!("Sandbox violation: {}", msg))
            }
            ToolError::ExecutionFailed { exit_code, stderr } => {
                ToolOutput {
                    success: false,
                    exit_code: Some(exit_code),
                    output: stderr,
                }
            }
            _ => ToolOutput::error(error.to_string()),
        }
    }
}
```

---

## Adding New Tools

### Step-by-Step Guide

#### 1. Create Handler File

**Location**: `core/src/tools/handlers/my_tool.rs`

```rust
use crate::tools::context::ToolContext;
use crate::error::Result;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
pub struct MyToolArgs {
    pub input: String,
    pub option: Option<bool>,
}

#[derive(Serialize)]
pub struct MyToolOutput {
    pub result: String,
    pub metadata: serde_json::Value,
}

pub async fn execute_my_tool(
    args: MyToolArgs,
    ctx: &ToolContext,
) -> Result<MyToolOutput> {
    // Implement tool logic
    Ok(MyToolOutput {
        result: format!("Processed: {}", args.input),
        metadata: serde_json::json!({"success": true}),
    })
}
```

#### 2. Create Tool Specification

**Location**: `core/src/tools/spec.rs`

```rust
fn create_my_tool_spec() -> ToolSpec {
    let mut properties = BTreeMap::new();
    
    properties.insert(
        "input".to_string(),
        JsonSchema::String {
            description: Some("Input text to process".to_string()),
        },
    );
    
    properties.insert(
        "option".to_string(),
        JsonSchema::Boolean {
            description: Some("Optional flag".to_string()),
        },
    );
    
    ToolSpec {
        name: "my_tool".to_string(),
        description: "Processes input text".to_string(),
        parameters: JsonSchema::Object {
            properties,
            required: Some(vec!["input".to_string()]),
            additional_properties: Some(false.into()),
        },
        strict: false,
    }
}
```

#### 3. Register Tool

**Location**: `core/src/tools/registry.rs`

```rust
pub fn build_tool_registry(config: &ToolsConfig) -> ToolRegistry {
    let mut registry = ToolRegistry::new();
    
    // ... existing tools ...
    
    // Register new tool
    registry.register(MyToolHandler::new());
    
    registry
}

struct MyToolHandler {
    spec: ToolSpec,
}

impl MyToolHandler {
    fn new() -> Self {
        Self {
            spec: create_my_tool_spec(),
        }
    }
}

#[async_trait]
impl ToolHandler for MyToolHandler {
    async fn execute(&self, args: Value) -> Result<ToolOutput> {
        let parsed: MyToolArgs = serde_json::from_value(args)?;
        let result = execute_my_tool(parsed, &self.context).await?;
        Ok(ToolOutput::success(serde_json::to_value(result)?))
    }
    
    fn name(&self) -> &str {
        &self.spec.name
    }
    
    fn description(&self) -> &str {
        &self.spec.description
    }
}
```

#### 4. Add to Tool List

**Location**: `core/src/tools/spec.rs`, function `create_tools_json_for_responses_api`

```rust
pub fn create_tools_json_for_responses_api(
    config: &ToolsConfig,
) -> Vec<ToolSpec> {
    let mut tools = vec![
        create_shell_tool(),
        create_apply_patch_tool(),
        create_read_file_tool(),
        // ... existing tools ...
        create_my_tool_spec(),  // Add here
    ];
    
    tools
}
```

#### 5. Write Tests

**Location**: `tests/tools/my_tool.test.rs`

```rust
#[tokio::test]
async fn test_my_tool() {
    let ctx = create_test_context();
    let args = MyToolArgs {
        input: "test".to_string(),
        option: Some(true),
    };
    
    let result = execute_my_tool(args, &ctx).await.unwrap();
    assert_eq!(result.result, "Processed: test");
}
```

---

## Related Documentation

- [07-security-sandboxing.md](./07-security-sandboxing.md) - Security model
- [11-tool-implementations.md](./11-tool-implementations.md) - Tool implementation details
- [15-code-reference.md](./15-code-reference.md) - Code reference

