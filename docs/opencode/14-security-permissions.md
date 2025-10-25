# OpenCode - Security & Permissions

> **Permission system and security model for safe AI agent operations**

---

## Overview

OpenCode's permission system ensures AI agents can't perform dangerous operations without approval:
- **Permission levels**: allow, deny, ask
- **Granular control**: Per-tool, per-command
- **Approval workflow**: Interactive confirmation
- **Audit trail**: All operations logged

**Files**:
- `permission/index.ts` - Permission system
- Agent configs define default permissions

---

## Permission Model

### Permission Levels

```typescript
type PermissionLevel = "allow" | "deny" | "ask"
```

- **allow** - Execute without asking
- **deny** - Reject automatically
- **ask** - Require user approval

### Permission Types

```typescript
interface Permissions {
  edit: PermissionLevel                    // File editing (edit, write, patch)
  bash: Record<string, PermissionLevel>    // Shell commands
  webfetch: PermissionLevel               // Web access
}
```

---

## Configuration

### Agent Permissions

**.opencode/config.json**:
```json
{
  "agents": {
    "default": {
      "permission": {
        "edit": "ask",
        "bash": {
          "*": "ask",
          "npm test": "allow",
          "npm run build": "allow",
          "rm -rf": "deny"
        },
        "webfetch": "allow"
      }
    },
    "readonly": {
      "permission": {
        "edit": "deny",
        "bash": { "*": "deny" },
        "webfetch": "allow"
      }
    },
    "trusted": {
      "permission": {
        "edit": "allow",
        "bash": { "*": "allow" },
        "webfetch": "allow"
      }
    }
  }
}
```

### Using Agents

```bash
# Default agent (asks for permissions)
opencode "Make changes"

# Readonly agent (no edits)
opencode --agent readonly "Analyze code"

# Trusted agent (no prompts)
opencode --agent trusted "Refactor quickly"
```

---

## Permission Workflow

### File Editing

```
AI requests edit
    │
    ▼
Check agent permission.edit
    │
    ├─"allow"──▶ Execute immediately
    │
    ├─"deny"───▶ Return error: "Edit permission denied"
    │
    └─"ask"────┐
               ▼
          Show prompt:
          ╔══════════════════════════╗
          ║ Allow edit to auth.ts?   ║
          ║                          ║
          ║ Old: password            ║
          ║ New: hashedPassword      ║
          ║                          ║
          ║ [Yes] [No] [Always]      ║
          ╚══════════════════════════╝
               │
               ├─Yes──────▶ Execute once
               ├─No───────▶ Cancel
               └─Always───▶ Update config to "allow"
```

### Shell Commands

```
AI requests: bash("npm install axios")
    │
    ▼
Check agent permission.bash
    │
    ├─ Exact match ("npm install axios"): "allow"
    │     └─▶ Execute immediately
    │
    ├─ Pattern match ("npm *"): "ask"
    │     └─▶ Prompt user
    │
    ├─ Wildcard ("*"): "ask"
    │     └─▶ Prompt user
    │
    └─ Default: "deny"
          └─▶ Return error
```

**Examples**:
```json
{
  "bash": {
    "npm test": "allow",           // Exact command
    "npm run *": "allow",          // Pattern
    "git status": "allow",
    "git push": "ask",             // Require approval
    "rm -rf": "deny",              // Never allow
    "*": "ask"                     // Default for others
  }
}
```

---

## Dangerous Commands

OpenCode detects and warns about dangerous commands:

**Patterns Detected**:
- `rm -rf` - Recursive deletion
- `sudo` - Elevated privileges
- `chmod 777` - Dangerous permissions
- `:(){:|:&};:` - Fork bomb
- `dd if=/dev/zero` - Disk wipe
- `mkfs` - Format filesystem

**Handling**:
```typescript
function isDangerous(command: string): boolean {
  const patterns = [
    /rm\s+-rf\s+[\/~]/,
    /sudo\s+rm/,
    /chmod\s+777/,
    // ... more patterns
  ]
  
  return patterns.some(p => p.test(command))
}

if (isDangerous(command)) {
  log.warn("Dangerous command detected", { command })
  
  // Require explicit approval
  const approved = await Permission.confirm({
    type: "dangerous",
    command,
    warning: "This command is potentially dangerous!"
  })
  
  if (!approved) {
    throw new Error("Dangerous command rejected")
  }
}
```

