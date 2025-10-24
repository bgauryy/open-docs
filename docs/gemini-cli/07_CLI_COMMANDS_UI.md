# Gemini CLI - Commands & UI Reference

**Generated:** 2024-10-24
**Purpose:** Complete documentation of all slash commands, keyboard shortcuts, and UI components

---

## Table of Contents

1. [Slash Commands Overview](#slash-commands-overview)
2. [Core Commands](#core-commands)
3. [Configuration Commands](#configuration-commands)
4. [Developer Commands](#developer-commands)
5. [Session Management](#session-management)
6. [Keyboard Shortcuts](#keyboard-shortcuts)
7. [UI Components](#ui-components)
8. [Hidden Commands](#hidden-commands)

---

## Slash Commands Overview

Gemini CLI supports **29+ slash commands** for managing the application, configuring settings, and interacting with tools.

**Command Structure:**

```
/command [subcommand] [args]
```

**Command Types:**

- **Built-in Commands** (`CommandKind.BUILT_IN`) - Core CLI functionality
- **Extension Commands** (`CommandKind.EXTENSION`) - Provided by extensions
- **Custom Commands** (`CommandKind.CUSTOM`) - User-defined commands

**Discovering Commands:**

```bash
/help           # Show help
/?              # Alias for /help
```

---

## Core Commands

### `/help` (alias: `/?`)

**Description:** Display help information for Gemini CLI

**Usage:**

```bash
/help
/?
```

**Output:** Opens help panel with:
- Available commands
- Keyboard shortcuts
- Quick tips
- Documentation links

---

### `/chat`

**Description:** Start a new chat session

**Usage:**

```bash
/chat [message]
```

**Behavior:**
- Creates new conversation context
- Clears current conversation history
- Optional: Start with initial message

---

### `/clear`

**Description:** Clear the terminal screen

**Usage:**

```bash
/clear
```

**Effect:**
- Clears visible terminal content
- Preserves conversation history
- Equivalent to `Ctrl+L`

---

### `/quit` (alias: `/exit`)

**Description:** Exit Gemini CLI

**Usage:**

```bash
/quit
```

**Equivalent:** `Ctrl+D` or `Ctrl+C`

---

## Configuration Commands

### `/settings`

**Description:** View and edit Gemini CLI settings

**Usage:**

```bash
/settings
```

**Action:** Opens settings dialog with:
- General settings
- UI preferences
- Tool configuration
- Model settings
- Privacy options

**Dialog Sections:**
- **General** - Editor, vim mode, updates
- **UI** - Theme, layout, visibility
- **Tools** - Sandbox, auto-accept, ripgrep
- **Model** - Model name, session limits
- **Context** - Memory files, discovery
- **Experimental** - Feature flags

---

### `/theme`

**Description:** Change color theme

**Usage:**

```bash
/theme [theme-name]
```

**Built-in Themes:**
- `dark`
- `light`
- `solarized-dark`
- `solarized-light`
- `monokai`
- `dracula`

**Custom Themes:** Define in settings:

```yaml
ui:
  customThemes:
    my-theme:
      name: "My Theme"
      colors:
        background: "#1e1e1e"
        foreground: "#d4d4d4"
```

---

### `/vim`

**Description:** Toggle Vim keybindings

**Usage:**

```bash
/vim
```

**Effect:** Enables/disables Vim-style navigation

**Setting:**

```yaml
general:
  vimMode: true
```

---

### `/editor [editor]`

**Description:** Set preferred external editor

**Usage:**

```bash
/editor code        # VSCode
/editor vim         # Vim
/editor subl        # Sublime Text
```

**Setting:**

```yaml
general:
  preferredEditor: "code"
```

---

### `/auth`

**Description:** Manage authentication

**Usage:**

```bash
/auth
```

**Action:** Opens authentication dialog with options:
- API Key
- OAuth
- Service Account

---

### `/privacy`

**Description:** Privacy and telemetry settings

**Usage:**

```bash
/privacy
```

**Action:** Opens privacy dialog:
- Usage statistics
- Telemetry settings
- Data collection preferences

---

### `/permissions`

**Description:** Manage folder trust settings

**Usage:**

```bash
/permissions
```

**Action:** Opens permissions dialog to configure trusted folders

**Requires:** `security.folderTrust.enabled: true`

---

## Developer Commands

### `/tools [desc]`

**Description:** List available Gemini CLI tools

**Usage:**

```bash
/tools              # List tool names
/tools desc         # Include descriptions
```

**Output:** Shows:
- Built-in tools (read_file, write_file, etc.)
- MCP tools (from external servers)
- Extension tools

**Filters:** Excludes MCP tools (see `/mcp` for those)

---

### `/mcp`

**Description:** Manage Model Context Protocol (MCP) servers

**Subcommands:**

#### `/mcp list` (aliases: `/mcp ls`, `/mcp nodesc`)

List configured MCP servers and tools

**Usage:**

```bash
/mcp list
/mcp ls
/mcp nodesc
```

**Output:**
- Server names
- Connection status
- Tool count
- Prompt count

---

#### `/mcp desc` (alias: `/mcp description`)

List with descriptions

**Usage:**

```bash
/mcp desc
/mcp description
```

**Output:** Same as `/mcp list` but includes tool/prompt descriptions

---

#### `/mcp schema`

List with full schemas

**Usage:**

```bash
/mcp schema
```

**Output:** Includes:
- Tool descriptions
- Parameter schemas
- Input/output types

---

#### `/mcp auth <server-name>`

Authenticate with OAuth-enabled MCP server

**Usage:**

```bash
/mcp auth my-server
```

**Flow:**
1. Opens browser for OAuth authentication
2. User authorizes application
3. Stores OAuth tokens
4. Re-discovers tools from server

**Requirements:**
- MCP server must have `oauth.enabled: true` in config
- Browser access

**Auto-Completion:** Tab-completes server names

---

#### `/mcp refresh`

Restart MCP servers and re-discover tools

**Usage:**

```bash
/mcp refresh
```

**Effect:**
- Stops all MCP servers
- Restarts servers
- Re-discovers tools
- Updates tool registry
- Reloads slash commands (from MCP prompts)

---

### `/extensions`

**Description:** Manage Gemini CLI extensions

**Usage:**

```bash
/extensions
```

**Action:** Opens extensions management interface

**Operations:**
- List installed extensions
- Install new extensions
- Update extensions
- Enable/disable extensions
- Remove extensions

**Extension Sources:**
- npm packages
- GitHub repositories
- Local directories

---

### `/memory`

**Description:** Show/manage memory (context) files

**Usage:**

```bash
/memory
```

**Output:** Displays:
- Discovered memory files (GEMINI.md, INSTRUCTIONS.md)
- File locations
- File sizes
- Last modified timestamps

**Memory Files:** Auto-discovered from:
- Current directory
- Parent directories (up to `context.discoveryMaxDirs`)
- Include directories (from settings)

---

### `/stats`

**Description:** Display usage statistics

**Usage:**

```bash
/stats
```

**Output:**
- Session duration
- Message count
- Token usage
- Tool calls
- API requests

---

### `/profile`

**Description:** Performance profiling

**Usage:**

```bash
/profile
```

**Output:**
- LLM response times
- Tool execution times
- Memory usage
- Cache statistics

---

## Session Management

### `/compress`

**Description:** Compress chat history to save context

**Usage:**

```bash
/compress
```

**Effect:**
- Summarizes old messages
- Reduces token usage
- Preserves important context

**Settings:**

```yaml
model:
  chatCompression:
    enabled: true
    strategy: "auto"
    threshold: 80000
```

---

### `/restore`

**Description:** Restore previous session

**Usage:**

```bash
/restore [session-id]
```

**Action:** Lists recent sessions or restores specified session

---

### `/init`

**Description:** Initialize Gemini CLI in project

**Usage:**

```bash
/init
```

**Effect:**
- Creates `.gemini/` directory
- Generates default `config.yaml`
- Creates `GEMINI.md` template

---

### `/terminal-setup`

**Description:** Setup terminal integration

**Usage:**

```bash
/terminal-setup
```

**Effect:**
- Configures shell aliases
- Sets up environment variables
- Adds completions

---

## Keyboard Shortcuts

### Navigation

| Shortcut | Action | Description |
|----------|--------|-------------|
| `Ctrl+A` | `HOME` | Move to start of line |
| `Ctrl+E` | `END` | Move to end of line |
| `Up` | `NAVIGATION_UP` | Previous message / Navigate up |
| `Down` | `NAVIGATION_DOWN` | Next message / Navigate down |
| `Ctrl+P` | `HISTORY_UP` | Previous command in history |
| `Ctrl+N` | `HISTORY_DOWN` | Next command in history |

---

### Editing

| Shortcut | Action | Description |
|----------|--------|-------------|
| `Ctrl+K` | `KILL_LINE_RIGHT` | Delete from cursor to end |
| `Ctrl+U` | `KILL_LINE_LEFT` | Delete from cursor to start |
| `Ctrl+C` | `CLEAR_INPUT` | Clear current input |
| `Ctrl+Backspace` | `DELETE_WORD_BACKWARD` | Delete word backward |
| `Cmd+Backspace` | `DELETE_WORD_BACKWARD` | Delete word backward (macOS) |

---

### Input

| Shortcut | Action | Description |
|----------|--------|-------------|
| `Return` | `SUBMIT` | Submit input (no modifiers) |
| `Ctrl+Return` | `NEWLINE` | New line in input |
| `Cmd+Return` | `NEWLINE` | New line in input (macOS) |
| `Shift+Return` | `NEWLINE` | New line in input |
| `Ctrl+J` | `NEWLINE` | New line in input |

---

### Auto-Completion

| Shortcut | Action | Description |
|----------|--------|-------------|
| `Tab` | `ACCEPT_SUGGESTION` | Accept auto-completion |
| `Right` | `EXPAND_SUGGESTION` | Expand suggestion |
| `Left` | `COLLAPSE_SUGGESTION` | Collapse suggestion |

---

### Application

| Shortcut | Action | Description |
|----------|--------|-------------|
| `Ctrl+L` | `CLEAR_SCREEN` | Clear screen |
| `Ctrl+X` | `OPEN_EXTERNAL_EDITOR` | Open external editor |
| `Ctrl+V` | `PASTE_CLIPBOARD_IMAGE` | Paste image from clipboard |
| `Ctrl+O` | `SHOW_ERROR_DETAILS` | Show error details |
| `Ctrl+T` | `SHOW_FULL_TODOS` | Show full TODO list |
| `Ctrl+G` | `TOGGLE_IDE_CONTEXT_DETAIL` | Toggle IDE context |
| `Cmd+M` | `TOGGLE_MARKDOWN` | Toggle markdown rendering |
| `Ctrl+S` | `SHOW_MORE_LINES` | Show more lines |
| `Ctrl+C` | `QUIT` | Quit application |
| `Ctrl+D` | `EXIT` | Exit application |

---

### Shell Mode

| Shortcut | Action | Description |
|----------|--------|-------------|
| `Ctrl+R` | `REVERSE_SEARCH` | Reverse search history |
| `Ctrl+F` | `TOGGLE_SHELL_INPUT_FOCUS` | Toggle shell input focus |

---

### Customizing Shortcuts

**Currently:** Keyboard shortcuts are hardcoded in `packages/cli/src/config/keyBindings.ts`

**Future:** May support user customization via settings

---

## UI Components

### Major Components

#### 1. **AppContainer**

**Location:** `packages/cli/src/ui/AppContainer.tsx`

**Purpose:** Main application container

**Features:**
- Message history rendering
- Input prompt
- Status indicators
- Error boundaries

---

#### 2. **InputPrompt**

**Location:** `packages/cli/src/ui/components/InputPrompt.tsx`

**Purpose:** User input field

**Features:**
- Multi-line input
- Auto-completion
- Command history
- Syntax highlighting
- Paste image support

---

#### 3. **MessageDisplay**

**Purpose:** Render conversation messages

**Message Types:**
- User messages
- AI responses
- Tool calls
- Error messages
- Info messages
- System messages

---

#### 4. **ToolConfirmation**

**Purpose:** Confirmation dialogs for tool execution

**Dialog Types:**
- **Edit Confirmation** - File diff preview
- **Command Confirmation** - Shell command preview
- **MCP Tool Confirmation** - External tool preview

**User Options:**
- Approve
- Reject
- Modify
- Always Approve

---

#### 5. **SettingsDialog**

**Location:** `packages/cli/src/ui/components/SettingsDialog.tsx`

**Purpose:** Settings management UI

**Sections:**
- General
- UI
- Tools
- Model
- Context
- Experimental
- MCP
- Extensions

---

#### 6. **ThemeDialog**

**Location:** `packages/cli/src/ui/components/ThemeDialog.tsx`

**Purpose:** Theme selection and customization

**Features:**
- Built-in theme preview
- Custom theme editor
- Live preview
- Export/import themes

---

#### 7. **HelpPanel**

**Purpose:** Display help information

**Sections:**
- Commands list
- Keyboard shortcuts
- Quick tips
- Documentation links

---

#### 8. **StatusBar** (Footer)

**Purpose:** Display status information

**Components:**
- Current working directory
- Sandbox status
- Model info (name, context usage)
- Memory usage (optional)

**Configuration:**

```yaml
ui:
  footer:
    hideCWD: false
    hideSandboxStatus: false
    hideModelInfo: false
  hideFooter: false
  showMemoryUsage: false
```

---

#### 9. **LoadingIndicator**

**Purpose:** Show loading state

**Features:**
- Witty loading phrases
- Progress spinner
- Estimated time
- Token count (if available)

**Phrases:** Customizable via `ui.customWittyPhrases`

**Disable:** `ui.accessibility.disableLoadingPhrases: true`

---

#### 10. **ErrorDisplay**

**Purpose:** Render error messages

**Error Types:**
- API errors (quota, network, auth)
- Tool execution errors
- Validation errors
- Internal errors

**Features:**
- Stack traces (debug mode)
- Error codes
- Retry suggestions
- Help links

---

### Dialog Types

#### Settings Dialog

**Trigger:** `/settings` command

**Sections:**
- General settings
- UI preferences
- Tool configuration
- Model settings
- Context settings
- Experimental features

---

#### Permissions Dialog

**Trigger:** `/permissions` command

**Features:**
- List trusted folders
- Add trusted folder
- Remove trusted folder
- Trust level configuration

---

#### Theme Dialog

**Trigger:** `/theme` command

**Features:**
- Theme selection
- Color customization
- Live preview
- Save custom theme

---

#### Extensions Dialog

**Trigger:** `/extensions` command

**Features:**
- List installed extensions
- Install extension (npm, GitHub, local)
- Update extensions
- Enable/disable extensions
- Remove extensions

---

#### Authentication Dialog

**Trigger:** `/auth` command

**Options:**
- API Key authentication
- OAuth authentication
- Service Account authentication

---

#### Privacy Dialog

**Trigger:** `/privacy` command

**Options:**
- Usage statistics toggle
- Telemetry configuration
- Data collection preferences

---

## Hidden Commands

### `/corgi`

**Description:** Display ASCII art corgi (Easter egg)

**Usage:**

```bash
/corgi
```

**Output:** ASCII art of a corgi dog

**Status:** Easter egg, not documented in official help

---

### `/about`

**Description:** Display about information

**Usage:**

```bash
/about
```

**Output:**
- Version information
- License
- Credits
- Repository link

---

### `/docs`

**Description:** Open documentation

**Usage:**

```bash
/docs [topic]
```

**Action:** Opens browser to documentation page

---

### `/bug`

**Description:** Report a bug

**Usage:**

```bash
/bug
```

**Action:** Opens bug report template or URL

**Configuration:**

```yaml
advanced:
  bugCommand:
    url: "https://github.com/google-gemini/gemini-cli/issues/new"
    includeSystemInfo: true
```

---

### `/setup-github`

**Description:** Setup GitHub integration

**Usage:**

```bash
/setup-github
```

**Action:** Configures GitHub extension

---

### `/model [model-name]`

**Description:** Switch Gemini model

**Usage:**

```bash
/model gemini-1.5-pro
/model gemini-2.0-flash-exp
```

**Effect:** Changes model for current session

**Persists:** No, session-only. Use settings for permanent change.

---

### `/ide`

**Description:** Toggle IDE integration mode

**Usage:**

```bash
/ide
```

**Effect:** Enables/disables IDE companion features

**Requires:** IDE extension (VSCode, Zed)

---

### `/copy`

**Description:** Copy last AI response to clipboard

**Usage:**

```bash
/copy
```

**Effect:** Copies last AI message to system clipboard

---

## Command Completion

### Auto-Completion

**Trigger:** `Tab` key

**Completion Types:**

1. **Command Names**
   - Complete `/` commands
   - Shows all available commands starting with prefix

2. **Command Arguments**
   - MCP server names (for `/mcp auth`)
   - Theme names (for `/theme`)
   - File paths (for file-related commands)

3. **Slash Command Hints**
   - After typing `/`, shows available commands
   - Displays command descriptions

---

### Custom Completions

**Extension Commands:** Extensions can provide custom completions

**Example:**

```typescript
const myCommand: SlashCommand = {
  name: 'my-command',
  completion: async (context, partialArg) => {
    // Return completion suggestions
    return ['option1', 'option2', 'option3'];
  },
};
```

---

## UI Customization

### Hiding UI Elements

```yaml
ui:
  hideBanner: false              # Hide application banner
  hideContextSummary: false      # Hide context summary (GEMINI.md, MCP)
  hideTips: false                # Hide helpful tips
  hideFooter: false              # Hide entire footer
  footer:
    hideCWD: false               # Hide current directory
    hideSandboxStatus: false     # Hide sandbox indicator
    hideModelInfo: false         # Hide model name/context
```

---

### Accessibility

```yaml
ui:
  accessibility:
    disableLoadingPhrases: false   # Disable witty loading phrases
    screenReader: false            # Plain text output for screen readers
```

---

### Performance

```yaml
ui:
  useFullWidth: false            # Use entire terminal width
  showLineNumbers: false         # Show line numbers in chat
```

---

## Command Gotchas

### 1. `/mcp auth` Requires Browser

**Issue:** OAuth authentication opens browser

**Workaround:** Use SSH tunneling or local port forwarding if running remotely

---

### 2. `/settings` Changes Require Restart

**Issue:** Some settings need restart to take effect

**Settings Requiring Restart:**
- `tools.sandbox`
- `general.vimMode`
- `experimental.*`
- `tools.core`

---

### 3. `/compress` May Lose Context

**Issue:** Chat compression summarizes old messages, potentially losing details

**Mitigation:** Important information may be lost. Use with caution.

---

### 4. `/tools` vs `/mcp list`

**Difference:**
- `/tools` - Shows built-in tools only
- `/mcp list` - Shows MCP tools only

**To see all tools:** Run both commands

---

### 5. `/theme` Custom Themes Not Validated

**Issue:** Invalid color values may break UI

**Recommendation:** Test custom themes before saving

---

### 6. `/extensions` May Break on Update

**Issue:** Extensions marked experimental, APIs may change

**Recommendation:** Pin extension versions in production

---

This comprehensive reference documents all slash commands, keyboard shortcuts, and UI components in Gemini CLI, including hidden commands.

