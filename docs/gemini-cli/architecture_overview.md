# Gemini CLI - Architecture Overview

**Generated:** 2024-10-24
**Purpose:** Comprehensive technical documentation of system architecture and core components

---

## Table of Contents

1. [System Architecture](#system-architecture)
2. [Package Structure](#package-structure)
3. [Core Components](#core-components)
4. [Data Flow](#data-flow)
5. [Technology Stack](#technology-stack)

---

## System Architecture

Gemini CLI is a sophisticated interactive command-line interface for Google's Gemini AI models, featuring a modular monorepo architecture with 5 main packages.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      User Interface Layer                    │
│  (@google/gemini-cli - CLI Package)                         │
│  - React/Ink-based Terminal UI                              │
│  - Keyboard bindings & input handling                       │
│  - Session management & history                             │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│                      Core Engine Layer                       │
│  (@google/gemini-cli-core)                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Tool System  │  │ Agent System │  │ MCP Client   │     │
│  │ Registry     │  │ Executor     │  │ Manager      │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Prompt       │  │ Shell Exec   │  │ Config       │     │
│  │ System       │  │ Service      │  │ Management   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│                    External Integrations                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ MCP Servers  │  │ VSCode IDE   │  │ A2A Server   │     │
│  │ (Stdio/HTTP) │  │ Companion    │  │ (Agent-2-    │     │
│  │              │  │              │  │  Agent)      │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│                      Gemini AI API                           │
│  - Model routing & selection                                │
│  - Streaming responses                                       │
│  - Function calling (tools)                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Package Structure

### 1. **@google/gemini-cli** (Main CLI Package)
- **Location:** `packages/cli/`
- **Entry Point:** `dist/index.js`
- **Bin Command:** `gemini`
- **Purpose:** User-facing CLI interface
- **Key Dependencies:**
  - `ink` v6.2.3 - React-based terminal UI
  - `yargs` v17.7.2 - CLI argument parsing
  - `react` v19.1.0 - UI rendering
  - `@google/gemini-cli-core` - Core functionality

### 2. **@google/gemini-cli-core** (Core Package)
- **Location:** `packages/core/`
- **Entry Point:** `dist/index.js`
- **Purpose:** Core business logic, tools, agents
- **Key Dependencies:**
  - `@google/genai` v1.16.0 - Gemini AI SDK
  - `@modelcontextprotocol/sdk` v1.11.0 - MCP integration
  - `@joshua.litt/get-ripgrep` v0.0.3 - Fast file search
  - `@xterm/headless` v5.5.0 - Terminal emulation
  - `web-tree-sitter` v0.25.10 - Code parsing
- **Lines of Code:** ~290 TypeScript files

### 3. **@google/gemini-cli-a2a-server** (Agent-to-Agent Server)
- **Location:** `packages/a2a-server/`
- **Entry Point:** `dist/index.js`
- **Bin Command:** `gemini-cli-a2a-server`
- **Purpose:** Agent-to-agent communication server
- **Key Dependencies:**
  - `@a2a-js/sdk` v0.3.2 - A2A protocol
  - `express` v5.1.0 - HTTP server
  - `@google-cloud/storage` v7.16.0 - GCS persistence

### 4. **gemini-cli-vscode-ide-companion** (VSCode Extension)
- **Location:** `packages/vscode-ide-companion/`
- **Entry Point:** `dist/extension.cjs`
- **Purpose:** VSCode integration for IDE features
- **Key Features:**
  - Diff editor integration
  - Workspace trust integration
  - IDE-specific commands
- **VSCode Engine:** `^1.99.0`

### 5. **@google/gemini-cli-test-utils** (Test Utilities)
- **Location:** `packages/test-utils/`
- **Purpose:** Shared test utilities and helpers

---

## Core Components

### 1. Tool System

**Architecture Pattern:** Declarative Tool Registry

```typescript
Tool Registration Flow:
User Request → Tool Builder → Tool Invocation → Execute → Result

Components:
- BaseDeclarativeTool: Abstract base class
- ToolRegistry: Central registration system
- ToolInvocation: Validated, executable tool instance
- ToolBuilder: Parameter validation & invocation creation
```

**Built-in Tools (13 core tools):**
1. `glob` - File pattern matching
2. `search_file_content` (grep) - Content search
3. `list_directory` (ls) - Directory listing
4. `read_file` - Read single file
5. `read_many_files` - Batch file reading
6. `replace` (edit) - File editing
7. `write_file` - File creation
8. `run_shell_command` - Shell execution
9. `google_web_search` - Web search
10. `web_fetch` - URL fetching
11. `save_memory` - User preference storage
12. `write_todos` - TODO list management (experimental)
13. `smart-edit` - AI-powered editing (experimental)

**Tool Discovery:**
- **MCP Tools:** Auto-discovered from MCP servers
- **Custom Tools:** Via `toolDiscoveryCommand` setting
- **Extension Tools:** Provided by extensions

### 2. MCP (Model Context Protocol) Integration

**Purpose:** Extensibility via external MCP servers

**Supported Transports:**
1. **Stdio** - Local process communication
2. **SSE** (Server-Sent Events) - HTTP streaming
3. **Streamable HTTP** - Bidirectional HTTP

**Features:**
- Auto-discovery of tools and prompts
- OAuth 2.0 authentication support
- Multi-server management
- Server status tracking
- Workspace context sharing

**MCP Client Manager:**
- Manages multiple MCP server connections
- Handles authentication flows
- Discovers tools and prompts dynamically
- Server lifecycle management (connect/disconnect/restart)

### 3. Agent System

**Purpose:** Autonomous task execution with specialized agents

**Architecture:**
```typescript
AgentDefinition → AgentExecutor → Tool Calls → complete_task
```

**Agent Types:**
1. **CodebaseInvestigatorAgent:**
   - Purpose: Deep codebase analysis
   - Max Turns: 15 (configurable)
   - Max Time: 5 minutes (configurable)
   - Tools: Read-only tools (ls, grep, glob, read_file, etc.)

**Agent Execution Flow:**
1. Initialize with inputs
2. Build system prompt + environment context
3. Loop: Call model → Execute tools → Check termination
4. Must call `complete_task` to finish
5. Return structured output

**Termination Modes:**
- `GOAL` - Task completed successfully
- `MAX_TURNS` - Exceeded max turn limit
- `TIMEOUT` - Exceeded time limit
- `ERROR` - Execution error
- `ABORTED` - User cancellation

### 4. Shell Execution Service

**Implementation:** Dual-mode execution

**Execution Methods:**
1. **PTY (Pseudo-Terminal) - Preferred:**
   - Libraries: `@lydell/node-pty` or `node-pty`
   - Features: Full terminal emulation, ANSI support
   - Platform: Cross-platform with fallback

2. **child_process - Fallback:**
   - Used when PTY unavailable
   - Limitations: No interactive shell support

**Features:**
- ANSI terminal emulation via `@xterm/headless`
- Binary detection and handling
- Output truncation (16MB limit)
- Process group management
- Streaming output support
- Abort signal handling

**Security:**
- Command validation
- Sandbox support (Docker/Podman/macOS Seatbelt)
- Permission confirmation system

### 5. Prompt System

**System Prompt Components:**
1. **Base Prompt:** Core instructions and workflows
2. **Environment Context:** CWD, directory structure
3. **User Memory:** Saved preferences via `save_memory` tool
4. **Conditional Sections:**
   - Git repository instructions
   - Sandbox warnings
   - Tool-specific guidance

**Dynamic Features:**
- Template string interpolation
- Conditional prompt sections
- Environment variable overrides (`GEMINI_SYSTEM_MD`)
- Compression for context management

**Prompt Configuration:**
- `GEMINI_SYSTEM_MD`: Override system prompt file path
- `GEMINI_WRITE_SYSTEM_MD`: Write generated prompt to file

### 6. Configuration System

**Configuration Layers (in precedence order):**
1. **CLI flags** (highest priority)
2. **Environment variables**
3. **Project config:** `.gemini/config.yaml` (workspace)
4. **User config:** `~/.config/gemini-cli/config.yaml`
5. **Extension configs:** Merged from installed extensions
6. **Defaults** (lowest priority)

**Merge Strategies:**
- `REPLACE` - Overwrite (default)
- `CONCAT` - Array concatenation
- `UNION` - Unique array merge
- `SHALLOW_MERGE` - Object merge

---

## Data Flow

### 1. User Request Flow

```
User Input
  ↓
Input Processor (@ commands, shell commands)
  ↓
Command Router
  ├→ /command → Built-in Command
  ├→ @file → File Inclusion
  └→ Query → Gemini AI
       ↓
    Tool Selection
       ↓
    Tool Execution (with confirmation)
       ↓
    Response Generation
       ↓
    UI Rendering
```

### 2. Tool Execution Flow

```
Tool Call Request
  ↓
Tool Registry Lookup
  ↓
Tool Builder (validation)
  ↓
Tool Invocation Creation
  ↓
Confirmation Check
  ├→ Auto-Accept (safe tools)
  ├→ Message Bus Policy
  └→ User Confirmation
       ↓
    Execute
  ├→ Success → Format Result
  └→ Error → Error Handling
       ↓
    Return to Model
```

### 3. MCP Tool Discovery Flow

```
Startup / /mcp restart
  ↓
Load MCP Server Configs
  ↓
For Each Server:
  ├→ Create Transport (Stdio/SSE/HTTP)
  ├→ Handle OAuth (if required)
  ├→ Connect Client
  ├→ Discover Tools
  ├→ Discover Prompts
  └→ Register Tools
       ↓
    Tool Registry Updated
```

---

## Technology Stack

### Core Technologies

**Runtime:**
- Node.js ≥20.0.0
- TypeScript 5.3.3

**UI Framework:**
- React 19.1.0
- Ink 6.2.3 (React for CLIs)

**AI/LLM:**
- @google/genai 1.16.0 (Gemini AI SDK)
- Streaming API support
- Function calling (tools)

**Protocols:**
- MCP (Model Context Protocol) SDK 1.11.0
- A2A (Agent-to-Agent) SDK 0.3.2

**Terminal:**
- @xterm/headless 5.5.0
- @lydell/node-pty 1.1.0
- ANSI color support

**Code Analysis:**
- web-tree-sitter 0.25.10
- tree-sitter-bash 0.25.0
- @joshua.litt/get-ripgrep 0.0.3

**Build & Development:**
- esbuild 0.25.0
- Vitest 3.1.1
- ESLint 9.24.0
- Prettier 3.5.3

### Optional Dependencies

**PTY Libraries (Platform-specific):**
- @lydell/node-pty-darwin-arm64
- @lydell/node-pty-darwin-x64
- @lydell/node-pty-linux-x64
- @lydell/node-pty-win32-arm64
- @lydell/node-pty-win32-x64
- node-pty 1.0.0 (fallback)

### Telemetry & Monitoring

- @google-cloud/logging 11.2.1
- OpenTelemetry SDK
- Cloud Monitoring Exporter
- Cloud Trace Exporter

---

## Security Architecture

### Multi-Layer Security

1. **Path Validation:**
   - Workspace boundary checks
   - Symlink resolution
   - Directory traversal prevention

2. **Command Validation:**
   - Whitelisted commands
   - Argument sanitization
   - No shell interpretation (uses `spawn`)

3. **Path Filtering:**
   - Auto-blocks sensitive files (.env, credentials, keys)
   - Respects .gitignore / .geminiignore
   - Blocks dependencies (node_modules, vendor)

4. **Sandbox Support:**
   - Docker container execution
   - Podman support
   - macOS Seatbelt profiles

5. **Folder Trust System:**
   - Trust levels: TRUST, DO_NOT_TRUST
   - Inherited trust from parent folders
   - IDE workspace integration

---

## Performance Characteristics

### Resource Limits

- **Tool Output:** 16MB max per tool execution
- **MCP Timeout:** 10 minutes default
- **Shell Timeout:** 30 seconds default
- **Discovery Max Dirs:** 200 (memory discovery)

### Optimization Features

- **Parallel Tool Execution:** Independent tools run concurrently
- **Streaming Output:** Real-time response rendering
- **Context Compression:** Automatic history compression
- **Ripgrep Integration:** Fast file content search
- **Tool Output Truncation:** Configurable line/char limits

---

## Extension System

### Extension Architecture

**Extension Format:**
```
extension-name/
  ├── gemini-extension.json  # Manifest
  ├── commands/              # Custom commands
  └── .gemini-cli-install-metadata.json  # Install info
```

**Extension Capabilities:**
1. MCP Server Registration
2. Tool Exclusions
3. Context File Names
4. Custom Settings

**Extension Management:**
- `/extensions install <source>` - Install from GitHub/NPM/local
- `/extensions update <name>` - Update extension
- `/extensions list` - List installed
- `/extensions uninstall <name>` - Remove extension

---

## Deployment Modes

### 1. Standalone CLI
Standard terminal usage with full features

### 2. IDE Integration (VSCode)
- Workspace-aware operations
- Diff editor integration
- Trust system integration

### 3. A2A Server Mode
- HTTP API for agent communication
- Task persistence (GCS)
- Multi-agent coordination

### 4. Sandboxed Execution
- Docker container
- Podman
- macOS Seatbelt (sandbox-exec)

---

## Key Design Patterns

1. **Declarative Tool System:** Separation of validation and execution
2. **Message Bus Pattern:** Pub/sub for tool confirmations
3. **Builder Pattern:** Tool invocation creation
4. **Registry Pattern:** Centralized tool/prompt management
5. **Strategy Pattern:** Multiple execution strategies (PTY/child_process)
6. **Repository Pattern:** Config/extension management

---

## Experimental Features

Features that may change or be removed:

1. **Codebase Investigator Agent** - Autonomous code analysis
2. **Model Router** - Automatic model selection based on complexity
3. **Write TODOs Tool** - TODO list management
4. **Smart Edit** - AI-powered file editing
5. **Prompt Completion** - AI-powered prompt suggestions

---

This architecture supports a highly extensible, secure, and performant CLI experience for interacting with Gemini AI models while maintaining strict security boundaries and flexible configuration.

