# Claude Agent SDK - Configuration Complete Reference

**SDK Version**: 0.1.22  
**Claude Code Runtime**: v2.1.42  
**Primary Sources**:
- SDK types: `sdkTypes.d.ts`
- Claude Code runtime (v2.1.42): `@anthropic-ai/claude-code/cli.js` (distributed bundle)

Note: The Agent SDK ultimately runs a Claude Code executable. Settings, CLI flags, and runtime behavior depend on the Claude Code version you run (see `comprehensive-guide.md` for `pathToClaudeCodeExecutable`).

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

Claude Code (and therefore the Agent SDK) uses a layered configuration system. Settings come from several sources and are merged with precedence; higher-precedence sources override lower-precedence ones for most keys.

```
Enterprise managed settings (policy; can constrain or override)
     ↓
CLI flags and SDK options (per run/session)
     ↓
Local settings (.claude/settings.local.json)
     ↓
Project settings (.claude/settings.json)
     ↓
User settings (~/.claude/settings.json)
     ↓
Defaults (lowest priority)
```

Important MCP note: project-shared MCP servers are configured via `.mcp.json` (not `.claude/settings.json`) and require per-user approval. Dynamic MCP servers can be loaded via `--mcp-config`. See `extraction/mcp-integration-complete.md` for details.

---

## Settings File Locations

### Directory Structure

```
~/.claude/                          # User-global Claude Code data
├── settings.json                   # User settings (shared across projects)
├── CLAUDE.md                       # User memory/rules
├── agents/                         # Personal custom agents
├── skills/                         # Personal skills
├── plugins/                        # Plugin cache + metadata
└── projects/<project-id>/          # Per-project state (transcripts, auto-memory, etc.)

<repo-root>/                        # Your project / repo
├── .claude/
│   ├── settings.json               # Project settings (committed)
│   ├── settings.local.json         # Local overrides (gitignored; per-user)
│   ├── agents/                     # Project custom agents
│   └── skills/                     # Project skills
└── .mcp.json                       # Project-shared MCP servers (approval-gated)

<managed>                           # Enterprise-managed (location varies)
├── managed-settings.json           # Managed settings (policy)
└── managed-mcp.json                # Managed MCP servers (enterprise MCP config)
```

### File Locations by Precedence

| Level | Location | Scope | Persisted | Shared |
|---|---|---|---|---|
| **Managed** | `managed-settings.json` | Organization policy | Yes | Yes (managed) |
| **CLI Flags / SDK options** | Command line / SDK call | Current run/session | No | No |
| **Local** | `.claude/settings.local.json` | Current working directory | Yes | No (gitignored) |
| **Project** | `.claude/settings.json` | Repo/project | Yes | Yes (committed) |
| **User** | `~/.claude/settings.json` | User global | Yes | No |
| **Default** | Claude Code defaults | Built-in | No | N/A |

Notes:
- Windows managed settings may exist at `C:\\Program Files\\ClaudeCode\\managed-settings.json` (legacy: `C:\\ProgramData\\ClaudeCode\\managed-settings.json`).
- MCP server configuration is split: user/local MCP servers can live under `mcpServers` in `settings.json` files, but project-shared MCP servers live in `.mcp.json` and are approval-gated.

---

## Settings Resolution Order

### Merge model (conceptual)

Claude Code loads settings from enabled sources and merges them. For most scalar keys, the later (higher-precedence) value overrides earlier ones; some keys (notably permissions and hooks) have additional managed-policy behavior.

Conceptually:

```ts
effective = merge(
  defaults,
  userSettings,            // ~/.claude/settings.json
  projectSettings,         // .claude/settings.json
  localSettings,           // .claude/settings.local.json
  flagSettings,            // CLI flags (and --settings file/json)
  policySettings           // managed-settings.json (enterprise policy)
)
```

