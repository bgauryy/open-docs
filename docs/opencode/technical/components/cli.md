# CLI Commands

**Path**: `packages/opencode/src/cli`
**Type**: API Layer
**File Count**: 20

## Description

CLI command handlers for all user-facing commands.

## Purpose

The CLI component provides the command-line interface for OpenCode. It handles argument parsing, command routing, and orchestrates the execution of user commands.

## Key Features

- Command parsing with Yargs
- Argument validation
- Help documentation
- Command aliases
- Interactive prompts
- Error handling and user feedback

## Component Files

### Command Handlers (in `cmd/` directory)
- `run.ts` - Interactive chat session
- `serve.ts` - HTTP server
- `acp.ts` - Agent Client Protocol server
- `mcp.ts` - Model Context Protocol management
- `tui.ts` - Terminal UI launcher
- `auth.ts` - Authentication management
- `agent.ts` - Agent workflows
- Additional utility commands

### Core CLI Files
- `index.ts` - CLI entry point
- `bootstrap.ts` - Application initialization
- `config.ts` - Configuration management

## Dependencies

### Internal Dependencies
- `packages/opencode/src/session` - Session management (15 imports)

### External Dependencies
- `yargs` - CLI framework

## Available Commands

### run
Start interactive coding session.
```bash
opencode run [message..] [--model MODEL] [--continue] [--session ID]
```

### serve
Start HTTP server.
```bash
opencode serve [--port PORT]
```

### acp
Start Agent Client Protocol server.
```bash
opencode acp
```

### mcp
Manage Model Context Protocol integrations.
```bash
opencode mcp [add|list|remove]
```

### tui
Launch terminal UI.
```bash
opencode tui
```

### auth
Manage authentication.
```bash
opencode auth [login|logout|list]
```

### agent
Run predefined agent workflows.
```bash
opencode agent [name]
```

## Usage Examples

### Basic Usage
```bash
# Get help
opencode --help

# Run command with options
opencode run "help me" --model claude-3-opus

# Chain commands
opencode auth login && opencode run "start coding"
```

### Configuration
```bash
# Set default model
opencode config set model claude-3-sonnet

# View configuration
opencode config get

# Reset configuration
opencode config reset
```

## Bootstrap Process

The bootstrap process initializes OpenCode:

1. Load configuration from `.opencode/config.json`
2. Initialize project context
3. Set up file watchers
4. Initialize tool registry
5. Configure AI provider
6. Load MCP servers (if configured)

```typescript
import { bootstrap } from './cli/bootstrap';

await bootstrap({
  cwd: process.cwd(),
  config: './opencode/config.json'
});
```

## Error Handling

The CLI provides user-friendly error messages:

```bash
$ opencode run

Error: No message provided
Usage: opencode run [message..] [options]

Try 'opencode run --help' for more information.
```

## Command Completion

OpenCode supports shell completion:

```bash
# Bash
opencode completion bash > /etc/bash_completion.d/opencode

# Zsh
opencode completion zsh > /usr/local/share/zsh/site-functions/_opencode

# Fish
opencode completion fish > ~/.config/fish/completions/opencode.fish
```

## Related Documentation

- [CLI Commands API Reference](../api-reference.md#cli-commands)
- [Interactive Chat Flow](../flows/run-command-flow.md)
- [Server Mode Flow](../flows/serve-command-flow.md)
