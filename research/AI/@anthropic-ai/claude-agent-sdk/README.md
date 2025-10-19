# Claude Agent SDK - Documentation

> Comprehensive documentation for the Claude Agent SDK (@anthropic-ai/claude-agent-sdk)

## What is the Claude Agent SDK?

The Claude Agent SDK is a TypeScript library that enables you to build AI agents powered by Claude. It provides:

- **Built-in Tools**: 17 tools for file operations, command execution, web search, and more
- **Custom Tools**: Add your own tools via the Model Context Protocol (MCP)
- **Agent System**: Create specialized sub-agents for complex tasks
- **Security Hooks**: Intercept and control every tool execution
- **Flexible Permissions**: From fully automatic to manual approval
- **Session Management**: Persistent conversations with branching and resuming

---

## Quick Start

```bash
npm install @anthropic-ai/claude-agent-sdk
```

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const result = await query({
    prompt: "Analyze this codebase and suggest improvements",
    apiKey: process.env.ANTHROPIC_API_KEY,
    workingDirectory: process.cwd()
});

console.log(result.messages);
```

---

## Documentation Index

| Guide | What You'll Learn |
|-------|-------------------|
| **[Comprehensive Guide](./claude-agent-sdk-comprehensive-guide.md)** | Complete overview of all features, patterns, and examples |
| **[Type System](./claude-agent-sdk-types-complete.md)** | All TypeScript types, interfaces, and schemas |
| **[Tool System](./claude-agent-sdk-tools-complete.md)** | Complete reference for all 17 built-in tools |
| **[Hooks & Permissions](./claude-agent-sdk-hooks-permissions-complete.md)** | Security, control, and permission management |
| **[Agent System](./claude-agent-sdk-agents-complete.md)** | Creating and managing specialized sub-agents |
| **[Implementation & Gotchas](./claude-agent-sdk-implementation-gotchas.md)** | Advanced features and best practices |
| **[System Prompts](./claude-agent-sdk-prompts-complete.md)** | Internal prompts and agent behavior |
| **[Undocumented Features](./claude-agent-sdk-undocumented.md)** | Hidden features and internal APIs |
| **[Design & Architecture](./claude-agent-sdk-design.md)** | Architecture overview and patterns |
| **[CLI Architecture](./architecture.md)** | Complete CLI commands, modules, and systems reference |
| **[CLI Analysis](./cli_analysis.md)** | SDK-CLI relationship and implementation discovery |
| **[Skills System](./CLAUDE-CODE-SKILLS-COMPREHENSIVE-DOCUMENTATION.md)** | Claude Code Skills: definition, loading, and execution |
| **[Internal Flows](./internal_flows.md)** | Tool internal implementation and execution flows |

---

## Quick Reference

### Core API

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

// Basic query
const result = await query({
    prompt: "Your prompt here",
    apiKey: process.env.ANTHROPIC_API_KEY
});

// Streaming query with runtime control
for await (const message of query({
    prompt: "Your prompt",
    stream: true
})) {
    if (message.type === 'assistant') {
        console.log(message.text);
    }

    // Runtime control (ONLY works with streaming!)
    message.setPermissionMode('acceptEdits');
    message.setModel('opus');
    message.setMaxThinkingTokens(10000);
    message.interrupt();
}
```

### 17 Built-in Tools

| Category | Tools |
|----------|-------|
| **File Operations** | FileRead, FileWrite, FileEdit, NotebookEdit |
| **File Discovery** | Glob, Grep |
| **Command Execution** | Bash, BashOutput, KillShell |
| **Web & Search** | WebFetch, WebSearch |
| **MCP Integration** | McpInput, ListMcpResources, ReadMcpResource |
| **Task Management** | TodoWrite |
| **Agent Control** | Agent, ExitPlanMode |

### 9 Hook Events

1. **PreToolUse** - Intercept before tool execution (can modify input or deny)
2. **PostToolUse** - Process after tool execution (can add context)
3. **Notification** - Handle agent notifications
4. **UserPromptSubmit** - Intercept user prompts (can inject context)
5. **SessionStart** - Session initialization (startup/resume/clear/compact)
6. **SessionEnd** - Session termination (with exit reason)
7. **Stop** - Agent stop events
8. **SubagentStop** - Subagent completion events
9. **PreCompact** - Before conversation compaction (manual/auto)

### 4 Permission Modes