Notes:
- `--setting-sources user,project,local` can restrict which filesystem sources are loaded. Managed settings and CLI flags still apply.
- Managed settings can *constrain* other sources (e.g., “only managed hooks” or “only managed permission rules”), even if those other sources would otherwise have higher precedence.

---

## Settings Schema

### settings.json schema and validation

Claude Code validates `settings.json` against a JSON Schema (the runtime references `https://json.schemastore.org/claude-code-settings.json`). The schema is large and evolves with the runtime; treat the schema (and the v2.1.42 docs in this folder) as the authoritative source of truth.

At a high level, v2.1.42 settings commonly include:

```ts
type SettingsJson = {
  $schema?: string;

  // Core behavior
  model?: string;
  fallbackModel?: string;
  availableModels?: string[];
  agent?: string;                 // selects an agent (built-in or custom)

  // Permissions + hooks (detailed shapes documented elsewhere)
  permissions?: unknown;          // see hooks-permissions-complete.md
  hooks?: unknown;                // see hooks-permissions-complete.md

  // Plugins
  enabledPlugins?: Record<string, boolean | string[] | undefined>;
  extraKnownMarketplaces?: Record<string, unknown>;

  // Retention and memory controls
  cleanupPeriodDays?: number;
  autoMemoryEnabled?: boolean;

  // MCP approval + enterprise policy controls (used with .mcp.json)
  enableAllProjectMcpServers?: boolean;
  enabledMcpjsonServers?: string[];
  disabledMcpjsonServers?: string[];
  allowedMcpServers?: unknown;
  deniedMcpServers?: unknown;
} & Record<string, unknown>;
```

Notes:
- Custom agents are defined on disk (project: `.claude/agents/`, personal: `~/.claude/agents/`). The `agent` setting selects which agent to use by default.
- Project-shared MCP servers are configured in `.mcp.json`, not in `.claude/settings.json` (see `extraction/mcp-integration-complete.md`).
- Auth-related settings exist (for enterprise or advanced setups), such as `forceLoginMethod`, `forceLoginOrgUUID`, and helpers like `apiKeyHelper`. Use `claude auth status` to confirm what authentication source is active.
- Plugins can declare LSP server configurations via `lspServers` in their manifest (including `.lsp.json` files). See `../lsp.md`.

### settings.json Example (Complete)

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "sonnet",
  "fallbackModel": "sonnet",
  "agent": "default",
  "cleanupPeriodDays": 30
}
```

---

## Environment Variables

Claude Code uses environment variables for authentication, provider selection, operational limits, and feature flags. For the vetted list and defaults (v2.1.42), see `extraction/cli-internal-constants.md`.

### Common environment variables (v2.1.42)

```bash
# Authentication
ANTHROPIC_API_KEY="sk-ant-..."           # API-key auth (Console / API-key workflows)
CLAUDE_CODE_OAUTH_TOKEN="..."            # OAuth token (when applicable)

# Model selection (optional)
ANTHROPIC_MODEL="sonnet"

# Debug/logging + UX
DEBUG="claude:*"
NO_COLOR=1

# MCP timeouts and MCP CLI enablement
MCP_TIMEOUT=30000
MCP_TOOL_TIMEOUT=100000000
ENABLE_EXPERIMENTAL_MCP_CLI=1

# Tool / output limits
MAX_MCP_OUTPUT_TOKENS=25000
BASH_DEFAULT_TIMEOUT_MS=120000
# BASH_MAX_TIMEOUT_MS defaults to max(600000, BASH_DEFAULT_TIMEOUT_MS) when unset
BASH_MAX_OUTPUT_LENGTH=30000
```

### Environment Variable Usage

**Setting Environment Variables**:

```bash
# One-time (current command)
ANTHROPIC_API_KEY="sk-ant-..." claude

# Session (current shell)
export ANTHROPIC_API_KEY="sk-ant-..."
export DEBUG="claude:*"
claude

