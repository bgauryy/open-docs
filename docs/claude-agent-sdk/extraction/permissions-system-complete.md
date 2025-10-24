# Claude Agent SDK - Permission System Complete Reference

**SDK Version**: 0.1.22
**Extraction Date**: 2025-10-24
**Source**: `sdkTypes.d.ts`, `cli.js`, analyzed from SDK source

---

## Table of Contents

1. [Overview](#overview)
2. [Permission Modes](#permission-modes)
3. [Permission Resolution](#permission-resolution)
4. [Permission Rules](#permission-rules)
5. [Permission Updates](#permission-updates)
6. [Permission Suggestions](#permission-suggestions)
7. [Hook Permission Override](#hook-permission-override)
8. [Real-World Patterns](#real-world-patterns)
9. [Gotchas & Best Practices](#gotchas--best-practices)

---

## Overview

The Claude Agent SDK implements a sophisticated **6-level permission system** that controls what operations agents can perform. This system balances security, usability, and flexibility through:

- **Permission Modes**: 4 operational modes for different workflows
- **Multi-Level Rules**: 6-level cascade from session to defaults
- **Dynamic Updates**: Runtime permission changes
- **Hook Integration**: Programmatic permission decisions
- **Suggestions**: Tool-provided permission recommendations

### Key Concepts

```
Permission Request
     ↓
Check Session Rules → Check Local Settings → Check Project Settings
     ↓                    ↓                        ↓
Check User Settings → Check Policy Settings → Check Permission Mode
     ↓
Decision: allow | deny | ask
```

---

## Permission Modes

Permission modes control the default behavior when no explicit rule matches.

### Mode 1: `default`

**Behavior**: Balance between safety and usability

```typescript
const DEFAULT_MODE_BEHAVIOR = {
  // File Operations
  Read: 'allow',              // ✅ Always allowed
  Write: 'ask',               // ⚠️ Ask user first
  Edit: 'ask',                // ⚠️ Ask user first
  Delete: 'ask',              // ⚠️ Ask user first
  
  // Execution
  Bash: 'ask',                // ⚠️ Ask user first
  BashOutput: 'allow',        // ✅ Reading output allowed
  
  // Discovery (safe operations)
  Glob: 'allow',              // ✅ Always allowed
  Grep: 'allow',              // ✅ Always allowed
  
  // Agent Operations
  Task: 'allow',              // ✅ Subagents allowed
  
  // User Interaction
  AskUserQuestion: 'allow',   // ✅ Always allowed
  
  // Web Operations
  WebFetch: 'ask',            // ⚠️ Ask user first
  WebSearch: 'ask',           // ⚠️ Ask user first
  
  // MCP Tools
  MCPTool: 'ask'              // ⚠️ Ask user first (default)
};
```

**When to Use**:
- Normal development workflows
- Interactive sessions
- Learning the SDK
- Default choice for most users

**Characteristics**:
- Safe operations auto-approved (Read, Grep, Glob)
- Risky operations require confirmation (Write, Bash)
- Good balance of security and productivity

---

### Mode 2: `acceptEdits`

**Behavior**: Auto-approve file modifications, still ask for execution

```typescript
const ACCEPT_EDITS_BEHAVIOR = {
  // File Operations - AUTO-APPROVED
  Read: 'allow',              // ✅ Auto-approved
  Write: 'allow',             // ✅ Auto-approved (different from default!)
  Edit: 'allow',              // ✅ Auto-approved (different from default!)
  Delete: 'ask',              // ⚠️ Still asks (destructive)
  
  // Execution - STILL ASK
  Bash: 'ask',                // ⚠️ Still asks (dangerous)
  
  // Discovery
  Glob: 'allow',              // ✅ Auto-approved
  Grep: 'allow',              // ✅ Auto-approved
  
  // Everything else: same as 'default'
};
```

**When to Use**:
- Rapid iteration on code
- Trusted codebases
- Refactoring workflows
- AI-assisted development

**Characteristics**:
- File writes auto-approved (no confirmation)
- Command execution still requires approval
- Faster development cycle
- Higher risk (auto-approves file changes)

**Example Session**:
```bash
$ claude-code --permission-mode acceptEdits

> Refactor the authentication system

# Agent performs multiple file edits without asking:
✅ Edit src/auth/login.ts (auto-approved)
✅ Edit src/auth/logout.ts (auto-approved)
✅ Write tests/auth.test.ts (auto-approved)

# But still asks for bash:
⚠️ Run tests? npm test [y/n]
```

---

### Mode 3: `bypassPermissions`

**Behavior**: **AUTO-APPROVE EVERYTHING** (⚠️ DANGEROUS)

```typescript
const BYPASS_BEHAVIOR = {
  // EVERYTHING AUTO-APPROVED
  '*': 'allow'  // All tools, all operations
};
```

**When to Use**:
- ⚠️ Trusted, isolated environments only
- Automated CI/CD pipelines
- Sandboxed testing
- Code generation scripts
- **NEVER** in production or with untrusted code

**Characteristics**:
- Zero user interaction
- Maximum risk
- Maximum speed
- No safety net

**Security Warning**:
```
⚠️ WARNING ⚠️
This mode allows the agent to:
- Delete any file
- Execute any command (rm -rf /, curl malicious-site.com)
- Modify system files
- Access secrets
- Make network requests

Use ONLY in isolated, sandboxed, or trusted environments!
```

**Example Session**:
```bash
$ claude-code --permission-mode bypassPermissions

> Delete all node_modules and reinstall

# Agent executes everything without asking:
✅ Bash: find . -name node_modules -type d -exec rm -rf {} + (auto-approved)
✅ Bash: npm install (auto-approved)
# No user confirmation required
```

---

### Mode 4: `plan`

**Behavior**: Plan-only mode, NO execution

```typescript
const PLAN_MODE_BEHAVIOR = {
  // ALL OPERATIONS BLOCKED (except planning)
  '*': 'deny',
  
  // Planning tools allowed
  Read: 'allow',              // ✅ Can read to understand
  Glob: 'allow',              // ✅ Can discover structure
  Grep: 'allow',              // ✅ Can search code
  Task: 'allow',              // ✅ Can delegate sub-planning
  
  // Special: TodoWrite allowed for plan creation
  TodoWrite: 'allow',         // ✅ Can create plan
  
  // Exit plan mode
  ExitPlanMode: 'allow'       // ✅ Can exit to execute
};
```

**When to Use**:
- Planning phase before execution
- Code review and analysis
- Understanding unfamiliar codebases
- Generating task lists
- Risk assessment

**Characteristics**:
- No file modifications
- No command execution
- Creates TODO list for later execution
- Safe exploration

**Workflow**:
```bash
$ claude-code --permission-mode plan

> Refactor the authentication system

Agent: [Planning mode]
1. Analyzing current auth implementation...
   ✅ Read src/auth/*.ts
   ✅ Grep for auth patterns
   
2. Creating refactoring plan...
   ✅ TodoWrite (5 tasks created)

3. Plan complete. Tasks:
   [ ] Update login.ts to use new token format
   [ ] Refactor logout.ts error handling  
   [ ] Add tests for new auth flow
   [ ] Update documentation
   [ ] Deploy to staging

Ready to execute? Use /exit-plan or ExitPlanMode tool.

> /exit-plan
[Switching to default mode for execution...]
```

---

## Permission Resolution

### 6-Level Cascade

Permission resolution follows a strict precedence order:

```
1. Session Rules (highest priority)
   ↓ No match
2. Local Settings (.claude/ in cwd)
   ↓ No match
3. Project Settings (.claude/ in git root)
   ↓ No match
4. User Settings (~/.claude/)
   ↓ No match
5. Policy Settings (enterprise/managed)
   ↓ No match
6. Permission Mode (default/acceptEdits/bypassPermissions/plan)
```

**First Match Wins**: Once a rule matches, resolution stops.

### Resolution Algorithm

```typescript
function resolvePermission(
  tool: string,
  input: unknown,
  context: PermissionContext
): 'allow' | 'deny' | 'ask' {
  
  // Level 1: Session rules (temporary, this session only)
  const sessionRule = matchRule(context.sessionRules, tool, input);
  if (sessionRule) return sessionRule.behavior;
  
  // Level 2: Local settings (cwd/.claude/)
  const localRule = matchRule(context.localSettings.rules, tool, input);
  if (localRule) return localRule.behavior;
  
  // Level 3: Project settings (git root/.claude/)
  const projectRule = matchRule(context.projectSettings.rules, tool, input);
  if (projectRule) return projectRule.behavior;
  
  // Level 4: User settings (~/.claude/)
  const userRule = matchRule(context.userSettings.rules, tool, input);
  if (userRule) return userRule.behavior;
  
  // Level 5: Policy settings (enterprise)
  const policyRule = matchRule(context.policySettings.rules, tool, input);
  if (policyRule) return policyRule.behavior;
  
  // Level 6: Permission mode (fallback)
  return context.permissionMode.defaultBehavior(tool);
}
```

### Resolution Examples

**Example 1: Session rule overrides all**:
```typescript
// Session rule: Allow all Bash commands
sessionRules: [{ tool: "Bash", behavior: "allow" }]

// User settings: Deny Bash
userSettings.rules: [{ tool: "Bash", behavior: "deny" }]

// Result: ALLOW (session wins)
```

**Example 2: Cascade to mode**:
```typescript
// No rules defined anywhere
sessionRules: []
localSettings.rules: []
projectSettings.rules: []
userSettings.rules: []
policySettings.rules: []

// Permission mode: default
permissionMode: "default"

// Result: ASK (falls through to mode default behavior)
```

**Example 3: Project rule beats user rule**:
```typescript
// Project settings (higher priority)
projectSettings.rules: [
  { tool: "Write", path: "src/**", behavior: "allow" }
]

// User settings (lower priority)
userSettings.rules: [
  { tool: "Write", behavior: "deny" }
]

// Write to src/app.ts: ALLOW (project rule matches first)
// Write to docs/README.md: DENY (no project match, falls to user rule)
```

---

## Permission Rules

### Rule Structure

```typescript
type PermissionRule = {
  // Tool matcher
  tool?: string;              // Tool name or pattern
  tools?: string[];           // Multiple tools
  
  // Path matcher (for file operations)
  path?: string;              // File path or glob pattern
  paths?: string[];           // Multiple paths
  
  // Command matcher (for Bash tool)
  command?: string;           // Command or pattern
  
  // Decision
  behavior: 'allow' | 'deny' | 'ask';
  
  // Optional metadata
  reason?: string;            // Why this rule exists
  expires?: number;           // Expiration timestamp (session rules)
};
```

### Rule Matching

**Tool Matching**:
```typescript
// Exact match
{ tool: "Bash", behavior: "allow" }  // Matches: Bash tool

// Pattern match
{ tool: "File*", behavior: "allow" }  // Matches: FileRead, FileWrite, FileEdit

// Multiple tools
{ tools: ["Read", "Write", "Edit"], behavior: "allow" }

// Wildcard
{ tool: "*", behavior: "deny" }  // Matches: everything
```

**Path Matching** (file operations):
```typescript
// Exact path
{ tool: "Write", path: "src/config.json", behavior: "allow" }

// Glob pattern
{ tool: "Write", path: "src/**/*.ts", behavior: "allow" }
{ tool: "Write", path: "src/**", behavior: "allow" }  // All in src/

// Multiple paths
{ 
  tool: "Write", 
  paths: ["src/**", "tests/**"], 
  behavior: "allow" 
}

// Deny pattern
{ tool: "Delete", path: "*.env", behavior: "deny" }  // Protect .env files
```

**Command Matching** (Bash tool):
```typescript
// Exact command
{ tool: "Bash", command: "npm test", behavior: "allow" }

// Command prefix
{ tool: "Bash", command: "git*", behavior: "allow" }  // All git commands

// Deny dangerous commands
{ tool: "Bash", command: "rm -rf*", behavior: "deny" }
{ tool: "Bash", command: "curl*", behavior: "ask" }
```

### Example Rule Sets

**Example 1: Project-specific rules**:
```json
{
  "permissions": {
    "rules": [
      {
        "tool": "Write",
        "paths": ["src/**/*.ts", "src/**/*.tsx"],
        "behavior": "allow",
        "reason": "Allow TypeScript file edits"
      },
      {
        "tool": "Bash",
        "command": "npm test",
        "behavior": "allow",
        "reason": "Allow running tests"
      },
      {
        "tool": "Bash",
        "command": "npm install*",
        "behavior": "ask",
        "reason": "Confirm dependency installs"
      },
      {
        "tool": "Delete",
        "behavior": "deny",
        "reason": "Never auto-delete files"
      }
    ]
  }
}
```

**Example 2: Security-focused rules**:
```json
{
  "permissions": {
    "rules": [
      {
        "tool": "Bash",
        "command": "rm -rf*",
        "behavior": "deny",
        "reason": "Block destructive rm commands"
      },
      {
        "tool": "Write",
        "path": "**/*.env",
        "behavior": "deny",
        "reason": "Protect environment files"
      },
      {
        "tool": "Write",
        "path": "**/.git/**",
        "behavior": "deny",
        "reason": "Protect git metadata"
      },
      {
        "tool": "WebFetch",
        "behavior": "ask",
        "reason": "Confirm external requests"
      }
    ]
  }
}
```

---

## Permission Updates

Permission rules can be updated at runtime via hooks or programmatically.

### Update Types

```typescript
type PermissionUpdate = 
  | { type: 'addRules'; rules: PermissionRule[] }
  | { type: 'replaceRules'; rules: PermissionRule[] }
  | { type: 'removeRules'; indices: number[] }
  | { type: 'setMode'; mode: PermissionMode }
  | { type: 'addDirectories'; directories: string[] }
  | { type: 'removeDirectories'; directories: string[] };
```

### Update Methods

**1. Add Rules** (append to existing):
```typescript
{
  type: 'addRules',
  rules: [
    { tool: "Bash", command: "npm test", behavior: "allow" },
    { tool: "Write", path: "temp/**", behavior: "allow" }
  ]
}
```

**2. Replace Rules** (clear and set new):
```typescript
{
  type: 'replaceRules',
  rules: [
    { tool: "*", behavior: "ask" }  // Reset to safe default
  ]
}
```

**3. Remove Rules** (by index):
```typescript
{
  type: 'removeRules',
  indices: [0, 2, 5]  // Remove rules at indices 0, 2, 5
}
```

**4. Change Mode**:
```typescript
{
  type: 'setMode',
  mode: 'acceptEdits'  // Switch to acceptEdits mode
}
```

**5. Add Directories** (grant access):
```typescript
{
  type: 'addDirectories',
  directories: [
    '/tmp/workspace',
    '/home/user/projects/other-repo'
  ]
}
```

**6. Remove Directories**:
```typescript
{
  type: 'removeDirectories',
  directories: ['/tmp/workspace']
}
```

### Runtime Updates via Query

```typescript
const query = await query({
  prompt: "Start working",
  options: { permissionMode: 'default' }
});

// Later: Change mode
await query.setPermissionMode('acceptEdits');

// Continue conversation with new permission mode
```

---

## Permission Suggestions

Tools can suggest permission rules to streamline workflows.

### Suggestion Structure

```typescript
type PermissionSuggestion = {
  tool: string;
  rule: PermissionRule;
  reason: string;
  confidence: 'high' | 'medium' | 'low';
};
```

### How Suggestions Work

```
1. Tool execution blocked (behavior: 'deny' or 'ask')
2. Tool analyzes context and input
3. Tool generates suggestion(s)
4. User presented with suggestion(s)
5. User accepts/rejects
6. If accepted: Rule added to session
```

### Example Suggestions

**Bash Tool Suggestion**:
```typescript
// User denies: Bash({ command: "npm test" })
// Tool suggests:
{
  tool: "Bash",
  rule: {
    tool: "Bash",
    command: "npm test",
    behavior: "allow"
  },
  reason: "Auto-approve npm test (safe, common operation)",
  confidence: "high"
}
```

**Write Tool Suggestion**:
```typescript
// User approves multiple writes to src/
// Tool suggests:
{
  tool: "Write",
  rule: {
    tool: "Write",
    path: "src/**",
    behavior: "allow"
  },
  reason: "You've approved 3 writes to src/ - auto-approve all?",
  confidence: "medium"
}
```

### Suggestion UI Flow

```bash
> Write to src/config.ts? [y/n/always]

y - Approve once
n - Deny once
always - Add rule: Allow writes to src/**

User: always

✅ Rule added: { tool: "Write", path: "src/**", behavior: "allow" }
   (This session only)
```

---

## Hook Permission Override

Hooks can override permission decisions programmatically.

### PreToolUse Hook Override

```typescript
const securityHook: HookCallback = async (input) => {
  if (input.hook_event_name === 'PreToolUse') {
    
    // Example: Block force pushes to main
    if (input.tool_name === 'Bash') {
      const cmd = input.tool_input.command;
      
      if (cmd.includes('git push --force') && cmd.includes('main')) {
        return {
          decision: 'block',  // Override: DENY
          systemMessage: '🚫 Force push to main is not allowed',
          hookSpecificOutput: {
            hookEventName: 'PreToolUse',
            permissionDecision: 'deny',
            permissionDecisionReason: 'Company policy: no force push to main'
          }
        };
      }
    }
    
    // Example: Auto-approve safe operations
    if (input.tool_name === 'Read') {
      return {
        decision: 'approve',  // Override: ALLOW
        hookSpecificOutput: {
          hookEventName: 'PreToolUse',
          permissionDecision: 'allow',
          permissionDecisionReason: 'Read operations always safe'
        }
      };
    }
  }
  
  return { decision: 'approve' };  // Default: proceed with normal resolution
};
```

### Hook Override Priority

```
Hook Decision → Overrides entire permission system
     ↓
If hook returns 'approve': Proceed to normal resolution
If hook returns 'block': Immediately deny (skip all rules)
```

**Important**: Hook overrides bypass ALL permission rules and modes!

---

## Real-World Patterns

### Pattern 1: Safe CI/CD Mode

```typescript
// CI/CD environment with safety guards
const cicdOptions = {
  permissionMode: 'bypassPermissions',  // Auto-approve
  hooks: {
    PreToolUse: [{
      hooks: [async (input) => {
        // But block dangerous operations
        if (input.tool_name === 'Bash') {
          const cmd = input.tool_input.command;
          
          // Block production access
          if (cmd.includes('prod') || cmd.includes('production')) {
            return { decision: 'block', systemMessage: '🚫 Production access blocked in CI/CD' };
          }
          
          // Block file deletion
          if (cmd.includes('rm -rf')) {
            return { decision: 'block', systemMessage: '🚫 rm -rf blocked' };
          }
        }
        
        return { decision: 'approve' };
      }]
    }]
  }
};
```

### Pattern 2: Rapid Development Mode

```typescript
// Fast iteration with undo safety
const devOptions = {
  permissionMode: 'acceptEdits',  // Auto-approve file edits
  additionalDirectories: [
    '/tmp/dev-workspace'  // Grant access to scratch space
  ],
  hooks: {
    PreToolUse: [{
      hooks: [async (input) => {
        // Auto-checkpoint before destructive operations
        if (input.tool_name === 'Delete' || 
            (input.tool_name === 'Bash' && input.tool_input.command.includes('rm'))) {
          await createCheckpoint();
        }
        return { decision: 'approve' };
      }]
    }]
  }
};
```

### Pattern 3: Secure Review Mode

```typescript
// Code review with strict permissions
const reviewOptions = {
  permissionMode: 'default',  // Ask for everything
  hooks: {
    PreToolUse: [{
      hooks: [async (input) => {
        // Log all operations
        await logOperation(input);
        
        // Deny any modifications
        if (['Write', 'Edit', 'Delete'].includes(input.tool_name)) {
          return {
            decision: 'block',
            systemMessage: '🚫 Read-only review mode - no modifications allowed'
          };
        }
        
        return { decision: 'approve' };
      }]
    }]
  }
};
```

### Pattern 4: Project-Specific Rules

```json
// .claude/settings.json in project root
{
  "permissions": {
    "mode": "acceptEdits",
    "rules": [
      // Allow safe operations
      {
        "tools": ["Read", "Glob", "Grep"],
        "behavior": "allow",
        "reason": "Discovery operations always safe"
      },
      
      // Allow edits to source
      {
        "tool": "Write",
        "paths": ["src/**/*.ts", "src/**/*.tsx"],
        "behavior": "allow",
        "reason": "TypeScript source files"
      },
      
      // Block production configs
      {
        "tool": "Write",
        "paths": ["**/*.production.*", "**/prod-*"],
        "behavior": "deny",
        "reason": "Protect production configs"
      },
      
      // Ask for dependency changes
      {
        "tool": "Write",
        "path": "package.json",
        "behavior": "ask",
        "reason": "Confirm dependency changes"
      },
      
      // Allow safe bash commands
      {
        "tool": "Bash",
        "command": "npm test",
        "behavior": "allow"
      },
      {
        "tool": "Bash",
        "command": "npm run lint",
        "behavior": "allow"
      },
      {
        "tool": "Bash",
        "command": "git status",
        "behavior": "allow"
      },
      {
        "tool": "Bash",
        "command": "git diff*",
        "behavior": "allow"
      }
    ]
  }
}
```

---

## Gotchas & Best Practices

### Gotchas

1. **Mode Changes Not Retroactive**:
   ```typescript
   // Deny a command in default mode
   await query.setPermissionMode('bypassPermissions');
   // Previous denials NOT reversed - only affects NEW requests
   ```

2. **First Match Wins** (can be surprising):
   ```typescript
   rules: [
     { tool: "*", behavior: "deny" },      // Rule 1: Deny all
     { tool: "Read", behavior: "allow" }   // Rule 2: Never reached!
   ]
   // All tools denied (including Read)
   // Fix: Put specific rules before wildcards
   ```

3. **Session Rules Cleared on Resume**:
   ```bash
   # Session 1: Add rule
   # Session 2: Resume
   # Session rules: CLEARED (must recreate)
   ```

4. **Hook Overrides Bypass Everything**:
   ```typescript
   // Even if permission mode is bypassPermissions:
   hook returns { decision: 'block' }
   // → Operation DENIED (hook wins)
   ```

5. **Path Matching is Prefix-Based**:
   ```typescript
   { path: "src/", behavior: "allow" }
   // Matches: src/app.ts ✅
   // Matches: src (file named "src") ✅ (might not want this!)
   // Better: path: "src/**"
   ```

6. **Command Pattern Matching Unclear**:
   ```typescript
   { command: "git*", behavior: "allow" }
   // What does * match?
   // - "git status" ✅
   // - "git push" ✅
   // - "git-lfs" ✅ (might not want this)
   // Documentation unclear on pattern syntax
   ```

### Best Practices

**1. Order Rules from Specific to General**:
```typescript
// ✅ Good
rules: [
  { tool: "Bash", command: "rm -rf*", behavior: "deny" },  // Specific
  { tool: "Bash", behavior: "ask" },                       // General
  { tool: "*", behavior: "allow" }                         // Wildcard
]

// ❌ Bad
rules: [
  { tool: "*", behavior: "allow" },                        // Wildcard first
  { tool: "Bash", command: "rm -rf*", behavior: "deny" }   // Never reached!
]
```

**2. Use Project Settings for Team Consistency**:
```json
// .claude/settings.json (committed to repo)
{
  "permissions": {
    "mode": "acceptEdits",
    "rules": [/* team-agreed rules */]
  }
}
```

**3. Use Session Rules for One-Off Permissions**:
```typescript
// Temporary permission for this session
await addSessionRule({
  tool: "Bash",
  command: "npm install --force",
  behavior: "allow",
  expires: Date.now() + 3600000  // 1 hour
});
```

**4. Document Permission Rules**:
```json
{
  "rules": [
    {
      "tool": "Write",
      "path": "src/**",
      "behavior": "allow",
      "reason": "Allow source edits (reviewed in PR)"  // ✅ Clear reason
    }
  ]
}
```

**5. Test Permission Rules**:
```typescript
// Test that dangerous operations are blocked
async function testPermissions() {
  // Should be denied
  try {
    await Bash({ command: "rm -rf /" });
    throw new Error("❌ Should have been blocked!");
  } catch (e) {
    console.log("✅ Dangerous command correctly blocked");
  }
  
  // Should be allowed
  await Read({ file_path: "src/app.ts" });
  console.log("✅ Safe operation allowed");
}
```

**6. Use Hooks for Dynamic Logic**:
```typescript
// Allow operations during business hours only
const businessHoursHook = async (input) => {
  const hour = new Date().getHours();
  
  if (hour < 9 || hour > 17) {
    // Outside business hours: ask for everything
    return { decision: 'approve' };  // Proceed to normal (ask)
  }
  
  // Business hours: auto-approve safe operations
  if (['Read', 'Grep', 'Glob'].includes(input.tool_name)) {
    return { decision: 'approve' };
  }
  
  return { decision: 'approve' };
};
```

---

## Summary

### Permission Mode Comparison

| Mode | File Writes | Bash | Best For |
|------|------------|------|----------|
| `default` | Ask | Ask | Normal development |
| `acceptEdits` | Allow | Ask | Rapid iteration |
| `bypassPermissions` | Allow | Allow | CI/CD, trusted environments |
| `plan` | Deny | Deny | Planning, code review |

### Resolution Priority (Highest to Lowest)

1. **Hook Decision** (if returns 'block' or specific decision)
2. **Session Rules** (temporary, this session)
3. **Local Settings** (cwd/.claude/)
4. **Project Settings** (git root/.claude/)
5. **User Settings** (~/.claude/)
6. **Policy Settings** (enterprise)
7. **Permission Mode** (default fallback)

### Key Takeaways

- ✅ Use `default` mode for safety (ask before risky operations)
- ✅ Use `acceptEdits` for rapid iteration (auto-approve file changes)
- ⚠️ Use `bypassPermissions` ONLY in isolated/trusted environments
- ✅ Use `plan` mode for safe exploration and planning
- ✅ Order rules from specific to general (first match wins)
- ✅ Use project settings for team consistency
- ✅ Use hooks for dynamic, programmatic permission logic
- ⚠️ Hook overrides bypass ALL rules (use with caution)
- ✅ Document permission rules with `reason` field
- ✅ Test permission rules to ensure security
