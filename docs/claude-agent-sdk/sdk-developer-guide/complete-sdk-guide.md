# Claude Agent SDK - Complete Developer Guide

**SDK Version**: 0.1.22
**Source**: SDK source code (`sdkTypes.d.ts`, `sdk.d.ts`, `sdk.mjs`)

---

## Table of Contents

1. [Introduction](#introduction)
2. [Installation & Setup](#installation--setup)
3. [Core Concepts](#core-concepts)
4. [Building Your First Agent](#building-your-first-agent)
5. [Advanced Agent Patterns](#advanced-agent-patterns)
6. [Custom Tool Development](#custom-tool-development)
7. [MCP Server Integration](#mcp-server-integration)
8. [Hooks & Middleware](#hooks--middleware)
9. [Permission Management](#permission-management)
10. [State & Session Management](#state--session-management)
11. [Production Deployment](#production-deployment)
12. [Testing & Debugging](#testing--debugging)
13. [Performance Optimization](#performance-optimization)
14. [Migration Guide](#migration-guide)

---

## Introduction

The Claude Agent SDK enables you to build sophisticated AI agents with:
- **Full API Access**: Direct Claude API integration
- **Tool System**: 17 built-in tools + custom tools
- **Agent Architecture**: Multi-agent workflows
- **Hook System**: 9 lifecycle hooks for customization
- **MCP Integration**: Connect to external tools and services
- **Type Safety**: Full TypeScript support

### Who This Guide Is For

- **SDK Developers**: Building custom agents programmatically
- **Integration Engineers**: Embedding Claude in applications
- **DevOps Engineers**: Deploying and scaling agents
- **Technical Architects**: Designing agent systems

---

## Installation & Setup

### Prerequisites

```json
{
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### Installation

```bash
npm install @anthropic-ai/claude-agent-sdk

# Or with specific version
npm install @anthropic-ai/claude-agent-sdk@0.1.22
```

### TypeScript Setup

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "lib": ["ES2022"],
    "types": ["node"]
  }
}
```

### API Key Configuration

```typescript
// Method 1: Environment variable (recommended)
process.env.ANTHROPIC_API_KEY = 'sk-ant-your-key-here';

// Method 2: Explicit in options (not recommended for production)
import { query } from '@anthropic-ai/claude-agent-sdk';

const result = await query({
  prompt: "Hello",
  options: {
    // Not recommended - exposes key in code
    apiKey: 'sk-ant-your-key-here'
  }
});
```

### Verification

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

// Test connection
const test = await query({
  prompt: "Hello, can you hear me?"
});

for await (const message of test) {
  if (message.type === 'assistant') {
    console.log('✓ SDK working!');
  }
}
```

---

## Core Concepts

### The Query Function

```typescript
/**
 * Main SDK export - creates an async generator for streaming
 * From sdk.d.ts
 */
export declare function query(params: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;
```

**Key Characteristics**:
- **Returns**: `AsyncGenerator<SDKMessage, void>`
- **Streaming**: Always streams responses
- **Async**: Non-blocking execution
- **Stateful**: Maintains conversation context

### Message Types

```typescript
/**
 * All message types from the SDK
 * From sdkTypes.d.ts
 */
export type SDKMessage = 
  | SDKUserMessage          // User input
  | SDKUserMessageReplay    // Replay of user message
  | SDKAssistantMessage     // Complete assistant response
  | SDKPartialAssistantMessage  // Streaming chunk
  | SDKResultMessage        // Final result with metrics
  | SDKSystemMessage        // System initialization
  | SDKCompactBoundaryMessage   // Context compaction marker
  | SDKHookResponseMessage; // Hook execution result
```

### The Query Interface

```typescript
/**
 * Query extends AsyncGenerator with control methods
 * From sdkTypes.d.ts
 */
export interface Query extends AsyncGenerator<SDKMessage, void> {
  // Control requests (streaming only)
  interrupt(): Promise<void>;
  setPermissionMode(mode: PermissionMode): Promise<void>;
  setModel(model?: string): Promise<void>;
  setMaxThinkingTokens(maxThinkingTokens: number | null): Promise<void>;
  
  // Metadata queries
  supportedCommands(): Promise<SlashCommand[]>;
  supportedModels(): Promise<ModelInfo[]>;
  mcpServerStatus(): Promise<McpServerStatus[]>;
  accountInfo(): Promise<AccountInfo>;
}
```

### Options Interface

```typescript
/**
 * Complete configuration options
 * From sdk.d.ts and sdkTypes.d.ts
 */
export type Options = {
  // API Configuration
  apiKey?: string;
  model?: string;
  fallbackModel?: string;
  maxThinkingTokens?: number;
  
  // Agent Configuration
  agents?: Record<string, AgentDefinition>;
  systemPrompt?: string | {
    type: 'preset';
    preset: 'claude_code';
    append?: string;
  };
  
  // Tool Configuration
  allowedTools?: string[];
  disallowedTools?: string[];
  
  // Permission Configuration
  permissionMode?: PermissionMode;
  canUseTool?: CanUseTool;
  additionalDirectories?: string[];
  
  // MCP Configuration
  mcpServers?: Record<string, McpServerConfig>;
  strictMcpConfig?: boolean;
  
  // Session Configuration
  cwd?: string;
  resume?: string;
  forkSession?: boolean;
  resumeSessionAt?: string;
  maxTurns?: number;
  
  // Hook Configuration
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;
  
  // Advanced Configuration
  includePartialMessages?: boolean;
  executable?: 'bun' | 'deno' | 'node';
  executableArgs?: string[];
  env?: { [envVar: string]: string | undefined };
  abortController?: AbortController;
};
```

---

## Building Your First Agent

### Basic Agent

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

async function basicAgent(userPrompt: string) {
  const result = await query({
    prompt: userPrompt,
    options: {
      model: 'sonnet',
      permissionMode: 'default'
    }
  });
  
  // Process streaming messages
  for await (const message of result) {
    switch (message.type) {
      case 'assistant':
        console.log('Assistant:', message.message.content);
        break;
        
      case 'result':
        console.log('Tokens used:', message.usage.input_tokens + message.usage.output_tokens);
        console.log('Cost: $', message.total_cost_usd.toFixed(4));
        break;
    }
  }
}

// Usage
await basicAgent("What files are in the current directory?");
```

### Agent with Custom System Prompt

```typescript
async function customAgent(prompt: string) {
  const result = await query({
    prompt,
    options: {
      systemPrompt: `
        You are a code review assistant specialized in TypeScript.
        
        Guidelines:
        - Focus on type safety and best practices
        - Identify potential bugs and edge cases
        - Suggest performance improvements
        - Be concise and actionable
      `
    }
  });
  
  for await (const message of result) {
    if (message.type === 'assistant') {
      console.log(message.message.content);
    }
  }
}
```

### Multi-Turn Conversation

```typescript
async function* conversationAgent(
  initialPrompt: string
): AsyncGenerator<string> {
  const conversation = await query({
    prompt: initialPrompt,
    options: {
      model: 'sonnet',
      maxTurns: 10  // Limit conversation length
    }
  });
  
  for await (const message of conversation) {
    if (message.type === 'assistant') {
      // Extract text content
      const content = message.message.content;
      const textBlocks = content.filter(block => block.type === 'text');
      const text = textBlocks.map(block => block.text).join('\n');
      
      yield text;
    } else if (message.type === 'result') {
      console.log('Conversation complete');
      console.log('Turns:', message.num_turns);
      console.log('Cost:', message.total_cost_usd);
    }
  }
}

// Usage
const conversation = conversationAgent("Help me refactor my code");
for await (const response of conversation) {
  console.log('Agent:', response);
  
  // Can send follow-up messages here if needed
}
```

---

## Advanced Agent Patterns

### Sub-Agent Delegation

```typescript
/**
 * Create agents that delegate to specialized sub-agents
 * From sdk.d.ts
 */
interface AgentDefinition {
  description: string;
  tools?: string[];      // Tool whitelist
  prompt: string;        // Agent's system prompt
  model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
}

async function delegatingAgent(task: string) {
  const result = await query({
    prompt: task,
    options: {
      model: 'sonnet',
      agents: {
        // Fast discovery agent
        'explorer': {
          description: 'Fast codebase exploration',
          tools: ['Glob', 'Grep', 'Read'],
          prompt: 'You are a fast code explorer. Find relevant files quickly.',
          model: 'haiku'
        },
        
        // Detailed analyzer
        'analyzer': {
          description: 'Deep code analysis',
          tools: ['Read', 'Grep', 'Task'],
          prompt: 'You are a code analyst. Provide detailed insights.',
          model: 'sonnet'
        },
        
        // Security reviewer
        'security': {
          description: 'Security audit',
          tools: ['Read', 'Grep'],
          prompt: 'You are a security expert. Find vulnerabilities.',
          model: 'opus'
        }
      }
    }
  });
  
  for await (const message of result) {
    if (message.type === 'assistant') {
      console.log('Main agent:', message.message.content);
    }
  }
}

// The main agent can now invoke sub-agents:
// "Use the explorer agent to find all authentication files"
// "Use the security agent to review this code"
```

### Stateful Agent with Memory

```typescript
class StatefulAgent {
  private conversationHistory: SDKUserMessage[] = [];
  private sessionId: string | null = null;
  
  async ask(prompt: string): Promise<string> {
    // Add to history
    this.conversationHistory.push({
      type: 'user',
      message: {
        role: 'user',
        content: prompt
      },
      parent_tool_use_id: null,
      session_id: this.sessionId || ''
    });
    
    const result = await query({
      // Pass conversation history for context
      prompt: this.conversationHistory.length === 1 
        ? prompt 
        : this.conversationHistory,
      options: {
        resume: this.sessionId || undefined
      }
    });
    
    let response = '';
    
    for await (const message of result) {
      if (message.type === 'system' && message.subtype === 'init') {
        // Capture session ID for resumption
        this.sessionId = message.session_id;
      } else if (message.type === 'assistant') {
        const content = message.message.content;
        const textBlocks = content.filter(b => b.type === 'text');
        response = textBlocks.map(b => b.text).join('\n');
      }
    }
    
    return response;
  }
  
  async resume(): Promise<void> {
    if (!this.sessionId) {
      throw new Error('No session to resume');
    }
    
    // Session will be resumed automatically via options.resume
  }
  
  clearHistory(): void {
    this.conversationHistory = [];
    this.sessionId = null;
  }
}

// Usage
const agent = new StatefulAgent();

await agent.ask("What's the capital of France?");
// Response: "Paris"

await agent.ask("What's its population?");
// Response: "About 2.1 million" (remembers we're talking about Paris!)
```

### Adaptive Model Selection

```typescript
class AdaptiveAgent {
  private modelUsage = {
    haiku: 0,
    sonnet: 0,
    opus: 0
  };
  
  async process(task: string) {
    // Analyze task complexity
    const complexity = this.analyzeComplexity(task);
    
    // Select appropriate model
    const model = this.selectModel(complexity);
    
    console.log(`Using ${model} for this task`);
    
    const result = await query({
      prompt: task,
      options: { model }
    });
    
    for await (const message of result) {
      if (message.type === 'result') {
        // Track model usage
        this.modelUsage[model as keyof typeof this.modelUsage]++;
        
        console.log(`Cost: $${message.total_cost_usd.toFixed(4)}`);
        console.log('Model distribution:', this.modelUsage);
      }
    }
  }
  
  private analyzeComplexity(task: string): 'simple' | 'medium' | 'complex' {
    const length = task.length;
    const keywords = ['analyze', 'complex', 'detailed', 'thorough'];
    
    if (task.length < 100 && !keywords.some(k => task.includes(k))) {
      return 'simple';
    } else if (keywords.some(k => task.includes(k))) {
      return 'complex';
    }
    return 'medium';
  }
  
  private selectModel(complexity: 'simple' | 'medium' | 'complex'): string {
    switch (complexity) {
      case 'simple':
        return 'haiku';   // Fast + cheap
      case 'complex':
        return 'opus';    // Accurate + expensive
      default:
        return 'sonnet';  // Balanced
    }
  }
}
```

---

## Custom Tool Development

### Creating a Custom MCP Tool

```typescript
/**
 * tool() function from SDK
 * From sdkTypes.d.ts
 */
import { tool, createSdkMcpServer } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

// Define custom tool
const weatherTool = tool(
  'get_weather',
  'Get current weather for a location',
  {
    location: z.string().describe('City name or coordinates'),
    units: z.enum(['celsius', 'fahrenheit']).optional()
  },
  async (args) => {
    // Tool implementation
    const weather = await fetchWeather(args.location, args.units);
    
    return {
      content: [{
        type: 'text',
        text: `Weather in ${args.location}: ${weather.temp}°${args.units === 'fahrenheit' ? 'F' : 'C'}, ${weather.condition}`
      }],
      isError: false
    };
  }
);

// Create MCP server with tools
const customServer = createSdkMcpServer({
  name: 'weather-service',
  version: '1.0.0',
  tools: [weatherTool]
});

// Use in query
const result = await query({
  prompt: "What's the weather in Tokyo?",
  options: {
    mcpServers: {
      'weather': customServer
    }
  }
});
```

### Advanced Tool with Error Handling

```typescript
const databaseTool = tool(
  'query_database',
  'Execute SQL query (read-only)',
  {
    query: z.string().describe('SQL SELECT query'),
    database: z.enum(['users', 'products', 'orders']),
    limit: z.number().min(1).max(100).optional().default(50)
  },
  async (args, extra) => {
    try {
      // Validate query is SELECT only
      if (!args.query.trim().toLowerCase().startsWith('select')) {
        return {
          content: [{
            type: 'text',
            text: 'Error: Only SELECT queries are allowed'
          }],
          isError: true
        };
      }
      
      // Execute query
      const connection = await getConnection(args.database);
      const results = await connection.query(
        args.query + ` LIMIT ${args.limit}`
      );
      
      // Format results
      return {
        content: [{
          type: 'text',
          text: JSON.stringify(results, null, 2)
        }],
        isError: false
      };
      
    } catch (error) {
      // Error handling
      console.error('Database error:', error);
      
      return {
        content: [{
          type: 'text',
          text: `Database error: ${error.message}`
        }],
        isError: true
      };
    }
  }
);
```

### Tool Suite Pattern

```typescript
/**
 * Create a suite of related tools
 */
function createDataTools() {
  const readTool = tool(
    'data_read',
    'Read data from storage',
    { key: z.string() },
    async (args) => {
      const data = await storage.get(args.key);
      return {
        content: [{ type: 'text', text: JSON.stringify(data) }],
        isError: false
      };
    }
  );
  
  const writeTool = tool(
    'data_write',
    'Write data to storage',
    {
      key: z.string(),
      value: z.any()
    },
    async (args) => {
      await storage.set(args.key, args.value);
      return {
        content: [{ type: 'text', text: 'Data written successfully' }],
        isError: false
      };
    }
  );
  
  const deleteTool = tool(
    'data_delete',
    'Delete data from storage',
    { key: z.string() },
    async (args) => {
      await storage.delete(args.key);
      return {
        content: [{ type: 'text', text: 'Data deleted successfully' }],
        isError: false
      };
    }
  );
  
  return createSdkMcpServer({
    name: 'data-service',
    version: '1.0.0',
    tools: [readTool, writeTool, deleteTool]
  });
}
```

---

## MCP Server Integration

### External MCP Servers

```typescript
/**
 * Configure external MCP servers
 * From sdkTypes.d.ts
 */
const result = await query({
  prompt: "List files and search GitHub issues",
  options: {
    mcpServers: {
      // stdio transport (external process)
      'filesystem': {
        type: 'stdio',
        command: 'npx',
        args: ['-y', '@modelcontextprotocol/server-filesystem', '/workspace'],
        env: {
          LOG_LEVEL: 'info'
        }
      },
      
      // HTTP transport (remote server)
      'github': {
        type: 'http',
        url: 'https://github-mcp.example.com',
        headers: {
          'Authorization': `Bearer ${process.env.GITHUB_TOKEN}`
        }
      },
      
      // SSE transport (server-sent events)
      'monitoring': {
        type: 'sse',
        url: 'https://monitoring.example.com/mcp',
        headers: {
          'API-Key': process.env.MONITORING_KEY
        }
      },
      
      // SDK transport (in-process)
      'custom': createSdkMcpServer({
        name: 'custom-tools',
        tools: [/* custom tools */]
      })
    }
  }
});
```

### MCP Server Health Monitoring

```typescript
async function monitorMCPServers(queryInstance: Query) {
  const status = await queryInstance.mcpServerStatus();
  
  for (const server of status) {
    console.log(`Server: ${server.name}`);
    console.log(`Status: ${server.status}`);
    
    if (server.serverInfo) {
      console.log(`Version: ${server.serverInfo.version}`);
    }
    
    if (server.status === 'failed') {
      console.error(`❌ Server ${server.name} is down!`);
      // Handle failure (reconnect, fallback, etc.)
    }
  }
}
```

---

## Hooks & Middleware

### Pre-Tool Use Hook (Validation)

```typescript
/**
 * Validate and modify tool calls before execution
 * From sdkTypes.d.ts
 */
const validationHook: HookCallback = async (input, toolUseID, { signal }) => {
  if (input.hook_event_name !== 'PreToolUse') {
    return { decision: 'approve' };
  }
  
  // Block dangerous bash commands
  if (input.tool_name === 'Bash') {
    const command = input.tool_input.command;
    
    const dangerous = ['rm -rf', 'mkfs', 'dd if=', '> /dev/'];
    if (dangerous.some(cmd => command.includes(cmd))) {
      return {
        decision: 'block',
        systemMessage: '🚫 Dangerous command blocked',
        hookSpecificOutput: {
          hookEventName: 'PreToolUse',
          permissionDecision: 'deny',
          permissionDecisionReason: 'Command contains dangerous patterns'
        }
      };
    }
  }
  
  return { decision: 'approve' };
};

// Use in query
const result = await query({
  prompt: "Do something",
  options: {
    hooks: {
      PreToolUse: [{
        hooks: [validationHook]
      }]
    }
  }
});
```

### Post-Tool Use Hook (Logging)

```typescript
const loggingHook: HookCallback = async (input) => {
  if (input.hook_event_name !== 'PostToolUse') {
    return {};
  }
  
  // Log to external service
  await logService.log({
    timestamp: new Date().toISOString(),
    tool: input.tool_name,
    input: input.tool_input,
    output: input.tool_response,
    sessionId: input.session_id
  });
  
  // Return async (fire-and-forget)
  return { async: true };
};
```

### Complete Hook Suite

```typescript
const hooks = {
  PreToolUse: [{
    hooks: [
      validationHook,
      securityHook,
      rateLimitHook
    ]
  }],
  
  PostToolUse: [{
    hooks: [
      loggingHook,
      metricsHook
    ]
  }],
  
  UserPromptSubmit: [{
    hooks: [
      async (input) => {
        // Add timestamp to all prompts
        return {
          hookSpecificOutput: {
            hookEventName: 'UserPromptSubmit',
            additionalContext: `\nTimestamp: ${new Date().toISOString()}`
          }
        };
      }
    ]
  }],
  
  SessionStart: [{
    hooks: [
      async (input) => {
        console.log('Session started:', input.session_id);
        return {};
      }
    ]
  }],
  
  SessionEnd: [{
    hooks: [
      async (input) => {
        console.log('Session ended:', input.reason);
        await cleanup(input.session_id);
        return {};
      }
    ]
  }]
};
```

---

## Permission Management

### Custom Permission Handler

```typescript
/**
 * Implement custom permission logic
 * From sdkTypes.d.ts
 */
const customPermissionHandler: CanUseTool = async (
  toolName,
  input,
  { signal, suggestions }
) => {
  // Always allow read operations
  if (['Read', 'Grep', 'Glob'].includes(toolName)) {
    return {
      behavior: 'allow',
      updatedInput: input
    };
  }
  
  // Check custom rules
  if (toolName === 'Bash') {
    const command = input.command as string;
    
    // Allow git commands
    if (command.startsWith('git ')) {
      return {
        behavior: 'allow',
        updatedInput: input
      };
    }
    
    // Block everything else
    return {
      behavior: 'deny',
      message: 'Only git commands are allowed',
      interrupt: false
    };
  }
  
  // Default: ask user
  return {
    behavior: 'allow',  // Or implement your own UI prompt
    updatedInput: input
  };
};

const result = await query({
  prompt: "Do something",
  options: {
    canUseTool: customPermissionHandler
  }
});
```

### Dynamic Permission Updates

```typescript
async function adaptivePermissions(queryInstance: Query) {
  // Start strict
  await queryInstance.setPermissionMode('default');
  
  // Process some work...
  
  // Detected safe environment, relax permissions
  await queryInstance.setPermissionMode('acceptEdits');
  
  // Continue work...
  
  // Task complete, return to strict
  await queryInstance.setPermissionMode('default');
}
```

---

## State & Session Management

### Session Persistence

```typescript
class PersistentAgent {
  async saveSession(queryInstance: Query): Promise<string> {
    // Sessions are automatically saved
    // Get session ID from system message
    let sessionId = '';
    
    for await (const message of queryInstance) {
      if (message.type === 'system' && message.subtype === 'init') {
        sessionId = message.session_id;
        break;
      }
    }
    
    // Store session ID
    await storage.set('last_session', sessionId);
    
    return sessionId;
  }
  
  async resumeSession(): Promise<Query> {
    const sessionId = await storage.get('last_session');
    
    return await query({
      prompt: "Continue where we left off",
      options: {
        resume: sessionId
      }
    });
  }
  
  async forkSession(sessionId: string, newPrompt: string): Promise<Query> {
    return await query({
      prompt: newPrompt,
      options: {
        resume: sessionId,
        forkSession: true  // Create copy
      }
    });
  }
}
```

---

## Production Deployment

### Error Handling & Resilience

```typescript
class ProductionAgent {
  private maxRetries = 3;
  private retryDelay = 1000;
  
  async execute(task: string): Promise<string> {
    let attempt = 0;
    
    while (attempt < this.maxRetries) {
      try {
        return await this.executeOnce(task);
      } catch (error) {
        attempt++;
        
        if (attempt >= this.maxRetries) {
          throw new Error(`Failed after ${this.maxRetries} attempts: ${error.message}`);
        }
        
        console.warn(`Attempt ${attempt} failed, retrying...`);
        await new Promise(resolve => setTimeout(resolve, this.retryDelay * attempt));
      }
    }
    
    throw new Error('Max retries exceeded');
  }
  
  private async executeOnce(task: string): Promise<string> {
    const result = await query({
      prompt: task,
      options: {
        model: 'sonnet',
        fallbackModel: 'haiku',  // Fallback if sonnet fails
        maxTurns: 10,
        abortController: new AbortController()  // For cancellation
      }
    });
    
    let response = '';
    
    for await (const message of result) {
      if (message.type === 'assistant') {
        const content = message.message.content;
        response = content.filter(b => b.type === 'text')
          .map(b => b.text).join('\n');
      } else if (message.type === 'result' && message.subtype !== 'success') {
        throw new Error('Execution failed');
      }
    }
    
    return response;
  }
}
```

### Monitoring & Metrics

```typescript
class MonitoredAgent {
  private metrics = {
    totalRequests: 0,
    totalCost: 0,
    totalTokens: 0,
    errorCount: 0,
    averageLatency: 0
  };
  
  async execute(task: string) {
    const startTime = Date.now();
    this.metrics.totalRequests++;
    
    try {
      const result = await query({
        prompt: task,
        options: { model: 'sonnet' }
      });
      
      for await (const message of result) {
        if (message.type === 'result') {
          // Track metrics
          this.metrics.totalCost += message.total_cost_usd;
          this.metrics.totalTokens += 
            message.usage.input_tokens + 
            message.usage.output_tokens;
          
          const latency = Date.now() - startTime;
          this.metrics.averageLatency = 
            (this.metrics.averageLatency * (this.metrics.totalRequests - 1) + latency) / 
            this.metrics.totalRequests;
          
          // Send to monitoring service
          await this.sendMetrics({
            latency,
            tokens: message.usage.input_tokens + message.usage.output_tokens,
            cost: message.total_cost_usd,
            success: true
          });
        }
      }
    } catch (error) {
      this.metrics.errorCount++;
      await this.sendMetrics({ success: false, error: error.message });
      throw error;
    }
  }
  
  getMetrics() {
    return {
      ...this.metrics,
      successRate: 
        (this.metrics.totalRequests - this.metrics.errorCount) / 
        this.metrics.totalRequests
    };
  }
  
  private async sendMetrics(data: any) {
    // Send to your monitoring service
    // await monitoring.track(data);
  }
}
```

---

## Testing & Debugging

### Unit Testing

```typescript
import { describe, it, expect } from 'vitest';

describe('CustomAgent', () => {
  it('should handle simple queries', async () => {
    const agent = new CustomAgent();
    const response = await agent.ask("What is 2+2?");
    
    expect(response).toContain('4');
  });
  
  it('should use appropriate model', async () => {
    const agent = new AdaptiveAgent();
    const spy = vi.spyOn(agent as any, 'selectModel');
    
    await agent.process("Simple task");
    expect(spy).toHaveReturnedWith('haiku');
    
    await agent.process("Complex analysis required");
    expect(spy).toHaveReturnedWith('opus');
  });
});
```

### Debugging Patterns

```typescript
// Enable debug logging
process.env.DEBUG = 'claude:*';

// Debug specific message types
const result = await query({
  prompt: "Debug test",
  options: {
    includePartialMessages: true  // Include streaming chunks
  }
});

for await (const message of result) {
  console.log('Message type:', message.type);
  console.log('Message:', JSON.stringify(message, null, 2));
}
```

---

## Performance Optimization

### Token Efficiency

```typescript
// Use prompt caching
const result = await query({
  prompt: "Analyze this codebase",
  options: {
    customSystemPrompt: largeCodebaseContext,  // Will be cached
    model: 'sonnet'
  }
});

// Use Haiku for discovery
const discovery = await query({
  prompt: "Find all API files",
  options: {
    model: 'haiku',  // 5x cheaper than sonnet
    agents: {
      'Explore': {
        description: 'Fast discovery',
        tools: ['Glob', 'Grep'],
        prompt: 'Find files quickly',
        model: 'haiku'
      }
    }
  }
});
```

### Parallel Execution

```typescript
// Execute multiple independent tasks in parallel
const [result1, result2, result3] = await Promise.all([
  query({ prompt: "Task 1", options: { model: 'haiku' } }),
  query({ prompt: "Task 2", options: { model: 'haiku' } }),
  query({ prompt: "Task 3", options: { model: 'haiku' } })
]);

// Process results
const responses = await Promise.all([
  collectResponse(result1),
  collectResponse(result2),
  collectResponse(result3)
]);
```

---

## Migration Guide

### From Claude Code SDK to Claude Agent SDK

```typescript
// OLD (Claude Code SDK)
import { claudeCode } from '@anthropic-ai/claude-code';

// NEW (Claude Agent SDK)
import { query } from '@anthropic-ai/claude-agent-sdk';

// API is largely compatible
const result = await query({
  prompt: "Your task",
  options: {
    // Same options structure
  }
});
```

**Breaking Changes**:
- Package name changed
- Some internal types renamed
- Default behavior changes

**Migration Checklist**:
- [ ] Update package.json dependency
- [ ] Update import statements
- [ ] Test all functionality
- [ ] Update type definitions
- [ ] Verify hook behavior
- [ ] Check tool configurations

---

## Summary

### SDK Architecture

```
query() → AsyncGenerator<SDKMessage>
          ↓
    [Streaming Pipeline]
          ↓
    Message Processing
          ↓
    Tool Execution
          ↓
    Hook Interception
          ↓
    Result Aggregation
```

### Key APIs

| API | Purpose |
|-----|---------|
| `query()` | Create streaming agent |
| `tool()` | Define custom tool |
| `createSdkMcpServer()` | Create MCP server |
| `Query.interrupt()` | Stop execution |
| `Query.setModel()` | Switch model |
| `Query.setPermissionMode()` | Change permissions |

### Best Practices Checklist

- [ ] Use TypeScript for type safety
- [ ] Implement error handling
- [ ] Add monitoring and metrics
- [ ] Use appropriate models
- [ ] Enable prompt caching
- [ ] Implement retry logic
- [ ] Handle interruptions
- [ ] Test thoroughly
- [ ] Monitor costs
- [ ] Document your agents