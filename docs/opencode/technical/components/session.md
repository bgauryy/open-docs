# Session Management

**Path**: `packages/opencode/src/session`
**Type**: Business Logic
**File Count**: 11

## Description

Session management, message handling, and prompts for AI conversations.

## Purpose

The session component manages stateful conversations with AI models. It handles message history, context management, tool orchestration, prompt engineering, and session persistence.

## Key Features

- Session creation and resumption
- Conversation history management
- Context window management
- Tool orchestration
- System prompt engineering
- Session persistence
- Multi-turn conversations
- Stream handling

## Component Files

- `index.ts` - Main session manager
- `message.ts` - Message handling
- `history.ts` - Conversation history
- `prompt.ts` - System prompts
- `context.ts` - Context management
- `storage.ts` - Session persistence
- `stream.ts` - Response streaming
- Additional session utilities

## Dependencies

### Internal Dependencies
- **Depends on**: `packages/opencode/src/provider` (8 imports)
- **Depends on**: `packages/opencode/src/tool` (12 imports)
- **Used by**: `packages/opencode/src/cli` (15 imports)
- **Used by**: `packages/opencode/src/server` (5 imports)
- **Used by**: `packages/opencode/src/acp` (3 imports)

## Usage

### Create Session

```typescript
import { Session } from './session';

// Create new session
const session = await Session.create({
  model: 'claude-3-sonnet',
  tools: ['bash', 'edit', 'read', 'write'],
  systemPrompt: 'You are a helpful coding assistant.',
  temperature: 0.7,
  maxTokens: 4096
});

console.log('Session ID:', session.id);
```

### Send Message

```typescript
// Send message and stream response
const response = session.message('Help me build a REST API');

for await (const chunk of response) {
  if (chunk.type === 'text') {
    process.stdout.write(chunk.content);
  } else if (chunk.type === 'tool_call') {
    console.log(`\nCalling tool: ${chunk.tool}`);
  } else if (chunk.type === 'tool_result') {
    console.log(`Tool result: ${chunk.result}`);
  }
}
```

### Resume Session

```typescript
// Resume existing session
const session = await Session.resume('abc123');

// Continue conversation
const response = await session.message('What was my previous question?');
```

### List Sessions

```typescript
// List all sessions
const sessions = await Session.list();

for (const session of sessions) {
  console.log(`${session.id}: ${session.messageCount} messages`);
}

// Get session info
const info = await Session.info('abc123');
console.log('Created:', info.createdAt);
console.log('Last message:', info.lastMessage);
```

### Delete Session

```typescript
// Delete session
await Session.delete('abc123');

// Delete all sessions
await Session.deleteAll();
```

## Session Configuration

```typescript
interface SessionOptions {
  // Required
  model: string;                    // AI model name

  // Optional
  tools?: string[];                 // Available tools
  systemPrompt?: string;            // Custom system prompt
  temperature?: number;             // 0-1, default 0.7
  maxTokens?: number;              // Max response tokens
  stopSequences?: string[];         // Stop sequences
  contextWindow?: number;           // Max context tokens
  autoSave?: boolean;              // Auto-save after each message
  sessionId?: string;              // Resume existing session
}
```

## Message Types

```typescript
interface Message {
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: string;
  tokens?: number;
  toolCalls?: ToolCall[];
  toolResults?: ToolResult[];
}

interface ToolCall {
  id: string;
  tool: string;
  params: Record<string, any>;
}

interface ToolResult {
  id: string;
  result: any;
  error?: string;
}
```

## Session State

```typescript
interface SessionState {
  id: string;
  model: string;
  createdAt: string;
  lastMessage: string;
  messageCount: number;
  tokenUsage: {
    input: number;
    output: number;
    total: number;
  };
  tools: string[];
  status: 'active' | 'idle' | 'error';
}
```

## Context Management

Sessions automatically manage context window:

```typescript
const session = await Session.create({
  model: 'claude-3-sonnet',
  contextWindow: 200000  // Max tokens
});

// Session automatically truncates old messages when context is full
await session.message('Long conversation...');

// Get current context usage
const usage = session.getContextUsage();
console.log(`Using ${usage.current} / ${usage.max} tokens`);
```

## System Prompts

### Default Prompt

```
You are an expert coding assistant. You help users write, understand, and debug code.

You have access to tools for:
- Reading and writing files
- Executing shell commands
- Searching code
- Getting diagnostics

Always explain your reasoning and ask for clarification when needed.
```

### Custom Prompts

```typescript
const session = await Session.create({
  model: 'claude-3-sonnet',
  systemPrompt: `You are a Python expert specializing in data science.

You should:
- Prefer pandas and numpy for data manipulation
- Use matplotlib for visualizations
- Write clean, well-documented code
- Suggest best practices`
});
```

### Prompt Templates

