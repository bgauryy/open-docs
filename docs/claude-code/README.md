# Claude Code - Documentation Hub

> Your guide to understanding Claude Code's implementation, features, and internals

## What is This?

Claude Code is Anthropic's official CLI tool for Claude, built on the Claude Agent SDK. This documentation hub provides deep technical insights into how Claude Code works under the hood, including implementation details, hidden features, and expert tips.

**What's covered:**
- Complete system prompts and agent behavior
- Architecture and implementation details
- Built-in tools and their capabilities
- Undocumented features and gotchas
- NPX tool integration patterns

---

## 📚 Documentation Guide

### Start Here

| Document | When to Read It |
|----------|----------------|
| **[Architecture](./claude-code-architecture.md)** | Your first stop - understand the big picture |
| **[All Prompts](./claude-code-all-prompts.md)** | See how Claude Code agents are guided |
| **[Gotchas & Hidden Issues](./claude-code-gotchas-and-hidden-issues.md)** | Before using Claude Code seriously - learn the pitfalls |

### Core References

| Document | What You'll Find |
|----------|-----------------|
| **[Internal Tools Deep Dive](./claude-code-internal-tools-deep-dive.md)** | Complete reference for all built-in tools |
| **[Implementation Review](./claude-code-implementation-review.md)** | Key modules, control flow, and algorithms |
| **[Undocumented Features](./claude-code-undocumented-features-and-prompts.md)** | Hidden behaviors and patterns (all verified) |

### Integration & Tools

| Document | What You'll Find |
|----------|-----------------|
| **[NPX Tools Extraction](./claude-code-npx-tools-extraction.md)** | How to integrate and use NPX tools |
| **[Docs vs NPX Comparison](./claude-code-docs-vs-npx-comparison.md)** | Coverage, reliability, and usage patterns |

---

## 🎯 Quick Navigation

**I want to...**

- **Understand how Claude Code works** → [Architecture](./claude-code-architecture.md)
- **See all system prompts** → [All Prompts](./claude-code-all-prompts.md)
- **Learn about built-in tools** → [Internal Tools Deep Dive](./claude-code-internal-tools-deep-dive.md)
- **Avoid common mistakes** → [Gotchas & Hidden Issues](./claude-code-gotchas-and-hidden-issues.md)
- **Find hidden features** → [Undocumented Features](./claude-code-undocumented-features-and-prompts.md)
- **Integrate NPX tools** → [NPX Tools Extraction](./claude-code-npx-tools-extraction.md)
- **Study the implementation** → [Implementation Review](./claude-code-implementation-review.md)

---

## 🚀 What is Claude Code?

### Overview
Claude Code is a command-line interface for Claude that provides:
- **17 Built-in Tools**: File operations, search, web, commands, and more
- **Agent System**: Specialized sub-agents for complex workflows
- **MCP Integration**: Add custom tools via Model Context Protocol
- **Security Hooks**: Control and intercept every action
- **Session Management**: Persistent conversations with branching

### Built on the SDK
Claude Code is implemented using the Claude Agent SDK. For SDK documentation, see:
→ [Claude Agent SDK Documentation](../claude-agent-sdk/README.md)

---

## 🔑 Key Insights

### System Prompts
Claude Code uses specialized system prompts to guide agent behavior:
- **Main System Prompt**: Core instructions for Claude Code behavior
- **Agent Prompts**: Specialized prompts for different agent types
- **Tool Usage Guidelines**: How agents should use each tool

→ See [All Prompts](./claude-code-all-prompts.md) for complete prompts

### Architecture
Process-based architecture with:
- **CLI Binary**: 9.3MB minified JavaScript bundle
- **Tool Implementation**: All 17 tools implemented in the CLI
- **Session Storage**: File-based conversation persistence
- **Hook System**: Intercept and control all operations

→ See [Architecture](./claude-code-architecture.md) for details

### Built-in Tools
17 tools across 6 categories:
- **File Operations**: FileRead, FileWrite, FileEdit, NotebookEdit
- **Discovery**: Glob, Grep
- **Execution**: Bash, BashOutput, KillShell
- **Web**: WebFetch, WebSearch
- **MCP**: McpInput, ListMcpResources, ReadMcpResource
- **Management**: TodoWrite, Agent, ExitPlanMode

