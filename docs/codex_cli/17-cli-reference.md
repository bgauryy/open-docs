# CLI Command Complete Reference

Complete reference for all Codex CLI commands, flags, and TUI slash commands.

---

## Command Line Interface

### codex (Interactive TUI)

Start an interactive Codex session with the Terminal User Interface.

```bash
codex [OPTIONS] [PROMPT]
```

**Arguments:**
- `[PROMPT]` - Optional initial user message to start conversation

**Options:**

**Model & Provider:**
- `-m, --model <MODEL>` - Model to use (e.g., `gpt-5-codex`, `o3`)
- `--model-provider <PROVIDER>` - Provider from `model_providers` config

**Approval & Sandbox:**
- `-a, --ask-for-approval <POLICY>` - Approval policy: `untrusted`, `on-failure`, `on-request`, `never`
- `--sandbox <MODE>` - Sandbox mode: `read-only`, `workspace-write`, `danger-full-access`
- `--full-auto` - Shorthand for `--sandbox workspace-write --ask-for-approval on-request`
- `--yolo` - Shorthand for `--dangerously-bypass-approvals-and-sandbox`
- `--dangerously-bypass-approvals-and-sandbox` - No sandbox, no approvals (use with caution)

**Working Directory:**
- `-C, --cd <DIR>` - Change working directory before starting
- `--add-dir <DIR>` - Add additional writable directory (can be specified multiple times)

**Configuration:**
- `-c, --config <KEY=VALUE>` - Override config value (e.g., `-c model="o3"`)
- `--profile <PROFILE>` - Use specific profile from config.toml
- `--skip-git-repo-check` - Don't require Git repository

**Input/Output:**
- `-i, --image <PATH>` - Attach image(s) to initial prompt (comma-separated)

**Other:**
- `-h, --help` - Print help
- `-V, --version` - Print version

**Examples:**

```bash
# Basic interactive session
codex

# Start with a prompt
codex "Fix the build errors"

# Use specific model
codex --model o3 "Review this PR"

# Read-only mode
codex --sandbox read-only

# With image input
codex --image screenshot.png "Explain this error"

# Change working directory
codex --cd /path/to/project

# Multiple writable directories
codex --cd frontend --add-dir ../backend --add-dir ../shared
```

---

### codex exec (Non-Interactive)

Run Codex in non-interactive mode for automation.

```bash
codex exec [OPTIONS] <PROMPT>
codex exec [OPTIONS] resume [RESUME_OPTIONS]
```

**Arguments:**
- `<PROMPT>` - User prompt to execute

**Resume Subcommand:**
```bash
codex exec resume [OPTIONS]
codex exec resume --last
codex exec resume <SESSION_ID>
```

**Options:**

**Model & Provider:**
- `-m, --model <MODEL>` - Model to use
- `--model-provider <PROVIDER>` - Provider from config

**Approval & Sandbox:**
- `-a, --ask-for-approval <POLICY>` - Approval policy (typically `never` for exec)
- `--sandbox <MODE>` - Sandbox mode (default: `read-only`)
- `--full-auto` - Enable workspace writes with on-request approvals

**Output:**
- `--json` - Output JSONL events instead of human-readable text
- `-o, --output-last-message <FILE>` - Write final message to file
- `--output-schema <FILE>` - JSON Schema for structured output

**Configuration:**
- `-c, --config <KEY=VALUE>` - Override config value
- `--profile <PROFILE>` - Use specific profile
- `--skip-git-repo-check` - Don't require Git repository

**Other:**
- `-h, --help` - Print help

**Examples:**

```bash
# Basic exec (read-only)
codex exec "Count lines of code in this project"

# With file edits
codex exec --full-auto "Fix linting errors"

# JSON output
codex exec --json "Analyze performance bottlenecks"

# Structured output
codex exec --output-schema schema.json "Extract project metadata"

# Save final message
codex exec -o result.txt "Summarize changes"

# Resume previous session
codex exec resume --last "Continue the refactoring"
codex exec resume 01933e84-1234-7890-abcd "Add more tests"
```

---

### codex resume (TUI Resume)

Resume an interactive session in the TUI.

```bash
codex resume [OPTIONS]
codex resume --last
codex resume <SESSION_ID>
```

**Arguments:**
- `<SESSION_ID>` - UUID of session to resume (optional)

**Options:**
- `--last` - Resume most recent session
- All standard TUI options (`--model`, `--sandbox`, etc.)

**Examples:**

```bash
# Show picker of recent sessions
codex resume

# Resume last session
codex resume --last

# Resume specific session
codex resume 01933e84-1234-7890-abcd-ef0123456789
```

---

### codex login

Authenticate with OpenAI/ChatGPT.

```bash
codex login [OPTIONS]
```

