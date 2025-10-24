# Claude Agent SDK - Configuration Complete Reference

**SDK Version**: 0.1.22
**Source**: `sdkTypes.d.ts`

---

## Table of Contents

1. [Overview](#overview)
2. [Settings File Locations](#settings-file-locations)
3. [Settings Resolution Order](#settings-resolution-order)
4. [Settings Schema](#settings-schema)
5. [Environment Variables](#environment-variables)
6. [CLI Flags](#cli-flags)
7. [Programmatic Options](#programmatic-options)
8. [Configuration Examples](#configuration-examples)
9. [Gotchas & Best Practices](#gotchas--best-practices)

---

## Overview

The Claude Agent SDK uses a **multi-level configuration system** with 6 resolution sources:

```
CLI Flags (highest priority)
     ↓
Session Settings (in-memory, temporary)
     ↓
Local Settings (.claude/ in cwd)
     ↓
Project Settings (.claude/ in git root)
     ↓
User Settings (~/.claude/)
     ↓
Policy Settings (enterprise)
     ↓
Default Values (lowest priority)
```

**First match wins** - Once a setting is found at any level, resolution stops.

---

## Settings File Locations

### Directory Structure

```
~/.claude/                          # User settings (global)
├── settings.json                   # User configuration
├── sessions/                       # Session storage
│   ├── <session-id-1>/
│   │   ├── transcript.json        # Conversation history
│   │   ├── checkpoints/           # Auto-checkpoints
│   │   │   ├── checkpoint-0.json
│   │   │   ├── checkpoint-5.json
│   │   │   └── checkpoint-10.json
│   │   └── file-snapshots/        # File content snapshots
│   │       ├── <hash-1>.txt
│   │       └── <hash-2>.txt
│   └── <session-id-2>/
│       └── ...
├── history.jsonl                   # Command history
└── cache/                          # Cache storage
    └── mcp/                        # MCP cache

<git-root>/.claude/                 # Project settings (team, committed)
├── settings.json                   # Project configuration
└── skills/                         # Project skills
    └── <skill-name>/
        └── SKILL.md

<cwd>/.claude/                      # Local settings (gitignored)
├── settings.json                   # Local configuration
└── skills/                         # Local skills (personal)

/etc/claude/                        # Policy settings (enterprise, optional)
└── settings.json                   # Organization-wide config
```

### File Locations by Precedence

| Level | Location | Scope | Persisted | Shared |
|-------|----------|-------|-----------|--------|
| **CLI Flags** | Command line | Current session | No | No |
| **Session** | In-memory | Current session | No | No |
| **Local** | `<cwd>/.claude/` | Local directory | Yes | No (gitignored) |
| **Project** | `<git-root>/.claude/` | Git repository | Yes | Yes (committed) |
| **User** | `~/.claude/` | User global | Yes | No |
| **Policy** | `/etc/claude/` | Enterprise-wide | Yes | Yes (managed) |
| **Default** | SDK | SDK default | No | N/A |

---

## Settings Resolution Order

### Resolution Algorithm

```typescript
function resolveConfig<T>(key: string): T {
  // 1. CLI flags (highest priority)
  if (cliFlags.has(key)) return cliFlags.get(key);
  
  // 2. Session settings (temporary, in-memory)
  if (sessionSettings.has(key)) return sessionSettings.get(key);
  
  // 3. Local settings (.claude/ in cwd)
  if (localSettings.has(key)) return localSettings.get(key);
  
  // 4. Project settings (.claude/ in git root)
  if (projectSettings.has(key)) return projectSettings.get(key);
  
  // 5. User settings (~/.claude/)
  if (userSettings.has(key)) return userSettings.get(key);
  
  // 6. Policy settings (enterprise)
  if (policySettings.has(key)) return policySettings.get(key);
  
  // 7. Default value
  return DEFAULT_CONFIG[key];
}
```

### Example Resolution

```typescript
// Scenario: permissionMode setting

// CLI flag
--permission-mode acceptEdits  // ← Wins (highest priority)

// Session setting (if no CLI flag)
sessionSettings.permissionMode = "bypassPermissions"

// Local setting (if no CLI flag or session)
<cwd>/.claude/settings.json:
{ "permissionMode": "default" }

// Project setting (if none above)
<git-root>/.claude/settings.json:
{ "permissionMode": "acceptEdits" }

// User setting (if none above)
~/.claude/settings.json:
{ "permissionMode": "default" }

// Policy setting (if none above)
/etc/claude/settings.json:
{ "permissionMode": "default" }

// Default (if no settings anywhere)
DEFAULT_PERMISSION_MODE = "default"
```

---

## Settings Schema

### Complete settings.json Schema

```typescript
interface Settings {
  // Permission Configuration
  permissions?: {
    mode?: PermissionMode;           // 'default' | 'acceptEdits' | 'bypassPermissions' | 'plan'
    rules?: PermissionRule[];        // Permission rules
  };
  
  // MCP Server Configuration
  mcpServers?: Record<string, McpServerConfig>;
  
  // Agent Configuration
  agents?: Record<string, AgentDefinition>;
  
  // Tool Configuration
  allowedTools?: string[];           // Tool whitelist
  disallowedTools?: string[];        // Tool blacklist
  
  // Model Configuration
  defaultModel?: string;             // Default model ID
  fallbackModel?: string;            // Fallback model
  
  // Session Configuration
  maxTurns?: number;                 // Max conversation turns
  maxThinkingTokens?: number;        // Extended thinking limit
  
  // Directory Access
  additionalDirectories?: string[];  // Extra permitted directories
  
  // UI Configuration
  statusLine?: StatusLineConfig;     // Status line settings
  outputStyle?: OutputStyleConfig;   // Output formatting
  
  // Feature Flags
  strictMcpConfig?: boolean;         // Strict MCP validation
  
  // Hook Configuration
  hooks?: HookConfig;                // Hook definitions
}
```

### settings.json Example (Complete)

```json
{
  "permissions": {
    "mode": "acceptEdits",
    "rules": [
      {
        "tool": "Bash",
        "command": "npm test",
        "behavior": "allow",
        "reason": "Always allow running tests"
      },
      {
        "tool": "Write",
        "path": "src/**",
        "behavior": "allow",
        "reason": "Allow source file edits"
      },
      {
        "tool": "Delete",
        "behavior": "deny",
        "reason": "Never auto-delete"
      }
    ]
  },
  
  "mcpServers": {
    "filesystem": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    },
    "github": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  },
  
  "agents": {
    "code-reviewer": {
      "description": "Reviews code for quality and security",
      "tools": ["Read", "Grep", "Bash(git*)"],
      "prompt": "You are a code reviewer. Check for security issues, bugs, and style violations.",
      "model": "sonnet"
    },
    "test-generator": {
      "description": "Generates unit tests",
      "tools": ["Read", "Write", "Grep"],
      "prompt": "Generate comprehensive unit tests with edge cases.",
      "model": "haiku"
    }
  },
  
  "allowedTools": [
    "Read",
    "Write",
    "Edit",
    "Glob",
    "Grep",
    "Bash",
    "Task"
  ],
  
  "disallowedTools": [
    "Delete",
    "WebSearch"
  ],
  
  "defaultModel": "claude-3-5-sonnet-20241022",
  "fallbackModel": "claude-3-5-haiku-20241022",
  
  "maxTurns": 100,
  "maxThinkingTokens": 10000,
  
  "additionalDirectories": [
    "/tmp/workspace",
    "/home/user/other-project"
  ],
  
  "strictMcpConfig": true,
  
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "script": "./hooks/validate-bash.js"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "*",
        "script": "./hooks/log-tool-usage.js"
      }
    ]
  }
}
```

---

## Environment Variables

### Core Environment Variables

```bash
# API Authentication
ANTHROPIC_API_KEY="sk-ant-..."           # Required: Anthropic API key

# Model Selection
ANTHROPIC_MODEL="claude-3-5-sonnet-20241022"  # Override default model

# Debugging
DEBUG="claude:*"                         # Enable debug logging
DEBUG="claude:tools"                     # Tool execution only
DEBUG="claude:mcp"                       # MCP only

# Terminal Configuration
NO_COLOR=1                               # Disable colored output
TERM="xterm-256color"                    # Terminal type
SHELL="/bin/zsh"                         # Shell for Bash tool

# MCP Configuration
MCP_SERVER_TIMEOUT=30000                 # MCP connection timeout (ms)

# Cache Configuration
CLAUDE_CACHE_DIR="~/.claude/cache"       # Cache directory

# Feature Flags
CLAUDE_EXPERIMENTAL_FEATURES=1           # Enable experimental features
```

### Environment Variable Usage

**Setting Environment Variables**:

```bash
# One-time (current command)
ANTHROPIC_API_KEY="sk-ant-..." claude-code

# Session (current shell)
export ANTHROPIC_API_KEY="sk-ant-..."
export DEBUG="claude:*"
claude-code

# Permanent (add to ~/.bashrc, ~/.zshrc, etc.)
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.zshrc
source ~/.zshrc
```

**`.env` File Support** (if supported):

```bash
# .env in project root or cwd
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_MODEL=claude-3-5-sonnet-20241022
DEBUG=claude:tools
```

---

## CLI Flags

### Complete CLI Flag Reference

```bash
# Session Management
claude-code                              # Start new session
claude-code --resume <session-id>       # Resume session
claude-code --fork-session <session-id> # Fork (copy) session
claude-code --resume-session-at <msg-id> # Resume from specific message

# Model Configuration
claude-code --model opus                 # Use Opus model
claude-code --model sonnet               # Use Sonnet (default)
claude-code --model haiku                # Use Haiku

# Permission Configuration
claude-code --permission-mode default
claude-code --permission-mode acceptEdits
claude-code --permission-mode bypassPermissions
claude-code --permission-mode plan

# Tool Configuration
claude-code --allowed-tools Read,Write,Bash
claude-code --disallowed-tools Delete,WebSearch

# Directory Access
claude-code --cwd /path/to/project
claude-code --additional-directories /tmp,/home/user/other

# Conversation Limits
claude-code --max-turns 50               # Max 50 turns
claude-code --max-thinking-tokens 20000  # Extended thinking limit

# MCP Configuration
claude-code --strict-mcp-config          # Strict MCP validation
claude-code --mcp-server-timeout 60000   # MCP timeout (ms)

# Debugging
claude-code --debug                      # Enable debug mode
claude-code --verbose                    # Verbose output

# Output Configuration
claude-code --no-color                   # Disable colors
claude-code --output-format json         # JSON output
```

### CLI Flag Examples

**Example 1: Rapid Development Mode**:
```bash
claude-code \
  --permission-mode acceptEdits \
  --model haiku \
  --allowed-tools Read,Write,Edit,Grep,Glob,Bash \
  --max-thinking-tokens 5000
```

**Example 2: Secure Review Mode**:
```bash
claude-code \
  --permission-mode plan \
  --allowed-tools Read,Grep,Glob \
  --cwd /path/to/code
```

**Example 3: CI/CD Mode**:
```bash
claude-code \
  --permission-mode bypassPermissions \
  --model haiku \
  --max-turns 10 \
  --no-color \
  --output-format json
```

---

## Programmatic Options

### SDK Options Interface

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const result = await query({
  prompt: "Your task here",
  options: {
    // API Configuration
    apiKey?: string;                     // API key (overrides env)
    
    // Model Configuration
    model?: string;                      // Model ID
    fallbackModel?: string;              // Fallback model
    maxThinkingTokens?: number;          // Extended thinking limit
    
    // Permission Configuration
    permissionMode?: PermissionMode;
    permissions?: PermissionRule[];
    additionalDirectories?: string[];
    
    // Tool Configuration
    allowedTools?: string[];
    disallowedTools?: string[];
    
    // Agent Configuration
    agents?: Record<string, AgentDefinition>;
    
    // MCP Configuration
    mcpServers?: Record<string, McpServerConfig>;
    strictMcpConfig?: boolean;
    
    // Session Configuration
    maxTurns?: number;
    forkSession?: string;                // Fork from session ID
    resumeSessionAt?: string;            // Resume at message ID
    
    // Hook Configuration
    hooks?: Record<HookEvent, HookCallbackMatcher[]>;
    
    // System Prompt Configuration
    systemPrompt?: string | {
      type: 'preset';
      preset: 'claude_code';
      append?: string;
    };
    
    // Feature Flags
    includePartialMessages?: boolean;
    executableRuntime?: 'node' | 'bun' | 'deno';
  }
});
```

### Programmatic Configuration Examples

**Example 1: Custom Agent with Hooks**:

```typescript
const result = await query({
  prompt: "Analyze the codebase",
  options: {
    model: 'claude-3-5-sonnet-20241022',
    permissionMode: 'acceptEdits',
    
    agents: {
      'analyzer': {
        description: 'Code analysis specialist',
        tools: ['Read', 'Grep', 'Glob'],
        prompt: 'Analyze code quality, performance, and security',
        model: 'sonnet'
      }
    },
    
    hooks: {
      PreToolUse: [{
        hooks: [async (input) => {
          console.log(`Tool: ${input.tool_name}`);
          return { decision: 'approve' };
        }]
      }],
      
      PostToolUse: [{
        hooks: [async (input) => {
          console.log(`Result: ${input.tool_response}`);
          return {};
        }]
      }]
    }
  }
});
```

**Example 2: MCP Integration**:

```typescript
const result = await query({
  prompt: "List files and fetch web content",
  options: {
    mcpServers: {
      'filesystem': {
        transport: 'stdio',
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-filesystem', '/workspace']
      },
      'web-fetch': {
        transport: 'http',
        url: 'http://localhost:3000/mcp'
      }
    },
    strictMcpConfig: true
  }
});
```

---

## Configuration Examples

### Example 1: Team Project Configuration

**File**: `.claude/settings.json` (git root, committed)

```json
{
  "permissions": {
    "mode": "default",
    "rules": [
      {
        "tools": ["Read", "Glob", "Grep"],
        "behavior": "allow",
        "reason": "Safe discovery always allowed"
      },
      {
        "tool": "Write",
        "paths": ["src/**/*.ts", "tests/**/*.ts"],
        "behavior": "allow",
        "reason": "TypeScript source and tests"
      },
      {
        "tool": "Write",
        "path": "package.json",
        "behavior": "ask",
        "reason": "Confirm dependency changes"
      },
      {
        "tool": "Bash",
        "command": "npm test",
        "behavior": "allow",
        "reason": "Safe to run tests"
      },
      {
        "tool": "Bash",
        "command": "npm run lint",
        "behavior": "allow",
        "reason": "Safe to run linter"
      },
      {
        "tool": "Delete",
        "behavior": "deny",
        "reason": "Never auto-delete files"
      }
    ]
  },
  
  "agents": {
    "test-generator": {
      "description": "Generate Jest tests",
      "tools": ["Read", "Write", "Grep"],
      "prompt": "Generate Jest tests following project conventions",
      "model": "sonnet"
    }
  },
  
  "defaultModel": "claude-3-5-sonnet-20241022",
  "fallbackModel": "claude-3-5-haiku-20241022",
  
  "mcpServers": {
    "github": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### Example 2: Personal User Configuration

**File**: `~/.claude/settings.json` (user global)

```json
{
  "permissions": {
    "mode": "acceptEdits"
  },
  
  "agents": {
    "quick-search": {
      "description": "Fast codebase search",
      "tools": ["Glob", "Grep", "Read"],
      "prompt": "Search quickly and summarize findings",
      "model": "haiku"
    }
  },
  
  "defaultModel": "claude-3-5-sonnet-20241022",
  
  "additionalDirectories": [
    "/Users/me/workspace",
    "/tmp/scratch"
  ]
}
```

### Example 3: Enterprise Policy Configuration

**File**: `/etc/claude/settings.json` (managed)

```json
{
  "permissions": {
    "mode": "default",
    "rules": [
      {
        "tool": "Bash",
        "command": "*prod*",
        "behavior": "deny",
        "reason": "No production access"
      },
      {
        "tool": "Bash",
        "command": "*production*",
        "behavior": "deny",
        "reason": "No production access"
      },
      {
        "tool": "WebFetch",
        "behavior": "deny",
        "reason": "No external web access"
      },
      {
        "tool": "WebSearch",
        "behavior": "deny",
        "reason": "No web search access"
      }
    ]
  },
  
  "strictMcpConfig": true,
  
  "maxThinkingTokens": 10000,
  
  "disallowedTools": [
    "WebFetch",
    "WebSearch"
  ]
}
```

---

## Gotchas & Best Practices

### Gotchas

1. **Settings Not Hot-Reloaded**:
   - Changing settings.json requires restart
   - Session settings cleared on exit

2. **Precedence Can Be Confusing**:
   ```json
   // User settings (~/.claude/)
   { "permissionMode": "bypassPermissions" }
   
   // Project settings (.claude/ in git root)
   { "permissionMode": "default" }
   
   // Result: User setting wins (higher precedence)
   // Project setting ignored!
   ```

3. **Environment Variable Substitution**:
   ```json
   {
     "mcpServers": {
       "github": {
         "env": {
           "GITHUB_TOKEN": "${GITHUB_TOKEN}"  // ✅ Substituted
         }
       }
     }
   }
   ```
   - Variables must exist in environment
   - No default value syntax

4. **Path Separators Platform-Specific**:
   ```json
   // ❌ Windows backslashes in JSON
   "path": "C:\\Users\\me\\project"
   
   // ✅ Forward slashes work everywhere
   "path": "C:/Users/me/project"
   ```

5. **CLI Flags Override Everything**:
   - Even policy settings
   - Can bypass security rules (be careful!)

### Best Practices

**1. Use Project Settings for Team Consistency**:
```bash
# Commit project settings to git
git add .claude/settings.json
git commit -m "Add Claude Code team settings"
```

**2. Use Local Settings for Personal Overrides**:
```bash
# .gitignore already includes .claude/ in cwd
echo "Personal settings here" > .claude/settings.json
```

**3. Document Configuration Decisions**:
```json
{
  "permissions": {
    "rules": [
      {
        "tool": "Delete",
        "behavior": "deny",
        "reason": "Team decision: no auto-deletes (2024-10-15)"
      }
    ]
  }
}
```

**4. Use Environment Variables for Secrets**:
```json
{
  "mcpServers": {
    "api": {
      "env": {
        "API_KEY": "${MY_API_KEY}"  // ✅ Never commit actual keys
      }
    }
  }
}
```

**5. Test Configuration Changes**:
```bash
# Test with dry-run or plan mode first
claude-code --permission-mode plan
```

---

## Summary

### Configuration Sources (Priority Order)

1. **CLI Flags** - Highest priority, current command only
2. **Session Settings** - In-memory, temporary
3. **Local Settings** - `.claude/` in cwd, gitignored
4. **Project Settings** - `.claude/` in git root, committed
5. **User Settings** - `~/.claude/`, user global
6. **Policy Settings** - `/etc/claude/`, enterprise-wide
7. **Defaults** - SDK defaults, lowest priority

### Key Configuration Files

| File | Location | Scope | Shared |
|------|----------|-------|--------|
| `settings.json` | `~/.claude/` | User global | No |
| `settings.json` | `<git-root>/.claude/` | Project | Yes (git) |
| `settings.json` | `<cwd>/.claude/` | Local | No (gitignored) |
| `settings.json` | `/etc/claude/` | Enterprise | Yes (managed) |
| `.env` | Project root | Project | No (gitignored) |

### Configuration Checklist

- [ ] Set `ANTHROPIC_API_KEY` environment variable
- [ ] Choose permission mode (default/acceptEdits/bypassPermissions/plan)
- [ ] Configure MCP servers (if using external tools)
- [ ] Define custom agents (if needed)
- [ ] Set up permission rules (for security)
- [ ] Configure allowed/disallowed tools
- [ ] Set model preferences
- [ ] Configure hooks (if needed)
- [ ] Test configuration in plan mode
- [ ] Document configuration decisions
- [ ] Commit project settings to git (if team)

---