→ See [Internal Tools Deep Dive](./claude-code-internal-tools-deep-dive.md)

---

## ⚠️ Important Things to Know

Before using Claude Code, read [Gotchas & Hidden Issues](./claude-code-gotchas-and-hidden-issues.md) to learn about:

1. **10-minute Bash timeout** - Hard limit on all command execution
2. **FileEdit uniqueness requirement** - old_string must be unique
3. **Grep default behavior** - Returns filenames only by default
4. **Permission escalation** - Can be bypassed programmatically
5. **Auto-compaction** - Conversations are automatically summarized
6. **Hook timeouts** - 5-second default, configurable
7. **Synthetic messages** - System can inject user messages
8. **Tool restrictions** - Whitelist only, no blacklist

→ See [Gotchas & Hidden Issues](./claude-code-gotchas-and-hidden-issues.md) for details

---

## 🎓 Learning Paths

### Path 1: Understanding Claude Code
1. [Architecture](./claude-code-architecture.md) - System design and components
2. [All Prompts](./claude-code-all-prompts.md) - How agents are guided
3. [Internal Tools Deep Dive](./claude-code-internal-tools-deep-dive.md) - Tool capabilities

### Path 2: Advanced Usage
1. [Gotchas & Hidden Issues](./claude-code-gotchas-and-hidden-issues.md) - Critical issues
2. [Undocumented Features](./claude-code-undocumented-features-and-prompts.md) - Hidden capabilities
3. [NPX Tools Extraction](./claude-code-npx-tools-extraction.md) - Tool integration

### Path 3: Deep Implementation Study
1. [Implementation Review](./claude-code-implementation-review.md) - Code analysis
2. [Architecture](./claude-code-architecture.md) - System design
3. [Docs vs NPX Comparison](./claude-code-docs-vs-npx-comparison.md) - Tooling patterns

---

## 📖 Documentation Structure

Each document serves a specific purpose:

- **Architecture** - High-level system design and data flow
- **Prompts** - System prompts that guide agent behavior
- **Implementation Reviews** - Code-level analysis and algorithms
- **Tool Documentation** - Capabilities and constraints of built-in tools
- **Gotchas** - Critical issues, limitations, and workarounds
- **Undocumented Features** - Verified hidden behaviors and patterns

All technical details, code examples, and implementation analysis live in the individual documents - this README is your map to find what you need.

---

## 🔗 Official Resources

- **GitHub**: https://github.com/anthropics/claude-code
- **Official Docs**: https://docs.claude.com/en/docs/claude-code
- **Discord**: https://anthropic.com/discord
- **Claude Agent SDK**: [Documentation Hub](../claude-agent-sdk/README.md)

---

## 📋 Complete Document Index

| Document | Description |
|----------|-------------|
| **[claude-code-all-prompts.md](./claude-code-all-prompts.md)** | Complete system prompts - all prompts extracted from CLI implementation |
| **[claude-code-architecture.md](./claude-code-architecture.md)** | High-level architecture - components, data flow, lifecycle, and integrations |
| **[claude-code-docs-vs-npx-comparison.md](./claude-code-docs-vs-npx-comparison.md)** | Tools Docs vs NPX tooling - coverage, freshness, reliability, and patterns |
| **[claude-code-gotchas-and-hidden-issues.md](./claude-code-gotchas-and-hidden-issues.md)** | Gotchas and expert tips - critical issues, hidden features, and limitations |
| **[claude-code-implementation-review.md](./claude-code-implementation-review.md)** | Implementation deep-dive - key modules, control flow, and edge cases |
| **[claude-code-internal-tools-deep-dive.md](./claude-code-internal-tools-deep-dive.md)** | Built-in tools reference - capabilities, invocation flow, and constraints |
| **[claude-code-npx-tools-extraction.md](./claude-code-npx-tools-extraction.md)** | NPX tools guide - discovery, wrapping, flags, and usage examples |
| **[claude-code-undocumented-features-and-prompts.md](./claude-code-undocumented-features-and-prompts.md)** | Undocumented features catalog - verified behaviors, features, and patterns |

---

## 📝 About This Documentation

This documentation complements the official docs by providing deeper technical details, implementation analysis, and comprehensive references.

