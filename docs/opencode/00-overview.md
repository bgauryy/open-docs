# OpenCode - Technical Overview

> **Comprehensive technical documentation for OpenCode - the open-source AI coding agent built for the terminal**

---

## What is OpenCode?

**OpenCode** is a 100% open-source, provider-agnostic AI coding agent specifically built for terminal users. It's designed by neovim users and the creators of [terminal.shop](https://terminal.shop) to push the limits of what's possible in the terminal interface.

**Tagline**: *"The AI coding agent built for the terminal."*

### Current Version
- **Version**: 0.15.17
- **Package**: `opencode-ai` (npm)
- **Repository**: https://github.com/sst/opencode
- **Website**: https://opencode.ai
- **Discord**: https://opencode.ai/discord

---

## Key Differentiators

### 1. **100% Open Source**
Unlike proprietary alternatives, OpenCode is completely open-source under the MIT license, allowing full transparency and community contributions.

### 2. **Provider-Agnostic Architecture**
Not coupled to any single AI provider. While Anthropic is recommended, OpenCode works with:
- **Anthropic** (Claude models)
- **OpenAI** (GPT models)
- **Google** (Gemini via Vertex AI)
- **Amazon Bedrock** (various models)
- **Local models** (via compatible APIs)

As models evolve and pricing drops, being provider-agnostic ensures OpenCode remains flexible and cost-effective.

### 3. **Out-of-the-Box LSP Support**
Built-in Language Server Protocol (LSP) integration provides:
- Real-time diagnostics
- Hover information
- Code intelligence
- Multi-language support

### 4. **Terminal-First Design**
A dedicated focus on Terminal User Interface (TUI) excellence:
- Rich interactive terminal experience
- Optimized for keyboard-driven workflows
- Designed for developers who live in the terminal
- No GUI dependencies required

### 5. **Client/Server Architecture**
Unique decoupled design where:
- **Server** runs on your machine (handles AI, file operations, state)
- **Clients** can connect from anywhere (TUI, desktop app, mobile app)
- Multiple clients can interact with the same server
- The TUI frontend is just one possible client

This architecture enables scenarios like:
- Running OpenCode on a powerful remote machine while controlling it from a lightweight client
- Mobile app control of your development environment
- Multiple team members collaborating through the same OpenCode instance

---

## Comparison with Claude Code

| Feature | OpenCode | Claude Code |
|---------|----------|-------------|
| **License** | Open Source (MIT) | Proprietary |
| **AI Provider** | Provider-agnostic | Anthropic only |
| **LSP Support** | Built-in | Limited |
| **Architecture** | Client/Server | Monolithic |
| **TUI Focus** | Primary interface | Secondary |
| **Extensibility** | Fully extensible | Limited |
| **Self-Hosting** | Yes | No |
| **Local Models** | Supported | Not supported |

OpenCode is very similar to Claude Code in terms of capability but with fundamental architectural differences that enable greater flexibility and community-driven development.

---

## Technology Stack

### Core Technologies

#### **Runtime & Language**
- **Bun 1.3+** - Primary JavaScript runtime (faster, modern alternative to Node.js)
- **TypeScript 5.8.2** - Type-safe development
- **Go 1.24.x** - TUI implementation (being migrated to TypeScript)

#### **Framework & Build**
- **Vite 7.1.4** - Build tool for desktop and web applications
- **SST (ServerlessStack)** - Infrastructure and deployment

#### **AI & LLM Integration**
- **Vercel AI SDK** (`ai` package) - Unified AI provider interface
- **@ai-sdk/amazon-bedrock** - Bedrock integration
- **@ai-sdk/google-vertex** - Google Vertex AI integration
- Built-in Anthropic and OpenAI support

#### **Protocols & Standards**
- **@agentclientprotocol/sdk** - Agent Client Protocol (ACP) for IDE integration
- **@modelcontextprotocol/sdk** - Model Context Protocol (MCP) for extensibility
- **vscode-jsonrpc** - JSON-RPC communication
- **vscode-languageserver-types** - LSP type definitions

#### **Code Intelligence**
- **tree-sitter** & **tree-sitter-bash** - Syntax tree parsing
- **web-tree-sitter** - Browser-compatible parser

#### **File Operations**
- **@parcel/watcher** - Fast file watching
- **chokidar** - File watching fallback
- **ignore** - .gitignore pattern matching
- **minimatch** - Glob pattern matching
- **fuzzysort** - Fuzzy file searching

#### **UI Frameworks**
- **SolidJS 1.9.9** - Desktop application UI (reactive framework)
- **Astro** - Documentation website (packages/web)
- **@kobalte/core** - UI primitives for SolidJS
- **TailwindCSS 4.1.11** - Utility-first CSS
- **@clack/prompts** - Beautiful terminal prompts

