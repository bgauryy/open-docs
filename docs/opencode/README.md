# OpenCode - Technical Documentation

> **Comprehensive technical documentation for OpenCode - the open-source, provider-agnostic AI coding agent**
> 
> **Version**: 0.15.17

---

## 📚 Documentation Suite

This documentation provides deep technical insights into OpenCode's architecture, implementation, and development. It complements the [official user documentation](https://opencode.ai/docs) by focusing on how OpenCode works internally.

**Total Coverage**: 27 documents • 310+ pages • 520+ code examples • 100+ diagrams

---

## 🚀 Quick Start

### For New Users
1. **[00-overview.md](./00-overview.md)** - What is OpenCode and why it matters
2. **[02-cli-reference.md](./02-cli-reference.md)** - Command-line usage and examples
3. **[05-system-prompts.md](./05-system-prompts.md)** - Customize with AGENTS.md

### For Developers
1. **[01-architecture.md](./01-architecture.md)** - System design and patterns
2. **[06-tool-system.md](./06-tool-system.md)** - Build custom tools
3. **[24-development-guide.md](./24-development-guide.md)** - Contributing guide

### For Integrators
1. **[11-acp-protocol.md](./11-acp-protocol.md)** - IDE integration via ACP
2. **[12-mcp-integration.md](./12-mcp-integration.md)** - Extend with MCP servers
3. **[15-server-architecture.md](./15-server-architecture.md)** - HTTP API reference

---

## 📖 Complete Documentation Index

### Core System (00-02)

| Document | Description | Best For |
|----------|-------------|----------|
| **[00-overview.md](./00-overview.md)** | Project introduction, features, tech stack | Understanding OpenCode's vision |
| **[01-architecture.md](./01-architecture.md)** | Client/server model, components, patterns | System design comprehension |
| **[02-cli-reference.md](./02-cli-reference.md)** | All CLI commands with examples | Daily usage and automation |

### Sessions & Prompts (03-05)

| Document | Description | Best For |
|----------|-------------|----------|
| **[03-session-management.md](./03-session-management.md)** | Session lifecycle, state, persistence | Understanding conversations |
| **[04-prompt-processing.md](./04-prompt-processing.md)** | Context assembly, prompt construction | Optimizing AI interactions |
| **[05-system-prompts.md](./05-system-prompts.md)** | AGENTS.md files, custom instructions | Customizing behavior |

### Tools & Execution (06-08)

| Document | Description | Best For |
|----------|-------------|----------|
| **[06-tool-system.md](./06-tool-system.md)** | Tool architecture, registry, permissions | Building custom tools |
| **[07-tool-implementations.md](./07-tool-implementations.md)** | All 14 built-in tools with examples | Using tools effectively |
| **08-lsp-integration.md** | Language Server Protocol integration | Code intelligence features |

### Providers & AI (09-10)

| Document | Description | Best For |
|----------|-------------|----------|
| **09-provider-system.md** | Provider abstraction, model registry | Multi-provider support |
| **10-llm-integration.md** | AI SDK usage, streaming, tokens | LLM communication details |

### Protocols & Integration (11-12)

| Document | Description | Best For |
|----------|-------------|----------|
| **11-acp-protocol.md** | Agent Client Protocol for IDEs | IDE integration (Zed, VS Code) |
| **12-mcp-integration.md** | Model Context Protocol servers | Extending capabilities |

### Configuration & Security (13-14)

| Document | Description | Best For |
|----------|-------------|----------|
| **13-configuration.md** | Config files, env vars, settings | Setup and customization |
| **14-security-permissions.md** | Permission system, sandboxing | Security and safety |

### Server & Projects (15-16)

| Document | Description | Best For |
|----------|-------------|----------|
| **15-server-architecture.md** | HTTP API, endpoints, WebSocket | Server development |
| **16-project-management.md** | Multi-project support, workspaces | Managing multiple projects |

### User Interfaces (17-19)

| Document | Description | Best For |
|----------|-------------|----------|
| **17-tui-implementation.md** | Terminal UI (Go-based) | TUI development |
| **18-desktop-application.md** | Desktop app (SolidJS) | GUI development |
| **19-web-console.md** | Web console, database, auth | Web interface |

