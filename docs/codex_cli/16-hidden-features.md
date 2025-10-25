# Hidden Features & Undocumented Commands

This guide documents features, commands, and configuration options that exist in the Codex codebase but are not mentioned in the official documentation.

**Source:** Discovered through systematic code analysis of Codex CLI source code
**Verification:** All features verified against actual implementation

---

## Table of Contents

- [Undocumented Slash Commands](#undocumented-slash-commands)
- [Experimental Feature Flags](#experimental-feature-flags)
- [Undocumented CLI Flags](#undocumented-cli-flags)
- [Undocumented Environment Variables](#undocumented-environment-variables)
- [Beta Features](#beta-features)
- [Internal Tools](#internal-tools)

---

## Undocumented Slash Commands

### /review

**Purpose:** Review current changes and find issues

**Usage:**
```
/review
/review @path/to/file
```

**What it does:**
- Analyzes git diff and untracked files
- Identifies potential bugs, security issues, code smells
- Provides suggestions for improvement

**Implementation:** `tui/src/slash_command.rs:16`

---

### /undo (BETA)

**Purpose:** Restore workspace to last Codex snapshot

**Availability:** Only when `BETA_FEATURE` environment variable is set

**Usage:**
```bash
export BETA_FEATURE=1
codex

# In TUI:
/undo
```

**What it does:**
- Reverts file changes made by Codex in the current session
- Uses git or filesystem snapshots to restore previous state
- **WARNING:** Experimental feature, may have edge cases

**Implementation:** `tui/src/slash_command.rs:18`

---

### /diff

**Purpose:** Show git diff including untracked files

**Usage:**
```
/diff
```

**What it does:**
- Displays git diff for tracked files
- Includes untracked files in the output
- Useful for seeing what Codex changed

**Output:** Formatted diff is inserted into the chat as a system message

**Implementation:** `tui/src/slash_command.rs:21`

---

### /mention

**Purpose:** Mention a file in the conversation

**Usage:**
```
/mention @path/to/file
/mention
```

**What it does:**
- Allows explicit file reference without asking model to read it
- Opens file picker if no path provided
- Useful for directing model's attention to specific files

**Tip:** You can also type `@filename` directly in messages for file search

**Implementation:** `tui/src/slash_command.rs:22`

---

### /feedback

**Purpose:** Send logs to maintainers

**Usage:**
```
/feedback
```

**What it does:**
- Collects session logs and diagnostic information
- Prepares feedback for Codex developers
- Helps report bugs and issues

**Privacy:** Check what data is collected before submitting

**Implementation:** `tui/src/slash_command.rs:27`

---

### /compact

**Purpose:** Manually trigger conversation compaction

**Usage:**
```
/compact
```

**What it does:**
- Forces immediate compaction of conversation history
- Summarizes older messages to save context window space
- Useful when approaching token limits

**When to use:**
- Long conversations nearing context limit
- Want to reduce token cost
- Need to free up context space

**Implementation:** `tui/src/slash_command.rs:17`

---

### /new

**Purpose:** Start a new chat during a conversation

**Usage:**
```
/new
```

**What it does:**
- Clears current transcript
- Starts fresh conversation
- Keeps same configuration (model, approvals, sandbox)

**Alias:** `/init` does the same thing

**Implementation:** `tui/src/slash_command.rs:15`

---

### /mcp

**Purpose:** List configured MCP tools

**Usage:**
```
/mcp
```

**What it does:**
- Shows all MCP servers and their connection status
- Lists available MCP tools
- Displays tool descriptions

**Useful for:** Debugging MCP integration

**Implementation:** `tui/src/slash_command.rs:24`

---

### /logout

**Purpose:** Log out of Codex

**Usage:**
```
/logout
```

**What it does:**
- Removes authentication credentials from `~/.codex/auth.json`
- Exits current session
- Requires re-authentication on next run

**Implementation:** `tui/src/slash_command.rs:25`

---

### /test-approval (Debug Only)

**Purpose:** Test approval request UI

**Availability:** Only in debug builds (`cargo build` without `--release`)

**Usage:**
```
/test-approval
```

**What it does:**
- Triggers a mock approval request
- Used for testing approval UI during development

**Implementation:** `tui/src/slash_command.rs:28-29`

---

## Experimental Feature Flags

### Feature System

Codex has a centralized feature flag system in `core/src/features.rs`.

**Configuration:**

```toml
[features]
unified_exec = true
streamable_shell = false
rmcp_client = true
apply_patch_freeform = false
view_image_tool = true
web_search_request = false
```

---

### unified_exec

**Stage:** Experimental
**Default:** Disabled

**Purpose:** Use single unified PTY-backed exec tool instead of separate tools

**Benefits:**
- Better terminal emulation
- Interactive command support
- Improved handling of shell sessions

**Enable:**
```toml
[features]
unified_exec = true
```

**Source:** `core/src/features.rs:29`

---

### streamable_shell

**Stage:** Experimental
**Default:** Disabled

**Purpose:** Use streamable exec-command/write-stdin tool pair

**Benefits:**
- Streaming command output
- Ability to write to stdin during execution
- Better for long-running interactive commands

**Enable:**
```toml
[features]
streamable_shell = true
```

**Source:** `core/src/features.rs:31`

---

### rmcp_client

**Stage:** Experimental
**Default:** Disabled

**Purpose:** Enable experimental RMCP features including OAuth login

**Benefits:**
- OAuth authentication for MCP servers
- Advanced MCP protocol features
- Streamable HTTP MCP servers

**Enable:**
```toml
experimental_use_rmcp_client = true

# Or via new features system:
[features]
rmcp_client = true
```

**Usage:**
```bash
codex mcp login <server-name>
```

**Source:** `core/src/features.rs:33`

---

### apply_patch_freeform

**Stage:** Beta
**Default:** Disabled

**Purpose:** Include freeform apply_patch tool (less structured than default)

**Enable:**
```toml
[features]
apply_patch_freeform = true
```

**Source:** `core/src/features.rs:35`

---

## Undocumented CLI Flags

### --oss

**Purpose:** Use Ollama/OSS model provider instead of OpenAI

**Usage:**
```bash
codex --oss
```

**What it does:**
- Switches to open-source model provider (Ollama)
- Uses `CODEX_OSS_BASE_URL` environment variable for endpoint
- Uses `CODEX_OSS_PORT` environment variable for port (default based on URL)

**Source:** `tui/src/cli.rs:oss`

**Related Environment Variables:**
- `CODEX_OSS_BASE_URL` - OSS endpoint URL
- `CODEX_OSS_PORT` - OSS endpoint port

---

### --search

**Purpose:** Enable web search tool

**Usage:**
```bash
codex --search
```

**What it does:**
- Enables the `web_search` tool for the model
- Allows model to search the internet for information
- Requires appropriate permissions/configuration

**Source:** `tui/src/cli.rs:search`

---

### --device-auth

**Purpose:** Use device code OAuth flow for authentication

**Usage:**
```bash
codex login --device-auth
```

**What it does:**
- Uses device code OAuth flow instead of browser-based flow
- Useful for headless environments or remote sessions
- Displays a code to enter on another device

**Source:** `tui/src/cli.rs:use_device_code`

---

### --experimental_issuer

**Purpose:** Use custom OAuth issuer URL

**Usage:**
```bash
codex login --experimental_issuer https://custom-auth.example.com
```

**What it does:**
- Overrides default OAuth issuer
- For testing or custom authentication setups
- **Hidden flag** - not shown in help

**Source:** `tui/src/cli.rs:issuer_base_url`

---

### --experimental_client-id

**Purpose:** Use custom OAuth client ID

**Usage:**
```bash
codex login --experimental_client-id custom-client-id
```

**What it does:**
- Overrides default OAuth client ID
- For testing or custom authentication setups
- **Hidden flag** - not shown in help

**Source:** `tui/src/cli.rs:client_id`

---

### --color

**Purpose:** Control color output in exec mode

**Usage:**
```bash
codex exec --color never "Task"
codex exec --color auto "Task"
codex exec --color always "Task"
```

**Options:**
- `never` - Never use colors
- `auto` - Auto-detect (default)
- `always` - Always use colors

**Source:** `exec/src/cli.rs:color`

---

## Undocumented Environment Variables

### Development & Debugging

#### CODEX_TUI_ROUNDED
**Purpose:** Enable rounded UI borders in TUI

**Usage:**
```bash
export CODEX_TUI_ROUNDED=1
codex
```

**Effect:** Changes TUI border style to rounded corners

**Source:** `cloud-tasks/src/ui.rs:65`

---

#### CODEX_TUI_RECORD_SESSION
**Purpose:** Enable session recording

**Usage:**
```bash
export CODEX_TUI_RECORD_SESSION=1
codex
```

**Effect:** Records TUI session for debugging/replay

**Source:** `tui/src/session_log.rs:81`

---

#### CODEX_TUI_SESSION_LOG_PATH
**Purpose:** Set custom path for session logs

**Usage:**
```bash
export CODEX_TUI_SESSION_LOG_PATH=/path/to/logs
export CODEX_TUI_RECORD_SESSION=1
codex
```

**Effect:** Saves session recordings to specified path

**Source:** `tui/src/session_log.rs:88`

---

### Cloud/Enterprise Features

#### CODEX_CLOUD_TASKS_MODE
**Purpose:** Set cloud tasks mode

**Usage:**
```bash
export CODEX_CLOUD_TASKS_MODE=internal
codex
```

**Effect:** Configures cloud tasks integration mode

**Source:** `cloud-tasks/src/lib.rs:35`

---

#### CODEX_CLOUD_TASKS_BASE_URL
**Purpose:** Set cloud tasks API endpoint

**Usage:**
```bash
export CODEX_CLOUD_TASKS_BASE_URL=https://tasks.example.com
codex
```

**Effect:** Uses custom cloud tasks endpoint

**Source:** `cloud-tasks/src/lib.rs:38`

---

#### CODEX_CLOUD_TASKS_FORCE_INTERNAL
**Purpose:** Force internal cloud tasks mode

**Usage:**
```bash
export CODEX_CLOUD_TASKS_FORCE_INTERNAL=1
codex
```

**Effect:** Forces use of internal cloud tasks

**Source:** `cloud-tasks/src/lib.rs:381`

---

### OSS/Ollama Configuration

#### CODEX_OSS_BASE_URL
**Purpose:** Set Ollama/OSS model endpoint

**Usage:**
```bash
export CODEX_OSS_BASE_URL=http://localhost:11434
codex --oss
```

**Effect:** Configures OSS model provider endpoint

**Source:** `core/src/model_provider_info.rs:322`

---

#### CODEX_OSS_PORT
**Purpose:** Set Ollama/OSS model port

**Usage:**
```bash
export CODEX_OSS_PORT=11434
codex --oss
```

**Effect:** Overrides default port for OSS endpoint

**Source:** `core/src/model_provider_info.rs:329`

---

### Package Management

#### CODEX_MANAGED_BY_NPM
**Purpose:** Indicate Codex is installed via npm

**Usage:**
```bash
export CODEX_MANAGED_BY_NPM=1
```

**Effect:**
- Disables update checks (managed by npm)
- Changes update instructions shown to user

**Source:** `tui/src/updates.rs:157-158`

---

#### CODEX_MANAGED_BY_BUN
**Purpose:** Indicate Codex is installed via Bun

**Usage:**
```bash
export CODEX_MANAGED_BY_BUN=1
```

**Effect:**
- Disables update checks (managed by Bun)
- Changes update instructions shown to user

**Source:** `tui/src/updates.rs:158`

---

### Testing & Development

#### CODEX_STARTING_DIFF
**Purpose:** Set initial diff for testing

**Usage:**
```bash
export CODEX_STARTING_DIFF="diff content"
codex
```

**Effect:** Injects starting diff for testing purposes

**Source:** `cloud-tasks-client/src/http.rs:234`

---

#### CODEX_APPLY_GIT_CFG
**Purpose:** Configure git apply behavior

**Usage:**
```bash
export CODEX_APPLY_GIT_CFG="config-value"
codex
```

**Effect:** Passes custom configuration to git apply

**Source:** `git-apply/src/lib.rs:48`

---

### MCP Server Development

#### MCP_STREAMABLE_HTTP_BIND_ADDR
**Purpose:** Set bind address for MCP HTTP server

**Usage:**
```bash
export MCP_STREAMABLE_HTTP_BIND_ADDR=0.0.0.0:8080
```

**Effect:** Configures HTTP MCP server listen address

**Source:** `rmcp-client/src/bin/test_streamable_http_server.rs:238`

---

#### MCP_EXPECT_BEARER
**Purpose:** Set expected bearer token for MCP HTTP server

**Usage:**
```bash
export MCP_EXPECT_BEARER=secret-token
```

**Effect:** Requires bearer authentication for MCP server

**Source:** `rmcp-client/src/bin/test_streamable_http_server.rs:268`

---

## Beta Features

### BETA_FEATURE Environment Variable

**Purpose:** Enable beta/experimental features

**Usage:**
```bash
export BETA_FEATURE=1
codex
```

**Features enabled:**
- `/undo` slash command
- Other unstable features

**Warning:** Beta features may be buggy or incomplete

**Source:** `tui/src/slash_command.rs:96-98`

---

## Internal Tools

### update_plan Tool

Codex has an internal plan/TODO tool for tracking multi-step tasks.

**Tool Name:** `update_plan`

**Data Structure:**

```rust
pub enum StepStatus {
    Pending,
    InProgress,
    Completed,
}

pub struct PlanItemArg {
    pub step: String,
    pub status: StepStatus,
}

pub struct UpdatePlanArgs {
    pub explanation: Option<String>,
    pub plan: Vec<PlanItemArg>,
}
```

**Purpose:**
- Track multi-step task progress
- Display step-by-step plan to user
- Update status as tasks complete

**Note:** This tool is used internally by the model to structure its work. At most one step can be `in_progress` at a time.

**Source:** `core/src/tools/handlers/plan.rs:47`

---

### Built-in Tools

The following tools are available in Codex (verified from `core/src/tools/spec.rs`):

**Shell/Execution:**
- `exec_command` - Execute commands with PTY
- `write_stdin` - Write to command stdin
- `shell` - Shell command execution
- `test_sync_tool` - Test synchronous tool

**File Operations:**
- `grep_files` - Search file contents
- `read_file` - Read file contents
- `list_dir` - List directory contents

**MCP Integration:**
- `list_mcp_resources` - List MCP resources
- `list_mcp_resource_templates` - List MCP resource templates
- `read_mcp_resource` - Read MCP resource

**Other:**
- `view_image` - View image files (when enabled)
- `update_plan` - Update task plan

---

## Git Integration Features

### Ghost Commits

Codex has internal git tooling for "ghost commits" (temporary snapshots):

**Purpose:**
- Create temporary commits for tracking changes
- Enable `/undo` functionality
- Track Codex-made changes separately from user commits

**Implementation:** `git-tooling/src/ghost_commits.rs`

**User-facing:** Not directly exposed, used internally for undo/snapshot features

---

## MCP Protocol Versions

Codex supports multiple MCP protocol versions:

**Supported versions:**
- `2025-03-26` (schema in `mcp-types/schema/2025-03-26/schema.json`)
- `2025-06-18` (schema in `mcp-types/schema/2025-06-18/schema.json`)

**Auto-detection:** Codex negotiates protocol version during MCP server initialization

**Source:** `mcp-types/` directory

---

## File Search Integration

### @ Token File Search

The `@` file search feature is more powerful than documented:

**Features:**
- Fuzzy filename matching
- Path-based search
- Real-time results as you type
- Works in slash command arguments (e.g., `/review @filename`)

**Implementation:** `app-server/src/fuzzy_file_search.rs`

**Tip:** Works in custom prompt arguments too!

---

## Session File Formats

### Internal Session Structure

Sessions stored in `~/.codex/sessions/<uuid>/` contain:

**Files:**
- `metadata.json` - Session metadata
- `transcript.json` - Full conversation
- `state.json` - Current state snapshot

**Undocumented state fields:**
- Pending approvals
- MCP connection states
- Compaction status
- Last turn ID

**Use case:** Advanced session manipulation or recovery

---

## Best Practices for Hidden Features

### Using Beta Features Safely

1. **Test in isolated environment** - Don't use beta features on production code
2. **Check version compatibility** - Beta features may break across versions
3. **Report issues** - Use `/feedback` to report beta feature bugs
4. **Backup important work** - Beta features like `/undo` may have bugs

### When to Use Experimental Flags

**Use experimental features when:**
- Testing new Codex capabilities
- Contributing to development
- Need specific experimental functionality

**Avoid when:**
- Working on critical projects
- Stability is important
- You can't afford downtime

---

## Feature Request Process

### Requesting Hidden Features Be Documented

If you find a hidden feature useful and want it documented:

1. Test the feature thoroughly
2. Document your findings
3. Open an issue on GitHub: https://github.com/openai/codex/issues
4. Include code references (file:line)
5. Explain use cases

---

## References

### Source Files

All features documented here were found in:

- `tui/src/slash_command.rs` - Slash command definitions
- `tui/src/cli.rs` - TUI CLI arguments
- `exec/src/cli.rs` - Exec CLI arguments
- `core/src/features.rs` - Feature flag system
- `core/src/model_provider_info.rs` - Model provider configuration
- `core/src/auth.rs` - Authentication
- `core/src/tools/spec.rs` - Tool definitions
- `core/src/tools/handlers/plan.rs` - Plan tool
- `protocol/src/custom_prompts.rs` - Custom prompt metadata
- `core/src/features/legacy.rs` - Legacy config mappings
- `git-tooling/src/ghost_commits.rs` - Git integration
- `cloud-tasks/src/lib.rs` - Cloud tasks integration
- `tui/src/updates.rs` - Update management
- `tui/src/session_log.rs` - Session recording

### Related Documentation

- [Code Reference](./15-code-reference.md) - Code structure reference
- [Configuration](./08-configuration.md) - Configuration options
- [Tool System](./06-tool-system.md) - Tool architecture
- [MCP Integration](./14-mcp-integration.md) - MCP features

---

**Last Updated:** October 25, 2025
**Discovery Method:** Systematic codebase analysis using ripgrep and source code inspection
**Verification:** All features verified against source code in Codex repository
