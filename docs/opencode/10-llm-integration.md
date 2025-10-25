# OpenCode - LLM Integration

> **Vercel AI SDK integration for streaming responses and tool calls**

---

## Overview

OpenCode uses [Vercel AI SDK](https://sdk.vercel.ai) for LLM communication, providing:
- **Unified interface** across providers
- **Streaming responses** for real-time updates
- **Tool calling** with structured outputs
- **Token management** and tracking
- **Retry logic** with exponential backoff

---

## AI SDK Integration

### Streaming Text

```typescript
import { streamText } from "ai"

const stream = streamText({
  model: provider.language,
  system: systemPrompts.join("\n\n"),
  messages: conversationHistory,
  tools: availableTools,
  maxTokens: 32_000,
  temperature: 0.7,
  abortSignal: lock.signal,
})

for await (const chunk of stream.fullStream) {
  switch (chunk.type) {
    case "text-delta":
      // Handle text chunks
      break
    case "tool-call":
      // Handle tool execution
      break
    case "finish-step":
      // Handle completion
      break
  }
}
```

### Non-Streaming (Generate)

```typescript
import { generateText } from "ai"

const result = await generateText({
  model: provider.language,
  system: prompt,
  prompt: userInput,
  maxRetries: 3,
})

console.log(result.text)
```

---

## Stream Event Types

### text-delta

```typescript
{
  type: "text-delta",
  text: "Here is "
}
```

**Action**: Append to message text part.

### tool-call

```typescript
{
  type: "tool-call",
  toolCallId: "call_abc123",
  toolName: "read",
  args: { filePath: "src/auth.ts" }
}
```

**Action**: Execute tool and return result.

### tool-result

```typescript
{
  type: "tool-result",
  toolCallId: "call_abc123",
  result: "File contents..."
}
```

**Action**: Feed result back to model.

### step-start

```typescript
{
  type: "step-start",
  stepId: "step_1"
}
```

**Action**: New reasoning step (for models like o1).

### finish-step

```typescript
{
  type: "finish-step",
  usage: {
    promptTokens: 1234,
    completionTokens: 567,
    totalTokens: 1801
  },
  providerMetadata: {...}
}
```

**Action**: Calculate cost and update session.

### error

```typescript
{
  type: "error",
  error: new Error("Rate limited")
}
```

**Action**: Retry or fail.

---

## Token Management

### Tracking Usage

```typescript
const usage = Session.getUsage({
  model: modelInfo,
  usage: {
    promptTokens: 1234,
    completionTokens: 567,
  },
  metadata: providerMetadata
})

message.tokens = {
  input: usage.tokens.input,
  output: usage.tokens.output,
  total: usage.tokens.total,
}

message.cost = usage.cost
```

### Cost Calculation

```typescript
const costPerInput = modelInfo.costPer1kInput ?? 0
const costPerOutput = modelInfo.costPer1kOutput ?? 0

const cost = 
  (tokens.input / 1000) * costPerInput +
  (tokens.output / 1000) * costPerOutput
```

### Token Limits

```typescript
const OUTPUT_TOKEN_MAX = 32_000

// Check overflow before streaming
if (session.totalTokens > model.inputTokenLimit * 0.8) {
  await SessionCompaction.run({ sessionID })
}
```

---

## Retry Logic

### Exponential Backoff

```typescript
const MAX_RETRIES = 10

for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
  try {
    const result = await streamText({...})
    return result
  } catch (error) {
    if (!isRetryable(error) || attempt >= MAX_RETRIES - 1) {
      throw error
    }
    
    const delayMs = getRetryDelayInMs(error, attempt)
    await sleep(delayMs)
    
    // Create retry part for transparency
    await Session.updatePart({
      type: "retry",
      attempt: attempt + 1,
      error: MessageV2.fromError(error)
    })
  }
}
```

### Retryable Errors

- **Rate limits** (429)
- **Temporary failures** (5xx)
- **Network errors**
- **Timeout errors**

**Not Retryable**:
- **Auth errors** (401, 403)
- **Invalid requests** (400)
- **Content policy** violations

---

## Error Handling

### Graceful Degradation

```typescript
try {
  const result = await streamText({...})
} catch (error) {
  if (error.code === "RATE_LIMITED") {
    // Switch to slower model or wait
    await sleep(60_000)
    return retry()
  }
  
  if (error.code === "CONTEXT_LENGTH_EXCEEDED") {
    // Compact session
    await SessionCompaction.run({...})
    return retry()
  }
  
  // Log and re-throw
  log.error("Stream error", { error })
  throw error
}
```

---

## Model-Specific Behaviors

### Anthropic (Claude)

- Supports vision (images)
- Requires system prompt in specific format
- Supports extended thinking mode
- Best for complex reasoning

### OpenAI (GPT)

- GPT-4: Broad knowledge, good balance
- o1: Extended reasoning, no streaming
- GPT-3.5: Fast, cost-effective

### Google (Gemini)

- 1M+ token context
- Fast inference
- Good for long documents

### Local (Ollama)

- No API costs
- Privacy-preserving
- Limited capabilities

---

## Best Practices

**Streaming**:
- Always use streaming for better UX
- Handle all event types
- Update UI incrementally
- Show progress indicators

**Error Handling**:
- Implement retries with backoff
- Log all errors with context
- Provide clear error messages
- Gracefully degrade on failures

**Token Management**:
- Track usage per session
- Calculate costs accurately
- Implement compaction when needed
- Set reasonable limits

**Performance**:
- Use abort signals for cancellation
- Implement timeouts
- Cache when possible
- Monitor latency

---

For implementation, see `packages/opencode/src/session/prompt.ts`.