# Permanent (add to ~/.bashrc, ~/.zshrc, etc.)
echo 'export ANTHROPIC_API_KEY="sk-ant-..."' >> ~/.zshrc
source ~/.zshrc
```

Claude Code does not automatically load `.env` files; set environment variables in your shell, CI environment, or a wrapper script.

---

## CLI Flags

Claude Code’s CLI is exposed as `claude` (v2.1.42). Run `claude --help` for the full list; below are the configuration-related flags and subcommands you most commonly need.

### Common configuration flags (v2.1.42)

```bash
# Settings sources
claude --settings <file-or-json>                 # load extra settings from a JSON file or JSON string
claude --setting-sources user,project,local      # restrict which filesystem sources load

# Model selection
claude --model sonnet
claude --fallback-model sonnet

# Permissions / tools
claude --permission-mode <mode>                  # see hooks-permissions-complete.md
claude --allowed-tools <tools...>
claude --disallowed-tools <tools...>
claude --tools <tools...>                        # built-in tool set selection

# Directory access (adds to permission scope)
claude --add-dir <directories...>

# MCP dynamic config
claude --mcp-config <configs...>                 # JSON strings or JSON file paths (each must contain { "mcpServers": { ... } })
claude --strict-mcp-config                       # only use MCP servers from --mcp-config

# Session management
claude --continue
claude --resume [session-id-or-search]
claude --fork-session                            # used with --resume/--continue to fork to a new session id
```

### Configuration subcommands (v2.1.42)

```bash
# Auth
claude auth login
claude auth status
claude auth logout
claude setup-token

# MCP server management (persisted config)
claude mcp list
claude mcp add <name> <commandOrUrl> [args...]

# Plugins
claude plugin list
claude plugin marketplace list
```

---

## Programmatic Options

### SDK Options Interface

The SDK `query({ prompt, options })` forwards most configuration through to the Claude Code executable it runs. The authoritative typing surface is `sdkTypes.d.ts`.

Commonly used `options` keys include:
- `model`, `fallbackModel`, `maxThinkingTokens`, `maxTurns`
- `permissionMode`, `permissionPromptToolName`
- `allowedTools`, `disallowedTools`, `tools`
- `additionalDirectories`
- `hooks`
- `mcpServers`, `strictMcpConfig`
- `resume`, `forkSession`, `resumeSessionAt`, `includePartialMessages`
- `betas`
- `pathToClaudeCodeExecutable`
- `appendSystemPrompt`
- `executable`, `executableArgs`
- `stderr` (callback)

Notes:
- The SDK types can lag behind the runtime CLI flags; treat the Claude Code v2.1.42 docs in this folder as the authoritative runtime surface.
- Custom agents are primarily configured via `.claude/agents/` (project) and `~/.claude/agents/` (personal), or via CLI flags (runtime). They are not represented as a structured `options.agents` object in `sdkTypes.d.ts`.

### Programmatic Configuration Example: MCP Integration

```typescript
const result = await query({
  prompt: "List files and fetch web content",
  options: {
    mcpServers: {
      'filesystem': {
        type: 'stdio',
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-filesystem', '/workspace']
      },
      'web-fetch': {
        type: 'http',
        url: 'https://example.com/mcp'
      }
    },
    strictMcpConfig: true
  }
});
```

---

## Configuration Examples

### Example 1: Team Project Configuration

**File**: `.claude/settings.json` (repo root, committed)

Keep this file small and team-safe. Prefer documenting sensitive values as environment variables and using policy/permissions docs for higher-risk settings.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "sonnet",
  "cleanupPeriodDays": 30,
  "enabledPlugins": {
    "formatter@anthropic-tools": true
  }
}
```

### Example 2: Local Overrides (per-user, gitignored)

**File**: `.claude/settings.local.json` (repo root, gitignored)

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "cleanupPeriodDays": 0,
  "autoMemoryEnabled": false
}
```

### Example 3: User Defaults

**File**: `~/.claude/settings.json`

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "sonnet",
  "cleanupPeriodDays": 30
}
```