#### **HTTP & APIs**
- **Hono 4.7.10** - Fast web framework for the server
- **@hono/zod-validator** - Request validation
- **hono-openapi** - OpenAPI specification generation

#### **Data Processing**
- **zod 4.1.8** - Schema validation
- **diff 8.0.2** - Diff algorithm for file changes
- **@pierre/precision-diffs** - Enhanced diff precision
- **decimal.js** - Arbitrary-precision decimal math
- **remeda 2.26.0** - TypeScript utility library
- **luxon** - Date/time handling

#### **Utilities**
- **yargs** - CLI argument parsing
- **gray-matter** - Markdown frontmatter parsing
- **turndown** - HTML to Markdown conversion
- **jsonc-parser** - JSON with comments support
- **ulid** - Unique ID generation
- **xdg-basedir** - XDG Base Directory Specification

#### **Authentication**
- **@openauthjs/openauth** - Authentication framework
- **@actions/core** & **@actions/github** - GitHub Actions integration
- **@octokit/rest** & **@octokit/graphql** - GitHub API clients

---

## Architecture Overview

### Monorepo Structure

OpenCode is organized as a monorepo with multiple packages:

```
packages/
├── opencode/          # Core CLI and server (114+ TypeScript files)
├── desktop/           # Desktop application (SolidJS)
├── console/           # Web console and management
│   ├── app/          # Frontend application
│   ├── core/         # Business logic and database
│   ├── function/     # Serverless functions
│   ├── mail/         # Email templates
│   └── resource/     # Cloud resources
├── tui/              # Terminal UI (Go - being deprecated)
├── web/              # Documentation website (Astro/Starlight)
├── ui/               # Shared UI components
├── sdk/              # SDKs for integration
│   ├── go/          # Go SDK
│   └── js/          # JavaScript SDK
├── plugin/           # Plugin system
├── slack/            # Slack integration
├── function/         # Shared functions
├── identity/         # Identity management
└── script/           # Build and utility scripts
```

### Core Components

#### **Server** (`packages/opencode/src/server/`)
- HTTP server with Hono framework
- RESTful API for project and session management
- WebSocket support for real-time updates
- Multi-project support with isolated workspaces

#### **Session Management** (`packages/opencode/src/session/`)
- Conversation state tracking
- Message history with v1 and v2 formats
- Compaction for long-running sessions
- Lock mechanism for concurrent safety
- Retry and revert capabilities

#### **Prompt System** (`packages/opencode/src/session/prompt.ts`)
- 54KB of sophisticated prompt construction logic
- Context assembly from files, LSP data, and history
- System prompt integration
- Tool description generation

#### **Tool System** (`packages/opencode/src/tool/`)
- Pluggable tool architecture
- Registry-based tool discovery
- Built-in tools for file operations, search, execution, LSP, patches, tasks, and web access

#### **LSP Integration** (`packages/opencode/src/lsp/`)
- Language server client and server management
- Multi-language support detection
- Diagnostics and hover information
- Integration with coding tools

#### **Provider System** (`packages/opencode/src/provider/`)
- Unified interface for multiple AI providers
- Model registry with configuration
- Response transformation and streaming
- Automatic fallback and retry logic

#### **Configuration** (`packages/opencode/src/config/`)
- 28KB configuration system
- Global and project-specific settings
- Provider credentials management
- Environment variable support

#### **ACP Protocol** (`packages/opencode/src/acp/`)
- Agent Client Protocol implementation for IDE integration
- Enables Zed, VS Code, and other IDEs to use OpenCode as an agent
- Session lifecycle management
- Client capability negotiation

#### **MCP Integration** (`packages/opencode/src/mcp/`)
- Model Context Protocol support
- Extensibility through MCP servers
- Custom tool and resource exposure

---

## Quick Start for Developers

### Installation

```bash
# Using package managers
npm i -g opencode-ai@latest        # npm
bun i -g opencode-ai@latest        # bun
pnpm i -g opencode-ai@latest       # pnpm
yarn global add opencode-ai@latest # yarn

# Using system package managers
brew install sst/tap/opencode      # macOS/Linux
scoop install extras/opencode       # Windows
choco install opencode              # Windows
paru -S opencode-bin                # Arch Linux

# Using the install script (YOLO)
curl -fsSL https://opencode.ai/install | bash
```

### Development Setup

Requirements:
- Bun 1.3.0+
- Go 1.24.0+ (for TUI, soon to be removed)
- Node.js 22+ (for some packages)

```bash
# Clone the repository
git clone https://github.com/sst/opencode.git
cd opencode

# Install dependencies
bun install

# Start development server
bun dev
```

### Basic Usage

