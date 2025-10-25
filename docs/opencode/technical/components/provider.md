# Provider Integration

**Path**: `packages/opencode/src/provider`
**Type**: Integration Layer
**File Count**: 4

## Description

AI provider integration for Anthropic, OpenAI, Google, and more.

## Purpose

The provider component abstracts AI provider APIs into a unified interface, enabling OpenCode to work with multiple AI providers seamlessly. It handles provider-specific details like authentication, streaming, tool calling, and error handling.

## Key Features

- Unified provider interface
- Multi-provider support
- Streaming responses
- Tool calling integration
- Token usage tracking
- Rate limiting and retries
- Provider-specific optimizations

## Component Files

- `provider.ts` - Main provider abstraction
- `anthropic.ts` - Anthropic (Claude) integration
- `openai.ts` - OpenAI integration
- `config.ts` - Provider configuration

## Dependencies

### Internal Dependencies
- Used by `packages/opencode/src/session` (8 imports)

### External Dependencies
- `ai` - Unified AI SDK (Vercel AI SDK)
- `@anthropic-ai/sdk` - Anthropic official SDK
- `openai` - OpenAI official SDK

## Supported Providers

### Anthropic (Claude)
- Claude 3 Opus
- Claude 3 Sonnet
- Claude 3 Haiku
- Claude 3.5 Sonnet

### OpenAI
- GPT-4
- GPT-4 Turbo
- GPT-3.5 Turbo
- Custom models

### Google Vertex AI
- Gemini Pro
- Gemini Pro Vision

### Amazon Bedrock
- Claude on Bedrock
- Other Bedrock models

### Local Models
- Ollama
- LM Studio
- Custom endpoints

## Usage

### Basic Provider Usage

```typescript
import { Provider } from './provider';

// Configure provider
const provider = Provider.create({
  name: 'anthropic',
  model: 'claude-3-sonnet-20240229',
  apiKey: process.env.ANTHROPIC_API_KEY
});

// Stream response
const stream = provider.stream({
  messages: [
    { role: 'user', content: 'Hello!' }
  ]
});

for await (const chunk of stream) {
  process.stdout.write(chunk.content);
}
```

### With Tools

```typescript
const stream = provider.stream({
  messages: [
    { role: 'user', content: 'List files in the current directory' }
  ],
  tools: [
    {
      name: 'bash',
      description: 'Execute shell commands',
      parameters: {
        type: 'object',
        properties: {
          command: { type: 'string' }
        }
      }
    }
  ]
});

for await (const chunk of stream) {
  if (chunk.type === 'text') {
    console.log('Text:', chunk.content);
  } else if (chunk.type === 'tool_call') {
    console.log('Tool call:', chunk.tool, chunk.params);
  }
}
```

### Provider Configuration

```typescript
// Anthropic
const anthropic = Provider.create({
  name: 'anthropic',
  model: 'claude-3-opus-20240229',
  apiKey: process.env.ANTHROPIC_API_KEY,
  baseUrl: 'https://api.anthropic.com',  // optional
  maxTokens: 4096,
  temperature: 0.7
});

// OpenAI
const openai = Provider.create({
  name: 'openai',
  model: 'gpt-4-turbo',
  apiKey: process.env.OPENAI_API_KEY,
  maxTokens: 4096,
  temperature: 0.7
});

// Local (Ollama)
const ollama = Provider.create({
  name: 'ollama',
  model: 'llama2',
  baseUrl: 'http://localhost:11434'
});
```

## Provider Interface

```typescript
interface Provider {
  // Stream text generation
  stream(options: StreamOptions): AsyncGenerator<TextChunk>;

  // Generate completion (non-streaming)
  generate(options: GenerateOptions): Promise<GenerateResult>;

  // Get provider info
  getInfo(): ProviderInfo;

  // Check availability
  isAvailable(): Promise<boolean>;
}

interface StreamOptions {
  messages: Message[];
  tools?: Tool[];
  temperature?: number;
  maxTokens?: number;
  stopSequences?: string[];
}

interface TextChunk {
  type: 'text' | 'tool_call' | 'tool_result';
  content?: string;
  tool?: string;
  params?: Record<string, any>;
  delta?: string;
}
```

## Provider Features Matrix

| Feature | Anthropic | OpenAI | Google | Bedrock | Local |
|---------|-----------|--------|--------|---------|-------|
| Streaming | ✓ | ✓ | ✓ | ✓ | ✓ |
| Tools | ✓ | ✓ | ✓ | ✓ | Limited |
| Vision | ✓ | ✓ | ✓ | ✓ | Limited |
| JSON Mode | ✓ | ✓ | ✓ | ✓ | Limited |
| System Prompts | ✓ | ✓ | ✓ | ✓ | ✓ |