**Options:**
- `--with-api-key` - Read API key from stdin (for usage-based billing)

**Examples:**

```bash
# ChatGPT login (default)
codex login

# API key login
printenv OPENAI_API_KEY | codex login --with-api-key

# From file
codex login --with-api-key < my_key.txt
```

**Notes:**
- ChatGPT login opens browser at `localhost:1455`
- API key never appears in shell history (use stdin)
- Credentials stored in `~/.codex/auth.json`

---

### codex mcp (MCP Management)

Manage Model Context Protocol servers.

```bash
codex mcp <SUBCOMMAND>
```

**Subcommands:**

#### add

Add an MCP server to config.

```bash
codex mcp add [OPTIONS] <NAME> [--] <COMMAND> [ARGS]...
```

**Arguments:**
- `<NAME>` - Server identifier
- `<COMMAND>` - Launch command
- `[ARGS]...` - Command arguments

**Options:**
- `--env <KEY=VALUE>` - Set environment variable (can be repeated)

**Example:**
```bash
codex mcp add docs -- npx -y @anthropic-ai/mcp-server-docs
codex mcp add my-server --env API_KEY=secret -- node server.js
```

#### list

List configured MCP servers.

```bash
codex mcp list [--json]
```

**Options:**
- `--json` - Output as JSON

**Example:**
```bash
codex mcp list
codex mcp list --json | jq
```

#### get

Show details for one MCP server.

```bash
codex mcp get <NAME> [--json]
```

**Arguments:**
- `<NAME>` - Server identifier

**Options:**
- `--json` - Output as JSON

**Example:**
```bash
codex mcp get docs
codex mcp get docs --json | jq .tools
```

#### remove

Remove an MCP server from config.

```bash
codex mcp remove <NAME>
```

**Arguments:**
- `<NAME>` - Server identifier

**Example:**
```bash
codex mcp remove docs
```

#### login

Authenticate to an OAuth-enabled MCP server.

```bash
codex mcp login <NAME>
```

**Arguments:**
- `<NAME>` - Server identifier

**Requirements:**
- Server must support OAuth
- `experimental_use_rmcp_client = true` in config

**Example:**
```bash
codex mcp login figma
```

#### logout

Log out from an OAuth-enabled MCP server.

```bash
codex mcp logout <NAME>
```

**Arguments:**
- `<NAME>` - Server identifier

**Example:**
```bash
codex mcp logout figma
```

---

### codex mcp-server

Run Codex as an MCP server.

```bash
codex mcp-server
```

Exposes two tools:
- `codex` - Run a Codex session
- `codex-reply` - Continue a session

**Example (with MCP Inspector):**
```bash
npx @modelcontextprotocol/inspector codex mcp-server
```

