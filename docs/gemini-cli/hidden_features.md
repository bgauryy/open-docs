# Gemini CLI - Undocumented Features Reference

**Generated:** 2024-10-24
**Purpose:** Documentation of undocumented features, experimental flags, TODOs, and deprecated functionality

**Note:** This document has been cleaned to include ONLY features that are NOT documented in official Gemini CLI documentation. Many features previously listed have been removed as they are officially documented in `docs/get-started/configuration.md` and other official docs.

---

## Table of Contents

1. [Undocumented Experimental Features](#undocumented-experimental-features)
2. [Undocumented Settings](#undocumented-settings)
3. [Environment Variable Flags](#environment-variable-flags)
4. [Deprecated Features](#deprecated-features)
5. [Development TODOs](#development-todos)
6. [Alpha/Beta Features](#alphabeta-features)
7. [Known Limitations](#known-limitations)

---

## Undocumented Experimental Features

### Extension Management

**Setting:** `experimental.extensionManagement`

**Type:** `boolean`
**Default:** `true`
**Requires Restart:** Yes
**Show in Dialog:** No
**Status:** EXPERIMENTAL (UNDOCUMENTED)

**Description:** Enable extension management features

**Configuration:**

```yaml
experimental:
  extensionManagement: true
```

**What it does:**
- Enables `/extensions` command
- Allows installing/updating/removing extensions
- Manages extension lifecycle

**Gotcha:** Despite being experimental and enabled by default, this is a core feature. Disabling it breaks extension functionality.

**Code Location:** `packages/cli/src/config/settingsSchema.ts:1070-1077`

**Documentation Status:** ❌ NOT documented in official docs

---

### Codebase Investigator

**Setting:** `experimental.codebaseInvestigatorSettings`

**Type:** `object`
**Default:** See below
**Requires Restart:** Yes
**Show in Dialog:** Partially
**Status:** EXPERIMENTAL (UNDOCUMENTED)

**Description:** Configuration for the Codebase Investigator agent - a specialized agent for deep codebase analysis

**Code Location:** `packages/cli/src/config/settingsSchema.ts:1088-1098+`

**Documentation Status:** ❌ NOT documented in official docs

---

#### Codebase Investigator - Enable

**Setting:** `experimental.codebaseInvestigatorSettings.enabled`

**Type:** `boolean`
**Default:** `false`
**Show in Dialog:** Yes

**Configuration:**

```yaml
experimental:
  codebaseInvestigatorSettings:
    enabled: true
```

---

#### Codebase Investigator - Max Turns

**Setting:** `experimental.codebaseInvestigatorSettings.maxNumTurns`

**Type:** `number`
**Default:** `15`
**Show in Dialog:** Yes

**Description:** Maximum number of turns (tool calls) for the Codebase Investigator agent

**Configuration:**

```yaml
experimental:
  codebaseInvestigatorSettings:
    maxNumTurns: 20      # Allow more investigation steps
```

---

#### Codebase Investigator - Max Time

**Setting:** `experimental.codebaseInvestigatorSettings.maxTimeMinutes`

**Type:** `number`
**Default:** `5` minutes
**Show in Dialog:** No

**Description:** Maximum time for the Codebase Investigator agent (in minutes)

**Configuration:**

```yaml
experimental:
  codebaseInvestigatorSettings:
    maxTimeMinutes: 10     # Allow longer investigations
```

---

#### Codebase Investigator - Thinking Budget

**Setting:** `experimental.codebaseInvestigatorSettings.thinkingBudget`

**Type:** `number`
**Default:** `-1` (unlimited)
**Show in Dialog:** No

**Description:** The thinking budget for the Codebase Investigator agent (extended thinking tokens)

**Configuration:**

```yaml
experimental:
  codebaseInvestigatorSettings:
    thinkingBudget: 10000    # Limit thinking tokens
```

**Note:** This refers to Gemini's "thinking" feature where the model performs internal reasoning before responding.

---

#### Codebase Investigator - Model

**Setting:** `experimental.codebaseInvestigatorSettings.model`

**Type:** `string`
**Default:** `DEFAULT_GEMINI_MODEL` (`gemini-2.5-pro`)
**Show in Dialog:** No

**Description:** The model to use for the Codebase Investigator agent

**Configuration:**

```yaml
experimental:
  codebaseInvestigatorSettings:
    model: "gemini-1.5-pro"    # Use more capable model
```

---

### Codebase Investigator - Usage

**How to activate:**

1. Enable the feature:
   ```yaml
   experimental:
     codebaseInvestigatorSettings:
       enabled: true
   ```

2. Use investigative queries:
   ```
   Investigate how authentication works across this codebase
   Trace the data flow from API request to database
   Find all error handling patterns in this project
   ```

**What makes it different from regular chat:**
- Dedicated investigation loop (up to 15 turns)
- Time-limited execution (5 minutes by default)
- Specialized for code analysis
- May use different model than main conversation

---

## Undocumented Settings

### 1. `general.retryFetchErrors`

**Type:** `boolean`
**Default:** `false`
**Status:** UNDOCUMENTED

**Description:** Retry on "exception TypeError: fetch failed sending request" errors

**Configuration:**

```yaml
general:
  retryFetchErrors: true
```

**Purpose:** Workaround for transient network errors

**Gotcha:** This is a workaround for specific error types, not a general retry mechanism

**Documentation Status:** ❌ NOT documented in official docs

---

### 2. `extensions.workspacesWithMigrationNudge`

**Type:** `array`
**Default:** `[]`
**Status:** UNDOCUMENTED
**Internal:** Yes

**Description:** List of workspaces for which the migration nudge has been shown

**Purpose:** Internal tracking for extension migration prompts

**Documentation Status:** ❌ NOT documented in official docs

---

## Environment Variable Flags

**Note:** None of these environment variables are documented in the official Gemini CLI documentation, despite many being functional and useful.

### 1. `GEMINI_SYSTEM_MD`

**Purpose:** Override system prompt

**Values:**
- `true` / `1` → Use `~/.gemini/system.md`
- `/path/to/file.md` → Use custom file
- `false` / `0` → Use built-in prompt (default)

**Example:**

```bash
export GEMINI_SYSTEM_MD=/path/to/custom-prompt.md
gemini
```

**Code Location:** `packages/core/src/core/prompts.ts:87`

**Status:** UNDOCUMENTED (but functional)

---

### 2. `GEMINI_WRITE_SYSTEM_MD`

**Purpose:** Write generated system prompt to file

**Values:**
- `true` / `1` → Write to `~/.gemini/system.md`
- `/path/to/file.md` → Write to custom path

**Example:**

```bash
export GEMINI_WRITE_SYSTEM_MD=true
gemini
```

**Use case:** Debug system prompt composition

**Code Location:** `packages/core/src/core/prompts.ts:278`

**Status:** UNDOCUMENTED

---

### 3. `BUILD_SANDBOX`

**Purpose:** Build sandbox image before running

**Values:** `true` / `1`

**Requirements:**
- Must use linked binary (not installed)
- Only works from gemini-cli repo

**Example:**

```bash
export BUILD_SANDBOX=1
gemini
```

**Status:** Development-only, UNDOCUMENTED

---

### 4. `GEMINI_CLI_SKIP_UPDATE_CHECK`

**Purpose:** Skip update check on startup

**Values:** `true` / `1`

**Example:**

```bash
export GEMINI_CLI_SKIP_UPDATE_CHECK=1
gemini
```

**Status:** UNDOCUMENTED

---

### 5. `GEMINI_CLI_NO_TELEMETRY`

**Purpose:** Disable telemetry

**Values:** `true` / `1`

**Example:**

```bash
export GEMINI_CLI_NO_TELEMETRY=1
gemini
```

**Alternative:** Use `privacy.usageStatisticsEnabled: false` in settings (documented)

**Status:** UNDOCUMENTED

---

### 6. `GEMINI_CLI_CONFIG_PATH`

**Purpose:** Override config file path

**Values:** Path to config file

**Example:**

```bash
export GEMINI_CLI_CONFIG_PATH=/path/to/custom/config.yaml
gemini
```

**Status:** UNDOCUMENTED

---

### 7. `GEMINI_CLI_TRUSTED_FOLDERS_PATH`

**Purpose:** Override trusted folders file path

**Values:** Path to trusted folders JSON

**Example:**

```bash
export GEMINI_CLI_TRUSTED_FOLDERS_PATH=/path/to/trustedFolders.json
gemini
```

**Status:** UNDOCUMENTED (but referenced in code)

---

### 8. `DEBUG` or `DEBUG_MODE`

**Purpose:** Enable debug mode

**Values:** `true` / `1`

**Example:**

```bash
export DEBUG=1
gemini
```

**Effect:**
- Verbose logging
- MCP stderr logging
- Debug inspector in sandbox

**Status:** Partially documented

---

### 9. `GEMINI_CLI_DEBUG`

**Purpose:** Gemini CLI-specific debug flag

**Values:** `true` / `1`

**Status:** UNDOCUMENTED

---

### 10. `GEMINI_CLI_IDE_PORT`

**Purpose:** Port for IDE companion server

**Default:** Auto-assigned

**Example:**

```bash
export GEMINI_CLI_IDE_PORT=3000
gemini --ide
```

**Status:** UNDOCUMENTED

---

### 11. `GEMINI_CLI_INTEGRATION_TEST`

**Purpose:** Mark as integration test run

**Values:** `"true"`

**Effect:**
- Changes container naming strategy
- Skips certain initialization steps

**Status:** Internal testing only, UNDOCUMENTED

---

### 12. `GEMINI_CLI_TEST_VAR`

**Purpose:** Test variable passed to sandbox

**Status:** Testing only, UNDOCUMENTED

---

## Deprecated Features

### 1. `extensions.disabled` Setting

**Status:** DEPRECATED (migrated automatically)

**Old Configuration:**

```yaml
extensions:
  disabled:
    - "unwanted-extension"
```

**Migration:** Automatically migrated to extension manager on startup

**Location:** `packages/cli/src/config/settings.ts:750-770`

**Migration Logic:**

```typescript
function migrateDeprecatedSettings(loadedSettings, extensionManager) {
  const settings = loadedSettings.forScope(scope).settings;

  if (settings.extensions?.disabled) {
    // Migrate to extension manager
    for (const extension of settings.extensions.disabled) {
      extensionManager.disableExtension(extension, scope);
    }

    // Remove deprecated setting
    settings.extensions.disabled = undefined;
  }
}
```

**New Approach:** Extensions are now managed via extension manager, not settings file.

**How to disable extensions now:**

```bash
/extensions disable <extension-name>
```

---

### 2. `suppressNotification` Field (IDE Protocol)

**Status:** DEPRECATED

**Location:** `packages/core/src/ide/types.ts:145-147`

**Old Schema:**

```typescript
interface IdeNotification {
  filePath: string;

  /**
   * @deprecated
   */
  suppressNotification?: boolean;
}
```

**Replacement:** Not documented. Field is ignored.

---

### 3. `updateHistoryItem()` Hook

**Status:** DEPRECATED

**Location:** `packages/cli/src/ui/hooks/useHistoryManager.ts:73-75`

**Deprecation Notice:**

```typescript
/**
 * @deprecated Prefer not to update history item directly as we are currently
 * rendering all history items in <Static /> for performance reasons. Only use
 * if ABSOLUTELY NECESSARY
 */
```

**Reason:** History items are rendered in `<Static />` for performance. Updating causes re-renders.

**Alternative:** Avoid updating history items after creation.

---

### 4. Legacy `mcp.serverCommand` Setting

**Status:** DEPRECATED (use `mcpServers` instead)

**Old Configuration:**

```yaml
mcp:
  serverCommand: "node mcp-server.js"
```

**New Configuration:**

```yaml
mcpServers:
  server-name:
    command: "node"
    args: ["mcp-server.js"]
```

---

## Development TODOs

### High Priority

#### 1. IDE Diff Race Condition

**Location:** `packages/core/src/tools/edit.ts:299-300`, `smart-edit.ts:660-661`

**Issue:**

```typescript
// TODO(chrstn): See https://github.com/google-gemini/gemini-cli/pull/5618#discussion_r2255413084
// for info on a possible race condition where the file is modified on disk while being edited.
```

**Description:** File may be modified on disk while user is editing in IDE diff viewer, causing conflicts.

**Impact:** MEDIUM - Could lead to lost changes or conflicts

---

### Medium Priority

#### 2. Configurable Exclusion Patterns

**Location:** `packages/core/src/tools/read-many-files.ts:102`

**Issue:**

```typescript
/**
 * TODO(adh): Consider making this configurable or extendable through a command line argument.
 */
function getDefaultExcludes(config?: Config): string[] {
  // Hardcoded exclusion patterns
}
```

**Description:** Exclusion patterns for `read_many_files` are hardcoded. Should be configurable.

**Impact:** LOW - Workaround exists (use `.geminiignore`)

---

### Low Priority

(Many other TODOs found in 66+ files - see full file list in previous search results)

---

## Alpha/Beta Features

### Zed Integration (Alpha)

**Location:** `packages/cli/src/zed-integration/acp.ts`

**Status:** ALPHA

**Description:** Integration with Zed editor via ACP (Agent Context Protocol)

**Keywords:** `experimental`, `alpha`, `preview`

**Configuration:** Not documented

---

### A2A Server (Beta)

**Location:** `packages/a2a-server/`

**Status:** BETA (per README)

**Description:** Agent-to-Agent communication server

**Keywords:** `experimental`, `beta`

**See:** `a2a_agent_system.md` for details

---

### VSCode IDE Companion (Beta)

**Location:** `packages/vscode-ide-companion/`

**Status:** BETA (per package.json: `"preview": true`)

**Description:** VSCode extension for IDE integration

**Keywords:** `preview`, `beta`

**See:** `ide_integration.md` for details

---

## Known Limitations

### 1. Codebase Investigator Incomplete

**Issue:** Feature is experimental and may not be fully functional

**Effect:** Unpredictable behavior, limited testing

**Recommendation:** Use with caution on production codebases

---

### 2. Extension Manager Experimental

**Issue:** Despite being enabled by default, extension management is marked experimental

**Effect:** API may change, extensions may break

**Recommendation:** Pin extension versions

---

### 3. IDE Diff Race Condition

**Issue:** File may be modified on disk while editing in IDE

**Effect:** Potential conflicts or lost changes

**Status:** Known bug, TODO to fix

---

## Officially Documented Features

**For documented features, please refer to:**
- Official Configuration Docs: `docs/get-started/configuration.md`
- Tools Documentation: `docs/tools/`
- Main Documentation Index: `docs/index.md`

---

**This document contains undocumented features not covered in official Gemini CLI documentation (as of October 2024).**
