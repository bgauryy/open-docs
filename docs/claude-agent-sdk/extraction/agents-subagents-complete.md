# Claude Agent SDK - Complete Agent System Documentation

**SDK Version**: 0.1.22
**Source**: `sdkTypes.d.ts`, `cli.js`

---

## Table of Contents

1. [Overview](#overview)
2. [System Prompts](#system-prompts)
3. [Agent Architecture](#agent-architecture)
4. [Built-in Agents](#built-in-agents)
5. [Agent Definition Structure](#agent-definition-structure)
6. [Configuring Agents](#configuring-agents)
7. [Context Management](#context-management)
8. [Agent Color System](#agent-color-system)
9. [Async Agent Execution](#async-agent-execution)
10. [Custom Agent Creation](#custom-agent-creation)
11. [Agent Performance & Token Optimization](#agent-performance--token-optimization)
12. [Real-World Patterns](#real-world-patterns)
13. [Internal Implementation Details](#internal-implementation-details)
14. [Gotchas & Best Practices](#gotchas--best-practices)

---

## Overview

The Claude Agent SDK provides a sophisticated multi-agent system that allows you to delegate specialized tasks to purpose-built sub-agents. This enables:

- **Task Specialization**: Dedicated agents for specific workflows
- **Token Efficiency**: Isolated context prevents token waste
- **Parallel Execution**: Async agents for concurrent operations
- **Model Selection**: Different models per agent (Opus, Sonnet, Haiku)
- **Tool Restriction**: Limit agent capabilities for security/performance

### Key Concepts

```
Main Conversation
     │
     ├─► Subagent (Explore) ─► Fast codebase scan
     ├─► Subagent (Plan) ─► Implementation planning
     ├─► Subagent (Bash) ─► Command execution
     ├─► Subagent (claude-code-guide) ─► Documentation queries
     └─► Subagent (general-purpose) ─► Complex task
```

**Benefits**:
- 40-70% token savings with isolated agents
- Faster responses (Haiku for simple tasks)
- Specialized agents for specific workflows
- Better security (tool restrictions)
- Clearer output (agent-specific formatting)

---

## System Prompts

The SDK uses three different system prompts depending on the execution context:

### 1. Standard Claude Code Prompt

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `"You are Claude Code, Anthropic's official CLI for Claude."`

**Usage:** Default for interactive Claude Code CLI sessions

```
You are Claude Code, Anthropic's official CLI for Claude.
```

### 2. SDK Mode Prompt

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `"running within the Claude Agent SDK"`.

**Usage:** When running within Claude Agent SDK (non-interactive)

```
You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.
```

### 3. Agent Mode Prompt

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `"You are a Claude agent, built on Anthropic's Claude Agent SDK."`

**Usage:** For subagents spawned via the Task tool

```
You are a Claude agent, built on Anthropic's Claude Agent SDK.
```

**Selection Logic:**
```javascript
function dM1(A){
  if(p3()==="vertex") return mOA;
  if(A?.isNonInteractive){
    if(A.hasAppendSystemPrompt){
      if(kr()==="claude-vscode") return dOA;
      return Nw9
    }
    return dOA
  }
  return mOA
}
```

---

## Agent Architecture

### Invocation Methods

#### 1. Via Task Tool (Primary)

```typescript
Task({
  subagent_type: "Explore",
  description: "Find auth functions",
  prompt: "Find all authentication functions in the codebase"
})
```

#### 2. Via Skill System

```markdown
<!-- SKILL.md -->
---
name: quick-explore
agent: Explore
---

Find {{$ARGUMENTS}} in the codebase using Glob and Grep.
```

#### 3. Via Slash Commands

```bash
/explore Find all TODO comments
```

### Agent Lifecycle

```
1. Agent Invocation → Task tool called with subagent_type
2. Context Setup → Fork or isolate based on forkContext
3. Model Selection → Use agent model or inherit parent
4. Tool Restriction → Apply allowed/disallowed tools
5. Execution → Agent processes prompt
6. Result Return → Output formatted and returned to parent
7. Context Cleanup → Isolated context discarded
```

---

## Built-in Agents

Built-in agents are provided by the CLI runtime and form the default subagent menu for the `Task` tool.

**Built-in set (v2.1.42, typical CLI entrypoint):**
- `Bash`
- `general-purpose`
- `statusline-setup`
- `Explore` (may be disabled by configuration)
- `Plan`
- `claude-code-guide` (not included for SDK entrypoints)

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `function aGA()` (built-in agent definitions).

### 1. Explore Agent

**Purpose**: Fast codebase exploration and discovery

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `agentType: 'Explore'`.

**Definition**:
```typescript
{
  agentType: "Explore",
  source: "built-in",
  model: "haiku",  // Resolves to claude-3-5-haiku-20241022
  disallowedTools: ["Task", "Edit", "Write", ...],  // Allows: Glob, Grep, Read, Bash
  getSystemPrompt: () => "...",
  whenToUse: "Specialized agent for larger codebase exploration tasks...",
  criticalSystemReminder_EXPERIMENTAL: "CRITICAL: This is a READ-ONLY task..."
}
```

**Characteristics**:
- **Model**: `haiku` by default
- **Context**: Isolated by default
- **Tools**: Read-only and search-oriented (the agent is constrained away from editing/writing)

**When to Use**:
- Initial codebase exploration
- Finding files by pattern
- Searching for specific code patterns
- Quick file content preview
- Directory structure analysis

**Usage Example**:
```typescript
// Basic exploration
Task({
  subagent_type: "Explore",
  description: "Find React components",
  prompt: "Find all React components that use useState"
})

// With a caller-specified thoroughness hint (convention)
Task({
  subagent_type: "Explore",
  description: "Audit API endpoints",
  prompt: "Very thorough: find all API endpoints and their authentication"
})
```

**Thoroughness (v2.1.42):**
- The built-in Explore agent’s instructions ask the caller to specify a thoroughness level (`quick`, `medium`, `very thorough`).
- This is a prompting convention: it is not a separate validated field in the Task tool schema.

---

### 2. general-purpose Agent

**Purpose**: Complex multi-step tasks with full tool access

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `agentType: 'general-purpose'`.

**Definition**:
```typescript
{
  agentType: "general-purpose",
  source: "built-in",
  tools: ["*"],                         // ALL tools available
  whenToUse: "General-purpose agent for researching complex questions...",
  getSystemPrompt: () => "You are an agent for Claude Code..."
  // No model specified - inherits from parent
  // No color specified - no visual distinction
}
```

**Characteristics**:
- **Model**: Inherits from the parent unless overridden (by agent definition or tool call)
- **Context**: Isolated unless the agent definition enables `forkContext`
- **Tools**: `["*"]` (all tools), subject to permission rules and runtime constraints

**When to Use**:
- Complex tasks requiring multiple tools
- Tasks needing broad tool coverage
- Multi-step workflows
- When tool restrictions are too limiting
- Tasks requiring Write/Edit tools

**Usage Example**:
```typescript
// Complex refactoring task
Task({
  subagent_type: "general-purpose",
  description: "Refactor auth",
  prompt: `
    Refactor the authentication system:
    1. Update all auth files to use new token format
    2. Add error handling
    3. Update tests
    4. Document changes
  `,
  // Optionally override model or run in background:
  // model: "sonnet",
  // run_in_background: true
})
```

---

### 3. Bash Agent

**Purpose**: Command execution specialist

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `agentType: 'Bash'`.

**Definition**:
```typescript
{
  agentType: "Bash",
  source: "built-in",
  model: "inherit",                     // Use parent model
  tools: ["Bash"],                      // Only Bash tool
  whenToUse: "Command execution specialist for running bash commands. Use this for git operations, command execution, and other terminal tasks.",
  getSystemPrompt: () => "..."
}
```

**Characteristics**:
- **Model**: Inherits from parent (typically Sonnet)
- **Context**: Isolated
- **Tools**: Bash only
- **Speed**: Variable (depends on command)

**When to Use**:
- Git operations
- Command execution
- Terminal tasks
- System operations

---

### 4. Plan Agent

**Purpose**: Software architect for implementation planning

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `agentType: 'Plan'`.

**Definition**:
```typescript
{
  agentType: "Plan",
  source: "built-in",
  model: "inherit",                     // Use parent model
  disallowedTools: ["Task", "Edit", "Write", ...],  // Same as Explore
  tools: _E.tools,                      // Uses Explore's tool set
  whenToUse: "Software architect agent for designing implementation plans. Use this when you need to plan the implementation strategy for a task. Returns step-by-step plans, identifies critical files, and considers architectural trade-offs.",
  getSystemPrompt: () => "You are a software architect and planning specialist...",
  criticalSystemReminder_EXPERIMENTAL: "CRITICAL: This is a READ-ONLY task..."
}
```

**Characteristics**:
- **Model**: Inherits from parent (typically Sonnet for accuracy)
- **Context**: Isolated (READ-ONLY)
- **Tools**: Same as Explore (Glob, Grep, Read, Bash) - no editing

**When to Use**:
- Planning implementation strategies
- Designing architectural approaches
- Identifying critical files for tasks
- Evaluating trade-offs before implementation

---

### 5. claude-code-guide Agent

**Purpose**: Documentation queries for Claude Code, SDK, and API

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `nGA = 'claude-code-guide'`.

**Definition**:
```typescript
{
  agentType: "claude-code-guide",
  source: "built-in",
  model: "haiku",                       // Fast for doc queries
  tools: ["Glob", "Grep", "Read", "WebFetch", "WebSearch"],
  permissionMode: "dontAsk",            // Never prompt; deny if a prompt would be required
  whenToUse: "Use this agent when the user asks questions (\"Can Claude...\", \"Does Claude...\", \"How do I...\") about: (1) Claude Code (the CLI tool) - features, hooks, slash commands, MCP servers, settings, IDE integrations, keyboard shortcuts; (2) Claude Agent SDK - building custom agents; (3) Claude API (formerly Anthropic API) - API usage, tool use, Anthropic SDK usage.",
  getSystemPrompt: ({toolUseContext}) => "..."
}
```

**Characteristics**:
- **Model**: Haiku (fast, efficient for documentation lookup)
- **Context**: Isolated
- **Tools**: File reading + web access for documentation
- **Special**: `permissionMode: "dontAsk"` - avoids interactive prompts (tools that would require prompting are denied)

**When to Use**:
- Questions about Claude Code features
- Claude Agent SDK usage
- Claude API documentation
- MCP server questions
- Settings and configuration help

---

### 6. statusline-setup Agent

**Purpose**: Configure terminal status line settings

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `agentType: 'statusline-setup'`.

**Definition**:
```typescript
{
  agentType: "statusline-setup",
  source: "built-in",
  model: "sonnet",                      // Resolves to claude-3-5-sonnet-20241022
  tools: ["Read", "Edit"],              // Config files only
  color: "orange",
  whenToUse: "Use this agent to configure the user's Claude Code status line setting.",
  getSystemPrompt: () => "You are a status line setup agent..."
}
```

**Characteristics**:
- **Model**: Sonnet (accuracy for config files)
- **Context**: Isolated
- **Tools**: Read and Edit only (safe, limited scope)

**When to Use**:
- Initial terminal setup
- Configure Claude Code UI settings
- Update status line configuration

**Usage Example**:
```typescript
Task({
  subagent_type: "statusline-setup",
  description: "Configure status line",
  prompt: "Configure status line for my terminal: ghostty"
})
```

---

### Removed/Plugin Agents

The following agent names are sometimes referenced in older writeups, but are not guaranteed to be built-in in v2.1.42:

#### output-style-setup (Not a Built-in Agent)
- **Status**: Not found as built-in agent
- **Likely**: Plugin feature or deprecated
- `output-styles` appears as a plugin configuration option, not an agent

#### security-review (Plugin Agent, Not Built-in)
- `security-review` is not part of the default built-in set returned by the CLI runtime.
- If installed as a plugin, agent types are typically **namespaced** by plugin name and file path (e.g., `security-review:agent-name`).

---

## Agent Definition Structure

### Complete TypeScript Definition

**⚠️ Note**: The actual implementation uses different field names than TypeScript definitions. Below shows the **runtime implementation** structure:

```typescript
type AgentDefinition = {
  // Identification
  agentType: string;                    // Unique agent identifier
  source: AgentSource;                  // Where agent is defined
  baseDir?: string;                     // Agent base directory

  // Execution Configuration
  model?: string | 'inherit';           // Model shorthand (e.g., "haiku", "sonnet") or inherit

  // Tool Access (multiple patterns)
  tools?: string[];                     // Tool whitelist (["*"] = all)
  disallowedTools?: string[];           // Tool blacklist (alternative to tools)

  // System Prompt
  getSystemPrompt: (context?: any) => string;  // Function returning system prompt
  whenToUse?: string;                   // Description of when to use this agent

  // Visual Identification
  color?: string;                       // UI color (e.g., "orange", not "_FOR_SUBAGENTS_ONLY")

  // Special Behaviors
  permissionMode?: string;              // e.g., "dontAsk" to avoid prompts (deny if a prompt would be required)
  criticalSystemReminder_EXPERIMENTAL?: string;  // Extra system reminder

  // Plugin Information (if applicable)
  plugin?: string;                      // Plugin name
  filename?: string;                    // Agent definition file
};

type AgentSource =
  | 'built-in'           // SDK-provided agents
  | 'userSettings'       // User's ~/.claude/
  | 'projectSettings'    // Project .claude/
  | 'policySettings'     // Enterprise policy
  | 'plugin'             // Plugin-provided
  | 'flagSettings';      // CLI flag override

type AgentColor = string;  // Simple string, e.g., "red", "blue", "orange"
// Note: Runtime uses plain color names, not "_FOR_SUBAGENTS_ONLY" suffix
```

### Field Explanations

**agentType**:
- Unique identifier for agent
- Used in Task tool: `subagent_type: "Explore"`
- Convention: lowercase with hyphens

**source**:
- Origin of agent definition
- Determines precedence for conflicts
- Later sources can override earlier ones for the same `agentType` (see “Agent Definition Sources and Precedence” below)

**model**:
- Model shorthand: `"haiku"`, `"sonnet"`, `"opus"`, or `"inherit"`
- `"inherit"` uses parent conversation model
- Shorthands resolve to full model IDs (e.g., "haiku" → "claude-3-5-haiku-20241022")
- Optional field (omission implies inherit)

**tools**:
- Array of tool names: `["Read", "Write", "Bash"]`
- Wildcard: `["*"]` (all tools)
- The CLI runtime parses a `ToolName(pattern)` form, but in v2.1.42 the pattern is only used for the `Task(...)` tool to restrict allowed agent types (other tools treat `ToolName(pattern)` like `ToolName`)
- Mutually exclusive with `disallowedTools`

**disallowedTools**:
- Array of tool names to prohibit
- Used by Explore and Plan agents
- All non-listed tools are allowed
- Mutually exclusive with `tools`

**getSystemPrompt**:
- Function returning the agent's system prompt
- Can accept context parameter
- Executed at agent invocation time

**whenToUse**:
- String describing when to use this agent
- Helps users/main agent select appropriate agent
- Shown in Task tool documentation

**color**:
- Visual identifier in UI
- Simple string: `"orange"`, `"blue"`, `"red"`, etc.
- No suffix required (not `"_FOR_SUBAGENTS_ONLY"`)
- Optional (assigned automatically if not specified)

---

## Configuring Agents

### Options Configuration

Agents are configured through the `Options` type:

```typescript
export type Options = Omit<BaseOptions, 'customSystemPrompt' | 'appendSystemPrompt'> & {
    agents?: Record<string, AgentDefinition>;
    settingSources?: SettingSource[];
    systemPrompt?: string | {
        type: 'preset';
        preset: 'claude_code';
        append?: string;
    };
};
```

### Claude Code CLI Agent Sources and Precedence (v2.1.42)

The Claude Code CLI runtime assembles `activeAgents` from multiple sources:

1. **Built-in agents** (shipped with the CLI)
2. **Plugin agents** (loaded from plugin `agents/` directories and configured agent paths)
3. **Agent files** (`.md` files discovered under):
   - Project: `.claude/agents/` (searched up the directory tree from the current working directory)
   - Personal: `~/.claude/agents/`
   - Policy-managed: a policy settings root (enterprise)
4. **Flag-defined agents** (JSON passed via CLI flags, merged after file/plugin sources)

Agents are deduplicated by `agentType`, with later sources overriding earlier ones. In the CLI runtime, the precedence order is:

```text
built-in < plugin < userSettings < projectSettings < flagSettings < policySettings
```

**Source (v2.1.42)**:
- Search: `function aGA()` (built-in agents)
- Search: `oK1 = zA(async () => {` (plugin agents)
- Search: `hp = zA(async function (A, q)` and `join(..., ".claude", "agents")` (agent file discovery)
- Search: `activeAgents: YI(` (dedup + merge)

### SDK API AgentDefinition (Public)

When using the SDK's `query()` function, use this structure:

```typescript
export type AgentDefinition = {
    description: string;       // Human-readable description
    tools?: string[];          // Optional array of allowed tool names
    prompt: string;            // Custom system prompt
    model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
```

### Setting Up Agents

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const response = await query({
    prompt: "Help me analyze this codebase",
    options: {
        agents: {
            // Code analysis specialist
            code_analyzer: {
                description: "Specialized agent for code analysis",
                tools: ["Grep", "Glob", "Read", "Bash"],
                prompt: "You are a code analysis expert. Focus on finding patterns, analyzing structure, and providing insights.",
                model: "sonnet"
            },

            // Documentation specialist
            doc_writer: {
                description: "Specialized agent for writing documentation",
                tools: ["Read", "Write", "Grep"],
                prompt: "You are a technical writer. Create clear, comprehensive documentation with examples.",
                model: "sonnet"
            },

            // Testing specialist
            test_runner: {
                description: "Specialized agent for running tests",
                tools: ["Bash", "Read", "Grep"],
                prompt: "You are a testing specialist. Run tests, analyze results, and report failures clearly.",
                model: "haiku"  // Faster model for quick test runs
            }
        }
    }
});
```

### Tool Restrictions

#### Available Tools
- `Task` - Delegate to subagents
- `Bash` - Execute shell commands
- `BashOutput` - Read background shell output
- `Read` - Read files
- `Write` - Write files
- `Edit` - Edit files
- `Glob` - Find files by pattern
- `Grep` - Search file contents
- `WebFetch` - Fetch web content
- `WebSearch` - Search the web
- `TodoWrite` - Manage todo lists
- `NotebookEdit` - Edit Jupyter notebooks
- `KillShell` - Kill background shells
- `Mcp` - MCP tool calls
- `ListMcpResources` - List MCP resources
- `ReadMcpResource` - Read MCP resources

#### Tool Restriction Strategy

```typescript
// Example: Research agent (read-only)
research_agent: {
    description: "Research specialist with read-only access",
    tools: ["Grep", "Glob", "Read", "WebSearch", "WebFetch"],
    prompt: "Research and gather information without modifying files",
    model: "sonnet"
}

// Example: Safe executor (limited modification)
safe_executor: {
    description: "Execute commands with file editing",
    tools: ["Bash", "Read", "Edit"],  // No Write (safer)
    prompt: "Execute commands and make targeted edits only",
    model: "sonnet"
}

// Example: Full access agent
full_access: {
    description: "Full capability agent",
    // tools omitted = inherits all tools
    prompt: "Handle complex tasks with full tool access",
    model: "opus"
}
```

---

## Context Management

### Forked Context (forkContext: true)

`forkContext` is a property on the **agent definition** (not a per-call parameter). When enabled, the subagent receives the main thread’s prior messages as context.

**Behavior (v2.1.42):**
- Subagent receives main-thread context messages via the Task tool runner.
- The runtime injects a “sub-agent entered” marker message to make the boundary explicit.
- Tool availability is still governed by the subagent’s resolved tool list and permission mode.

**Impact:** Forked-context agents can be easier to prompt (you can reference earlier work), but they may increase context size and cost depending on the session history.

**Use Cases**:
- Agent needs conversation context
- Multi-step tasks requiring history
- Tasks referencing previous work
- Context-dependent decisions

**Example:** Prefer a dedicated custom agent configured with `forkContext: true` for workflows that repeatedly reference prior context.

---

### Isolated Context (forkContext: false)

When `forkContext` is absent/false, each Task invocation starts with fresh context (the subagent only sees the prompt you provide).

**Use Cases**:
- Independent tasks (exploration, search)
- Token optimization
- Fast simple operations
- No context needed

**Example**:
```typescript
// Agent has no context, must be explicit
Task({
  subagent_type: "Explore",
  description: "Find auth files",
  prompt: "Find all files containing 'authentication' in src/auth/"
})
```

**Important**: Isolated agents require complete, self-contained prompts

---

## Agent Color System

### Color Model (v2.1.42)

Claude Code’s UI uses a small fixed palette of base colors:

```text
red, blue, green, yellow, purple, orange, pink, cyan
```

In UI rendering, these are mapped to internal style tokens suffixed with `_FOR_SUBAGENTS_ONLY` (for example, `orange` → `orange_FOR_SUBAGENTS_ONLY`).

**Characteristics**:
- 8 base colors are supported
- Agent definitions may specify a color by name (e.g., `orange`)
- The runtime maps recognized color names to internal subagent color tokens

**Notes:**
- `general-purpose` intentionally returns no subagent color token.
- In team UI contexts, agents may also be assigned colors sequentially per agent ID.

**Source (v2.1.42)**:
- Search: `rO = { red: "red_FOR_SUBAGENTS_ONLY",` (color token mapping)
- Search: `function jc(A) {` and `let K = nO[` (sequential color assignment)

### UI Rendering

Colors are used for:
- Agent name display in output
- Progress indicators
- Error messages from agent
- Result formatting

**Example Output**:
```
[blue] Explore agent: Starting codebase scan...
[blue] Found 23 matching files
[blue] Result: See attached file list
```

---

## Async Agent Execution

### Synchronous Agents (Default)

**Behavior**:
```
User → Task → Agent starts → [WAIT] → Agent completes → Result returned
```

**Characteristics**:
- Blocks until completion
- Result immediately available
- Typical for most use cases

**Example**:
```typescript
const result = Task({
  subagent_type: "Explore",
  description: "Find auth files",
  prompt: "Find authentication files"
});
// Returns a "completed" result payload (see Output Shapes below)
```

---

### Background Agents (run_in_background: true)

**Behavior**:
```
User → Task(run_in_background=true) → [IMMEDIATE RETURN: outputFile] → Agent continues in background
Later → Read/tail outputFile to check progress and final output
```

**Characteristics**:
- Returns immediately with an `outputFile` path
- Agent continues in the background
- Progress and final output are written to the output file

**Use Cases**:
- Long-running analysis
- Parallel agent execution
- Non-blocking workflows
- Background processing

**Example**:
```typescript
// Start background agent
const launched = Task({
  subagent_type: "general-purpose",
  description: "Background audit",
  prompt: "Run a thorough audit of the current changes and summarize risks",
  run_in_background: true
});
// launched.status === "async_launched"
// launched.outputFile is the file path to read for progress/results
```

**Checking progress/results (CLI runtime):**
- Read the output file with `Read({ file_path: launched.outputFile })`
- Or tail it via `Bash({ command: "tail -n 50 <outputFile>" })`

**Availability notes (v2.1.42):**
- Background agents can be disabled via environment/config (the runtime may omit `run_in_background` from the input schema).
- In “in-process teammate” contexts, background agents are rejected; use `run_in_background: false`.

---

### Resuming Agents (resume)

The Task tool supports a `resume` parameter that continues a prior agent by agent ID, preserving that agent’s previous transcript/context.

```typescript
const first = Task({
  subagent_type: "general-purpose",
  description: "Initial analysis",
  prompt: "Analyze the auth module and propose improvements"
});

const followUp = Task({
  subagent_type: "general-purpose",
  description: "Follow-up",
  prompt: "Now implement the top 2 improvements you proposed",
  resume: first.agentId
});
```

**Runtime notes (v2.1.42):**
- Resuming a still-running background agent is rejected; stop it or wait for completion first.

---

## Custom Agent Creation

### Method 1: Personal agent files

**Location:** `~/.claude/agents/*.md`

Agent definitions are loaded from Markdown files with YAML frontmatter. The file name (or `name` in frontmatter) becomes the agent identifier.

Example: `~/.claude/agents/code-reviewer.md`

```markdown
---
name: code-reviewer
description: "Review code changes for project standards"
model: sonnet
tools:
  - Read
  - Grep
  - Bash
permissionMode: default
forkContext: "false"
color: purple
---

You are a strict code reviewer. Focus on correctness, security, and project conventions.
Return a concise pass/fail summary with actionable fixes.
```

### Method 2: Project agent files

**Location:** `<project-root>/.claude/agents/*.md`

Project agents work the same way as personal agents, but are loaded from the nearest `.claude/agents/` directories while walking up from the current working directory.

### Method 3: Plugin-provided agents

Plugins can ship agents in an `agents/` directory (and/or explicitly configured agent paths). Plugin agents are **namespaced** by plugin and path to avoid collisions (for example: `my-plugin:subdir:agent-name`).

### Method 4: Programmatic (SDK)

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

const result = await query({
  prompt: "Analyze the codebase",
  options: {
    agents: {
      "analyzer": {
        description: "Code analysis agent",
        tools: ["Read", "Grep", "Glob"],
        prompt: "Analyze code quality and suggest improvements",
        model: "sonnet"
      }
    }
  }
});
```

**Note:** This SDK `options.agents` surface is distinct from Claude Code’s CLI agent file system (`.claude/agents/*.md`).

### SDK Agent Definition Schema (public)

```typescript
type AgentDefinition = {
  description: string;         // Human-readable description
  tools?: string[];            // Allowed tools (default: all)
  prompt: string;              // System prompt for agent
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
```

---

## Agent Performance & Token Optimization

### What affects performance (v2.1.42)

- **Model**: `haiku` is typically faster/cheaper; `sonnet`/`opus` can improve reasoning quality at higher cost/latency.
- **Context mode**: `forkContext: true` can increase context size because the subagent receives prior messages.
- **Tool set**: narrower tool access reduces what the agent can do (and can reduce permission overhead), but may require more user-provided context.
- **Backgrounding**: `run_in_background: true` lets you continue while the agent writes progress to `outputFile`.

### Practical guidelines

- Prefer direct tools (`Read`, `Grep`, `Glob`) when you already know what you need to do.
- Use `Explore` for open-ended codebase discovery that would require many Grep/Read passes.
- Use `Plan` for planning-only work (read-only constraints).
- Use `general-purpose` when you need broad tool coverage and multi-step autonomy.

---

## Real-World Patterns

### Pattern 1: Two-stage workflow (Explore → general-purpose)

```typescript
const files = Task({
  subagent_type: "Explore",
  description: "Find API routes",
  prompt: "Find all API route files"
});

const analysis = Task({
  subagent_type: "general-purpose",
  description: "Analyze API security",
  prompt: `Analyze these API files for security issues:\n\n${JSON.stringify(files)}`
});
```

### Pattern 2: Background long-running work

```typescript
const launched = Task({
  subagent_type: "general-purpose",
  description: "Background report",
  prompt: "Generate a detailed report of recent changes and risks",
  run_in_background: true
});

// Later:
// Read({ file_path: launched.outputFile })
```

### Pattern 3: Custom agent for a repeated workflow

```typescript
Task({
  subagent_type: "code-reviewer",
  description: "Review branch",
  prompt: "Review current branch vs main and report issues"
});
```

---

## Internal Implementation Details

### Agent Tool (Task Tool)

**Internal Name:** `Task` (CLI) vs `Agent` (SDK API)

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `var ZK = 'Task'`.

### AgentInput Interface

The Claude Code CLI runtime tool schema is richer than the minimal `AgentInput` shape often shown in older SDK docs.

**Task tool input schema (v2.1.42 runtime):**

```typescript
type TaskToolInput = {
  // Required
  description: string;   // short (3–5 words)
  prompt: string;        // task instructions
  subagent_type: string; // agent identifier (e.g. "Explore")

  // Optional
  model?: "sonnet" | "opus" | "haiku";
  resume?: string;               // resume a prior agentId
  run_in_background?: boolean;   // background execution (if enabled)
  max_turns?: number;            // internal/warmup control

  // Team/teammate spawning (when team context is enabled)
  name?: string;
  team_name?: string;
  mode?: "default" | "acceptEdits" | "bypassPermissions" | "delegate" | "dontAsk" | "plan";
};
```

**Task tool output shapes (v2.1.42 runtime):**
- `status: "completed"` → includes `agentId`, `content[]`, token/tool-use metrics, and `usage`
- `status: "async_launched"` → includes `agentId` and `outputFile` for progress/results
- `status: "teammate_spawned"` → includes teammate metadata when spawning into a team context

**Source (v2.1.42)**: Search `@anthropic-ai/claude-code/cli.js` for `subagent_type: x.string()`, `run_in_background`, and `outputFile`.

### Internal Agent Execution Flow

At a high level, the runtime does the following:

1. **Load/merge agent definitions** from built-ins, plugins, and `.claude/agents/*.md` files, then deduplicate by `agentType`.
2. **Apply permissions** to restrict available agent types and tools (both via permission rules and agent `tools`/`disallowedTools`).
3. **Select an agent definition** matching `subagent_type` (and validate required MCP servers, if any).
4. **Resolve model** (agent definition model + optional tool-call override).
5. **Build the subagent system prompt** (`agentDefinition.getSystemPrompt({ toolUseContext })`).
6. **Decide context mode**:
   - `forkContext: true` → include main-thread messages and inject a boundary marker message
   - otherwise → start from only the provided prompt
7. **Execute** synchronously or in the background (`run_in_background`), returning one of the output shapes described above.

**Primary anchors (v2.1.42):**
- Built-in agent list: Search `function aGA()`
- Agent loading/dedup: Search `YF1 =` / `activeAgents: YI(`
- Agent file discovery: Search `hp =` / `join(..., ".claude", "agents")`
- Plugin agents: Search `oK1 =` / `agentsPath`
- Task tool runner: Search `name: ZK` / `subagent_type` / `outputFile`
- Tool/agent restrictions: Search `allowedAgentTypes`

### Agent Architect Prompt

**Source (v2.1.42)**: In `@anthropic-ai/claude-code/cli.js`, search for `"You are an elite AI agent architect"`.

**Purpose:** Used by Claude to generate new agent definitions

```javascript
`You are an elite AI agent architect specializing in crafting high-performance agent configurations...

When a user describes what they want an agent to do, you will:

1. **Extract Core Intent**: Identify the fundamental purpose, key responsibilities, and success criteria...
2. **Design Expert Persona**: Create a compelling expert identity...
3. **Architect Comprehensive Instructions**: Develop a system prompt...
4. **Optimize for Performance**: Include decision-making frameworks...
5. **Create Identifier**: Design a concise, descriptive identifier...
6. **Example agent descriptions**: Include examples of when this agent should be used...

Your output must be a valid JSON object with exactly these fields:
{
  "identifier": "unique-agent-identifier",
  "whenToUse": "Use this agent when... [includes examples]",
  "systemPrompt": "You are... [complete system prompt]"
}
```

### Session Information

Agents are tracked in system messages:

```typescript
export type SDKSystemMessage = SDKMessageBase & {
    type: 'system';
    subtype: 'init';
    agents?: string[];  // List of available agent types
    apiKeySource: ApiKeySource;
    claude_code_version: string;
    cwd: string;
    tools: string[];
    mcp_servers: { name: string; status: string; }[];
    model: string;
    permissionMode: PermissionMode;
    slash_commands: string[];
    output_style: string;
};
```

### Hook Integration

Claude Code emits hook events around subagent lifecycle:
- `SubagentStart`
- `SubagentStop`

See `open-docs/docs/claude-agent-sdk/hooks-permissions-complete.md` for the complete event payloads and output semantics.

**Source (v2.1.42)**: Search `@anthropic-ai/claude-code/cli.js` for `hook_event_name: "SubagentStart"` and `hook_event_name: "SubagentStop"`.

### Model Usage Tracking

```typescript
export type SDKResultMessage = {
    type: 'result';
    subtype: 'success';
    // ...
    modelUsage: {
        [modelName: string]: ModelUsage;
    };
    // ...
};

export type ModelUsage = {
    inputTokens: number;
    outputTokens: number;
    cacheReadInputTokens: number;
    cacheCreationInputTokens: number;
    webSearchRequests: number;
    costUSD: number;
    contextWindow: number;
};
```

### Tool Name Constants (v2.1.42)

| Tool | Constant | Value | Source Line | Notes |
|------|----------|-------|-------------|-------|
| Grep | `e3` | "Grep" | ~212 | File content search |
| Glob | `PY` | "Glob" | ~212 | File pattern matching |
| Read | `_q` | "Read" | ~212 | Read files |
| Write | `G5` | "Write" | ~212 | Write files |
| Edit | `bq` | "Edit" | ~212 | Edit files |
| Bash | `I4` | "Bash" | ~212 | Execute commands |
| Task | `ZK` | "Task" | 153330 | Agent invocation |
| WebFetch | `mO` | "WebFetch" | 152633 | Fetch web content |
| WebSearch | `xL` | "WebSearch" | 153406 | Search the web |

---

## Gotchas & Best Practices

### Gotchas

1. **Isolated agents have no main-thread context by default**:
   ```typescript
   // ❌ Bad: unclear reference
   Task({
     subagent_type: "Explore",
     description: "Find errors",
     prompt: "Search that file for errors"
   });

   // ✅ Good: explicit target
   Task({
     subagent_type: "Explore",
     description: "Scan login errors",
     prompt: "Search src/auth/login.ts for error handling"
   });
   ```

2. **Agent Color Collision** (8 colors only):
   - If you have 10+ custom agents, colors will repeat
   - Rely on agent name, not just color

3. **Tool Restrictions Strictly Enforced**:
   ```typescript
   // Explore agent tries to use Write tool
   // ❌ Error: Tool "Write" not allowed for agent "Explore"
   ```

4. **Background agents write to an output file**:
   - When `run_in_background: true`, the Task tool returns `outputFile`
   - Read/tail `outputFile` to retrieve progress and final output

5. **Model Inheritance Can Be Expensive**:
   ```typescript
   // Prefer explicit model selection when cost/latency matters
   Task({
     subagent_type: "general-purpose",
     description: "Quick check",
     prompt: "Summarize the change",
     model: "haiku"
   });
   ```

6. **general-purpose Agent No Color**:
   - No visual distinction in UI
   - Can be confusing if used frequently

### Best Practices

**1. Choose the Right Agent**:
```typescript
// ✅ Use Explore for discovery
Task({ subagent_type: "Explore", description: "Find files", prompt: "Find relevant files for auth" })

// ✅ Use general-purpose for complex tasks
Task({ subagent_type: "general-purpose", description: "Refactor auth", prompt: "Refactor auth system..." })
```

**2. Provide Complete Prompts for Isolated Agents**:
```typescript
Task({
  subagent_type: "Explore",
  description: "Scan components",
  prompt: `
    Find all React components in src/components/ that:
    - Use useState hook
    - Have more than 200 lines
    - Don't have TypeScript types
    
    Return: File paths with line counts
  `
});
```

**3. Specify an output format inside the prompt**:
```typescript
Task({
  subagent_type: "general-purpose",
  description: "Audit branch",
  prompt: `
    Audit current branch.

    Format:
    Risk Level: HIGH/MEDIUM/LOW
    Issues Found: [list]
    Recommendations: [list]
  `
});
```

**4. Use background mode for long operations**:
```typescript
const launched = Task({
  subagent_type: "general-purpose",
  description: "Long audit",
  prompt: "Complete a thorough audit of the codebase and summarize risks",
  run_in_background: true
});

// Later: Read({ file_path: launched.outputFile })
```

**5. Combine Agent Types for Workflow**:
```typescript
// 1. Explore (fast discovery)
const files = Task({ subagent_type: "Explore", description: "Find entrypoints", prompt: "Find entrypoints and key modules" });

// 2. general-purpose (detailed work)
const analysis = Task({ subagent_type: "general-purpose", description: "Deep analysis", prompt: `Analyze based on:\n${JSON.stringify(files)}` });
```

---

## Summary

### Agent Selection Matrix

| Task Type | Agent | Why |
|-----------|-------|-----|
| Find files/code | Explore | Fast Haiku, isolated, read-only file tools |
| Implementation planning | Plan | Isolated, read-only, same tools as Explore |
| Command execution | Bash | Specialized for terminal operations |
| Documentation queries | claude-code-guide | Fast Haiku, web access, documentation focus |
| Complex refactoring | general-purpose | Full tools, all capabilities |
| Configuration | statusline-setup | Minimal tools, config-specific |
| Custom workflow | Custom agent | Tailored tools and prompt |

### Key Takeaways

- Built-in agents are loaded in the CLI runtime and can be combined with custom agents from `.claude/agents/*.md` and plugin agents.
- Task tool inputs use `subagent_type`, `description`, and `prompt` (plus optional overrides like `model`, `resume`, `run_in_background`).
- Prefer explicit prompts for isolated agents; use forked-context agents only when you truly need main-thread history.
- `dontAsk` avoids interactive permission prompts (it does not “auto-approve”).
