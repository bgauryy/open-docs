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

### OpenCode Settings

```bash
# Custom config path
export OPENCODE_CONFIG="/path/to/config.json"

# Inline config (JSON string)
export OPENCODE_CONFIG_CONTENT='{"provider":"anthropic"}'

# Working directory
export OPENCODE_DIRECTORY="/path/to/project"

# Debug logging
export DEBUG="*"
export DEBUG="opencode:*"
export DEBUG="opencode:session,opencode:tool"
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