### Example 4: Project-shared MCP servers

**File**: `.mcp.json`

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    }
  }
}
```

Project-shared `.mcp.json` servers require per-user approval before they are used; approvals are tracked in local settings (see `extraction/mcp-integration-complete.md`).

---

## Gotchas & Best Practices

### Gotchas

1. **Settings Not Hot-Reloaded**:
   - Many settings are evaluated at startup; restart Claude Code (or re-run the SDK call) after edits to be safe.
   - CLI flags and SDK options apply only to the current run/session.

2. **Precedence Can Be Confusing**:
   - For filesystem settings, the typical override order is: user → project → local → CLI flags → managed policy.
   - `--setting-sources user,project,local` can hide a layer entirely (by not loading it).

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
   - `${VAR}` and `${VAR:-default}` are supported where MCP config expands variables.
   - Missing variables typically produce warnings during config load (not hard failures).

4. **Path Separators Platform-Specific**:
   ```json
   // ❌ Windows backslashes in JSON
   "path": "C:\\Users\\me\\project"
   
   // ✅ Forward slashes work everywhere
   "path": "C:/Users/me/project"
   ```

5. **MCP configuration is split across files**:
   - User/local MCP servers can be configured via `mcpServers` in `~/.claude/settings.json` and `.claude/settings.local.json`.
   - Project-shared MCP servers live in `.mcp.json` and are approval-gated per user/per project.
   - Dynamic MCP servers come from `--mcp-config` (optionally with `--strict-mcp-config`).

### Best Practices

**1. Use the right file for the scope**:
```bash
# Team-shared
git add .claude/settings.json

# Personal overrides (keep gitignored)
git add .claude/settings.local.json
```

**2. Prefer built-in management commands when available**:
- MCP servers: `claude mcp add/list/get/remove`
- Plugins: `claude plugin ...` / `/plugin`

**3. Use environment variables for secrets**:
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

**4. Validate changes safely**:
```bash
claude --permission-mode plan
claude mcp list
```

---

## Summary

### Configuration sources (conceptual precedence)

1. **Managed settings** (`managed-settings.json`) — enterprise policy (can constrain or override)
2. **CLI flags / SDK options** — per-run configuration
3. **Local settings** (`.claude/settings.local.json`)
4. **Project settings** (`.claude/settings.json`)
5. **User settings** (`~/.claude/settings.json`)
6. **Defaults**

### Key Configuration Files

| File / Dir | Typical location | Purpose |
|---|---|---|
| `settings.json` | `~/.claude/settings.json` | User defaults across projects |
| `settings.json` | `.claude/settings.json` | Team-shared project settings |
| `settings.local.json` | `.claude/settings.local.json` | Per-user project overrides |
| `.mcp.json` | `.mcp.json` | Project-shared MCP servers (approval-gated) |
| `agents/` | `.claude/agents/`, `~/.claude/agents/` | Custom agent definitions |
| `skills/` | `.claude/skills/`, `~/.claude/skills/` | Skills and slash commands |
| `managed-settings.json` | platform-specific | Enterprise managed policy settings |

### Configuration Checklist

- [ ] Authenticate (either `claude auth login` or `ANTHROPIC_API_KEY` / `CLAUDE_CODE_OAUTH_TOKEN` depending on your workflow)
- [ ] Put settings in the right scope (`~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`)
- [ ] Configure MCP servers if needed (`.mcp.json`, `claude mcp add`, `--mcp-config`)
- [ ] Configure permissions and hooks (see `hooks-permissions-complete.md`)
- [ ] Configure plugins/marketplaces if needed (see `plugins.md`)
- [ ] Test configuration in plan mode
- [ ] Document configuration decisions
- [ ] Commit project settings to git (if team)

---

