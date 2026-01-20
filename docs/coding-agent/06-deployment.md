# Deployment

This comprehensive guide covers installation, configuration, authentication, and deployment of pi-coding-agent for development, team, and production environments.

## Table of Contents

- [Installation Methods](#installation-methods)
- [Configuration Directories](#configuration-directories)
- [Environment Variables](#environment-variables)
- [Authentication Setup](#authentication-setup)
- [Custom Models Configuration](#custom-models-configuration)
- [Project-Level Settings](#project-level-settings)
- [Platform-Specific Setup](#platform-specific-setup)
- [Extensions and Skills Deployment](#extensions-and-skills-deployment)
- [Production Considerations](#production-considerations)
- [CI/CD Integration](#cicd-integration)
- [Troubleshooting](#troubleshooting)

---

## Installation Methods

### NPM Global Installation (Recommended)

The simplest and most common installation method uses npm:

```bash
npm install -g @mariozechner/pi-coding-agent
```

After installation, verify it works:

```bash
pi --version
pi --help
```

The `pi` command will be available system-wide. This method automatically handles dependencies and updates via `npm update -g @mariozechner/pi-coding-agent`.

**Requirements:**
- Node.js 20.0.0 or higher
- npm (comes with Node.js)

### Standalone Binary Installation

Pre-built binaries are available for all major platforms from [GitHub Releases](https://github.com/badlogic/pi-mono/releases):

| Platform | Archive | Architecture |
|----------|---------|--------------|
| macOS Apple Silicon | `pi-darwin-arm64.tar.gz` | ARM64 (M1/M2/M3) |
| macOS Intel | `pi-darwin-x64.tar.gz` | x86_64 |
| Linux x64 | `pi-linux-x64.tar.gz` | x86_64 |
| Linux ARM64 | `pi-linux-arm64.tar.gz` | ARM64 |
| Windows x64 | `pi-windows-x64.zip` | x86_64 |

**macOS/Linux Installation:**

```bash
# Download and extract
tar -xzf pi-darwin-arm64.tar.gz

# Make executable (if needed)
chmod +x ./pi

# Run directly
./pi

# Or move to PATH
sudo mv ./pi /usr/local/bin/
```

**Windows Installation:**

```powershell
# Extract the zip file
Expand-Archive pi-windows-x64.zip -DestinationPath C:\pi

# Add to PATH or run directly
C:\pi\pi.exe
```

**macOS Binary Signing Note:**

The standalone binary is unsigned. If macOS blocks execution with a security warning, remove the quarantine attribute:

```bash
xattr -c ./pi
```

Alternatively, right-click the file in Finder, select "Open", and confirm in the security dialog.

### Building from Source

Building from source requires [Bun](https://bun.sh) 1.0+ runtime:

```bash
# Clone the monorepo
git clone https://github.com/badlogic/pi-mono.git

# Install dependencies (from root)
cd pi-mono && npm install

# Build all packages
npm run build

# Build the coding agent binary
cd packages/coding-agent
npm run build:binary

# Run the binary
./dist/pi

# Or link for development
npm link
```

**Development Mode:**

For active development without rebuilding:

```bash
cd packages/coding-agent
npm run build  # TypeScript only (no binary)
node dist/cli.js
```

---

## Configuration Directories

Pi uses a hierarchical configuration system with global and project-level settings.

### Global Configuration Directory

**Default Location:** `~/.pi/agent/`

**Override:** Set the `PI_CODING_AGENT_DIR` environment variable:

```bash
export PI_CODING_AGENT_DIR=/custom/path/to/config
```

**Directory Structure:**

```
~/.pi/agent/
├── auth.json            # API keys and OAuth credentials
├── models.json          # Custom model definitions
├── settings.json        # Global settings
├── keybindings.json     # Custom keyboard shortcuts
├── themes/              # Custom themes
│   └── *.json
├── extensions/          # Global extensions
│   └── *.ts
├── skills/              # Global skills
│   └── */SKILL.md
├── prompts/             # Global prompt templates
│   └── *.md
├── tools/               # Custom tools directory
├── bin/                 # Custom binaries
├── sessions/            # Saved sessions organized by project
│   └── <project-path>/
│       └── *.jsonl
├── cache/               # Provider-specific caches
│   └── openai-codex/    # OpenAI Codex prompt cache
├── pi-debug.log         # Debug log file (when PI_TIMING=1)
└── AGENTS.md            # Global context file
```

### Project Configuration Directory

**Location:** `<project-root>/.pi/`

Project-specific configuration overrides global settings:

```
<project>/.pi/
├── settings.json        # Project settings (merged with global)
├── extensions/          # Project-specific extensions
│   └── *.ts
├── skills/              # Project-specific skills
│   └── */SKILL.md
└── prompts/             # Project prompt templates
    └── *.md
```

### Configuration File Purposes

| File | Purpose |
|------|---------|
| `auth.json` | Stores API keys and OAuth tokens (600 permissions) |
| `models.json` | Defines custom models and provider overrides |
| `settings.json` | User preferences, defaults, and behavior settings |
| `keybindings.json` | Custom keyboard shortcut mappings |
| `AGENTS.md` | Context instructions loaded into system prompt |

---

## Environment Variables

### Core Configuration

| Variable | Purpose | Default |
|----------|---------|---------|
| `PI_CODING_AGENT_DIR` | Override config directory | `~/.pi/agent` |
| `PI_SKIP_VERSION_CHECK` | Skip npm version check at startup | Not set |
| `PI_TIMING` | Enable startup timing instrumentation | Not set |
| `PI_HARDWARE_CURSOR` | Show hardware cursor in terminal | Not set |

### API Key Variables

Pi checks these environment variables for provider authentication:

| Provider | Environment Variable | Description |
|----------|---------------------|-------------|
| Anthropic | `ANTHROPIC_API_KEY` | Anthropic Claude API key |
| Anthropic | `ANTHROPIC_OAUTH_TOKEN` | Anthropic OAuth token (alternative) |
| OpenAI | `OPENAI_API_KEY` | OpenAI GPT API key |
| Google | `GEMINI_API_KEY` | Google Gemini API key |
| Mistral | `MISTRAL_API_KEY` | Mistral AI API key |
| Groq | `GROQ_API_KEY` | Groq API key |
| Cerebras | `CEREBRAS_API_KEY` | Cerebras API key |
| xAI | `XAI_API_KEY` | xAI Grok API key |
| OpenRouter | `OPENROUTER_API_KEY` | OpenRouter API key |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` | Vercel AI Gateway API key |
| ZAI | `ZAI_API_KEY` | ZAI API key |
| MiniMax | `MINIMAX_API_KEY` | MiniMax API key |
| MiniMax (China) | `MINIMAX_CN_API_KEY` | MiniMax China API key |
| OpenCode Zen | `OPENCODE_API_KEY` | OpenCode API key |

### Amazon Bedrock Variables

| Variable | Purpose |
|----------|---------|
| `AWS_PROFILE` | AWS profile name from `~/.aws/credentials` |
| `AWS_ACCESS_KEY_ID` | IAM access key |
| `AWS_SECRET_ACCESS_KEY` | IAM secret key |
| `AWS_BEARER_TOKEN_BEDROCK` | Bedrock bearer token authentication |
| `AWS_REGION` | AWS region (default: `us-east-1`) |

### Google Cloud Variables

| Variable | Purpose |
|----------|---------|
| `GOOGLE_CLOUD_PROJECT` | Google Cloud project ID for paid subscriptions |
| `GOOGLE_CLOUD_PROJECT_ID` | Alternative project ID variable |

### Editor Variables

| Variable | Purpose |
|----------|---------|
| `VISUAL` | Primary external editor for Ctrl+G |
| `EDITOR` | Fallback external editor |
| `HOME` | Home directory (used for path display) |
| `USERPROFILE` | Windows home directory |

### Terminal Detection Variables

| Variable | Purpose |
|----------|---------|
| `COLORTERM` | Terminal color capability detection |
| `WT_SESSION` | Windows Terminal session detection |
| `TERM` | Terminal type for color fallback |
| `COLORFGBG` | Terminal background color detection |

---

## Authentication Setup

Pi supports multiple authentication methods with a priority order:
1. Runtime overrides (via SDK `setRuntimeApiKey`)
2. Stored credentials in `auth.json`
3. Environment variables

### auth.json Format

Create `~/.pi/agent/auth.json` with proper permissions:

```bash
touch ~/.pi/agent/auth.json
chmod 600 ~/.pi/agent/auth.json
```

**API Key Authentication:**

```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-api03-..." },
  "openai": { "type": "api_key", "key": "sk-..." },
  "google": { "type": "api_key", "key": "AIza..." },
  "mistral": { "type": "api_key", "key": "..." },
  "groq": { "type": "api_key", "key": "gsk_..." }
}
```

**OAuth Credentials (managed by /login):**

```json
{
  "anthropic": {
    "type": "oauth",
    "access": "access_token_here",
    "refresh": "refresh_token_here",
    "expires": 1735689600000
  }
}
```

OAuth tokens are automatically refreshed when expired. The file uses file locking for concurrent access safety.

### OAuth Provider Setup

Use the `/login` command within pi to authenticate with OAuth providers:

```bash
pi
/login  # Select provider from menu
```

**Anthropic Console (Claude Pro/Max):**

1. Run `/login` and select `anthropic`
2. Browser opens to Anthropic Console
3. Authorize the application
4. Paste the authorization code when prompted
5. Credentials stored for subscription access

**GitHub Copilot:**

1. Run `/login` and select `github-copilot`
2. Press Enter for github.com or enter your GitHub Enterprise Server domain
3. Browser opens to GitHub device authorization
4. Enter the displayed code
5. Credentials stored for Copilot subscription models

**Note:** If you get "model not supported" error, enable it in VS Code: Copilot Chat > model selector > select model > "Enable".

**Google Gemini CLI:**

1. Run `/login` and select `google-gemini-cli`
2. Browser opens to Google OAuth
3. Authorize with your Google account
4. Uses production Cloud Code Assist endpoint (free tier with rate limits)

**Google Antigravity:**

1. Run `/login` and select `google-antigravity`
2. Browser opens to Google OAuth
3. Authorize with your Google account
4. Access to experimental models: Gemini 3, Claude (sonnet/opus thinking), GPT-OSS

**OpenAI Codex (ChatGPT Plus/Pro):**

1. Run `/login` and select `openai-codex`
2. Browser opens to OpenAI OAuth
3. Requires ChatGPT Plus or Pro subscription
4. Prompt cache stored under `~/.pi/agent/cache/openai-codex/`

**Logout:**

```bash
pi
/logout  # Select provider to clear credentials
```

### API Key Priority

When multiple authentication methods exist, pi uses this priority:

1. **Runtime overrides** - Set via SDK (`authStorage.setRuntimeApiKey`)
2. **auth.json credentials** - API keys or OAuth tokens
3. **Environment variables** - Provider-specific env vars

Auth file credentials always override environment variables.

---

## Custom Models Configuration

Add custom models and providers via `~/.pi/agent/models.json`. This file is validated against a JSON schema and automatically reloaded when opening the `/model` selector.

### Full Schema

```json
{
  "providers": {
    "<provider-name>": {
      "baseUrl": "https://api.example.com/v1",
      "apiKey": "API_KEY_OR_ENV_VAR_OR_COMMAND",
      "api": "openai-completions",
      "authHeader": false,
      "headers": {
        "X-Custom-Header": "value"
      },
      "models": [
        {
          "id": "model-id",
          "name": "Display Name",
          "api": "openai-completions",
          "reasoning": false,
          "input": ["text", "image"],
          "cost": {
            "input": 0.0,
            "output": 0.0,
            "cacheRead": 0.0,
            "cacheWrite": 0.0
          },
          "contextWindow": 128000,
          "maxTokens": 32000,
          "headers": {},
          "compat": {}
        }
      ]
    }
  }
}
```

### Supported API Types

| API Type | Description |
|----------|-------------|
| `openai-completions` | OpenAI Chat Completions API |
| `openai-responses` | OpenAI Responses API |
| `openai-codex-responses` | OpenAI Codex Responses API |
| `anthropic-messages` | Anthropic Messages API |
| `google-generative-ai` | Google Generative AI API |
| `bedrock-converse-stream` | AWS Bedrock Converse Stream API |

### API Key Resolution

The `apiKey` field supports three formats:

1. **Literal string**: Use the value directly
2. **Environment variable name**: Reads from `process.env[value]`
3. **Shell command (prefix with `!`)**: Executes command and uses stdout

```json
{
  "providers": {
    "my-provider": {
      "apiKey": "MY_API_KEY_ENV_VAR",
      "baseUrl": "..."
    },
    "vault-provider": {
      "apiKey": "!vault read -field=api_key secret/my-api",
      "baseUrl": "..."
    }
  }
}
```

Shell command results are cached for performance.

### Example: Local Ollama

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "apiKey": "OLLAMA_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "llama-3.1-8b",
          "name": "Llama 3.1 8B (Local)",
          "reasoning": false,
          "input": ["text"],
          "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
          "contextWindow": 128000,
          "maxTokens": 32000
        }
      ]
    }
  }
}
```

### Example: Custom Anthropic Proxy

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1",
      "apiKey": "ANTHROPIC_API_KEY"
    }
  }
}
```

This overrides the built-in Anthropic provider's base URL without replacing models.

### Provider Override vs Replacement

- **Override (no models array):** Modifies built-in provider settings (baseUrl, headers)
- **Replacement (with models array):** Completely replaces the built-in provider

---

## Project-Level Settings

### settings.json Schema

Create `~/.pi/agent/settings.json` for global settings:

```json
{
  "theme": "dark",
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "steeringMode": "one-at-a-time",
  "followUpMode": "one-at-a-time",
  "hideThinkingBlock": false,
  "quietStartup": false,
  "collapseChangelog": false,
  "doubleEscapeAction": "fork",
  "editorPaddingX": 0,
  "showHardwareCursor": false,

  "enabledModels": ["anthropic/*", "*gpt*"],

  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },

  "branchSummary": {
    "reserveTokens": 16384
  },

  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000
  },

  "skills": {
    "enabled": true,
    "enableCodexUser": true,
    "enableClaudeUser": true,
    "enableClaudeProject": true,
    "enablePiUser": true,
    "enablePiProject": true,
    "enableSkillCommands": true,
    "customDirectories": [],
    "ignoredSkills": [],
    "includeSkills": []
  },

  "terminal": {
    "showImages": true
  },

  "images": {
    "autoResize": true,
    "blockImages": false
  },

  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 10240,
    "high": 20480
  },

  "markdown": {
    "codeBlockIndent": ""
  },

  "shellPath": "C:\\Program Files\\Git\\bin\\bash.exe",
  "shellCommandPrefix": "shopt -s expand_aliases",

  "extensions": ["/path/to/global/extension.ts"]
}
```

### Project Settings Merging

Create `<project>/.pi/settings.json` for project-specific overrides. Project settings are deep-merged with global settings:

```json
{
  "defaultModel": "claude-opus-4-5",
  "defaultThinkingLevel": "high",
  "skills": {
    "ignoredSkills": ["deprecated-skill"]
  }
}
```

The merge behavior:
- Scalar values: Project overrides global
- Objects: Deep merge (project values override matching keys)
- Arrays: Project replaces global (no merging)

### Team Sharing

Share project settings by committing `.pi/settings.json`:

```bash
# .gitignore - keep auth private
.pi/auth.json

# Share these files
git add .pi/settings.json
git add .pi/skills/
git add .pi/extensions/
git add .pi/prompts/
```

---

## Platform-Specific Setup

### macOS

**Binary Signing:**

Standalone binaries are unsigned. Remove quarantine:

```bash
xattr -c ./pi
```

Or allow in System Preferences > Security & Privacy after first run attempt.

**Recommended Terminals:**
- iTerm2 (full Kitty protocol support)
- Kitty (native protocol support)
- Ghostty (with configuration)

### Linux

**Permissions:**

Ensure config directory permissions:

```bash
mkdir -p ~/.pi/agent
chmod 700 ~/.pi/agent
chmod 600 ~/.pi/agent/auth.json  # If exists
```

**Dependencies:**

Most distributions include required dependencies. For minimal installs:

```bash
# Debian/Ubuntu
apt-get install bash coreutils

# Alpine
apk add bash coreutils
```

### Windows

**Bash Requirement:**

Pi requires a bash shell on Windows. Detection order:

1. Custom path from `settings.json`
2. Git Bash (`C:\Program Files\Git\bin\bash.exe`)
3. `bash.exe` on PATH (Cygwin, MSYS2, WSL)

**Recommended:** Install [Git for Windows](https://git-scm.com/download/win)

**Custom Shell Path:**

```json
{
  "shellPath": "C:\\cygwin64\\bin\\bash.exe"
}
```

**Alias Expansion:**

Bash runs in non-interactive mode. Enable aliases:

```json
{
  "shellCommandPrefix": "shopt -s expand_aliases\neval \"$(grep '^alias ' ~/.bashrc)\""
}
```

**Windows Terminal Configuration:**

Add to `settings.json` (Ctrl+Shift+,):

```json
{
  "actions": [
    {
      "command": { "action": "sendInput", "input": "\u001b[13;2u" },
      "keys": "shift+enter"
    }
  ]
}
```

### Terminal Configuration

Pi uses the [Kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) for reliable modifier key detection.

**Ghostty** - Add to `~/.config/ghostty/config`:
```
keybind = alt+backspace=text:\x1b\x7f
keybind = shift+enter=text:\n
```

**WezTerm** - Create `~/.wezterm.lua`:
```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.enable_kitty_keyboard = true
return config
```

**VS Code Terminal** - Add to `keybindings.json`:
```json
{
  "key": "shift+enter",
  "command": "workbench.action.terminal.sendSequence",
  "args": { "text": "\u001b[13;2u" },
  "when": "terminalFocus"
}
```

**IntelliJ IDEA:** The built-in terminal has limited escape sequence support. Consider using a dedicated terminal emulator.

---

## Extensions and Skills Deployment

### Extension Locations

Extensions are TypeScript files loaded at startup:

| Location | Scope | Priority |
|----------|-------|----------|
| `~/.pi/agent/extensions/*.ts` | Global | Loaded first |
| `<project>/.pi/extensions/*.ts` | Project | Loaded after global |
| Paths in `settings.json` `extensions` | Custom | Loaded as specified |

**Extension Requirements:**
- Must be `.ts` files
- Compiled at runtime using jiti
- Access to `pi` API object for hooks and commands

### Skill Locations

Skills are markdown files with YAML frontmatter:

| Location | Source Name | Format |
|----------|-------------|--------|
| `~/.codex/skills/*/SKILL.md` | codex-user | Recursive |
| `~/.claude/skills/*.md` | claude-user | Claude format |
| `<project>/.claude/skills/*.md` | claude-project | Claude format |
| `~/.pi/agent/skills/*/SKILL.md` | user | Recursive |
| `<project>/.pi/skills/*/SKILL.md` | project | Recursive |
| Custom directories in settings | custom | Recursive |

**Skill Format:**

```markdown
---
description: Short description for the skill
---

# Skill Name

Detailed instructions for the skill...
```

**Controlling Skill Sources:**

```json
{
  "skills": {
    "enableCodexUser": true,
    "enableClaudeUser": true,
    "enableClaudeProject": true,
    "enablePiUser": true,
    "enablePiProject": true,
    "customDirectories": ["/shared/skills"],
    "ignoredSkills": ["deprecated-skill"],
    "includeSkills": ["only-these-skills"]
  }
}
```

---

## Production Considerations

### Security Best Practices

1. **File Permissions:**
   ```bash
   chmod 700 ~/.pi/agent
   chmod 600 ~/.pi/agent/auth.json
   chmod 600 ~/.pi/agent/models.json
   ```

2. **API Key Management:**
   - Never commit `auth.json` to version control
   - Use environment variables in CI/CD
   - Consider secrets managers for production:
     ```json
     {
       "providers": {
         "my-provider": {
           "apiKey": "!vault read -field=key secret/api"
         }
       }
     }
     ```

3. **Network Security:**
   - Use HTTPS for all API endpoints
   - Configure proxy settings if required
   - Consider VPN for sensitive operations

### Session Storage

Sessions are stored as JSONL files in `~/.pi/agent/sessions/<project-path>/`:

- Each session creates a new timestamped file
- Sessions include full conversation history
- Consider periodic cleanup for disk space:
  ```bash
  find ~/.pi/agent/sessions -name "*.jsonl" -mtime +30 -delete
  ```

### Performance Tuning

**Compaction Settings:**

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

**Retry Settings:**

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000
  }
}
```

---

## CI/CD Integration

### Print Mode

For non-interactive scripts, use print mode:

```bash
pi -p "Generate a changelog from recent commits" --provider anthropic --model claude-sonnet-4-20250514
```

**Flags:**
- `-p` or `--print`: Print mode (non-interactive)
- `--no-session`: Don't persist session
- `--tools`: Limit available tools

**Example CI Script:**

```bash
#!/bin/bash
export ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY}"

pi -p "Review the changes in this PR and suggest improvements" \
   --provider anthropic \
   --model claude-sonnet-4-20250514 \
   --no-session \
   --tools read,grep,find,ls
```

### RPC Mode

For programmatic control, use RPC mode:

```bash
pi --mode rpc --no-session
```

RPC mode uses JSON protocol over stdin/stdout:

**Send command:**
```json
{"type": "prompt", "message": "Hello, world!"}
```

**Receive events:**
```json
{"type": "message_update", "assistantMessageEvent": {"type": "text_delta", "delta": "Hello!"}}
```

**Common RPC Commands:**
- `prompt` - Send a message to the agent
- `get_state` - Get current session state
- `set_model` - Switch models
- `abort` - Cancel current operation
- `compact` - Manually compact context
- `bash` - Execute shell command

See `docs/rpc.md` for full protocol documentation.

### SDK Integration

For Node.js applications, use the SDK directly:

```typescript
import { createAgentSession, discoverAuthStorage, discoverModels, SessionManager } from "@mariozechner/pi-coding-agent";

const authStorage = discoverAuthStorage();
const modelRegistry = discoverModels(authStorage);

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage,
  modelRegistry,
});

session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("Analyze this codebase");
```

See `docs/sdk.md` and `examples/sdk/` for detailed examples.

---

## Troubleshooting

### Installation Issues

**npm install fails:**
```bash
# Clear npm cache
npm cache clean --force

# Try with verbose logging
npm install -g @mariozechner/pi-coding-agent --verbose

# Check Node.js version
node --version  # Must be >= 20.0.0
```

**Binary not found after npm install:**
```bash
# Check npm global bin directory
npm config get prefix

# Add to PATH
export PATH="$(npm config get prefix)/bin:$PATH"
```

### Permission Problems

**auth.json permission denied:**
```bash
# Fix permissions
chmod 600 ~/.pi/agent/auth.json
chown $(whoami) ~/.pi/agent/auth.json
```

**Cannot write to config directory:**
```bash
# Create with proper ownership
mkdir -p ~/.pi/agent
chmod 700 ~/.pi/agent
```

### API Key Issues

**"No models available" error:**
1. Check API key is set:
   ```bash
   echo $ANTHROPIC_API_KEY
   ```
2. Verify auth.json format:
   ```bash
   cat ~/.pi/agent/auth.json | jq .
   ```
3. Test API key directly:
   ```bash
   curl -H "x-api-key: $ANTHROPIC_API_KEY" \
        -H "anthropic-version: 2023-06-01" \
        https://api.anthropic.com/v1/models
   ```

**OAuth token expired:**
```bash
pi
/logout anthropic
/login  # Re-authenticate
```

**OAuth port conflict (Port 1455 in use):**
- Close the conflicting process
- Or paste the auth code/URL when prompted

### Model Configuration Issues

**models.json validation error:**
```bash
# Check JSON syntax
cat ~/.pi/agent/models.json | jq .

# Common issues:
# - Missing required fields (baseUrl, apiKey with models)
# - Invalid API type
# - Missing model cost fields
```

### Terminal Issues

**Keyboard shortcuts not working:**
- Ensure terminal supports Kitty keyboard protocol
- Configure terminal-specific settings (see Platform-Specific Setup)
- Try `PI_HARDWARE_CURSOR=1 pi` for cursor visibility

**Colors not displaying correctly:**
```bash
# Check terminal color support
echo $TERM
echo $COLORTERM

# Force truecolor
export COLORTERM=truecolor
```

### Debug Mode

Enable timing instrumentation:
```bash
PI_TIMING=1 pi
```

Check debug log:
```bash
cat ~/.pi/agent/pi-debug.log
```

---

---

## Deployment Checklist

Use this comprehensive checklist to ensure a successful production deployment of pi-coding-agent.

### Pre-Deployment Checklist

#### Authentication and API Keys

- [ ] API keys obtained for all required providers
- [ ] API keys stored securely (auth.json with 600 permissions or environment variables)
- [ ] OAuth authentication tested for subscription services (Claude Pro, GitHub Copilot)
- [ ] API key rotation policy established for production environments
- [ ] Secrets manager integration configured (if using Vault, AWS Secrets Manager, etc.)

#### Custom Models Configuration

- [ ] Custom models defined in `models.json` (if required)
- [ ] Custom model API endpoints tested and accessible
- [ ] Model authentication working (API keys resolved correctly)
- [ ] Cost tracking configured for custom models
- [ ] Model fallback strategy defined

#### Extensions and Customization

- [ ] Required extensions installed in `~/.pi/agent/extensions/` or `.pi/extensions/`
- [ ] Extension dependencies installed (if extensions have `package.json`)
- [ ] Extension error handling tested
- [ ] Custom tools registered and tested
- [ ] Extension performance impact measured

#### Settings and Configuration

- [ ] Global settings configured in `~/.pi/agent/settings.json`
- [ ] Project-level settings configured in `<project>/.pi/settings.json`
- [ ] Default model and provider set
- [ ] Thinking level configured appropriately
- [ ] Compaction settings tuned for workload
- [ ] Retry settings configured
- [ ] Shell path configured (especially for Windows)

#### Security Review

- [ ] File permissions verified (700 for directories, 600 for sensitive files)
- [ ] No sensitive credentials in version control
- [ ] `.gitignore` includes `.pi/auth.json` and sensitive files
- [ ] Network access validated (HTTPS endpoints, proxy if needed)
- [ ] Extension code reviewed for security issues
- [ ] Session storage location reviewed (avoid shared filesystems)

#### Platform-Specific Setup

##### macOS
- [ ] Binary quarantine removed (`xattr -c ./pi`)
- [ ] Terminal supports Kitty keyboard protocol (iTerm2, Kitty, or configured)
- [ ] Homebrew dependencies installed (if using npm installation)

##### Linux
- [ ] Config directory permissions set (`chmod 700 ~/.pi/agent`)
- [ ] Required system packages installed (bash, coreutils)
- [ ] Binary executable (`chmod +x ./pi`)

##### Windows
- [ ] Git for Windows installed (for bash shell)
- [ ] Shell path configured in settings.json
- [ ] Alias expansion enabled in settings
- [ ] Windows Terminal keyboard shortcuts configured

#### Testing and Validation

- [ ] Installation verified (`pi --version`)
- [ ] Basic prompt tested (`pi -p "Hello, world!"`)
- [ ] Tool execution tested (read, bash, edit, write)
- [ ] Model switching tested
- [ ] Session persistence tested
- [ ] Extension loading verified
- [ ] Print mode tested for CI/CD use cases
- [ ] RPC mode tested (if using programmatic integration)

### Post-Deployment Checklist

#### Monitoring and Maintenance

- [ ] Session directory size monitoring configured
- [ ] Debug logs location identified (`~/.pi/agent/pi-coding-agent-debug.log`)
- [ ] Session cleanup strategy implemented (delete old sessions)
- [ ] Cost tracking implemented (monitor `SessionStats.cost`)
- [ ] Performance metrics collected (token usage, response times)

#### Documentation and Training

- [ ] Team onboarded with pi-coding-agent basics
- [ ] Project-specific extensions documented
- [ ] Custom tools and commands documented
- [ ] Settings explained to team members
- [ ] Troubleshooting guide shared

#### Backup and Recovery

- [ ] Auth credentials backed up securely
- [ ] Custom models.json backed up
- [ ] Important sessions backed up (if needed)
- [ ] Extension code in version control
- [ ] Recovery procedure documented

---

## Migration Guide

### Migrating from a Different Pi Version

If you're upgrading from an older version of pi-coding-agent, follow this guide to ensure a smooth transition.

#### Breaking Changes Between Versions

**From v1.x to v2.x:**

1. **Session Format Change** - Sessions now use JSONL v3 format
   ```bash
   # Sessions are automatically migrated on first load
   # Old sessions remain compatible
   ```

2. **Auth Storage Changes** - OAuth format updated
   ```bash
   # Re-authenticate OAuth providers
   pi
   /logout anthropic
   /login anthropic
   ```

3. **Settings Schema Updates**
   ```bash
   # Backup existing settings
   cp ~/.pi/agent/settings.json ~/.pi/agent/settings.json.backup

   # Update settings manually or let pi regenerate defaults
   rm ~/.pi/agent/settings.json
   pi  # Will create new settings.json
   ```

#### Step-by-Step Migration Process

**Step 1: Backup Current Configuration**

```bash
# Backup entire config directory
cp -r ~/.pi/agent ~/.pi/agent.backup

# Or backup specific files
cp ~/.pi/agent/auth.json ~/.pi/agent/auth.json.backup
cp ~/.pi/agent/models.json ~/.pi/agent/models.json.backup
cp ~/.pi/agent/settings.json ~/.pi/agent/settings.json.backup
```

**Step 2: Install New Version**

```bash
# For npm installation
npm install -g @mariozechner/pi-coding-agent@latest

# For standalone binary
# Download latest from GitHub releases and replace existing binary
wget https://github.com/badlogic/pi-mono/releases/latest/download/pi-<platform>.tar.gz
tar -xzf pi-<platform>.tar.gz
sudo mv pi /usr/local/bin/pi
```

**Step 3: Verify Installation**

```bash
pi --version
# Should show the new version

pi --help
# Check for new flags and options
```

**Step 4: Migrate Authentication**

```bash
# Start pi and check available models
pi
/models

# If OAuth providers fail, re-authenticate
/logout <provider>
/login <provider>
```

**Step 5: Update Settings**

Review and update your settings.json:

```bash
# Check for deprecated settings
pi
# Look for warnings about unknown settings

# Update settings as needed
# New settings can be added via /settings or by editing the file
```

**Step 6: Test Extensions and Skills**

```bash
# Verify extensions load correctly
pi
# Check startup messages for extension errors

# If extensions fail, update them for new API
# See extension migration guide in examples/
```

**Step 7: Test Sessions**

```bash
# Open an existing session
pi
/sessions  # Select an old session

# Verify it loads correctly
# Sessions are auto-migrated to new format
```

### Migrating from a Different Coding Assistant

If you're switching from another AI coding assistant (e.g., Cursor, Aider, GitHub Copilot CLI, etc.), this guide helps you transition.

#### From Cursor

**Key Differences:**
- Pi runs in the terminal, not in a VS Code fork
- Pi uses explicit tools (read, edit, bash) instead of file watching
- Pi supports multiple models and providers

**Migration Steps:**

1. **Export Cursor Settings:**
   - Note your preferred model settings
   - Export any custom rules or instructions

2. **Create Pi Configuration:**
   ```json
   {
     "defaultProvider": "anthropic",
     "defaultModel": "claude-sonnet-4-20250514",
     "defaultThinkingLevel": "medium"
   }
   ```

3. **Convert Cursor Rules to Pi Context:**
   ```bash
   # Create global context file
   cat > ~/.pi/agent/AGENTS.md << 'EOF'
   # Coding Guidelines

   [Your Cursor rules here]
   EOF
   ```

4. **Learn Pi Commands:**
   - Cursor: Ctrl+K → Pi: Just type and press Enter
   - Cursor: Cmd+L → Pi: `/new` for new session
   - Cursor: Ask to edit → Pi: Agent uses `edit` tool automatically

#### From Aider

**Key Differences:**
- Pi has a richer UI with themes and interactive features
- Pi supports session branching and tree navigation
- Pi has built-in compaction for long conversations

**Migration Steps:**

1. **Import Aider Settings:**
   ```bash
   # If you have a .aider.conf.yml
   # Manually translate to Pi settings.json
   ```

2. **Convert Model Configuration:**
   ```json
   {
     "defaultModel": "gpt-4",  // Aider default
     "defaultProvider": "openai"
   }
   ```

3. **Set Up Repository Context:**
   ```bash
   # Aider uses .aider.chat.history.md
   # Pi uses session files in ~/.pi/agent/sessions/
   # Context files can be created in .pi/ directory
   ```

4. **Adjust Workflow:**
   - Aider: `aider file1.py file2.py` → Pi: Agent uses read tool as needed
   - Aider: `/add file.py` → Pi: Just mention the file, agent will read it
   - Aider: `/commit` → Pi: Agent can run git commands via bash tool

#### From GitHub Copilot CLI

**Key Differences:**
- Pi is conversational, not single-command focused
- Pi maintains conversation history across sessions
- Pi supports multiple providers, not just OpenAI

**Migration Steps:**

1. **Authenticate with GitHub Copilot Provider:**
   ```bash
   pi
   /login github-copilot
   ```

2. **Adjust Command Style:**
   - Copilot CLI: `gh copilot suggest "list files"` → Pi: `pi -p "list files"`
   - Copilot CLI: `gh copilot explain "git command"` → Pi: `pi -p "explain: git command"`

3. **Use Conversational Style:**
   ```bash
   # Instead of one-off commands, have conversations
   pi
   # "Help me set up a new feature"
   # "Add error handling to the login function"
   # "Write tests for this"
   ```

#### General Migration Tips

1. **Start with Print Mode:**
   ```bash
   # Get familiar with Pi's output style
   pi -p "Explain how pi works"
   ```

2. **Explore Interactive Mode:**
   ```bash
   # Try the full TUI experience
   pi
   # Use /help to see all commands
   ```

3. **Set Up Project Context:**
   ```bash
   mkdir -p .pi
   cat > .pi/AGENTS.md << 'EOF'
   # Project-Specific Instructions

   This project uses [framework/language].
   Follow these conventions:
   - [convention 1]
   - [convention 2]
   EOF
   ```

4. **Create Custom Extensions:**
   ```bash
   # Add project-specific commands
   cat > .pi/extensions/project-tools.ts << 'EOF'
   export default function (pi) {
     pi.registerCommand("test", {
       description: "Run project tests",
       handler: async (args, ctx) => {
         // Custom test logic
       },
     });
   }
   EOF
   ```

5. **Practice Common Workflows:**
   - Code review: `pi -p "Review the changes in this PR"`
   - Bug fixing: `pi` then describe the bug conversationally
   - Refactoring: `pi` then explain what you want to refactor
   - Documentation: `pi -p "Generate docs for this module"`

---

## Additional Resources

- **SDK Documentation:** `docs/sdk.md`
- **RPC Protocol:** `docs/rpc.md`
- **Extension Development:** See existing extensions in `examples/`
- **GitHub Issues:** [github.com/badlogic/pi-mono/issues](https://github.com/badlogic/pi-mono/issues)
- **Discord Community:** [discord.com/invite/nKXTsAcmbT](https://discord.com/invite/nKXTsAcmbT)
