# Codex CLI - Configuration System (Implementation Details)

> **📚 Official User Guide**: For user-facing configuration instructions, see [Official config.md](../../context/codex/docs/config.md)
>
> **🎯 This Document**: Focuses on internal implementation details, Rust code structures, and configuration system architecture for developers.

---

## Quick Links

- **User Guide**: `/context/codex/docs/config.md` - How to configure Codex
- **This Doc**: Implementation details for developers
- **Related**: [04-llm-integration.md](./04-llm-integration.md) - Provider implementation details

---

## Table of Contents
- [Configuration Hierarchy](#configuration-hierarchy)
- [Config File Format](#config-file-format)
- [Environment Variables](#environment-variables)
- [CLI Flags](#cli-flags)
- [Platform-Specific Paths](#platform-specific-paths)
- [Complete Configuration Reference](#complete-configuration-reference)

---

## Configuration Hierarchy

### Priority Order (Highest to Lowest)

```
1. CLI Flags (--model, --approval-mode, etc.)
       ↓
2. Environment Variables (CODEX_*, OPENAI_API_KEY, etc.)
       ↓
3. Config File (~/.codex/config.{yaml,json})
       ↓
4. Default Values (hardcoded in Rust)
```

**Example**:
```bash
# Default: o4-mini
# Config file: gpt-4
# CLI flag: --model gpt-3.5-turbo
# Result: gpt-3.5-turbo (CLI wins)
```

### Configuration Loading

**Implementation**: `core/src/config.rs`

**Actual Config Structure** (from source):
```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Config {
    /// Optional override of model selection
    pub model: String,

    /// Model used specifically for review sessions
    pub review_model: String,

    pub model_family: ModelFamily,

    /// Size of the context window for the model, in tokens
    pub model_context_window: Option<i64>,

    // ... many more fields (see config.rs:78-200+)
}
```

**Loading Flow** (simplified for clarity):
```rust
// Simplified representation of actual loading logic
pub fn load_config() -> Result<Config> {
    // 1. Load defaults
    let mut config = Config::default();

    // 2. Find and load config.toml file
    if let Some(config_file) = find_config_file()? {
        let toml_config = parse_toml_file(&config_file)?;
        config.apply_toml_config(toml_config)?;
    }

    // 3. Apply environment variables
    config.apply_env_overrides()?;

    // 4. Apply CLI flags (done by ConfigOverrides)

    Ok(config)
}
```

**Note**: Code examples in this doc are simplified for readability. See actual source at `codex-rs/core/src/config.rs` for complete implementation.

---

## Config File Format

### Supported Formats

- **YAML**: `~/.codex/config.yaml` (recommended)
- **JSON**: `~/.codex/config.json`

Both formats are equivalent in functionality.

### Basic Configuration (YAML)

```yaml
# ~/.codex/config.yaml

# Model settings
model: o4-mini
provider: openai

# Approval settings
approvalMode: suggest  # suggest | auto-edit | full-auto
fullAutoErrorMode: ask-user  # ask-user | ignore-and-continue

# UI settings
notify: true  # Desktop notifications

# History settings
history:
  maxSize: 1000
  saveHistory: true
  sensitivePatterns: []
```

### Basic Configuration (JSON)

```json
{
  "model": "o4-mini",
  "provider": "openai",
  "approvalMode": "suggest",
  "fullAutoErrorMode": "ask-user",
  "notify": true,
  "history": {
    "maxSize": 1000,
    "saveHistory": true,
    "sensitivePatterns": []
  }
}
```

### Advanced Configuration

```yaml
# Model and provider settings
model: gpt-4
provider: openai

# Multiple provider configurations
providers:
  openai:
    name: OpenAI
    baseURL: https://api.openai.com/v1
    envKey: OPENAI_API_KEY
  
  azure:
    name: AzureOpenAI
    baseURL: https://YOUR_DEPLOYMENT.openai.azure.com/openai
    envKey: AZURE_OPENAI_API_KEY
  
  ollama:
    name: Ollama
    baseURL: http://localhost:11434/v1
    envKey: OLLAMA_API_KEY  # Optional
  
  gemini:
    name: Gemini
    baseURL: https://generativelanguage.googleapis.com/v1beta/openai
    envKey: GEMINI_API_KEY

# Reasoning settings (for o1/o3/o4 models)
reasoningEffort: medium  # low | medium | high
reasoningSummary: auto   # auto | concise | detailed

# Approval and sandbox settings
approvalMode: auto-edit
fullAutoErrorMode: ask-user
sandboxMode: workspace-write  # read-only | workspace-write | danger-full-access
networkAccess: restricted     # restricted | enabled

# Project documentation
projectDocMaxBytes: 32768  # 32KB
projectDocFallbackFilenames:
  - .codex.md
  - CONTRIBUTING.md

# Custom user instructions
userInstructions: |
  Always use TypeScript.
  Write tests for all public APIs.

# MCP server configuration
mcpServers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
  
  github:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-github"]
    env:
      GITHUB_TOKEN: ${GITHUB_TOKEN}

# History configuration
history:
  maxSize: 1000
  saveHistory: true
  sensitivePatterns:
    - "password"
    - "api_key"
    - "secret"

# UI preferences
notify: true
colorScheme: auto  # auto | light | dark

# Model context window override
modelContextWindow: 128000
modelAutoCompactTokenLimit: 100000
effectiveContextWindowPercent: 90
```

---

## Environment Variables

### API Keys

```bash
# OpenAI (default)
export OPENAI_API_KEY="sk-..."

# Azure OpenAI
export AZURE_OPENAI_API_KEY="..."
export AZURE_OPENAI_API_VERSION="2024-04-01-preview"  # Optional

# Other providers
export GEMINI_API_KEY="..."
export ANTHROPIC_API_KEY="..."
export MISTRAL_API_KEY="..."
export DEEPSEEK_API_KEY="..."
export XAI_API_KEY="..."
export GROQ_API_KEY="..."
export OLLAMA_API_KEY="..."  # Usually not needed
```

### Codex-Specific Variables

```bash
# Disable project documentation
export CODEX_DISABLE_PROJECT_DOC=1

# Change home directory
export CODEX_HOME="$HOME/.config/codex"

# Non-interactive mode
export CODEX_QUIET_MODE=1

# Custom prompts directory
export CODEX_PROMPTS_DIR="$HOME/my-prompts"

# Debug mode
export DEBUG=true

# Managed by package manager (auto-detected)
export CODEX_MANAGED_BY_NPM=1
export CODEX_MANAGED_BY_BUN=1

# Sandbox environment (set by Codex itself)
export CODEX_SANDBOX=seatbelt  # or landlock
export CODEX_SANDBOX_NETWORK_DISABLED=1
```

### Environment Variable Loading

**Implementation**: `core/src/config.rs`

```rust
impl Config {
    pub fn apply_env_overrides(&mut self) -> Result<()> {
        // Model override
        if let Ok(model) = env::var("CODEX_MODEL") {
            self.model = Some(model);
        }
        
        // Provider override
        if let Ok(provider) = env::var("CODEX_PROVIDER") {
            self.provider = Some(provider);
        }
        
        // Disable project docs
        if env::var("CODEX_DISABLE_PROJECT_DOC").is_ok() {
            self.project_doc_max_bytes = 0;
        }
        
        // API keys (loaded per-provider)
        // ...
        
        Ok(())
    }
}
```

---

## CLI Flags

### Common Flags

```bash
# Model selection
codex --model gpt-4 "your prompt"
codex -m o1-preview "your prompt"

# Approval mode
codex --approval-mode full-auto "your prompt"
codex -a auto-edit "your prompt"

# Provider selection
codex --provider azure "your prompt"

# Quiet/non-interactive mode
codex --quiet "your prompt"
codex -q "your prompt"

# Disable project documentation
codex --no-project-doc "your prompt"

# Desktop notifications
codex --notify "your prompt"
codex --no-notify "your prompt"

# JSON output (for CI/CD)
codex --json -q "your prompt"
```

### All Available Flags

**Implementation**: `cli/src/main.rs`

**Actual CLI Structure** (from source at cli/src/main.rs:36):
```rust
/// Codex CLI
///
/// If no subcommand is specified, options will be forwarded to the interactive CLI.
#[derive(Debug, Parser)]
#[clap(
    author,
    version,
    subcommand_negates_reqs = true,
    bin_name = "codex",
    override_usage = "codex [OPTIONS] [PROMPT]\n       codex [OPTIONS] <COMMAND> [ARGS]"
)]
pub struct Cli {
    // Flags are defined throughout the struct
    // See actual file for complete list (~100+ lines)
}
```

**Common Flag Examples** (simplified representation):
```rust
// Model selection
#[clap(short = 'm', long)]
model: Option<String>,

// Sandbox mode
#[clap(long = "sandbox")]
sandbox: Option<SandboxMode>,

// Approval policy
#[clap(short = 'a', long = "ask-for-approval")]
approval_policy: Option<AskForApproval>,

// Other common flags...
// See cli/src/main.rs for complete definition
```

**📝 Note**: The actual CLI structure is more complex. This is a simplified view. For complete flag list, see official docs or run `codex --help`.

### Flag Examples

```bash
# Full auto with GPT-4
codex -a full-auto -m gpt-4 "refactor this module"

# Quiet mode with JSON output (CI/CD)
codex -q --json "lint and fix all files"

# Use Ollama locally
codex --provider ollama --model llama2 "explain this code"

# High reasoning effort for o1
codex --model o1-preview --reasoning-effort high "solve this algorithm problem"

# Disable notifications
codex --no-notify "long running task"
```

---

## Platform-Specific Paths

### Configuration Directory

| Platform | Path |
|----------|------|
| **macOS** | `~/Library/Application Support/codex/` |
| **Linux** | `~/.config/codex/` or `~/.codex/` |
| **Windows** | `%APPDATA%\codex\` |

**Implementation**: `core/src/config.rs`

```rust
pub fn find_codex_home() -> Result<PathBuf> {
    // Check explicit override
    if let Ok(home) = env::var("CODEX_HOME") {
        return Ok(PathBuf::from(home));
    }
    
    // Platform-specific default
    #[cfg(target_os = "macos")]
    {
        let home = env::var("HOME")?;
        Ok(PathBuf::from(home).join("Library/Application Support/codex"))
    }
    
    #[cfg(target_os = "linux")]
    {
        if let Ok(xdg_config) = env::var("XDG_CONFIG_HOME") {
            Ok(PathBuf::from(xdg_config).join("codex"))
        } else {
            let home = env::var("HOME")?;
            Ok(PathBuf::from(home).join(".config/codex"))
        }
    }
    
    #[cfg(target_os = "windows")]
    {
        let appdata = env::var("APPDATA")?;
        Ok(PathBuf::from(appdata).join("codex"))
    }
}
```

### File Locations

```
~/.codex/  (or platform equivalent)
├── config.yaml           # Main configuration
├── config.json           # Alternative format
├── auth.json             # Authentication credentials
├── history/              # Conversation history
│   ├── session_001.json
│   ├── session_002.json
│   └── ...
├── prompts/              # Custom prompts
│   ├── perf.md
│   ├── security.md
│   └── ...
├── cache/                # Temporary cache
│   └── model_info.json
└── logs/                 # Application logs
    ├── codex.log
    └── error.log
```

---

## Complete Configuration Reference

### All Options (Alphabetical)

```yaml
# Approval mode: how much autonomy the agent has
# Options: suggest | auto-edit | full-auto
approvalMode: suggest

# Color scheme for UI
# Options: auto | light | dark
colorScheme: auto

# Context window percentage to use (safety margin)
# Range: 1-100
effectiveContextWindowPercent: 90

# Error handling in full-auto mode
# Options: ask-user | ignore-and-continue
fullAutoErrorMode: ask-user

# History settings
history:
  maxSize: 1000                    # Max history entries
  saveHistory: true                # Whether to save
  sensitivePatterns: []            # Patterns to filter

# MCP servers configuration
mcpServers:
  server_name:
    command: "path/to/server"
    args: ["--flag", "value"]
    env:
      VAR_NAME: "value"

# Model selection
# Examples: gpt-4, o1-preview, o4-mini, claude-3-opus
model: o4-mini

# Auto-compaction threshold (tokens)
modelAutoCompactTokenLimit: 100000

# Override model context window
modelContextWindow: 128000

# Network access mode
# Options: restricted | enabled
networkAccess: restricted

# Desktop notifications
notify: true

# Max bytes to load from AGENTS.md files
projectDocMaxBytes: 32768

# Fallback filenames if AGENTS.md not found
projectDocFallbackFilenames:
  - .codex.md
  - CONTRIBUTING.md

# Provider selection
# Options: openai | azure | gemini | ollama | mistral | deepseek | xai | groq
provider: openai

# Provider configurations
providers:
  provider_name:
    name: "Display Name"
    baseURL: "https://api.example.com/v1"
    envKey: "ENV_VAR_NAME"

# Reasoning effort (o-series models only)
# Options: low | medium | high
reasoningEffort: medium

# Reasoning summary mode (o-series models only)
# Options: auto | concise | detailed
reasoningSummary: auto

# Filesystem sandbox mode
# Options: read-only | workspace-write | danger-full-access
sandboxMode: workspace-write

# Custom user instructions (always included in prompts)
userInstructions: |
  Your custom instructions here.
  Multiple lines supported.

# Additional writable directories (beyond workspace)
writableRoots:
  - /path/to/dir1
  - /path/to/dir2
```

### Type Reference

```typescript
interface Config {
  // Model settings
  model?: string;
  provider?: string;
  providers?: Record<string, ProviderConfig>;
  modelContextWindow?: number;
  modelAutoCompactTokenLimit?: number;
  effectiveContextWindowPercent?: number;
  
  // Reasoning settings
  reasoningEffort?: 'low' | 'medium' | 'high';
  reasoningSummary?: 'auto' | 'concise' | 'detailed';
  
  // Approval settings
  approvalMode?: 'suggest' | 'auto-edit' | 'full-auto';
  fullAutoErrorMode?: 'ask-user' | 'ignore-and-continue';
  
  // Sandbox settings
  sandboxMode?: 'read-only' | 'workspace-write' | 'danger-full-access';
  networkAccess?: 'restricted' | 'enabled';
  writableRoots?: string[];
  
  // Project documentation
  projectDocMaxBytes?: number;
  projectDocFallbackFilenames?: string[];
  userInstructions?: string;
  
  // History
  history?: {
    maxSize?: number;
    saveHistory?: boolean;
    sensitivePatterns?: string[];
  };
  
  // MCP
  mcpServers?: Record<string, McpServerConfig>;
  
  // UI
  notify?: boolean;
  colorScheme?: 'auto' | 'light' | 'dark';
}

interface ProviderConfig {
  name: string;
  baseURL: string;
  envKey: string;
}

interface McpServerConfig {
  command: string;
  args?: string[];
  env?: Record<string, string>;
}
```

---

## Configuration Examples

### Example 1: OpenAI with Auto-Edit

```yaml
model: gpt-4
provider: openai
approvalMode: auto-edit
notify: true
projectDocMaxBytes: 65536
```

### Example 2: Ollama Local Setup

```yaml
model: llama2
provider: ollama
approvalMode: full-auto
fullAutoErrorMode: ignore-and-continue
networkAccess: enabled

providers:
  ollama:
    name: Ollama
    baseURL: http://localhost:11434/v1
    envKey: OLLAMA_API_KEY
```

### Example 3: Azure OpenAI Enterprise

```yaml
model: gpt-4
provider: azure
approvalMode: suggest
sandboxMode: read-only
networkAccess: restricted

providers:
  azure:
    name: AzureOpenAI
    baseURL: https://mycompany.openai.azure.com/openai
    envKey: AZURE_OPENAI_API_KEY

history:
  saveHistory: false  # Compliance requirement
  sensitivePatterns:
    - "password"
    - "api_key"
    - "token"
    - "secret"
```

### Example 4: CI/CD Pipeline

```yaml
model: o4-mini
provider: openai
approvalMode: full-auto
fullAutoErrorMode: ignore-and-continue
sandboxMode: workspace-write
notify: false
history:
  saveHistory: false
projectDocMaxBytes: 16384
```

### Example 5: Development with MCP

```yaml
model: gpt-4
provider: openai
approvalMode: auto-edit

mcpServers:
  filesystem:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-filesystem", "./src"]
  
  database:
    command: npx
    args: ["-y", "@modelcontextprotocol/server-postgres"]
    env:
      DATABASE_URL: postgresql://localhost/mydb
```

---

## Troubleshooting

### Config Not Loading

**Issue**: Changes to config file not reflected

**Solution**:
```bash
# Verify config location
codex --help  # Shows config path

# Check file syntax
cat ~/.codex/config.yaml
# or
cat ~/.config/codex/config.yaml

# Check for YAML syntax errors
yamllint ~/.codex/config.yaml
```

### Environment Variables Not Working

**Issue**: `OPENAI_API_KEY` not recognized

**Solution**:
```bash
# Verify variable is set
echo $OPENAI_API_KEY

# Export in current shell
export OPENAI_API_KEY="sk-..."

# Or add to config file
echo "export OPENAI_API_KEY='sk-...'" >> ~/.zshrc
source ~/.zshrc
```

### Provider Not Found

**Issue**: "Provider 'custom' not found"

**Solution**:
```yaml
# Define custom provider in config
providers:
  custom:
    name: Custom
    baseURL: https://api.custom.com/v1
    envKey: CUSTOM_API_KEY

# Then select it
provider: custom
```

---

## Related Documentation

- [01-overview.md](./01-overview.md) - Quick start configuration
- [04-llm-integration.md](./04-llm-integration.md) - Provider details
- [07-security-sandboxing.md](./07-security-sandboxing.md) - Security settings
- [13-authentication.md](./13-authentication.md) - Auth configuration

