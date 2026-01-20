# API Reference

## Overview

This document provides a comprehensive reference for Gemini CLI's public APIs, including core exports, built-in tools, CLI commands, and extension points.

## Core Package Exports

The `@google/gemini-cli-core` package exports the following public APIs:

### Configuration

```typescript
import { 
  Storage,
  Config,
  DEFAULT_GEMINI_MODEL,
  DEFAULT_GEMINI_FLASH_MODEL,
  DEFAULT_GEMINI_FLASH_LITE_MODEL,
  DEFAULT_GEMINI_EMBEDDING_MODEL,
} from '@google/gemini-cli-core';
```

| Export | Type | Description |
|--------|------|-------------|
| `Storage` | Class | Configuration storage management |
| `Config` | Class | Central configuration container |
| `DEFAULT_GEMINI_MODEL` | Constant | Default model identifier |
| `DEFAULT_GEMINI_MODEL_AUTO` | Constant | Auto-routing model |

### Telemetry

```typescript
import {
  ClearcutLogger,
  IdeConnectionEvent,
  ExtensionInstallEvent,
  logModelSlashCommand,
} from '@google/gemini-cli-core';
```

### Utilities

```typescript
import {
  serializeTerminalToObject,
  detectIdeFromEnv,
  getCodeAssistServer,
  getExperiments,
} from '@google/gemini-cli-core';
```

## GeminiChat Class

The main class for managing chat sessions with Gemini.

### Constructor

```typescript
class GeminiChat {
  constructor(
    config: Config,
    systemInstruction?: string,
    tools?: Tool[],
    history?: Content[],
    resumedSessionData?: ResumedSessionData
  )
}
```

### Methods

#### `send(message: string): AsyncGenerator<ContentChunk>`

Sends a message to the model and yields response chunks.

```typescript
const chat = new GeminiChat(config);
for await (const chunk of chat.send('Hello, Gemini!')) {
  console.log(chunk.text);
}
```

#### `setSystemInstruction(instruction: string): void`

Updates the system instruction for the conversation.

```typescript
chat.setSystemInstruction('You are a helpful coding assistant.');
```

## Built-in Tools

### File System Tools

#### GlobTool

Search for files matching glob patterns.

```typescript
interface GlobToolParams {
  pattern: string;           // Glob pattern (e.g., "**/*.ts")
  case_sensitive?: boolean;  // Case sensitivity (default: false)
}
```

**Example:**
```
Find all TypeScript files: {"pattern": "**/*.ts"}
```

#### ReadFileTool

Read contents of a single file.

```typescript
interface ReadFileToolParams {
  path: string;        // File path to read
  offset?: number;     // Starting line (1-indexed)
  limit?: number;      // Number of lines to read
}
```

#### ReadManyFilesTool

Batch read multiple files efficiently.

```typescript
interface ReadManyFilesToolParams {
  paths: string[];     // Array of file paths
}
```

#### EditTool

Create or modify files.

```typescript
interface EditToolParams {
  file_path: string;   // Target file path
  new_string: string;  // Content to write/insert
  old_string?: string; // Content to replace (for edits)
  create_file?: boolean; // Create if doesn't exist
}
```

#### LSTool

List directory contents.

```typescript
interface LSToolParams {
  path: string;        // Directory path
  max_depth?: number;  // Recursion depth
}
```

### Search Tools

#### GrepTool

Search file contents using regex.

```typescript
interface GrepToolParams {
  pattern: string;     // Search pattern (regex)
  path?: string;       // Directory to search
  include?: string;    // File pattern to include
}
```

#### RipGrepTool

Fast search using ripgrep.

```typescript
interface RipGrepToolParams {
  pattern: string;     // Search pattern
  path?: string;       // Search path
  case_sensitive?: boolean;
  file_type?: string;  // e.g., "ts", "py"
}
```

### Shell Tools

#### ShellTool

Execute shell commands.

```typescript
interface ShellToolParams {
  command: string;     // Command to execute
  timeout?: number;    // Timeout in ms
  cwd?: string;        // Working directory
}
```

**Security:** Requires user confirmation before execution.

### Web Tools

#### WebFetchTool

Fetch web page content.

```typescript
interface WebFetchToolParams {
  url: string;         // URL to fetch
  headers?: Record<string, string>;
}
```

#### WebSearchTool

Perform Google Search queries.

```typescript
interface WebSearchToolParams {
  query: string;       // Search query
}
```

**Note:** Uses Google Search grounding for real-time information.

### Task Management

#### WriteTodosTool

Manage task lists.

```typescript
interface WriteTodosToolParams {
  todos: Array<{
    id: string;
    content: string;
    status: 'pending' | 'in_progress' | 'completed' | 'cancelled';
  }>;
  merge?: boolean;     // Merge with existing todos
}
```

### Agent Tools

#### DelegateToAgentTool

Delegate tasks to specialized agents.

```typescript
interface DelegateToAgentToolParams {
  agent: string;       // Agent identifier
  task: string;        // Task description
  context?: Record<string, unknown>;
}
```

### MCP Tools

#### DiscoveredMCPTool

Wrapper for tools discovered from MCP servers.

```typescript
// MCP tools are dynamically discovered and registered
const mcpTools = await mcpClientManager.discoverTools();
```

## CLI Commands

### Main Commands

| Command | Description |
|---------|-------------|
| `gemini` | Start interactive session |
| `gemini -p "prompt"` | Non-interactive mode |
| `gemini -m model` | Specify model |
| `gemini --help` | Show help |

