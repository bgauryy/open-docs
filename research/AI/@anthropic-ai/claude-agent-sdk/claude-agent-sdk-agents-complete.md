# Claude Agent SDK - Complete Agent System Documentation

## Table of Contents

1. [Overview](#overview)
2. [System Prompts](#system-prompts)
3. [Agent Tool (Task Tool)](#agent-tool-task-tool)
4. [Agent Definition Structures](#agent-definition-structures)
5. [Built-in Agents](#built-in-agents)
6. [Agent Architecture](#agent-architecture)
7. [Configuring Agents](#configuring-agents)
8. [Tool Restrictions](#tool-restrictions)
9. [Model Requirements](#model-requirements)
10. [Subagent Delegation Patterns](#subagent-delegation-patterns)
11. [Agent Lifecycle](#agent-lifecycle)
12. [Session Information](#session-information)
13. [Best Practices](#best-practices)
14. [Examples](#examples)
15. [Advanced Topics](#advanced-topics)
16. [Source Code References](#source-code-references)

---

## Overview

The Claude Agent SDK provides a powerful agent system that enables specialized task delegation through subagents. The agent system allows the main agent to delegate specific tasks to specialized agents with custom prompts, tool restrictions, and model configurations.

### Core Capabilities

Each agent runs in its own context with:

- **Custom system prompt** - Defines the agent's expertise and behavior
- **Specific tools** - Limited set of tools appropriate for the agent's role
- **Independent context** - Separate conversation history and memory
- **Model selection** - Can use different Claude models (sonnet, haiku, opus, inherit)
- **Async execution** - Some agents can run in the background

### Core Concepts

1. **Main Agent**: The primary agent that orchestrates the conversation and can delegate tasks to subagents
2. **Subagents**: Specialized agents that handle specific tasks with customized configurations
3. **Agent Delegation**: Performed through the `Agent` tool (previously called `Task` tool)
4. **Agent Definitions**: Configuration objects that define agent behavior, capabilities, and restrictions

---

## System Prompts

### Three Main System Prompts

The SDK uses three different system prompts depending on the execution context:

#### 1. Standard Claude Code Prompt (`mOA`)
**Source:** cli.js:285
**Usage:** Default for interactive Claude Code CLI sessions

```
You are Claude Code, Anthropic's official CLI for Claude.
```

#### 2. SDK Mode Prompt (`Nw9`)
**Source:** cli.js:285
**Usage:** When running within Claude Agent SDK (non-interactive)

```
You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.
```

#### 3. Agent Mode Prompt (`dOA`)
**Source:** cli.js:285
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

## Agent Tool (Task Tool)

### Task Tool Constant
**Source:** cli.js:212
```javascript
var Y3="Task"
```

**Note:** In the SDK public API, this is referred to as the `Agent` tool, but internally it's called the `Task` tool.

### Task Tool Description (Line 2556)

The Task tool enables launching specialized agents. Key components:

**Full Description Template:**
```javascript
async function WWQ(A){
  return`Launch a new agent to handle complex, multi-step tasks autonomously.

Available agent types and the tools they have access to:
${A.map((Q)=>{
  let Z="";
  if(Q?.isAsync||Q?.forkContext)
    Z="Properties: "+(Q?.isAsync?"runs in background; ":"")+(Q?.forkContext?"access to current context; ":"");
  return`- ${Q.agentType}: ${Q.whenToUse} (${Z}Tools: ${Q.tools.join(", ")})`
}).join(` `)}

When using the ${Y3} tool, you must specify a subagent_type parameter to select which agent type to use.

When NOT to use the Agent tool:
- If you want to read a specific file path, use the ${w6.name} or ${xw.name} tool instead
- If searching for a specific class definition, use ${xw.name} tool instead
- If searching code within a specific file or set of 2-3 files, use ${w6.name} tool instead
- Other tasks not related to agent descriptions above

Usage notes:
- Launch multiple agents concurrently whenever possible
- Agent returns single message - result not visible to user
- For async agents, use AgentOutputTool to retrieve results
- Each agent invocation is stateless
- Tell agent whether to write code or just research
- If agent should be used proactively, try without user asking first
...
[Includes detailed examples]
`
}
```

### AgentInput Interface (SDK Public API)

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

### Tool Schema (Internal CLI Implementation)

```javascript
// Input Schema
Bw8 = x.object({
  description: x.string().describe("A short (3-5 word) description of the task"),
  prompt: x.string().describe("The task for the agent to perform"),
  subagent_type: x.string().describe("The type of specialized agent to use")
})

// Output Schema (Union of possible responses)
Yw8 = x.union([
  Zw8,  // Completed status
  Gw8,  // Async launched status
  QUQ   // Error status
])
```

### Using the Agent Tool

When the main agent needs to delegate work, it uses the `Agent` tool with:

1. **description**: Brief task summary (3-5 words)
2. **prompt**: Detailed task instructions for the subagent
3. **subagent_type**: Name/key of the predefined agent to use

### Example Usage

```typescript
// Agent tool call
{
    "description": "Analyze codebase",
    "prompt": "Search for all React components and analyze their prop types",
    "subagent_type": "code_analyzer"
}
```

---

## Agent Definition Structures

### ⚠️ Two Different Structures

There are **two different agent definition structures** depending on whether you're using the SDK public API or looking at the internal CLI implementation:

### 1. SDK Public API Structure

**Used when:** Configuring agents through the SDK's `query()` options

```typescript
export type AgentDefinition = {
    description: string;
    tools?: string[];
    prompt: string;
    model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
```

#### AgentDefinition Fields (SDK API)

##### `description` (required)
- **Type**: `string`
- **Purpose**: Human-readable description of the agent's purpose
- **Usage**: Helps identify what the agent specializes in

##### `tools` (optional)
- **Type**: `string[]`
- **Purpose**: Array of tool names that this agent is allowed to use
- **Behavior**:
  - If specified: Agent can ONLY use the listed tools (whitelist)
  - If omitted: Agent inherits all tools from parent agent
- **Examples**: `["Bash", "Read", "Write"]`, `["Grep", "Glob", "Read"]`

##### `prompt` (required)
- **Type**: `string`
- **Purpose**: System prompt that defines the agent's behavior and instructions
- **Usage**: Customizes how the agent approaches tasks
- **Best Practices**:
  - Be specific about the agent's role
  - Include task-specific guidelines
  - Define output format expectations

##### `model` (optional)
- **Type**: `'sonnet' | 'opus' | 'haiku' | 'inherit'`
- **Purpose**: Specifies which Claude model variant to use for this agent
- **Default**: If not specified, inherits from parent agent
- **Options**:
  - `'sonnet'`: Claude Sonnet (balanced performance and cost)
  - `'opus'`: Claude Opus (highest capability)
  - `'haiku'`: Claude Haiku (fastest, most economical)
  - `'inherit'`: Use the same model as the parent agent

### 2. Internal CLI Implementation Structure

**Used by:** The internal CLI implementation in cli.js

```typescript
interface AgentDefinition {
  // Required fields
  agentType: string;           // Unique identifier (e.g., "code-reviewer")
  whenToUse: string;           // Description of when Claude should use this agent
  tools: string[] | ["*"];     // Tool names or "*" for all tools
  systemPrompt: string;        // Complete system prompt for the agent
  source: "built-in" | "plugin" | "userSettings" | "projectSettings" | "policySettings" | "flagSettings";

  // Optional fields
  model?: "sonnet" | "haiku" | "opus" | "inherit";  // Defaults to sonnet
  baseDir?: string;            // Directory for plugin/custom agents
  filename?: string;           // Original filename for custom agents
  color?: string;              // UI color (red|blue|green|yellow|purple|orange|pink|cyan)
  isAsync?: boolean;           // Whether agent runs in background
  forkContext?: boolean;       // Whether agent gets parent's context
  callback?: () => void;       // Function to call after agent completes
}
```

### Agent Tool Resolution

**Source:** cli.js (S51 function)

Agents can specify tools in three ways:

1. **All tools**: `["*"]` - Agent gets access to all available tools
2. **Specific tools**: `["Read", "Write", "Glob"]` - Explicit list
3. **Tool references**: `[ND, bF, x8]` - Variable references resolved at runtime

**Example resolution:**
```javascript
// Tool constants
var ND = "Glob"
var bF = "Grep"
var x8 = "Read"
var q4 = "Bash"

// Agent definition
tools: [ND, bF, x8, q4]
// Resolves to: ["Glob", "Grep", "Read", "Bash"]
```

---

## Built-in Agents

### 1. General-Purpose Agent

**Source:** cli.js:2810
**Variable:** `Tt1`

```javascript
{
  agentType: "general-purpose",
  whenToUse: "General-purpose agent for researching complex questions, searching for code, and executing multi-step tasks. When you are searching for a keyword or file and are not confident that you will find the right match in the first few tries use this agent to perform the search for you.",
  tools: ["*"],  // Has access to ALL tools
  systemPrompt: `You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's message, you should use the tools available to complete the task. Do what has been asked; nothing more, nothing less. When you complete the task simply respond with a detailed writeup.

Your strengths:
- Searching for code, configurations, and patterns across large codebases
- Analyzing multiple files to understand system architecture
- Investigating complex questions that require exploring many files
- Performing multi-step research tasks

Guidelines:
- For file searches: Use Grep or Glob when you need to search broadly. Use Read when you know the specific file path.
- For analysis: Start broad and narrow down. Use multiple search strategies if the first doesn't yield results.
- Be thorough: Check multiple locations, consider different naming conventions, look for related files.
- NEVER create files unless they're absolutely necessary for achieving your goal. ALWAYS prefer editing an existing file to creating a new one.
- NEVER proactively create documentation files (*.md) or README files. Only create documentation files if explicitly requested.
- In your final response always share relevant file names and code snippets. Any file paths you return in your response MUST be absolute. Do NOT use relative paths.
- For clear communication, avoid using emojis.`,
  source: "built-in",
  baseDir: "built-in",
  model: "sonnet",
  isAsync: false
}
```

---

### 2. Explore Agent (File Search Specialist)

**Source:** cli.js:3016
**Variable:** `hj`

```javascript
{
  agentType: "Explore",
  whenToUse: 'Fast agent specialized for exploring codebases. Use this when you need to quickly find files by patterns (eg. "src/components/**/*.tsx"), search code for keywords (eg. "API endpoints"), or answer questions about the codebase (eg. "how do API endpoints work?"). When calling this agent, specify the desired thoroughness level: "quick" for basic searches, "medium" for moderate exploration, or "very thorough" for comprehensive analysis across multiple locations and naming conventions.',
  tools: [ND, bF, x8, q4],  // Glob, Grep, Read, Bash
  systemPrompt: `You are a file search specialist for Claude Code, Anthropic's official CLI for Claude. You excel at thoroughly navigating and exploring codebases.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

Guidelines:
- Use ${ND} for broad file pattern matching
- Use ${bF} for searching file contents with regex
- Use ${x8} when you know the specific file path you need to read
- Use ${q4} for file operations like copying, moving, or listing directory contents
- Adapt your search approach based on the thoroughness level specified by the caller
- Return file paths as absolute paths in your final response
- For clear communication, avoid using emojis
- Do not create any files, or run bash commands that modify the user's system state in any way

Complete the user's search request efficiently and report your findings clearly.`,
  source: "built-in",
  baseDir: "built-in",
  model: "haiku",  // Uses faster/cheaper model
  isAsync: false
}
```

---

### 3. Statusline-Setup Agent

**Source:** cli.js:2938
**Variable:** `jMQ`

```javascript
{
  agentType: "statusline-setup",
  whenToUse: "Use this agent to configure the user's Claude Code status line setting.",
  tools: ["Read", "Edit"],
  systemPrompt: `You are a status line setup agent for Claude Code. Your job is to create or update the statusLine command in the user's Claude Code settings.

When asked to convert the user's shell PS1 configuration, follow these steps:
1. Read the user's shell configuration files in this order of preference:
   - ~/.zshrc
   - ~/.bashrc
   - ~/.bash_profile
   - ~/.profile

2. Extract the PS1 value using this regex pattern: /(?:^|\\n)\\s*(?:export\\s+)?PS1\\s*=\\s*["']([^"']+)["']/m

3. Convert PS1 escape sequences to shell commands:
   - \\u → $(whoami)
   - \\h → $(hostname -s)
   - \\H → $(hostname)
   - \\w → $(pwd)
   - \\W → $(basename "$(pwd)")
   - \\$ → $
   - \\n → \\n
   - \\t → $(date +%H:%M:%S)
   - \\d → $(date "+%a %b %d")
   - \\@ → $(date +%I:%M%p)
   - \\# → #
   - \\! → !

4. When using ANSI color codes, be sure to use \`printf\`. Do not remove colors. Note that the status line will be printed in a terminal using dimmed colors.

5. If the imported PS1 would have trailing "$" or ">" characters in the output, you MUST remove them.

6. If no PS1 is found and user did not provide other instructions, ask for further instructions.

How to use the statusLine command:
1. The statusLine command will receive the following JSON input via stdin:
   {
     "session_id": "string",
     "transcript_path": "string",
     "cwd": "string",
     "model": {
       "id": "string",
       "display_name": "string"
     },
     "workspace": {
       "current_dir": "string",
       "project_dir": "string"
     },
     "version": "string",
     "output_style": {
       "name": "string"
     }
   }

   You can use this JSON data in your command like:
   - $(cat | jq -r '.model.display_name')
   - $(cat | jq -r '.workspace.current_dir')
   - $(cat | jq -r '.output_style.name')

2. For longer commands, you can save a new file in the user's ~/.claude directory

3. Update the user's ~/.claude/settings.json with:
   {
     "statusLine": {
       "type": "command",
       "command": "your_command_here"
     }
   }

4. If ~/.claude/settings.json is a symlink, update the target file instead.

Guidelines:
- Preserve existing settings when updating
- Return a summary of what was configured
- If the script includes git commands, they should skip optional locks
- IMPORTANT: At the end inform parent agent that "statusline-setup" agent must be used for further status line changes
- Ensure user is informed they can ask Claude to continue to make changes`,
  source: "built-in",
  baseDir: "built-in",
  model: "sonnet",
  color: "orange"
}
```

---

### 4. Output-Style-Setup Agent

**Source:** cli.js:2868
**Variable:** `RMQ`

```javascript
{
  agentType: "output-style-setup",
  whenToUse: "Use this agent to create a Claude Code output style.",
  tools: [x8, wJ, R3, ND, bF],  // Read, Write, Edit, Glob, Grep
  systemPrompt: `Your job is to create a custom output style, which modifies the Claude Code system prompt, based on the user's description.

[Full prompt for creating custom output styles]`,
  source: "built-in",
  baseDir: "built-in",
  model: "sonnet",
  color: "blue",
  callback: () => { PMQ() }
}
```

---

## Agent Architecture

### Agent Architect Prompt

**Source:** cli.js:3047
**Variable:** `jw8`
**Purpose:** Used by Claude to generate new agent definitions

```javascript
`You are an elite AI agent architect specializing in crafting high-performance agent configurations. Your expertise lies in translating user requirements into precisely-tuned agent specifications that maximize effectiveness and reliability.

**Important Context**: You may have access to project-specific instructions from CLAUDE.md files and other context that may include coding standards, project structure, and custom requirements. Consider this context when creating agents to ensure they align with the project's established patterns and practices.

When a user describes what they want an agent to do, you will:

1. **Extract Core Intent**: Identify the fundamental purpose, key responsibilities, and success criteria for the agent. Look for both explicit requirements and implicit needs. Consider any project-specific context from CLAUDE.md files. For agents that are meant to review code, you should assume that the user is asking to review recently written code and not the whole codebase, unless the user has explicitly instructed you otherwise.

2. **Design Expert Persona**: Create a compelling expert identity that embodies deep domain knowledge relevant to the task. The persona should inspire confidence and guide the agent's decision-making approach.

3. **Architect Comprehensive Instructions**: Develop a system prompt that:
   - Establishes clear behavioral boundaries and operational parameters
   - Provides specific methodologies and best practices for task execution
   - Anticipates edge cases and provides guidance for handling them
   - Incorporates any specific requirements or preferences mentioned by the user
   - Defines output format expectations when relevant
   - Aligns with project-specific coding standards and patterns from CLAUDE.md

4. **Optimize for Performance**: Include:
   - Decision-making frameworks appropriate to the domain
   - Quality control mechanisms and self-verification steps
   - Efficient workflow patterns
   - Clear escalation or fallback strategies

5. **Create Identifier**: Design a concise, descriptive identifier that:
   - Uses lowercase letters, numbers, and hyphens only
   - Is typically 2-4 words joined by hyphens
   - Clearly indicates the agent's primary function
   - Is memorable and easy to type
   - Avoids generic terms like "helper" or "assistant"

6. **Example agent descriptions**:
   - In the 'whenToUse' field, include examples of when this agent should be used
   - Examples should show the assistant using the Task tool (not responding directly)
   - If agent should be used proactively, include examples showing this

Your output must be a valid JSON object with exactly these fields:
{
  "identifier": "unique-agent-identifier",
  "whenToUse": "Use this agent when... [includes examples]",
  "systemPrompt": "You are... [complete system prompt]"
}

Key principles:
- Be specific rather than generic - avoid vague instructions
- Include concrete examples when they would clarify behavior
- Balance comprehensiveness with clarity
- Ensure agent has enough context to handle variations
- Make agent proactive in seeking clarification when needed
- Build in quality assurance and self-correction mechanisms

Remember: The agents you create should be autonomous experts capable of handling their designated tasks with minimal additional guidance.`
```

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
            },

            // Refactoring specialist
            refactor_agent: {
                description: "Specialized agent for code refactoring",
                tools: ["Read", "Edit", "Bash", "Grep"],
                prompt: "You are a refactoring expert. Improve code quality while maintaining functionality.",
                model: "opus"  // Most capable model for complex refactoring
            }
        }
    }
});
```

---

## Tool Restrictions

### Tool Access Control

- **Whitelist Approach**: When `tools` array is specified, agent can ONLY use those tools
- **Inheritance**: When `tools` is omitted, agent inherits all available tools
- **No Blacklist**: There is no `disallowedTools` field; use whitelist only

### Available Tools

Common SDK tools that can be restricted:
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

### Tool Restriction Strategy

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

## Model Requirements

### Model Selection Guidelines

1. **Haiku** (`'haiku'`)
   - **Use for**: Simple tasks, fast execution, cost optimization
   - **Examples**: Running tests, simple searches, file operations
   - **Characteristics**: Fastest, most economical

2. **Sonnet** (`'sonnet'`)
   - **Use for**: Balanced tasks, general purpose work
   - **Examples**: Code analysis, documentation, moderate complexity
   - **Characteristics**: Good balance of capability and cost

3. **Opus** (`'opus'`)
   - **Use for**: Complex tasks, critical work, deep analysis
   - **Examples**: Complex refactoring, architectural decisions
   - **Characteristics**: Highest capability, slower, more expensive

4. **Inherit** (`'inherit'`)
   - **Use for**: Same capability as parent needed
   - **Behavior**: Uses parent agent's model
   - **Default**: When `model` field is omitted

### Model Usage Tracking

The SDK tracks model usage per agent in the result:

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

---

## Subagent Delegation Patterns

### Pattern 1: Sequential Delegation

```typescript
// Main agent delegates tasks in sequence
agents: {
    analyzer: {
        description: "Analyze code structure",
        tools: ["Grep", "Glob", "Read"],
        prompt: "Analyze the codebase structure and identify components",
        model: "sonnet"
    },
    documenter: {
        description: "Write documentation",
        tools: ["Read", "Write"],
        prompt: "Based on analysis, create comprehensive documentation",
        model: "sonnet"
    }
}

// Usage flow:
// 1. Main agent delegates to 'analyzer'
// 2. Analyzer returns results
// 3. Main agent delegates to 'documenter' with analysis results
```

### Pattern 2: Parallel Specialization

```typescript
// Different agents for different aspects
agents: {
    frontend_specialist: {
        description: "Frontend code specialist",
        tools: ["Grep", "Read", "Edit"],
        prompt: "Handle React/UI components and styling",
        model: "sonnet"
    },
    backend_specialist: {
        description: "Backend code specialist",
        tools: ["Grep", "Read", "Edit", "Bash"],
        prompt: "Handle API routes, database, and server logic",
        model: "sonnet"
    },
    test_specialist: {
        description: "Testing specialist",
        tools: ["Bash", "Read", "Write"],
        prompt: "Write and run tests for all components",
        model: "haiku"
    }
}
```

### Pattern 3: Hierarchical Delegation

```typescript
// Agents can delegate to other agents
agents: {
    project_manager: {
        description: "Coordinates other agents",
        tools: ["Agent", "Read"],  // Can delegate
        prompt: "Coordinate tasks across specialized agents",
        model: "opus"
    },
    code_writer: {
        description: "Writes code",
        tools: ["Read", "Write", "Edit"],
        prompt: "Implement features as directed",
        model: "sonnet"
    },
    reviewer: {
        description: "Reviews code",
        tools: ["Read", "Grep"],
        prompt: "Review code for quality and issues",
        model: "sonnet"
    }
}

// Flow: project_manager -> code_writer -> reviewer
```

### Pattern 4: Safety-Constrained Agents

```typescript
agents: {
    safe_researcher: {
        description: "Read-only research agent",
        tools: ["Grep", "Glob", "Read", "WebSearch"],  // No write/edit
        prompt: "Research and gather information only",
        model: "sonnet"
    },
    verified_writer: {
        description: "Careful file writer",
        tools: ["Read", "Write"],  // No Bash execution
        prompt: "Write files only after careful verification",
        model: "sonnet"
    }
}
```

---

## Agent Lifecycle

### Agent Execution Flow

1. **Delegation**
   - Main agent calls `Agent` tool with `subagent_type`
   - SDK looks up agent definition by key
   - New agent context is created with custom configuration

2. **Initialization**
   - Subagent receives custom system prompt
   - Tool restrictions are applied
   - Model selection is configured

3. **Execution**
   - Subagent processes the delegated task
   - Only allowed tools can be used
   - Operates with specified model

4. **Completion**
   - Subagent returns results to main agent
   - Session statistics are tracked
   - Main agent continues with results

5. **Hooks**
   - `SubagentStop` hook fires before subagent concludes
   - Allows interception and continuation logic

### Internal Agent Execution Flow (CLI Implementation)

#### 1. Agent Invocation (via Task Tool)
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

---

## Session Information

### Agent Tracking in System Messages

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

The `agents` field lists all configured agent type names available for delegation.

---

## Best Practices

### 1. Agent Design

- **Single Responsibility**: Each agent should have a clear, focused purpose
- **Descriptive Names**: Use clear agent type names (e.g., `code_analyzer` not `agent1`)
- **Appropriate Tools**: Only include tools needed for the agent's specific task
- **Clear Prompts**: Write detailed system prompts that define behavior precisely

### 2. Tool Restrictions

- **Principle of Least Privilege**: Only grant tools that are necessary
- **Read-Only Agents**: For research/analysis, exclude Write/Edit/Bash
- **Safe Execution**: For file operations, consider excluding Bash
- **Delegation Control**: Carefully consider which agents can use the `Agent` tool

### 3. Model Selection

- **Cost Optimization**: Use Haiku for simple tasks to reduce costs
- **Quality Needs**: Use Opus for critical/complex tasks
- **Default to Sonnet**: Good balance for most tasks
- **Inherit Wisely**: Use inherit when consistency with parent is important

### 4. Delegation Strategy

- **Break Down Complex Tasks**: Use multiple specialized agents instead of one complex agent
- **Sequential for Dependencies**: When output of one agent feeds another
- **Parallel for Independence**: Different agents for independent aspects
- **Avoid Deep Nesting**: Limit agent delegation depth for maintainability

### 5. Error Handling

- **Graceful Failures**: Design prompts to handle errors constructively
- **Result Validation**: Main agent should validate subagent results
- **Fallback Strategies**: Have backup approaches when delegation fails

---

## Examples

### Example 1: Code Review System

```typescript
options: {
    agents: {
        linter: {
            description: "Code linting specialist",
            tools: ["Bash", "Read", "Grep"],
            prompt: "Run linters and analyze code quality issues",
            model: "haiku"
        },
        security_checker: {
            description: "Security analysis specialist",
            tools: ["Grep", "Read", "WebSearch"],
            prompt: "Check for security vulnerabilities and best practices",
            model: "sonnet"
        },
        style_reviewer: {
            description: "Code style reviewer",
            tools: ["Read", "Grep"],
            prompt: "Review code style, naming, and documentation",
            model: "sonnet"
        }
    }
}
```

### Example 2: Documentation Generator

```typescript
options: {
    agents: {
        code_reader: {
            description: "Code structure analyzer",
            tools: ["Glob", "Grep", "Read"],
            prompt: "Analyze code structure and extract key information",
            model: "sonnet"
        },
        doc_writer: {
            description: "Documentation author",
            tools: ["Read", "Write"],
            prompt: "Write clear, comprehensive documentation with examples",
            model: "sonnet"
        },
        example_generator: {
            description: "Code example creator",
            tools: ["Read", "Bash"],
            prompt: "Create working code examples for documentation",
            model: "sonnet"
        }
    }
}
```

### Example 3: Test Suite Manager

```typescript
options: {
    agents: {
        test_writer: {
            description: "Test case author",
            tools: ["Read", "Write", "Grep"],
            prompt: "Write comprehensive unit and integration tests",
            model: "sonnet"
        },
        test_runner: {
            description: "Test execution specialist",
            tools: ["Bash", "Read"],
            prompt: "Execute test suites and analyze results",
            model: "haiku"
        },
        coverage_analyzer: {
            description: "Coverage analysis specialist",
            tools: ["Bash", "Read", "Grep"],
            prompt: "Analyze test coverage and identify gaps",
            model: "sonnet"
        }
    }
}
```

---

## Advanced Topics

### Dynamic Agent Configuration

While agents are typically defined in options, you can potentially modify agent behavior through:

- **System Prompt Customization**: Use the main `systemPrompt` option
- **Tool Availability**: Configure through `allowedTools` in base options
- **Model Switching**: Use `setModel()` control method during execution

### Permission Integration

Agents operate within the same permission system:

```typescript
export type Options = {
    // ... other options
    canUseTool?: CanUseTool;
    permissionMode?: PermissionMode;
    permissionPromptToolName?: string;
};
```

All agents (main and subagents) respect:
- Permission mode settings
- Tool permission callbacks
- Permission rules and directories

### MCP Integration

Agents can use MCP (Model Context Protocol) tools:

```typescript
agents: {
    mcp_specialist: {
        description: "MCP tool specialist",
        tools: ["Mcp", "ListMcpResources", "ReadMcpResource"],
        prompt: "Interact with MCP servers for extended capabilities",
        model: "sonnet"
    }
}
```

---

## Source Code References

### Key Files and Line Numbers

| Component | Source File | Line Number | Variable |
|-----------|-------------|-------------|----------|
| System Prompts | cli.js | 285 | `mOA`, `Nw9`, `dOA` |
| Task Tool Name | cli.js | 212 | `Y3` |
| Task Tool Description | cli.js | 2556 | `WWQ()` function |
| General-Purpose Agent | cli.js | 2810 | `Tt1` |
| Statusline-Setup Agent | cli.js | 2938 | `jMQ` |
| Output-Style-Setup Agent | cli.js | 2868 | `RMQ` |
| Explore Agent | cli.js | 3016 | `hj` |
| Agent Architect Prompt | cli.js | 3047 | `jw8` |
| Agent List Function | cli.js | ~3033 | `Ba0()` |

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

## Summary

The Claude Agent SDK's agent system provides:

1. **Flexible Delegation**: Delegate tasks to specialized subagents
2. **Tool Control**: Fine-grained control over tool access per agent
3. **Model Selection**: Choose appropriate models for different tasks
4. **Cost Optimization**: Use faster/cheaper models where appropriate
5. **Safety**: Restrict capabilities through tool whitelisting
6. **Scalability**: Build complex workflows with agent hierarchies

### Key Takeaways

- **Two Agent Definition Structures**: SDK public API uses `description`, `tools`, `prompt`, `model`; internal CLI uses `agentType`, `whenToUse`, `systemPrompt`, etc.
- **Agent Tool/Task Tool**: Same tool, different names (Agent in SDK API, Task in CLI)
- **AgentInput** delegates tasks with `subagent_type`, `description`, and `prompt`
- **Tool Restrictions** use whitelist approach via `tools` array
- **Model Options**: `sonnet`, `opus`, `haiku`, or `inherit`
- **Delegation Patterns**: Sequential, parallel, hierarchical, or safety-constrained
- **Best Practice**: Single responsibility, least privilege, appropriate model selection
