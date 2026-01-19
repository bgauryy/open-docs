# OpenCode - Configuration

> **Complete configuration reference for OpenCode**

---

## Overview

OpenCode supports configuration at multiple levels:
- **Global** - `~/.opencode/config.json`
- **Project** - `.opencode/config.json`
- **CLI flags** - Command-line overrides
- **Environment variables** - Runtime settings

**Files**:
- `config/config.ts` (757 lines, 28KB) - Configuration system
- `flag/flag.ts` - CLI flag definitions

---

## Configuration Files

### Structure

**config.json** (or **config.jsonc** with comments):
```json
{
  "provider": "anthropic",
  "model": "claude-3-5-sonnet",
  
  "instructions": [
    "docs/coding-standards.md",
    "~/my-global-rules.md"
  ],
  
  "agents": {
    "default": {
      "permission": {
        "edit": "ask",
        "bash": { "*": "ask" },
        "webfetch": "allow"
      }
    }
  },
  
  "lsp": {
    "typescript": {
      "enabled": true,
      "command": "typescript-language-server",
      "args": ["--stdio"]
    }
  },
  
  "mcp": {
    "servers": {
      "database": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-postgres"],
        "env": {
          "DATABASE_URL": "${DATABASE_URL}"
        }
      }
    }
  },
  
  "exclude": [
    "**/node_modules/**",
    "**/.git/**",
    "**/dist/**"
  ]
}
```

### Discovery Order

1. **Global**: `~/.opencode/config.json`
2. **Project ancestors** (bottom-up): `.opencode/config.json`
3. **CLI override**: `--config path/to/config.json`
4. **Environment**: `OPENCODE_CONFIG_CONTENT` (JSON string)

**Merging**: Deep merge, project overrides global.

---

## Configuration Options

### Provider & Model

```json
{
  "provider": "anthropic" | "openai" | "google" | "bedrock" | "ollama",
  "model": "claude-3-5-sonnet" | "gpt-4o" | "gemini-1.5-pro",
  "temperature": 0.7,
  "maxTokens": 4096
}
```

### Instructions

```json
{
  "instructions": [
    "docs/guidelines.md",      // Relative to project
    "~/global-rules.md",       // Home directory
    ".github/**/*.md"          // Glob pattern
  ]
}
```

### Agents

```json
{
  "agents": {
    "default": {
      "permission": {
        "edit": "allow" | "deny" | "ask",
        "bash": {
          "*": "ask",
          "npm test": "allow",
          "rm -rf": "deny"
        },
        "webfetch": "ask"
      },
      "system": "Custom system prompt",
      "tools": {
        "bash": true,
        "edit": true,
        "webfetch": false
      }
    },
    "readonly": {
      "permission": {
        "edit": "deny",
        "bash": { "*": "deny" },
        "webfetch": "allow"
      }
    }
  }
}
```

### LSP

```json
{
  "lsp": {
    "typescript": {
      "enabled": true,
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "initializationOptions": {}
    },
    "python": {
      "enabled": true,
      "command": "pyright-langserver",
      "args": ["--stdio"]
    }
  }
}
```

### MCP

```json
{
  "mcp": {
    "servers": {
      "server-name": {
        "command": "command-to-run",
        "args": ["arg1", "arg2"],
        "env": {
          "KEY": "value",
          "TOKEN": "${ENV_VAR}"
        }
      }
    }
  }
}
```

### File Exclusions

```json
{
  "exclude": [
    "**/node_modules/**",
    "**/.git/**",
    "**/dist/**",
    "**/*.test.ts"
  ]
}
```

---

## Environment Variables

