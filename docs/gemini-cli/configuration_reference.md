# Gemini CLI - Configuration Reference

**Generated:** 2024-10-24
**Purpose:** Complete documentation of all configuration schemas, settings, and environment variables

---

## Table of Contents

1. [Configuration System Overview](#configuration-system-overview)
2. [Configuration File Locations](#configuration-file-locations)
3. [Complete Settings Schema](#complete-settings-schema)
4. [Environment Variables](#environment-variables)
5. [Configuration Layers & Merging](#configuration-layers--merging)
6. [Keyboard Shortcuts](#keyboard-shortcuts)
7. [Extension Configuration](#extension-configuration)
8. [Hidden/Undocumented Settings](#hiddenundocumented-settings)

---

## Configuration System Overview

### Architecture

Gemini CLI uses a **layered configuration system** with multiple sources:

```
Final Config = merge(
  Defaults,
  User Config (~/.config/gemini-cli/config.yaml),
  Project Config (.gemini/config.yaml),
  Extension Configs (from installed extensions),
  Environment Variables,
  CLI Flags
)
```

**Precedence (highest to lowest):**
1. CLI Flags
2. Environment Variables  
3. Project Config
4. User Config
5. Extension Configs
6. Defaults

---

## Configuration File Locations

### User Configuration

**Path:** `~/.config/gemini-cli/config.yaml` (or `%APPDATA%\gemini-cli\config.yaml` on Windows)

**Purpose:** User-wide settings

**Example:**

```yaml
model:
  name: gemini-2.0-flash-exp
  maxSessionTurns: 50

ui:
  theme: "dark"
  showLineNumbers: true

tools:
  autoAccept: false
  useRipgrep: true
```

---

### Project Configuration

**Path:** `.gemini/config.yaml` (in project root)

**Purpose:** Project-specific settings

**Example:**

```yaml
tools:
  exclude: ["dangerous_tool"]
  
context:
  fileName: ["INSTRUCTIONS.md", "PROJECT_CONTEXT.md"]

mcpServers:
  project-tools:
    command: "node"
    args: ["./tools/mcp-server.js"]
```

---

### Extension Configuration

**Path:** `<extension-dir>/gemini-extension.json`

**Purpose:** Extension-provided settings

**Example:**

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "mcpServers": {
    "extension-server": {
      "command": "node",
      "args": ["server.js"]
    }
  },
  "excludeTools": ["unwanted_tool"],
  "settings": [
    {
      "key": "extension.customSetting",
      "default": true,
      "description": "Custom extension setting"
    }
  ]
}
```

---

## Complete Settings Schema

### General Settings

#### `general.preferredEditor`

- **Type:** `string`
- **Default:** `undefined`
- **Description:** The preferred editor to open files in
- **Example:** `"code"`, `"vim"`, `"subl"`

---

#### `general.vimMode`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Enable Vim keybindings

---

#### `general.disableAutoUpdate`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Disable automatic updates

---

#### `general.disableUpdateNag`

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Disable update notification prompts

---

#### `general.checkpointing.enabled`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Description:** Enable session checkpointing for recovery

---

#### `general.enablePromptCompletion`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable AI-powered prompt completion suggestions while typing

---

#### `general.retryFetchErrors`

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Retry on "exception TypeError: fetch failed sending request" errors

---

#### `general.debugKeystrokeLogging`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Enable debug logging of keystrokes to the console

---

#### `general.sessionRetention.enabled`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Enable automatic session cleanup

---

#### `general.sessionRetention.maxAge`

- **Type:** `string`
- **Default:** `undefined`
- **Description:** Maximum age of sessions to keep (e.g., "30d", "7d", "24h", "1w")
- **Example:** `"30d"` (30 days)

---

#### `general.sessionRetention.maxCount`

- **Type:** `number`
- **Default:** `undefined`
- **Description:** Maximum number of sessions to keep (most recent)

---

#### `general.sessionRetention.minRetention`

- **Type:** `string`
- **Default:** `"24h"`
- **Description:** Minimum retention period (safety limit)

---

### Output Settings

#### `output.format`

- **Type:** `enum`
- **Default:** `"text"`
- **Show in Dialog:** Yes
- **Options:** 
  - `"text"` - Human-readable text output
  - `"json"` - JSON-formatted output
- **Description:** The format of the CLI output

---

### UI Settings

#### `ui.theme`

- **Type:** `string`
- **Default:** `undefined` (auto-detect)
- **Description:** The color theme for the UI
- **Built-in Themes:**
  - `"dark"`
  - `"light"`
  - `"solarized-dark"`
  - `"solarized-light"`
  - `"monokai"`
  - `"dracula"`

---

#### `ui.customThemes`

- **Type:** `object`
- **Default:** `{}`
- **Description:** Custom theme definitions
- **Example:**

```yaml
ui:
  customThemes:
    my-theme:
      name: "My Theme"
      colors:
        background: "#1e1e1e"
        foreground: "#d4d4d4"
        primary: "#4fc3f7"
```

---

#### `ui.hideWindowTitle`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Hide the window title bar

---

#### `ui.showStatusInTitle`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Show Gemini CLI status and thoughts in the terminal window title

---

#### `ui.hideTips`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Hide helpful tips in the UI

---

#### `ui.hideBanner`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Hide the application banner

---

#### `ui.hideContextSummary`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Hide the context summary (GEMINI.md, MCP servers) above the input

---

#### `ui.footer.hideCWD`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Hide the current working directory path in the footer

---

#### `ui.footer.hideSandboxStatus`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Hide the sandbox status indicator in the footer

---

#### `ui.footer.hideModelInfo`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Hide the model name and context usage in the footer

---

#### `ui.hideFooter`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Hide the footer from the UI

---

#### `ui.showMemoryUsage`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Display memory usage information in the UI

---

#### `ui.showLineNumbers`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Show line numbers in the chat

---

#### `ui.showCitations`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Show citations for generated text in the chat

---

#### `ui.useFullWidth`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Use the entire width of the terminal for output

---

#### `ui.customWittyPhrases`

- **Type:** `array`
- **Default:** `[]`
- **Description:** Custom witty phrases to display during loading
- **Example:**

```yaml
ui:
  customWittyPhrases:
    - "Analyzing the flux capacitor..."
    - "Consulting the AI overlords..."
```

---

#### `ui.accessibility.disableLoadingPhrases`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Disable loading phrases for accessibility

---

#### `ui.accessibility.screenReader`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Render output in plain-text to be more screen reader accessible

---

### IDE Settings

#### `ide.enabled`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable IDE integration mode

---

#### `ide.hasSeenNudge`

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Whether the user has seen the IDE integration nudge
- **Internal:** Yes (not shown in UI)

---

### Privacy Settings

#### `privacy.usageStatisticsEnabled`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Description:** Enable collection of usage statistics

---

### Telemetry Settings

#### `telemetry`

- **Type:** `object`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** Telemetry configuration
- **Structure:**

```typescript
interface TelemetrySettings {
  enabled: boolean;
  endpoint?: string;
  projectId?: string;
}
```

---

### Model Settings

#### `model.name`

- **Type:** `string`
- **Default:** `undefined` (uses DEFAULT_GEMINI_MODEL)
- **Description:** The Gemini model to use for conversations
- **Examples:**
  - `"gemini-2.0-flash-exp"`
  - `"gemini-1.5-pro"`
  - `"gemini-1.5-flash"`

---

#### `model.maxSessionTurns`

- **Type:** `number`
- **Default:** `-1` (unlimited)
- **Show in Dialog:** Yes
- **Description:** Maximum number of user/model/tool turns to keep in a session

---

#### `model.summarizeToolOutput`

- **Type:** `object`
- **Default:** `undefined`
- **Description:** Settings for summarizing tool output
- **Structure:**

```yaml
model:
  summarizeToolOutput:
    tool_name:
      tokenBudget: 1000
```

---

#### `model.chatCompression`

- **Type:** `object`
- **Default:** `undefined`
- **Description:** Chat compression settings
- **Structure:**

```typescript
interface ChatCompressionSettings {
  enabled: boolean;
  strategy?: 'auto' | 'manual';
  threshold?: number;
}
```

---

#### `model.skipNextSpeakerCheck`

- **Type:** `boolean`
- **Default:** `true`
- **Show in Dialog:** Yes
- **Description:** Skip the next speaker check

---

### Context Settings

#### `context.fileName`

- **Type:** `string | string[]`
- **Default:** `undefined`
- **Description:** The name(s) of the context file(s)
- **Default Value:** `["GEMINI.md", "INSTRUCTIONS.md"]`
- **Example:**

```yaml
context:
  fileName: ["PROJECT_CONTEXT.md", "RULES.md"]
```

---

#### `context.importFormat`

- **Type:** `string`
- **Default:** `undefined`
- **Options:** `"tree"` | `"flat"`
- **Description:** The format to use when importing memory

---

#### `context.discoveryMaxDirs`

- **Type:** `number`
- **Default:** `200`
- **Show in Dialog:** Yes
- **Description:** Maximum number of directories to search for memory

---

#### `context.includeDirectories`

- **Type:** `array`
- **Default:** `[]`
- **Merge Strategy:** `CONCAT`
- **Description:** Additional directories to include in the workspace context
- **Example:**

```yaml
context:
  includeDirectories:
    - "/path/to/external/docs"
    - "/path/to/shared/lib"
```

---

#### `context.loadMemoryFromIncludeDirectories`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Whether to load memory files from include directories

---

#### `context.fileFiltering.respectGitIgnore`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Respect .gitignore files when searching

---

#### `context.fileFiltering.respectGeminiIgnore`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Respect .geminiignore files when searching

---

#### `context.fileFiltering.enableRecursiveFileSearch`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable recursive file search functionality

---

#### `context.fileFiltering.disableFuzzySearch`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Disable fuzzy search when searching for files

---

### Tools Settings

#### `tools.sandbox`

- **Type:** `boolean | string`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** Sandbox execution environment
- **Values:**
  - `false` - No sandbox
  - `true` - Auto-detect (Docker/Podman)
  - `"docker"` - Use Docker
  - `"podman"` - Use Podman
  - `"sandbox-exec"` - Use macOS Seatbelt

---

#### `tools.shell.enableInteractiveShell`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Use node-pty for an interactive shell experience. Fallback to child_process still applies.

---

#### `tools.shell.pager`

- **Type:** `string`
- **Default:** `"cat"`
- **Description:** The pager command to use for shell output

---

#### `tools.shell.showColor`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Show color in shell output

---

#### `tools.autoAccept`

- **Type:** `boolean`
- **Default:** `false`
- **Show in Dialog:** Yes
- **Description:** Automatically accept and execute tool calls that are considered safe (e.g., read-only operations)

---

#### `tools.core`

- **Type:** `array`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** Paths to core tool definitions

---

#### `tools.allowed`

- **Type:** `array`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** A list of tool names that will bypass the confirmation dialog
- **Example:**

```yaml
tools:
  allowed:
    - "read_file"
    - "glob"
    - "search_file_content"
```

---

#### `tools.exclude`

- **Type:** `array`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Merge Strategy:** `UNION`
- **Description:** Tool names to exclude from discovery
- **Example:**

```yaml
tools:
  exclude:
    - "dangerous_tool"
    - "deprecated_tool"
```

---

#### `tools.discoveryCommand`

- **Type:** `string`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** Command to run for tool discovery
- **Example:** `"node discover-tools.js"`

---

#### `tools.callCommand`

- **Type:** `string`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** Command to run for tool calls
- **Example:** `"node call-tool.js"`

---

#### `tools.useRipgrep`

- **Type:** `boolean`
- **Default:** `true`
- **Show in Dialog:** Yes
- **Description:** Use ripgrep for file content search instead of the fallback implementation

---

#### `tools.enableToolOutputTruncation`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable truncation of large tool outputs

---

#### `tools.truncateToolOutputThreshold`

- **Type:** `number`
- **Default:** `50000` characters
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Truncate tool output if it is larger than this many characters. Set to -1 to disable.

---

#### `tools.truncateToolOutputLines`

- **Type:** `number`
- **Default:** `100` lines
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** The number of lines to keep when truncating tool output

---

#### `tools.enableMessageBusIntegration`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable policy-based tool confirmation via message bus integration

---

### MCP Settings

#### `mcpServers`

- **Type:** `object`
- **Default:** `{}`
- **Requires Restart:** Yes
- **Merge Strategy:** `SHALLOW_MERGE`
- **Description:** Configuration for MCP servers
- **Structure:**

```yaml
mcpServers:
  server-name:
    # Stdio transport
    command: "node"
    args: ["server.js"]
    env:
      API_KEY: "xxx"
    cwd: "/path/to/server"
    
    # OR HTTP transport
    url: "https://mcp-server.example.com"
    httpUrl: "https://mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer token"
    
    # OR OAuth
    oauth:
      enabled: true
      authorizationUrl: "https://auth.example.com/authorize"
      tokenUrl: "https://auth.example.com/token"
      scopes: ["read", "write"]
    
    # Optional
    trust: true  # Auto-approve tools from this server
    timeout: 600000  # 10 minutes
    includeTools: ["tool1", "tool2"]  # Only these tools
    excludeTools: ["tool3"]  # All except these
    
    # Extension metadata (if from extension)
    extension:
      id: "extension-id"
```

---

#### `mcp.serverCommand`

- **Type:** `string`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** Command to start an MCP server (legacy, use `mcpServers` instead)

---

#### `mcp.allowed`

- **Type:** `array`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** A list of MCP servers to allow

---

#### `mcp.excluded`

- **Type:** `array`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** A list of MCP servers to exclude

---

### Smart Edit & TODOs

#### `useSmartEdit`

- **Type:** `boolean`
- **Default:** `true`
- **Description:** Enable the smart-edit tool instead of the replace tool
- **Category:** Advanced

---

#### `useWriteTodos`

- **Type:** `boolean`
- **Default:** `false`
- **Description:** Enable the write_todos_list tool
- **Category:** Advanced
- **Status:** Experimental

---

### Security Settings

#### `security.disableYoloMode`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Disable YOLO mode, even if enabled by a flag
- **Note:** "YOLO mode" bypasses all confirmations (dangerous)

---

#### `security.folderTrust.enabled`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable folder trust system

---

#### `security.auth.selectedType`

- **Type:** `string`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Options:** `"api_key"` | `"oauth"` | `"service_account"`
- **Description:** The currently selected authentication type

---

#### `security.auth.enforcedType`

- **Type:** `string`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** The required auth type. If this does not match the selected auth type, the user will be prompted to re-authenticate.

---

#### `security.auth.useExternal`

- **Type:** `boolean`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Description:** Whether to use an external authentication flow

---

### Advanced Settings

#### `advanced.autoConfigureMemory`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Description:** Automatically configure Node.js memory limits

---

#### `advanced.dnsResolutionOrder`

- **Type:** `string`
- **Default:** `undefined`
- **Requires Restart:** Yes
- **Options:** `"ipv4first"` | `"verbatim"`
- **Description:** The DNS resolution order

---

#### `advanced.excludedEnvVars`

- **Type:** `array`
- **Default:** `["DEBUG", "DEBUG_MODE"]`
- **Merge Strategy:** `UNION`
- **Description:** Environment variables to exclude from project context

---

#### `advanced.bugCommand`

- **Type:** `object`
- **Default:** `undefined`
- **Description:** Configuration for the bug report command
- **Structure:**

```typescript
interface BugCommandSettings {
  url?: string;
  includeSystemInfo?: boolean;
}
```

---

### Experimental Settings

#### `experimental.extensionManagement`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Description:** Enable extension management features

---

#### `experimental.useModelRouter`

- **Type:** `boolean`
- **Default:** `true`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable model routing to route requests to the best model based on complexity

---

#### `experimental.codebaseInvestigatorSettings.enabled`

- **Type:** `boolean`
- **Default:** `false`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Enable the Codebase Investigator agent

---

#### `experimental.codebaseInvestigatorSettings.maxNumTurns`

- **Type:** `number`
- **Default:** `15`
- **Requires Restart:** Yes
- **Show in Dialog:** Yes
- **Description:** Maximum number of turns for the Codebase Investigator agent

---

#### `experimental.codebaseInvestigatorSettings.maxTimeMinutes`

- **Type:** `number`
- **Default:** `5`
- **Requires Restart:** Yes
- **Description:** Maximum time for the Codebase Investigator agent (in minutes)

---

#### `experimental.codebaseInvestigatorSettings.thinkingBudget`

- **Type:** `number`
- **Default:** `-1` (unlimited)
- **Requires Restart:** Yes
- **Description:** The thinking budget for the Codebase Investigator agent

---

#### `experimental.codebaseInvestigatorSettings.model`

- **Type:** `string`
- **Default:** `DEFAULT_GEMINI_MODEL`
- **Requires Restart:** Yes
- **Description:** The model to use for the Codebase Investigator agent

---

### Extensions Settings

#### `extensions.disabled`

- **Type:** `array`
- **Default:** `[]`
- **Requires Restart:** Yes
- **Merge Strategy:** `UNION`
- **Description:** List of disabled extensions
- **Example:**

```yaml
extensions:
  disabled:
    - "unwanted-extension"
```

---

#### `extensions.workspacesWithMigrationNudge`

- **Type:** `array`
- **Default:** `[]`
- **Merge Strategy:** `UNION`
- **Description:** List of workspaces for which the migration nudge has been shown
- **Internal:** Yes

---

## Environment Variables

### Core Environment Variables

#### `GEMINI_API_KEY`

- **Purpose:** Gemini AI API key
- **Required:** Yes (unless using OAuth/service account)
- **Example:** `export GEMINI_API_KEY="AIza..."`

---

#### `GOOGLE_API_KEY`

- **Purpose:** Alias for `GEMINI_API_KEY`
- **Required:** No
- **Note:** Falls back to `GEMINI_API_KEY` if not set

---

#### `GOOGLE_CSE_ID`

- **Purpose:** Google Custom Search Engine ID (for `google_web_search` tool)
- **Required:** No (only if using web search)

---

### Sandbox Environment Variables

#### `SANDBOX`

- **Purpose:** Enable sandbox execution
- **Values:**
  - `false` - Disable sandbox
  - `docker` - Use Docker
  - `podman` - Use Podman
  - `sandbox-exec` - Use macOS Seatbelt
- **Example:** `export SANDBOX=docker`

---

#### `GEMINI_SANDBOX`

- **Purpose:** Alias for `SANDBOX`
- **Values:** Same as `SANDBOX`

---

### Prompt Customization

#### `GEMINI_SYSTEM_MD`

- **Purpose:** Override system prompt with custom file
- **Values:**
  - `true` / `1` - Use `~/.gemini/system.md`
  - `/path/to/file.md` - Use custom file
  - `false` / `0` - Use built-in prompt
- **Example:** `export GEMINI_SYSTEM_MD=/path/to/custom-prompt.md`

---

#### `GEMINI_WRITE_SYSTEM_MD`

- **Purpose:** Write generated system prompt to file
- **Values:**
  - `true` / `1` - Write to `~/.gemini/system.md`
  - `/path/to/file.md` - Write to custom path
- **Example:** `export GEMINI_WRITE_SYSTEM_MD=true`

---

### Debug & Development

#### `DEBUG`

- **Purpose:** Enable debug mode
- **Values:** `true` / `1` - Enable
- **Effect:** Verbose logging, MCP stderr logging
- **Example:** `export DEBUG=1`

---

#### `DEBUG_MODE`

- **Purpose:** Alias for `DEBUG`

---

#### `GEMINI_CLI_DEBUG`

- **Purpose:** Gemini CLI-specific debug flag

---

### Authentication

#### `GOOGLE_APPLICATION_CREDENTIALS`

- **Purpose:** Path to service account JSON file
- **Required:** Only for service account auth
- **Example:** `export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json`

---

### IDE Integration

#### `GEMINI_CLI_IDE_PORT`

- **Purpose:** Port for IDE companion server
- **Default:** Auto-assigned
- **Example:** `export GEMINI_CLI_IDE_PORT=3000`

---

### A2A Server

#### `CODER_AGENT_PORT`

- **Purpose:** Port for A2A server
- **Default:** `41242`
- **Example:** `export CODER_AGENT_PORT=8080`

---

#### `GOOGLE_CLOUD_PROJECT`

- **Purpose:** Google Cloud project ID (for A2A persistence)
- **Required:** Only for A2A with GCS

---

### Node.js Specific

#### `NODE_ENV`

- **Purpose:** Node environment
- **Values:** `development` | `production`
- **Effect:** Changes error reporting, logging

---

#### `NODE_OPTIONS`

- **Purpose:** Node.js options
- **Example:** `export NODE_OPTIONS="--max-old-space-size=4096"`

---

### Other

#### `TERM`

- **Purpose:** Terminal type
- **Default:** `xterm-256color` (set by Gemini CLI)

---

#### `PAGER`

- **Purpose:** Pager command for shell output
- **Default:** `cat` (overridden by Gemini CLI)
- **Override:** Use `tools.shell.pager` setting

---

## Configuration Layers & Merging

### Merge Strategies

#### REPLACE (default)

Overwrites the value completely:

```yaml
# User config
tools:
  exclude: ["tool1"]

# Project config
tools:
  exclude: ["tool2"]

# Result: ["tool2"] (project replaces user)
```

---

#### CONCAT

Concatenates arrays:

```yaml
# User config
context:
  includeDirectories: ["/user/dir"]

# Project config
context:
  includeDirectories: ["/project/dir"]

# Result: ["/user/dir", "/project/dir"]
```

---

#### UNION

Merges arrays with unique values:

```yaml
# User config
tools:
  exclude: ["tool1", "tool2"]

# Extension config
tools:
  exclude: ["tool2", "tool3"]

# Result: ["tool1", "tool2", "tool3"]
```

---

#### SHALLOW_MERGE

Merges objects (top-level keys):

```yaml
# User config
mcpServers:
  server1:
    command: "node server1.js"

# Project config
mcpServers:
  server2:
    command: "node server2.js"

# Result:
mcpServers:
  server1:
    command: "node server1.js"
  server2:
    command: "node server2.js"
```

---

### Configuration Priority Example

Given these sources:

**Defaults:**
```yaml
model:
  name: "gemini-2.0-flash-exp"
  maxSessionTurns: -1
```

**User Config (`~/.config/gemini-cli/config.yaml`):**
```yaml
model:
  name: "gemini-1.5-pro"
  maxSessionTurns: 50
```

**Project Config (`.gemini/config.yaml`):**
```yaml
model:
  maxSessionTurns: 100
```

**Environment Variable:**
```bash
export GEMINI_MODEL="gemini-2.0-flash-thinking-exp"
```

**Result:**
```yaml
model:
  name: "gemini-2.0-flash-thinking-exp"  # From env var (highest priority)
  maxSessionTurns: 100  # From project config (overrides user)
```

---

## Keyboard Shortcuts

### Complete Shortcut Reference

All keyboard shortcuts are defined in `packages/cli/src/config/keyBindings.ts`:

#### Basic Bindings

| Command | Keys | Description |
|---------|------|-------------|
| `RETURN` | `Return` | Default return key |
| `ESCAPE` | `Escape` | Escape key |

---

#### Cursor Movement

| Command | Keys | Description |
|---------|------|-------------|
| `HOME` | `Ctrl+A` | Move to start of line |
| `END` | `Ctrl+E` | Move to end of line |

---

#### Text Deletion

| Command | Keys | Description |
|---------|------|-------------|
| `KILL_LINE_RIGHT` | `Ctrl+K` | Delete from cursor to end |
| `KILL_LINE_LEFT` | `Ctrl+U` | Delete from cursor to start |
| `CLEAR_INPUT` | `Ctrl+C` | Clear current input |
| `DELETE_WORD_BACKWARD` | `Ctrl+Backspace`, `Cmd+Backspace` | Delete word backward |

---

#### Screen Control

| Command | Keys | Description |
|---------|------|-------------|
| `CLEAR_SCREEN` | `Ctrl+L` | Clear screen |

---

#### History Navigation

| Command | Keys | Description |
|---------|------|-------------|
| `HISTORY_UP` | `Ctrl+P` | Previous history |
| `HISTORY_DOWN` | `Ctrl+N` | Next history |
| `NAVIGATION_UP` | `Up` | Navigate up |
| `NAVIGATION_DOWN` | `Down` | Navigate down |

---

#### Auto-completion

| Command | Keys | Description |
|---------|------|-------------|
| `ACCEPT_SUGGESTION` | `Tab`, `Return` | Accept suggestion |
| `COMPLETION_UP` | `Up`, `Ctrl+P` | Move up in completion |
| `COMPLETION_DOWN` | `Down`, `Ctrl+N` | Move down in completion |

---

#### Text Input

| Command | Keys | Description |
|---------|------|-------------|
| `SUBMIT` | `Return` (no modifiers) | Submit input |
| `NEWLINE` | `Ctrl+Return`, `Cmd+Return`, `Shift+Return`, `Ctrl+J`, `Paste+Return` | New line in input |

---

#### External Tools

| Command | Keys | Description |
|---------|------|-------------|
| `OPEN_EXTERNAL_EDITOR` | `Ctrl+X` | Open external editor |
| `PASTE_CLIPBOARD_IMAGE` | `Ctrl+V` | Paste image from clipboard |

---

#### App Level Bindings

| Command | Keys | Description |
|---------|------|-------------|
| `SHOW_ERROR_DETAILS` | `Ctrl+O` | Show error details |
| `SHOW_FULL_TODOS` | `Ctrl+T` | Show full TODO list |
| `TOGGLE_IDE_CONTEXT_DETAIL` | `Ctrl+G` | Toggle IDE context |
| `TOGGLE_MARKDOWN` | `Cmd+M` | Toggle markdown rendering |
| `QUIT` | `Ctrl+C` | Quit application |
| `EXIT` | `Ctrl+D` | Exit application |
| `SHOW_MORE_LINES` | `Ctrl+S` | Show more lines |

---

#### Shell Commands

| Command | Keys | Description |
|---------|------|-------------|
| `REVERSE_SEARCH` | `Ctrl+R` | Reverse search history |
| `SUBMIT_REVERSE_SEARCH` | `Return` | Submit reverse search |
| `ACCEPT_SUGGESTION_REVERSE_SEARCH` | `Tab` | Accept suggestion in reverse search |
| `TOGGLE_SHELL_INPUT_FOCUS` | `Ctrl+F` | Toggle shell input focus |

---

#### Suggestion Expansion

| Command | Keys | Description |
|---------|------|-------------|
| `EXPAND_SUGGESTION` | `Right` | Expand suggestion |
| `COLLAPSE_SUGGESTION` | `Left` | Collapse suggestion |

---

### Customizing Shortcuts

**Currently:** Keyboard shortcuts are hardcoded in `defaultKeyBindings`

**Future:** May support user customization via config

---

## Extension Configuration

### Extension Manifest

**File:** `gemini-extension.json`

**Schema:**

```typescript
interface ExtensionConfig {
  name: string;              // Extension name
  version: string;           // Semantic version
  mcpServers?: Record<string, MCPServerConfig>; // MCP servers to register
  contextFileName?: string | string[]; // Context file names to add
  excludeTools?: string[];   // Tools to exclude
  settings?: ExtensionSetting[]; // Custom settings
}
```

---

### Extension Settings

**Schema:**

```typescript
interface ExtensionSetting {
  key: string;               // Setting key (e.g., "extension.setting")
  default: unknown;          // Default value
  description?: string;      // Human-readable description
  type?: string;             // Type hint
}
```

---

### Extension Install Metadata

**File:** `.gemini-cli-install-metadata.json`

**Purpose:** Tracks extension installation details

**Schema:**

```typescript
interface ExtensionInstallMetadata {
  name: string;
  version: string;
  installedAt: string;       // ISO timestamp
  source: string;            // Installation source (github, npm, local)
  originalPath?: string;     // Original path if local
}
```

---

## Hidden/Undocumented Settings

### Internal Settings

These settings exist but are not documented in official docs:

#### `general.checkpointing.enabled`

- **Purpose:** Session recovery via checkpointing
- **Status:** Experimental, disabled by default

---

#### `general.retryFetchErrors`

- **Purpose:** Auto-retry on specific fetch errors
- **Status:** Workaround for network issues

---

#### `tools.core`

- **Purpose:** Override core tool definitions
- **Status:** Advanced, for development only

---

#### `advanced.autoConfigureMemory`

- **Purpose:** Auto-configure Node.js `--max-old-space-size`
- **Status:** Experimental

---

### Undocumented Environment Variables

#### `GEMINI_CLI_SKIP_UPDATE_CHECK`

- **Purpose:** Skip update check on startup
- **Values:** `true` / `1`

---

#### `GEMINI_CLI_NO_TELEMETRY`

- **Purpose:** Disable telemetry
- **Values:** `true` / `1`

---

#### `GEMINI_CLI_CONFIG_PATH`

- **Purpose:** Override config file path
- **Values:** Path to config file

---

This comprehensive configuration reference documents every setting, environment variable, and keyboard shortcut in Gemini CLI, including hidden and undocumented options.

