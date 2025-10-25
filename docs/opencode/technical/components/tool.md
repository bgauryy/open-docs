# Tools

**Path**: `packages/opencode/src/tool`
**Type**: API Layer
**File Count**: 36

## Description

Tool implementations for AI actions including bash, edit, read, write, and more.

## Purpose

The tool component provides the AI with executable capabilities. Tools are functions that the AI can call to interact with the file system, execute commands, search code, and perform various operations.

## Key Features

- 36+ built-in tools
- Tool registry pattern
- Parameter validation
- Error handling
- Tool composition
- Custom tool support
- MCP tool integration
- Tool result streaming

## Component Files

### Core Tools
- `bash.ts` - Execute shell commands
- `edit.ts` - Edit files with string replacement
- `read.ts` - Read file contents
- `write.ts` - Write content to files
- `grep.ts` - Search using ripgrep
- `glob.ts` - Find files with patterns

### Code Intelligence Tools
- `lsp-diagnostics.ts` - Get LSP diagnostics
- `lsp-definition.ts` - Go to definition
- `lsp-references.ts` - Find references

### Utility Tools
- `todo.ts` - Task management
- `ask.ts` - Ask user questions
- Additional utility tools

### Tool Infrastructure
- `registry.ts` - Tool registration and execution
- `types.ts` - Tool type definitions
- `validator.ts` - Parameter validation

## Dependencies

### Internal Dependencies
- **Depends on**: `packages/opencode/src/file` (6 imports)
- **Depends on**: `packages/opencode/src/lsp` (2 imports)
- **Used by**: `packages/opencode/src/session` (12 imports)

## Tool Registry

### Register Tool

```typescript
import { ToolRegistry } from './tool/registry';

ToolRegistry.register({
  name: 'custom_tool',
  description: 'Perform custom operation',
  parameters: {
    type: 'object',
    properties: {
      param: {
        type: 'string',
        description: 'Parameter description'
      }
    },
    required: ['param']
  },
  handler: async (params) => {
    // Tool implementation
    return { success: true, result: params.param };
  }
});
```

### Execute Tool

```typescript
import { Tool } from './tool';

const result = await Tool.execute('bash', {
  command: 'npm test',
  timeout: 30000
});

console.log(result.stdout);
```

### List Tools

```typescript
const tools = Tool.list();

for (const tool of tools) {
  console.log(`${tool.name}: ${tool.description}`);
}
```

## Built-in Tools

### File Operations

#### read
Read file contents with optional line range.

```typescript
await Tool.execute('read', {
  file_path: '/path/to/file.ts',
  offset: 10,      // optional
  limit: 50        // optional
});
```

#### write
Write content to file.

```typescript
await Tool.execute('write', {
  file_path: '/path/to/file.ts',
  content: 'file content...'
});
```

#### edit
Edit file with exact string replacement.

```typescript
await Tool.execute('edit', {
  file_path: '/path/to/file.ts',
  old_string: 'old code',
  new_string: 'new code'
});
```

### Search Tools

#### grep
Search using ripgrep.

```typescript
await Tool.execute('grep', {
  pattern: 'function.*export',
  path: './src',
  type: 'ts'
});
```

#### glob
Find files matching pattern.

```typescript
await Tool.execute('glob', {
  pattern: '**/*.ts',
  path: './src'
});
```

### Execution Tools

#### bash
Execute shell commands.

```typescript
await Tool.execute('bash', {
  command: 'npm test',
  timeout: 30000
});
```

### Code Intelligence

#### lsp_diagnostics
Get language server diagnostics.

```typescript
await Tool.execute('lsp_diagnostics', {
  uri: 'file:///path/to/file.ts'
});
```

### Task Management

#### todowrite
Write todo list.

```typescript
await Tool.execute('todowrite', {
  todos: [
    {
      content: 'Implement feature',
      status: 'pending',
      activeForm: 'Implementing feature'
    }
  ]
});
```

