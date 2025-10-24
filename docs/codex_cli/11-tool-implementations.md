# Codex CLI - Tool Implementations

## Table of Contents
- [Shell Tool](#shell-tool)
- [Apply Patch Tool](#apply-patch-tool)
- [File Operations](#file-operations)
- [Plan Tool](#plan-tool)
- [MCP Integration](#mcp-integration)

---

## Shell Tool

**Location**: `core/src/tools/handlers/shell.rs`

### Purpose

Execute shell commands in a PTY (pseudo-terminal) with sandboxing and output streaming.

### Implementation

```rust
pub struct ShellHandler {
    context: Arc<ToolContext>,
}

#[derive(Deserialize)]
pub struct ShellArgs {
    pub command: Vec<String>,
    pub workdir: Option<PathBuf>,
    pub with_escalated_permissions: Option<bool>,
    pub justification: Option<String>,
}

impl ShellHandler {
    pub async fn execute(&self, args: ShellArgs) -> Result<ToolOutput> {
        // 1. Validate command safety
        if is_dangerous_command(&args.command) {
            return Err("Command blocked for safety".into());
        }
        
        // 2. Check approval needed
        if self.needs_approval(&args)? {
            request_approval(&args.command, &args.justification).await?;
        }
        
        // 3. Apply sandboxing
        let sandboxed_cmd = self.apply_sandbox(&args.command)?;
        
        // 4. Resolve working directory
        let workdir = args.workdir
            .unwrap_or_else(|| self.context.cwd.clone());
        
        // 5. Execute in PTY
        let output = self.exec_in_pty(&sandboxed_cmd, &workdir).await?;
        
        // 6. Format output
        Ok(ToolOutput {
            stdout: output.stdout,
            stderr: output.stderr,
            exit_code: output.exit_code,
            duration: output.duration,
        })
    }
    
    fn apply_sandbox(&self, command: &[String]) -> Result<Vec<String>> {
        match self.context.config.sandbox_mode {
            SandboxMode::DangerFullAccess => {
                Ok(command.to_vec())
            }
            _ => {
                // Wrap with platform sandbox
                #[cfg(target_os = "macos")]
                {
                    wrap_with_seatbelt(command, &self.context.cwd)
                }
                
                #[cfg(target_os = "linux")]
                {
                    // Landlock applied at process level
                    Ok(command.to_vec())
                }
            }
        }
    }
    
    async fn exec_in_pty(
        &self,
        command: &[String],
        workdir: &Path,
    ) -> Result<ExecOutput> {
        use codex_exec::exec_command;
        
        let mut cmd = tokio::process::Command::new(&command[0]);
        cmd.args(&command[1..]);
        cmd.current_dir(workdir);
        cmd.stdout(Stdio::piped());
        cmd.stderr(Stdio::piped());
        
        // Set up environment
        cmd.env_clear();
        cmd.envs(self.context.env_vars.clone());
        
        let start = Instant::now();
        let mut child = cmd.spawn()?;
        
        // Stream output
        let stdout = BufReader::new(child.stdout.take().unwrap());
        let stderr = BufReader::new(child.stderr.take().unwrap());
        
        let (stdout_text, stderr_text) = tokio::try_join!(
            read_to_string(stdout),
            read_to_string(stderr),
        )?;
        
        let status = child.wait().await?;
        let duration = start.elapsed();
        
        Ok(ExecOutput {
            stdout: stdout_text,
            stderr: stderr_text,
            exit_code: status.code().unwrap_or(-1),
            duration,
        })
    }
}
```

### Sandboxing (macOS)

```rust
fn wrap_with_seatbelt(
    command: &[String],
    workspace: &Path,
) -> Result<Vec<String>> {
    let policy = generate_seatbelt_policy(workspace)?;
    
    Ok(vec![
        "/usr/bin/sandbox-exec".to_string(),
        "-p".to_string(),
        policy,
        "--".to_string(),
        command[0].clone(),
        // ... rest of command
    ])
}

fn generate_seatbelt_policy(workspace: &Path) -> Result<String> {
    let base_policy = include_str!("../seatbelt_base_policy.sbpl");
    
    let policy = base_policy
        .replace("{{WORKSPACE}}", workspace.to_str().unwrap())
        .replace("{{TMPDIR}}", env::temp_dir().to_str().unwrap());
    
    Ok(policy)
}
```

---

## Apply Patch Tool

**Location**: `core/src/tools/handlers/apply_patch.rs`

### Purpose

Apply code changes using unified diff format or JSON structured edits.

### Two Variants

#### 1. Freeform (Unified Diff)

```rust
#[derive(Deserialize)]
pub struct ApplyPatchFreeformArgs {
    pub patch: String,  // Unified diff format
}

pub async fn apply_patch_freeform(
    args: ApplyPatchFreeformArgs,
    ctx: &ToolContext,
) -> Result<Vec<ApplyPatchAction>> {
    // 1. Parse unified diff
    let patch = parse_unified_diff(&args.patch)?;
    
    // 2. Validate files
    for file in &patch.files {
        ctx.validate_write_path(&file.path)?;
    }
    
    // 3. Apply hunks
    let mut actions = Vec::new();
    for file_patch in patch.files {
        let action = apply_file_patch(&file_patch, ctx).await?;
        actions.push(action);
    }
    
    Ok(actions)
}

fn parse_unified_diff(patch_text: &str) -> Result<Patch> {
    use codex_apply_patch::parse_patch;
    
    let patch = parse_patch(patch_text)?;
    Ok(patch)
}

async fn apply_file_patch(
    file_patch: &FilePatch,
    ctx: &ToolContext,
) -> Result<ApplyPatchAction> {
    let path = ctx.cwd.join(&file_patch.path);
    
    // Read original
    let original = tokio::fs::read_to_string(&path).await?;
    
    // Apply hunks
    let mut lines: Vec<&str> = original.lines().collect();
    for hunk in &file_patch.hunks {
        apply_hunk(&mut lines, hunk)?;
    }
    
    // Write modified
    let modified = lines.join("\n");
    tokio::fs::write(&path, &modified).await?;
    
    // Track change
    ctx.diff_tracker.record_file_modification(
        path.clone(),
        original,
        modified,
    )?;
    
    Ok(ApplyPatchAction {
        path,
        hunks_applied: file_patch.hunks.len(),
        lines_added: count_additions(&file_patch.hunks),
        lines_removed: count_deletions(&file_patch.hunks),
    })
}
```

#### 2. JSON (Structured)

```rust
#[derive(Deserialize)]
pub struct ApplyPatchJsonArgs {
    pub file_path: PathBuf,
    pub original_content: String,
    pub updated_content: String,
}

pub async fn apply_patch_json(
    args: ApplyPatchJsonArgs,
    ctx: &ToolContext,
) -> Result<ApplyPatchAction> {
    let path = ctx.cwd.join(&args.file_path);
    
    // Validate
    ctx.validate_write_path(&path)?;
    
    // Read current content
    let current = tokio::fs::read_to_string(&path).await?;
    
    // Verify original matches
    if current != args.original_content {
        return Err("Original content mismatch - file may have changed".into());
    }
    
    // Write updated content
    tokio::fs::write(&path, &args.updated_content).await?;
    
    // Track change
    ctx.diff_tracker.record_file_modification(
        path.clone(),
        args.original_content,
        args.updated_content.clone(),
    )?;
    
    Ok(ApplyPatchAction {
        path,
        hunks_applied: 1,
        lines_added: count_new_lines(&args.updated_content),
        lines_removed: count_new_lines(&args.original_content),
    })
}
```

### Patch Format Parser

```rust
// Using lark grammar
// Location: core/src/tools/handlers/tool_apply_patch.lark

start: patch_file+

patch_file: file_header hunk+

file_header: "*** Update File:" path
           | "diff --git" path path

hunk: hunk_header hunk_line+

hunk_header: "@@" range range "@@" text?

range: "-" NUMBER ("," NUMBER)?
     | "+" NUMBER ("," NUMBER)?

hunk_line: context_line
         | addition_line
         | deletion_line

context_line: /[^-+@].*/ NEWLINE
addition_line: "+" /.*/ NEWLINE
deletion_line: "-" /.*/ NEWLINE
```

---

## File Operations

### Read File

**Location**: `core/src/tools/handlers/read_file.rs`

```rust
#[derive(Deserialize)]
pub struct ReadFileArgs {
    pub path: PathBuf,
    pub start_line: Option<usize>,
    pub end_line: Option<usize>,
    pub max_lines: Option<usize>,
}

pub async fn read_file(
    args: ReadFileArgs,
    ctx: &ToolContext,
) -> Result<String> {
    let full_path = ctx.cwd.join(&args.path);
    
    // Validate
    ctx.validate_read_path(&full_path)?;
    
    // Read file
    let content = tokio::fs::read_to_string(&full_path).await?;
    
    // Apply line range
    let lines: Vec<&str> = content.lines().collect();
    let total_lines = lines.len();
    
    let start = args.start_line.unwrap_or(1).saturating_sub(1);
    let end = args.end_line.unwrap_or(total_lines).min(total_lines);
    let max = args.max_lines.unwrap_or(250).min(250); // Cap at 250
    
    let actual_end = (start + max).min(end);
    let selected = &lines[start..actual_end];
    
    // Format with line numbers
    let formatted: Vec<String> = selected
        .iter()
        .enumerate()
        .map(|(i, line)| format!("{:>6}| {}", start + i + 1, line))
        .collect();
    
    let mut output = formatted.join("\n");
    
    // Add metadata
    if actual_end < end {
        output.push_str(&format!(
            "\n\n[... {} more lines omitted. Total: {} lines]",
            end - actual_end,
            total_lines
        ));
    }
    
    Ok(output)
}
```

### List Directory

**Location**: `core/src/tools/handlers/list_dir.rs`

```rust
#[derive(Deserialize)]
pub struct ListDirArgs {
    pub path: PathBuf,
    pub recursive: Option<bool>,
    pub max_depth: Option<usize>,
    pub pattern: Option<String>,
    pub sort_by: Option<SortBy>,
}

#[derive(Deserialize)]
pub enum SortBy {
    Name,
    Size,
    Modified,
    Extension,
}

pub async fn list_dir(
    args: ListDirArgs,
    ctx: &ToolContext,
) -> Result<Vec<DirEntry>> {
    let full_path = ctx.cwd.join(&args.path);
    ctx.validate_read_path(&full_path)?;
    
    let mut entries = if args.recursive.unwrap_or(false) {
        list_recursive(&full_path, args.max_depth.unwrap_or(3)).await?
    } else {
        list_single_level(&full_path).await?
    };
    
    // Filter by pattern
    if let Some(pattern) = args.pattern {
        let regex = Regex::new(&pattern)?;
        entries.retain(|e| regex.is_match(&e.name));
    }
    
    // Sort
    match args.sort_by.unwrap_or(SortBy::Name) {
        SortBy::Name => entries.sort_by(|a, b| a.name.cmp(&b.name)),
        SortBy::Size => entries.sort_by_key(|e| e.size),
        SortBy::Modified => entries.sort_by_key(|e| e.modified),
        SortBy::Extension => entries.sort_by(|a, b| {
            a.extension.cmp(&b.extension)
        }),
    }
    
    Ok(entries)
}

async fn list_single_level(path: &Path) -> Result<Vec<DirEntry>> {
    let mut entries = Vec::new();
    let mut dir = tokio::fs::read_dir(path).await?;
    
    while let Some(entry) = dir.next_entry().await? {
        let metadata = entry.metadata().await?;
        let name = entry.file_name().to_string_lossy().to_string();
        
        entries.push(DirEntry {
            name,
            path: entry.path(),
            size: metadata.len(),
            is_dir: metadata.is_dir(),
            is_file: metadata.is_file(),
            modified: metadata.modified().ok(),
            extension: entry.path()
                .extension()
                .and_then(|s| s.to_str())
                .map(String::from),
        });
    }
    
    Ok(entries)
}
```

### Grep Files

**Location**: `core/src/tools/handlers/grep_files.rs`

```rust
#[derive(Deserialize)]
pub struct GrepFilesArgs {
    pub pattern: String,
    pub path: PathBuf,
    pub file_pattern: Option<String>,
    pub case_sensitive: Option<bool>,
    pub max_results: Option<usize>,
}

pub async fn grep_files(
    args: GrepFilesArgs,
    ctx: &ToolContext,
) -> Result<Vec<GrepMatch>> {
    let search_path = ctx.cwd.join(&args.path);
    ctx.validate_read_path(&search_path)?;
    
    // Use ripgrep for performance
    let mut cmd = tokio::process::Command::new("rg");
    
    cmd.arg("--json")  // JSON output for parsing
       .arg("--heading")
       .arg("--line-number")
       .arg("--no-messages");  // Suppress errors
    
    // Case sensitivity
    if !args.case_sensitive.unwrap_or(true) {
        cmd.arg("-i");
    }
    
    // Max results
    if let Some(max) = args.max_results {
        cmd.arg("--max-count").arg(max.to_string());
    }
    
    // File pattern
    if let Some(file_pat) = args.file_pattern {
        cmd.arg("--glob").arg(file_pat);
    }
    
    // Pattern and path
    cmd.arg(&args.pattern);
    cmd.arg(&search_path);
    
    // Execute
    let output = cmd.output().await?;
    
    // Parse ripgrep JSON output
    let matches = parse_ripgrep_json(&output.stdout)?;
    
    Ok(matches)
}

fn parse_ripgrep_json(output: &[u8]) -> Result<Vec<GrepMatch>> {
    let mut matches = Vec::new();
    
    for line in output.split(|&b| b == b'\n') {
        if line.is_empty() { continue; }
        
        let json: Value = serde_json::from_slice(line)?;
        
        if json["type"] == "match" {
            let data = &json["data"];
            matches.push(GrepMatch {
                path: PathBuf::from(data["path"]["text"].as_str().unwrap()),
                line_number: data["line_number"].as_u64().unwrap() as usize,
                line_text: data["lines"]["text"].as_str().unwrap().to_string(),
                match_start: data["submatches"][0]["start"].as_u64().unwrap() as usize,
                match_end: data["submatches"][0]["end"].as_u64().unwrap() as usize,
            });
        }
    }
    
    Ok(matches)
}
```

---

## Plan Tool

**Location**: `core/src/tools/handlers/plan.rs`

### Purpose

Create and update task plans with status tracking.

### Implementation

```rust
pub const PLAN_TOOL: ToolSpec = ToolSpec {
    name: "update_plan",
    description: "Create or update a task plan with steps and status",
    parameters: /* JSON schema */,
};

#[derive(Deserialize)]
pub struct UpdatePlanArgs {
    pub steps: Vec<PlanStep>,
    pub explanation: Option<String>,
}

#[derive(Deserialize, Serialize, Clone)]
pub struct PlanStep {
    pub id: String,
    pub description: String,
    pub status: PlanStatus,
}

#[derive(Deserialize, Serialize, Clone)]
pub enum PlanStatus {
    Pending,
    InProgress,
    Completed,
    Cancelled,
}

pub async fn update_plan(
    args: UpdatePlanArgs,
    ctx: &ToolContext,
) -> Result<()> {
    // 1. Validate plan structure
    validate_plan(&args.steps)?;
    
    // 2. Update state
    ctx.plan_state.update(args.steps.clone()).await?;
    
    // 3. Emit event for UI
    ctx.tx_event.send(Event::PlanUpdated {
        steps: args.steps,
        explanation: args.explanation,
    }).await?;
    
    Ok(())
}

fn validate_plan(steps: &[PlanStep]) -> Result<()> {
    // At least 2 steps
    if steps.len() < 2 {
        return Err("Plan must have at least 2 steps".into());
    }
    
    // Unique IDs
    let ids: HashSet<_> = steps.iter().map(|s| &s.id).collect();
    if ids.len() != steps.len() {
        return Err("Plan steps must have unique IDs".into());
    }
    
    // Exactly one in_progress
    let in_progress_count = steps.iter()
        .filter(|s| matches!(s.status, PlanStatus::InProgress))
        .count();
    
    if in_progress_count > 1 {
        return Err("Only one step can be in_progress at a time".into());
    }
    
    // Description not empty
    for step in steps {
        if step.description.trim().is_empty() {
            return Err("Step descriptions cannot be empty".into());
        }
    }
    
    Ok(())
}
```

---

## MCP Integration

### Call MCP Tool

**Location**: `core/src/tools/handlers/mcp.rs`

```rust
#[derive(Deserialize)]
pub struct McpCallToolArgs {
    pub server: String,
    pub tool_name: String,
    pub arguments: Value,
}

pub async fn mcp_call_tool(
    args: McpCallToolArgs,
    ctx: &ToolContext,
) -> Result<ToolOutput> {
    // 1. Get MCP connection
    let connection = ctx.mcp_manager
        .get_connection(&args.server)
        .await?;
    
    // 2. Call tool
    let request = mcp_types::CallToolRequest {
        name: args.tool_name,
        arguments: Some(args.arguments),
    };
    
    let response = connection.call_tool(request).await?;
    
    // 3. Format result
    match response {
        mcp_types::CallToolResult::Success { content } => {
            Ok(ToolOutput {
                success: true,
                output: format_mcp_content(&content),
            })
        }
        mcp_types::CallToolResult::Error { error } => {
            Ok(ToolOutput {
                success: false,
                output: format!("MCP tool error: {}", error),
            })
        }
    }
}
```

### Read MCP Resource

**Location**: `core/src/tools/handlers/mcp_resource.rs`

```rust
#[derive(Deserialize)]
pub struct McpReadResourceArgs {
    pub server: String,
    pub uri: String,
}

pub async fn mcp_read_resource(
    args: McpReadResourceArgs,
    ctx: &ToolContext,
) -> Result<String> {
    // 1. Get MCP connection
    let connection = ctx.mcp_manager
        .get_connection(&args.server)
        .await?;
    
    // 2. Read resource
    let request = mcp_types::ReadResourceRequest {
        uri: args.uri,
    };
    
    let response = connection.read_resource(request).await?;
    
    // 3. Format content
    let formatted = response.contents
        .iter()
        .map(|content| match content {
            mcp_types::ResourceContent::Text { text } => text.clone(),
            mcp_types::ResourceContent::Blob { blob, mime_type } => {
                format!("[Binary data: {} bytes, type: {}]", blob.len(), mime_type)
            }
        })
        .collect::<Vec<_>>()
        .join("\n\n");
    
    Ok(formatted)
}
```

---

## Related Documentation

- [06-tool-system.md](./06-tool-system.md) - Tool architecture
- [07-security-sandboxing.md](./07-security-sandboxing.md) - Security implementation
- [14-mcp-integration.md](./14-mcp-integration.md) - MCP details

