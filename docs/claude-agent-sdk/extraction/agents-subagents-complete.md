# Claude Agent SDK - Complete Agent System Documentation

**SDK Version**: 0.1.22
**Source**: `sdkTypes.d.ts`, `cli.js`

---

## Table of Contents

1. [Overview](#overview)
2. [System Prompts](#system-prompts)
3. [Agent Architecture](#agent-architecture)
4. [Built-in Agents (All 5)](#built-in-agents-all-5)
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
     ├─► Subagent (security-review) ─► Security audit
     └─► Subagent (general-purpose) ─► Complex task
```

**Benefits**:
- 40-60% token savings with isolated agents
- Faster responses (Haiku for simple tasks)
- Better security (tool restrictions)
- Clearer output (agent-specific formatting)

---

## System Prompts

The SDK uses three different system prompts depending on the execution context:

### 1. Standard Claude Code Prompt
**Source:** cli.js:285 (variable `mOA`)  
**Usage:** Default for interactive Claude Code CLI sessions

```
You are Claude Code, Anthropic's official CLI for Claude.
```

### 2. SDK Mode Prompt
**Source:** cli.js:285 (variable `Nw9`)  
**Usage:** When running within Claude Agent SDK (non-interactive)

```
You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.
```

### 3. Agent Mode Prompt
**Source:** cli.js:285 (variable `dOA`)  
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
  agent_type: "Explore",
  prompt: "Find all authentication functions in the codebase",
  expected_output: "List of files and function names"
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
1. Agent Invocation → Task tool called with agent_type
2. Context Setup → Fork or isolate based on forkContext
3. Model Selection → Use agent model or inherit parent
4. Tool Restriction → Apply allowed/disallowed tools
5. Execution → Agent processes prompt
6. Result Return → Output formatted and returned to parent
7. Context Cleanup → Isolated context discarded
```

---

## Built-in Agents (All 5)

### 1. Explore Agent

**Purpose**: Fast codebase exploration and discovery

**Definition**:
```typescript
{
  agentType: "Explore",
  source: "built-in",
  model: "claude-3-5-haiku-20241022",  // Fast & cheap
  forkContext: false,                   // Isolated context
  isAsync: false,                       // Synchronous
  allowedTools: ["Glob", "Grep", "Read", "Bash"],
  color: "blue_FOR_SUBAGENTS_ONLY"
}
```

**Characteristics**:
- **Model**: Haiku (fastest, most cost-effective)
- **Context**: Isolated (no parent conversation history)
- **Tools**: File discovery only (Glob, Grep, Read, Bash)
- **Speed**: ~5-15 seconds typical execution
- **Token Usage**: 40-60% less than main agent

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
  agent_type: "Explore",
  prompt: "Find all React components that use useState",
  expected_output: "List of component files with line numbers"
})

// With thoroughness level
Task({
  agent_type: "Explore",
  prompt: "very thorough: Find all API endpoints and their authentication",
  expected_output: "Complete list with authentication methods"
})
```

**Thoroughness Levels**:
```typescript
const THOROUGHNESS_PATTERNS = {
  quick: /\bquick\b/i,           // Fast, surface-level scan
  medium: /\bmedium\b/i,         // Balanced approach (default)
  thorough: /\b(very )?thorough\b/i  // Deep, comprehensive search
};
```

**Token Efficiency**:
```
Main agent (with full context): ~50,000 tokens
Explore agent (isolated): ~15,000 tokens
Savings: 70% reduction in context tokens
```

---

### 2. general-purpose Agent

**Purpose**: Complex multi-step tasks with full tool access

**Definition**:
```typescript
{
  agentType: "general-purpose",
  source: "built-in",
  model: "inherit",                     // Use parent model
  forkContext: true,                    // Includes parent context
  isAsync: false,                       // Synchronous
  allowedTools: ["*"],                  // ALL tools available
  color: undefined                      // No color (not visually distinguished)
}
```

**Characteristics**:
- **Model**: Inherits from parent (typically Sonnet)
- **Context**: Forked (includes full conversation history)
- **Tools**: Unrestricted access to all tools
- **Speed**: ~30-120 seconds typical execution
- **Token Usage**: Similar to main agent (no savings)

**When to Use**:
- Complex tasks requiring multiple tools
- Tasks needing conversation context
- Multi-step workflows
- When tool restrictions are too limiting
- Tasks requiring Write/Edit tools

**Usage Example**:
```typescript
// Complex refactoring task
Task({
  agent_type: "general-purpose",
  prompt: `
    Refactor the authentication system:
    1. Update all auth files to use new token format
    2. Add error handling
    3. Update tests
    4. Document changes
  `,
  expected_output: "Summary of changes with file list"
})
```

**Context Impact**:
```
Parent context: 50,000 tokens
Forked agent: 50,000 + agent output (~5,000) = 55,000 tokens
No token savings, but better organization
```

---

### 3. statusline-setup Agent

**Purpose**: Configure terminal status line settings

**Definition**:
```typescript
{
  agentType: "statusline-setup",
  source: "built-in",
  model: "claude-3-5-sonnet-20241022",
  forkContext: false,                   // Isolated
  isAsync: false,
  allowedTools: ["Read", "Edit"],       // Config files only
  color: "green_FOR_SUBAGENTS_ONLY"
}
```

**Characteristics**:
- **Model**: Sonnet (accuracy for config files)
- **Context**: Isolated
- **Tools**: Read and Edit only (safe, limited scope)
- **Speed**: ~10-20 seconds
- **Token Usage**: Minimal

**When to Use**:
- Initial terminal setup
- Configure Claude Code UI settings
- Update status line configuration

**Usage Example**:
```typescript
Task({
  agent_type: "statusline-setup",
  prompt: "Configure status line for my terminal: ghostty",
  expected_output: "Status line configuration complete"
})
```

---

### 4. output-style-setup Agent

**Purpose**: Create and configure custom output styles

**Definition**:
```typescript
{
  agentType: "output-style-setup",
  source: "built-in",
  model: "claude-3-5-sonnet-20241022",
  forkContext: false,                   // Isolated
  isAsync: false,
  allowedTools: ["Read", "Write", "Edit", "Glob", "Grep"],
  color: "yellow_FOR_SUBAGENTS_ONLY"
}
```

**Characteristics**:
- **Model**: Sonnet
- **Context**: Isolated
- **Tools**: File operations for style creation
- **Speed**: ~15-30 seconds
- **Token Usage**: Low

**When to Use**:
- Create custom output formatting
- Configure response styles
- Setup project-specific output templates

**Usage Example**:
```typescript
Task({
  agent_type: "output-style-setup",
  prompt: "Create a concise output style for code reviews",
  expected_output: "Output style created and configured"
})
```

---

### 5. security-review Agent

**Purpose**: Security audit of code changes (git branch)

**Definition**:
```typescript
{
  agentType: "security-review",
  source: "built-in",
  model: "claude-3-5-sonnet-20241022",
  forkContext: false,                   // Isolated
  isAsync: false,
  allowedTools: [
    "Bash(git diff:*)",           // Git commands only
    "Bash(git status:*)",
    "Bash(git log:*)",
    "Bash(git show:*)",
    "Bash(git remote show:*)",
    "Read",
    "Glob",
    "Grep",
    "LS",
    "Task"
  ],
  color: "red_FOR_SUBAGENTS_ONLY"
}
```

**Characteristics**:
- **Model**: Sonnet (accuracy for security)
- **Context**: Isolated
- **Tools**: Git-restricted Bash + file reading
- **Speed**: ~30-60 seconds (depends on changes)
- **Token Usage**: Medium (analyzing diffs)

**Special Feature**: Bash tool is restricted to git commands only
```typescript
// Allowed
Bash({ command: "git diff main" })
Bash({ command: "git log -10" })

// Blocked (not git commands)
Bash({ command: "npm install" })  // ❌ Blocked
Bash({ command: "rm -rf /" })      // ❌ Blocked
```

**When to Use**:
- Pre-commit security review
- Branch change analysis
- Vulnerability scanning
- Code audit before merge

**Usage Example**:
```typescript
Task({
  agent_type: "security-review",
  prompt: "Review security implications of changes in current branch vs main",
  expected_output: "Security assessment with risk level and recommendations"
})
```

---

## Agent Definition Structure

### Complete TypeScript Definition

```typescript
type AgentDefinition = {
  // Identification
  agentType: string;                    // Unique agent identifier
  source: AgentSource;                  // Where agent is defined
  
  // Execution Configuration
  model: string | 'inherit';            // Model ID or inherit from parent
  forkContext?: boolean;                // Include parent context (default: false)
  isAsync?: boolean;                    // Background execution (default: false)
  
  // Tool Access
  allowedTools: string[];               // Tool whitelist (["*"] = all)
  
  // Visual Identification
  color?: AgentColor;                   // UI color for agent
  
  // Plugin Information (if applicable)
  plugin?: string;                      // Plugin name
  baseDir?: string;                     // Plugin base directory
  filename?: string;                    // Agent definition file
};

type AgentSource = 
  | 'built-in'           // SDK-provided agents
  | 'userSettings'       // User's ~/.claude/
  | 'projectSettings'    // Project .claude/
  | 'policySettings'     // Enterprise policy
  | 'plugin'             // Plugin-provided
  | 'flagSettings';      // CLI flag override

type AgentColor = 
  | "red_FOR_SUBAGENTS_ONLY"
  | "blue_FOR_SUBAGENTS_ONLY"
  | "green_FOR_SUBAGENTS_ONLY"
  | "yellow_FOR_SUBAGENTS_ONLY"
  | "purple_FOR_SUBAGENTS_ONLY"
  | "orange_FOR_SUBAGENTS_ONLY"
  | "pink_FOR_SUBAGENTS_ONLY"
  | "cyan_FOR_SUBAGENTS_ONLY";
```

### Field Explanations

**agentType**:
- Unique identifier for agent
- Used in Task tool: `agent_type: "Explore"`
- Convention: lowercase with hyphens

**source**:
- Origin of agent definition
- Determines precedence for conflicts
- `built-in` cannot be overridden

**model**:
- Anthropic model ID or `"inherit"`
- `"inherit"` uses parent conversation model
- Supports: opus, sonnet, haiku variants

**forkContext**:
- `true`: Include parent conversation history (context-aware)
- `false`: Start fresh (isolated, token-efficient)
- Default: `false`

**isAsync**:
- `true`: Execute in background, return immediately
- `false`: Wait for completion, return result
- Default: `false`

**allowedTools**:
- Array of tool names: `["Read", "Write", "Bash"]`
- Wildcard: `["*"]` (all tools)
- Pattern matching: `["Bash(git*)", "Read"]` (git commands only)

**color**:
- Visual identifier in UI
- 8 available colors
- Suffix `_FOR_SUBAGENTS_ONLY` required
- Assigned automatically if not specified

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
- `Agent` - Delegate to subagents
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

**Behavior**:
- Includes full parent conversation history
- Agent has access to all previous context
- Output added back to main conversation

**Token Impact**:
```
Parent context: 50,000 tokens
Agent processing: + 10,000 tokens
Total: 60,000 tokens (no savings)
```

**Use Cases**:
- Agent needs conversation context
- Multi-step tasks requiring history
- Tasks referencing previous work
- Context-dependent decisions

**Example**:
```typescript
// Agent can reference previous conversation
Task({
  agent_type: "general-purpose",  // forkContext: true
  prompt: "Update the function we discussed earlier",
  expected_output: "Function updated"
})
// ✅ Agent knows which function (from context)
```

---

### Isolated Context (forkContext: false)

**Behavior**:
- Fresh context, no parent history
- Agent only sees its own prompt
- Output summarized when returned

**Token Impact**:
```
Parent context: 50,000 tokens (ignored)
Agent processing: 15,000 tokens
Total: 15,000 tokens (70% savings)
```

**Use Cases**:
- Independent tasks (exploration, search)
- Token optimization
- Fast simple operations
- No context needed

**Example**:
```typescript
// Agent has no context, must be explicit
Task({
  agent_type: "Explore",  // forkContext: false
  prompt: "Find all files containing 'authentication' in src/auth/",
  expected_output: "List of matching files"
})
// ✅ All information in prompt
```

**Important**: Isolated agents require complete, self-contained prompts

---

## Agent Color System

### Color Assignment Algorithm

```typescript
const AGENT_COLORS = [
  "red", "blue", "green", "yellow",
  "purple", "orange", "pink", "cyan"
];

function assignAgentColor(agentType: string): string {
  // Hash agent type to index
  const hash = hashString(agentType);
  const index = hash % AGENT_COLORS.length;
  return AGENT_COLORS[index] + "_FOR_SUBAGENTS_ONLY";
}
```

**Characteristics**:
- Deterministic: Same agent type always gets same color
- 8 available colors
- Visual distinction in terminal UI
- Suffix pattern: `_FOR_SUBAGENTS_ONLY`

**Built-in Agent Colors**:
```typescript
const BUILT_IN_COLORS = {
  "Explore": "blue_FOR_SUBAGENTS_ONLY",
  "statusline-setup": "green_FOR_SUBAGENTS_ONLY",
  "output-style-setup": "yellow_FOR_SUBAGENTS_ONLY",
  "security-review": "red_FOR_SUBAGENTS_ONLY",
  "general-purpose": undefined  // No color assignment
};
```

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
  agent_type: "Explore",
  prompt: "Find authentication files",
  expected_output: "File list"
});
// Waits ~10 seconds
// result contains file list
```

---

### Async Agents (isAsync: true)

**Behavior**:
```
User → Task → Agent starts → [IMMEDIATE RETURN: agent_id] → Agent continues in background
Later → AgentOutput → Retrieve result by agent_id
```

**Characteristics**:
- Returns immediately with agent ID
- Agent continues in background
- Retrieve result later with AgentOutputTool

**Use Cases**:
- Long-running analysis
- Parallel agent execution
- Non-blocking workflows
- Background processing

**Example**:
```typescript
// Start async agent
const agentId = Task({
  agent_type: "security-review",
  prompt: "Complete security audit of codebase",
  expected_output: "Security report",
  is_async: true  // Override agent default
});
// Returns immediately: "agent_abc123"

// Continue other work...
Write({ file_path: "notes.md", contents: "..." });

// Retrieve result later
const result = AgentOutput({ agent_id: agentId });
// Returns: agent output if complete, or status if still running
```

**Agent Status States**:
```typescript
type AgentStatus = 
  | { status: 'running', progress?: string }
  | { status: 'complete', result: string }
  | { status: 'error', error: string };
```

---

## Custom Agent Creation

### Method 1: User Settings (Global)

**Location**: `~/.claude/settings.json`

```json
{
  "agents": {
    "my-custom-agent": {
      "description": "Custom agent for my workflows",
      "tools": ["Read", "Write", "Grep", "Bash"],
      "prompt": "You are a specialized agent for code refactoring...",
      "model": "sonnet"
    }
  }
}
```

### Method 2: Project Settings

**Location**: `<project-root>/.claude/settings.json`

```json
{
  "agents": {
    "project-reviewer": {
      "description": "Project-specific code reviewer",
      "tools": ["Read", "Grep", "Bash(git*)"],
      "prompt": "Review code changes according to project standards...",
      "model": "haiku"
    }
  }
}
```

### Method 3: Programmatic (SDK)

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

### Agent Definition Schema

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

### Token Efficiency Comparison

| Agent Type | Context | Avg Tokens | Savings vs Main |
|------------|---------|-----------|-----------------|
| Main (Sonnet) | Full | 50,000 | Baseline |
| general-purpose | Forked | 50,000+ | 0% (adds overhead) |
| Explore (Haiku) | Isolated | 15,000 | 70% |
| statusline-setup | Isolated | 8,000 | 84% |
| security-review | Isolated | 25,000 | 50% |

### Execution Time Comparison

| Agent Type | Model | Typical Duration | Use Case |
|------------|-------|------------------|----------|
| Explore | Haiku | 5-15s | Fast discovery |
| general-purpose | Sonnet | 30-120s | Complex tasks |
| statusline-setup | Sonnet | 10-20s | Config tasks |
| output-style-setup | Sonnet | 15-30s | Style setup |
| security-review | Sonnet | 30-60s | Security audit |

### Cost Optimization Strategies

**Strategy 1: Use Haiku for Simple Tasks**
```typescript
// ❌ Expensive (Sonnet for simple search)
const files = mainAgent.search("Find TODO comments");
// Cost: ~$0.50, Time: 15s

// ✅ Efficient (Haiku isolated agent)
const files = Task({
  agent_type: "Explore",
  prompt: "Find all TODO comments"
});
// Cost: ~$0.05, Time: 8s (90% cost reduction)
```

**Strategy 2: Parallel Async Agents**
```typescript
// ❌ Sequential (slow)
const security = Task({ agent_type: "security-review", ... });
const exploration = Task({ agent_type: "Explore", ... });
// Total time: 60s + 10s = 70s

// ✅ Parallel (fast)
const securityId = Task({ agent_type: "security-review", is_async: true });
const explorationId = Task({ agent_type: "Explore", is_async: true });
const security = AgentOutput({ agent_id: securityId });
const exploration = AgentOutput({ agent_id: explorationId });
// Total time: max(60s, 10s) = 60s (10s saved)
```

**Strategy 3: Isolated Context for Independence**
```typescript
// ❌ Forked context (unnecessary tokens)
{ forkContext: true }  // 50k tokens context loaded

// ✅ Isolated context (minimal tokens)
{ forkContext: false }  // 0 tokens context
// Savings: 50k tokens × $0.003/1k = $0.15 per invocation
```

---

## Real-World Patterns

### Pattern 1: Multi-Stage Analysis

```typescript
// Stage 1: Fast exploration (Haiku)
const files = Task({
  agent_type: "Explore",
  prompt: "Find all API route files",
  expected_output: "List of files"
});

// Stage 2: Detailed analysis (Sonnet)
const analysis = Task({
  agent_type: "general-purpose",
  prompt: `Analyze these API files for security issues: ${files}`,
  expected_output: "Security assessment"
});

// Stage 3: Security review (specialized)
const review = Task({
  agent_type: "security-review",
  prompt: "Review current branch for vulnerabilities",
  expected_output: "Security report"
});
```

### Pattern 2: Parallel Independent Tasks

```typescript
// Launch all agents simultaneously
const [filesId, securityId, testsId] = [
  Task({
    agent_type: "Explore",
    prompt: "Find all test files",
    is_async: true
  }),
  Task({
    agent_type: "security-review",
    prompt: "Audit current changes",
    is_async: true
  }),
  Task({
    agent_type: "general-purpose",
    prompt: "Generate test coverage report",
    is_async: true
  })
];

// Retrieve results when needed
const files = AgentOutput({ agent_id: filesId });
const security = AgentOutput({ agent_id: securityId });
const tests = AgentOutput({ agent_id: testsId });
```

### Pattern 3: Specialized Agent Delegation

```typescript
// Create project-specific agent
const codeReviewer = {
  "code-reviewer": {
    description: "Enforce project code standards",
    tools: ["Read", "Grep", "Bash(git*)"],
    prompt: `
      Review code changes for:
      - PEP 8 compliance (Python)
      - No console.log statements
      - All functions have docstrings
      - Tests updated for changes
    `,
    model: "sonnet"
  }
};

// Use in workflow
Task({
  agent_type: "code-reviewer",
  prompt: "Review current branch vs main",
  expected_output: "Pass/fail with specific issues"
});
```

---

## Internal Implementation Details

### Agent Tool (Task Tool)

**Internal Name:** `Task` (CLI) vs `Agent` (SDK API)
**Source:** cli.js:212 (variable `Y3="Task"`)

### AgentInput Interface

```typescript
export interface AgentInput {
    /**
     * A short (3-5 word) description of the task
     */
    description: string;
    /**
     * The task for the agent to perform
     */
    prompt: string;
    /**
     * The type of specialized agent to use for this task
     */
    subagent_type: string;
}
```

### Internal Agent Execution Flow

#### 1. Agent Invocation
```javascript
async*call({prompt:A, subagent_type:B, description:Q}, Z, G, Y)
```

#### 2. Agent Lookup
```javascript
let I = Z.options.agentDefinitions.activeAgents.find((z)=>z.agentType===B)
```

#### 3. Model Selection
```javascript
let F = jy1(I.model, Z.options.mainLoopModel)
```

#### 4. Tool Resolution
```javascript
let V = S51(I, Z.options.tools).resolvedTools
```

#### 5. Message Construction
```javascript
let K = I?.forkContext ? Z.messages : void 0
let D = I?.forkContext ? ZUQ(A,Y) : [mA({content:A})]
```

#### 6. System Prompt Construction
```javascript
let O = I.systemPrompt ? [I.systemPrompt] : [cLQ]
let R = await lLQ(O, X, L)  // L = additional working directories
```

#### 7. Agent Execution
```javascript
for await(let p of Rt1({
  agentDefinition:I,
  promptMessages:K,
  toolUseContext:Z,
  canUseTool:G,
  forkContextMessages:V,
  isAsync:I?.isAsync||!1,
  recordMessagesToSessionStorage:!0,
  querySource:MwQ(I.agentType, X)
}))
```

#### 8. Result Processing
```javascript
// For async agents
yield {type:"result", data:{status:"async_launched", agentId:K, ...}}

// For sync agents
let q = Iw8(z, D)  // Extract metrics and content
yield {type:"result", data:{status:"completed", prompt:A, ...q}}
```

### Agent Architect Prompt

**Source:** cli.js:3047 (variable `jw8`)
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

```typescript
export type SubagentStopHookInput = BaseHookInput & {
    hook_event_name: 'SubagentStop';
    stop_hook_active: boolean;
};

// Hook behavior:
// Exit code 0 - stdout/stderr not shown
// Exit code 2 - show stderr to subagent and continue
// Other exit codes - show stderr to user only
```

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

### Tool Name Constants

| Tool | Constant | Value | Source Line |
|------|----------|-------|-------------|
| Grep | `bF` | "Grep" | ~212 |
| Glob | `ND` | "Glob" | ~212 |
| Read | `x8` | "Read" | ~212 |
| Write | `wJ` | "Write" | ~212 |
| Edit | `R3` | "Edit" | ~212 |
| Bash | `q4` | "Bash" | ~212 |
| Task | `Y3` | "Task" | 212 |

---

## Gotchas & Best Practices

### Gotchas

1. **Isolated Agents Have No Context**:
   ```typescript
   // ❌ Bad: Reference to "that file" unclear
   Task({
     agent_type: "Explore",  // forkContext: false
     prompt: "Search that file for errors"
   });
   
   // ✅ Good: Explicit file path
   Task({
     agent_type: "Explore",
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

4. **Async Agents Don't Auto-Return**:
   - Must explicitly call `AgentOutput` to retrieve result
   - No automatic notification when complete

5. **Model Inheritance Can Be Expensive**:
   ```typescript
   // Parent uses Opus (expensive)
   { model: "inherit" }  // Agent also uses Opus
   
   // Better: Explicit model
   { model: "haiku" }  // Force cheaper model
   ```

6. **general-purpose Agent No Color**:
   - No visual distinction in UI
   - Can be confusing if used frequently

### Best Practices

**1. Choose the Right Agent**:
```typescript
// ✅ Use Explore for discovery
Task({ agent_type: "Explore", prompt: "Find files..." })

// ✅ Use general-purpose for complex tasks
Task({ agent_type: "general-purpose", prompt: "Refactor auth system..." })

// ✅ Use security-review for security
Task({ agent_type: "security-review", prompt: "Audit changes..." })
```

**2. Provide Complete Prompts for Isolated Agents**:
```typescript
Task({
  agent_type: "Explore",
  prompt: `
    Find all React components in src/components/ that:
    - Use useState hook
    - Have more than 200 lines
    - Don't have TypeScript types
    
    Return: File paths with line counts
  `,
  expected_output: "List of components: file:lines"
});
```

**3. Specify Expected Output Format**:
```typescript
Task({
  agent_type: "security-review",
  prompt: "Audit current branch",
  expected_output: `
    Format:
    Risk Level: HIGH/MEDIUM/LOW
    Issues Found: [list]
    Recommendations: [list]
  `
});
```

**4. Use Async for Long Operations**:
```typescript
// Long-running security audit
const auditId = Task({
  agent_type: "security-review",
  prompt: "Complete codebase security audit",
  is_async: true
});

// Continue other work...

// Check later
const result = AgentOutput({ agent_id: auditId });
```

**5. Combine Agent Types for Workflow**:
```typescript
// 1. Explore (fast discovery)
const files = Task({ agent_type: "Explore", ... });

// 2. general-purpose (detailed work)
const analysis = Task({ agent_type: "general-purpose", ... });

// 3. security-review (specialized audit)
const security = Task({ agent_type: "security-review", ... });
```

---

## Summary

### Agent Selection Matrix

| Task Type | Agent | Why |
|-----------|-------|-----|
| Find files/code | Explore | Fast Haiku, isolated, file tools only |
| Complex refactoring | general-purpose | Full tools, forked context |
| Security audit | security-review | Git-restricted bash, security focus |
| Configuration | statusline/output-style | Minimal tools, config-specific |
| Custom workflow | Custom agent | Tailored tools and prompt |

### Token Efficiency Ranking

1. **Explore** (Haiku + isolated): 70-84% savings
2. **statusline-setup** (isolated): 84% savings
3. **security-review** (isolated): 50% savings
4. **output-style-setup** (isolated): 50-70% savings
5. **general-purpose** (forked): 0% savings (actually adds overhead)

### Key Takeaways

- ✅ Use isolated agents (forkContext: false) for token efficiency
- ✅ Use Explore agent for codebase discovery (40-70% faster + cheaper)
- ✅ Provide complete prompts to isolated agents (no context available)
- ✅ Use async agents for parallel execution and long tasks
- ✅ Create custom agents for repeated specialized workflows
- ⚠️ Avoid general-purpose for simple tasks (expensive + slow)
- ⚠️ Remember tool restrictions are strictly enforced
- ⚠️ 8 color limit for visual distinction