### Developer Resources (20-23)

| Document | Description | Best For |
|----------|-------------|----------|
| **20-file-system.md** | File ops, watching, searching | File manipulation |
| **21-utilities-helpers.md** | Logging, errors, locks | Utility functions |
| **22-authentication.md** | Auth flows, token management | Provider authentication |
| **23-sdks-integrations.md** | Go/JS SDKs, plugins, GitHub Action | Integration development |

### Development & Flows (24-26)

| Document | Description | Best For |
|----------|-------------|----------|
| **24-development-guide.md** | Setup, building, testing, contributing | Contributors |
| **25-flow-diagrams.md** | Visual flow references for all systems | Understanding execution paths |
| **[26-resource-memory-management.md](./26-resource-memory-management.md)** | Memory optimization, resource cleanup, OS patterns | Performance optimization |

---

## 🎯 Documentation by Use Case

### I Want to Use OpenCode

**Getting Started**:
1. Install: See [00-overview.md](./00-overview.md#installation)
2. Commands: See [02-cli-reference.md](./02-cli-reference.md)
3. Customize: See [05-system-prompts.md](./05-system-prompts.md)

**Daily Workflow**:
- Run interactive: `opencode` or `opencode tui`
- Quick tasks: `opencode run "your prompt"`
- Continue session: `opencode --continue`

### I Want to Extend OpenCode

**Custom Tools**:
1. Learn tool system: [06-tool-system.md](./06-tool-system.md)
2. See examples: [07-tool-implementations.md](./07-tool-implementations.md)
3. Create in `.opencode/tool/mytool.ts`

**MCP Servers**:
1. Understand MCP: [12-mcp-integration.md](./12-mcp-integration.md)
2. Add server: `opencode mcp add <package>`
3. Configure in `.opencode/config.json`

### I Want to Integrate OpenCode

**IDE Integration (ACP)**:
1. Protocol spec: [11-acp-protocol.md](./11-acp-protocol.md)
2. Add to Zed: Update `~/.config/zed/settings.json`
3. Add to VS Code: Configure ACP extension

**API Integration**:
1. Start server: `opencode serve --port 8080`
2. API reference: [15-server-architecture.md](./15-server-architecture.md)
3. Use HTTP/WebSocket endpoints

### I Want to Contribute

**Development Setup**:
1. Requirements: Bun 1.3.0+, Go 1.24.0+, Node >=22
2. Install: `bun install`
3. Run: `bun dev`
4. Guide: [24-development-guide.md](./24-development-guide.md)

**Code Contribution**:
1. Architecture: [01-architecture.md](./01-architecture.md)
2. Code style: See AGENTS.md files
3. Testing: `bun test`
4. Submit PR with clear description

---

## 🎓 Learning Paths

### Path 1: User → Power User

**Goal**: Master OpenCode for daily coding

**Steps**:
1. [00-overview.md](./00-overview.md) - Understand capabilities
2. [02-cli-reference.md](./02-cli-reference.md) - Learn all commands
3. [05-system-prompts.md](./05-system-prompts.md) - Create AGENTS.md
4. [07-tool-implementations.md](./07-tool-implementations.md) - Master tools
5. [13-configuration.md](./13-configuration.md) - Optimize config

**Time**: 2-3 hours

### Path 2: Developer → Contributor

**Goal**: Contribute code to OpenCode

**Steps**:
1. [01-architecture.md](./01-architecture.md) - System design
2. [03-session-management.md](./03-session-management.md) - Core abstractions
3. [04-prompt-processing.md](./04-prompt-processing.md) - Prompt system
4. [06-tool-system.md](./06-tool-system.md) - Tool architecture
5. [24-development-guide.md](./24-development-guide.md) - Dev setup

**Time**: 4-6 hours

### Path 3: Integrator → Builder

**Goal**: Build on OpenCode platform

**Steps**:
1. [11-acp-protocol.md](./11-acp-protocol.md) - IDE integration
2. [12-mcp-integration.md](./12-mcp-integration.md) - MCP servers
3. [15-server-architecture.md](./15-server-architecture.md) - HTTP API
4. [23-sdks-integrations.md](./23-sdks-integrations.md) - SDKs
5. [06-tool-system.md](./06-tool-system.md) - Custom tools

**Time**: 3-5 hours

---

## 🔍 Key Topics

### Architecture & Design

**Client/Server Model**:
- Server handles AI, files, state
- Multiple clients possible (TUI, desktop, web, IDE)
- Protocol-based communication (HTTP, WebSocket, JSON-RPC)

**Key Patterns**:
- Namespace organization
- Event-driven architecture
- Provider abstraction
- Tool registry system

**Technology Stack**:
- Bun 1.3.0+ (runtime)
- TypeScript 5.8.2 (language)
- Hono 4.7.10 (HTTP framework)
- SolidJS 1.9.9 (desktop UI)
- Zod 4.1.8 (validation)
- Go 1.24.0+ (TUI, SDK)

### Core Features

**Provider-Agnostic AI**:
- Anthropic (Claude)
- OpenAI (GPT)
- Google (Gemini)
- Amazon Bedrock
- Local models

**LSP Integration**:
- Real-time diagnostics
- Hover information
- Multi-language support
- Auto-configuration

**Tool System**:
- 14 built-in tools
- Custom tool support
- Plugin architecture
- Permission control

**Multi-Project**:
- Isolated sessions
- Per-project config
- Multiple worktrees
- Concurrent projects

---

## 📊 Documentation Statistics

- **Total Documents**: 27
- **Total Lines**: ~15,500
- **Code Examples**: 520+
- **Diagrams**: 100+
- **API Endpoints**: 30+
- **CLI Commands**: 15+
- **Built-in Tools**: 13
- **Protocols**: 2 (ACP, MCP)

---

## 📦 Version Requirements

**OpenCode Version**: 0.15.17

**Runtime Requirements**:
- **Bun**: 1.3.0 or higher
- **Node.js**: 22 or higher (for some packages)
- **Go**: 1.24.0 or higher (for TUI/SDK)

**Key Dependencies**:
- TypeScript: 5.8.2
- Hono: 4.7.10
- AI SDK: 5.0.8
- SolidJS: 1.9.9
- Zod: 4.1.8

---

## 🔗 External Resources

**Official**:
- Website: https://opencode.ai
- GitHub: https://github.com/sst/opencode
- Discord: https://opencode.ai/discord
- Docs: https://opencode.ai/docs

**Community**:
- X/Twitter: https://x.com/opencode
- Discussions: GitHub Discussions
- Issues: GitHub Issues

**Related Projects**:
- SST: https://sst.dev
- Terminal Shop: https://terminal.shop
- ACP Spec: https://agentclientprotocol.com
- MCP Spec: https://modelcontextprotocol.io

---

## 💡 Tips for Reading

**First Time**:
- Start with [00-overview.md](./00-overview.md)
- Read [01-architecture.md](./01-architecture.md) for big picture
- Jump to topics that interest you

**Reference Use**:
- Use browser search (Cmd/Ctrl+F)
- Check diagrams in [25-flow-diagrams.md](./25-flow-diagrams.md)
- Code examples show real patterns

**Learning**:
- Follow learning paths above
- Try examples hands-on
- Check source code for details

---

## 🤝 Contributing to Docs

Found an error or want to improve documentation?

1. **Issues**: Open GitHub issue with `documentation` label
2. **PRs**: Submit PR with clear description
3. **Discussion**: Ask in Discord #docs channel

**Guidelines**:
- Clear, concise writing
- Code examples for concepts
- Diagrams for complex flows
- Cross-references between docs

---

## 📝 Document Conventions

**Code Blocks**:
- TypeScript for type examples
- Bash for CLI examples
- JSON for config examples
- Markdown for documentation

**Cross-References**:
- `[Document Name](./filename.md)` for docs
- `[Section Name](./filename.md#section)` for sections
- `packages/opencode/src/...` for source files

**Terminology**:
- **Session**: Conversation with AI
- **Tool**: Action AI can take
- **Agent**: Behavior configuration
- **Provider**: AI model provider
- **Instance**: Project instance

---

**OpenCode** - Open source, provider-agnostic, terminal-first AI coding for everyone.

Built by the team at [SST](https://sst.dev) • Created by [@thdxr](https://x.com/thdxr) and [@jayair](https://x.com/jayair)

