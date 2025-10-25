# OpenCode - CLI Reference

> **Complete reference for all OpenCode command-line interface commands, flags, and usage patterns**

---

## Table of Contents

- [Entry Point](#entry-point)
- [Command Overview](#command-overview)
- [Core Commands](#core-commands)
- [Protocol Commands](#protocol-commands)
- [Management Commands](#management-commands)
- [Debug Commands](#debug-commands)
- [Global Flags](#global-flags)
- [Examples](#examples)

---

## Entry Point

OpenCode uses **yargs** for CLI argument parsing. The main entry point is:

**File**: `packages/opencode/src/index.ts`

```typescript
const cli = yargs(hideBin(process.argv))
  .scriptName("opencode")
  .help("help", "show help")
  .version("version", "show version number", Installation.VERSION)
  .alias("version", "v")
  .option("print-logs", {
    describe: "print logs to stderr",
    type: "boolean",
  })
  .option("log-level", {
    describe: "log level",
    type: "string",
    choices: ["DEBUG", "INFO", "WARN", "ERROR"],
  })
  .usage("\n" + UI.logo())
  .command(AcpCommand)
  .command(McpCommand)
  .command(TuiCommand)
  .command(AttachCommand)
  .command(RunCommand)
  .command(GenerateCommand)
  .command(DebugCommand)
  .command(AuthCommand)
  .command(AgentCommand)
  .command(UpgradeCommand)
  .command(ServeCommand)
  .command(ModelsCommand)
  .command(StatsCommand)
  .command(ExportCommand)
  .command(GithubCommand)
  .strict()
```

---

## Command Overview

### Command Hierarchy

```
opencode
├── (default)           # Interactive TUI (same as 'opencode tui')
├── run                 # Run with direct prompt
├── tui                 # Full-screen terminal interface
├── serve               # Headless HTTP server
├── acp                 # Agent Client Protocol server
├── mcp                 # Model Context Protocol tools
│   └── add            # Add MCP server
├── auth                # Authentication management
│   ├── list           # List providers
│   └── login          # Authenticate provider
├── agent               # Agent management (list/create/delete)
├── models              # List available models
├── stats               # Show project statistics
├── export              # Export session
├── attach              # Attach to running session
├── upgrade             # Upgrade OpenCode
├── generate            # Generate code/files
├── github              # GitHub integration
│   ├── install        # Install GitHub app
│   └── run            # Run in GitHub Action
└── debug               # Debug utilities
    ├── config         # Show configuration
    ├── file           # File analysis
    ├── lsp            # LSP debugging
    ├── ripgrep        # Search testing
    ├── scrap          # Temporary data cleanup
    └── snapshot       # Snapshot inspection
```

### Quick Reference Table

| Command | Purpose | Primary Use Case |
|---------|---------|------------------|
| `opencode` | Interactive TUI | Daily coding with AI assistance |
| `run` | One-shot prompt | Quick questions, automation scripts |
| `serve` | HTTP API server | Custom clients, team server |
| `acp` | IDE integration | Zed, VS Code, other ACP clients |
| `mcp` | Extend with MCP servers | Add custom tools and context |
| `auth` | Provider authentication | Setup API keys |
| `models` | List available models | See what you can use |
| `tui` | Full-screen interface | Immersive coding sessions |

---

## Core Commands

### Default Command (TUI)

**Usage**: `opencode [project]`

The default command when no subcommand is specified. Launches the full-screen Terminal User Interface.

**Arguments**:
- `[project]` - Path to project directory (default: current directory)

**Options**:
```
--model, -m <provider/model>   Model to use (e.g., anthropic/claude-3-5-sonnet-20241022)
--continue, -c                  Continue the last session
--session, -s <id>              Continue specific session by ID
--prompt, -p <text>             Initial prompt to send
--agent <name>                  Agent to use
--port <number>                 Server port (default: random)
--hostname, -h <host>           Server hostname (default: 127.0.0.1)
```

**Examples**:
```bash
# Start in current directory
opencode

# Start in specific directory
opencode ~/projects/my-app

# Continue last session
opencode --continue

# Use specific model
opencode -m anthropic/claude-3-5-sonnet-20241022

# Start with prompt
opencode -p "Add error handling to auth.ts"

# Use specific agent
opencode --agent build
```

**Behavior**:
1. Checks if providers are configured (if not, prompts for auth)
2. Starts HTTP server for TUI to connect to
3. Launches Go-based TUI binary (or builds it in dev mode)
4. Connects TUI to server via HTTP
5. Auto-upgrades in background if new version available
6. Auto-installs IDE integrations if not present

---

### `run` - Direct Prompt Execution

**Usage**: `opencode run [message...]`

Execute a single prompt without launching the TUI. Perfect for automation, quick questions, and scripting.

**Arguments**:
- `[message...]` - The prompt message (multiple words form complete message)

**Options**:
```
--command <cmd>                Command to run (uses message as arguments)
--continue, -c                 Continue the last session
--session, -s <id>             Continue specific session
--share                        Share the session (get shareable link)
--model, -m <provider/model>   Model to use
--agent <agent>                Agent to use
--format <type>                Output format: default | json
--file, -f <path>              Attach file(s) to message (multiple allowed)
```

**Examples**:

```bash
# Simple prompt
opencode run "Add type annotations to user.ts"

# Continue previous session
opencode run --continue "Now add tests for that"

# Attach files
opencode run -f auth.ts -f types.ts "Review these files for security issues"

# Share session and get link
opencode run --share "Implement OAuth flow"

# Use specific model
opencode run -m openai/gpt-4 "Explain this algorithm"

# JSON output for scripting
opencode run --format json "List all TODO items" | jq '.text'

# Run command with arguments
opencode run --command fix-types "auth.ts user.ts"

# Pipe input
cat error.log | opencode run "Analyze this error"
```

**Output Formats**:

**Default (formatted)**:
```
| Edit    config.ts
| Bash    npm test
| Read    README.md

I've added the requested features...
```

**JSON (structured)**:
```json
{"type":"tool_use","timestamp":1234567890,"sessionID":"...","part":{...}}
{"type":"text","timestamp":1234567890,"text":"I've added..."}
{"type":"step_finish","timestamp":1234567890}
```

**Tool Output**:
The run command displays tool usage in real-time:

| Tool | Display | Color |
|------|---------|-------|
| `edit` | Edit | Green |
| `write` | Write | Green |
| `read` | Read | Cyan |
| `bash` | Bash | Red |
| `grep` | Grep | Blue |
| `glob` | Glob | Blue |
| `todowrite` | Todo | Yellow |

**Stdin Support**:
```bash
# Append stdin to message
echo "function buggy() { ... }" | opencode run "Fix this function"

# Process file content
cat large-file.log | opencode run "Summarize errors"
```

**Exit Codes**:
- `0` - Success
- `1` - Error occurred (file not found, session error, etc.)

---

### `serve` - HTTP Server Mode

**Usage**: `opencode serve`

Start a headless OpenCode server that exposes an HTTP API. Perfect for:
- Custom client development
- Team shared server
- Remote access scenarios
- Testing and debugging

**Options**:
```
--port, -p <number>      Port to listen on (default: random)
--hostname, -h <host>    Hostname to bind to (default: 127.0.0.1)
```

**Examples**:
```bash
# Start on random port
opencode serve

# Specific port
opencode serve --port 8080

# Bind to all interfaces
opencode serve --hostname 0.0.0.0 --port 3000

# Production setup
opencode serve --hostname 127.0.0.1 --port 8080
```

**Output**:
```
opencode server listening on http://127.0.0.1:8080
```

**API Endpoints**:
Once running, the server exposes RESTful API:
- `GET /project` - List projects
- `POST /project/init` - Create project
- `GET /project/:id/session` - List sessions
- `POST /project/:id/session` - Create session
- `POST /project/:id/session/:sid/message` - Send message
- `GET /project/:id/session/:sid/message` - Get messages
- `POST /project/:id/session/:sid/compact` - Compact session

See [15-server-architecture.md](./15-server-architecture.md) for complete API reference.

---

### `tui` - Terminal User Interface

**Usage**: `opencode tui [project]`

Explicitly launch the full-screen TUI (this is the same as the default command).

**Arguments**:
- `[project]` - Path to project directory (default: current directory)

**Options**: (same as default command)
```
--model, -m <provider/model>
--continue, -c
--session, -s <id>
--prompt, -p <text>
--agent <name>
--port <number>
--hostname, -h <host>
```

**Examples**:
```bash
opencode tui
opencode tui ~/projects/backend
opencode tui --model anthropic/claude-3-opus-20240229
```

---

## Protocol Commands

### `acp` - Agent Client Protocol

**Usage**: `opencode acp`

Start an ACP server that communicates via JSON-RPC over stdio. Enables IDE integration with editors like Zed, VS Code, and others that support the Agent Client Protocol.

**Options**:
```
--cwd <path>    Working directory for the agent (default: current directory)
```

**Examples**:
```bash
# Start ACP server
opencode acp

# Specific directory
opencode acp --cwd /path/to/project
```

**IDE Integration**:

**Zed** (`~/.config/zed/settings.json`):
```json
{
  "agent_servers": {
    "OpenCode": {
      "command": "opencode",
      "args": ["acp"]
    }
  }
}
```

**VS Code** (via ACP extension):
```json
{
  "acp.agents": {
    "opencode": {
      "command": "opencode",
      "args": ["acp"]
    }
  }
}
```

**Protocol Details**:
- **Transport**: JSON-RPC 2.0 over stdio
- **Methods**: `initialize`, `session/new`, `session/load`, `session/prompt`
- **Capabilities**: File read/write, permission requests, terminal support
- **Version**: ACP v1

See [11-acp-protocol.md](./11-acp-protocol.md) for complete protocol documentation.

---

### `mcp` - Model Context Protocol

**Usage**: `opencode mcp [command]`

Manage Model Context Protocol servers for extending OpenCode with custom tools and resources.

#### `mcp add` - Add MCP Server

**Usage**: `opencode mcp add <url>`

Add an MCP server to your OpenCode configuration.

**Arguments**:
- `<url>` - URL or npm package of MCP server

**Examples**:
```bash
# Add from npm
opencode mcp add @modelcontextprotocol/server-filesystem

# Add from GitHub
opencode mcp add github:username/mcp-server

# Add local server
opencode mcp add file:///path/to/server
```

**What it does**:
1. Resolves the MCP server package
2. Adds to `.opencode/config.json`
3. Makes tools/resources available in sessions

**Configuration Result**:
```json
{
  "mcp": {
    "servers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem"]
      }
    }
  }
}
```

See [12-mcp-integration.md](./12-mcp-integration.md) for MCP details.

---

## Management Commands

### `auth` - Authentication Management

**Usage**: `opencode auth [subcommand]`

Manage authentication for AI providers.

#### `auth list` - List Providers

**Usage**: `opencode auth list`

Display all available providers and their authentication status.

**Output**:
```
┌─────────────┬────────────┬──────────────────────────┐
│ Provider    │ Status     │ Models                   │
├─────────────┼────────────┼──────────────────────────┤
│ anthropic   │ ✓ Authed   │ claude-3-5-sonnet, ...   │
│ openai      │ ✗ Not auth │ gpt-4, gpt-4-turbo, ...  │
│ google      │ ✓ Authed   │ gemini-pro, ...          │
└─────────────┴────────────┴──────────────────────────┘
```

#### `auth login` - Authenticate Provider

**Usage**: `opencode auth login [provider]`

Interactively authenticate with an AI provider.

**Arguments**:
- `[provider]` - Provider name (anthropic, openai, google, bedrock)

**Examples**:
```bash
# Interactive selection
opencode auth login

# Specific provider
opencode auth login anthropic
opencode auth login openai
opencode auth login google
```

**Interactive Flow**:
```
? Select provider: anthropic
? Enter API key: sk-ant-***
✓ Authentication successful
✓ Found 5 models: claude-3-5-sonnet, ...
```

**Where Keys Are Stored**:
- **Global**: `~/.opencode/config.json`
- **Project**: `.opencode/config.json`
- **Environment**: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.

**Priority**: CLI > Environment > Project Config > Global Config

---

### `agent` - Agent Management

**Usage**: `opencode agent [subcommand]`

Manage custom agents with specific behaviors and prompts.

**Subcommands**:
- `list` - List all agents
- `create <name>` - Create new agent
- `delete <name>` - Delete agent
- `edit <name>` - Edit agent configuration

**Agent Structure**:
```
.opencode/agent/
├── build.md           # Build/coding agent
├── review.md          # Code review agent
└── docs.md            # Documentation agent
```

**Example Agent** (`.opencode/agent/build.md`):
```markdown
# Build Agent

You are a senior software engineer focused on building robust, maintainable code.

## Guidelines
- Write tests for all new functions
- Use TypeScript strict mode
- Follow project style guide
- Add JSDoc comments
- Handle errors explicitly

## Model
anthropic/claude-3-5-sonnet-20241022
```

---

### `models` - List Available Models

**Usage**: `opencode models`

List all models from authenticated providers.

**Output**:
```
anthropic/claude-3-5-sonnet-20241022
anthropic/claude-3-5-haiku-20241022
anthropic/claude-3-opus-20240229
openai/gpt-4-turbo
openai/gpt-4
openai/gpt-3.5-turbo
google/gemini-pro
google/gemini-ultra
```

**Usage in Commands**:
```bash
# Use specific model
opencode run -m anthropic/claude-3-opus-20240229 "Complex reasoning task"

# In TUI
opencode --model openai/gpt-4-turbo
```

---

### `stats` - Project Statistics

**Usage**: `opencode stats`

Display project statistics and metadata.

**Output**:
```
Project: my-app
Path: /Users/me/projects/my-app

Files: 234
Lines: 15,402
Languages:
  - TypeScript: 78%
  - JavaScript: 15%
  - JSON: 5%
  - Other: 2%

Sessions: 12
Messages: 156
```

---

### `export` - Export Session

**Usage**: `opencode export <session-id>`

Export a session to a shareable format.

**Arguments**:
- `<session-id>` - Session identifier

**Options**:
```
--format <type>    Output format: json | markdown (default: json)
--output <path>    Output file (default: stdout)
```

**Examples**:
```bash
# Export to stdout
opencode export sess_abc123

# Export to file
opencode export sess_abc123 --output session.json

# Markdown format
opencode export sess_abc123 --format markdown --output session.md
```

---

### `attach` - Attach to Session

**Usage**: `opencode attach <session-id>`

Attach to a running session in a new terminal.

**Arguments**:
- `<session-id>` - Session identifier

**Example**:
```bash
opencode attach sess_abc123
```

---

### `upgrade` - Upgrade OpenCode

**Usage**: `opencode upgrade`

Upgrade OpenCode to the latest version.

**Detection**:
Automatically detects installation method:
- npm/bun/pnpm/yarn
- Homebrew
- Scoop/Chocolatey
- Manual install script

**Example**:
```bash
opencode upgrade
```

**Output**:
```
✓ Found new version: 0.15.18
✓ Upgrading via npm...
✓ Successfully upgraded to 0.15.18
```

---

### `generate` - Code Generation

**Usage**: `opencode generate [type]`

Generate code, files, or project scaffolding.

**Types**:
- `component` - Generate UI component
- `api` - Generate API endpoint
- `test` - Generate test file
- `types` - Generate TypeScript types
- `config` - Generate configuration file

**Examples**:
```bash
opencode generate component Button
opencode generate api users
opencode generate test auth.ts
```

---

### `github` - GitHub Integration

**Usage**: `opencode github [subcommand]`

Integrate OpenCode with GitHub repositories and actions.

#### `github install` - Install GitHub App

**Usage**: `opencode github install`

Install OpenCode GitHub app for repository integration.

**Example**:
```bash
opencode github install
```

Opens browser to GitHub app installation page.

#### `github run` - Run in GitHub Action

**Usage**: `opencode github run`

Execute OpenCode within a GitHub Actions workflow.

**Example** (`.github/workflows/opencode.yml`):
```yaml
name: OpenCode Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: sst/opencode@v1
        with:
          prompt: "Review this PR for security issues"
```

---

## Debug Commands

### `debug` - Debug Utilities

**Usage**: `opencode debug [subcommand]`

Debugging and diagnostic tools for OpenCode development.

#### `debug config` - Show Configuration

**Usage**: `opencode debug config`

Display current configuration with merged values.

**Output**:
```json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-***",
      "models": ["claude-3-5-sonnet", ...]
    }
  },
  "mcp": {
    "servers": {...}
  },
  "lsp": {...}
}
```

#### `debug file` - File Analysis

**Usage**: `opencode debug file <path>`

Analyze file for parsing, language detection, and content extraction.

**Arguments**:
- `<path>` - File to analyze

**Example**:
```bash
opencode debug file src/auth.ts
```

#### `debug lsp` - LSP Debugging

**Usage**: `opencode debug lsp [subcommand]`

Debug Language Server Protocol integration.

**Subcommands**:
- `symbols` - List workspace symbols
- `diagnostics <file>` - Get file diagnostics
- `hover <file> <line> <col>` - Get hover information

**Examples**:
```bash
opencode debug lsp symbols
opencode debug lsp diagnostics src/auth.ts
opencode debug lsp hover src/auth.ts 10 5
```

#### `debug ripgrep` - Search Testing

**Usage**: `opencode debug ripgrep <pattern>`

Test ripgrep search functionality.

**Arguments**:
- `<pattern>` - Search pattern

**Example**:
```bash
opencode debug ripgrep "TODO"
```

#### `debug scrap` - Cleanup Temporary Data

**Usage**: `opencode debug scrap`

Clean up temporary files and caches.

#### `debug snapshot` - Snapshot Inspection

**Usage**: `opencode debug snapshot <session-id>`

Inspect session snapshots for debugging.

---

## Global Flags

These flags work with any command:

### `--print-logs`
Print logs to stderr during execution.

```bash
opencode run "test" --print-logs
```

### `--log-level <level>`
Set logging level: `DEBUG`, `INFO`, `WARN`, `ERROR`

```bash
opencode serve --log-level DEBUG
```

### `--help`
Show command help.

```bash
opencode --help
opencode run --help
opencode auth --help
```

### `--version`, `-v`
Show version number.

```bash
opencode --version
# Output: 0.15.17
```

---

## Examples

### Daily Development Workflow

```bash
# Start OpenCode in project
cd ~/projects/my-app
opencode

# Or continue previous session
opencode --continue

# Use specific model for complex task
opencode -m anthropic/claude-3-opus-20240229
```

### Quick One-Off Tasks

```bash
# Quick question
opencode run "What's the time complexity of binary search?"

# Code review
opencode run -f auth.ts "Review for security issues"

# Continue refactoring
opencode run --continue "Now update the tests"
```

### Automation Scripts

```bash
#!/bin/bash
# auto-review.sh - Automated code review script

# Get all changed files
CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)

# Review with OpenCode
opencode run --format json \
  -f $CHANGED_FILES \
  "Review these changes for bugs, security issues, and style problems" \
  | jq -r '.text' \
  > review.md

echo "Review complete: review.md"
```

### Team Server Setup

```bash
# Start server on dedicated machine
opencode serve --hostname 0.0.0.0 --port 8080

# Connect from any team member
curl http://team-server:8080/project
```

### IDE Integration

```bash
# Use OpenCode as Zed agent
# (configured in ~/.config/zed/settings.json)
zed ~/projects/my-app  # OpenCode automatically available
```

### Custom Agent Workflow

```bash
# Create specialized agents
opencode agent create testing
# Edit .opencode/agent/testing.md with test-focused guidelines

# Use the agent
opencode --agent testing "Add tests for auth module"
```

---

## Environment Variables

OpenCode respects these environment variables:

### `OPENCODE`
Set to `"1"` by OpenCode itself to detect running inside OpenCode.

### `OPENCODE_INSTALL_DIR`
Custom installation directory.

### `OPENCODE_BIN_PATH`
Path to OpenCode binary (for development).

### `OPENCODE_AUTO_SHARE`
Auto-share sessions if set to `"1"`.

### `OPENCODE_DISABLE_AUTOUPDATE`
Disable automatic upgrades if set to `"1"`.

### Provider API Keys
```bash
ANTHROPIC_API_KEY=sk-ant-***
OPENAI_API_KEY=sk-***
GOOGLE_API_KEY=***
AWS_ACCESS_KEY_ID=***
AWS_SECRET_ACCESS_KEY=***
```

### Logging
```bash
OPENCODE_LOG_LEVEL=DEBUG
OPENCODE_PRINT_LOGS=1
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Error (file not found, invalid arguments, etc.) |
| 2 | Authentication required |
| 130 | Interrupted (Ctrl+C) |

---

## Tips & Tricks

### Use Shell Aliases

```bash
# ~/.bashrc or ~/.zshrc
alias oc="opencode run"
alias octui="opencode"
alias ocontinue="opencode --continue"

# Quick usage
oc "Add error handling"
ocontinue "Now add tests"
```

### Pipe Input

```bash
# Send file content
cat large-log.txt | opencode run "Summarize errors"

# Send command output
git log --oneline -10 | opencode run "Suggest better commit messages"

# Send multiple files
find . -name "*.ts" -exec cat {} \; | opencode run "Review all TypeScript files"
```

### Output Processing

```bash
# Extract just the text
opencode run --format json "Answer: 2+2" | jq -r '.text'

# Save to file
opencode run "Generate README" > README.md

# Process with other tools
opencode run "List security issues" | grep "CRITICAL"
```

### Session Management

```bash
# List recent sessions
ls ~/.opencode/data/projects/*/sessions/

# Continue specific session
opencode run --session sess_abc123 "Continue where we left off"

# Export and share
opencode export sess_abc123 --format markdown > session.md
```

---

## See Also

- **[00-overview.md](./00-overview.md)** - OpenCode overview
- **[01-architecture.md](./01-architecture.md)** - System architecture
- **[15-server-architecture.md](./15-server-architecture.md)** - HTTP API reference
- **[11-acp-protocol.md](./11-acp-protocol.md)** - ACP protocol details
- **[12-mcp-integration.md](./12-mcp-integration.md)** - MCP integration guide
- **[13-configuration.md](./13-configuration.md)** - Configuration system

