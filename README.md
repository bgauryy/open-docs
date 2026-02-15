#  open-docs 
**Powered by [Octocode](https://github.com/bgauryy/octocode-mcp)🐙** 

> **The missing manual for AI CLI tools.**  
> Deep-dive documentation, implementation secrets, and architecture internals extracted directly from source code.

---

**For developers, researchers, and power users** who need to look under the hood. Official documentation tells you *what* a tool does. We show you *how it works*, *what's undocumented*, and *how to build on top of it*.

### 💡 Why this exists
Open source doesn't always mean well-documented. Critical details are often buried in code. We extract:
- 🛠️ **Internals & Architecture** — How the systems are actually built
- 🧩 **Undocumented Features** — Hidden APIs, beta functionality, and system prompts
- ⚠️ **Real-world Gotchas** — Edge cases, security models, and implementation constraints
- 🔌 **Integration Patterns** — How to properly interface with these agents

---

## 📖 Documentation Library

### [1. Claude Agent SDK](docs/claude-agent-sdk/README.md)
> **Anthropic** • *TypeScript* • [GitHub Repo](https://github.com/anthropics/claude-agent-sdk)

The official SDK for building Claude-powered agents.
- **Key Features**: Built-in tool system, MCP integration, specialized sub-agents.
- **Deep Dive**: Security hooks, permission modes, and agent lifecycle management.

### [2. Gemini CLI](docs/gemini-cli/README.md)
> **Google** • *TypeScript* • [GitHub Repo](https://github.com/google-gemini/gemini-cli)

Google's terminal CLI for Gemini.
- **Key Features**: Multi-package architecture, agent-to-agent communication, VS Code integration.
- **Deep Dive**: Tool definitions, runtime environment, and extension points.

### [3. Codex CLI](docs/codex_cli/README.md)
> **OpenAI** • *Rust* • [GitHub Repo](https://github.com/openai/codex)

High-performance terminal coding agent.
- **Key Features**: Rust-based performance, advanced sandboxing, security policies.
- **Deep Dive**: Execution environment, prompt engineering strategies, and state management.

### [4. OpenCode](docs/opencode/README.md)
> **Community** • *TypeScript* • [GitHub Repo](https://github.com/opencode-ai/opencode)

Provider-agnostic AI coding agent for the terminal.
- **Key Features**: Supports OpenAI/Anthropic/Google/Bedrock, built-in LSP, Client/Server architecture.
- **Deep Dive**: ACP/MCP protocol integration, TUI implementation, and provider abstraction layers.

### [5. Pi Coding Agent](docs/coding-agent/README.md)
> **@mariozechner** • *TypeScript* • [GitHub Repo](https://github.com/badlogic/pi-mono)

Lightweight but powerful CLI coding agent with session management.
- **Key Features**: Read/Bash/Edit/Write toolset, robust session handling.
- **Deep Dive**: API reference, data models, and session format specifications.

---

## 🤝 Contribute & Request

**Found an undocumented tool?**  
[Open an issue](https://github.com/bgauryy/open-docs/issues) with the repo URL. We're always looking for new CLI agents to analyze.

**Questions?**  
Feel free to open discussions about architecture, specific implementation details, or if you need clarification on any docs.

## ⭐ Support the Project
If these docs saved you time or helped you build something cool, please consider **starring the repo** to help others find it.

---

### ⚖️ Disclaimer
*Research and educational purposes only. Content derived from public open-source repositories. Verify all information independently. No warranty provided.*

**Enterprise Support**: Need internal documentation for your own AI agents? Reach out: [bgauryy@octocodeai.com](mailto:bgauryy@octocodeai.com)
