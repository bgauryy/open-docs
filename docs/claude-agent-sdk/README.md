# Claude Agent SDK - Documentation Hub

> Your guide to building AI agents with Claude

## What is This?

The Claude Agent SDK (`@anthropic-ai/claude-agent-sdk`) is a TypeScript library that lets you build AI agents powered by Claude. This documentation hub helps you understand how it works, what it can do, and how to use it effectively.

**Note**: This is an in-depth technical analysis. For official documentation, visit: https://docs.claude.com/en/api/agent-sdk/overview

**What makes it special?**
- 17 built-in tools for file operations, commands, web search, and more
- Add your own tools via Model Context Protocol (MCP)
- Specialized sub-agents for complex workflows
- Security hooks to control every action
- Flexible permissions from manual to fully automatic

---

## 📚 Documentation Guide

### Start Here

| Document | When to Read It |
|----------|----------------|
| **[Comprehensive Guide](./comprehensive-guide.md)** | Your first stop - overview of everything the SDK can do |
| **[Type System](./types-complete.md)** | When you need TypeScript types and interfaces |
| **[Implementation & Gotchas](./implementation-gotchas.md)** | Before building anything serious - learn the pitfalls |

### Core Features

| Document | What You'll Find |
|----------|-----------------|
| **[Tool System](./tools-complete.md)** | Complete reference for all 17 built-in tools |
| **[Hooks & Permissions](./hooks-permissions-complete.md)** | Security, control flow, and permission management |
| **[Agent System](./agents-complete.md)** | Creating specialized sub-agents for complex tasks |
| **[Memory & Context](./memory-and-context.md)** | Session management, context persistence, and memory best practices |

### Deep Dives

| Document | What You'll Find |
|----------|-----------------|
| **[System Prompts](./prompts-complete.md)** | Internal prompts that guide agent behavior |
| **[Plugin System](./plugins.md)** | Complete guide to CLI plugin marketplace system |
| **[Undocumented Features](./undocumented.md)** | Hidden features and internal APIs not in official docs |
| **[Design & Architecture](./design.md)** | Architecture patterns and design decisions |

### Claude Code Implementation

**Looking for Claude Code user documentation?** See the [Claude Code Documentation Hub](../claude-code/README.md)

| Document | What You'll Find |
|----------|-----------------|
| **[CLI Architecture](./architecture.md)** | Complete CLI commands, modules, and systems reference |
| **[Plugin System](./plugins.md)** | Plugin & marketplace commands (`claude plugin`/`marketplace`) |
| **[CLI Analysis](./cli_analysis.md)** | How the SDK and CLI work together |
| **[SDK Implementation Analysis](./implementaion-analysis.md)** | Process architecture and communication patterns |
| **[Skills System](./skills-documentaion.md)** | Claude Code's skill definition and execution |
| **[Internal Flows](./internal-flows.md)** | How tools work internally |

---

## 🎯 Quick Navigation

**I want to...**

- **Get started quickly** → [Comprehensive Guide](./comprehensive-guide.md)
- **Understand the 17 tools** → [Tool System](./tools-complete.md)
- **Build secure agents** → [Hooks & Permissions](./hooks-permissions-complete.md)
- **Create sub-agents** → [Agent System](./agents-complete.md)
- **Manage memory & sessions** → [Memory & Context](./memory-and-context.md)
- **Avoid common mistakes** → [Implementation & Gotchas](./implementation-gotchas.md)
- **Add custom tools** → [Comprehensive Guide - MCP Section](./comprehensive-guide.md)
- **Install CLI plugins** → [Plugin System](./plugins.md)
- **Understand internals** → [System Prompts](./prompts-complete.md) + [Undocumented Features](./undocumented.md)
- **Study Claude Code** → [CLI Architecture](./architecture.md) + [CLI Analysis](./cli_analysis.md)

---

## 🚀 Getting Started

### Installation
```bash
npm install @anthropic-ai/claude-agent-sdk
```

### First Query
See [Comprehensive Guide](./comprehensive-guide.md) for detailed examples.