## Tool Definition

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  parameters: JSONSchema;
  handler: (params: any) => Promise<any>;
  before?: (params: any) => Promise<void>;
  after?: (result: any) => Promise<void>;
  validate?: (params: any) => boolean;
}
```

## Parameter Validation

Tools use JSON Schema for parameter validation:

```typescript
ToolRegistry.register({
  name: 'example_tool',
  description: 'Example tool',
  parameters: {
    type: 'object',
    properties: {
      required_param: {
        type: 'string',
        description: 'Required parameter',
        minLength: 1
      },
      optional_param: {
        type: 'number',
        description: 'Optional parameter',
        minimum: 0,
        maximum: 100,
        default: 50
      }
    },
    required: ['required_param']
  },
  handler: async (params) => {
    // params are validated before reaching here
    return { result: 'success' };
  }
});
```

## Error Handling

```typescript
try {
  const result = await Tool.execute('bash', {
    command: 'invalid-command'
  });
} catch (error) {
  if (error.code === 'VALIDATION_ERROR') {
    console.error('Invalid parameters:', error.message);
  } else if (error.code === 'EXECUTION_ERROR') {
    console.error('Tool failed:', error.message);
  } else if (error.code === 'TIMEOUT') {
    console.error('Tool timed out');
  }
}
```

### Error Types

```typescript
class ToolError extends Error {
  code: string;
  tool: string;
  params?: any;

  constructor(code: string, message: string, tool: string) {
    super(message);
    this.code = code;
    this.tool = tool;
  }
}
```

## Tool Hooks

Tools support lifecycle hooks:

```typescript
ToolRegistry.register({
  name: 'example',
  description: 'Example with hooks',
  parameters: { /* ... */ },

  // Before execution
  before: async (params) => {
    console.log('Executing with params:', params);
    // Validate, log, modify params
  },

  // Main execution
  handler: async (params) => {
    return { result: 'success' };
  },

  // After execution
  after: async (result) => {
    console.log('Result:', result);
    // Log, transform result, trigger events
  }
});
```

## Tool Composition

Create composite tools from existing tools:

```typescript
ToolRegistry.register({
  name: 'refactor_function',
  description: 'Refactor a function across multiple files',
  parameters: {
    type: 'object',
    properties: {
      functionName: { type: 'string' },
      newName: { type: 'string' }
    }
  },
  handler: async (params) => {
    // 1. Find files containing function
    const files = await Tool.execute('grep', {
      pattern: `function ${params.functionName}`,
      type: 'ts'
    });

    // 2. Edit each file
    for (const file of files) {
      await Tool.execute('edit', {
        file_path: file,
        old_string: `function ${params.functionName}`,
        new_string: `function ${params.newName}`
      });
    }

    // 3. Run tests
    const testResult = await Tool.execute('bash', {
      command: 'npm test'
    });

    return {
      filesModified: files.length,
      testsPass: testResult.exitCode === 0
    };
  }
});
```

## Custom Tools

### Simple Custom Tool

```typescript
ToolRegistry.register({
  name: 'hello',
  description: 'Say hello',
  parameters: {
    type: 'object',
    properties: {
      name: { type: 'string' }
    }
  },
  handler: async ({ name }) => {
    return { message: `Hello, ${name}!` };
  }
});
```

### Advanced Custom Tool

```typescript
import { Tool } from './tool';
import { z } from 'zod';

class DatabaseTool extends Tool {
  name = 'database_query';
  description = 'Execute database query';

  // Use Zod for validation
  schema = z.object({
    query: z.string(),
    params: z.array(z.any()).optional()
  });

  async execute(params: z.infer<typeof this.schema>) {
    const { query, params: queryParams } = params;

    // Connect to database
    const db = await this.getConnection();

    // Execute query
    const result = await db.query(query, queryParams);

    return {
      rows: result.rows,
      rowCount: result.rowCount
    };
  }

  private async getConnection() {
    // Database connection logic
  }
}