| Mode | Behavior | Use Case |
|------|----------|----------|
| **`default`** | Prompt user for permissions | Standard interactive mode |
| **`acceptEdits`** | Auto-approve file edits only | Review mode |
| **`bypassPermissions`** | Skip all permission checks | Automation/CI |
| **`plan`** | Planning mode, no execution | Task planning |

---

## Advanced Features

### Runtime Control

Control the agent dynamically during execution (requires streaming mode):
```typescript
for await (const msg of query({ stream: true })) {
    msg.interrupt();                    // Stop execution
    msg.setPermissionMode('acceptEdits'); // Change permissions
    msg.setModel('opus');                // Switch models
    msg.setMaxThinkingTokens(10000);    // Adjust thinking budget
}
```

### Account Information

Get account details and usage information:

```typescript
const info = await message.getAccountInfo();
// Returns: { email, usage, limits, ... }
```

### MCP Server Status

Monitor the status of connected MCP servers:

```typescript
const status = await message.getMcpServersStatus();
// Returns: Map<serverName, { connected, tools, resources }>
```

### Session Management

**Fork conversations** to create branches:
```typescript
await query({
    sessionId: existingSession,
    fork: true
});
```

**Resume from a specific message**:
```typescript
await query({
    sessionId: existingSession,
    resumeFromMessageId: messageId
});
```

### Hook Timeout Configuration

Adjust the timeout for hook callbacks:

```typescript
await query({
    hookCallbackTimeoutMs: 10000  // Default is 5000ms
});
```

---

## Agent System

Create specialized sub-agents for complex tasks:

```typescript
// Define specialized subagent
const agentDefinition: AgentDefinition = {
    description: "Specialized code review agent",
    tools: ["FileRead", "Grep", "Glob"],  // Whitelist approach
    model: "opus",  // haiku | sonnet | opus | inherit
    prompt: "You are a code review specialist..."
};

// Delegate via Agent tool
{
    tool: "Agent",
    input: {
        subagent_type: "code-review",
        prompt: "Review this PR"
    }
}
```

**Tool Restriction**: Whitelist only - when `tools` array is specified, agent can **ONLY** use those tools.

**Model Selection**:
- `haiku` - Fast, economical (~$0.25/M input tokens)
- `sonnet` - Balanced, recommended default (~$3/M input tokens)
- `opus` - Highest capability (~$15/M input tokens)
- `inherit` - Use parent agent's model

---

## Getting Started

### Installation

```bash
npm install @anthropic-ai/claude-agent-sdk
```

### Basic Usage

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const result = await query({
    prompt: "Analyze this codebase",
    apiKey: process.env.ANTHROPIC_API_KEY,
    workingDirectory: process.cwd()
});

console.log(result.messages);
```

### With Hooks (Security Example)

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

await query({
    prompt: "Your task",
    hooks: [
        {
            callback: async (hookInput) => {
                // Block dangerous commands
                if (hookInput.hook_event_name === 'PreToolUse') {
                    if (hookInput.tool_name === 'Bash' &&
                        hookInput.input.command.includes('rm -rf')) {
                        return {
                            deny: true,
                            message: "Blocked: dangerous command",
                            interrupt: true
                        };
                    }
                }
                return { allow: true };
            }
        }
    ]
});
```

### With Custom Tools (MCP)

```typescript
import { query, tool, createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk';

// Define custom tool
const weatherTool = tool(
    'get_weather',
    'Get current weather for a location',
    {
        type: 'object',
        properties: {
            location: { type: 'string', description: 'City name' }
        },
        required: ['location']
    },
    async ({ location }) => {
        // Your implementation
        return { temp: 72, condition: 'sunny' };
    }
);

// Create in-process MCP server
const server = createSdkMcpServer({
    name: 'weather-server',
    version: '1.0.0',
    tools: [weatherTool]
});

// Use in query
await query({
    prompt: "What's the weather in SF?",
    mcpServers: {
        weather: { type: 'sdk', server }
    }
});
```

---

## Common Pitfalls

### 1. Control Methods Require Streaming

**Issue**: `interrupt()`, `setPermissionMode()`, etc. only work with `stream: true`

```typescript
// ✅ WORKS
for await (const msg of query({ stream: true })) {
    msg.setPermissionMode('acceptEdits');
}

// ❌ FAILS - Methods not available
const result = await query({ prompt: "..." });
result.interrupt(); // ERROR!
```

### 2. Bash 10-Minute Hard Timeout

**Issue**: All Bash commands have a hard 10-minute limit

