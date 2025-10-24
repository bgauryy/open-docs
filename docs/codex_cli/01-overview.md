# Codex CLI - Overview

## Table of Contents
- [What is Codex CLI](#what-is-codex-cli)
- [Key Capabilities](#key-capabilities)
- [Architecture Philosophy](#architecture-philosophy)
- [System Requirements](#system-requirements)
- [Quick Start Guide](#quick-start-guide)

---

## What is Codex CLI

Codex CLI is an **open-source, terminal-based coding agent** developed by OpenAI. It's a sophisticated AI-powered assistant that can understand natural language prompts, execute code, manipulate files, and iterate on tasks autonomously—all while running securely sandboxed on your local machine.

### Key Characteristics

- **Native Rust Implementation**: High-performance, memory-safe core written in Rust
- **Terminal-First**: Designed for developers who live in the command line
- **Streaming & Async**: Real-time response streaming with async I/O throughout
- **Multi-Provider**: Works with OpenAI, Azure, Gemini, Ollama, and other compatible APIs
- **Safety-First**: Multiple layers of sandboxing and approval mechanisms
- **Extensible**: Plugin system via MCP (Model Context Protocol), custom prompts, and project documentation

### Legacy Note

> **Important**: This documentation covers the current **Rust implementation** (`codex-rs/`). The older TypeScript implementation (`codex-cli/`) has been superseded. The Node.js wrapper at `codex-cli/bin/codex.js` is minimal and only spawns the native Rust binary.

---

## Key Capabilities

### 1. Natural Language Interaction

```bash
codex "fix the lint errors in this project"
codex "add dark mode to the settings page"
codex "explain what this regex does: ^(?=.*[A-Z]).{8,}$"
```

The agent understands context, reads files, executes commands, and iterates until complete.

### 2. Autonomous Task Execution

Three approval modes control autonomy:

| Mode | What Agent May Do Without Asking |
|------|----------------------------------|
| **Suggest** (default) | Read any file in the repo |
| **Auto Edit** | Read AND apply patches to files |
| **Full Auto** | Read/write files + execute shell commands (sandboxed) |

### 3. File Operations

- **Read files** with smart chunking
- **Apply patches** using unified diff format
- **Search code** with grep/ripgrep integration
- **List directories** with filtering
- **Track changes** across turns

### 4. Shell Command Execution

- PTY-based interactive command execution
- Sandboxed by default (network disabled, directory restricted)
- Real-time output streaming
- Timeout and resource limits

### 5. Code Review & Analysis

```bash
codex "review this PR for bugs and security issues"
```

Specialized review mode with structured finding output.

### 6. Planning & Progress Tracking

Built-in planning tool for complex multi-step tasks:
- Break down tasks into steps
- Track progress in real-time
- Visualize completion status

### 7. Project Context Awareness

- Automatically discovers and loads `AGENTS.md` files
- Detects git repository and includes relevant context
- Understands project structure
- Loads custom user prompts from `~/.codex/prompts/`

### 8. Multi-Provider Support

Compatible with any OpenAI API-compatible endpoint:
- OpenAI (GPT-4, o1, o3, o4-mini)
- Azure OpenAI
- Google Gemini
- Ollama (local models)
- Anthropic (via compatible proxies)
- And more...

---

## Architecture Philosophy

### Design Principles

1. **Safety Through Layers**: Multiple independent security mechanisms
2. **Streaming Everything**: Async/streaming architecture for responsiveness
3. **Explicit Over Implicit**: Clear tool boundaries, no "magic"
4. **Extensibility**: Plugin architecture via MCP
5. **Developer Experience**: Built for terminal users, keyboard-first

### Core Architectural Decisions

**Why Rust?**
- Memory safety without garbage collection
- Excellent async/await support (tokio)
- Fast compilation and execution
- Strong type system catches errors at compile time

**Why Streaming?**
- Immediate feedback to users
- Better UX for long-running operations
- Efficient memory usage
- Natural fit for LLM response streaming

**Why Tool-Based?**
- Clear security boundaries
- Easy to audit and control
- LLM can't directly execute arbitrary code
- Composable and extensible

---

## System Requirements

### Operating Systems

| OS | Support Level | Sandboxing |
|----|---------------|------------|
| **macOS 12+** | Full support | Apple Seatbelt |
| **Linux** (Ubuntu 20.04+, Debian 10+) | Full support | Landlock (optional Docker) |
| **Windows 11** | Via WSL2 | Linux sandboxing in WSL |

### Runtime Requirements

- **Node.js**: 16+ (for wrapper only, 20 LTS recommended)
- **Git**: 2.23+ (optional but recommended)
- **Memory**: 4GB minimum, 8GB recommended

### Build Requirements (for development)

- **Rust**: Latest stable (see `rust-toolchain.toml`)
- **Cargo**: Bundled with Rust
- **System Libraries**: Standard build tools (gcc/clang, pkg-config)

---

## Quick Start Guide

### Installation

**From npm (Recommended):**

```bash
npm install -g @openai/codex
```

**From source:**

```bash
git clone https://github.com/openai/codex.git
cd codex/codex-rs
cargo build --release
# Binary at: target/release/codex
```

### Configuration

Set your API key:

```bash
export OPENAI_API_KEY="your-api-key-here"
```

Or create `~/.codex/config.yaml`:

```yaml
model: o4-mini
approvalMode: suggest
providers:
  openai:
    name: OpenAI
    baseURL: https://api.openai.com/v1
    envKey: OPENAI_API_KEY
```

### Basic Usage

**Interactive mode:**

```bash
codex
```

**With initial prompt:**

```bash
codex "create a todo list app"
```

**Non-interactive (quiet mode):**

```bash
codex -q "update the changelog"
```

**Full auto mode:**

```bash
codex --approval-mode full-auto "run tests and fix any failures"
```

### First Steps

1. **Test basic functionality:**
   ```bash
   codex "show me the files in this directory"
   ```

2. **Try a simple edit:**
   ```bash
   codex "add a comment to the main function explaining what it does"
   ```

3. **Use project documentation:**
   - Create `AGENTS.md` in your project root
   - Add project-specific instructions
   - Codex will automatically load them

4. **Create custom prompts:**
   ```bash
   mkdir -p ~/.codex/prompts
   echo "Review this code for performance issues" > ~/.codex/prompts/perf.md
   # Use with: /perf
   ```

---

## Next Steps

- **Architecture**: Read [02-architecture.md](./02-architecture.md) for system design
- **Prompts**: See [05-system-prompts.md](./05-system-prompts.md) for customization
- **Security**: Review [07-security-sandboxing.md](./07-security-sandboxing.md) for safety model
- **Configuration**: Check [08-configuration.md](./08-configuration.md) for all options

---

## Resources

- **GitHub Repository**: https://github.com/openai/codex
- **Documentation**: See the `docs/` directory
- **Community**: GitHub Discussions and Issues
- **License**: Apache 2.0