### Provider API Keys

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
export GOOGLE_API_KEY="..."
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
```

### Core Configuration Variables

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `OPENCODE_AUTO_SHARE` | Automatically share sessions to cloud | `false` | `true` |
| `OPENCODE_CONFIG` | Custom config file path | `~/.opencode/config.json` | `/path/to/config.json` |
| `OPENCODE_CONFIG_DIR` | Custom config directory | `~/.opencode` | `/custom/config/dir` |
| `OPENCODE_CONFIG_CONTENT` | Inline JSON config content | - | `{"model":"claude-3-5-sonnet"}` |
| `OPENCODE_PERMISSION` | Default permission mode | `default` | `auto-approve`, `yolo` |
| `OPENCODE_CLIENT` | Client identifier | `cli` | `ide`, `api` |
| `OPENCODE_SERVER_PASSWORD` | HTTP Basic Auth password | - | `secret123` |
| `OPENCODE_SERVER_USERNAME` | HTTP Basic Auth username | - | `admin` |
| `OPENCODE_GIT_BASH_PATH` | Custom Git Bash path (Windows) | Auto-detect | `C:\Git\bin\bash.exe` |
| `OPENCODE_DIRECTORY` | Working directory | Current dir | `/path/to/project` |

### Disable Flags

These flags disable specific features. Set to any truthy value (`true`, `1`, `yes`) to disable.

#### `OPENCODE_DISABLE_AUTOUPDATE`
Disables automatic update checks on startup.
```bash
export OPENCODE_DISABLE_AUTOUPDATE=true
```
**Use when**: Running in CI/CD, air-gapped environments, or managing updates manually.

#### `OPENCODE_DISABLE_PRUNE`
Disables automatic session pruning (cleanup of old sessions).
```bash
export OPENCODE_DISABLE_PRUNE=true
```
**Use when**: You want to keep all session history indefinitely.

#### `OPENCODE_DISABLE_TERMINAL_TITLE`
Prevents OpenCode from modifying the terminal title.
```bash
export OPENCODE_DISABLE_TERMINAL_TITLE=true
```
**Use when**: Terminal title management conflicts with other tools.

#### `OPENCODE_DISABLE_DEFAULT_PLUGINS`
Skips loading of default bundled plugins.
```bash
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
```
**Use when**: You want a minimal installation or have conflicting plugins.

#### `OPENCODE_DISABLE_LSP_DOWNLOAD`
Prevents automatic download of LSP servers.
```bash
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
```
**Use when**: Managing LSP servers manually or in restricted environments.

#### `OPENCODE_DISABLE_AUTOCOMPACT`
Disables automatic session compaction.
```bash
export OPENCODE_DISABLE_AUTOCOMPACT=true
```
**Use when**: You prefer manual compaction or have specific context requirements.

#### `OPENCODE_DISABLE_MODELS_FETCH`
Prevents fetching available models from provider APIs.
```bash
export OPENCODE_DISABLE_MODELS_FETCH=true
```
**Use when**: Offline mode or when using fixed model configurations.

#### `OPENCODE_DISABLE_CLAUDE_CODE`
Disables all Claude Code compatibility features.
```bash
export OPENCODE_DISABLE_CLAUDE_CODE=true
```
**Use when**: You don't need Claude Code compatibility and want a leaner setup.

#### `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT`
Skips Claude Code-style prompt formatting.
```bash
export OPENCODE_DISABLE_CLAUDE_CODE_PROMPT=true
```
**Use when**: Using custom prompt styles incompatible with Claude Code format.

#### `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS`
Skips discovery of `.claude/skills/` directory.
```bash
export OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=true
```
**Use when**: Not using Claude Code skills or having conflicting skill definitions.

#### `OPENCODE_DISABLE_SHARE`
Completely disables session sharing functionality.
```bash
export OPENCODE_DISABLE_SHARE=true
```
**Use when**: Privacy requirements or air-gapped environments.

### Experimental Flags

These flags enable features that are in development or testing. They may change or be removed.

#### `OPENCODE_EXPERIMENTAL`
**Master switch** - Enables ALL experimental features at once.
```bash
export OPENCODE_EXPERIMENTAL=true
```
**Warning**: May introduce instability. Use individual flags for production.

#### `OPENCODE_ENABLE_EXA`
Enables Exa AI-powered web and code search.
```bash
export OPENCODE_ENABLE_EXA=true
```
**Required for**: `websearch` and `codesearch` tools.

#### `OPENCODE_EXPERIMENTAL_PLAN_MODE`
Enables plan mode for research and planning workflows.
```bash
export OPENCODE_EXPERIMENTAL_PLAN_MODE=true
```
**Enables**: `plan_enter` and `plan_exit` tools.

#### `OPENCODE_EXPERIMENTAL_LSP_TOOL`
Enables LSP tools for code intelligence.
```bash
export OPENCODE_EXPERIMENTAL_LSP_TOOL=true
```
**Enables**: `lsp-diagnostics`, `lsp-hover` tools.

#### `OPENCODE_EXPERIMENTAL_FILEWATCHER`
Uses the new file watcher implementation.
```bash
export OPENCODE_EXPERIMENTAL_FILEWATCHER=true
```
**Benefits**: Better performance, lower resource usage.

#### `OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER`
Completely disables file watching.
```bash
export OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER=true
```
**Use when**: File watching causes issues or is unnecessary.

#### `OPENCODE_EXPERIMENTAL_ICON_DISCOVERY`
Enables icon discovery feature in TUI.
```bash
export OPENCODE_EXPERIMENTAL_ICON_DISCOVERY=true
```

#### `OPENCODE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT`
Disables automatic clipboard copy when selecting text.
```bash
export OPENCODE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT=true
```

#### `OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH`
Sets maximum output length for bash tool (in bytes).
```bash
export OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH=200000  # 200KB
```
**Default**: 100000 (100KB)

#### `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS`
Sets default timeout for bash commands (in milliseconds).
```bash
export OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS=60000  # 60 seconds
```
**Default**: 30000 (30 seconds)

#### `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX`
Sets maximum output tokens for LLM responses.
```bash
export OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX=8192
```
**Default**: Provider-specific

#### `OPENCODE_EXPERIMENTAL_OXFMT`
Enables experimental output formatter.
```bash
export OPENCODE_EXPERIMENTAL_OXFMT=true
```

#### `OPENCODE_EXPERIMENTAL_LSP_TY`
Enables LSP type feature (experimental).
```bash
export OPENCODE_EXPERIMENTAL_LSP_TY=true
```

### Configuration Precedence

Environment variables are loaded in this order (later overrides earlier):

1. **Default values** (built into OpenCode)
2. **System config** (`/etc/opencode/config.json`)
3. **User config** (`~/.opencode/config.json`)
4. **Project config** (`.opencode/config.json`)
5. **Environment variables** (`OPENCODE_*`)
6. **CLI arguments** (`--model`, `--permission`, etc.)
7. **Inline config** (`OPENCODE_CONFIG_CONTENT`)

### Configuration Examples

#### Minimal Privacy Setup
```bash
export OPENCODE_DISABLE_SHARE=true
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_MODELS_FETCH=true
```

#### Full Experimental Mode
```bash
export OPENCODE_EXPERIMENTAL=true
export OPENCODE_ENABLE_EXA=true
```

#### CI/CD Optimized
```bash
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_TERMINAL_TITLE=true
export OPENCODE_PERMISSION=auto-approve
```

#### Development Setup
```bash
export OPENCODE_EXPERIMENTAL_PLAN_MODE=true
export OPENCODE_EXPERIMENTAL_LSP_TOOL=true
export OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS=60000
```

---

## CLI Flags

### Global Flags

```bash
--config PATH              # Custom config file
--provider PROVIDER        # Override provider
--model MODEL              # Override model
--agent AGENT              # Use specific agent
--system PROMPT            # Custom system prompt
--directory PATH           # Working directory
--no-color                 # Disable color output
--json                     # JSON output
--debug                    # Enable debug logging
```

### Command-Specific

```bash
# run
opencode run --continue --attach file.ts "prompt"

