# Project Overview

## Summary

OpenCode is a multi-language, client/server AI coding agent with a monorepo architecture. It provides AI-powered coding assistance through multiple interfaces including CLI, TUI, HTTP server, desktop app, and web interface. The project integrates with various AI providers and supports multiple protocols (MCP, ACP, LSP) for extensibility.

## Technology Stack

| Category | Technologies |
|----------|-------------|
| **Runtime** | Bun 1.3.0+ (primary), Node.js 22+ (compatible), Go 1.21+ |
| **Languages** | TypeScript (337 files), Go (154 files), JavaScript (1 file) |
| **Package Manager** | Bun with workspace support |
| **Build Tool** | Turbo |
| **CLI Framework** | Yargs |
| **Backend Framework** | Hono |
| **Frontend Framework** | SolidJS |
| **Documentation** | Astro |
| **Infrastructure** | SST |
| **Testing** | Bun test |

## AI Provider Support

OpenCode supports multiple AI providers for flexibility:

- Anthropic (Claude)
- OpenAI
- Google Vertex AI
- Amazon Bedrock
- Local models

## Protocol Support

| Protocol | Description | Implementation |
|----------|-------------|----------------|
| **MCP** | Model Context Protocol | `packages/opencode/src/mcp` |
| **ACP** | Agent Client Protocol | `packages/opencode/src/acp` |
| **LSP** | Language Server Protocol | `packages/opencode/src/lsp` |

## Project Structure

OpenCode is organized as a monorepo with 20 packages:

### Core Package

- **opencode** (`packages/opencode`) - Main CLI package with agent orchestration
  - Entry point: `packages/opencode/src/index.ts:1`
  - Binary: `packages/opencode/bin/opencode`

### User Interfaces

- **tui** (`packages/tui`) - Terminal UI (Go-based, Bubbletea)
  - Entry point: `packages/tui/cmd/main.go:1`
- **desktop** (`packages/desktop`) - Desktop application (Vite + SolidJS)
  - Entry point: `packages/desktop/src/main.tsx:1`
- **web** (`packages/web`) - Documentation website (Astro)
  - Entry point: `packages/web/src/pages/index.astro:1`

### Console Packages

- **console/app** (`packages/console/app`) - Console application frontend
- **console/core** (`packages/console/core`) - Console core functionality
- **console/function** (`packages/console/function`) - Console backend functions
- **console/mail** (`packages/console/mail`) - Email functionality
- **console/resource** (`packages/console/resource`) - Resource management

### SDKs

- **sdk/js** (`packages/sdk/js`) - JavaScript SDK
- **sdk/go** (`packages/sdk/go`) - Go SDK

### Utilities & Libraries

- **function** (`packages/function`) - Function utilities
- **plugin** (`packages/plugin`) - Plugin system
- **script** (`packages/script`) - Script utilities
- **ui** (`packages/ui`) - UI component library (SolidJS)

### Integrations

- **slack** (`packages/slack`) - Slack integration

## Core Components

### Protocol Layer
- **acp** (`packages/opencode/src/acp`) - Agent Client Protocol implementation (4 files)
- **lsp** (`packages/opencode/src/lsp`) - Language Server Protocol client/server (4 files)
- **mcp** (`packages/opencode/src/mcp`) - Model Context Protocol integration (1 file)

### API Layer
- **cli** (`packages/opencode/src/cli`) - CLI command handlers (20 files)
- **server** (`packages/opencode/src/server`) - HTTP server using Hono (3 files)
- **tool** (`packages/opencode/src/tool`) - Tool implementations (36 files)

### Business Logic
- **agent** (`packages/opencode/src/agent`) - Agent orchestration and generation (2 files)
- **session** (`packages/opencode/src/session`) - Session management, message handling, prompts (11 files)

### Integration Layer
- **provider** (`packages/opencode/src/provider`) - AI provider integration (4 files)

### Data Layer
- **file** (`packages/opencode/src/file`) - File operations: fzf, ripgrep, watcher (6 files)

### Supporting Components
- **auth** (`packages/opencode/src/auth`) - Authentication via GitHub Copilot (2 files)
- **config** (`packages/opencode/src/config`) - Configuration management (2 files)

## Entry Points

| Entry Point | Purpose | Language |
|-------------|---------|----------|
| `packages/opencode/src/index.ts:1` | Main CLI entry | TypeScript |
| `packages/opencode/bin/opencode` | CLI binary | Shell |
| `packages/tui/cmd/main.go:1` | Terminal UI | Go |
| `packages/desktop/src/main.tsx:1` | Desktop app | TypeScript |
| `packages/web/src/pages/index.astro:1` | Documentation site | Astro |

## External Dependencies

Key external dependencies used across the project:

- `yargs` - CLI argument parsing
- `hono` - HTTP server framework
- `ai` - AI SDK for unified provider interface
- `@anthropic-ai/sdk` - Anthropic Claude integration
- `@modelcontextprotocol/sdk` - MCP protocol implementation
- `@agentclientprotocol/sdk` - ACP protocol implementation
- `tree-sitter` - Code parsing and analysis
- `@parcel/watcher` - File system watching
- `chokidar` - File system watching (alternative)
- `zod` - Schema validation
- `solid-js` - Reactive UI framework
- `vite` - Frontend build tool
- `astro` - Static site generator
- `sst` - Infrastructure as code
- `@openauthjs/openauth` - Authentication framework

## Testing

- **Framework**: Bun test
- **Test Files**: 5 test files
- **Coverage**: Tests present for core functionality

## Project Type

OpenCode is categorized as an **AI coding agent** that combines multiple frameworks:

- CLI framework (Yargs)
- Web frameworks (Astro, SolidJS, Vite)
- Backend frameworks (Hono, SST)
- TUI framework (Bubbletea in Go)
