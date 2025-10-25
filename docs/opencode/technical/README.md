## Documentation Structure

### Core Documentation

- [01 - Overview](./01-overview.md) - Project overview, tech stack, and structure
- [02 - Architecture](./02-architecture.md) - System architecture and component layers
- [API Reference](./api-reference.md) - Complete API documentation

### Flow Documentation

- [Interactive Chat Session](./flows/run-command-flow.md) - Main CLI chat interaction flow
- [Server Mode](./flows/serve-command-flow.md) - HTTP server for remote access
- [Agent Client Protocol](./flows/acp-agent-flow.md) - ACP protocol communication
- [Model Context Protocol Integration](./flows/mcp-integration-flow.md) - MCP integration flow
- [Tool Execution](./flows/tool-execution-flow.md) - Tool execution workflow

### Component Documentation

- [ACP (Agent Client Protocol)](./components/acp.md)
- [Agent Orchestration](./components/agent.md)
- [Authentication](./components/auth.md)
- [CLI Commands](./components/cli.md)
- [Configuration](./components/config.md)
- [File Operations](./components/file.md)
- [LSP (Language Server Protocol)](./components/lsp.md)
- [MCP (Model Context Protocol)](./components/mcp.md)
- [Provider Integration](./components/provider.md)
- [Server](./components/server.md)
- [Session Management](./components/session.md)
- [Tools](./components/tool.md)

## Quick Start

OpenCode is a CLI tool that provides AI-powered coding assistance through multiple interfaces:

```bash
# Interactive chat session
opencode run "help me build a feature"

# Start HTTP server
opencode serve --port 3000

# Launch terminal UI
opencode tui

# Start ACP server
opencode acp
```

## Key Features

- **Multiple AI Providers**: Anthropic (Claude), OpenAI, Google Vertex AI, Amazon Bedrock, and local models
- **Protocol Support**: MCP (Model Context Protocol), ACP (Agent Client Protocol), LSP (Language Server Protocol)
- **Rich Tool System**: 36+ tools for file operations, code editing, shell execution, and more
- **Multi-Interface**: CLI, TUI (Go-based), HTTP server, desktop app, and web interface
- **Monorepo Architecture**: 20 packages including TypeScript core, Go TUI, and multi-language SDKs

## Technology Stack

- **Runtime**: Bun 1.3.0+ (primary), Node.js 22+ (compatible)
- **Languages**: TypeScript (primary), Go (TUI)
- **Frameworks**: Yargs (CLI), Hono (backend), SolidJS (frontend), Astro (docs)
- **Package Manager**: Bun with workspace support
- **Build Tool**: Turbo