# tui
opencode tui --port 8080

# serve
opencode serve --port 8080 --host 0.0.0.0

# models
opencode models --json
```

---

## Configuration Directories

### Global Directory

`~/.opencode/`
```
~/.opencode/
├── config.json           # Global config
├── agent/                # Custom agents
│   ├── default.md
│   └── security.md
├── tool/                 # Custom tools
│   └── mytool.ts
└── cache/                # Cache directory
```

### Project Directory

`.opencode/`
```
.opencode/
├── config.json           # Project config
├── agent/                # Project agents
└── tool/                 # Project tools
```

---

## Example Configurations

### Minimal

```json
{
  "provider": "anthropic",
  "model": "claude-3-5-sonnet"
}
```

### Development

```json
{
  "provider": "anthropic",
  "model": "claude-3-5-sonnet",
  "instructions": ["AGENTS.md"],
  "exclude": ["**/node_modules/**", "**/dist/**"]
}
```

### Production

```json
{
  "provider": "bedrock",
  "model": "anthropic.claude-3-sonnet",
  "agents": {
    "default": {
      "permission": {
        "edit": "ask",
        "bash": {
          "*": "deny",
          "npm test": "allow"
        }
      }
    }
  },
  "lsp": { "typescript": { "enabled": true } },
  "mcp": {
    "servers": {
      "database": {...}
    }
  }
}
```

---

## Best Practices

**Security**:
- Never commit API keys
- Use environment variables
- Restrict bash permissions
- Review auto-generated configs

**Organization**:
- Use project configs for project-specific settings
- Use global configs for personal preferences
- Document custom settings

**Performance**:
- Exclude large directories
- Configure LSP per-language
- Limit token usage
- Cache when possible

---

For implementation, see `packages/opencode/src/config/`.



---

# Enhanced Configuration Documentation

---

## Environment Variables - Complete Reference

### Core Configuration Variables

| Variable | Description | Default | Example |
|----------|-------------|---------|---------|
| `OPENCODE_AUTO_SHARE` | Automatically share sessions to cloud | `false` | `true` |
| `OPENCODE_CONFIG` | Custom config file path | `~/.opencode/config.json` | `/path/to/config.json` |
| `OPENCODE_CONFIG_DIR` | Custom config directory | `~/.opencode` | `/custom/config/dir` |
| `OPENCODE_CONFIG_CONTENT` | Inline JSON config content | - | `{"model":"claude-3-5-sonnet"}` |
| `OPENCODE_PERMISSION` | Default permission mode | `default` | `auto-approve`, `yolo` |
| `OPENCODE_CLIENT` | Client identifier | `cli` | `ide`, `api` |
| `OPENCODE_SERVER_PASSWORD` | HTTP Basic Auth password | - | `secret123` |
| `OPENCODE_SERVER_USERNAME` | HTTP Basic Auth username | - | `admin` |
| `OPENCODE_GIT_BASH_PATH` | Custom Git Bash path (Windows) | Auto-detect | `C:\Git\bin\bash.exe` |

---

## Disable Flags

These flags disable specific features. Set to any truthy value (`true`, `1`, `yes`) to disable.

### `OPENCODE_DISABLE_AUTOUPDATE`
Disables automatic update checks on startup.
```bash
export OPENCODE_DISABLE_AUTOUPDATE=true
```
**Use when**: Running in CI/CD, air-gapped environments, or managing updates manually.

### `OPENCODE_DISABLE_PRUNE`
Disables automatic session pruning (cleanup of old sessions).
```bash
export OPENCODE_DISABLE_PRUNE=true
```
**Use when**: You want to keep all session history indefinitely.

### `OPENCODE_DISABLE_TERMINAL_TITLE`
Prevents OpenCode from modifying the terminal title.
```bash
export OPENCODE_DISABLE_TERMINAL_TITLE=true
```
**Use when**: Terminal title management conflicts with other tools.

### `OPENCODE_DISABLE_DEFAULT_PLUGINS`
Skips loading of default bundled plugins.
```bash
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
```
**Use when**: You want a minimal installation or have conflicting plugins.

### `OPENCODE_DISABLE_LSP_DOWNLOAD`
Prevents automatic download of LSP servers.
```bash
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
```
**Use when**: Managing LSP servers manually or in restricted environments.

### `OPENCODE_DISABLE_AUTOCOMPACT`
Disables automatic session compaction.
```bash
export OPENCODE_DISABLE_AUTOCOMPACT=true
```
**Use when**: You prefer manual compaction or have specific context requirements.

### `OPENCODE_DISABLE_MODELS_FETCH`
Prevents fetching available models from provider APIs.
```bash
export OPENCODE_DISABLE_MODELS_FETCH=true
```
**Use when**: Offline mode or when using fixed model configurations.

### `OPENCODE_DISABLE_CLAUDE_CODE`
Disables all Claude Code compatibility features.
```bash
export OPENCODE_DISABLE_CLAUDE_CODE=true
```
**Use when**: You don't need Claude Code compatibility and want a leaner setup.

### `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT`
Skips Claude Code-style prompt formatting.
```bash
export OPENCODE_DISABLE_CLAUDE_CODE_PROMPT=true
```
**Use when**: Using custom prompt styles incompatible with Claude Code format.

### `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS`
Skips discovery of `.claude/skills/` directory.
```bash
export OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=true
```
**Use when**: Not using Claude Code skills or having conflicting skill definitions.

### `OPENCODE_DISABLE_SHARE`
Completely disables session sharing functionality.
```bash
export OPENCODE_DISABLE_SHARE=true
```
**Use when**: Privacy requirements or air-gapped environments.

---

## Experimental Flags

These flags enable features that are in development or testing. They may change or be removed.

### `OPENCODE_EXPERIMENTAL`
**Master switch** - Enables ALL experimental features at once.
```bash
export OPENCODE_EXPERIMENTAL=true
```
**Warning**: May introduce instability. Use individual flags for production.

### `OPENCODE_ENABLE_EXA`
Enables Exa AI-powered web and code search.
```bash
export OPENCODE_ENABLE_EXA=true
```
**Required for**: `websearch` and `codesearch` tools.

### `OPENCODE_EXPERIMENTAL_PLAN_MODE`
Enables plan mode for research and planning workflows.
```bash
export OPENCODE_EXPERIMENTAL_PLAN_MODE=true
```
**Enables**: `plan_enter` and `plan_exit` tools.

### `OPENCODE_EXPERIMENTAL_LSP_TOOL`
Enables LSP tools for code intelligence.
```bash
export OPENCODE_EXPERIMENTAL_LSP_TOOL=true
```
**Enables**: `lsp-diagnostics`, `lsp-hover` tools.

### `OPENCODE_EXPERIMENTAL_FILEWATCHER`
Uses the new file watcher implementation.
```bash
export OPENCODE_EXPERIMENTAL_FILEWATCHER=true
```
**Benefits**: Better performance, lower resource usage.

### `OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER`
Completely disables file watching.
```bash
export OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER=true
```
**Use when**: File watching causes issues or is unnecessary.

### `OPENCODE_EXPERIMENTAL_ICON_DISCOVERY`
Enables icon discovery feature in TUI.
```bash
export OPENCODE_EXPERIMENTAL_ICON_DISCOVERY=true
```

### `OPENCODE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT`
Disables automatic clipboard copy when selecting text.
```bash
export OPENCODE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT=true
```

### `OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH`
Sets maximum output length for bash tool (in bytes).
```bash
export OPENCODE_EXPERIMENTAL_BASH_MAX_OUTPUT_LENGTH=200000  # 200KB
```
**Default**: 100000 (100KB)

### `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS`
Sets default timeout for bash commands (in milliseconds).
```bash
export OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS=60000  # 60 seconds
```
**Default**: 30000 (30 seconds)

### `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX`
Sets maximum output tokens for LLM responses.
```bash
export OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX=8192
```
**Default**: Provider-specific

### `OPENCODE_EXPERIMENTAL_OXFMT`
Enables experimental output formatter.
```bash
export OPENCODE_EXPERIMENTAL_OXFMT=true
```

### `OPENCODE_EXPERIMENTAL_LSP_TY`
Enables LSP type feature (experimental).
```bash
export OPENCODE_EXPERIMENTAL_LSP_TY=true
```

---

## Configuration Precedence

Environment variables are loaded in this order (later overrides earlier):

1. **Default values** (built into OpenCode)
2. **System config** (`/etc/opencode/config.json`)
3. **User config** (`~/.opencode/config.json`)
4. **Project config** (`.opencode/config.json`)
5. **Environment variables** (`OPENCODE_*`)
6. **CLI arguments** (`--model`, `--permission`, etc.)
7. **Inline config** (`OPENCODE_CONFIG_CONTENT`)

---

## Configuration Examples

### Minimal Privacy Setup
```bash
export OPENCODE_DISABLE_SHARE=true
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_MODELS_FETCH=true
```

### Full Experimental Mode
```bash
export OPENCODE_EXPERIMENTAL=true
export OPENCODE_ENABLE_EXA=true
```

### CI/CD Optimized
```bash
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_TERMINAL_TITLE=true
export OPENCODE_PERMISSION=auto-approve
```

### Development Setup
```bash
export OPENCODE_EXPERIMENTAL_PLAN_MODE=true
export OPENCODE_EXPERIMENTAL_LSP_TOOL=true
export OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS=60000
```

---

## Related Documentation

- [02-cli-reference.md](./02-cli-reference.md) - CLI options
- [14-security-permissions.md](./14-security-permissions.md) - Permission modes
- [24-development-guide.md](./24-development-guide.md) - Development setup