```typescript
import { PromptTemplate } from './session';

const template = PromptTemplate.create({
  role: 'backend developer',
  language: 'typescript',
  framework: 'express',
  style: 'functional'
});

const session = await Session.create({
  model: 'claude-3-sonnet',
  systemPrompt: template.render()
});
```

## Tool Integration

### Register Tools

```typescript
const session = await Session.create({
  model: 'claude-3-sonnet',
  tools: ['bash', 'edit', 'read', 'write', 'grep', 'glob']
});

// Register custom tool
session.registerTool({
  name: 'custom_tool',
  description: 'Perform custom operation',
  parameters: {
    type: 'object',
    properties: {
      param: { type: 'string' }
    }
  },
  handler: async (params) => {
    return { result: 'success' };
  }
});
```

### Tool Execution Flow

1. AI requests tool call
2. Session validates tool and parameters
3. Session executes tool via registry
4. Tool result returned to session
5. Session sends result back to AI
6. AI continues with result

## Session Persistence

Sessions are automatically persisted to disk:

```typescript
// Sessions stored in ~/.opencode/sessions/
~/.opencode/sessions/
  abc123/
    metadata.json    // Session info
    messages.json    // Message history
    state.json       // Current state
```

### Manual Save/Load

```typescript
// Save session manually
await session.save();

// Load session from disk
const session = await Session.load('abc123');

// Export session
const exported = await session.export();
console.log(JSON.stringify(exported, null, 2));

// Import session
const session = await Session.import(exportedData);
```

## Streaming

Sessions support streaming responses:

```typescript
const stream = session.message('Help me build a feature');

for await (const chunk of stream) {
  // Process chunks as they arrive
  if (chunk.type === 'text') {
    process.stdout.write(chunk.content);
  }
}
```

### Stream Events

```typescript
const stream = session.message('Help me');

stream.on('start', () => {
  console.log('Response started');
});

stream.on('text', (chunk) => {
  console.log('Text:', chunk.content);
});

stream.on('tool_call', (call) => {
  console.log('Tool call:', call.tool);
});

stream.on('end', (summary) => {
  console.log('Done. Tokens:', summary.tokens);
});

stream.on('error', (error) => {
  console.error('Error:', error);
});
```

## Multi-Turn Conversations

```typescript
const session = await Session.create({
  model: 'claude-3-sonnet'
});

// Turn 1
await session.message('Create a function to calculate fibonacci');

// Turn 2
await session.message('Now add memoization to optimize it');

// Turn 3
await session.message('Write tests for the function');

// Session maintains full context
```

## Error Handling

```typescript
try {
  const session = await Session.create({
    model: 'claude-3-sonnet'
  });

  await session.message('Help me build a feature');
} catch (error) {
  if (error.code === 'RATE_LIMIT') {
    console.error('Rate limit exceeded');
  } else if (error.code === 'CONTEXT_LENGTH_EXCEEDED') {
    console.error('Context too long, clearing history');
    await session.clearHistory({ keepLast: 10 });
  } else if (error.code === 'TOOL_ERROR') {
    console.error('Tool execution failed:', error.message);
  }
}
```

## Session Events

```typescript
const session = await Session.create({
  model: 'claude-3-sonnet'
});

// Listen to events
session.on('message', (msg) => {
  console.log('New message:', msg.role);
});

session.on('tool_call', (call) => {
  console.log('Tool called:', call.tool);
});

session.on('token_usage', (usage) => {
  console.log('Tokens used:', usage.total);
});

session.on('error', (error) => {
  console.error('Session error:', error);
});

session.on('save', () => {
  console.log('Session saved');
});
```

## Advanced Features

### Message Editing

```typescript
// Get message history
const messages = await session.getMessages();

// Edit a message
await session.editMessage(messageId, {
  content: 'Updated message content'
});

// Delete a message
await session.deleteMessage(messageId);
```

### Context Pruning

```typescript
// Auto-prune old messages
const session = await Session.create({
  model: 'claude-3-sonnet',
  contextWindow: 200000,
  pruneStrategy: 'sliding-window',  // Keep recent messages
  pruneThreshold: 0.9               // Prune at 90% full
});

// Manual pruning
await session.pruneContext({
  keepLast: 20,           // Keep last 20 messages
  keepSystem: true,       // Always keep system prompt
  keepImportant: true     // Keep messages marked important
});
```

### Session Forking

```typescript
// Fork session at specific point
const forked = await session.fork({
  fromMessage: messageId,
  newSessionId: 'forked-abc123'
});

// Explore alternative conversation path
await forked.message('Try a different approach');
```

## Performance Considerations

- Message history cached in memory
- Lazy loading of old messages
- Efficient context window management
- Tool execution parallelization
- Streaming reduces latency
- Session pooling for server mode

## Related Documentation

- [Session.create API](../api-reference.md#sessioncreate)
- [Session.message API](../api-reference.md#sessionmessage)
- [Interactive Chat Flow](../flows/run-command-flow.md)
- [Provider Integration](./provider.md)
- [Tool Execution](./tool.md)
