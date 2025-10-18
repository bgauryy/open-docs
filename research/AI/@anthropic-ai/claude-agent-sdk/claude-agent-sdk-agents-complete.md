# Claude Agent SDK - Complete Agent System Documentation

## Overview

The Claude Agent SDK provides a powerful agent system that enables specialized task delegation through subagents. The agent system allows the main agent to delegate specific tasks to specialized agents with custom prompts, tool restrictions, and model configurations.

## Agent Architecture

### Core Concepts

1. **Main Agent**: The primary agent that orchestrates the conversation and can delegate tasks to subagents
2. **Subagents**: Specialized agents that handle specific tasks with customized configurations
3. **Agent Delegation**: Performed through the `Agent` tool (previously called `Task` tool)
4. **Agent Definitions**: Configuration objects that define agent behavior, capabilities, and restrictions

## Agent Definition Structure

### AgentDefinition Type

```typescript
export type AgentDefinition = {
    description: string;
    tools?: string[];
    prompt: string;
    model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
```

### AgentDefinition Fields

#### 1. `description` (required)
- **Type**: `string`
- **Purpose**: Human-readable description of the agent's purpose
- **Usage**: Helps identify what the agent specializes in

#### 2. `tools` (optional)
- **Type**: `string[]`
- **Purpose**: Array of tool names that this agent is allowed to use
- **Behavior**:
  - If specified: Agent can ONLY use the listed tools (whitelist)
  - If omitted: Agent inherits all tools from parent agent
- **Examples**: `["Bash", "Read", "Write"]`, `["Grep", "Glob", "Read"]`

#### 3. `prompt` (required)
- **Type**: `string`
- **Purpose**: System prompt that defines the agent's behavior and instructions
- **Usage**: Customizes how the agent approaches tasks
- **Best Practices**:
  - Be specific about the agent's role
  - Include task-specific guidelines
  - Define output format expectations

#### 4. `model` (optional)
- **Type**: `'sonnet' | 'opus' | 'haiku' | 'inherit'`
- **Purpose**: Specifies which Claude model variant to use for this agent
- **Default**: If not specified, inherits from parent agent
- **Options**:
  - `'sonnet'`: Claude Sonnet (balanced performance and cost)
  - `'opus'`: Claude Opus (highest capability)
  - `'haiku'`: Claude Haiku (fastest, most economical)
  - `'inherit'`: Use the same model as the parent agent

## Agent Tool (Task Delegation)

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

## Summary

The Claude Agent SDK's agent system provides:

1. **Flexible Delegation**: Delegate tasks to specialized subagents
2. **Tool Control**: Fine-grained control over tool access per agent
3. **Model Selection**: Choose appropriate models for different tasks
4. **Cost Optimization**: Use faster/cheaper models where appropriate
5. **Safety**: Restrict capabilities through tool whitelisting
6. **Scalability**: Build complex workflows with agent hierarchies

### Key Takeaways

- **AgentDefinition** defines agent behavior with `description`, `tools`, `prompt`, and `model`
- **Agent Tool** (AgentInput) delegates tasks with `subagent_type`, `description`, and `prompt`
- **Tool Restrictions** use whitelist approach via `tools` array
- **Model Options**: `sonnet`, `opus`, `haiku`, or `inherit`
- **Delegation Patterns**: Sequential, parallel, hierarchical, or safety-constrained
- **Best Practice**: Single responsibility, least privilege, appropriate model selection
