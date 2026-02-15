# Claude Code Plugin System - Complete Reference

**Extracted from Source Code Analysis**

Version: 2.1.42  
Last Updated: February 15, 2026

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture](#system-architecture)
3. [Complete Command Reference](#complete-command-reference)
4. [Internal Flows](#internal-flows)
5. [File Structure & Naming](#file-structure--naming)
6. [Data Structures](#data-structures)
7. [Implementation Details](#implementation-details)
8. [Marketplace System](#marketplace-system)
9. [Troubleshooting Guide](#troubleshooting-guide)
10. [Advanced Usage](#advanced-usage)

---

## Executive Summary

### What is the Plugin System?

Claude Code includes a **marketplace-based plugin system** for extending the CLI through installable plugins. This is a CLI-only feature - plugins are installed and managed via `claude plugin` and `claude plugin marketplace` commands.

### Key Facts

- **11 Commands Total**: 7 plugin commands + 4 marketplace commands
- **Plugin Cache Directory**: `~/.claude/plugins/` (or `~/.claude/cowork_plugins/` in cowork mode; override with `CLAUDE_CODE_PLUGIN_CACHE_DIR`)
- **Marketplace Support**: GitHub repos, Git URLs, HTTP URLs, local directories/files
- **Manifest Validation**: `claude plugin validate <path>` validates plugin and marketplace manifests
- **Telemetry**: All commands send usage analytics (if enabled)
- **Exit Codes (validate)**: 0 (success), 1 (validation failed), 2 (unexpected error)

### Quick Start

```bash
# Add a marketplace
claude plugin marketplace add owner/repo

# Install a plugin
claude plugin install my-plugin@my-marketplace

# List installed plugins
claude plugin list

# Validate a plugin manifest
claude plugin validate ./my-plugin
```

### Important Distinction

There is **NO SDK-level plugin system** for programmatic use. The plugin system is CLI-only:

| Feature | CLI Plugin System |
|---------|------------------|
| **Type** | Manifest-based (JSON files) |
| **Commands** | `claude plugin ...`, `claude plugin marketplace ...` |
| **Discovery** | Marketplaces |
| **Installation** | `plugin install` command |
| **Configuration** | `~/.claude/settings.json` (plus plugin cache under `~/.claude/plugins/`) |
| **Use Case** | CLI tool extensions |

---

## System Architecture

### Component Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code CLI                           │
│  ┌───────────────────────────────────────────────────────┐  │
│  │            Command Parser & Router                     │  │
│  │  - plugin install/uninstall/list/enable/disable/update │  │
│  │  - plugin validate                                     │  │
│  │  - plugin marketplace add/list/remove/update            │  │
│  └────────────────────┬──────────────────────────────────┘  │
│                       │                                       │
│  ┌────────────────────▼──────────────────────────────────┐  │
│  │         Core Functions                                 │  │
│  │  • pluginValidateHandler()                             │  │
│  │  • pluginListHandler()                                 │  │
│  │  • pluginInstallHandler()                              │  │
│  │  • pluginUninstallHandler()                            │  │
│  │  • pluginEnableHandler()                               │  │
│  │  • pluginDisableHandler()                              │  │
│  │  • pluginUpdateHandler()                               │  │
│  │  • marketplaceAddHandler()                             │  │
│  │  • marketplaceListHandler()                            │  │
│  │  • marketplaceRemoveHandler()                          │  │
│  │  • marketplaceUpdateHandler()                          │  │
│  └────────────────────┬──────────────────────────────────┘  │
│                       │                                       │
│  ┌────────────────────▼──────────────────────────────────┐  │
│  │      Configuration & State Management                  │  │
│  │  • settings.json (enabledPlugins, plugin config)        │  │
│  │  • known_marketplaces.json (marketplace list)           │  │
│  │  • installed_plugins.json (installed plugin state)      │  │
│  └────────────────────┬──────────────────────────────────┘  │
└────────────────────────┼──────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
┌──────────────────┐           ┌──────────────────┐
│  File System     │           │   Remote Sources │
│  ~~~~~~~~~~~~~~~~ │           │  ~~~~~~~~~~~~~~~ │
│  ~/.claude/      │           │  • GitHub repos  │
│    ├─ settings.json          │  • Git URLs      │
│    ├─ plugins/               │  • HTTP URLs     │
│    │   ├─ known_marketplaces.json
│    │   └─ installed_plugins.json
│    └─ ...                    │  • Local paths   │
└──────────────────┘           └──────────────────┘
```

### Data Flow: Plugin Installation

```
User runs: claude plugin install my-plugin
           │
           ▼
┌──────────────────────────────┐
│ 1. Parse Plugin Identifier   │ → Parses "plugin@marketplace"
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 2. Resolve Marketplace        │ → Searches configured marketplaces
│    (known marketplaces)       │   for plugin definition
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 3. Check Installation Status  │ → Reads merged settings + installed state
│                               │   Checks enabledPlugins and installed_plugins.json
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 4. Download Manifest          │ → Fetches plugin.json from
│                               │   marketplace source
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 5. Validate Manifest          │ → Validates plugin/marketplace manifest schema
│                               │   Checks required fields and formats
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 6. Resolve Dependencies       │ → Checks dependencies in manifest
│                               │   (Currently manual, no auto-install)
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 7. Download Plugin Files      │ → Downloads plugin archive
│                               │   Caches under ~/.claude/plugins/ (versioned cache)
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 8. Run Activation Script      │ → Executes activate.sh/activate.js
│    (if present)               │   (Optional lifecycle hook)
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 9. Update Configuration       │ → Updates enabledPlugins in settings.json
│                               │   (scope: user/project/local)
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 10. Send Telemetry            │ → "tengu_plugin_install_command"
│                               │   Reports usage analytics
└──────────┬───────────────────┘
           │
           ▼
      Installation Complete
      ✓ Plugin active on next session
```

### Plugin Lifecycle States

```
┌──────────────────┐
│   Available      │  Plugin listed in marketplace
│  (in marketplace)│  Not yet installed
└────────┬─────────┘
         │
         │ claude plugin install <plugin>
         ▼
┌──────────────────┐
│   Installed      │  Files on disk at ~/.claude/plugins/
│ (files on disk)  │  May be enabled or disabled
└────────┬─────────┘
         │
         │ claude plugin enable <plugin>
         ▼
┌──────────────────┐
│     Enabled      │  `enabledPlugins[plugin@marketplace] = true` in settings
│  (in settings)   │  Not active until session restart
└────────┬─────────┘
         │
         │ Session restart / CLI init
         ▼
┌──────────────────┐
│     Active       │  Plugin loaded and running
│ (in session)     │  Functionality available
└────────┬─────────┘
         │
         │ claude plugin disable <plugin>
         ▼
┌──────────────────┐
│    Disabled      │  `enabledPlugins[plugin@marketplace] = false` in settings
│  (in settings)   │  Files remain on disk (cache)
└────────┬─────────┘
         │
         │ claude plugin uninstall <plugin>
         ▼
┌──────────────────┐
│    Uninstalled   │  Files deleted from disk
│   (removed)      │  Removed from settings and installed state
└──────────────────┘
```

### Configuration File Hierarchy

```
~/.claude/
├── settings.json                 # Main settings (including enabledPlugins)
│   └── enabledPlugins: {         # Plugin enablement (plugin@marketplace → true/false)
│         "my-plugin@my-market": true
│       }
│
└── plugins/                      # Plugin cache directory (or cowork_plugins/)
    ├── known_marketplaces.json   # Configured marketplaces (sources + metadata)
    ├── installed_plugins.json    # Installed plugins state (derived/synced)
    ├── marketplaces/             # Downloaded marketplace manifests
    ├── cache/                    # Versioned plugin cache copies
    └── npm-cache/                # Optional npm cache for marketplace sources
```

---

## Complete Command Reference

### Plugin Commands

#### 1. `claude plugin install <plugin>`

**Purpose**: Install a plugin from configured marketplaces

**Syntax**:
```bash
claude plugin install <plugin> [options]
```

**Arguments**:
- `<plugin>` - Plugin identifier (required)

**Plugin Identifier Formats**:
```bash
# Search all marketplaces
claude plugin install my-plugin

# Specific marketplace
claude plugin install my-plugin@marketplace-name

# Specific version
claude plugin install my-plugin@1.2.3

# Marketplace AND version
claude plugin install my-plugin@marketplace-name@1.2.3
```

**Aliases**:
```bash
claude plugin i <plugin>
```

**Implementation**: `pluginInstallHandler`

**Internal Flow**:
```
1. Parse plugin identifier (plugin + marketplace)
2. Load configured marketplaces (including cached marketplaces and repo-provided entries)
3. Search marketplaces for plugin match
4. Validate not already installed (check `installed_plugins.json`)
5. Fetch plugin manifest from source
6. Validate manifest schema
7. Check dependency conflicts
8. Download plugin archive
9. Cache/extract under `~/.claude/plugins/` (versioned cache)
10. Run activation script (if present)
11. Update settings (scope-specific `settings.json`)
12. Update `enabledPlugins` mapping
13. Send telemetry: tengu_plugin_install_command
```

**Exit Codes**:
- `0` - Success
- `1` - Installation failed

**Success Output**:
```
✓ Successfully installed plugin: my-plugin
```

**Error Cases**:
```javascript
"Plugin not found in any marketplace"
"Plugin '<name>' is already installed"
"Manifest validation failed"
"Failed to download plugin manifest"
"Activation script failed: <error>"
```

**Example Usage**:
```bash
# Basic install
claude plugin install prettier-formatter

# From specific marketplace
claude plugin install prettier-formatter@official

# Specific version
claude plugin install prettier-formatter@2.1.0

# Full specification
claude plugin install prettier-formatter@official@2.1.0
```

---

#### 2. `claude plugin uninstall <plugin>`

**Purpose**: Remove an installed plugin

**Syntax**:
```bash
claude plugin uninstall <plugin>
```

**Arguments**:
- `<plugin>` - Plugin name (required)

**Aliases**:
```bash
claude plugin remove <plugin>
claude plugin rm <plugin>
```

**Implementation**: `pluginUninstallHandler`

**Internal Flow**:
```
1. Check if plugin exists in ~/.claude/plugins/
2. Check if plugin is enabled (read config)
3. If enabled: Run deactivation script (if present)
4. Remove plugin directory
5. Clean up unused dependencies (if not used by others)
6. Update settings (scope-specific `settings.json`)
7. Update `enabledPlugins` mapping
8. Send telemetry: tengu_plugin_uninstall_command
```

**Exit Codes**:
- `0` - Success
- `1` - Uninstallation failed

**Success Output**:
```
✓ Successfully uninstalled plugin: my-plugin
```

**Error Cases**:
```javascript
"Plugin not found"
"Plugin '<name>' is not installed"
"Deactivation script failed: <error>" // Warning only, continues
"File removal permission denied"
```

**Example Usage**:
```bash
# Basic uninstall
claude plugin uninstall prettier-formatter

# Using alias
claude plugin rm old-plugin
```

**Note**: Deactivation script failures are non-fatal warnings.

---

#### 3. `claude plugin enable <plugin>`

**Purpose**: Enable a previously disabled plugin

**Syntax**:
```bash
claude plugin enable <plugin>
```

**Arguments**:
- `<plugin>` - Plugin name (required)

**Implementation**: `pluginEnableHandler`

**Internal Flow**:
```
1. Check if plugin is installed (~/.claude/plugins/)
2. Check if plugin is already enabled
3. Run activation script (if present)
4. Update settings (scope-specific `settings.json`):
   - Set `enabledPlugins[plugin@marketplace] = true`
5. Send telemetry: tengu_plugin_enable_command
6. Plugin will be active in NEXT session (not immediate)
```

**Exit Codes**:
- `0` - Success
- `1` - Enable failed

**Success Output**:
```
✓ Successfully enabled plugin: my-plugin
Plugin will be active in the next session
```

**Error Cases**:
```javascript
"Plugin not found"
"Plugin '<name>' is not installed"
"Plugin '<name>' is already enabled"
"Activation script failed: <error>"
```

**Example Usage**:
```bash
# Enable a disabled plugin
claude plugin enable my-plugin
```

**Important**: Changes take effect in the **NEXT session**, not immediately. You must restart your Claude CLI session.

---

#### 4. `claude plugin disable <plugin>`

**Purpose**: Disable an enabled plugin without uninstalling

**Syntax**:
```bash
claude plugin disable <plugin>
```

**Arguments**:
- `<plugin>` - Plugin name (required)

**Implementation**: `pluginDisableHandler`

**Internal Flow**:
```
1. Check if plugin is installed
2. Check if plugin is currently enabled
3. Run deactivation script (if present)
4. Update settings (scope-specific `settings.json`):
   - Set `enabledPlugins[plugin@marketplace] = false`
5. Send telemetry: tengu_plugin_disable_command
6. Plugin will be inactive in NEXT session
```

**Exit Codes**:
- `0` - Success
- `1` - Disable failed

**Success Output**:
```
✓ Successfully disabled plugin: my-plugin
Plugin will be inactive in the next session
```

**Error Cases**:
```javascript
"Plugin not found"
"Plugin '<name>' is not installed"
"Plugin '<name>' is already disabled"
"Deactivation script failed: <error>" // Warning only
```

**Example Usage**:
```bash
# Temporarily disable a plugin
claude plugin disable debug-tools
```

**Note**: Plugin files remain on disk and can be re-enabled without re-downloading.

---

#### 5. `claude plugin validate <path>`

**Purpose**: Validate a plugin or marketplace manifest file

**Syntax**:
```bash
claude plugin validate <path>
```

**Arguments**:
- `<path>` - Path to manifest file (`plugin.json` or `marketplace.json`)

**Implementation**: `pt1(manifestPath)` function

**Validation Checks**:

**Required Fields (Plugin)**:
- `name` - Alphanumeric + hyphens/underscores, valid pattern: `/^[a-zA-Z0-9][a-zA-Z0-9_-]*[a-zA-Z0-9]$/`
- `version` - Semantic versioning format: `^\d+\.\d+\.\d+$` (e.g., "1.0.0")
- `description` - Non-empty string

**Optional Fields (Warnings if Missing)**:
- `author` - String or object with `{name, email}`
- `repository` - URL or object with `{type, url}`
- `license` - SPDX identifier (e.g., "MIT", "Apache-2.0")

**Required Fields (Marketplace)**:
- `name` - Marketplace identifier
- `version` - Semantic versioning
- `description` - Description of marketplace
- `source` - Source configuration object
- `plugins` - Array of plugin definitions

**Telemetry**: None (local validation only)

**Exit Codes**:
- `0` - Validation passed (with or without warnings)
- `1` - Validation failed (errors found)
- `2` - Unexpected error during validation

**Success Output**:
```
Validating plugin manifest: /path/to/plugin.json

✓ Validation passed
```

**Success with Warnings**:
```
Validating plugin manifest: /path/to/plugin.json

⚠ Found 1 warning:
  ▸ author: Recommended field missing

✓ Validation passed with warnings
```

**Error Output**:
```
Validating plugin manifest: /path/to/plugin.json

✗ Found 2 errors:
  ▸ version: Must be valid semver (e.g., "1.0.0")
  ▸ name: Required field missing

✗ Validation failed
```

**Example Usage**:
```bash
# Validate plugin manifest
claude plugin validate ./my-plugin

# Validate marketplace manifest
claude plugin validate ./my-marketplace
```

**Common Validation Errors**:
```javascript
// Name format errors
"-my-plugin"      // ✗ Cannot start with hyphen
"my-plugin-"      // ✗ Cannot end with hyphen
"my plugin"       // ✗ No spaces allowed
"my_plugin_123"   // ✓ Valid

// Version format errors
"v1.0.0"          // ✗ No 'v' prefix
"1.0"             // ✗ Must be three parts
"1.0.0-beta"      // ✗ No pre-release tags
"1.0.0"           // ✓ Valid
```

---

#### 6. `claude plugin list`

**Purpose**: List installed plugins (and optionally available plugins from marketplaces)

**Syntax**:
```bash
claude plugin list [options]
```

**Options**:
- `--json` - Output machine-readable JSON
- `--available` - Include available plugins from marketplaces (requires `--json`)

**Examples**:
```bash
# Human-readable list
claude plugin list

# JSON output
claude plugin list --json

# Include available plugins from configured marketplaces
claude plugin list --json --available
```

---

#### 7. `claude plugin update <plugin>`

**Purpose**: Update an installed plugin to the latest version (restart required to apply)

**Syntax**:
```bash
claude plugin update <plugin> [options]
```

**Options**:
- `-s, --scope <scope>` - Installation scope: user, project, or local (default: user)

---

### Marketplace Commands

#### 8. `claude plugin marketplace add <source>`

**Purpose**: Add a plugin marketplace from various sources

**Syntax**:
```bash
claude plugin marketplace add <source> [options]
```

**Arguments**:
- `<source>` - Marketplace source (required)

**Source Formats**:

1. **GitHub Repository** (owner/repo):
```bash
claude plugin marketplace add anthropics/claude-plugins
```

2. **Direct URL**:
```bash
claude plugin marketplace add https://example.com/marketplace.json
```

3. **Local Directory**:
```bash
claude plugin marketplace add ./local-marketplace
```

4. **Local File**:
```bash
claude plugin marketplace add /path/to/marketplace.json
```

5. **Git URL**:
```bash
claude plugin marketplace add git://github.com/org/repo.git
```

**Implementation**: `Ii(source, progressCallback)` function

**Source Parsing**: `dt1(source)` function determines source type

**Source Type Detection Logic**:
```javascript
// dt1() function logic:
if (/^[a-zA-Z0-9-]+\/[a-zA-Z0-9-]+$/.test(source)) {
    return { source: "github", repo: source };
}
if (/^https?:\/\//.test(source)) {
    return { source: "url", url: source };
}
if (/^git:\/\//.test(source)) {
    return { source: "git", url: source };
}
if (fs.existsSync(source)) {
    const stats = fs.statSync(source);
    return stats.isDirectory() 
        ? { source: "directory", path: source }
        : { source: "file", path: source };
}
return null; // Invalid format
```

**Internal Flow**:
```
1. Parse source format with dt1(source)
2. Validate source format is recognized
3. Fetch marketplace manifest from source
4. Validate manifest with pt1(path)
5. Store marketplace configuration
6. Update cache with VF()
7. Add to `~/.claude/plugins/known_marketplaces.json` (or `~/.claude/cowork_plugins/known_marketplaces.json`)
8. Send telemetry: tengu_marketplace_added
```

**Telemetry Properties**:
```javascript
{
  source_type: "github" | "git" | "url" | "directory" | "file"
}
```

**Exit Codes**:
- `0` - Success
- `1` - Add failed

**Success Output**:
```
Adding marketplace...
✓ Successfully added marketplace: my-marketplace
```

**Error Cases**:
```javascript
"Invalid marketplace source format. Try: owner/repo, https://..., or ./path"
"Network error fetching manifest"
"Manifest validation failed"
"Marketplace already exists"
```

**Example Usage**:
```bash
# Add official marketplace
claude plugin marketplace add anthropics/claude-plugins

# Add from URL
claude plugin marketplace add https://plugins.mycompany.com/marketplace.json

# Add local development marketplace
claude plugin marketplace add ~/my-plugins
```

---

#### 9. `claude plugin marketplace list`

**Purpose**: List all configured marketplaces

**Syntax**:
```bash
claude plugin marketplace list
```

**Arguments**: None

**Implementation**: `marketplaceListHandler`

**Internal Flow**:
```
1. Read `~/.claude/plugins/known_marketplaces.json` (or cowork_plugins)
2. Load marketplace metadata
3. Format output with source details
```

**Telemetry**: None (read-only operation)

**Exit Codes**:
- `0` - Success

**Success Output**:
```
Configured marketplaces:

  ▸ official
    Source: GitHub (anthropics/claude-plugins)

  ▸ community
    Source: URL (https://community-plugins.dev/marketplace.json)

  ▸ local-dev
    Source: Directory (/Users/dev/my-plugins)
```

**Empty Output**:
```
No marketplaces configured
```

**Example Usage**:
```bash
# List all marketplaces
claude plugin marketplace list
```

---

#### 10. `claude plugin marketplace remove <name>`

**Purpose**: Remove a configured marketplace

**Syntax**:
```bash
claude plugin marketplace remove <name>
```

**Arguments**:
- `<name>` - Marketplace name (required)

**Aliases**:
```bash
claude plugin marketplace rm <name>
```

**Implementation**: `marketplaceRemoveHandler`

**Internal Flow**:
```
1. Check if marketplace exists in `known_marketplaces.json`
2. Remove marketplace configuration
3. Update cache with VF()
4. Update `~/.claude/plugins/known_marketplaces.json` (or cowork_plugins)
5. Send telemetry: tengu_marketplace_removed
6. NOTE: Does NOT uninstall plugins from marketplace
```

**Telemetry Properties**:
```javascript
{
  marketplace_name: "<name>"  // Anonymized
}
```

**Exit Codes**:
- `0` - Success
- `1` - Remove failed

**Success Output**:
```
✓ Successfully removed marketplace: my-marketplace
```

**Error Cases**:
```javascript
"Marketplace '<name>' not found"
```

**Important Note**: Removing a marketplace does **NOT** uninstall plugins from that marketplace. Plugins remain installed but will not receive updates.

**Example Usage**:
```bash
# Remove marketplace
claude plugin marketplace remove old-marketplace

# Using alias
claude plugin marketplace rm test-marketplace
```

---

#### 11. `claude plugin marketplace update [name]`

**Purpose**: Update marketplace(s) from their source

**Syntax**:
```bash
claude plugin marketplace update [name]
```

**Arguments**:
- `[name]` - Marketplace name (optional)
  - If provided: Updates specific marketplace
  - If omitted: Updates ALL marketplaces

**Implementation Functions**:
- `bg1(name, progressCallback)` - Update single marketplace
- `q5B()` - Update all marketplaces

**Internal Flow (Single)**:
```
1. Fetch latest manifest from source
2. Compare with cached version
3. Update plugin listings
4. Report changes (new/updated/removed plugins)
5. Update cache with VF()
6. Send telemetry: tengu_marketplace_updated
```

**Internal Flow (All)**:
```
1. Get list of all marketplaces with pY()
2. Update each marketplace in parallel
3. Report aggregate results
4. Send telemetry: tengu_marketplace_updated_all
```

**Telemetry Events**:
- Single: `tengu_marketplace_updated` with `{marketplace_name}`
- All: `tengu_marketplace_updated_all` with `{count}`

**Exit Codes**:
- `0` - Success
- `1` - Update failed

**Success Output (Single)**:
```
Updating marketplace: my-marketplace...
✓ Successfully updated marketplace: my-marketplace
```

**Success Output (All)**:
```
Updating 3 marketplace(s)...
✓ Successfully updated 3 marketplace(s)
```

**Error Cases**:
```javascript
"Marketplace '<name>' not found"
"Network error fetching manifest"
"No marketplaces configured" // When updating all with none configured
```

**Example Usage**:
```bash
# Update specific marketplace
claude plugin marketplace update official

# Update all marketplaces
claude plugin marketplace update
```

---

## Internal Flows

### Marketplace Resolution Flow

```
Plugin Install Request: "my-plugin@marketplace@1.2.0"
           │
           ▼
┌──────────────────────────────┐
│ Parse Identifier             │
│ ~~~~~~~~~~~~~~~~~~~~~~~~~~   │
│ name: "my-plugin"            │
│ marketplace: "marketplace"   │
│ version: "1.2.0"             │
└──────────┬───────────────────┘
           │
           ▼
     Has marketplace?
           ├─ YES ──┐
           │        │
           │        ▼
           │  ┌──────────────────────────┐
           │  │ Search Specific          │
           │  │ Marketplace Only         │
           │  └──────────┬───────────────┘
           │             │
           │             ▼
           │       Found in marketplace?
           │             ├─ YES ─┐
           │             │       │
           │             NO      │
           │             │       │
           │             ▼       │
           │        ERROR: Not   │
           │        found        │
           │                     │
           NO                    │
           │                     │
           ▼                     │
    ┌──────────────────────┐    │
    │ Search ALL           │    │
    │ Marketplaces         │    │
    │ (pY() loads list)    │    │
    └──────────┬───────────┘    │
               │                 │
               ▼                 │
      For each marketplace       │
      Check if plugin exists     │
               │                 │
               ├─ Found ─────────┤
               │                 │
               NO                │
               │                 │
               ▼                 │
          Next marketplace       │
               │                 │
               ├─ None left      │
               │                 │
               ▼                 │
          ERROR: Not found       │
                                 │
                                 ▼
                           ┌─────────────────┐
                           │ Plugin Found!   │
                           │ Return metadata │
                           └─────────────────┘
```

### Cache Update Flow (VF)

```
Marketplace Operation Complete
           │
           ▼
┌──────────────────────────────┐
│ VF() - Update Cache          │
├──────────────────────────────┤
│ 1. Read known_marketplaces.json │
│ 2. Update timestamp          │
│ 3. Rebuild plugin index      │
│ 4. Write back to disk        │
└──────────┬───────────────────┘
           │
           ▼
      Cache Updated
```

**Called After**:
- `plugin marketplace add`
- `plugin marketplace remove`
- `plugin marketplace update`

**NOT Called After**:
- Plugin operations (plugins don't cache)

### Error Handling Flow

```
Error Occurs
     │
     ▼
┌────────────────────────────────┐
│ Q(error, action)               │
│ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~   │
│ Centralized error handler      │
│ Search: function Q(W, J)       │
└────────┬───────────────────────┘
         │
         ├─ 1. Log error to telemetry
         │    └─ Y1(error, Xc)
         │
         ├─ 2. Format error message
         │    └─ console.error(`${z0.cross} Failed to ${action}: ${error.message}`)
         │
         └─ 3. Exit with code 1
              └─ process.exit(1)
```

**Error Handler Implementation**:
```javascript
function Q(W, J) {
    // W = error, J = action description
    Y1(W instanceof Error ? W : Error(String(W)), Xc);
    console.error(
        `${z0.cross} Failed to ${J}: ${W instanceof Error ? W.message : String(W)}`
    );
    process.exit(1);
}
```

**Icons Used**:
- `z0.cross` = `✗` (error)
- `z0.tick` = `✓` (success)
- `z0.pointer` = `▸` (list item)
- `z0.warning` = `⚠` (warning)

---

## File Structure & Naming

### Directory Structure

```
~/.claude/
├── settings.json                 # Main settings (user scope)
│   └── Contains:
│       • enabledPlugins: Record<string, boolean>
│       • extraKnownMarketplaces: Record<string, MarketplaceDefinition> (optional)
│       • telemetryEnabled: boolean (if supported by your build)
│
└── plugins/                      # Plugin cache directory (or cowork_plugins/)
    └── Contains:
        • known_marketplaces.json    # Configured marketplaces (sources + metadata)
        • installed_plugins.json     # Installed plugins state
        • marketplaces/              # Downloaded marketplace manifests
        • cache/                     # Versioned plugin cache copies
        • npm-cache/                 # Optional npm cache for marketplace sources
```

### Plugin Manifest (`.claude-plugin/plugin.json`)

**Location**: `<plugin-root>/.claude-plugin/plugin.json` (installed plugins are cached under `~/.claude/plugins/` or `~/.claude/cowork_plugins/`)

**Common Fields (not exhaustive)**:
```json
{
  "name": "string",              // REQUIRED: Plugin identifier
  "version": "string",           // REQUIRED: Semver format "X.Y.Z"
  "description": "string",       // REQUIRED: Human-readable description
  
  "author": {                    // OPTIONAL: Author information
    "name": "string",
    "email": "string"
  },
  
  "repository": {                // OPTIONAL: Source repository
    "type": "git",
    "url": "string"
  },
  
  "license": "string",           // OPTIONAL: SPDX identifier
  
  "activate": "string",          // OPTIONAL: Path to activation script
  "deactivate": "string",        // OPTIONAL: Path to deactivation script
  
  "dependencies": {              // OPTIONAL: NPM dependencies
    "package-name": "version"
  },
  
  "peerDependencies": {          // OPTIONAL: Required peer deps
    "@anthropic-ai/claude-agent-sdk": "version"
  },
  
  "permissions": {               // OPTIONAL: Requested permissions
    "filesystem": ["read", "write"],
    "network": ["https://api.example.com"],
    "environment": ["API_KEY", "DATABASE_URL"]
  },

  "lspServers": {},              // OPTIONAL: LSP server configs (see lsp.md)
  
  "config": {                    // OPTIONAL: Plugin-specific config
    "key": "value"
  }
}
```

**Example**:
```json
{
  "name": "prettier-formatter",
  "version": "2.1.0",
  "description": "Code formatter using Prettier",
  "author": {
    "name": "Anthropic",
    "email": "support@anthropic.com"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/anthropics/claude-plugins.git"
  },
  "license": "MIT",
  "activate": "./scripts/activate.sh",
  "deactivate": "./scripts/deactivate.sh",
  "dependencies": {
    "prettier": "^3.0.0"
  },
  "peerDependencies": {
    "@anthropic-ai/claude-agent-sdk": "^0.1.22"
  },
  "permissions": {
    "filesystem": ["read", "write"],
    "network": []
  }
}
```

### Marketplace Manifest (`marketplace.json`)

**Location**: Source-dependent (GitHub, URL, local, etc.)

**Complete Schema**:
```json
{
  "name": "string",              // REQUIRED: Marketplace identifier
  "version": "string",           // REQUIRED: Semver format "X.Y.Z"
  "description": "string",       // REQUIRED: Marketplace description
  
  "source": {                    // REQUIRED: Source configuration
    "source": "github" | "git" | "url" | "directory" | "file",
    // For GitHub:
    "repo": "owner/repo",
    // For Git/URL:
    "url": "string",
    // For Directory/File:
    "path": "string"
  },
  
  "plugins": [                   // REQUIRED: Available plugins
    {
      "name": "string",
      "version": "string",
      "description": "string",
      "downloadUrl": "string",   // URL to plugin archive
      "manifestUrl": "string",   // URL to plugin.json
      "tags": ["string"],        // OPTIONAL: Search tags
      "homepage": "string",      // OPTIONAL: Plugin homepage
      "dependencies": {          // OPTIONAL: Plugin dependencies
        "other-plugin": "version"
      }
    }
  ],
  
  "updateFrequency": "daily" | "weekly" | "manual",  // OPTIONAL
  "autoUpdate": boolean,         // OPTIONAL: Auto-update plugins
  
  "metadata": {                  // OPTIONAL: Marketplace metadata
    "maintainer": "string",
    "homepage": "string",
    "supportUrl": "string"
  }
}
```

**Example**:
```json
{
  "name": "official",
  "version": "2.0.0",
  "description": "Official Claude plugin marketplace",
  "source": {
    "source": "github",
    "repo": "anthropics/claude-plugins"
  },
  "plugins": [
    {
      "name": "prettier-formatter",
      "version": "2.1.0",
      "description": "Code formatter using Prettier",
      "downloadUrl": "https://github.com/anthropics/claude-plugins/releases/download/v2.1.0/prettier-formatter.tar.gz",
      "manifestUrl": "https://github.com/anthropics/claude-plugins/blob/main/prettier-formatter/plugin.json",
      "tags": ["formatting", "javascript", "typescript"],
      "homepage": "https://github.com/anthropics/claude-plugins/tree/main/prettier-formatter"
    }
  ],
  "updateFrequency": "daily",
  "autoUpdate": false
}
```

### Settings (`~/.claude/settings.json`)

**Common Fields (not exhaustive)**:
```json
{
  "enabledPlugins": {
    "prettier-formatter@my-marketplace": true,
    "eslint-checker@my-marketplace": true,
    "old-formatter@my-marketplace": false
  },

  "extraKnownMarketplaces": {
    "my-marketplace": {
      "source": { "type": "github", "repo": "owner/repo" }
    }
  },

  "pluginConfig": {
    "prettier-formatter@my-marketplace": {
      "tabWidth": 2,
      "semi": true
    },
    "eslint-checker@my-marketplace": {
      "autoFix": false
    }
  },

  "telemetryEnabled": true
}
```

### Naming Conventions

#### Plugin Names

**Pattern**: `/^[a-zA-Z0-9][a-zA-Z0-9_-]*[a-zA-Z0-9]$/`

**Rules**:
- Must start with alphanumeric character
- Must end with alphanumeric character
- Can contain: letters, numbers, hyphens (`-`), underscores (`_`)
- Cannot contain spaces or special characters

**Examples**:
```javascript
// ✓ Valid names
"my-plugin"
"my_plugin"
"plugin123"
"My-Cool_Plugin-2"

// ✗ Invalid names
"-my-plugin"          // Cannot start with hyphen
"my-plugin-"          // Cannot end with hyphen
"my plugin"           // No spaces
"my@plugin"           // No special chars
"_plugin"             // Cannot start with underscore
```

#### Version Numbers

**Pattern**: `/^\d+\.\d+\.\d+$/`

**Format**: Semantic Versioning (MAJOR.MINOR.PATCH)

**Rules**:
- Must be three numbers separated by dots
- No prefix (no "v")
- No pre-release tags (no "-beta", "-alpha")
- No build metadata (no "+20130313144700")

**Examples**:
```javascript
// ✓ Valid versions
"1.0.0"
"2.3.4"
"10.20.30"

// ✗ Invalid versions
"v1.0.0"              // No prefix
"1.0"                 // Must be three parts
"1.0.0-beta"          // No pre-release tags
"1.0.0+build"         // No build metadata
```

---

## Data Structures

### TypeScript Interfaces

#### Plugin Definition

```typescript
interface Plugin {
    // Required fields
    name: string;
    version: string;        // Format: "X.Y.Z"
    description: string;
    
    // Optional metadata
    author?: string | {
        name: string;
        email?: string;
    };
    repository?: string | {
        type: string;
        url: string;
    };
    license?: string;       // SPDX identifier
    
    // Lifecycle hooks
    activate?: string;      // Path to activation script
    deactivate?: string;    // Path to deactivation script
    
    // Dependencies
    dependencies?: Record<string, string>;
    peerDependencies?: Record<string, string>;
    
    // Security
    permissions?: {
        filesystem?: string[];    // ["read", "write"]
        network?: string[];       // ["https://api.example.com"]
        environment?: string[];   // ["API_KEY", "DATABASE_URL"]
    };
    
    // Plugin-specific configuration
    config?: Record<string, any>;
}
```

#### Marketplace Definition

```typescript
interface Marketplace {
    // Required fields
    name: string;
    version: string;
    description: string;
    source: MarketplaceSource;
    plugins: PluginDefinition[];
    
    // Optional settings
    updateFrequency?: "daily" | "weekly" | "manual";
    autoUpdate?: boolean;
    
    // Optional metadata
    metadata?: {
        maintainer?: string;
        homepage?: string;
        supportUrl?: string;
    };
}

type MarketplaceSource =
    | { source: "github"; repo: string }
    | { source: "git"; url: string }
    | { source: "url"; url: string }
    | { source: "directory"; path: string }
    | { source: "file"; path: string };

interface PluginDefinition {
    name: string;
    version: string;
    description: string;
    downloadUrl: string;
    manifestUrl: string;
    tags?: string[];
    homepage?: string;
    dependencies?: Record<string, string>;
}
```

#### Validation Result

```typescript
interface ValidationResult {
    success: boolean;
    fileType: "plugin" | "marketplace";
    filePath: string;
    
    errors: Array<{
        path: string;        // e.g., "plugins[0].name"
        message: string;     // Human-readable error
    }>;
    
    warnings: Array<{
        path: string;
        message: string;
    }>;
}
```

#### Settings

```typescript
interface Settings {
    enabledPlugins?: Record<string, boolean>; // plugin@marketplace → true/false
    extraKnownMarketplaces?: Record<string, any>;
    pluginConfig?: Record<string, any>;       // per-plugin config (keyed by plugin@marketplace)
    telemetryEnabled?: boolean;

    // MCP project-shared server approvals (when applicable)
    enabledMcpjsonServers?: string[];
    disabledMcpjsonServers?: string[];
    enableAllProjectMcpServers?: boolean;
}
```

---

## Implementation Details

### Core Functions Reference

| Function | Purpose | Location | Parameters |
|----------|---------|----------|------------|
| `_SQ(name)` | Install plugin | Line 3730+ | `name: string` |
| `kSQ(name)` | Uninstall plugin | Line 3730+ | `name: string` |
| `xSQ(name)` | Enable plugin | Line 3730+ | `name: string` |
| `vSQ(name)` | Disable plugin | Line 3730+ | `name: string` |
| `pt1(path)` | Validate manifest | Line 3725+ | `path: string` |
| `Ii(source, cb)` | Add marketplace | Line 3728+ | `source: ParsedSource, cb: ProgressCallback` |
| `vg1(name)` | Remove marketplace | Line 3729+ | `name: string` |
| `bg1(name, cb)` | Update marketplace | Line 3729+ | `name: string, cb: ProgressCallback` |
| `pY()` | List marketplaces | Line 3729+ | None |
| `q5B()` | Update all marketplaces | Line 3729+ | None |
| `dt1(source)` | Parse source format | Line 3728+ | `source: string` |
| `VF()` | Update cache | Line 3728+ | None |
| `v4()` | Read local config | ~3725 | None |
| `q0()` | Read user config | ~3725 | None |
| `LG(config)` | Write local config | ~3725 | `config: UserConfig` |
| `Q(err, action)` | Error handler | Line 3725 | `err: Error, action: string` |
| `Z1(event, data)` | Telemetry reporter | ~3725 | `event: string, data: object` |

### Validation Patterns

#### Name Validation

```javascript
const NAME_PATTERN = /^[a-zA-Z0-9][a-zA-Z0-9_-]*[a-zA-Z0-9]$/;

function validateName(name) {
    if (!NAME_PATTERN.test(name)) {
        return {
            valid: false,
            error: "name: Must start and end with alphanumeric characters " +
                   "and contain only letters, numbers, hyphens, and underscores"
        };
    }
    return { valid: true };
}
```

#### Version Validation

```javascript
const VERSION_PATTERN = /^\d+\.\d+\.\d+$/;

function validateVersion(version) {
    if (!VERSION_PATTERN.test(version)) {
        return {
            valid: false,
            error: "version: Must be valid semver (e.g., '1.0.0')"
        };
    }
    return { valid: true };
}
```

### Telemetry Events

All commands send telemetry events if `telemetryEnabled: true` in config.

**Plugin Events**:
```javascript
"tengu_plugin_install_command"      // { plugin: string }
"tengu_plugin_uninstall_command"    // { plugin: string }
"tengu_plugin_enable_command"       // { plugin: string }
"tengu_plugin_disable_command"      // { plugin: string }
```

**Marketplace Events**:
```javascript
"tengu_marketplace_added"           // { source_type: string }
"tengu_marketplace_removed"         // { marketplace_name: string }
"tengu_marketplace_updated"         // { marketplace_name: string }
"tengu_marketplace_updated_all"     // { count: number }
```

**Telemetry Implementation**:
```javascript
function Z1(event, data) {
    if (!config.telemetryEnabled) return;
    
    // Anonymize data
    const anonymized = anonymizeData(data);
    
    // Send to analytics endpoint
    sendAnalytics(event, anonymized);
}
```

### Configuration Management Functions

#### Read Local Config (`v4`)

```javascript
function v4() {
    const localConfigPath = path.join(process.cwd(), '.claude', 'settings.json');
    if (fs.existsSync(localConfigPath)) {
        return JSON.parse(fs.readFileSync(localConfigPath, 'utf8'));
    }
    return {};
}
```

#### Read User Config (`q0`)

```javascript
function q0() {
    const userConfigPath = path.join(os.homedir(), '.claude', 'settings.json');
    if (fs.existsSync(userConfigPath)) {
        return JSON.parse(fs.readFileSync(userConfigPath, 'utf8'));
    }
    return getDefaultConfig();
}
```

#### Write Local Config (`LG`)

```javascript
function LG(config) {
    const configDir = path.join(process.cwd(), '.claude');
    if (!fs.existsSync(configDir)) {
        fs.mkdirSync(configDir, { recursive: true });
    }
    const configPath = path.join(configDir, 'settings.json');
    fs.writeFileSync(configPath, JSON.stringify(config, null, 2));
}
```

---

## Marketplace System

### Source Type Support

The marketplace system supports five source types, each with different characteristics:

| Source Type | Format | Use Case | Update Method |
|-------------|--------|----------|---------------|
| **GitHub** | `owner/repo` | Public GitHub repos | GitHub API fetch |
| **Git** | `git://url` | Git repositories | Git clone/pull |
| **URL** | `https://...` | HTTP-accessible files | HTTP GET |
| **Directory** | `./path` or `/path` | Local development | File system read |
| **File** | `path/to/file.json` | Single manifest file | File system read |

### Source Resolution Process

```
User Input: "anthropics/claude-plugins"
           │
           ▼
┌──────────────────────────────┐
│ dt1() Parse Source           │
└──────────┬───────────────────┘
           │
           ▼
    Match against patterns:
    ├─ /^[a-zA-Z0-9-]+\/[a-zA-Z0-9-]+$/ → GitHub
    ├─ /^https?:\/\//                   → URL
    ├─ /^git:\/\//                       → Git
    └─ fs.existsSync(source)
        ├─ isDirectory()                 → Directory
        └─ isFile()                      → File
           │
           ▼
    Return: { source: "github", repo: "anthropics/claude-plugins" }
```

### Marketplace Caching

**Cache Location**: `~/.claude/plugins/known_marketplaces.json` (or `~/.claude/cowork_plugins/known_marketplaces.json`)

**Cache Structure**:
```json
{
  "[marketplace-name]": {
    "name": "marketplace-name",
    "version": "1.0.0",
    "source": { ... },
    "plugins": [ ... ],
    "cachedAt": "2025-10-24T12:00:00Z",
    "lastUpdated": "2025-10-24T12:00:00Z"
  }
}
```

**Cache Update Triggers**:
1. `plugin marketplace add` - Initial cache creation
2. `plugin marketplace update` - Refresh from source
3. `plugin marketplace remove` - Remove from cache

**Cache Access**:
- Implemented by the marketplace handlers used by `claude plugin marketplace ...` commands.

### Plugin Resolution Algorithm

When installing `claude plugin install my-plugin@marketplace` (or `claude plugin install my-plugin`):

```
1. Parse identifier
   ├─ name: "my-plugin"
   └─ marketplace: "marketplace"  (optional)

2. Load marketplaces
   └─ Load known marketplaces + any repo-provided marketplaces

3. Search strategy:
   ├─ If marketplace specified:
   │  └─ Search ONLY that marketplace
   │     ├─ Found → return plugin
   │     └─ Not found → ERROR
   │
   └─ If marketplace NOT specified:
      └─ Search ALL marketplaces in order
         ├─ Found in first → return plugin
         ├─ Not found → try next
         └─ None found → ERROR

4. Version matching:
   └─ Version selection is determined by the marketplace entry you install.
```

### Marketplace Update Strategy

**Update Frequency Options**:
- `"daily"` - Check for updates once per day
- `"weekly"` - Check for updates once per week
- `"manual"` - Never auto-update (default)

**Update Process**:
1. Fetch latest manifest from source
2. Compare with cached version
3. Detect changes:
   - New plugins added
   - Plugins updated
   - Plugins removed
4. Update cache
5. Report changes to user

**Important**: Marketplace updates do NOT auto-update installed plugins. You must manually reinstall plugins to get new versions.

---

## Troubleshooting Guide

### Common Issues

#### Issue: "Plugin not found in any marketplace"

**Cause**: Plugin name doesn't match any marketplace entry

**Debug Steps**:
```bash
# 1. Check marketplace is configured
claude plugin marketplace list

# 2. Update marketplace
claude plugin marketplace update

# 3. Try with marketplace qualifier
claude plugin install plugin-name@marketplace-name
```

**Common Reasons**:
- Typo in plugin name
- Marketplace not configured
- Marketplace cache outdated
- Plugin not in any configured marketplace

---

#### Issue: "Manifest validation failed"

**Cause**: Plugin manifest has errors

**Debug Steps**:
```bash
# 1. Validate manifest locally
claude plugin validate ~/.claude/plugins/plugin-name/plugin.json

# 2. Check for common issues:
# - Name format: ^[a-zA-Z0-9][a-zA-Z0-9_-]*[a-zA-Z0-9]$
# - Version format: ^\d+\.\d+\.\d+$
# - Required fields: name, version, description

# 3. Common validation errors:
"-my-plugin"      # ✗ Cannot start with hyphen
"v1.0.0"          # ✗ No 'v' prefix in version
"1.0"             # ✗ Version must be three parts
```

**Fix Examples**:
```json
// ✗ Invalid
{
  "name": "-my-plugin",        // Starts with hyphen
  "version": "v1.0.0",         // Has 'v' prefix
  "description": ""            // Empty description
}

// ✓ Valid
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "A valid plugin"
}
```

---

#### Issue: "Activation script failed"

**Cause**: Plugin activation script encountered error

**Debug Steps**:
```bash
# 1. Check if script exists
ls -la ~/.claude/plugins/plugin-name/scripts/activate.sh

# 2. Check script permissions
chmod +x ~/.claude/plugins/plugin-name/scripts/activate.sh

# 3. Run script manually to see error
bash ~/.claude/plugins/plugin-name/scripts/activate.sh

# 4. Check script logs (if any)
cat ~/.claude/debug/latest
```

**Common Activation Script Issues**:
- Missing execute permission
- Missing dependencies
- Syntax errors in script
- Environment variables not set

---

#### Issue: "Plugin is already installed"

**Cause**: Plugin already exists, trying to reinstall

**Solutions**:
```bash
# Option 1: Uninstall first
claude plugin uninstall plugin-name
claude plugin install plugin-name

# Option 2: Update via marketplace
claude plugin marketplace update
claude plugin uninstall plugin-name
claude plugin install plugin-name
```

---

#### Issue: "Invalid marketplace source format"

**Cause**: Marketplace source doesn't match any recognized pattern

**Valid Formats**:
```bash
# GitHub (owner/repo)
claude plugin marketplace add anthropics/claude-plugins

# URL (must start with http:// or https://)
claude plugin marketplace add https://example.com/marketplace.json

# Git (must start with git://)
claude plugin marketplace add git://github.com/org/repo.git

# Local (must be valid path)
claude plugin marketplace add ./marketplace
claude plugin marketplace add /absolute/path/to/marketplace
```

**Invalid Examples**:
```bash
# ✗ Missing protocol
claude plugin marketplace add example.com/marketplace.json

# ✗ Malformed GitHub
claude plugin marketplace add anthropics-claude-plugins

# ✗ Relative path that doesn't exist
claude plugin marketplace add ../nonexistent
```

---

#### Issue: Plugin enabled but not working

**Cause**: Plugin changes require session restart

**Solution**:
```bash
# 1. Enable plugin
claude plugin enable my-plugin

# 2. Restart Claude CLI session
# Exit current session and start new one

# 3. Verify plugin is loaded
# Check if plugin functionality is available
```

**Why**: Plugins are loaded during session initialization, not dynamically.

---

#### Issue: "Permission denied" errors

**Cause**: Insufficient permissions to access plugin directories

**Debug Steps**:
```bash
# 1. Check plugin directory permissions
ls -la ~/.claude/plugins/

# 2. Fix permissions if needed
chmod -R u+rw ~/.claude/plugins/

# 3. Check if running as correct user
whoami

# 4. Verify ownership
ls -l ~/.claude/
```

**Common Permission Issues**:
- Directory not writable
- Files owned by different user
- Parent directory lacks execute permission

---

### Advanced Debugging

#### Enable Debug Logging

```bash
# Set environment variable
export DEBUG=1

# Run command
claude plugin install my-plugin

# Check logs
cat ~/.claude/debug/latest
```

#### Check Configuration Files

```bash
# User settings
cat ~/.claude/settings.json

# Marketplace config
cat ~/.claude/plugins/known_marketplaces.json

# Installed plugin state
cat ~/.claude/plugins/installed_plugins.json

# Plugin manifests
find ~/.claude/plugins -path "*/.claude-plugin/plugin.json" -exec cat {} \;
```

#### Verify File Integrity

```bash
# List all plugin files
find ~/.claude/plugins -type f

# Check for missing manifests
for dir in ~/.claude/plugins/*/; do
  if [ ! -f "${dir}.claude-plugin/plugin.json" ]; then
    echo "Missing manifest: $dir"
  fi
done

# Check manifest validity
for manifest in ~/.claude/plugins/*/.claude-plugin/plugin.json; do
  echo "Validating: $manifest"
  claude plugin validate "$manifest"
done
```

---

## Advanced Usage

### Creating Custom Marketplace

**Step 1: Create marketplace manifest**

Create `marketplace.json`:
```json
{
  "name": "my-company-plugins",
  "description": "Internal company plugins",
  "version": "1.0.0",
  "source": {
    "source": "directory",
    "path": "/company/plugins"
  },
  "plugins": [
    {
      "name": "company-formatter",
      "version": "1.0.0",
      "description": "Company code formatter",
      "downloadUrl": "https://internal.company.com/plugins/formatter.tar.gz",
      "manifestUrl": "https://internal.company.com/plugins/formatter/plugin.json",
      "tags": ["formatting", "company"],
      "homepage": "https://internal.company.com/plugins/formatter"
    }
  ],
  "updateFrequency": "daily",
  "autoUpdate": false
}
```

**Step 2: Validate marketplace manifest**

```bash
claude plugin validate ./marketplace.json
```

**Step 3: Add to Claude**

```bash
claude plugin marketplace add /company/plugins/marketplace.json
```

**Step 4: Install plugins**

```bash
claude plugin install company-formatter@my-company-plugins
```

---

### Plugin Development Workflow

**Step 1: Create plugin structure**

```bash
mkdir my-plugin
cd my-plugin

# Create manifest
cat > plugin.json << 'EOF'
{
  "name": "my-plugin",
  "version": "0.1.0",
  "description": "My custom plugin",
  "author": "Your Name",
  "license": "MIT"
}
EOF

# Create source directory
mkdir src
echo "console.log('Plugin loaded');" > src/index.js
```

**Step 2: Create lifecycle scripts (optional)**

```bash
mkdir scripts

# Activation script
cat > scripts/activate.sh << 'EOF'
#!/bin/bash
echo "Activating my-plugin..."
# Setup tasks here
EOF

# Deactivation script
cat > scripts/deactivate.sh << 'EOF'
#!/bin/bash
echo "Deactivating my-plugin..."
# Cleanup tasks here
EOF

chmod +x scripts/*.sh
```

**Step 3: Validate manifest**

```bash
claude plugin validate ./plugin.json
```

**Step 4: Test locally**

```bash
# Create local development marketplace
mkdir -p ~/dev/marketplace

cat > ~/dev/marketplace/marketplace.json << 'EOF'
{
  "name": "dev",
  "version": "1.0.0",
  "description": "Development marketplace",
  "source": {
    "source": "directory",
    "path": "~/dev/marketplace"
  },
  "plugins": [
    {
      "name": "my-plugin",
      "version": "0.1.0",
      "description": "My custom plugin",
      "downloadUrl": "file:///absolute/path/to/my-plugin",
      "manifestUrl": "file:///absolute/path/to/my-plugin/plugin.json"
    }
  ]
}
EOF

# Add marketplace
claude plugin marketplace add ~/dev/marketplace/marketplace.json

# Install plugin
claude plugin install my-plugin@dev
```

---

### Batch Operations

**Install multiple plugins**

```bash
#!/bin/bash

PLUGINS=(
  "prettier-formatter"
  "eslint-checker"
  "git-helper"
)

for plugin in "${PLUGINS[@]}"; do
  echo "Installing $plugin..."
  claude plugin install "$plugin"
done
```

**Update all marketplaces and plugins**

```bash
#!/bin/bash

# Update all marketplaces
echo "Updating marketplaces..."
claude plugin marketplace update

# Get list of enabled plugin IDs
PLUGINS=$(claude plugin list --json | jq -r '.[] | select(.enabled==true) | .id')

# Update each plugin to the latest available version (restart required)
for plugin in $PLUGINS; do
  echo "Updating $plugin..."
  claude plugin update "$plugin"
done

echo "All plugins updated. Restart Claude Code to apply updates."
```

---

## Gotchas & Limitations

### Critical Gotchas

#### 1. **Enable/Disable Changes Require Restart**

**Problem**: Plugin enable/disable commands do NOT take effect immediately.

**Evidence**:
```javascript
// From source code
console.log("Plugin will be active in the next session");
```

**Solution**: Restart your Claude CLI session after enabling/disabling plugins.

**Why**: Plugins are loaded during session initialization, not dynamically.

---

#### 2. **Marketplace Removal Does NOT Uninstall Plugins**

**Problem**: Removing a marketplace leaves plugins installed but orphaned.

**Impact**: Plugins remain functional but won't receive updates.

**Solution**: Manually uninstall plugins before removing marketplace:
```bash
# Uninstall plugins first
claude plugin uninstall plugin-from-marketplace

# Then remove marketplace
claude plugin marketplace remove marketplace-name
```

---

#### 3. **No Automatic Dependency Resolution**

**Problem**: Plugin dependencies must be manually installed.

**Missing**: No equivalent to `npm install` dependency resolution.

**Solution**: Check plugin documentation for required dependencies and install them separately.

---

#### 4. **Manifest Validation is Strict**

**Problem**: Even minor manifest errors block installation.

**Common Failures**:
```javascript
"-my-plugin"      // ✗ Name starts with hyphen
"my-plugin-"      // ✗ Name ends with hyphen
"v1.0.0"          // ✗ Version has 'v' prefix
"1.0"             // ✗ Version missing patch number
```

**Solution**: Use `claude plugin validate` before publishing.

---

#### 5. **No Plugin Versioning in Config**

**Problem**: Config only stores plugin names, not versions.

**Config Structure**:
```json
{
  "enabledPlugins": ["plugin-name"]  // No version!
}
```

**Impact**: Can't have multiple versions of same plugin.

**Workaround**: Include version in plugin name: `"my-plugin-v2"`

---

#### 6. **Activation Scripts Run in Separate Process**

**Problem**: Activation scripts don't have access to Claude internals.

**Limitation**: Scripts are shell/Node scripts, not JavaScript hooks.

**Use scripts only for**:
- File system setup
- External service initialization
- Environment variable validation

**NOT for**:
- Modifying Claude behavior
- Accessing Claude APIs
- Runtime hooks

---

#### 7. **No Rollback on Failed Install**

**Problem**: Partial installations leave system in inconsistent state.

**Missing**: Transaction-like behavior.

**Impact**: Failed installs may require manual cleanup.

**Solution**: Always validate manifests before installing:
```bash
claude plugin validate .
```

---

#### 8. **Telemetry Cannot Be Disabled Per-Command**

**Problem**: Plugin commands always send telemetry if globally enabled.

**Solution**: Disable telemetry globally in `~/.claude/settings.json`:
```json
{
  "telemetryEnabled": false
}
```

---

### Limitations

#### 1. **No Plugin Search Command**

**Missing**: `claude plugin search <query>`

**Workaround**:
```bash
# Dump installed + available plugins (includes marketplace entries)
claude plugin list --json --available | jq '.available[] | {pluginId, description, marketplaceName, version}'
```

---

#### 2. **No Plugin Info Command**

**Missing**: `claude plugin info <plugin>`

**Workaround**: Read plugin manifest directly:
```bash
find ~/.claude/plugins -path "*/.claude-plugin/plugin.json" -maxdepth 6 -print -exec cat {} \;
```

---

#### 3. **No Dependency Graph Visualization**

**Missing**: Understanding which plugins depend on each other.

**Impact**: Uninstalling a plugin may break others.

---

#### 4. **No Plugin Conflict Detection**

**Problem**: Two plugins can override same functionality.

**Missing**: Pre-install conflict checking.

**Impact**: Last installed plugin wins.

---

#### 5. **No Plugin Sandboxing**

**Security Issue**: Plugins run with full Claude CLI permissions.

**Risk**: Malicious plugins can access filesystem, network, etc.

**Mitigation**: Only install plugins from trusted sources.

---

#### 6. **No Automatic Updates**

**Problem**: Plugins don't auto-update even when marketplace updates.

**Workaround**: Manual re-install:
```bash
claude plugin marketplace update
claude plugin uninstall old-plugin
claude plugin install old-plugin  # Gets new version
```

---

#### 7. **No Plugin Development Tools**

**Missing**:
- `claude plugin create` - Scaffold new plugin
- `claude plugin test` - Test plugin locally
- `claude plugin publish` - Publish to marketplace

**Impact**: Plugin development is manual process.

---

## Appendix

### Exit Codes

```typescript
enum ExitCode {
    Success = 0,           // Operation successful
    GeneralError = 1,      // General error
    ValidationError = 2,   // Validation failed (validate command only)
}
```

### Icon Reference

```javascript
// From source code (z0 object)
z0.tick = "✓"              // Success
z0.cross = "✗"             // Error
z0.pointer = "▸"           // List item
z0.warning = "⚠"           // Warning
z0.checkboxOn = "☑"        // Checked
z0.checkboxOff = "☐"       // Unchecked
```

### Version History

**Current Version**: 2.1.42

**CLI Version Check**:
```bash
claude --version
# Output: 2.1.42
```

### Related Documentation

- **CLI Architecture**: `architecture.md`
- **MCP Integration**: Shared system with plugins
- **Configuration Reference**: Part of this document
- **Skills System**: `skills-documentaion.md`

### License & Legal

```
Copyright © Anthropic PBC. All rights reserved.

Use is subject to the Legal Agreements outlined here:
https://docs.claude.com/en/docs/claude-code/legal-and-compliance
```

---

**END OF DOCUMENT**

*This documentation was extracted from actual source code analysis of `@anthropic-ai/claude-code` version 2.1.42 (`@anthropic-ai/claude-code/cli.js`).*

