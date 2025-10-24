# Codex CLI - Complete Technical Documentation

> **Comprehensive technical documentation for Codex CLI, OpenAI's terminal-based AI coding agent**

[![Documentation Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/openai/codex)
[![License](https://img.shields.io/badge/license-Apache%202.0-green.svg)](https://www.apache.org/licenses/LICENSE-2.0)
[![Status](https://img.shields.io/badge/status-complete-brightgreen.svg)]()

---

## 📚 What is This?

This is a comprehensive technical documentation suite covering every aspect of **Codex CLI** - a sophisticated, Rust-based terminal coding agent. Whether you're contributing to the project, building integrations, or simply want to understand how it works, this documentation has you covered.

**Coverage**: 15 in-depth documents • 150+ pages • 200+ code examples • 30+ architecture diagrams

---

## 🚀 Quick Start

### New to Codex CLI?
Start here to understand what Codex is and how to use it:
- **[01 - Overview](./01-overview.md)** - What is Codex CLI, key features, and quick start guide

### Want to Understand the Architecture?
Learn how the system is designed:
- **[02 - Architecture](./02-architecture.md)** - System design, components, and data flow
- **[10 - Implementation](./10-implementation.md)** - Entry points, event loops, and core orchestrator

### Need to Configure or Deploy?
Configuration and security references:
- **[08 - Configuration](./08-configuration.md)** - Complete config reference and examples
- **[07 - Security & Sandboxing](./07-security-sandboxing.md)** - Security model and approval policies

### Building Tools or Extensions?
Developer guides for extending Codex:
- **[06 - Tool System](./06-tool-system.md)** - Tool architecture and how to add new tools
- **[11 - Tool Implementations](./11-tool-implementations.md)** - Detailed tool code examples

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

---

## 🎯 Documentation by Use Case

### I Want To...

#### **Understand How It Works**
1. Start with [01 - Overview](./01-overview.md) for the big picture
2. Read [02 - Architecture](./02-architecture.md) for system design
3. Check [03 - Prompt Processing](./03-prompt-processing.md) for request flow

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
4. [07 - Security & Sandboxing](./07-security-sandboxing.md) - Understand safety

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

- **Total Documents**: 15 comprehensive guides
- **Total Words**: ~80,000 words
- **Code Examples**: 200+ with syntax highlighting
- **Architecture Diagrams**: 30+ ASCII art diagrams
- **Tables & References**: 50+ comparison tables
- **Cross-References**: 100+ internal links
- **Coverage**: Architecture, Tools, Security, Config, State, UI, Integration

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
| **Documentation Version** | 1.0.0 |
| **Codex CLI Version** | Latest Development |
| **Last Updated** | October 23, 2025 |
| **Format** | GitHub-flavored Markdown |

---

## 📄 License

**Documentation**: Technical analysis provided as-is  
**Codex CLI**: Apache License 2.0 © OpenAI

---

## 🔗 Quick Links

- [Codex GitHub Repository](https://github.com/openai/codex)
- [Official OpenAI Documentation](https://openai.com)
- [Model Context Protocol](https://modelcontextprotocol.io)

---

**Ready to dive in?** Start with [01 - Overview](./01-overview.md) or jump to any document that interests you!
