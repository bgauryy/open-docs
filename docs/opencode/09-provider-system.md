# OpenCode - Provider System

> **Provider-agnostic architecture supporting multiple AI providers**

---

## Overview

OpenCode's provider system abstracts LLM providers, enabling:
- **Multi-provider support** - Anthropic, OpenAI, Google, Bedrock, local models
- **Unified interface** - Same API for all providers
- **Model registry** - Centralized model definitions
- **Dynamic discovery** - Auto-detect available models

**Files**:
- `provider/provider.ts` (620 lines, 20KB) - Provider interface
- `provider/models.ts` - Model registry
- `provider/transform.ts` - Response transformation

---

## Provider Interface

```typescript
export namespace Provider {
  export interface Info {
    id: string                    // Provider identifier
    name: string                  // Display name
    language: LanguageModel       // Vercel AI SDK model
    
    // List available models
    models(): Promise<ModelInfo[]>
    
    // Get specific model
    getModel(modelID: string): Promise<ModelInfo>
  }
  
  export interface ModelInfo {
    id: string
    name: string
    provider: string
    inputTokenLimit: number
    outputTokenLimit: number
    modalities?: {
      input?: ("text" | "image" | "audio")[]
      output?: ("text" | "image" | "audio")[]
    }
  }
}
```

---

## Supported Providers

### Anthropic (Claude)

```typescript
{
  id: "anthropic",
  name: "Anthropic",
  models: [
    "claude-3-5-sonnet-20241022",
    "claude-3-opus-20240229",
    "claude-3-haiku-20240307"
  ]
}
```

**Authentication**: `ANTHROPIC_API_KEY` env var

### OpenAI (GPT)

```typescript
{
  id: "openai",
  name: "OpenAI",
  models: [
    "gpt-4o",
    "gpt-4-turbo",
    "gpt-3.5-turbo",
    "o1-preview"
  ]
}
```

**Authentication**: `OPENAI_API_KEY` env var

### Google (Gemini)

```typescript
{
  id: "google",
  name: "Google",
  models: [
    "gemini-1.5-pro",
    "gemini-1.5-flash"
  ]
}
```

**Authentication**: `GOOGLE_API_KEY` env var

### Amazon Bedrock

```typescript
{
  id: "bedrock",
  name: "Amazon Bedrock",
  models: [
    "anthropic.claude-3-sonnet-20240229-v1:0",
    "anthropic.claude-3-haiku-20240307-v1:0"
  ]
}
```

**Authentication**: AWS credentials

### Local Models

```typescript
{
  id: "ollama",
  name: "Ollama",
  models: [
    "llama3",
    "mistral",
    "codellama"
  ]
}
```

**Setup**: Install Ollama + models

---

## Model Registry

**Central Registry** (`provider/models.ts`):

```typescript
export const Models = {
  "anthropic/claude-3-5-sonnet": {
    id: "claude-3-5-sonnet-20241022",
    name: "Claude 3.5 Sonnet",
    provider: "anthropic",
    inputTokenLimit: 200_000,
    outputTokenLimit: 8_192,
    costPer1kInput: 0.003,
    costPer1kOutput: 0.015,
    modalities: {
      input: ["text", "image"],
      output: ["text"]
    }
  },
  // ... more models
}
```

---

## Usage

### List All Models

```bash
opencode models
```

Output:
```
Available Models:

Anthropic:
  - claude-3-5-sonnet (200K context)
  - claude-3-opus (200K context)
  
OpenAI:
  - gpt-4o (128K context)
  - gpt-4-turbo (128K context)

Google:
  - gemini-1.5-pro (1M context)
```

### Use Specific Model

```bash
opencode --model anthropic/claude-3-opus "Complex task"
opencode --model openai/gpt-4o "Quick question"
```

### Default Model

**.opencode/config.json**:
```json
{
  "provider": "anthropic",
  "model": "claude-3-5-sonnet"
}
```

---

## Adding Custom Providers

```typescript
// .opencode/provider/custom.ts
import { Provider } from "opencode"

export const CustomProvider: Provider.Info = {
  id: "custom",
  name: "Custom Provider",
  
  async models() {
    return [{
      id: "custom-model",
      name: "Custom Model",
      provider: "custom",
      inputTokenLimit: 100_000,
      outputTokenLimit: 4_096,
    }]
  },
  
  language: createCustomLanguageModel({...})
}
```

---

## Best Practices

**Model Selection**:
- Use Claude for complex reasoning
- Use GPT-4 for broad knowledge
- Use Gemini for long context
- Use local models for privacy

**Cost Optimization**:
- Use cheaper models for simple tasks
- Cache responses when possible
- Monitor token usage

**Performance**:
- Set reasonable token limits
- Use streaming for responsiveness
- Implement timeouts

---

For implementation, see `packages/opencode/src/provider/`.