**Solution**: Use background execution for long tasks
```typescript
// Background execution
{ tool: "Bash", input: { command: "npm test", run_in_background: true } }

// Poll output later
{ tool: "BashOutput", input: { bash_id: "shell_id" } }

// Kill if needed
{ tool: "KillShell", input: { shell_id: "shell_id" } }
```

### 3. Permission Escalation Risk

**Issue**: Can programmatically bypass ALL permissions

```typescript
// This bypasses all permission checks!
message.setPermissionMode('bypassPermissions');
```

**Mitigation**: Use PreToolUse hook to prevent mode changes
```typescript
if (hookInput.tool_name === 'setPermissionMode') {
    return { deny: true, message: "Permission mode locked" };
}
```

### 4. Tool Whitelist Not Blacklist

**Issue**: Agent tool restrictions use whitelist approach

```typescript
// ✅ Agent can ONLY use FileRead and Grep
{ tools: ["FileRead", "Grep"] }

// ❌ No way to say "everything except X"
// Must list all allowed tools explicitly
```

### 5. Hook Matcher Syntax Undocumented

**Issue**: `HookCallbackMatcher.matcher` pattern syntax is completely undocumented

**Workaround**: Use callback logic instead
```typescript
// Don't rely on matcher
hooks: [{
    matcher: "???",  // Syntax unknown
    callback: async (input) => {
        // Use logic in callback instead
        if (input.tool_name === 'Bash') { ... }
    }
}]
```

### 6. Compact Boundary Messages

**Issue**: SDK auto-summarizes long conversations, inserting `compact_boundary` messages

**Impact**: Message indices shift; use `message_id` for stable references

```typescript
// ❌ Unstable - indices change after compaction
const msg = messages[5];

// ✅ Stable - message_id persists
const msg = messages.find(m => m.message_id === targetId);
```

### 7. Parent Tool Use Tracking

**Note**: Tool uses create a tree structure via `parent_tool_use_id`

**Impact**: Essential for understanding execution flow and dependencies

```typescript
// Each tool use has:
{
    tool_use_id: "unique_id",
    parent_tool_use_id: "parent_id" | null,  // Creates tree
    tool_name: "Bash",
    input: { ... }
}
```

### 8. Synthetic User Messages

**Note**: System can inject messages with `type: 'user'` that weren't from actual user

**Detection**: Check for synthetic flag or hook context

### 9. FileEdit Requires Unique Strings

**Issue**: FileEdit old_string must be unique in file or edit fails

**Solutions**:
1. Provide larger context to make unique
2. Use `replace_all: true` for global replace
3. Use FileWrite to overwrite entire file

### 10. Grep Default Mode Returns Filenames Only

**Issue**: Default `output_mode` is `files_with_matches` (just paths)

```typescript
// ❌ Returns only filenames
{ tool: "Grep", input: { pattern: "TODO" } }

// ✅ Returns matching lines
{
    tool: "Grep",
    input: {
        pattern: "TODO",
        output_mode: "content",  // Must specify!
        "-n": true  // Show line numbers
    }
}
```

---

## Learning Paths

**Getting Started**:
1. Read the [Comprehensive Guide](./claude-agent-sdk-comprehensive-guide.md) for an overview
2. Explore the [Tool System](./claude-agent-sdk-tools-complete.md) to understand available tools
3. Review [Implementation & Gotchas](./claude-agent-sdk-implementation-gotchas.md) for best practices

**Building Secure Agents**:
1. Start with [Hooks & Permissions](./claude-agent-sdk-hooks-permissions-complete.md)
2. Learn about [Tool Restrictions](./claude-agent-sdk-agents-complete.md)
3. Review security patterns in [Implementation & Gotchas](./claude-agent-sdk-implementation-gotchas.md)

**Understanding Internals**:
1. Read about [System Prompts](./claude-agent-sdk-prompts-complete.md) and agent behavior
2. Explore the [Agent System](./claude-agent-sdk-agents-complete.md) architecture
3. Dive into [CLI Analysis](./cli_analysis.md) for SDK-CLI implementation details
4. Review [Skills System](./CLAUDE-CODE-SKILLS-COMPREHENSIVE-DOCUMENTATION.md) for Claude Code skills

---

## Official Resources

- **GitHub**: https://github.com/anthropics/claude-agent-sdk-typescript
- **Official Docs**: https://docs.claude.com/en/api/agent-sdk/overview
- **Discord**: https://anthropic.com/discord

---

## License

- **SDK**: Copyright © Anthropic PBC, MIT License
- **Documentation**: For educational and reference purposes