ToolRegistry.register(new DatabaseTool());
```

## Tool Results

Tool results should be structured:

```typescript
interface ToolResult {
  success: boolean;
  data?: any;
  error?: string;
  metadata?: {
    duration?: number;
    tokens?: number;
    cached?: boolean;
  };
}
```

## Streaming Tool Results

For long-running tools:

```typescript
ToolRegistry.register({
  name: 'long_running_task',
  description: 'Task that streams progress',
  parameters: { /* ... */ },
  handler: async function* (params) {
    yield { type: 'progress', message: 'Starting...', percent: 0 };

    // Do work
    await doStep1();
    yield { type: 'progress', message: 'Step 1 complete', percent: 33 };

    await doStep2();
    yield { type: 'progress', message: 'Step 2 complete', percent: 66 };

    await doStep3();
    yield { type: 'progress', message: 'Step 3 complete', percent: 100 };

    return { success: true };
  }
});
```

## Tool Security

### Sandboxing

Tools execute in a sandboxed environment:

```typescript
const sandbox = {
  cwd: '/workspace',              // Restrict working directory
  env: { NODE_ENV: 'development' }, // Filter environment
  timeout: 120000,                // Max execution time
  maxMemory: 1024 * 1024 * 1024, // Max memory (1GB)
  allowNetwork: false,            // Network access
  allowFileWrite: true,           // File write access
  allowedCommands: ['npm', 'git'] // Command whitelist
};

await Tool.execute('bash', {
  command: 'npm test'
}, { sandbox });
```

### Permission Model

```typescript
ToolRegistry.register({
  name: 'dangerous_operation',
  description: 'Requires elevated permissions',
  permissions: ['file:write', 'network:all'],
  handler: async (params) => {
    // Only executes if user granted permissions
  }
});
```

## Tool Caching

Cache tool results for performance:

```typescript
ToolRegistry.register({
  name: 'expensive_operation',
  description: 'Expensive operation',
  cache: {
    enabled: true,
    ttl: 3600,              // 1 hour
    key: (params) => JSON.stringify(params)
  },
  handler: async (params) => {
    // Result cached for 1 hour
    return await expensiveOperation(params);
  }
});
```

## Tool Metrics

Track tool usage:

```typescript
const metrics = Tool.getMetrics();

console.log('Most used tools:', metrics.topTools);
console.log('Average execution time:', metrics.avgDuration);
console.log('Error rate:', metrics.errorRate);
console.log('Cache hit rate:', metrics.cacheHitRate);

// Per-tool metrics
const bashMetrics = Tool.getMetrics('bash');
console.log('Bash executions:', bashMetrics.count);
console.log('Bash avg duration:', bashMetrics.avgDuration);
```

## MCP Tool Integration

Integrate MCP server tools:

```typescript
import { MCP } from '../mcp';

// Connect to MCP server
const mcp = await MCP.connect({ server: 'filesystem', ... });

// Get tools from MCP
const mcpTools = await mcp.listTools();

// Register MCP tools
for (const tool of mcpTools) {
  ToolRegistry.register({
    name: `mcp_filesystem_${tool.name}`,
    description: tool.description,
    parameters: tool.inputSchema,
    handler: async (params) => {
      return await mcp.callTool(tool.name, params);
    }
  });
}
```

## Best Practices

1. **Descriptive Names**: Use clear, action-oriented names
2. **Detailed Descriptions**: Help AI understand when to use tool
3. **Validate Parameters**: Use JSON Schema validation
4. **Handle Errors Gracefully**: Return useful error messages
5. **Document Examples**: Provide usage examples
6. **Consider Performance**: Cache when appropriate
7. **Security First**: Validate and sanitize inputs
8. **Atomic Operations**: Keep tools focused and atomic

## Testing Tools

```typescript
import { Tool } from './tool';

describe('CustomTool', () => {
  it('should execute successfully', async () => {
    const result = await Tool.execute('custom_tool', {
      param: 'value'
    });

    expect(result.success).toBe(true);
  });

  it('should validate parameters', async () => {
    await expect(
      Tool.execute('custom_tool', {})
    ).rejects.toThrow('Missing required parameter');
  });
});
```

## Related Documentation

- [Tool Execution Flow](../flows/tool-execution-flow.md)
- [Tools API Reference](../api-reference.md#tools)
- [Session Management](./session.md)
- [File Operations](./file.md)
- [MCP Integration](./mcp.md)
