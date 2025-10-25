# open-docs

> Deep-dive documentation for AI CLI tools — the stuff official docs don't cover.

**For developers, researchers, and power users** who need to understand how these tools actually work under the hood. We extract undocumented features, hidden APIs, architecture internals, gotchas, and real implementation patterns directly from source code.

**Why use these docs?** Official docs tell you *what* to do. We show you *how it works*, *what's not documented*, and *what breaks*.

*All content extracted from public open-source repositories — no proprietary info, no secrets, just better organized knowledge.*

## 💡 Motivation

Open source doesn't automatically mean well-documented. While these projects are publicly available, critical details often remain buried in implementation code:

- **Undocumented features** — Experimental APIs, beta functionality, and hidden capabilities not in official docs
- **Implementation internals** — How things actually work vs. what the public API exposes
- **System prompts** — The actual instructions sent to LLMs that shape agent behavior
- **Real-world patterns** — Gotchas, edge cases, and integration nuances learned from source code

This repository started when I tried to understand **how the Task tool works under the hood** across different CLI agents. That single question led to extracting architecture diagrams, security models, tool systems, and undocumented flows that weren't explained anywhere else.

**For developers and agents:** Use this as a reference when building AI workflows, understanding tool implementations, or debugging complex behaviors. These docs bridge the gap between "read the source" and "trust the marketing page."

## 📚 Available Documentation

### 1. Claude Agent SDK
**[→ Full Documentation](docs/claude-agent-sdk/README.md)**

TypeScript library by Anthropic for building AI agents powered by Claude. 17 built-in tools, MCP integration, security hooks, specialized sub-agents, and flexible permission modes.

### 2. Gemini CLI
**[→ Full Documentation](docs/gemini-cli/README.md)**

Google's terminal CLI for Gemini AI. 5-package system with 15+ tools, agent-to-agent communication, VS Code integration, and MCP support.

### 3. Codex CLI
**[→ Full Documentation](docs/codex_cli/README.md)**

OpenAI's Rust-based terminal coding agent. High-performance with advanced tools, security sandboxing, and comprehensive approval policies.

### 4. OpenCode
**[→ Full Documentation](docs/opencode/README.md)**

Open-source, provider-agnostic AI coding agent built for the terminal. Client/server architecture with built-in LSP, supports multiple AI providers (Anthropic, OpenAI, Google, Bedrock), and features ACP/MCP protocol integration.

---

## 🔍 Recommended: Octocode MCP

Smart AI code and GitHub research MCP server for exploring this repository. Provides advanced pattern matching, semantic search, and cross-reference analysis across all documentation.

→ [github.com/bgauryy/octocode-mcp](https://github.com/bgauryy/octocode-mcp) | [Demo](https://www.youtube.com/watch?v=S2pcEjHo6CM)



https://github.com/user-attachments/assets/825ebe61-4000-47a5-beb8-f241dde41a73



## 💬 Request Documentation

Know an AI/CLI tool with incomplete docs, hidden features, or complex internals? [Open an issue](https://github.com/bgauryy/open-docs/issues) with the repo URL and what's missing.

**Have specific questions about documented repositories?** Also welcome! Open an issue with your questions about architecture, implementation details, or undocumented behaviors.

## ⭐ Support

Star on GitHub • Share with others • Request documentation for your projects • Sponsorships help maintain this project

## ⚖️ Disclaimer

Research and educational purposes only. Content derived from public open-source resources. Verify all information independently. No warranty provided. Use at your own risk.

---

**Enterprise Documentation:** Need enterprise-level documentation for your organization's codebase? Reach out: [bgauryy@octocodeai.com](mailto:bgauryy@octocodeai.com)
