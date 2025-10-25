# Codex CLI - Complete Technical Documentation

> **Comprehensive technical documentation for Codex CLI, OpenAI's terminal-based AI coding agent**

[![Documentation Version](https://img.shields.io/badge/version-2.0.0-blue.svg)](https://github.com/openai/codex)
[![License](https://img.shields.io/badge/license-Apache%202.0-green.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Status](https://img.shields.io/badge/status-enhanced-brightgreen.svg)]()
[![Cross-Referenced](https://img.shields.io/badge/cross--referenced-official%20docs-blue.svg)]()

---

## 📚 What is This?

This is a comprehensive **technical documentation suite** covering the **implementation details and internals** of **Codex CLI** - OpenAI's sophisticated, Rust-based terminal coding agent.

### 🎯 For Developers & Contributors
This documentation focuses on:
- **Internal architecture** and implementation details
- **Rust source code** analysis and patterns
- **System internals** that official docs don't cover
- **Development guides** for extending Codex

### 📖 For End Users
Official user documentation is available at:
- **Official Docs**: https://github.com/openai/codex/tree/main/docs
- **Getting Started**: Start there for usage instructions

### 🔗 Complementary, Not Competing
These docs **complement** official documentation:
- **Official Docs**: How to **use** Codex (configuration, commands, features)
- **This Suite**: How Codex **works** internally (architecture, code, patterns)

**Coverage**: **23** in-depth documents • **250+** pages • **300+** code examples • **50+** architecture diagrams

---

## 🚀 Quick Start

### ⚠️ START HERE: Official vs Custom Docs
**Required Reading**: [00 - Official vs Custom Documentation](./00-OFFICIAL-VS-CUSTOM.md)

**Understand the difference**:
- **Official Docs** (OpenAI) = **How to use** Codex → [Go to official docs](../../context/codex/docs/)
- **Custom Docs** (This Suite) = **How Codex works** internally → You're here!

### 🎯 Quick Navigation

#### New to Codex CLI?
1. **[Official Getting Started](../../context/codex/docs/getting-started.md)** ← Start here for usage
2. **[01 - Overview](./01-overview.md)** ← Then read this for architecture overview
3. **[23 - Flow Diagrams](./23-flow-diagrams.md)** ← Visual guides to understand flows

#### Want to Understand the Architecture?
1. **[02 - Architecture](./02-architecture.md)** - Complete system design
2. **[23 - Flow Diagrams](./23-flow-diagrams.md)** - Visual flow references
3. **[10 - Implementation](./10-implementation.md)** - Core implementation patterns

#### Need to Configure or Deploy?
1. **[Official config.md](../../context/codex/docs/config.md)** ← User configuration guide
2. **[08 - Configuration](./08-configuration.md)** ← Internal config system details
3. **[Official sandbox.md](../../context/codex/docs/sandbox.md)** ← Security usage guide
4. **[07 - Security & Sandboxing](./07-security-sandboxing.md)** ← Security implementation

#### Building Tools or Extensions?
1. **[06 - Tool System](./06-tool-system.md)** - Tool architecture deep-dive
2. **[11 - Tool Implementations](./11-tool-implementations.md)** - Real code examples
3. **[21 - Tool System Guide](./21-tool-system-practical.md)** - Practical development guide

#### Understanding Automation (Exec Mode)?
1. **[Official exec.md](../../context/codex/docs/exec.md)** ← How to use `codex exec`
2. **[22 - Exec Mode Internals](./22-exec-mode-internals.md)** ← Implementation details

---

## 📖 Complete Documentation Index

### 🏗️ Architecture & Design
Understand how Codex CLI is built and why it works the way it does.

| Document | Description | Best For |
|----------|-------------|----------|
| **[01 - Overview](./01-overview.md)** | Introduction, capabilities, and quick start | First-time users |
| **[02 - Architecture](./02-architecture.md)** | System design, components, patterns | Architects, contributors |
| **[10 - Implementation](./10-implementation.md)** | Entry points, event loops, async patterns | Core developers |

### 🤖 LLM & Prompt System
How Codex processes prompts and communicates with language models.

| Document | Description | Best For |
|----------|-------------|----------|
| **[03 - Prompt Processing](./03-prompt-processing.md)** | Prompt lifecycle and assembly | Understanding context flow |
| **[04 - LLM Integration](./04-llm-integration.md)** | Model clients, streaming, providers | API integrations |
| **[05 - System Prompts](./05-system-prompts.md)** | AGENTS.md, custom prompts, hierarchy | Customizing behavior |

### 🔧 Tools & Execution
The tool system that allows Codex to take actions.

| Document | Description | Best For |
|----------|-------------|----------|
| **[06 - Tool System](./06-tool-system.md)** | Tool architecture, registry, execution | Building new tools |
| **[11 - Tool Implementations](./11-tool-implementations.md)** | Detailed tool code and examples | Tool developers |

### 🔒 Security & Configuration
How Codex stays safe and how to configure it.

| Document | Description | Best For |
|----------|-------------|----------|
| **[07 - Security & Sandboxing](./07-security-sandboxing.md)** | Sandbox modes, approvals, safety | Security auditing |
| **[08 - Configuration](./08-configuration.md)** | Config files, env vars, CLI flags | Setup and deployment |

### 💾 State & Interface
How Codex manages state and presents information.

| Document | Description | Best For |
|----------|-------------|----------|
| **[09 - State Management](./09-state-management.md)** | Sessions, history, diff tracking | Understanding state flow |
| **[12 - UI Layer](./12-ui-layer.md)** | Terminal UI, rendering, interactions | UI development |

### 🔌 Integration & Reference
Authentication, external integrations, and code reference.

| Document | Description | Best For |
|----------|-------------|----------|
| **[13 - Authentication](./13-authentication.md)** | Auth flows, token management | Auth integration |
| **[14 - MCP Integration](./14-mcp-integration.md)** | Model Context Protocol support | MCP server authors |
| **[15 - Code Reference](./15-code-reference.md)** | File index and code navigation | Finding implementations |
| **[16 - Hidden Features](./16-hidden-features.md)** | Undocumented commands, flags, env vars | Power users, developers |
| **[17 - CLI Reference](./17-cli-reference.md)** | Complete CLI commands and flags | Command-line users |
| **[18 - MCP Development](./18-mcp-development.md)** | Building MCP servers from scratch | MCP developers |

### ⚡ Advanced Topics & Practical Guides
Automation, visual references, and optimization.

| Document | Description | Best For |
|----------|-------------|----------|
| **[19 - Performance](./19-performance.md)** | Speed, cost, quality optimization | Tuning and optimization |
| **[20 - State Management Guide](./20-state-management-practical.md)** | Practical state management guide | Session management |
| **[21 - Tool System Guide](./21-tool-system-practical.md)** | Practical tool development guide | Tool creators |
| **[22 - Exec Mode Internals](./22-exec-mode-internals.md)** | Non-interactive automation implementation | CI/CD, scripting |
| **[23 - Flow Diagrams](./23-flow-diagrams.md)** | Complete visual flow references | Understanding system flows |

---

## 🎯 Documentation by Use Case

### I Want To...

#### **Understand How It Works**
1. Start with [01 - Overview](./01-overview.md) for the big picture
2. Read [02 - Architecture](./02-architecture.md) for system design
3. Check [03 - Prompt Processing](./03-prompt-processing.md) for request flow
4. Explore [16 - Hidden Features](./16-hidden-features.md) for undocumented capabilities

#### **Configure and Deploy Codex**
1. Read [08 - Configuration](./08-configuration.md) for all options
2. Review [07 - Security & Sandboxing](./07-security-sandboxing.md) for security settings
3. Check [13 - Authentication](./13-authentication.md) for auth setup

#### **Build Custom Tools**
1. Study [06 - Tool System](./06-tool-system.md) for architecture
2. Review [11 - Tool Implementations](./11-tool-implementations.md) for examples
3. Reference [15 - Code Reference](./15-code-reference.md) for finding code

#### **Customize Behavior**
1. Read [05 - System Prompts](./05-system-prompts.md) for prompt customization
2. Learn about AGENTS.md files and custom prompts
3. Check [08 - Configuration](./08-configuration.md) for behavior settings
4. Discover [16 - Hidden Features](./16-hidden-features.md) for experimental flags and env vars

#### **Integrate with External Systems**
1. Review [14 - MCP Integration](./14-mcp-integration.md) for MCP protocol
2. Check [04 - LLM Integration](./04-llm-integration.md) for provider support
3. Study [13 - Authentication](./13-authentication.md) for auth patterns

#### **Contribute to Development**
1. Read [02 - Architecture](./02-architecture.md) for design patterns
2. Study [10 - Implementation](./10-implementation.md) for core code
3. Review [15 - Code Reference](./15-code-reference.md) for navigation

---

## 🎓 Learning Paths

### **Path 1: User → Power User**
For those who want to use Codex effectively:
1. [01 - Overview](./01-overview.md) - Learn the basics
2. [08 - Configuration](./08-configuration.md) - Customize your setup
3. [05 - System Prompts](./05-system-prompts.md) - Create custom prompts
4. [16 - Hidden Features](./16-hidden-features.md) - Unlock undocumented capabilities
5. [07 - Security & Sandboxing](./07-security-sandboxing.md) - Understand safety

### **Path 2: Developer → Contributor**
For those who want to contribute code:
1. [02 - Architecture](./02-architecture.md) - Understand the design
2. [10 - Implementation](./10-implementation.md) - Core implementation details
3. [09 - State Management](./09-state-management.md) - How state works
4. [06 - Tool System](./06-tool-system.md) - Tool architecture
5. [11 - Tool Implementations](./11-tool-implementations.md) - Tool examples

### **Path 3: Architect → Integrator**
For those building on top of Codex:
1. [02 - Architecture](./02-architecture.md) - System design
2. [04 - LLM Integration](./04-llm-integration.md) - API integration
3. [14 - MCP Integration](./14-mcp-integration.md) - MCP protocol
4. [13 - Authentication](./13-authentication.md) - Auth patterns
5. [06 - Tool System](./06-tool-system.md) - Extension points

---

## 📊 Documentation Stats

- **Total Documents**: **23** comprehensive guides
- **Total Words**: ~**120,000** words
- **Code Examples**: **300+** with syntax highlighting
- **Architecture Diagrams**: **50+** ASCII art flow diagrams
- **Tables & References**: **80+** comparison tables
- **Cross-References**: **200+** internal links (+ official docs)
- **Coverage**: Architecture, Tools, Security, Config, State, UI, Integration, Hidden Features, CLI Reference, MCP Development, Performance Tuning, Exec Mode, Flow Diagrams

---

## 🛠️ Technical Details

### What's Documented
- ✅ **Complete Architecture** - Every component and interaction
- ✅ **All Tools** - Full tool system with implementation details
- ✅ **Security Model** - Sandboxing, approvals, and safety mechanisms
- ✅ **Configuration** - Every config option and environment variable
- ✅ **Prompt System** - How prompts are processed and customized
- ✅ **State Management** - Sessions, history, and diff tracking
- ✅ **LLM Integration** - All supported providers and APIs
- ✅ **MCP Protocol** - Model Context Protocol integration

### Based On
- **Codex Version**: Latest development build
- **Analysis**: 150+ source files, 50,000+ LOC examined
- **Rust Components**: `codex-rs/core/`, `codex-rs/cli/`, `codex-rs/tui/`
- **System Prompts**: All official prompts analyzed
- **Validation**: All code examples from actual source

---

## 🤝 Contributing

Found an issue or want to improve the docs?

1. **For Documentation Issues**: Open an issue describing what's unclear or incorrect
2. **For Code Questions**: Check [15 - Code Reference](./15-code-reference.md) first
3. **For Feature Requests**: See [02 - Architecture](./02-architecture.md) to understand design constraints

---

## 📋 Version Information

| Property | Value |
|----------|-------|
| **Documentation Version** | 2.0.0 (Enhanced) |
| **Codex CLI Version** | Latest Development |
| **Last Updated** | October 25, 2025 |
| **Format** | GitHub-flavored Markdown |
| **Cross-Referenced** | ✅ Linked to official docs |
| **Verified** | ✅ Code examples checked against source |

---

## 📄 License

**Documentation**: Technical analysis provided as-is  
**Codex CLI**: Apache License 2.0 © OpenAI

---

## 🔗 Quick Links

### Official Resources
- **[Official Codex Documentation](../../context/codex/docs/)** - User guides from OpenAI (START HERE for usage)
- **[Codex GitHub Repository](https://github.com/openai/codex)** - Source code
- **[Model Context Protocol](https://modelcontextprotocol.io)** - MCP specification

### Analysis Documents
- **[Official vs Custom Comparison](./00-OFFICIAL-VS-CUSTOM.md)** - Complete comparison with detailed doc-by-doc analysis
- **[Analysis Results](../../DOCS_ANALYSIS_RESULTS.md)** - Documentation analysis findings

### Navigation Guides
- **[00 - Official vs Custom](./00-OFFICIAL-VS-CUSTOM.md)** - Understand the relationship
- **[23 - Flow Diagrams](./23-flow-diagrams.md)** - Visual system flows
- **[15 - Code Reference](./15-code-reference.md)** - Source code navigation

---

## 🎯 Where to Start?

**Choose your path**:

1. **I'm a new user** → [Official Getting Started](../../context/codex/docs/getting-started.md) first, then [01 - Overview](./01-overview.md)

2. **I want to understand internals** → [23 - Flow Diagrams](./23-flow-diagrams.md) then [02 - Architecture](./02-architecture.md)

3. **I'm building tools/extensions** → [06 - Tool System](./06-tool-system.md) and [11 - Tool Implementations](./11-tool-implementations.md)

4. **I'm setting up CI/CD** → [Official exec.md](../../context/codex/docs/exec.md) then [22 - Exec Mode Internals](./22-exec-mode-internals.md)

5. **I want to see comparisons** → [Official vs Custom Comparison](./00-OFFICIAL-VS-CUSTOM.md)

---

**Ready to dive in?** Start with [23 - Flow Diagrams](./23-flow-diagrams.md) for visual overviews, or [01 - Overview](./01-overview.md) for architecture introduction!