## Token Usage Tracking

```typescript
const stream = provider.stream({ messages: [...] });

let totalTokens = 0;
for await (const chunk of stream) {
  if (chunk.usage) {
    totalTokens += chunk.usage.totalTokens;
  }
}

console.log('Total tokens used:', totalTokens);
```

## Error Handling

```typescript
try {
  const stream = provider.stream({ messages: [...] });
  for await (const chunk of stream) {
    process.stdout.write(chunk.content);
  }
} catch (error) {
  if (error.code === 'RATE_LIMIT') {
    console.error('Rate limit exceeded, retrying...');
    await sleep(error.retryAfter);
  } else if (error.code === 'INVALID_API_KEY') {
    console.error('Invalid API key');
  } else if (error.code === 'CONTEXT_LENGTH_EXCEEDED') {
    console.error('Context too long');
  }
}
```

## Rate Limiting

Providers include automatic rate limiting:

```typescript
const provider = Provider.create({
  name: 'anthropic',
  model: 'claude-3-sonnet',
  rateLimit: {
    maxRequests: 50,      // Max requests per minute
    maxTokens: 100000     // Max tokens per minute
  }
});
```

## Retry Logic

Automatic retries for transient errors:

```typescript
const provider = Provider.create({
  name: 'anthropic',
  model: 'claude-3-sonnet',
  retry: {
    maxRetries: 3,
    initialDelay: 1000,
    maxDelay: 10000,
    backoff: 'exponential'
  }
});
```

## Provider Selection

```typescript
// Auto-select based on configuration
const provider = await Provider.fromConfig();

// Select by name
const provider = await Provider.select('anthropic');

// Select best available
const provider = await Provider.selectBest({
  requiredFeatures: ['streaming', 'tools'],
  preferredModels: ['claude-3-opus', 'gpt-4']
});
```

## Model Information

```typescript
const provider = Provider.create({ name: 'anthropic', model: 'claude-3-sonnet' });
const info = provider.getInfo();

console.log('Provider:', info.name);
console.log('Model:', info.model);
console.log('Max tokens:', info.maxTokens);
console.log('Supports tools:', info.supportsTools);
console.log('Supports vision:', info.supportsVision);
```

## Provider-Specific Features

### Anthropic Features

```typescript
// Extended thinking (Claude 3.5)
const provider = Provider.create({
  name: 'anthropic',
  model: 'claude-3-5-sonnet',
  thinkingBudget: 10000  // Thinking tokens
});

// Prompt caching
const provider = Provider.create({
  name: 'anthropic',
  model: 'claude-3-sonnet',
  cacheSystemPrompt: true
});
```

### OpenAI Features

```typescript
// JSON mode
const provider = Provider.create({
  name: 'openai',
  model: 'gpt-4-turbo',
  responseFormat: { type: 'json_object' }
});

// Seed for reproducibility
const provider = Provider.create({
  name: 'openai',
  model: 'gpt-4',
  seed: 12345
});
```

## Custom Providers

```typescript
import { Provider } from './provider';

class CustomProvider extends Provider {
  async stream(options: StreamOptions) {
    // Custom implementation
    const response = await fetch(this.baseUrl, {
      method: 'POST',
      body: JSON.stringify(options)
    });

    for await (const chunk of response.body) {
      yield { type: 'text', content: chunk };
    }
  }
}

Provider.register('custom', CustomProvider);
```

## Performance Considerations

- Connection pooling for HTTP requests
- Streaming reduces latency
- Prompt caching saves tokens
- Parallel tool calls (when supported)
- Efficient token counting

## Cost Optimization

```typescript
// Use cheaper model for simple tasks
const simple = Provider.create({
  name: 'anthropic',
  model: 'claude-3-haiku'  // Cheapest
});

// Use expensive model for complex tasks
const complex = Provider.create({
  name: 'anthropic',
  model: 'claude-3-opus'  // Most capable
});

// Cache system prompts (Anthropic)
const provider = Provider.create({
  name: 'anthropic',
  model: 'claude-3-sonnet',
  cacheSystemPrompt: true  // Save 90% on repeated prompts
});
```

## Related Documentation

- [Provider.stream API](../api-reference.md#providerstream)
- [Session Management](./session.md)
- [Interactive Chat Flow](../flows/run-command-flow.md)