---

## 🔑 Key Concepts

### The Tool System
17 built-in tools organized into categories:
- **File Operations**: Read, write, edit files and notebooks
- **Discovery**: Search files by name or content
- **Execution**: Run commands and scripts
- **Web**: Fetch pages and search
- **MCP**: Integrate custom tools
- **Management**: Task tracking and agent delegation

→ See [Tool System](./tools-complete.md) for complete reference

### Security & Control
Control agent behavior through:
- **9 Hook Events**: Intercept and modify every action
- **4 Permission Modes**: From manual approval to full automation
- **Runtime Control**: Change behavior mid-execution
- **Tool Whitelisting**: Restrict agent capabilities

→ See [Hooks & Permissions](./hooks-permissions-complete.md)

### Agent Architecture
Build complex workflows with:
- **Sub-agents**: Specialized agents for specific tasks
- **Model Selection**: Choose haiku/sonnet/opus per agent
- **Tool Restrictions**: Whitelist allowed tools per agent
- **Custom Prompts**: Define agent behavior and expertise

→ See [Agent System](./agents-complete.md)

---

## ⚠️ Important Things to Know

Before building with the SDK, read [Implementation & Gotchas](./implementation-gotchas.md) to learn about:

1. **Runtime control only works with streaming** - `interrupt()`, `setPermissionMode()`, etc.
2. **Bash has 10-minute hard timeout** - Use background execution for long tasks
3. **Permission escalation is possible** - Can bypass all security programmatically
4. **Tool restrictions are whitelist-only** - Can't say "everything except X"
5. **Hook matcher syntax is undocumented** - Use callback logic instead
6. **Conversation auto-compaction** - Message indices can shift
7. **FileEdit requires unique strings** - Must be unambiguous in file
8. **Grep returns filenames by default** - Must specify `output_mode: "content"`

→ See [Implementation & Gotchas](./implementation-gotchas.md) for details

---

## 🎓 Learning Paths

### Path 1: Building Your First Agent
1. [Comprehensive Guide](./comprehensive-guide.md) - Get the overview
2. [Tool System](./tools-complete.md) - Learn what tools do
3. [Memory & Context](./memory-and-context.md) - Understand sessions and context
4. [Implementation & Gotchas](./implementation-gotchas.md) - Avoid mistakes

### Path 2: Building Secure Agents
1. [Hooks & Permissions](./hooks-permissions-complete.md) - Understand security
2. [Agent System](./agents-complete.md) - Learn tool restrictions
3. [Implementation & Gotchas](./implementation-gotchas.md) - Security patterns

### Path 3: Understanding the Internals
1. [System Prompts](./prompts-complete.md) - How agents are guided
2. [Memory & Context](./memory-and-context.md) - Session architecture and persistence
3. [Undocumented Features](./undocumented.md) - Hidden capabilities
4. [Design & Architecture](./design.md) - Architecture patterns
5. [CLI Analysis](./cli_analysis.md) - How Claude Code uses the SDK

### Path 4: Studying Claude Code
1. [CLI Architecture](./architecture.md) - Complete CLI reference
2. [CLI Analysis](./cli_analysis.md) - SDK-CLI relationship
3. [Skills System](./skills-documentaion.md) - How skills work
4. [Internal Flows](./internal-flows.md) - Tool implementations

---

## 📖 Documentation Structure

Each document in this collection serves a specific purpose:

- **Guides** teach you how to use features
- **References** document APIs and types
- **Analyses** explain implementation details
- **Architecture** describes system design

All technical details, code examples, and API documentation live in the individual documents - this README is your map to find what you need.

---

## 🔗 Official Resources

- [**Official Agent SDK Documentation**](https://docs.claude.com/en/api/agent-sdk/overview) - Official docs from Anthropic
- [GitHub Repository](https://github.com/anthropics/claude-agent-sdk-typescript)
- [Anthropic Discord](https://anthropic.com/discord)

---

## 📝 About This Documentation

This documentation complements the official docs by providing deeper technical details, implementation analysis, and comprehensive references.
