# Codex CLI - Security & Sandboxing

## Table of Contents
- [Security Model Overview](#security-model-overview)
- [Filesystem Sandboxing](#filesystem-sandboxing)
- [Network Sandboxing](#network-sandboxing)
- [Approval Policies](#approval-policies)
- [Platform-Specific Sandboxing](#platform-specific-sandboxing)
- [Command Safety](#command-safety)

---

## Security Model Overview

### Defense in Depth

Codex CLI implements **multiple independent security layers**:

```
┌─────────────────────────────────────────────────────┐
│ Layer 1: Approval Policies                         │
│   User controls what agent can do autonomously     │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│ Layer 2: Command Safety Checks                     │
│   Allowlist/blocklist for command validation       │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│ Layer 3: Path Validation                           │
│   Ensures file access stays within boundaries      │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│ Layer 4: Platform Sandboxing                       │
│   OS-level isolation (Seatbelt/Landlock)           │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│ Layer 5: Resource Limits                           │
│   Timeout, output size, memory constraints         │
└─────────────────────────────────────────────────────┘
```

### Security Principles

1. **Least Privilege**: Agent starts with minimal permissions
2. **Explicit Escalation**: User must approve privileged operations
3. **Fail Secure**: Errors default to denying access
4. **Transparency**: All actions are logged and visible
5. **Isolation**: Commands run in restricted environments

---

## Filesystem Sandboxing

### Modes

#### 1. Read-Only

**Configuration**:
```yaml
sandbox_mode: read-only
```

**Permissions**:
- ✅ Read any file
- ❌ Write any file
- ❌ Delete files
- ❌ Create directories

**Use Case**: Maximum safety when exploring unfamiliar code

**Approval Required For**:
- Any write operation
- Any file modification
- Running commands that might write

#### 2. Workspace-Write

**Configuration**:
```yaml
sandbox_mode: workspace-write
```

**Permissions**:
- ✅ Read any file
- ✅ Write within workspace (`$CWD`)
- ✅ Write to `$TMPDIR`
- ✅ Write to `~/.codex/`
- ❌ Write outside workspace

**Use Case**: Default mode, balances safety and productivity

**Approval Required For**:
- Writing outside workspace
- Modifying system files
- Installing global packages

**Implementation**: `core/src/sandboxing/mod.rs`

```rust
pub enum SandboxMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
}

pub fn validate_write_path(
    path: &Path,
    mode: SandboxMode,
    workspace: &Path,
) -> Result<()> {
    match mode {
        SandboxMode::ReadOnly => {
            Err("Write operation not allowed in read-only mode")
        }
        SandboxMode::WorkspaceWrite => {
            // Check if path is within workspace
            let canonical = path.canonicalize()?;
            if !canonical.starts_with(workspace) && 
               !is_allowed_write_location(&canonical) {
                Err("Write outside workspace not allowed")
            } else {
                Ok(())
            }
        }
        SandboxMode::DangerFullAccess => Ok(()),
    }
}

fn is_allowed_write_location(path: &Path) -> bool {
    // Always allow: /tmp, ~/.codex
    path.starts_with("/tmp") ||
    path.starts_with(env::temp_dir()) ||
    path.starts_with(codex_home())
}
```

#### 3. Danger-Full-Access

**Configuration**:
```yaml
sandbox_mode: danger-full-access
```

**Permissions**:
- ✅ Read any file
- ✅ Write any file
- ✅ Delete files
- ✅ Modify system files

**Use Case**: When you fully trust the agent or need system-wide changes

**Warning**: Provides no filesystem protection

### Writable Roots

Additional directories can be marked as writable:

```yaml
writable_roots:
  - /var/www/html
  - /opt/myapp
```

---

## Network Sandboxing

### Modes

#### 1. Restricted (Default)

**Configuration**:
```yaml
network_access: restricted
```

**Behavior**:
- ❌ Network disabled by default for all commands
- ✅ LLM API calls always allowed
- ✅ Approval can enable network per-command

**Implementation**: Platform-specific

**macOS (Seatbelt)**:
```scheme
(deny network*)
(allow network* (remote ip "api.openai.com:443"))
```

**Linux (iptables)**:
```bash
# Block all outbound except OpenAI
iptables -A OUTPUT -d api.openai.com -j ACCEPT
iptables -A OUTPUT -j DROP
```

#### 2. Enabled

**Configuration**:
```yaml
network_access: enabled
```

**Behavior**:
- ✅ Network access allowed for all commands
- No restrictions

**Use Case**: When commands need network (npm install, curl, etc.)

### Approval for Network

In `restricted` mode, agent can request network access:

```rust
// Request escalated permissions
let result = execute_with_approval(
    "npm install express",
    ApprovalReason::NetworkRequired,
).await?;
```

User sees:
```
╭───────────────────────────────────────╮
│ Approval Required: Network Access    │
├───────────────────────────────────────┤
│ Command: npm install express          │
│ Reason: Package installation          │
│                                       │
│ Allow network access? (y/n)          │
╰───────────────────────────────────────╯
```

---

## Approval Policies

### Four Modes

| Mode | Auto-Allowed | Requires Approval | Use Case |
|------|--------------|-------------------|----------|
| **untrusted** | Safe read commands | Most operations | Exploring unknown code |
| **on-failure** | All (sandboxed) | Failed commands | Development workflow |
| **on-request** | All (sandboxed) | Explicit requests | Power user mode |
| **never** | Everything | Nothing | CI/CD, full automation |

### 1. Untrusted Mode

**Configuration**:
```bash
codex --approval-mode untrusted
```

**Behavior**:
- Allowlist of safe commands auto-approved
- Everything else requires approval

**Safe Commands**:
```rust
// core/src/command_safety/is_safe_command.rs
const SAFE_COMMANDS: &[&str] = &[
    "cat", "ls", "pwd", "echo", "whoami",
    "git status", "git log", "git diff",
    "npm list", "cargo --version",
    "rg", "grep", "find",
    // ... read-only operations
];
```

**Requires Approval**:
- File writes
- Package installs
- System modifications
- Network access

### 2. On-Failure Mode

**Configuration**:
```bash
codex --approval-mode on-failure
```

**Behavior**:
- Commands run in sandbox
- If sandbox blocks it, request approval to run unsandboxed

**Flow**:
```
1. Try command in sandbox
     │
     ├─→ Success → Done
     │
     └─→ Failure (sandbox violation)
          │
          ├─→ Request approval
          │
          ├─→ If approved → Run unsandboxed
          │
          └─→ If denied → Report error
```

**Example**:
```bash
# First attempt (sandboxed, fails)
$ npm install express
Error: Network access denied

# Request approval
╭──────────────────────────────────────╮
│ Command failed in sandbox            │
│ Retry with elevated permissions?    │
│ (y/n)                                │
╰──────────────────────────────────────╯

# If approved, runs unsandboxed
$ npm install express  # Works
```

### 3. On-Request Mode

**Configuration**:
```bash
codex --approval-mode on-request
```

**Behavior**:
- Agent explicitly requests escalation when needed
- User sets `with_escalated_permissions: true` in tool call

**Implementation**:
```rust
// Agent requests escalation
{
  "name": "shell",
  "arguments": {
    "command": ["sudo", "apt", "install", "nginx"],
    "with_escalated_permissions": true,
    "justification": "Need to install system package"
  }
}
```

**User Interface**:
```
╭──────────────────────────────────────╮
│ Escalated Permissions Requested     │
├──────────────────────────────────────┤
│ Command: sudo apt install nginx      │
│ Reason: Need to install system pkg   │
│                                      │
│ Allow? (y/n)                         │
╰──────────────────────────────────────╯
```

### 4. Never Mode

**Configuration**:
```bash
codex --approval-mode never
```

**Behavior**:
- No interactive prompts ever
- Agent must work within constraints
- Used for CI/CD pipelines

**Example Use**:
```yaml
# GitHub Actions
- name: Auto-fix issues
  run: |
    codex --approval-mode never \
          --approval-mode auto-edit \
          "fix all lint errors"
```

---

## Platform-Specific Sandboxing

### macOS: Seatbelt

**Implementation**: `core/src/seatbelt.rs`

**Policy File**: `core/src/seatbelt_base_policy.sbpl`

```scheme
(version 1)
(deny default)

; Allow reading most files
(allow file-read*)

; Restrict writing to specific locations
(allow file-write*
    (subpath "/tmp")
    (subpath (param "TMPDIR"))
    (subpath (param "WORKSPACE"))
    (subpath (param "HOME" "/.codex")))

; Block network by default
(deny network*)

; Allow OpenAI API
(allow network-outbound
    (remote tcp "api.openai.com:443"))

; Allow process operations
(allow process-exec
    (subpath "/bin")
    (subpath "/usr/bin"))
```

**Usage**:
```rust
pub fn wrap_with_seatbelt(
    command: &[String],
    workspace: &Path,
) -> Vec<String> {
    vec![
        "/usr/bin/sandbox-exec".to_string(),
        "-p".to_string(),
        generate_policy(workspace),
        command[0].clone(),
        // ... rest of command
    ]
}
```

**Features**:
- File access control
- Network isolation
- Process restrictions
- IPC limitations

### Linux: Landlock

**Implementation**: `core/src/landlock.rs`

```rust
use landlock::{
    Access, AccessFs, Ruleset, RulesetAttr, RulesetCreatedAttr,
    ABI,
};

pub fn apply_landlock(workspace: &Path) -> Result<()> {
    let abi = ABI::V1;
    
    let mut ruleset = Ruleset::new()
        .handle_access(AccessFs::from_all(abi))?
        .create()?;
    
    // Allow reading everything
    ruleset = ruleset
        .add_rule(PathBeneath::new("/", AccessFs::from_read(abi)))?;
    
    // Allow writing to workspace
    ruleset = ruleset
        .add_rule(PathBeneath::new(workspace, AccessFs::from_all(abi)))?;
    
    // Allow writing to /tmp
    ruleset = ruleset
        .add_rule(PathBeneath::new("/tmp", AccessFs::from_all(abi)))?;
    
    // Restrict current process
    ruleset.restrict_self()?;
    
    Ok(())
}
```

**Features**:
- Filesystem access control
- Per-path granularity
- Kernel-enforced
- Minimal overhead

### Docker (Linux Alternative)

**Script**: `codex-cli/scripts/run_in_container.sh`

```bash
#!/bin/bash

# Build container with firewall rules
docker build -t codex-sandbox .

# Run with volume mounts
docker run -it --rm \
  --network codex-net \
  -v "$PWD:/workspace:rw" \
  -v "$HOME/.codex:/root/.codex:rw" \
  -e OPENAI_API_KEY \
  codex-sandbox \
  "$@"
```

**Firewall** (`scripts/init_firewall.sh`):
```bash
# Create ipset for allowed IPs
ipset create openai_ips hash:ip

# Add OpenAI API IPs
ipset add openai_ips $(dig +short api.openai.com)

# Block all outbound except OpenAI
iptables -A OUTPUT -m set --match-set openai_ips dst -j ACCEPT
iptables -A OUTPUT -j DROP
```

---

## Command Safety

### Allowlist (Safe Commands)

**Location**: `core/src/command_safety/is_safe_command.rs`

```rust
pub fn is_safe_command(cmd: &[String]) -> bool {
    if cmd.is_empty() {
        return false;
    }
    
    let program = &cmd[0];
    let safe_programs = [
        // Readers
        "cat", "less", "more", "head", "tail",
        "ls", "pwd", "find", "which", "whereis",
        
        // Searchers
        "grep", "rg", "ag", "ack",
        
        // Git (read-only)
        "git log", "git status", "git diff", "git show",
        
        // Package managers (info only)
        "npm list", "cargo --version", "pip list",
        
        // System info
        "uname", "whoami", "env", "date",
    ];
    
    safe_programs.iter().any(|&safe| {
        program == safe || cmd.join(" ").starts_with(safe)
    })
}
```

### Blocklist (Dangerous Commands)

**Location**: `core/src/command_safety/is_dangerous_command.rs`

```rust
pub fn is_dangerous_command(cmd: &[String]) -> bool {
    if cmd.is_empty() {
        return false;
    }
    
    let command_str = cmd.join(" ");
    
    // Destructive operations
    let dangerous_patterns = [
        "rm -rf /",
        "rm -rf ~",
        "rm -rf .",
        "dd if=/dev/zero",
        "mkfs.",
        "> /dev/sda",
        
        // System modifications
        "sudo rm",
        "chmod 777",
        "chown root",
        
        // Privilege escalation
        "sudo su",
        "sudo -i",
        
        // Network exposure
        "chmod +x /tmp",
        "curl | sh",
        "wget | bash",
    ];
    
    dangerous_patterns.iter().any(|&pattern| {
        command_str.contains(pattern)
    })
}
```

### Windows Safe Commands

**Location**: `core/src/command_safety/windows_safe_commands.rs`

```rust
pub fn is_safe_windows_command(cmd: &[String]) -> bool {
    let program = &cmd[0];
    let safe = [
        "dir", "type", "more",
        "findstr", "where",
        "echo", "set",
        // ... Windows equivalents
    ];
    
    safe.contains(&program.as_str())
}
```

---

## Resource Limits

### Timeouts

```rust
// core/src/exec.rs
pub const DEFAULT_TIMEOUT: Duration = Duration::from_secs(30);
pub const MAX_TIMEOUT: Duration = Duration::from_secs(300); // 5 min

pub async fn exec_with_timeout(
    cmd: &[String],
    timeout: Duration,
) -> Result<Output> {
    tokio::time::timeout(timeout, exec_command(cmd))
        .await
        .map_err(|_| CodexErr::Timeout)?
}
```

### Output Size Limits

```rust
pub const DEFAULT_MAX_OUTPUT_SIZE: usize = 10 * 1024 * 1024; // 10 MB

pub async fn exec_with_limits(
    cmd: &[String],
) -> Result<Output> {
    let mut child = Command::new(&cmd[0])
        .args(&cmd[1..])
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()?;
    
    let mut output = Vec::new();
    let mut reader = BufReader::new(child.stdout.take().unwrap());
    
    while output.len() < DEFAULT_MAX_OUTPUT_SIZE {
        let mut buf = vec![0; 8192];
        let n = reader.read(&mut buf).await?;
        if n == 0 { break; }
        output.extend_from_slice(&buf[..n]);
    }
    
    if output.len() >= DEFAULT_MAX_OUTPUT_SIZE {
        output.extend_from_slice(b"\n[output truncated]");
    }
    
    Ok(Output { stdout: output, ... })
}
```

### Memory Limits

```rust
// Enforced by OS sandbox policies
(sandbox-memory-limit 1073741824)  ; 1 GB
```

---

## Best Practices

### For Users

1. **Start Restrictive**: Use `untrusted` or `on-failure` initially
2. **Review Approvals**: Read what you're approving carefully
3. **Use Version Control**: Always have git backup before `full-auto`
4. **Check Workspace**: Ensure correct directory before running

### For Developers

1. **Validate Early**: Check permissions before execution
2. **Fail Secure**: Default to denying access on errors
3. **Log Everything**: Record all security decisions
4. **Test Sandboxing**: Verify sandbox actually blocks operations

### For Organizations

1. **Audit Logs**: Review what agents are doing
2. **Restrict Models**: Only allow specific models/providers
3. **Network Policies**: Control what APIs can be accessed
4. **Workspace Isolation**: Run in dedicated directories

---

## Related Documentation

- [06-tool-system.md](./06-tool-system.md) - Tool implementation
- [08-configuration.md](./08-configuration.md) - Security configuration
- [10-implementation.md](./10-implementation.md) - Implementation details