See [Advanced Guide](./advanced.md#mcp-server) for details.

---

### codex sandbox (Debugging)

Test sandbox behavior on your platform.

```bash
codex sandbox macos [--full-auto] <COMMAND> [ARGS]...
codex sandbox linux [--full-auto] <COMMAND> [ARGS]...
```

**Aliases:**
- `codex debug seatbelt` (macOS)
- `codex debug landlock` (Linux)

**Arguments:**
- `<COMMAND>` - Command to run in sandbox
- `[ARGS]...` - Command arguments

**Options:**
- `--full-auto` - Use `workspace-write` sandbox instead of `read-only`

**Examples:**

```bash
# Test read-only sandbox (macOS)
codex sandbox macos ls /

# Test workspace-write sandbox (Linux)
codex sandbox linux --full-auto touch /tmp/test.txt

# Test network access (should fail in read-only)
codex sandbox macos curl https://example.com
```

---

### codex completion

Generate shell completion scripts.

```bash
codex completion <SHELL>
```

**Shells:**
- `bash`
- `zsh`
- `fish`

**Examples:**

```bash
# Install Bash completions
codex completion bash > /usr/local/etc/bash_completion.d/codex

# Install Zsh completions
codex completion zsh > /usr/local/share/zsh/site-functions/_codex

# Install Fish completions
codex completion fish > ~/.config/fish/completions/codex.fish
```

---

## TUI Slash Commands

Commands available in the interactive TUI composer (type `/` to see menu).

### Built-in Slash Commands

#### /status

Show session status and configuration.

```
/status
```

**Displays:**
- Model and provider
- Approval policy and sandbox mode
- Working directory and writable roots
- MCP server connection status
- Token usage (if available)
- Rate limits (if available)

#### /model

Change model mid-session.

```
/model <MODEL_NAME>
```

**Examples:**
```
/model o3
/model gpt-5-codex
/model gpt-4o
```

**Notes:**
- Requires model to be supported by current provider
- Does not change provider

#### /approvals

Change approval policy mid-session.

```
/approvals
```

**Opens picker** with options:
- Read Only
- Auto
- Full Access

**Note:** Also accessible via `/sandbox` alias

#### /init

Initialize a new conversation (clear current session).

```
/init
```

**Effect:**
- Clears transcript
- Keeps same model, approval policy, sandbox mode
- Resets token count

#### /prompts:<name>

Run a custom prompt from `~/.codex/prompts/`.

```
/prompts:<name> [ARGS...]
```

**Examples:**
```
/prompts:draftpr feature_name=auth
/prompts:review FILE=src/main.rs FOCUS=error_handling
```

See [Prompts Reference](./prompts.md) for details.

---

### TUI Keyboard Shortcuts

**Composer:**
- `Enter` - Send message
- `Ctrl+C` or `Cmd+C` - Copy selected text
- `Ctrl+V` or `Cmd+V` - Paste (including images)
- `@` - Trigger file search
- `Tab` or `Enter` - Select file from search
- `Esc` - Cancel file search / Enter backtrack mode
- `Esc` `Esc` - Enter edit previous message mode

**Navigation:**
- `↑` / `↓` - Scroll transcript
- `PgUp` / `PgDn` - Page up/down
- `Home` / `End` - Jump to top/bottom

**Backtrack Mode:**
- `Esc` (in empty composer) - Prime backtrack mode
- `Esc` (repeatedly) - Navigate to older user messages
- `Enter` - Select message to edit
- `Esc` (in preview) - Cancel backtrack

---

## Environment Variables

### CODEX_HOME

Override default home directory (`~/.codex`).

```bash
export CODEX_HOME=/custom/path
codex
```

### CODEX_API_KEY

Override API key (exec mode only).

```bash
export CODEX_API_KEY=sk-...
codex exec "Task"
```

### OPENAI_API_KEY

API key for authentication (alternative to `codex login`).

```bash
export OPENAI_API_KEY=sk-...
printenv OPENAI_API_KEY | codex login --with-api-key
```

### OPENAI_BASE_URL

Override OpenAI API base URL.

```bash
export OPENAI_BASE_URL=https://custom.api.endpoint
codex
```

### RUST_LOG

Control log verbosity.

```bash
export RUST_LOG=codex_core=debug
codex
```

**Levels:**
- `error` - Errors only (default for exec)
- `warn` - Warnings and errors
- `info` - Info messages (default for TUI)
- `debug` - Debug messages
- `trace` - Verbose trace

### NO_COLOR

Disable colored output.

```bash
export NO_COLOR=1
codex exec "Task"
```

---

## Exit Codes

**codex exec:**

- `0` - Success
- `1` - Generic error
- `2` - Configuration error
- `3` - Authentication error
- `4` - API error
- `5` - Tool execution error
- `130` - Interrupted by user (Ctrl+C)

**codex (TUI):**

- `0` - Normal exit
- `1` - Error during startup
- `130` - Interrupted by user (Ctrl+C)

---

## Configuration Precedence

When the same setting is specified in multiple places:

1. **Command-line flag** (highest priority)
2. **`-c/--config` flag** (e.g., `-c model="o3"`)
3. **Profile** (via `--profile` or `profile` in config.toml)
4. **config.toml** (root level)
5. **Built-in default** (lowest priority)

**Example:**

```bash
# config.toml has:
# model = "gpt-5"
# profile = "default"
#
# [profiles.fast]
# model = "gpt-4o"

codex --profile fast --model o3 "Task"
# Uses: o3 (flag overrides profile)

codex --profile fast "Task"
# Uses: gpt-4o (profile overrides config.toml root)

codex "Task"
# Uses: gpt-5 (config.toml root)
```

---

## Common Workflows

### Quick code review

```bash
codex --sandbox read-only "Review changes in this PR"
```

### Fix build errors

```bash
codex --full-auto "Fix all build errors and run tests"
```

### Generate tests

```bash
codex --sandbox workspace-write "Add unit tests for src/auth.rs"
```

### Export conversation

```bash
codex resume --last
# In TUI, work on task
# Exit with Ctrl+D
cat ~/.codex/sessions/$(ls -t ~/.codex/sessions/ | head -1)/transcript.json
```

### Automated analysis

```bash
codex exec --json "Find security vulnerabilities" | jq '.items[] | select(.type == "agent_message")'
```

### Multi-directory project

```bash
codex --cd apps/frontend --add-dir ../backend --add-dir ../shared "Refactor API client"
```

---

## References

- [Getting Started](./getting-started.md) - Basic usage and tips
- [Config Reference](./config.md) - Configuration options
- [Sandbox & Approvals](./sandbox.md) - Security model
- [Exec Reference](./exec.md) - Non-interactive mode details
- [Custom Prompts](./prompts.md) - Slash command customization