```bash
# Interactive session in current directory
opencode

# Run with specific prompt
opencode "Add error handling to auth.ts"

# Start TUI interface
opencode tui

# Start server mode
opencode serve

# Agent Client Protocol (for IDE integration)
opencode acp

# Model Context Protocol
opencode mcp

# View available models
opencode models

# Authenticate with providers
opencode auth login
```

---

## Key Features

### 1. **Multiple Interface Modes**
- **Interactive CLI** - Direct command-line prompting
- **TUI** - Full-screen terminal interface
- **Server Mode** - HTTP API for custom clients
- **ACP Mode** - IDE agent integration
- **Desktop App** - Native-feeling GUI (SolidJS)

### 2. **Advanced File Operations**
- Precise code editing with diff-based patching
- Multi-file editing support
- Ripgrep-powered search
- FZF fuzzy finding
- File watching for real-time updates
- Respect for .gitignore patterns

### 3. **Intelligent Code Understanding**
- LSP-based code intelligence
- Syntax tree analysis via tree-sitter
- Diagnostics and error detection
- Hover information and type inference
- Symbol navigation

### 4. **Flexible Provider Support**
- Dynamic model switching
- Cost-aware model selection
- Streaming response support
- Rate limiting and retry logic
- Local model compatibility

### 5. **Rich Tool Ecosystem**
- File read/write/edit/multiedit
- Bash command execution
- LSP diagnostics and hover
- Task and TODO management
- Web search and fetch
- Patch application and review

### 6. **Multi-Project Management**
- Multiple simultaneous projects
- Per-project configuration
- Isolated sessions per project
- Different worktrees support

### 7. **Session Management**
- Persistent conversation history
- Session compaction for long chats
- Revert and retry capabilities
- Session sharing and export
- Lock-based concurrency control

---

## Use Cases

### **Solo Development**
- Interactive coding assistant
- Code review and refactoring
- Bug fixing and debugging
- Documentation generation
- Test writing

### **Team Collaboration**
- Shared OpenCode server
- Multiple clients connecting
- Consistent coding assistant across team
- Code style enforcement
- Knowledge sharing

### **IDE Integration**
- Zed IDE agent
- VS Code extension (via ACP)
- JetBrains integration potential
- Any ACP-compatible editor

### **CI/CD Automation**
- GitHub Actions integration
- Automated code review
- Test generation
- Documentation updates

### **Custom Workflows**
- HTTP API for custom tools
- MCP servers for domain-specific tasks
- Plugin development
- Custom client applications

---

## What Makes OpenCode Different?

### **Developer-First Philosophy**
Built by terminal users for terminal users. Every design decision prioritizes:
- Keyboard-driven workflows
- Fast, responsive interactions
- Minimal cognitive load
- Maximum transparency

### **Architectural Flexibility**
The client/server split enables:
- Remote development scenarios
- Resource-intensive operations on powerful machines
- Lightweight clients on any device
- Future-proof extensibility

### **Community-Driven**
Open-source nature means:
- Full transparency in how it works
- Community contributions welcome
- No vendor lock-in
- Privacy and security you can verify

### **Protocol-First Integration**
Support for ACP and MCP means:
- Standards-based extensibility
- IDE integration without custom code
- Tool ecosystem growth
- Future protocol compatibility

---

## Documentation Roadmap

This documentation suite covers:

1. **Architecture** - System design, components, patterns
2. **CLI Reference** - Commands, flags, examples
3. **Session Management** - Lifecycle, state, persistence
4. **Prompt System** - Construction, context, templates
5. **Tool System** - Architecture, built-in tools, development
6. **LSP Integration** - Language servers, diagnostics, configuration
7. **Provider System** - AI providers, models, configuration
8. **Protocols** - ACP and MCP implementation details
9. **Configuration** - Settings, environment, customization
10. **Security** - Permissions, sandboxing, safety
11. **Server Architecture** - API, endpoints, WebSocket
12. **Project Management** - Multi-project support, workspaces
13. **User Interfaces** - TUI, desktop, web console
14. **File System** - Operations, watching, searching
15. **Utilities** - Logging, errors, helpers
16. **Authentication** - Providers, tokens, credentials
17. **SDKs** - Go, JavaScript, plugin development
18. **Development Guide** - Contributing, testing, building
19. **Flow Diagrams** - Visual system flows

---

## Community & Support

- **Discord**: https://opencode.ai/discord
- **GitHub Issues**: https://github.com/sst/opencode/issues
- **Documentation**: https://opencode.ai/docs
- **X/Twitter**: https://x.com/opencode

---

## Next Steps

- **[01-architecture.md](./01-architecture.md)** - Dive into system architecture
- **[02-cli-reference.md](./02-cli-reference.md)** - Explore all CLI commands
- **[24-development-guide.md](./24-development-guide.md)** - Start contributing

---

**OpenCode** - Open source, provider-agnostic, terminal-first AI coding for everyone.

