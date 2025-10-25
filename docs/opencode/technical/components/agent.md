# Agent Orchestration

**Path**: `packages/opencode/src/agent`
**Type**: Business Logic
**File Count**: 2

## Description

Agent orchestration and generation for predefined workflows.

## Purpose

The agent component enables defining and executing predefined agent workflows. These are reusable patterns that combine multiple operations, tools, and interactions into automated workflows.

## Key Features

- Predefined workflow execution
- Agent template system
- Multi-step automation
- Custom agent creation
- Agent composition

## Component Files

- `index.ts` - Agent orchestration logic
- `generator.ts` - Agent workflow generation

## Usage

### Run Predefined Agent

```bash
# Run agent workflow
opencode agent documentation-generator

# Run agent with parameters
opencode agent code-reviewer --files "src/**/*.ts"
```

### Define Custom Agent

```typescript
import { Agent } from './agent';

const agent = Agent.define({
  name: 'code-reviewer',
  description: 'Review code for best practices',
  steps: [
    {
      tool: 'glob',
      params: { pattern: '**/*.ts' }
    },
    {
      tool: 'read',
      params: { file_path: '${file}' }
    },
    {
      prompt: 'Review this code for best practices and security issues'
    }
  ]
});

await agent.execute();
```

## Agent Types

### Documentation Generator
Generates comprehensive documentation for codebases.

### Code Reviewer
Reviews code for best practices, security, and performance.

### Test Generator
Automatically generates test cases.

### Refactoring Agent
Identifies and suggests refactoring opportunities.

## Agent Workflow Format

```typescript
interface AgentWorkflow {
  name: string;
  description: string;
  steps: AgentStep[];
  config?: AgentConfig;
}

interface AgentStep {
  tool?: string;
  params?: Record<string, any>;
  prompt?: string;
  condition?: (context: any) => boolean;
}
```

## Related Documentation

- [CLI Commands](../api-reference.md#agentcommand)
- [Session Management](./session.md)
