# Technical Stack

## Overview

Gemini CLI is built as a TypeScript monorepo using modern JavaScript tooling and frameworks. This document details the core technologies, their purposes, and architectural decisions.

## Core Technologies

### Runtime Environment

| Technology | Version | Purpose |
|------------|---------|---------|
| Node.js | ≥20.0.0 | Runtime environment |
| TypeScript | ^5.3.3 | Type-safe JavaScript |
| ES Modules | Native | Module system (type: "module") |

### Package Management

| Technology | Purpose |
|------------|---------|
| npm | Package manager |
| npm workspaces | Monorepo management |

## Framework Choices

### Terminal UI - Ink (React for CLI)

The CLI uses **Ink** (a forked version `@jrichman/ink@6.4.7`) for building the terminal user interface:

```typescript
// Example UI component
import { Box, Text } from 'ink';

export const ChatMessage = ({ content, role }) => (
  <Box flexDirection="column">
    <Text color={role === 'user' ? 'blue' : 'green'}>
      {content}
    </Text>
  </Box>
);
```

**Why Ink?**
- Component-based UI development with React patterns
- Declarative rendering for complex terminal layouts
- Hot reloading during development
- Rich ecosystem of components (spinners, gradients, etc.)

### API Client - @google/genai

The official Google Generative AI client (`@google/genai@1.30.0`) provides:
- Type-safe API interactions
- Streaming support for responses
- Tool calling (function calling) support
- Multi-modal content handling

```typescript
import { GoogleGenerativeAI } from '@google/genai';

const genAI = new GoogleGenerativeAI(apiKey);
const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' });
```

### Schema Validation - Zod

**Zod** (`^3.25.76`) provides runtime type validation:

```typescript
import { z } from 'zod';

const ConfigSchema = z.object({
  model: z.string().default('gemini-2.5-flash'),
  temperature: z.number().min(0).max(2).optional(),
});

type Config = z.infer<typeof ConfigSchema>;
```

**Benefits:**
- TypeScript-first schema declaration
- Runtime validation with detailed errors
- Automatic type inference
- Works well with Gemini API tool definitions

## External Integrations

### Gemini API

Primary AI backend using Google's Gemini models:
- **Models**: gemini-2.5-flash, gemini-2.5-pro, gemini-3
- **Features**: Multi-turn chat, streaming, function calling, grounding
- **Authentication**: OAuth, API keys, Vertex AI

### Model Context Protocol (MCP)

MCP integration (`@modelcontextprotocol/sdk@^1.23.0`) enables:
- Custom tool servers
- Resource providers
- OAuth-based authentication for third-party services

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js';

const client = new Client({ name: 'gemini-cli' });
await client.connect(transport);
const tools = await client.listTools();
```

### OpenTelemetry

Comprehensive observability stack:

| Package | Purpose |
|---------|---------|
| `@opentelemetry/sdk-node` | Core SDK |
| `@opentelemetry/exporter-trace-otlp-http` | Trace export |
| `@opentelemetry/exporter-metrics-otlp-http` | Metrics export |
| `@google-cloud/opentelemetry-cloud-trace-exporter` | GCP integration |

### Authentication Libraries

| Library | Purpose |
|---------|---------|
| `google-auth-library` | Google OAuth and service accounts |
| `keytar` | Secure credential storage (optional) |

## Build System

### esbuild

The project uses **esbuild** for fast bundling:

```javascript
// esbuild.config.js
import esbuild from 'esbuild';

await esbuild.build({
  entryPoints: ['packages/cli/index.ts'],
  bundle: true,
  platform: 'node',
  target: 'node20',
  format: 'esm',
  outfile: 'bundle/gemini.js',
});
```

**Build Pipeline:**
1. TypeScript compilation via esbuild
2. Asset copying (WASM files, etc.)
3. Bundle creation for distribution

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true
  }
}
```

## Testing Framework

### Vitest

The project uses **Vitest** (`^3.2.4`) for testing:

```typescript
import { describe, it, expect, vi } from 'vitest';

describe('GeminiChat', () => {
  it('should send messages', async () => {
    const chat = new GeminiChat(mockConfig);
    const response = await chat.send('Hello');
    expect(response).toBeDefined();
  });
});
```

**Configuration:**
- Separate configs per workspace
- Co-located test files (`*.test.ts`)
- Coverage via `@vitest/coverage-v8`

## Development Dependencies

### Linting and Formatting

| Tool | Purpose |
|------|---------|
| ESLint | Code linting with TypeScript rules |
| Prettier | Code formatting |
| Husky | Git hooks |
| lint-staged | Pre-commit linting |

### Development Tools

| Tool | Purpose |
|------|---------|
| tsx | TypeScript execution |
| cross-env | Cross-platform env vars |
| glob | File pattern matching |

## UI Component Libraries

### Ink Ecosystem

```json
{
  "ink": "npm:@jrichman/ink@6.4.7",
  "ink-gradient": "^3.0.0",
  "ink-spinner": "^5.0.0"
}
```

### Syntax Highlighting

```json
{
  "highlight.js": "^11.11.1",
  "lowlight": "^3.3.0"
}
```

## File System and Shell

### File Operations

| Library | Purpose |
|---------|---------|
| `fdir` | Fast directory traversal |
| `glob` | Pattern matching |
| `picomatch` | Glob pattern matcher |
| `ignore` | .gitignore handling |

### Shell Integration

| Library | Purpose |
|---------|---------|
| `node-pty` | PTY support (optional) |
| `shell-quote` | Shell command parsing |
| `@xterm/headless` | Terminal emulation |

### Search Tools

| Library | Purpose |
|---------|---------|
| `@joshua.litt/get-ripgrep` | ripgrep binary |
| `fzf` | Fuzzy finder |
| `web-tree-sitter` | Code parsing |

## HTTP and Network

### HTTP Client

```json
{
  "undici": "^7.10.0",
  "https-proxy-agent": "^7.0.6"
}
```

### Content Processing

| Library | Purpose |
|---------|---------|
| `html-to-text` | HTML conversion |
| `marked` | Markdown parsing |
| `mime` | MIME type detection |
| `chardet` | Character encoding detection |

## Architecture Summary

```mermaid
graph TB
    subgraph Runtime
        Node[Node.js 20+]
        ESM[ES Modules]
    end
    
    subgraph UI
        Ink[Ink/React]
        HLJS[highlight.js]
    end
    
    subgraph Core
        Zod[Zod Schemas]
        GenAI[@google/genai]
        MCP[MCP SDK]
    end
    
    subgraph Build
        ESBuild[esbuild]
        TS[TypeScript]
    end
    
    subgraph Test
        Vitest[Vitest]
        MSW[MSW]
    end
    
    Node --> ESM
    ESM --> UI
    ESM --> Core
    Build --> Node
    Test --> Core
```

## Version Compatibility

| Dependency | Minimum Version | Notes |
|------------|-----------------|-------|
| Node.js | 20.0.0 | Required for ES modules |
| npm | 10.x | Workspace support |
| TypeScript | 5.3.3 | Template literal types |
| React | 19.x | Required by Ink |