### Options

| Option | Description |
|--------|-------------|
| `-p, --prompt` | Initial prompt (non-interactive) |
| `-m, --model` | Model to use |
| `--output-format` | Output format: text, json, stream-json |
| `--include-directories` | Additional context directories |
| `--sandbox` | Sandbox mode: false, docker, podman |

### Subcommands

#### `gemini extensions`

Manage extensions.

```bash
gemini extensions install <package>
gemini extensions enable <name>
gemini extensions disable <name>
gemini extensions list
```

#### `gemini skills`

Manage custom skills.

```bash
gemini skills list
gemini skills enable <name>
gemini skills disable <name>
```

#### `gemini mcp`

Manage MCP servers.

```bash
gemini mcp list
gemini mcp configure <server>
```

## Authentication

### OAuth (Login with Google)

```typescript
// The CLI handles OAuth flow automatically
// Users are prompted to authenticate on first run
```

### API Key

```bash
export GEMINI_API_KEY="your-api-key"
gemini
```

### Vertex AI

```bash
export GOOGLE_API_KEY="your-api-key"
export GOOGLE_GENAI_USE_VERTEXAI=true
gemini
```

### Programmatic Authentication

```typescript
import { Config } from '@google/gemini-cli-core';

const config = new Config({
  apiKey: process.env.GEMINI_API_KEY,
  // or
  oauth: {
    clientId: 'your-client-id',
    clientSecret: 'your-client-secret',
  },
});
```

## MCP Integration

### Configuration

MCP servers are configured in `~/.gemini/config.yaml`:

```yaml
mcpServers:
  my-server:
    command: npx
    args: ["-y", "@my-org/mcp-server"]
    env:
      API_KEY: ${MY_API_KEY}
```

### McpClientManager

```typescript
class McpClientManager {
  // Connect to all configured servers
  async initialize(): Promise<void>;
  
  // Discover tools from connected servers
  async discoverTools(): Promise<DiscoveredMCPTool[]>;
  
  // Execute a tool call
  async callTool(
    server: string,
    tool: string,
    params: Record<string, unknown>
  ): Promise<ToolResult>;
}
```

## Agent System

### AgentRegistry

```typescript
class AgentRegistry {
  // Register a new agent
  protected registerAgent(name: string, spec: AgentSpec): void;
  
  // Get agent by name
  getAgent(name: string): AgentSpec | undefined;
  
  // List all agents
  listAgents(): string[];
}
```

### LocalAgentExecutor

```typescript
class LocalAgentExecutor<TOutput extends z.ZodTypeAny> {
  constructor(
    config: Config,
    agentSpec: AgentSpec,
    outputSchema: TOutput
  );
  
  // Execute the agent task
  async execute(
    task: string,
    context?: Record<string, unknown>
  ): Promise<z.infer<TOutput>>;
}
```

## Prompt Management

### PromptRegistry

```typescript
class PromptRegistry {
  // Register a prompt template
  register(name: string, template: string): void;
  
  // Get rendered prompt
  render(name: string, variables: Record<string, string>): string;
}
```

## Token Management

### Token Counting

```typescript
import { estimateTokenCountSync } from '@google/gemini-cli-core';

const tokens = estimateTokenCountSync(content);
```

### Token Caching

Token caching is automatic for repeated context. Configure via:

```typescript
const config = new Config({
  // Enable caching for context files
  cacheContextFiles: true,
});
```

## Error Handling

### Error Classes

```typescript
import {
  FatalError,
  AgentExecutionStoppedError,
  AgentExecutionBlockedError,
  ModelNotFoundError,
} from '@google/gemini-cli-core';
```

### Error Handling Pattern

```typescript
try {
  await chat.send(prompt);
} catch (error) {
  if (error instanceof AgentExecutionStoppedError) {
    // User stopped execution
  } else if (error instanceof AgentExecutionBlockedError) {
    // Safety block triggered
  } else if (error instanceof ModelNotFoundError) {
    // Invalid model specified
  }
}
```

## Type Definitions

### Content Types

```typescript
interface Content {
  role: 'user' | 'model' | 'function';
  parts: Part[];
}

interface Part {
  text?: string;
  functionCall?: FunctionCall;
  functionResponse?: FunctionResponse;
}
```

### Tool Types

```typescript
interface ToolResult {
  success: boolean;
  output?: string;
  error?: string;
}

interface ToolCallConfirmationDetails {
  tool: string;
  params: Record<string, unknown>;
  description: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}
```

## Extension Points

### Creating Custom Tools

```typescript
import { BaseDeclarativeTool, ToolResult } from '@google/gemini-cli-core';

export class MyCustomTool extends BaseDeclarativeTool<MyParams, ToolResult> {
  static readonly toolName = 'my_custom_tool';
  
  static readonly description = 'Description of my tool';
  
  static readonly parameterSchema = z.object({
    param1: z.string(),
  });
  
  async execute(signal: AbortSignal): Promise<ToolResult> {
    // Implementation
    return { success: true, output: 'Result' };
  }
}
```

### Creating Custom Commands

```typescript
// packages/cli/src/commands/my-command.ts
import type { CommandModule } from 'yargs';

export const myCommand: CommandModule = {
  command: 'my-command',
  describe: 'Description of my command',
  handler: async (argv) => {
    // Implementation
  },
};
```
