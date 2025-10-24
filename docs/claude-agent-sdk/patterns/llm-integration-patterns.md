# Claude Agent SDK - LLM Integration Patterns & Internal Logic

**SDK Version**: 0.1.22
**Source**: SDK source code analysis, documented patterns

---

## Table of Contents

1. [Overview](#overview)
2. [Model Selection & Management](#model-selection--management)
3. [Message Streaming Architecture](#message-streaming-architecture)
4. [Token Management & Cost Tracking](#token-management--cost-tracking)
5. [Extended Thinking (Ultrathink)](#extended-thinking-ultrathink)
6. [API Request Patterns](#api-request-patterns)
7. [Error Handling & Retry Logic](#error-handling--retry-logic)
8. [Cache Optimization Patterns](#cache-optimization-patterns)
9. [Real-World Integration Patterns](#real-world-integration-patterns)
10. [Best Practices](#best-practices)

---

## Overview

The Claude Agent SDK implements sophisticated patterns for LLM integration, focusing on:
- **Streaming Architecture**: Async generator-based streaming
- **Token Efficiency**: Prompt caching and optimization
- **Model Management**: Dynamic model selection and fallbacks
- **Cost Tracking**: Per-model usage and cost calculation
- **Thinking Budget**: Extended thinking token management

---

## Model Selection & Management

### Model Hierarchy

```typescript
// From SDK analysis
const MODEL_HIERARCHY = {
  // Claude 4 (Latest)
  opus: "claude-opus-4-20250514",
  
  // Claude 3.5 (Current)
  sonnet: "claude-3-5-sonnet-20241022",
  haiku: "claude-3-5-haiku-20241022",
  
  // Legacy (for fallback)
  "claude-3-opus-20240229": "deprecated",
  "claude-3-sonnet-20240229": "deprecated"
};
```

### Model Selection Algorithm

```typescript
/**
 * Model selection follows this priority:
 * 1. Explicit model parameter
 * 2. Agent-specific model
 * 3. Session model
 * 4. Default model (sonnet)
 */
function resolveModel(context: {
  explicitModel?: string;
  agentModel?: string;
  sessionModel?: string;
}): string {
  // 1. Explicit parameter (highest priority)
  if (context.explicitModel) {
    return resolveModelAlias(context.explicitModel);
  }
  
  // 2. Agent model (for subagents)
  if (context.agentModel && context.agentModel !== 'inherit') {
    return resolveModelAlias(context.agentModel);
  }
  
  // 3. Session model
  if (context.sessionModel) {
    return resolveModelAlias(context.sessionModel);
  }
  
  // 4. Default
  return "claude-3-5-sonnet-20241022";
}

/**
 * Model alias resolution
 */
function resolveModelAlias(alias: string): string {
  const ALIASES = {
    'opus': 'claude-opus-4-20250514',
    'sonnet': 'claude-3-5-sonnet-20241022',
    'haiku': 'claude-3-5-haiku-20241022'
  };
  
  return ALIASES[alias] || alias; // Return full ID if not an alias
}
```

### Model Fallback Pattern

```typescript
/**
 * Fallback model configuration
 * Used when primary model fails or is unavailable
 */
interface FallbackConfig {
  primaryModel: string;
  fallbackModel?: string;
  retryAttempts: number;
}

async function executeWithFallback(
  config: FallbackConfig,
  execute: (model: string) => Promise<Response>
): Promise<Response> {
  try {
    return await execute(config.primaryModel);
  } catch (error) {
    if (config.fallbackModel && isModelError(error)) {
      console.warn(`Primary model ${config.primaryModel} failed, falling back to ${config.fallbackModel}`);
      return await execute(config.fallbackModel);
    }
    throw error;
  }
}
```

### Model Capabilities Matrix

| Model | Speed | Cost | Context | Best For |
|-------|-------|------|---------|----------|
| **Haiku** | ⚡⚡⚡ Very Fast | 💰 Cheap | 200K | Simple tasks, discovery |
| **Sonnet** | ⚡⚡ Fast | 💰💰 Medium | 200K | General purpose, balanced |
| **Opus** | ⚡ Slower | 💰💰💰 Expensive | 200K | Complex reasoning, accuracy |

### Dynamic Model Switching

```typescript
/**
 * Switch model mid-conversation
 * Pattern used for adaptive model selection
 */
const query = await query({ 
  prompt: "Initial task",
  options: { model: 'sonnet' }
});

// Process some messages...

// Switch to Opus for complex reasoning
await query.setModel('opus');

// Continue conversation with Opus...

// Switch back to Sonnet for simple tasks
await query.setModel('sonnet');
```

**Use Cases**:
- Start with Haiku for discovery → Switch to Sonnet for implementation
- Use Sonnet for most work → Switch to Opus for critical decisions
- Downgrade to Haiku for simple confirmations

---

## Message Streaming Architecture

### Async Generator Pattern

```typescript
/**
 * Core streaming architecture
 * Query returns an AsyncGenerator that yields SDK messages
 */
interface Query extends AsyncGenerator<SDKMessage, void> {
  // Streaming interface
  [Symbol.asyncIterator](): AsyncIterator<SDKMessage>;
  next(): Promise<IteratorResult<SDKMessage, void>>;
  
  // Control methods
  interrupt(): Promise<void>;
  setPermissionMode(mode: PermissionMode): Promise<void>;
  setModel(model?: string): Promise<void>;
  setMaxThinkingTokens(tokens: number | null): Promise<void>;
  
  // Metadata queries
  supportedCommands(): Promise<SlashCommand[]>;
  supportedModels(): Promise<ModelInfo[]>;
  mcpServerStatus(): Promise<McpServerStatus[]>;
  accountInfo(): Promise<AccountInfo>;
}
```

### Message Flow Pipeline

```
1. User Input → SDKUserMessage
     ↓
2. Hook: UserPromptSubmit
     ↓
3. API Request (streaming)
     ↓
4. Stream Events → SDKPartialAssistantMessage
     ↓
5. Tool Use Detection
     ↓
6. Hook: PreToolUse
     ↓
7. Tool Execution
     ↓
8. Hook: PostToolUse
     ↓
9. Continue Streaming
     ↓
10. SDKResultMessage (final metrics)
```

### Streaming Implementation Pattern

```typescript
/**
 * Consume streaming messages
 */
async function* streamConversation(
  prompt: string,
  options?: Options
): AsyncGenerator<SDKMessage> {
  const query = await query({ prompt, options });
  
  try {
    for await (const message of query) {
      // Process each message type
      switch (message.type) {
        case 'assistant':
          // Complete assistant message
          handleAssistantMessage(message);
          break;
          
        case 'stream_event':
          // Partial streaming update
          handleStreamEvent(message);
          break;
          
        case 'result':
          // Final result with metrics
          handleResult(message);
          break;
          
        case 'system':
          // System messages (init, compact, hooks)
          handleSystemMessage(message);
          break;
          
        case 'user':
          // User message replay
          handleUserMessage(message);
          break;
      }
      
      yield message;
    }
  } catch (error) {
    handleStreamError(error);
  }
}
```

### Partial Message Handling

```typescript
/**
 * Handle streaming chunks (SDKPartialAssistantMessage)
 */
function handleStreamEvent(event: SDKPartialAssistantMessage) {
  const { event: rawEvent } = event;
  
  switch (rawEvent.type) {
    case 'message_start':
      // Message initialization
      console.log('Starting message...');
      break;
      
    case 'content_block_start':
      // New content block (text or tool use)
      if (rawEvent.content_block.type === 'text') {
        console.log('Text block started');
      } else if (rawEvent.content_block.type === 'tool_use') {
        console.log('Tool use started:', rawEvent.content_block.name);
      }
      break;
      
    case 'content_block_delta':
      // Streaming content chunk
      if (rawEvent.delta.type === 'text_delta') {
        process.stdout.write(rawEvent.delta.text); // Stream text
      } else if (rawEvent.delta.type === 'input_json_delta') {
        // Tool input streaming
        process.stdout.write(rawEvent.delta.partial_json);
      }
      break;
      
    case 'content_block_stop':
      // Content block complete
      console.log('\nBlock complete');
      break;
      
    case 'message_delta':
      // Message metadata update (stop_reason, usage)
      if (rawEvent.delta.stop_reason) {
        console.log('Stop reason:', rawEvent.delta.stop_reason);
      }
      if (rawEvent.usage) {
        console.log('Token usage:', rawEvent.usage);
      }
      break;
      
    case 'message_stop':
      // Message complete
      console.log('Message complete');
      break;
  }
}
```

### Progressive UI Updates

```typescript
/**
 * Real-time UI updates from streaming
 * Pattern for React/Ink-based UIs
 */
function useStreamingMessage() {
  const [text, setText] = useState('');
  const [thinking, setThinking] = useState('');
  const [toolUse, setToolUse] = useState<ToolUse | null>(null);
  
  async function processStream(query: Query) {
    for await (const message of query) {
      if (message.type === 'stream_event') {
        const event = message.event;
        
        if (event.type === 'content_block_delta') {
          if (event.delta.type === 'text_delta') {
            // Append streaming text
            setText(prev => prev + event.delta.text);
          } else if (event.delta.type === 'thinking_delta') {
            // Append thinking content
            setThinking(prev => prev + event.delta.thinking);
          }
        } else if (event.type === 'content_block_start') {
          if (event.content_block.type === 'tool_use') {
            setToolUse(event.content_block);
          }
        }
      }
    }
  }
  
  return { text, thinking, toolUse, processStream };
}
```

---

## Token Management & Cost Tracking

### Token Usage Structure

```typescript
/**
 * Complete token usage tracking
 * Extracted from SDK types
 */
interface NonNullableUsage {
  input_tokens: number;
  output_tokens: number;
  cache_creation_input_tokens: number;
  cache_read_input_tokens: number;
}

interface ModelUsage {
  inputTokens: number;
  outputTokens: number;
  cacheReadInputTokens: number;
  cacheCreationInputTokens: number;
  webSearchRequests: number;
  costUSD: number;
  contextWindow: number;
}

interface SDKResultMessage {
  type: 'result';
  subtype: 'success';
  usage: NonNullableUsage;              // Overall usage
  modelUsage: {                         // Per-model breakdown
    [modelName: string]: ModelUsage;
  };
  total_cost_usd: number;
  num_turns: number;
  duration_ms: number;
  duration_api_ms: number;
}
```

### Cost Calculation Pattern

```typescript
/**
 * Calculate cost from token usage
 * Pricing as of SDK v0.1.22
 */
const MODEL_PRICING = {
  'claude-opus-4-20250514': {
    input: 15.00 / 1_000_000,      // $15 per MTok
    output: 75.00 / 1_000_000,     // $75 per MTok
    cacheWrite: 18.75 / 1_000_000, // $18.75 per MTok (cache creation)
    cacheRead: 1.50 / 1_000_000    // $1.50 per MTok (cache hit)
  },
  'claude-3-5-sonnet-20241022': {
    input: 3.00 / 1_000_000,
    output: 15.00 / 1_000_000,
    cacheWrite: 3.75 / 1_000_000,
    cacheRead: 0.30 / 1_000_000
  },
  'claude-3-5-haiku-20241022': {
    input: 1.00 / 1_000_000,
    output: 5.00 / 1_000_000,
    cacheWrite: 1.25 / 1_000_000,
    cacheRead: 0.10 / 1_000_000
  }
};

function calculateCost(
  model: string,
  usage: NonNullableUsage
): number {
  const pricing = MODEL_PRICING[model];
  if (!pricing) return 0;
  
  const inputCost = usage.input_tokens * pricing.input;
  const outputCost = usage.output_tokens * pricing.output;
  const cacheWriteCost = usage.cache_creation_input_tokens * pricing.cacheWrite;
  const cacheReadCost = usage.cache_read_input_tokens * pricing.cacheRead;
  
  return inputCost + outputCost + cacheWriteCost + cacheReadCost;
}
```

### Cache Efficiency Tracking

```typescript
/**
 * Analyze cache performance
 */
function analyzeCacheEfficiency(usage: NonNullableUsage) {
  const totalInput = usage.input_tokens + usage.cache_read_input_tokens;
  const cacheHitRate = usage.cache_read_input_tokens / totalInput;
  
  const normalCost = totalInput * MODEL_PRICING['claude-3-5-sonnet-20241022'].input;
  const actualCost = 
    usage.input_tokens * MODEL_PRICING['claude-3-5-sonnet-20241022'].input +
    usage.cache_read_input_tokens * MODEL_PRICING['claude-3-5-sonnet-20241022'].cacheRead;
  
  const savings = normalCost - actualCost;
  const savingsPercent = (savings / normalCost) * 100;
  
  return {
    cacheHitRate: `${(cacheHitRate * 100).toFixed(1)}%`,
    savings: `$${savings.toFixed(4)}`,
    savingsPercent: `${savingsPercent.toFixed(1)}%`
  };
}

// Example output:
// {
//   cacheHitRate: "85.3%",
//   savings: "$0.0245",
//   savingsPercent: "90.0%"
// }
```

### Token Budget Management

```typescript
/**
 * Monitor and enforce token budgets
 */
class TokenBudgetManager {
  private budget: number;
  private spent: number = 0;
  
  constructor(budget: number) {
    this.budget = budget;
  }
  
  canAfford(estimatedTokens: number): boolean {
    return (this.spent + estimatedTokens) <= this.budget;
  }
  
  recordUsage(usage: NonNullableUsage) {
    const total = 
      usage.input_tokens + 
      usage.output_tokens + 
      usage.cache_creation_input_tokens;
    
    this.spent += total;
    
    if (this.spent > this.budget) {
      throw new Error(`Budget exceeded: ${this.spent}/${this.budget} tokens`);
    }
  }
  
  getRemaining(): number {
    return Math.max(0, this.budget - this.spent);
  }
  
  getUtilization(): number {
    return (this.spent / this.budget) * 100;
  }
}

// Usage:
const budget = new TokenBudgetManager(100_000); // 100K token budget

for await (const message of query) {
  if (message.type === 'result') {
    budget.recordUsage(message.usage);
    console.log(`Budget remaining: ${budget.getRemaining()} tokens`);
    console.log(`Utilization: ${budget.getUtilization().toFixed(1)}%`);
  }
}
```

---

## Extended Thinking (Ultrathink)

### Thinking Token Configuration

```typescript
/**
 * Extended thinking constants
 * Extracted from CLI internal constants
 */
const THINKING_CONFIG = {
  // Maximum thinking tokens (from source)
  MAX_THINKING_TOKENS: 31999,
  
  // Default (no extended thinking)
  DEFAULT_THINKING_TOKENS: 0,
  
  // Trigger keywords
  TRIGGERS: [
    'ultrathink',
    'think ultra hard',
    'think ultrahard'
  ]
};
```

### Ultrathink Detection Pattern

```typescript
/**
 * Detect if prompt requests extended thinking
 */
function detectUltrathink(prompt: string): boolean {
  const normalized = prompt.toLowerCase().trim();
  
  return THINKING_CONFIG.TRIGGERS.some(trigger => 
    normalized === trigger || normalized.startsWith(trigger + ':')
  );
}

/**
 * Apply thinking budget based on prompt
 */
function resolveThinkingBudget(prompt: string, explicit?: number | null): number {
  // Explicit parameter takes precedence
  if (explicit !== undefined && explicit !== null) {
    return Math.min(explicit, THINKING_CONFIG.MAX_THINKING_TOKENS);
  }
  
  // Check for ultrathink keyword
  if (detectUltrathink(prompt)) {
    return THINKING_CONFIG.MAX_THINKING_TOKENS;
  }
  
  // Default: no extended thinking
  return THINKING_CONFIG.DEFAULT_THINKING_TOKENS;
}
```

### Extended Thinking Usage

```typescript
/**
 * Use extended thinking for complex problems
 */
async function solveComplexProblem(problem: string) {
  // Explicit ultrathink request
  const result1 = await query({
    prompt: `ultrathink: ${problem}`,
    options: {} // Will detect ultrathink keyword
  });
  
  // Or explicit parameter
  const result2 = await query({
    prompt: problem,
    options: {
      maxThinkingTokens: 31999 // Maximum budget
    }
  });
  
  // Or dynamic mid-conversation
  const queryInstance = await query({
    prompt: "Start analysis",
    options: { maxThinkingTokens: 0 }
  });
  
  // ... some conversation ...
  
  // Enable extended thinking for complex step
  await queryInstance.setMaxThinkingTokens(31999);
  
  // ... complex reasoning ...
  
  // Disable for simple steps
  await queryInstance.setMaxThinkingTokens(0);
}
```

### Model-Specific Thinking Limits

```typescript
/**
 * Different models have different thinking capabilities
 * Extracted from implementation
 */
const MODEL_THINKING_LIMITS = {
  'claude-opus-4-20250514': 8192,
  'claude-opus-4-1-20250805': 8192,
  'claude-3-5-sonnet-20241022': 0, // No extended thinking
  'claude-3-5-haiku-20241022': 0   // No extended thinking
};

function getMaxThinkingTokens(model: string): number {
  return MODEL_THINKING_LIMITS[model] || 0;
}
```

---

## API Request Patterns

### Request Construction

```typescript
/**
 * Build API request for Claude
 */
interface AnthropicRequest {
  model: string;
  messages: Message[];
  max_tokens: number;
  system?: string;
  temperature?: number;
  thinking?: {
    type: 'enabled';
    budget_tokens: number;
  };
  tools?: Tool[];
  stream: boolean;
}

function buildRequest(context: {
  model: string;
  messages: Message[];
  systemPrompt?: string;
  maxThinkingTokens?: number;
  tools?: Tool[];
}): AnthropicRequest {
  const request: AnthropicRequest = {
    model: context.model,
    messages: context.messages,
    max_tokens: 8192, // Standard max
    stream: true      // Always stream
  };
  
  // Add system prompt
  if (context.systemPrompt) {
    request.system = context.systemPrompt;
  }
  
  // Add extended thinking
  if (context.maxThinkingTokens && context.maxThinkingTokens > 0) {
    request.thinking = {
      type: 'enabled',
      budget_tokens: context.maxThinkingTokens
    };
  }
  
  // Add tools
  if (context.tools && context.tools.length > 0) {
    request.tools = context.tools;
  }
  
  return request;
}
```

### Retry Logic Pattern

```typescript
/**
 * Retry with exponential backoff
 */
async function apiRequestWithRetry<T>(
  makeRequest: () => Promise<T>,
  options: {
    maxRetries?: number;
    baseDelay?: number;
    maxDelay?: number;
  } = {}
): Promise<T> {
  const maxRetries = options.maxRetries ?? 3;
  const baseDelay = options.baseDelay ?? 1000;
  const maxDelay = options.maxDelay ?? 30000;
  
  let lastError: Error;
  
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await makeRequest();
    } catch (error) {
      lastError = error as Error;
      
      // Don't retry on certain errors
      if (isNonRetryableError(error)) {
        throw error;
      }
      
      if (attempt < maxRetries) {
        const delay = Math.min(
          baseDelay * Math.pow(2, attempt),
          maxDelay
        );
        
        console.warn(`Request failed (attempt ${attempt + 1}/${maxRetries}), retrying in ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
  
  throw new Error(`Request failed after ${maxRetries} retries: ${lastError.message}`);
}

function isNonRetryableError(error: any): boolean {
  // Don't retry on auth errors, invalid requests, etc.
  return error.status === 401 || 
         error.status === 403 || 
         error.status === 400;
}
```

---

## Error Handling & Retry Logic

### Error Classification

```typescript
/**
 * Classify errors for appropriate handling
 */
enum ErrorType {
  // Network errors (retryable)
  NETWORK = 'network',
  TIMEOUT = 'timeout',
  
  // API errors
  RATE_LIMIT = 'rate_limit',      // 429 (retryable with backoff)
  SERVER_ERROR = 'server_error',  // 500-599 (retryable)
  
  // Client errors (non-retryable)
  AUTH_ERROR = 'auth_error',       // 401, 403
  INVALID_REQUEST = 'invalid_request', // 400
  NOT_FOUND = 'not_found',         // 404
  
  // Application errors
  PERMISSION_DENIED = 'permission_denied',
  TOOL_ERROR = 'tool_error',
  VALIDATION_ERROR = 'validation_error'
}

function classifyError(error: any): ErrorType {
  if (error.code === 'ECONNREFUSED' || error.code === 'ETIMEDOUT') {
    return ErrorType.NETWORK;
  }
  
  const status = error.status || error.response?.status;
  
  if (status === 429) return ErrorType.RATE_LIMIT;
  if (status === 401 || status === 403) return ErrorType.AUTH_ERROR;
  if (status === 400) return ErrorType.INVALID_REQUEST;
  if (status === 404) return ErrorType.NOT_FOUND;
  if (status >= 500) return ErrorType.SERVER_ERROR;
  
  if (error.message?.includes('permission')) return ErrorType.PERMISSION_DENIED;
  if (error.message?.includes('tool')) return ErrorType.TOOL_ERROR;
  
  return ErrorType.VALIDATION_ERROR;
}
```

### Comprehensive Error Handler

```typescript
/**
 * Handle all error types appropriately
 */
async function handleAPIError(error: any, context: {
  attempt: number;
  maxRetries: number;
}): Promise<'retry' | 'fail' | 'abort'> {
  const errorType = classifyError(error);
  
  switch (errorType) {
    case ErrorType.RATE_LIMIT:
      // Extract retry-after header
      const retryAfter = error.response?.headers['retry-after'];
      const delay = retryAfter ? parseInt(retryAfter) * 1000 : 60000;
      
      console.warn(`Rate limited. Retrying after ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
      return 'retry';
      
    case ErrorType.NETWORK:
    case ErrorType.TIMEOUT:
    case ErrorType.SERVER_ERROR:
      // Retryable errors
      if (context.attempt < context.maxRetries) {
        console.warn(`${errorType} error. Retrying...`);
        return 'retry';
      }
      return 'fail';
      
    case ErrorType.AUTH_ERROR:
      console.error('Authentication failed. Check your API key.');
      return 'abort';
      
    case ErrorType.INVALID_REQUEST:
      console.error('Invalid request:', error.message);
      return 'abort';
      
    case ErrorType.PERMISSION_DENIED:
      console.error('Permission denied:', error.message);
      return 'abort';
      
    default:
      console.error('Unexpected error:', error);
      return 'fail';
  }
}
```

---

## Cache Optimization Patterns

### Prompt Caching Strategy

```typescript
/**
 * Optimize prompt caching for repeated content
 * Cache-eligible content must be:
 * - At least 1024 tokens
 * - In system prompt or early user messages
 * - Repeated across multiple requests
 */
interface CacheConfig {
  // Mark content for caching
  system: [
    { type: 'text', text: string, cache_control?: { type: 'ephemeral' } }
  ];
}

function buildCachedRequest(
  staticContext: string,  // Large, repeated content
  dynamicPrompt: string
): AnthropicRequest {
  return {
    model: 'claude-3-5-sonnet-20241022',
    max_tokens: 8192,
    system: [
      {
        type: 'text',
        text: staticContext,
        cache_control: { type: 'ephemeral' } // Cache this!
      }
    ],
    messages: [
      {
        role: 'user',
        content: dynamicPrompt
      }
    ],
    stream: true
  };
}
```

**Caching Benefits**:
- **Cost Reduction**: 90% cheaper for cached tokens
- **Latency Reduction**: Faster response (cached content pre-processed)
- **Ideal For**: Large codebases, documentation, repeated context

### Cache Hit Rate Optimization

```typescript
/**
 * Maximize cache hits through consistent context
 */
class CacheOptimizer {
  private contextCache: Map<string, string> = new Map();
  
  /**
   * Generate stable context hash
   */
  private hashContext(context: string): string {
    // Use consistent hashing
    return crypto.createHash('sha256').update(context).digest('hex');
  }
  
  /**
   * Get cached context or create new
   */
  getCachedContext(key: string, generator: () => string): string {
    const hash = this.hashContext(key);
    
    if (!this.contextCache.has(hash)) {
      this.contextCache.set(hash, generator());
    }
    
    return this.contextCache.get(hash)!;
  }
  
  /**
   * Build request with optimized caching
   */
  buildOptimizedRequest(
    codebaseContext: string,
    userPrompt: string
  ): AnthropicRequest {
    // Ensure context is stable for caching
    const stableContext = this.getCachedContext(
      'codebase',
      () => codebaseContext
    );
    
    return buildCachedRequest(stableContext, userPrompt);
  }
}
```

---

## Real-World Integration Patterns

### Pattern 1: Long-Running Task with Progress

```typescript
/**
 * Execute long task with progress updates
 */
async function executeLongTask(task: string) {
  const query = await query({
    prompt: task,
    options: {
      model: 'sonnet',
      maxThinkingTokens: 0  // Start without extended thinking
    }
  });
  
  let currentStep = '';
  let thinkingEnabled = false;
  
  for await (const message of query) {
    if (message.type === 'stream_event') {
      const event = message.event;
      
      // Detect complex reasoning step
      if (event.type === 'content_block_start' && 
          event.content_block.type === 'text') {
        const text = event.content_block.text || '';
        
        if (text.includes('complex') || text.includes('analyze deeply')) {
          // Enable extended thinking for complex step
          if (!thinkingEnabled) {
            await query.setMaxThinkingTokens(31999);
            thinkingEnabled = true;
            console.log('✓ Enabled extended thinking');
          }
        }
      }
      
      // Show progress
      if (event.type === 'content_block_delta' && 
          event.delta.type === 'text_delta') {
        currentStep += event.delta.text;
        
        // Update UI
        process.stdout.write('.');
      }
    } else if (message.type === 'result') {
      // Task complete
      console.log('\n✓ Task complete');
      console.log(`Tokens used: ${message.usage.input_tokens + message.usage.output_tokens}`);
      console.log(`Cost: $${message.total_cost_usd.toFixed(4)}`);
    }
  }
}
```

### Pattern 2: Multi-Model Workflow

```typescript
/**
 * Use different models for different task phases
 */
async function multiModelWorkflow(task: string) {
  // Phase 1: Discovery with Haiku (fast + cheap)
  console.log('Phase 1: Discovery (Haiku)...');
  const discoveryQuery = await query({
    prompt: `Explore the codebase: ${task}`,
    options: {
      model: 'haiku',
      agents: {
        'Explore': {
          description: 'Fast codebase discovery',
          tools: ['Glob', 'Grep', 'Read'],
          prompt: 'Find relevant files quickly',
          model: 'haiku'
        }
      }
    }
  });
  
  let discoveredFiles: string[] = [];
  for await (const msg of discoveryQuery) {
    if (msg.type === 'result') {
      discoveredFiles = extractFiles(msg);
    }
  }
  
  // Phase 2: Implementation with Sonnet (balanced)
  console.log('Phase 2: Implementation (Sonnet)...');
  const implQuery = await query({
    prompt: `Implement changes in: ${discoveredFiles.join(', ')}`,
    options: { model: 'sonnet' }
  });
  
  for await (const msg of implQuery) {
    // Process implementation...
  }
  
  // Phase 3: Review with Opus (highest quality)
  console.log('Phase 3: Review (Opus)...');
  const reviewQuery = await query({
    prompt: 'Review all changes for correctness and security',
    options: { model: 'opus' }
  });
  
  for await (const msg of reviewQuery) {
    // Process review...
  }
}
```

### Pattern 3: Cost-Optimized Agent

```typescript
/**
 * Optimize costs through intelligent model and cache usage
 */
class CostOptimizedAgent {
  private budget: TokenBudgetManager;
  private cacheOptimizer: CacheOptimizer;
  
  constructor(budgetTokens: number) {
    this.budget = new TokenBudgetManager(budgetTokens);
    this.cacheOptimizer = new CacheOptimizer();
  }
  
  async execute(task: string, context: string) {
    // Build request with caching
    const request = this.cacheOptimizer.buildOptimizedRequest(context, task);
    
    // Use cheapest model that can handle task
    const model = this.selectOptimalModel(task);
    
    const query = await query({
      prompt: task,
      options: {
        model,
        // Reuse cached context
        customSystemPrompt: request.system[0].text
      }
    });
    
    for await (const message of query) {
      if (message.type === 'result') {
        // Track usage
        this.budget.recordUsage(message.usage);
        
        // Log savings from cache
        const cacheAnalysis = analyzeCacheEfficiency(message.usage);
        console.log(`Cache hit rate: ${cacheAnalysis.cacheHitRate}`);
        console.log(`Savings: ${cacheAnalysis.savings} (${cacheAnalysis.savingsPercent})`);
      }
    }
  }
  
  private selectOptimalModel(task: string): string {
    // Simple task → Haiku
    if (task.length < 200 && !task.includes('complex')) {
      return 'haiku';
    }
    
    // Complex task → Opus
    if (task.includes('analyze deeply') || task.includes('complex reasoning')) {
      return 'opus';
    }
    
    // Default → Sonnet
    return 'sonnet';
  }
}
```

---

## Best Practices

### 1. Model Selection

✅ **Do**:
- Use Haiku for discovery and simple tasks
- Use Sonnet for general-purpose work
- Use Opus only when accuracy is critical
- Switch models dynamically based on task complexity

❌ **Don't**:
- Use Opus for all tasks (expensive!)
- Use Haiku for complex reasoning
- Forget to consider cost vs. quality tradeoffs

### 2. Token Management

✅ **Do**:
- Monitor token usage and costs
- Use prompt caching for repeated content
- Track cache hit rates
- Set token budgets for long-running tasks

❌ **Don't**:
- Ignore cache optimization opportunities
- Send large repeated context without caching
- Exceed token budgets unexpectedly

### 3. Extended Thinking

✅ **Do**:
- Use ultrathink for genuinely complex problems
- Disable for simple tasks to save tokens
- Use model-appropriate thinking budgets

❌ **Don't**:
- Enable extended thinking by default
- Use thinking tokens for simple queries
- Forget thinking has token costs

### 4. Error Handling

✅ **Do**:
- Implement retry logic with exponential backoff
- Handle rate limits gracefully
- Classify errors appropriately
- Log errors for debugging

❌ **Don't**:
- Retry non-retryable errors
- Ignore rate limit headers
- Give up immediately on transient errors

### 5. Streaming

✅ **Do**:
- Always use streaming for better UX
- Handle partial messages incrementally
- Update UI progressively
- Support interruption

❌ **Don't**:
- Wait for complete response before showing anything
- Ignore stream events
- Block UI during streaming

---

## Summary

### Key Patterns

| Pattern | Use Case | Benefit |
|---------|----------|---------|
| **Async Generator** | Message streaming | Real-time UI updates |
| **Model Fallback** | API reliability | Resilience to failures |
| **Token Budgeting** | Cost control | Prevent overruns |
| **Prompt Caching** | Repeated context | 90% cost reduction |
| **Extended Thinking** | Complex reasoning | Better quality |
| **Multi-Model** | Workflow optimization | Cost + quality balance |

### Performance Optimization Checklist

- [ ] Use appropriate model for task complexity
- [ ] Enable prompt caching for repeated content
- [ ] Track and optimize cache hit rates
- [ ] Set token budgets for cost control
- [ ] Use extended thinking selectively
- [ ] Implement retry logic with backoff
- [ ] Handle errors gracefully
- [ ] Stream responses for better UX
- [ ] Monitor API usage and costs
- [ ] Use Haiku for discovery tasks

### Cost Optimization Checklist

- [ ] Start with Haiku, upgrade if needed
- [ ] Cache large static context
- [ ] Reuse cached prompts
- [ ] Disable thinking for simple tasks
- [ ] Track savings from cache hits
- [ ] Set and enforce token budgets
- [ ] Use Explore agent (70-84% savings)
- [ ] Switch models based on complexity