---

## Approval UI

### Interactive Prompt

**Terminal**:
```
┌─────────────────────────────────────────┐
│ Permission Request                      │
├─────────────────────────────────────────┤
│                                         │
│ Tool: bash                              │
│ Command: npm install axios              │
│                                         │
│ This will:                              │
│ - Download and install package          │
│ - Modify package.json                   │
│ - Update node_modules                   │
│                                         │
│ Allow? [y/n/a]                          │
│   y - Yes (this time)                   │
│   n - No (cancel)                       │
│   a - Always (update config)            │
│                                         │
└─────────────────────────────────────────┘
```

### Batch Approval

For multiple operations:
```
┌─────────────────────────────────────────┐
│ Multiple Permissions Requested          │
├─────────────────────────────────────────┤
│                                         │
│ 1. Edit src/auth.ts                     │
│ 2. Edit src/user.ts                     │
│ 3. Run: npm test                        │
│                                         │
│ Allow all? [y/n/r]                      │
│   y - Yes (all)                         │
│   n - No (cancel all)                   │
│   r - Review (one by one)               │
│                                         │
└─────────────────────────────────────────┘
```

---

## Audit Trail

### Logging

All operations are logged:

```typescript
log.info("Tool execution", {
  tool: "bash",
  args: { command: "npm install axios" },
  permission: "ask",
  approved: true,
  user: "alice",
  timestamp: Date.now()
})
```

### Session History

View permission decisions:
```bash
opencode debug session --show-permissions
```

Output:
```
Session: session_abc123

Permissions:
  12:34:56 - bash "npm test" - ALLOWED (config)
  12:35:12 - edit auth.ts - ASKED → APPROVED
  12:36:03 - bash "rm file.txt" - ASKED → DENIED
  12:37:45 - webfetch https://api.example.com - ALLOWED (config)
```

---

## Security Best Practices

### Configuration

**Development**:
```json
{
  "agents": {
    "default": {
      "permission": {
        "edit": "ask",
        "bash": {
          "npm test": "allow",
          "npm run dev": "allow",
          "*": "ask"
        }
      }
    }
  }
}
```

**Production**:
```json
{
  "agents": {
    "default": {
      "permission": {
        "edit": "deny",
        "bash": {
          "*": "deny"
        },
        "webfetch": "allow"
      }
    }
  }
}
```

### Guidelines

1. **Start restrictive** - Use "ask" or "deny" by default
2. **Whitelist commands** - Allow specific safe commands
3. **Never auto-allow dangerous commands**
4. **Review periodically** - Audit permission logs
5. **Use agents** - Different agents for different trust levels
6. **Document decisions** - Comment permission rules

---

## Bypassing Permissions (Development)

For development/testing ONLY:

```bash
# Trust all operations (DANGEROUS)
opencode --agent trusted "Do anything"

# Or via environment
export OPENCODE_TRUST_ALL=true
opencode "Dangerous operations"
```

⚠️ **WARNING**: Never use in production!

---

## Implementation Details

### Permission Check

```typescript
export async function check(ctx: {
  sessionID: string
  type: "edit" | "bash" | "webfetch"
  details: Record<string, any>
}): Promise<boolean> {
  const agent = await Agent.get(ctx.sessionID)
  const level = getPermissionLevel(agent.permission, ctx.type, ctx.details)
  
  if (level === "allow") return true
  if (level === "deny") return false
  
  // level === "ask"
  const approved = await promptUser({
    type: ctx.type,
    details: ctx.details,
  })
  
  return approved
}
```

### Permission Storage

```typescript
// Store approval for session
const approvals = new Map<string, Set<string>>()

function rememberApproval(sessionID: string, key: string) {
  if (!approvals.has(sessionID)) {
    approvals.set(sessionID, new Set())
  }
  approvals.get(sessionID)!.add(key)
}

function wasApproved(sessionID: string, key: string): boolean {
  return approvals.get(sessionID)?.has(key) ?? false
}
```

---

## Summary

OpenCode's security model:
- **Permission levels** prevent unauthorized actions
- **Approval workflow** for sensitive operations
- **Dangerous command detection** warns users
- **Audit trail** tracks all operations
- **Configurable per-agent** for flexibility

Always start with restrictive permissions and gradually allow safe operations as needed.

---

For implementation, see `packages/opencode/src/permission/`.

