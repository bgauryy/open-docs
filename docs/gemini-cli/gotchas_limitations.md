# Gemini CLI - Gotchas & Limitations Reference

**Generated:** 2024-10-24
**Purpose:** Comprehensive documentation of known issues, limitations, and workarounds

---

## Table of Contents

1. [Critical Gotchas](#critical-gotchas)
2. [Security Limitations](#security-limitations)
3. [Configuration Gotchas](#configuration-gotchas)
4. [Tool Limitations](#tool-limitations)
5. [MCP Gotchas](#mcp-gotchas)
6. [IDE Integration Limitations](#ide-integration-limitations)
7. [Sandbox Limitations](#sandbox-limitations)
8. [Performance Issues](#performance-issues)
9. [Platform-Specific Issues](#platform-specific-issues)
10. [Workarounds & Solutions](#workarounds--solutions)

---

## Critical Gotchas

### 1. No Workspace Boundary Enforcement

**Severity:** 🔴 CRITICAL

**Issue:** Tools can access entire filesystem, not just workspace

**Impact:**
- Can read `/etc/passwd`, system files
- Can write to any location (subject to OS permissions)
- Path traversal possible

**Mitigation:**
- Use sandbox execution (`tools.sandbox: true`)
- Enable folder trust (`security.folderTrust.enabled: true`)
- Run as non-root user

**Status:** By design (flexibility vs security tradeoff)

---

### 2. YOLO Mode Disables All Confirmations

**Severity:** 🔴 CRITICAL

**Issue:** `tools.autoAccept: true` bypasses ALL safety checks

**Impact:**
- No file diff previews
- No command confirmations
- Automatic tool execution
- Potential data loss

**Mitigation:**
- **Never use in production**
- Set `security.disableYoloMode: true` to prevent accidental enable
- Use `tools.allowed` for selective auto-approval

**Workaround:**
```yaml
security:
  disableYoloMode: true  # Cannot be overridden by flags

tools:
  allowed:               # Selective auto-approval
    - "read_file"
    - "glob"
```

---

### 3. Ignored Files Not Protected

**Severity:** 🟡 MEDIUM

**Issue:** `.gitignore` doesn't prevent `read_file` / `write_file` access

**Impact:**
- Can read `.env` files
- Can access secrets
- Can modify ignored files

**Behavior:**
- `.gitignore` only affects `glob`, `list_directory`, `search_file_content`
- `read_file`, `write_file` bypass ignore patterns

**Mitigation:**
- Use sandbox
- Store secrets outside workspace
- Use OS-level file permissions

---

## Security Limitations

### 1. No Built-in Authentication (A2A Server)

**Severity:** 🔴 CRITICAL (A2A only)

**Issue:** A2A HTTP API has no authentication

**Impact:**
- Anyone on localhost can execute commands
- No user identification
- No authorization checks

**Mitigation:**
- Deploy behind API Gateway with auth
- Use Cloud IAP (Identity-Aware Proxy)
- Implement custom auth middleware
- Only expose on private network

---

### 2. Secrets in Environment Variables

**Severity:** 🟡 MEDIUM

**Issue:** API keys passed as environment variables to sandbox

**Impact:**
- Visible in container inspection
- Stored in process environment
- Potentially logged

**Mitigation:**
- Use service account with key file (more secure than env var)
- Rotate keys regularly
- Use secret management services (GCP Secret Manager, etc.)

---

### 3. Symlink Following

**Severity:** 🟡 MEDIUM

**Issue:** File operations follow symlinks without validation

**Impact:**
- Could escape workspace via symlinks
- Access files outside intended directory

**Status:** No current protection

**Mitigation:** Use sandbox with restricted mounts

---

## Configuration Gotchas

### 1. Settings Requiring Restart

**Issue:** Many settings need restart to take effect

**Affected Settings:**
- All `experimental.*` settings
- `tools.sandbox`
- `general.vimMode`
- `tools.core`
- `mcpServers` (any changes)

**Workaround:** Restart Gemini CLI after changing these settings

**Indicator:** Setting has `requiresRestart: true` in schema

---

### 2. "Always Approve" is Session-Only

**Issue:** Users expect permanent auto-approval, but it's session-only

**Behavior:**
- Selecting "Always Approve" in confirmation dialog sets `ApprovalMode.AUTO_EDIT`
- Only lasts for current session
- Does NOT persist across restarts

**Permanent Solution:** Add to `tools.allowed` in config

```yaml
tools:
  allowed:
    - "write_file"
    - "replace"
```

---

### 3. Model Router Override

**Issue:** When `experimental.useModelRouter: true`, specified model may be ignored

**Impact:**
- Setting `model.name: "gemini-1.5-pro"` may still use `gemini-2.0-flash-exp` for simple queries
- Unexpected model usage
- Different pricing

**Workaround:** Disable model router for consistent model usage

```yaml
experimental:
  useModelRouter: false  # Force use of model.name
```

---

### 4. IDE Trust Precedence

**Issue:** IDE trust overrides local trust file

**Behavior:**
- If IDE integration enabled (`GEMINI_CLI_IDE_SERVER_PORT` set), IDE trust takes precedence
- Local `trustedFolders.json` ignored
- User cannot override IDE trust decision

**Workaround:** Trust workspace in IDE, not via `/permissions` command

---

## Tool Limitations

### 1. `replace` Tool Exact Match Requirement

**Severity:** 🟡 MEDIUM

**Issue:** `old_string` must match file content EXACTLY (including whitespace)

**Impact:**
- Edits fail if whitespace differs
- Indentation changes break replacement
- AI must predict exact file content

**Solution:** Use `smart_edit` tool (default)

```yaml
useSmartEdit: true  # Default, uses LLM to correct mismatches
```

**Tradeoff:** `smart_edit` is slower (additional LLM calls)

---

### 2. `write_todos` Tool Not Integrated

**Issue:** TODO list not prominently displayed in UI

**Impact:**
- TODOs sent to LLM but user doesn't see them easily
- Must check tool call results

**Status:** Experimental, `useWriteTodos: false` by default

**Workaround:** Enable and check conversation history

```yaml
useWriteTodos: true
```

---

### 3. Tool Output Truncation

**Issue:** Large tool outputs truncated automatically

**Default Threshold:** 50,000 characters

**Behavior:**
- Only first and last `truncateToolOutputLines` (default: 100) lines kept
- Middle content discarded
- May lose important information

**Configuration:**

```yaml
tools:
  truncateToolOutputThreshold: 100000  # Increase limit
  truncateToolOutputLines: 200        # Keep more lines
  # OR
  truncateToolOutputThreshold: -1     # Disable truncation
```

**Gotcha:** Disabling truncation can cause API quota errors if output is huge

---

## MCP Gotchas

### 1. All-or-Nothing Trust

**Issue:** Cannot selectively trust tools from MCP server

**Behavior:**
- `trust: true` → All tools from server auto-approved
- `trust: false` → All tools require confirmation
- No per-tool trust

**Workaround:** Use `includeTools` / `excludeTools`

```yaml
mcpServers:
  server-name:
    command: "node server.js"
    trust: false
    includeTools:        # Only these tools
      - "safe_tool_1"
      - "safe_tool_2"
```

---

### 2. MCP OAuth Requires Browser

**Issue:** `/mcp auth` opens browser for OAuth flow

**Impact:**
- Doesn't work in headless environments
- Requires user interaction
- Difficult to automate

**Workaround:**
- Use SSH tunneling with port forwarding
- Pre-authenticate and store tokens
- Use API key auth instead (if supported by server)

---

### 3. MCP Server Restart Clears State

**Issue:** `/mcp refresh` restarts servers, losing session state

**Impact:**
- OAuth tokens may be lost
- Server-side state cleared
- Active connections dropped

**Behavior:** By design, but can be surprising

---

## IDE Integration Limitations

### 1. VSCode Extension Preview Status

**Issue:** Extension marked as preview (`"preview": true`)

**Impact:**
- APIs may change
- Features may be incomplete
- Breaking changes possible

**Recommendation:** Test thoroughly, pin extension version

---

### 2. Diff Race Condition

**Severity:** 🟡 MEDIUM

**Issue:** File may be modified on disk while diff is open

**Status:** Known issue (TODO in code)

**Impact:**
- User edits in IDE
- Gemini CLI modifies same file
- Conflicts or lost changes

**Workaround:** Avoid editing files externally while diff is open

---

### 3. IDE Server No Authentication

**Issue:** IDE server on localhost has no auth token

**Impact:**
- Any process on localhost can connect
- Local privilege escalation possible

**Status:** Future improvement needed

**Mitigation:** Only affects local processes, limited risk

---

## Sandbox Limitations

### 1. Nested Sandboxes Prevented

**Issue:** Cannot run Gemini CLI in sandbox inside another sandbox

**Detection:** `SANDBOX` environment variable checked

**Impact:** Breaks sandbox in certain deployment scenarios

**Workaround:** None, by design

---

### 2. Virtual Environment Remapping

**Issue:** If `VIRTUAL_ENV` under workdir, remapped to `.gemini-cli/sandbox.venv`

**Impact:**
- Original venv NOT mounted (prevents host binary execution)
- Must recreate venv inside sandbox
- Additional setup time

**Workaround:** Use `.gemini-cli/sandbox.bashrc` to recreate venv

```bash
# .gemini-cli/sandbox.bashrc
if [ -d "$VIRTUAL_ENV" ] && [ ! -f "$VIRTUAL_ENV/bin/python" ]; then
  python3 -m venv "$VIRTUAL_ENV"
  source "$VIRTUAL_ENV/bin/activate"
  pip install -r requirements.txt
fi
```

---

### 3. Platform-Specific Sandbox Support

**Issue:** Sandbox type depends on platform

**macOS:**
- `sandbox-exec` (Seatbelt) - Built-in
- Docker/Podman - Must install

**Linux:**
- Docker/Podman only
- No built-in sandbox

**Windows:**
- Docker/Podman only
- Docker Desktop required

**Gotcha:** Auto-detection may not pick optimal sandbox

**Explicit Configuration:**

```yaml
tools:
  sandbox: "docker"  # Force Docker
```

---

### 4. File Permissions on Linux

**Issue:** Rootful Docker without userns-remap has permission issues

**Impact:**
- Files created in container owned by root
- Cannot edit from host

**Detection:** Auto-detected on Debian/Ubuntu

**Behavior:**
- Creates user with matching UID/GID inside container
- Can be forced with `SANDBOX_SET_UID_GID=1`

**Gotcha:** May not work on all Linux distros

---

## Performance Issues

### 1. Large Tool Outputs

**Issue:** Returning large tool outputs (>100KB) to LLM

**Impact:**
- Increased token usage
- Slower API responses
- Higher costs
- May hit API limits

**Mitigation:**
- Use pagination (`charLength: 10000`)
- Use `matchString` mode for targeted reads
- Enable tool output truncation

---

### 2. Ripgrep vs Fallback

**Issue:** Fallback search is 10-100x slower than ripgrep

**Behavior:**
- If `tools.useRipgrep: false`, uses Node.js-based search
- Much slower on large codebases

**Recommendation:** Always use ripgrep (default)

```yaml
tools:
  useRipgrep: true  # Default, keep enabled
```

---

### 3. Chat Compression Overhead

**Issue:** Compressing chat history makes additional API call

**Impact:**
- Extra latency
- Additional token usage (for compression prompt)
- May lose context

**Configuration:**

```yaml
model:
  chatCompression:
    enabled: true
    strategy: "auto"     # Or "manual"
    threshold: 80000     # Compress when > 80K tokens
```

**Gotcha:** Compression itself uses tokens

---

## Platform-Specific Issues

### 1. Windows Path Handling

**Issue:** Windows paths converted in sandbox: `C:\path` → `/c/path`

**Impact:**
- May break tools expecting Windows paths
- Path separators differ

**Workaround:** Use forward slashes in paths when possible

---

### 2. macOS Seatbelt Limitations

**Issue:** macOS Seatbelt profiles are complex and restrictive

**Profiles:**
- `permissive-open` - Most permissive, network allowed
- `restrictive-closed` - Most restrictive, no network

**Gotcha:** Wrong profile may block legitimate operations

**Configuration:**

```bash
export SEATBELT_PROFILE=permissive-open  # Default
```

---

### 3. Linux Kernel Version

**Issue:** Some sandbox features require Linux kernel 5.0+

**Impact:**
- Older kernels may not support all isolation features
- Security reduced

**Check:**

```bash
uname -r  # Check kernel version
```

---

## Workarounds & Solutions

### Workaround Matrix

| Issue | Severity | Workaround | Status |
|-------|----------|------------|--------|
| No workspace boundary | 🔴 CRITICAL | Use sandbox | Permanent |
| YOLO mode dangerous | 🔴 CRITICAL | `security.disableYoloMode` | Permanent |
| Ignored files accessible | 🟡 MEDIUM | OS permissions | Partial |
| Settings need restart | 🟢 LOW | Restart CLI | Permanent |
| Model router override | 🟡 MEDIUM | `useModelRouter: false` | Workaround |
| Replace tool strict | 🟡 MEDIUM | Use `smart_edit` | Solved |
| MCP all-or-nothing trust | 🟡 MEDIUM | `includeTools` filter | Workaround |
| Diff race condition | 🟡 MEDIUM | Avoid concurrent edits | TODO to fix |
| Virtual env remapping | 🟢 LOW | `sandbox.bashrc` | Workaround |
| Large tool outputs | 🟡 MEDIUM | Pagination | Workaround |

---

### Quick Fixes

#### Prevent Dangerous Operations

```yaml
# Disable YOLO mode permanently
security:
  disableYoloMode: true

# Enable folder trust
security:
  folderTrust:
    enabled: true

# Enable sandbox
tools:
  sandbox: true
```

---

#### Optimize Performance

```yaml
# Use ripgrep
tools:
  useRipgrep: true

# Enable tool output truncation
tools:
  enableToolOutputTruncation: true
  truncateToolOutputThreshold: 50000
  truncateToolOutputLines: 100
```

---

#### Selective Trust

```yaml
# Auto-approve safe tools
tools:
  allowed:
    - "read_file"
    - "glob"
    - "search_file_content"

# Trust specific MCP server
mcpServers:
  trusted-server:
    command: "node server.js"
    trust: true
    includeTools:
      - "safe_tool"
```

---

## Summary

**Total Gotchas:** 30+ documented

**Critical Issues:** 3 (workspace boundary, YOLO mode, A2A auth)

**Medium Issues:** 15+ (configuration, tools, MCP, IDE)

**Low Issues:** 10+ (performance, platform-specific)

**Status:**
- ✅ **Solved:** 5 issues (smart_edit, etc.)
- 🔧 **Workaround Available:** 20+ issues
- ⚠️ **Known Limitation:** 5+ issues
- 🚧 **TODO:** 3 issues (diff race, etc.)

This comprehensive gotchas reference documents all known issues, limitations, and workarounds in Gemini CLI.

