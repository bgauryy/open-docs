# Gemini CLI - Security & Permissions Reference

**Generated:** 2024-10-24
**Purpose:** Complete documentation of security mechanisms, permissions, trust models, and sandbox execution

---

## Table of Contents

1. [Security Architecture Overview](#security-architecture-overview)
2. [Trust System](#trust-system)
3. [Sandbox Execution](#sandbox-execution)
4. [Tool Confirmation System](#tool-confirmation-system)
5. [File System Security](#file-system-security)
6. [Authentication & Authorization](#authentication--authorization)
7. [Security Environment Variables](#security-environment-variables)
8. [Approval Modes](#approval-modes)
9. [Security Best Practices](#security-best-practices)
10. [Known Security Gotchas](#known-security-gotchas)

---

## Security Architecture Overview

Gemini CLI implements a **multi-layered security model** with several independent but complementary systems:

```
┌───────────────────────────────────────────────────────────┐
│                    User Input / Request                     │
└─────────────────────────┬─────────────────────────────────┘
                          ↓
┌───────────────────────────────────────────────────────────┐
│               LAYER 1: Trust System                        │
│  - Folder Trust (workspace-level)                          │
│  - IDE Trust Integration                                   │
└─────────────────────────┬─────────────────────────────────┘
                          ↓
┌───────────────────────────────────────────────────────────┐
│          LAYER 2: Tool Confirmation                        │
│  - Auto-accept safe operations                             │
│  - User confirmation for write operations                  │
│  - Approval modes (YOLO / AUTO_EDIT / DEFAULT)            │
└─────────────────────────┬─────────────────────────────────┘
                          ↓
┌───────────────────────────────────────────────────────────┐
│            LAYER 3: Sandbox Execution                      │
│  - Docker/Podman/Seatbelt isolation                       │
│  - Network restrictions                                    │
│  - File system restrictions                                │
└─────────────────────────┬─────────────────────────────────┘
                          ↓
┌───────────────────────────────────────────────────────────┐
│        LAYER 4: File System Protection                     │
│  - Read/write permission checks                            │
│  - Directory traversal prevention                          │
└─────────────────────────┬─────────────────────────────────┘
                          ↓
                    Execution
```

---

## Trust System

### Overview

The **Folder Trust** system allows users to explicitly mark directories as trusted or untrusted.

**Status:** Experimental (disabled by default)

---

### Trust Levels

Defined in `packages/cli/src/config/trustedFolders.ts`:

```typescript
enum TrustLevel {
  TRUST_FOLDER = 'TRUST_FOLDER',       // Trust this exact folder
  TRUST_PARENT = 'TRUST_PARENT',       // Trust the parent folder
  DO_NOT_TRUST = 'DO_NOT_TRUST',       // Explicitly untrust
}
```

---

### Trust Sources

Gemini CLI checks trust from two sources (in order):

1. **IDE Trust** (if `ide.enabled` is true)
   - Provided by `ideContextStore.get()?.workspaceState?.isTrusted`
   - Used when running as IDE companion
   
2. **Local Trust File** (fallback)
   - File: `~/.gemini-cli/trustedFolders.json`
   - Environment variable: `GEMINI_CLI_TRUSTED_FOLDERS_PATH`

---

### Trust File Format

**Location:** `~/.gemini-cli/trustedFolders.json`

**Format:**

```json
{
  "/path/to/trusted/project": "TRUST_FOLDER",
  "/path/to/parent/folder": "TRUST_PARENT",
  "/path/to/untrusted/project": "DO_NOT_TRUST"
}
```

---

### Trust Resolution Algorithm

From `packages/cli/src/config/trustedFolders.ts:76-110`:

```typescript
isPathTrusted(location: string): boolean | undefined {
  const trustedPaths: string[] = [];
  const untrustedPaths: string[] = [];

  // Build lists
  for (const rule of this.rules) {
    switch (rule.trustLevel) {
      case TrustLevel.TRUST_FOLDER:
        trustedPaths.push(rule.path);
        break;
      case TrustLevel.TRUST_PARENT:
        trustedPaths.push(path.dirname(rule.path));
        break;
      case TrustLevel.DO_NOT_TRUST:
        untrustedPaths.push(rule.path);
        break;
    }
  }

  // Check trusted (uses isWithinRoot for hierarchical check)
  for (const trustedPath of trustedPaths) {
    if (isWithinRoot(location, trustedPath)) {
      return true;
    }
  }

  // Check untrusted (exact path match only)
  for (const untrustedPath of untrustedPaths) {
    if (path.normalize(location) === path.normalize(untrustedPath)) {
      return false;
    }
  }

  return undefined;  // No explicit trust/untrust = prompt user
}
```

**Key Behavior:**
- `TRUST_FOLDER` and `TRUST_PARENT` use hierarchical matching (child folders inherit trust)
- `DO_NOT_TRUST` uses exact path matching only (does NOT apply to child folders)
- Returning `undefined` means "not decided" - user will be prompted

---

### Enabling Trust System

**Configuration:**

```yaml
# .gemini/config.yaml
security:
  folderTrust:
    enabled: true
```

**CLI Command:**

```bash
/permissions  # Opens trust management dialog
```

---

### Trust Gotchas

1. **Disabled by default**
   - Setting: `security.folderTrust.enabled` defaults to `false`
   - If disabled, all folders are treated as trusted

2. **IDE trust takes precedence**
   - If running as IDE companion, local trust file is ignored
   - IDE trust value is always used

3. **Untrusted path matching is exact**
   - `DO_NOT_TRUST` on `/project` does NOT untrust `/project/subfolder`
   - Must explicitly untrust each path

4. **File format is JSON, not YAML**
   - Uses `trustedFolders.json`, not `trustedFolders.yaml`
   - Supports JSON comments via `strip-json-comments`

---

## Sandbox Execution

### Overview

Gemini CLI can execute shell commands in isolated sandbox environments using:

1. **Docker** (container-based)
2. **Podman** (container-based)
3. **Seatbelt** (macOS-specific, Seatbelt sandbox)

---

### Sandbox Detection & Configuration

**Auto-detection** (from `packages/cli/src/config/sandboxConfig.ts:30-88`):

```typescript
function getSandboxCommand(sandbox?: boolean | string): SandboxConfig['command'] | '' {
  // 1. Check if already inside sandbox
  if (process.env['SANDBOX']) {
    return '';  // Don't nest sandboxes
  }

  // 2. Check environment variable (highest priority)
  const envVar = process.env['GEMINI_SANDBOX'];
  
  // 3. Explicit sandbox command
  if (typeof sandbox === 'string') {
    if (!['docker', 'podman', 'sandbox-exec'].includes(sandbox)) {
      throw error;
    }
    return sandbox;
  }

  // 4. Auto-detect (if sandbox === true)
  if (os.platform() === 'darwin' && commandExists.sync('sandbox-exec')) {
    return 'sandbox-exec';  // macOS default
  } else if (commandExists.sync('docker') && sandbox === true) {
    return 'docker';
  } else if (commandExists.sync('podman') && sandbox === true) {
    return 'podman';
  }

  return '';  // No sandbox
}
```

**Priority:**
1. `SANDBOX` env var set → Already in sandbox, don't nest
2. `GEMINI_SANDBOX` env var → Use specified command
3. CLI flag/setting → Use specified value
4. Auto-detect → macOS Seatbelt > Docker > Podman

---

### Sandbox Configuration

#### Settings

```yaml
tools:
  sandbox: true               # Auto-detect (docker/podman/sandbox-exec)
  # OR
  sandbox: "docker"           # Explicit Docker
  # OR
  sandbox: "podman"           # Explicit Podman
  # OR
  sandbox: "sandbox-exec"     # Explicit Seatbelt
  # OR
  sandbox: false              # Disable sandbox
```

---

#### Environment Variables

##### Core Variables

- **`SANDBOX`**
  - Set automatically inside sandbox to container/sandbox name
  - Used to detect nested sandbox execution
  - Example: `SANDBOX=gemini-cli-sandbox-0`

- **`GEMINI_SANDBOX`**
  - Set by user to choose sandbox type
  - Values: `"docker"`, `"podman"`, `"sandbox-exec"`, `"true"`, `"false"`

- **`GEMINI_SANDBOX_IMAGE`**
  - Override default sandbox image
  - Default from `package.json:config.sandboxImageUri`

---

##### Sandbox Customization

- **`SEATBELT_PROFILE`** (macOS only)
  - Seatbelt profile name
  - Default: `"permissive-open"`
  - Built-in profiles:
    - `permissive-open` - Permissive, network allowed
    - `permissive-closed` - Permissive, no network
    - `permissive-proxied` - Permissive, network via proxy
    - `restrictive-open` - Restrictive, network allowed
    - `restrictive-closed` - Restrictive, no network
    - `restrictive-proxied` - Restrictive, network via proxy

- **`SANDBOX_FLAGS`**
  - Additional flags for `docker run` / `podman run`
  - Example: `SANDBOX_FLAGS="--cpus 2 --memory 4g"`

- **`SANDBOX_ENV`**
  - Comma-separated `KEY=VALUE` pairs to pass into sandbox
  - Example: `SANDBOX_ENV="API_KEY=xxx,DEBUG=1"`

- **`SANDBOX_MOUNTS`**
  - Comma-separated mount specs: `from:to:opts`
  - Example: `SANDBOX_MOUNTS="/host/path:/container/path:ro"`

- **`SANDBOX_PORTS`**
  - Comma-separated port mappings
  - Example: `SANDBOX_PORTS="3000,8080"`

- **`SANDBOX_SET_UID_GID`**
  - Run sandbox with current user's UID/GID
  - Values: `"1"`, `"true"` (enable), `"0"`, `"false"` (disable)
  - Default: `true` on Debian/Ubuntu Linux, `false` otherwise

---

##### Proxy Support

- **`GEMINI_SANDBOX_PROXY_COMMAND`**
  - Command to run proxy for sandbox network access
  - Proxy runs in separate container on port 8877
  - Example: `GEMINI_SANDBOX_PROXY_COMMAND="squid -N"`

- **`HTTPS_PROXY` / `https_proxy`**
  - HTTPS proxy URL (passed to sandbox)
  
- **`HTTP_PROXY` / `http_proxy`**
  - HTTP proxy URL (passed to sandbox)
  
- **`NO_PROXY` / `no_proxy`**
  - Comma-separated no-proxy list

---

##### Development & Debugging

- **`BUILD_SANDBOX`**
  - Build sandbox image before running (requires linked binary)
  - Only works from gemini-cli repo
  - Example: `BUILD_SANDBOX=1`

- **`DEBUG`**
  - Enable Node.js inspector in sandbox
  - Exposes port 9229 for debugging

- **`DEBUG_PORT`**
  - Override debug port (default: 9229)

---

### Docker/Podman Sandbox

#### Container Configuration

From `packages/cli/src/utils/sandbox.ts:412-764`:

**Mounted Paths:**

1. **Working Directory**
   - Host: `process.cwd()`
   - Container: `/workdir` (or Windows-converted path)
   - Mode: Read-write

2. **User Settings**
   - Host: `~/.gemini-cli`
   - Container: `/home/node/.gemini-cli`
   - Mode: Read-write

3. **OS Temp Dir**
   - Host: `os.tmpdir()`
   - Container: Same path
   - Mode: Read-write

4. **Google Cloud Config** (if exists)
   - Host: `~/.config/gcloud`
   - Container: Same path
   - Mode: Read-only

5. **Service Account Key** (if `GOOGLE_APPLICATION_CREDENTIALS` set)
   - Host: Path from env var
   - Container: Same path
   - Mode: Read-only

6. **Virtual Environment** (if under workdir)
   - Host: `$VIRTUAL_ENV`
   - Container: Remapped to `.gemini-cli/sandbox.venv`
   - Mode: Read-write

---

#### Environment Variables Passed to Container

**Authentication:**
- `GEMINI_API_KEY`
- `GOOGLE_API_KEY`
- `GOOGLE_APPLICATION_CREDENTIALS`
- `GOOGLE_CLOUD_PROJECT`
- `GOOGLE_CLOUD_LOCATION`

**Model Configuration:**
- `GEMINI_MODEL`
- `GOOGLE_GENAI_USE_VERTEXAI`
- `GOOGLE_GENAI_USE_GCA`

**Terminal:**
- `TERM`
- `COLORTERM`

**IDE Integration:**
- `GEMINI_CLI_IDE_SERVER_PORT`
- `GEMINI_CLI_IDE_WORKSPACE_PATH`
- `TERM_PROGRAM`

**Other:**
- `NODE_OPTIONS`
- `SANDBOX` (set to container name)
- `HOME` (if forcing UID/GID)
- `VIRTUAL_ENV` (if applicable)

---

#### Network Configuration

**Default:** Host network access via `host.docker.internal`

**With Proxy:**
1. Create internal network: `gemini-cli-sandbox`
2. Create proxy network: `gemini-cli-sandbox-proxy`
3. Run proxy container on port 8877
4. Connect sandbox to internal network
5. Set `HTTPS_PROXY=http://gemini-cli-sandbox-proxy:8877`

---

#### User/Permission Handling

**Default:** Run as `node` user (UID 1000)

**Debian/Ubuntu Linux (auto-detected):**
- Run container as root
- Create user inside container matching host UID/GID
- Switch to that user before running Gemini CLI
- Prevents permission issues with mounted volumes

**Force UID/GID:**
```bash
export SANDBOX_SET_UID_GID=1
```

---

### macOS Seatbelt Sandbox

#### Configuration

**Seatbelt Profile:** `.gemini-cli/sandbox-macos-<profile>.sb`

**Built-in Profiles:** 6 profiles (in `packages/cli/src/utils/sandbox-macos-*.sb`)

**Profile Variables:**

From `packages/cli/src/utils/sandbox.ts:228-265`:

```bash
sandbox-exec \
  -D TARGET_DIR=/path/to/project \
  -D TMP_DIR=/tmp \
  -D HOME_DIR=/Users/user \
  -D CACHE_DIR=/Users/user/Library/Caches \
  -D INCLUDE_DIR_0=/path/to/include/dir/0 \
  -D INCLUDE_DIR_1=/path/to/include/dir/1 \
  -D INCLUDE_DIR_2=/dev/null \
  -D INCLUDE_DIR_3=/dev/null \
  -D INCLUDE_DIR_4=/dev/null \
  -f /path/to/profile.sb \
  sh -c "SANDBOX=sandbox-exec NODE_OPTIONS='...' gemini ..."
```

**Key Features:**
- Always passes 5 `INCLUDE_DIR_*` variables (defaults to `/dev/null` if unused)
- Includes directories from `context.includeDirectories` setting
- Filters out `TARGET_DIR` from include directories

---

#### Proxy Support

- Uses `GEMINI_SANDBOX_PROXY_COMMAND` to start proxy process
- Proxy runs as detached background process (not container)
- Sets `HTTPS_PROXY` / `HTTP_PROXY` environment variables
- Waits for proxy to be ready: `curl http://localhost:8877`

---

#### Limitations

- **Cannot use `BUILD_SANDBOX`** - Building sandbox images not supported
- **No custom Dockerfile** - Must use built-in profiles
- **macOS only** - Seatbelt is Apple-specific technology

---

### Sandbox Gotchas

1. **Nested Sandboxes Prevented**
   - If `SANDBOX` env var is set, sandbox is disabled
   - Cannot run Gemini CLI in sandbox inside another sandbox

2. **Image Must Exist**
   - Docker/Podman images are pulled if missing
   - For custom images, must pre-build or set `GEMINI_SANDBOX_IMAGE`

3. **Virtual Environment Remapping**
   - If `VIRTUAL_ENV` is under workdir, it's remapped to `.gemini-cli/sandbox.venv`
   - Original venv is NOT mounted (prevents host binary execution)
   - Must recreate venv inside sandbox using `.gemini-cli/sandbox.bashrc`

4. **File Permissions (Linux)**
   - Rootful Docker without userns-remap has permission issues
   - Auto-detected on Debian/Ubuntu: creates user with matching UID/GID
   - Override with `SANDBOX_SET_UID_GID=1` or `0`

5. **Network Isolation**
   - With proxy: sandbox uses internal network (no direct internet)
   - Without proxy: sandbox can access host network via `host.docker.internal`

6. **Windows Path Handling**
   - Windows paths are converted: `C:\path` → `/c/path`
   - May cause issues with some tools

---

## Tool Confirmation System

### Overview

Gemini CLI requires user confirmation for **potentially dangerous operations** (write, execute).

**Safe operations** (read-only) can be auto-approved.

---

### Approval Modes

Defined in `packages/core/src/tools/tool-invocations.ts`:

```typescript
enum ApprovalMode {
  DEFAULT = 'default',         // Ask for every operation (default mode)
  AUTO_EDIT = 'autoEdit',      // Auto-approve file edits only
  YOLO = 'yolo',               // Auto-approve everything (dangerous!)
}
```

---

### Default Behavior

**By Tool:**

| Tool | Requires Confirmation | Auto-Accept Condition |
|------|----------------------|----------------------|
| `read_file` | No | Always auto-approved |
| `search_file_content` | No | Always auto-approved |
| `list_directory` | No | Always auto-approved |
| `glob` | No | Always auto-approved |
| `write_file` | Yes | `ApprovalMode.AUTO_EDIT` |
| `replace` | Yes | `ApprovalMode.AUTO_EDIT` |
| `smart_edit` | Yes | `ApprovalMode.AUTO_EDIT` |
| `run_shell_command` | Yes | Never (always requires confirmation) |
| `execute_command` | Yes | Never |
| `run_code_action` | Yes | IDE confirmations |
| `list_code_actions` | No | Always auto-approved |
| `save_memory` | No | Always auto-approved |
| MCP tools | Varies | Based on server trust |

---

### Configuration

#### Settings

```yaml
tools:
  autoAccept: false           # Disable all confirmations (YOLO mode)
  allowed:                    # Tools that bypass confirmation
    - "read_file"
    - "glob"
    - "search_file_content"
```

#### Environment Variable

```bash
# Disable YOLO mode even if set by flag
security:
  disableYoloMode: true
```

---

### Confirmation Flow

From `packages/core/src/tools/tool-invocations.ts`:

```
Tool Call Request
    ↓
Check Approval Mode
    ↓
┌─────────────────────────────────┐
│ Mode = YOLO?                    │ → Yes → Auto-approve
└─────────────────────────────────┘
    ↓ No
┌─────────────────────────────────┐
│ Tool in allowed list?           │ → Yes → Auto-approve
└─────────────────────────────────┘
    ↓ No
┌─────────────────────────────────┐
│ Call shouldConfirmExecute()     │
│ Returns ToolCallConfirmationDetails?│
└─────────────────────────────────┘
    ↓ Yes (needs confirmation)
┌─────────────────────────────────┐
│ Show Confirmation Dialog        │
│ - Display file diff / command   │
│ - User approves / rejects /     │
│   modifies / "always approve"   │
└─────────────────────────────────┘
    ↓
Execute Tool
```

---

### Confirmation Types

#### Edit Confirmation

**Type:** `ToolEditConfirmationDetails`

**Fields:**
- `type: 'edit'`
- `title`: Dialog title
- `fileName`: Name of file
- `filePath`: Full path
- `fileDiff`: Unified diff format
- `originalContent`: Current file content
- `newContent`: Proposed content
- `ideConfirmation`: Optional IDE diff viewer integration

**User Options:**
- **Approve** → Execute once
- **Reject** → Cancel
- **Modify** → Edit proposed content, then approve
- **Always Approve** → Set `ApprovalMode.AUTO_EDIT`

---

#### Command Confirmation

**Type:** `ToolCommandConfirmationDetails`

**Fields:**
- `type: 'command'`
- `title`: Dialog title
- `command`: Shell command to execute
- `cwd`: Working directory
- `env`: Environment variables (optional)

**User Options:**
- **Approve** → Execute once
- **Reject** → Cancel
- **Modify** → Edit command, then approve

---

### IDE Integration

**IDE Diff Confirmation:**

From `packages/cli/src/ui/hooks/usePermissionsModifyTrust.ts`:

```typescript
const ideConfirmation = ideClient.openDiff(
  filePath,
  proposedContent
);

// Later, when user confirms:
const result = await ideConfirmation;
if (result.status === 'accepted' && result.content) {
  // Use IDE-modified content instead
  params.content = result.content;
}
```

**Behavior:**
- If IDE integration is enabled, opens diff in IDE (e.g., VSCode)
- User can edit in IDE before accepting
- Modified content is used instead of AI-proposed content

---

### Auto-Approval Settings

#### Via Configuration

```yaml
tools:
  autoAccept: true            # DANGER: Auto-approve everything
  allowed:
    - "write_file"            # Only auto-approve write_file
    - "replace"
```

---

#### Via CLI Flag

```bash
gemini --auto-accept          # Enable auto-accept
```

---

#### Via In-Session Action

**User selects "Always Approve" in confirmation dialog:**

```typescript
onConfirm: async (outcome: ToolConfirmationOutcome) => {
  if (outcome === ToolConfirmationOutcome.ProceedAlways) {
    this.config.setApprovalMode(ApprovalMode.AUTO_EDIT);
  }
}
```

**Effect:** All subsequent file edit operations are auto-approved (for current session)

---

### MCP Tool Confirmation

**Trust-Based:**

From `packages/core/src/tools/mcp-client.ts`:

```typescript
const mcpServerConfig = config.mcpServers[serverName];
if (mcpServerConfig?.trust === true) {
  // Auto-approve tools from this server
  return false;
}

// Otherwise, show confirmation
return {
  type: 'mcp_tool',
  serverName,
  toolName,
  params,
};
```

**Configuration:**

```yaml
mcpServers:
  trusted-server:
    command: "node server.js"
    trust: true                # Auto-approve all tools
  
  untrusted-server:
    command: "node other.js"
    trust: false               # Require confirmation
```

---

### Confirmation Gotchas

1. **YOLO Mode is Dangerous**
   - `tools.autoAccept: true` disables ALL confirmations
   - No diff preview, no ability to review
   - Can be disabled with `security.disableYoloMode: true`

2. **"Always Approve" is Session-Only**
   - Selecting "Always Approve" only applies to current session
   - Does NOT persist across restarts
   - Must use `tools.allowed` setting for permanent auto-approval

3. **IDE Confirmation May Timeout**
   - `ideConfirmation` is a promise that resolves when IDE closes diff
   - If user never closes diff, confirmation hangs indefinitely
   - No timeout mechanism

4. **MCP Trust is All-or-Nothing**
   - Cannot selectively trust specific tools from a server
   - Either trust entire server or none
   - Workaround: Use `includeTools` / `excludeTools`

---

## File System Security

### Permissions

#### Read Permissions

**All read tools check file/directory exists** but do NOT verify read permissions before attempting.

**Error handling:**
- `ENOENT` → File not found
- `EACCES` → Permission denied
- `EISDIR` → Expected file, got directory
- `ENOTDIR` → Expected directory, got file

---

#### Write Permissions

**`write_file` tool:**

From `packages/core/src/tools/write-file.ts:240-272`:

```typescript
try {
  const dirName = path.dirname(file_path);
  
  // Create parent directories if missing
  if (!fs.existsSync(dirName)) {
    fs.mkdirSync(dirName, { recursive: true });
  }

  await fileSystemService.writeTextFile(file_path, fileContent);

} catch (error) {
  if (isNodeError(error)) {
    if (error.code === 'EACCES') {
      return {
        error: {
          message: `Permission denied writing to file: ${file_path}`,
          type: ToolErrorType.PERMISSION_DENIED,
        },
      };
    } else if (error.code === 'ENOSPC') {
      return {
        error: {
          message: `No space left on device: ${file_path}`,
          type: ToolErrorType.NO_SPACE_LEFT,
        },
      };
    } else if (error.code === 'EISDIR') {
      return {
        error: {
          message: `Target is a directory, not a file: ${file_path}`,
          type: ToolErrorType.TARGET_IS_DIRECTORY,
        },
      };
    }
  }
}
```

**Error Types:**
- `PERMISSION_DENIED` (EACCES)
- `NO_SPACE_LEFT` (ENOSPC)
- `TARGET_IS_DIRECTORY` (EISDIR)
- `FILE_WRITE_FAILURE` (generic)

---

### Path Restrictions

#### No Explicit Path Validation

**Current Behavior:**
- Gemini CLI does NOT validate paths against workspace root
- Tools can access ANY path on file system (subject to OS permissions)
- No protection against directory traversal (e.g., `../../etc/passwd`)

---

#### Workspace-Relative Paths

**Display only:**

Most tools display paths relative to `config.getTargetDir()` for readability:

```typescript
const relativePath = makeRelative(
  absolutePath,
  config.getTargetDir()
);
```

**But execution uses absolute paths**, so relative display does not restrict access.

---

### File Filtering

#### .gitignore and .geminiignore

**Respected by:**
- `glob` tool
- `list_directory` tool (with `context.fileFiltering.respectGitIgnore`)
- `search_file_content` tool (via ripgrep `--ignore`)

**Configuration:**

```yaml
context:
  fileFiltering:
    respectGitIgnore: true        # Respect .gitignore
    respectGeminiIgnore: true     # Respect .geminiignore
```

**NOT respected by:**
- `read_file` (can read any file, even ignored)
- `write_file` (can write to ignored files)

---

### Security Gotchas

1. **No Workspace Boundary Enforcement**
   - Tools can read/write files ANYWHERE on system
   - Sandbox provides isolation, but outside sandbox there's no restriction
   - Trust system is optional and doesn't block file access

2. **Ignored Files Not Protected**
   - `.gitignore` only affects listing/search tools
   - `read_file` and `write_file` can still access ignored files
   - Secrets in `.env` files are NOT automatically protected

3. **Symlink Following**
   - File system operations follow symlinks
   - Could allow access outside workspace via symlink
   - No symlink validation or restriction

4. **Parent Directory Creation**
   - `write_file` automatically creates parent directories
   - Could create directories outside workspace
   - No validation of parent paths

---

## Authentication & Authorization

### API Authentication

**Three methods:**

1. **API Key**
   - Environment variable: `GEMINI_API_KEY` or `GOOGLE_API_KEY`
   - Setting: `security.auth.selectedType: "api_key"`

2. **OAuth**
   - Setting: `security.auth.selectedType: "oauth"`
   - Uses Google OAuth flow
   - Requires `security.auth.useExternal: true` for browser-based flow

3. **Service Account**
   - Environment variable: `GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json`
   - Setting: `security.auth.selectedType: "service_account"`

---

### Auth Type Enforcement

**Configuration:**

```yaml
security:
  auth:
    selectedType: "api_key"       # Current auth type
    enforcedType: "oauth"         # Required auth type (optional)
```

**Behavior:**

If `enforcedType` is set and doesn't match `selectedType`:
- User is prompted to re-authenticate
- Cannot use Gemini CLI until auth type matches

---

### MCP Server Authentication

**HTTP MCP Servers:**

```yaml
mcpServers:
  api-server:
    httpUrl: "https://api.example.com/mcp"
    headers:
      Authorization: "Bearer token"
    
    # OR OAuth
    oauth:
      enabled: true
      authorizationUrl: "https://auth.example.com/authorize"
      tokenUrl: "https://auth.example.com/token"
      scopes: ["read", "write"]
```

---

## Security Environment Variables

### Critical Security Variables

| Variable | Purpose | Security Impact |
|----------|---------|-----------------|
| `GEMINI_API_KEY` | Gemini API authentication | **HIGH** - Full API access |
| `GOOGLE_API_KEY` | Google API authentication | **HIGH** - Full API access |
| `GOOGLE_APPLICATION_CREDENTIALS` | Service account key path | **CRITICAL** - GCP access |
| `GEMINI_SANDBOX` | Sandbox execution | **MEDIUM** - Disable for faster but unsafe execution |
| `SANDBOX` | Inside sandbox indicator | **HIGH** - Bypassed in nested sandboxes |
| `GEMINI_CLI_TRUSTED_FOLDERS_PATH` | Trust file location | **MEDIUM** - Override trust decisions |
| `SEATBELT_PROFILE` | macOS sandbox profile | **MEDIUM** - Profile determines restrictions |
| `SANDBOX_FLAGS` | Additional Docker flags | **MEDIUM** - Could weaken isolation |

---

### Environment Variable Passing

**Automatically passed to sandbox:**

- `GEMINI_API_KEY`
- `GOOGLE_API_KEY`
- `GOOGLE_APPLICATION_CREDENTIALS`
- `GOOGLE_CLOUD_PROJECT`
- `TERM`, `COLORTERM`

**NOT passed:**

- User environment variables (unless in `SANDBOX_ENV`)
- `PATH` (filtered to only workdir-relative paths)
- `PYTHONPATH` (filtered)

---

## Approval Modes

### Mode Summary

| Mode | Description | Use Case |
|------|-------------|----------|
| `DEFAULT` (`default`) | Confirm every operation | Default, safest |
| `AUTO_EDIT` (`autoEdit`) | Auto-approve file edits | Trusted file operations |
| `YOLO` (`yolo`) | Auto-approve everything | Dangerous! |

---

### Setting Approval Mode

#### Via Setting

```yaml
tools:
  autoAccept: true        # Enables YOLO mode
```

---

#### Via CLI Flag

```bash
gemini --auto-accept      # Enables YOLO mode
```

---

#### In-Session

User selects "Always Approve" in confirmation dialog:
- Sets mode to `AUTO_EDIT` (for file edits only)
- Does NOT set mode to `YOLO` (still requires confirmation for shell commands)

---

## Security Best Practices

### For Users

1. **Enable Folder Trust**
   ```yaml
   security:
     folderTrust:
       enabled: true
   ```

2. **Use Sandbox**
   ```yaml
   tools:
     sandbox: true
   ```

3. **Review Confirmations**
   - Don't blindly approve
   - Check file diffs before accepting
   - Review shell commands carefully

4. **Limit MCP Server Trust**
   ```yaml
   mcpServers:
     server-name:
       trust: false        # Require confirmation
       includeTools:       # Whitelist specific tools
         - "safe_tool"
   ```

5. **Use Service Account (not API key)**
   - Service accounts have scoped permissions
   - Easier to rotate credentials
   - Better audit trail

---

### For Developers

1. **Implement `shouldConfirmExecute()`**
   - All write/execute tools must return confirmation details
   - Provide meaningful context (file diff, command preview)

2. **Use `ApprovalMode` Correctly**
   - Never auto-approve shell commands
   - File edits can be auto-approved in `AUTO_EDIT` mode
   - Read operations don't need confirmation

3. **Handle Permission Errors**
   - Catch `EACCES`, `ENOSPC`, `EISDIR`
   - Return specific error types (`ToolErrorType`)
   - Don't expose sensitive paths in error messages

4. **Validate Tool Parameters**
   - Use Zod schemas for validation
   - Sanitize file paths (prevent injection)
   - Validate command arguments

---

## Known Security Gotchas

### High Risk

1. **No Workspace Boundary**
   - Tools can access entire filesystem
   - Mitigation: Use sandbox

2. **YOLO Mode Disables All Checks**
   - `tools.autoAccept: true` is extremely dangerous
   - No diff preview, no confirmation
   - Mitigation: Set `security.disableYoloMode: true`

3. **Secrets in Environment**
   - API keys passed to sandbox as env vars
   - Visible in container inspection
   - Mitigation: Use service account with key file

4. **Symlink Following**
   - File operations follow symlinks
   - Could escape workspace via symlinks
   - Mitigation: Use sandbox with restricted mounts

---

### Medium Risk

1. **IDE Trust Precedence**
   - IDE trust overrides local trust file
   - User cannot override IDE trust decision
   - Mitigation: None (intended behavior)

2. **MCP Server All-or-Nothing Trust**
   - Cannot selectively trust tools from server
   - Mitigation: Use `includeTools` / `excludeTools`

3. **Parent Directory Creation**
   - `write_file` creates parent directories automatically
   - Could create unexpected directories
   - Mitigation: Review file paths in confirmations

4. **Ignored Files Not Protected**
   - `.gitignore` doesn't prevent `read_file` / `write_file`
   - Secrets in ignored files can be accessed
   - Mitigation: Use sandbox, trust system

---

### Low Risk

1. **Session-Only "Always Approve"**
   - User expectation: permanent auto-approval
   - Reality: session-only
   - Mitigation: Document clearly

2. **Virtual Env Remapping**
   - Sandbox remaps `VIRTUAL_ENV` to `.gemini-cli/sandbox.venv`
   - User must recreate venv inside sandbox
   - Mitigation: Document in sandbox setup

3. **Confirmation Timeout**
   - IDE confirmations can hang indefinitely
   - Mitigation: Add timeout mechanism (future)

---

This comprehensive security reference documents all permission models, trust mechanisms, sandbox execution, and security boundaries in Gemini CLI, including gotchas and best practices.

