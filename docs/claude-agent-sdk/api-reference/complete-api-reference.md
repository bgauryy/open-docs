# Claude Agent SDK - Complete API Reference

**SDK Version**: 0.1.22
**Source**: SDK source code

---

## Table of Contents

1. [External API Endpoints](#external-api-endpoints)
2. [Authentication & Authorization](#authentication--authorization)
3. [Network Architecture](#network-architecture)
4. [SDK Public API](#sdk-public-api)
5. [Model Reference](#model-reference)
6. [Message Protocol](#message-protocol)
7. [Tool API Reference](#tool-api-reference)
8. [Hook API Reference](#hook-api-reference)
9. [Configuration API](#configuration-api)
10. [Type Definitions](#type-definitions)

---

## External API Endpoints

### Anthropic API Endpoints

**Production Environment** (Default):

```typescript
const PRODUCTION_ENDPOINTS = {
  // Primary API
  API_BASE: 'https://api.anthropic.com',
  API_VERSION: '2023-06-01',
  
  // Console (Dashboard)
  CONSOLE_BASE: 'https://console.anthropic.com',
  
  // Claude.ai (Consumer)
  CLAUDE_AI_BASE: 'https://claude.ai',
  
  // OAuth & Authentication
  OAUTH_AUTHORIZE: 'https://console.anthropic.com/oauth/authorize',
  OAUTH_TOKEN: 'https://console.anthropic.com/v1/oauth/token',
  OAUTH_API_KEY: 'https://api.anthropic.com/api/oauth/claude_cli/create_api_key',
  OAUTH_ROLES: 'https://api.anthropic.com/api/oauth/claude_cli/roles',
  
  // OAuth Success URLs
  CONSOLE_SUCCESS: 'https://console.anthropic.com/buy_credits?returnUrl=/oauth/code/success%3Fapp%3Dclaude-code',
  CLAUDEAI_SUCCESS: 'https://console.anthropic.com/oauth/code/success?app=claude-code',
  MANUAL_REDIRECT: 'https://console.anthropic.com/oauth/code/callback',
  
  // OAuth Client
  CLIENT_ID: '9d1c250a-e61b-44d9-88ed-5944d1962f5e',
  
  // OAuth Scopes
  SCOPES: [
    'org:create_api_key',
    'user:profile',
    'user:inference'
  ]
};
```

**API Version**: `2023-06-01` (Header: `anthropic-version`)

---

### API Request Structure

#### Messages API (Streaming)

**Endpoint**: `POST https://api.anthropic.com/v1/messages`

**Headers**:
```http
anthropic-version: 2023-06-01
content-type: application/json
x-api-key: sk-ant-api03-...
anthropic-beta: max-tokens-3-5-sonnet-2024-07-15
```

**Request Body**:
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 8192,
  "messages": [
    {
      "role": "user",
      "content": "Hello, Claude!"
    }
  ],
  "system": "You are a helpful assistant.",
  "thinking": {
    "type": "enabled",
    "budget_tokens": 31999
  },
  "tools": [
    {
      "name": "get_weather",
      "description": "Get weather for a location",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": {
            "type": "string",
            "description": "City name"
          }
        },
        "required": ["location"]
      }
    }
  ],
  "stream": true,
  "temperature": 1.0,
  "top_p": 1.0,
  "top_k": 0
}
```

**Response (Streaming)**:
```
event: message_start
data: {"type":"message_start","message":{"id":"msg_abc123",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"!"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":2}}

event: message_stop
data: {"type":"message_stop"}
```

---

### Rate Limits

**Production Limits** (as of 2025-10):

| Model | Tier | Requests/min | Tokens/min | Tokens/day |
|-------|------|--------------|------------|------------|
| **Claude 3.5 Sonnet** | Free | 5 | 40,000 | 1,000,000 |
| **Claude 3.5 Sonnet** | Tier 1 | 50 | 40,000 | 5,000,000 |
| **Claude 3.5 Sonnet** | Tier 2 | 1,000 | 80,000 | 25,000,000 |
| **Claude Opus 4** | All | 10 | 10,000 | 3,000,000 |
| **Claude 3.5 Haiku** | All | 50 | 50,000 | 5,000,000 |

**Rate Limit Headers**:
```http
x-ratelimit-limit: 50
x-ratelimit-remaining: 49
x-ratelimit-reset: 2025-10-24T12:00:00Z
retry-after: 60
```

---

## Authentication & Authorization

### Authentication Flow (Internal)

```
┌──────────────────────────────────────────────────────────┐
│  1. SDK Initialization                                   │
│     ↓                                                     │
│  Check API Key Sources (in order):                       │
│     ↓                                                     │
│  ┌──────────────────────────────────────────────┐        │
│  │ a. options.apiKey (explicit parameter)       │        │
│  │    ↓ (if not found)                          │        │
│  │ b. ANTHROPIC_API_KEY env var                 │        │
│  │    ↓ (if not found)                          │        │
│  │ c. ~/.claude/settings.json                   │        │
│  │    ↓ (if not found)                          │        │
│  │ d. .claude/settings.json (project)           │        │
│  │    ↓ (if not found)                          │        │
│  │ e. OAuth token (if authenticated)            │        │
│  │    ↓ (if not found)                          │        │
│  │ f. ERROR: No API key found                   │        │
│  └──────────────────────────────────────────────┘        │
│     ↓                                                     │
│  2. API Key Retrieved                                    │
│     ↓                                                     │
│  3. Add to Request Headers                               │
│     x-api-key: sk-ant-api03-...                          │
│     ↓                                                     │
│  4. Send Request to api.anthropic.com                    │
│     ↓                                                     │
│  5. Anthropic validates key                              │
│     ↓                                                     │
│  ┌──────────────────────────────────┐                    │
│  │ Valid?                           │                    │
│  └──┬───────────────────────────┬───┘                    │
│     │ Yes                       │ No                     │
│     ↓                           ↓                        │
│  200 OK                      401 Unauthorized            │
│  Continue                    Error: Invalid API key      │
└──────────────────────────────────────────────────────────┘
```

### API Key Sources

#### 1. Explicit Parameter

```typescript
const result = await query({
  prompt: "Hello",
  options: {
    apiKey: "sk-ant-api03-your-key-here"  // Direct parameter
  }
});
```

**Priority**: 🔴 Highest (overrides everything)  
**Use Case**: Testing, temporary keys  
**Security**: ⚠️ Not recommended - key visible in code

#### 2. Environment Variable

```bash
export ANTHROPIC_API_KEY="sk-ant-api03-your-key-here"
```

```typescript
// SDK automatically reads from process.env
const result = await query({ prompt: "Hello" });
```

**Priority**: 🟡 High  
**Use Case**: Development, CI/CD  
**Security**: ✓ Good - not in code

#### 3. User Settings

**File**: `~/.claude/settings.json`

```json
{
  "apiKey": "sk-ant-api03-your-key-here"
}
```

**Priority**: 🟢 Medium  
**Use Case**: Personal development  
**Security**: ✓ Good - per-user

#### 4. Project Settings

**File**: `.claude/settings.json` (in project root)

```json
{
  "apiKey": "sk-ant-api03-team-key-here"
}
```

**Priority**: 🔵 Medium-Low  
**Use Case**: Team projects  
**Security**: ⚠️ Add to .gitignore!

#### 5. OAuth Token

```bash
# Authenticate via OAuth
npx @anthropic-ai/claude-agent-sdk auth login
```

**Priority**: 🟣 Low  
**Use Case**: Enterprise SSO  
**Security**: ✓ Best - temporary tokens

---

### OAuth 2.0 Flow (Complete Internal Implementation)

#### Authorization Code Flow with PKCE

```
┌──────────────────────────────────────────────────────────┐
│  1. User runs: npx ... auth login                        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  2. SDK generates PKCE values:                           │
│     code_verifier = random(43-128 chars)                 │
│     code_challenge = base64url(sha256(code_verifier))    │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  3. SDK starts local HTTP server on random port          │
│     Redirect URI: http://localhost:[PORT]/callback       │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  4. SDK constructs authorization URL:                    │
│                                                           │
│  https://console.anthropic.com/oauth/authorize?          │
│    client_id=9d1c250a-e61b-44d9-88ed-5944d1962f5e        │
│    &response_type=code                                   │
│    &redirect_uri=http://localhost:[PORT]/callback        │
│    &scope=org:create_api_key+user:profile+user:inference │
│    &code_challenge=[CHALLENGE]                           │
│    &code_challenge_method=S256                           │
│    &state=[RANDOM_STATE]                                 │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  5. SDK opens URL in browser                             │
│     User sees Anthropic login page                       │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  6. User authenticates with Anthropic                    │
│     - Email + Password                                   │
│     - Or Google/GitHub SSO                               │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  7. User authorizes application                          │
│     Grants permissions: create_api_key, profile, inference│
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  8. Anthropic redirects to callback:                     │
│                                                           │
│  http://localhost:[PORT]/callback?                       │
│    code=[AUTH_CODE]                                      │
│    &state=[SAME_STATE]                                   │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  9. SDK receives callback, validates state               │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  10. SDK exchanges code for token:                       │
│                                                           │
│  POST https://console.anthropic.com/v1/oauth/token       │
│  {                                                        │
│    "grant_type": "authorization_code",                   │
│    "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e",  │
│    "code": "[AUTH_CODE]",                                │
│    "redirect_uri": "http://localhost:[PORT]/callback",   │
│    "code_verifier": "[VERIFIER]"                         │
│  }                                                        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  11. Anthropic returns access token:                     │
│  {                                                        │
│    "access_token": "eyJ...",                             │
│    "token_type": "Bearer",                               │
│    "expires_in": 3600,                                   │
│    "refresh_token": "..."                                │
│  }                                                        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  12. SDK uses access token to create API key:            │
│                                                           │
│  POST https://api.anthropic.com/api/oauth/               │
│       claude_cli/create_api_key                          │
│  Authorization: Bearer eyJ...                            │
│  {                                                        │
│    "name": "Claude Code CLI",                            │
│    "scopes": ["user:inference"]                          │
│  }                                                        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  13. Anthropic returns API key:                          │
│  {                                                        │
│    "api_key": "sk-ant-api03-..."                         │
│  }                                                        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  14. SDK stores API key in ~/.claude/settings.json       │
│  {                                                        │
│    "apiKey": "sk-ant-api03-...",                         │
│    "apiKeySource": "temporary",                          │
│    "email": "user@example.com",                          │
│    "organization": "Company Name"                        │
│  }                                                        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│  15. Authentication complete!                            │
│      User can now use SDK                                │
└──────────────────────────────────────────────────────────┘
```

#### OAuth Endpoints

```typescript
const OAUTH_CONFIG = {
  // Client ID (public, not secret)
  CLIENT_ID: '9d1c250a-e61b-44d9-88ed-5944d1962f5e',
  
  // Authorization endpoint
  AUTHORIZE_URL: 'https://console.anthropic.com/oauth/authorize',
  
  // Token exchange endpoint
  TOKEN_URL: 'https://console.anthropic.com/v1/oauth/token',
  
  // API key creation endpoint
  API_KEY_URL: 'https://api.anthropic.com/api/oauth/claude_cli/create_api_key',
  
  // User roles endpoint
  ROLES_URL: 'https://api.anthropic.com/api/oauth/claude_cli/roles',
  
  // Required scopes
  SCOPES: [
    'org:create_api_key',  // Create API keys for organization
    'user:profile',         // Access user profile info
    'user:inference'        // Use Claude for inference
  ],
  
  // OAuth protocol version
  PROTOCOL_VERSION: 'oauth-2025-04-20'
};
```

---

### API Key Types

**From `ApiKeySource` type**:

```typescript
type ApiKeySource = 
  | 'user'      // User-level key (personal)
  | 'project'   // Project-level key (team)
  | 'org'       // Organization-level key (enterprise)
  | 'temporary' // OAuth temporary key
```

**Permissions by Type**:

| Type | Scope | Duration | Use Case |
|------|-------|----------|----------|
| **user** | Personal workspace | Permanent | Individual development |
| **project** | Project workspace | Permanent | Team collaboration |
| **org** | Organization-wide | Permanent | Enterprise deployment |
| **temporary** | Session-based | Expires (1 hour default) | SSO authentication |

---

## Network Architecture

### Request Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│  User Application                                       │
│  ┌──────────────────────────────────────────┐           │
│  │ query({ prompt: "..." })                 │           │
│  └────────────────┬─────────────────────────┘           │
└───────────────────┼──────────────────────────────────────┘
                    │
                    │ HTTPS (TLS 1.2+)
                    │
┌───────────────────▼──────────────────────────────────────┐
│  Anthropic API (api.anthropic.com)                      │
│  ┌──────────────────────────────────────────┐           │
│  │ Load Balancer (443)                      │           │
│  └────────────────┬─────────────────────────┘           │
│                   │                                      │
│  ┌────────────────▼─────────────────────────┐           │
│  │ API Gateway                              │           │
│  │ • Rate limiting                          │           │
│  │ • Authentication                         │           │
│  │ • Request validation                     │           │
│  └────────────────┬─────────────────────────┘           │
│                   │                                      │
│  ┌────────────────▼─────────────────────────┐           │
│  │ Claude Inference Service                 │           │
│  │ • Model routing                          │           │
│  │ • Token counting                         │           │
│  │ • Response streaming                     │           │
│  └────────────────┬─────────────────────────┘           │
└───────────────────┼──────────────────────────────────────┘
                    │
                    │ Server-Sent Events (SSE)
                    │
┌───────────────────▼──────────────────────────────────────┐
│  User Application (streaming response)                  │
└──────────────────────────────────────────────────────────┘
```

### Network Endpoints & Protocols

| Endpoint | Protocol | Port | Purpose |
|----------|----------|------|---------|
| **api.anthropic.com** | HTTPS | 443 | Primary API |
| **console.anthropic.com** | HTTPS | 443 | OAuth & Dashboard |
| **claude.ai** | HTTPS | 443 | Consumer interface |

### DNS Resolution

```bash
# Primary API
$ nslookup api.anthropic.com
# Returns multiple IPs (load balanced)

# Expected IPs (may change):
# AWS CloudFront distribution
# Multiple edge locations worldwide
```

**Note**: Anthropic uses AWS infrastructure. IPs are dynamic and may change. Always resolve via DNS.

### SSL/TLS Configuration

```typescript
const TLS_CONFIG = {
  protocol: 'TLS 1.2+',  // Minimum version
  ciphers: [
    'TLS_AES_128_GCM_SHA256',
    'TLS_AES_256_GCM_SHA384',
    'TLS_CHACHA20_POLY1305_SHA256'
  ],
  certificates: 'AWS Certificate Manager'
};
```

### Proxy Support

```bash
# HTTP proxy
export HTTP_PROXY=http://proxy.company.com:8080

# HTTPS proxy (for API calls)
export HTTPS_PROXY=http://proxy.company.com:8080

# No proxy for local
export NO_PROXY=localhost,127.0.0.1
```

SDK automatically respects proxy environment variables.

---

## SDK Public API

### Main Export: `query()`

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

export function query(params: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;
```

**Returns**: `Query` (AsyncGenerator)

**Usage**:
```typescript
const result = await query({
  prompt: "Hello, Claude!",
  options: {
    model: 'sonnet',
    permissionMode: 'default'
  }
});

for await (const message of result) {
  // Process streaming messages
}
```

---

### Query Interface

```typescript
export interface Query extends AsyncGenerator<SDKMessage, void> {
  // Streaming interface
  [Symbol.asyncIterator](): AsyncIterator<SDKMessage>;
  next(): Promise<IteratorResult<SDKMessage, void>>;
  
  // Control methods
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

**Methods**:

#### `interrupt()`
Stop execution immediately.

```typescript
const result = await query({ prompt: "Long task" });

// After 5 seconds, interrupt
setTimeout(async () => {
  await result.interrupt();
}, 5000);
```

#### `setPermissionMode(mode)`
Change permission mode mid-conversation.

```typescript
const result = await query({ prompt: "Start" });

// Later, relax permissions
await result.setPermissionMode('acceptEdits');
```

#### `setModel(model)`
Switch model dynamically.

```typescript
const result = await query({ prompt: "Simple task" });

// Switch to Opus for complex reasoning
await result.setModel('opus');
```

#### `setMaxThinkingTokens(tokens)`
Enable/disable extended thinking.

```typescript
const result = await query({ prompt: "Task" });

// Enable Ultrathink for complex step
await result.setMaxThinkingTokens(31999);
```

#### `supportedCommands()`
Get available slash commands.

```typescript
const commands = await result.supportedCommands();
// Returns: [{ name: '/help', description: '...' }, ...]
```

#### `supportedModels()`
Get available models.

```typescript
const models = await result.supportedModels();
// Returns: [
//   { 
//     name: 'claude-3-5-sonnet-20241022',
//     maxTokens: 8192,
//     contextWindow: 200000
//   },
//   ...
// ]
```

#### `mcpServerStatus()`
Check MCP server health.

```typescript
const status = await result.mcpServerStatus();
// Returns: [
//   { 
//     name: 'my-server',
//     status: 'connected',
//     serverInfo: { version: '1.0.0' }
//   },
//   ...
// ]
```

#### `accountInfo()`
Get account information.

```typescript
const account = await result.accountInfo();
// Returns: {
//   email: 'user@example.com',
//   organization: 'Company',
//   subscriptionType: 'Pro',
//   tokenSource: 'api_key',
//   apiKeySource: 'user'
// }
```

---

### Tool Creation: `tool()`

```typescript
export function tool<T extends ZodSchema>(
  name: string,
  description: string,
  inputSchema: T,
  execute: (args: z.infer<T>, extra: ToolExecutionContext) => Promise<ToolResult>
): Tool;
```

**Usage**:
```typescript
import { tool } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

const weatherTool = tool(
  'get_weather',
  'Get current weather for a location',
  {
    location: z.string().describe('City name'),
    units: z.enum(['celsius', 'fahrenheit']).optional()
  },
  async (args) => {
    const weather = await fetchWeather(args.location, args.units);
    return {
      content: [{
        type: 'text',
        text: `Weather in ${args.location}: ${weather.temp}°`
      }],
      isError: false
    };
  }
);
```

---

### MCP Server Creation: `createSdkMcpServer()`

```typescript
export function createSdkMcpServer(config: {
  name: string;
  version?: string;
  tools: Tool[];
  resources?: Resource[];
}): McpSdkServerConfig;
```

**Usage**:
```typescript
import { createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk';

const server = createSdkMcpServer({
  name: 'custom-tools',
  version: '1.0.0',
  tools: [weatherTool, calculatorTool]
});

const result = await query({
  prompt: "Use custom tools",
  options: {
    mcpServers: {
      'custom': server
    }
  }
});
```

---

## Model Reference

### Available Models

| Model ID | Name | Context | Max Output | Cost (Input/Output) |
|----------|------|---------|------------|---------------------|
| `claude-opus-4-20250514` | **Claude Opus 4** | 200K | 8K | $15/$75 per MTok |
| `claude-3-5-sonnet-20241022` | **Claude 3.5 Sonnet** | 200K | 8K | $3/$15 per MTok |
| `claude-3-5-haiku-20241022` | **Claude 3.5 Haiku** | 200K | 8K | $1/$5 per MTok |

### Model Aliases

```typescript
const MODEL_ALIASES = {
  'opus': 'claude-opus-4-20250514',
  'sonnet': 'claude-3-5-sonnet-20241022',
  'haiku': 'claude-3-5-haiku-20241022'
};
```

**Usage**:
```typescript
// Use alias
{ model: 'sonnet' }

// Use full ID
{ model: 'claude-3-5-sonnet-20241022' }
```

### Model Capabilities

| Feature | Opus 4 | Sonnet 3.5 | Haiku 3.5 |
|---------|--------|------------|-----------|
| **Vision** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Tool Use** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Extended Thinking** | ✅ Yes (8K) | ❌ No | ❌ No |
| **Prompt Caching** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Context Window** | 200K | 200K | 200K |
| **Max Output** | 8K | 8K | 8K |
| **Speed** | ⚡ Slow | ⚡⚡ Fast | ⚡⚡⚡ Very Fast |
| **Cost** | 💰💰💰 High | 💰💰 Medium | 💰 Low |

---

## Message Protocol

### Message Types

```typescript
export type SDKMessage = 
  | SDKUserMessage                // User input
  | SDKUserMessageReplay          // User message replay
  | SDKAssistantMessage           // Complete assistant response
  | SDKPartialAssistantMessage    // Streaming chunk
  | SDKResultMessage              // Final result with metrics
  | SDKSystemMessage              // System initialization
  | SDKCompactBoundaryMessage     // Context compaction marker
  | SDKHookResponseMessage;       // Hook execution result
```

### SDKSystemMessage

```typescript
type SDKSystemMessage = {
  type: 'system';
  subtype: 'init';
  session_id: string;
  agents?: string[];
  apiKeySource: 'user' | 'project' | 'org' | 'temporary';
  claude_code_version: string;
  cwd: string;
  tools: string[];
  mcp_servers: {
    name: string;
    description?: string;
  }[];
};
```

### SDKResultMessage

```typescript
type SDKResultMessage = {
  type: 'result';
  subtype: 'success' | 'error';
  usage: {
    input_tokens: number;
    output_tokens: number;
    cache_creation_input_tokens: number;
    cache_read_input_tokens: number;
  };
  modelUsage: {
    [modelName: string]: {
      inputTokens: number;
      outputTokens: number;
      cacheReadInputTokens: number;
      cacheCreationInputTokens: number;
      webSearchRequests: number;
      costUSD: number;
      contextWindow: number;
    };
  };
  total_cost_usd: number;
  num_turns: number;
  duration_ms: number;
  duration_api_ms: number;
};
```

---

## Tool API Reference

**See**: [Complete Tool System Documentation](../extraction/tools-system-complete.md)

All 17 built-in tools with complete TypeScript schemas, parameters, and examples.

---

## Hook API Reference

**See**: [Complete Hooks System Documentation](../extraction/hooks-system-complete.md)

All 9 hook events with input/output structures and patterns.

---

## Configuration API

**See**: [Complete Configuration Documentation](../extraction/configuration-complete.md)

7-level configuration resolution with complete settings.json schema.

---

## Type Definitions

**See**: [Complete Type System Documentation](../extraction/sdk-types-complete.md)

All TypeScript types exported by the SDK.

---

## Summary

### Quick Reference

**Primary API**:
- Base URL: `https://api.anthropic.com`
- Version: `2023-06-01`
- Auth: `x-api-key` header

**OAuth Endpoints**:
- Authorize: `https://console.anthropic.com/oauth/authorize`
- Token: `https://console.anthropic.com/v1/oauth/token`
- API Key: `https://api.anthropic.com/api/oauth/claude_cli/create_api_key`

**Models**:
- Opus 4: `claude-opus-4-20250514` ($15/$75)
- Sonnet 3.5: `claude-3-5-sonnet-20241022` ($3/$15)
- Haiku 3.5: `claude-3-5-haiku-20241022` ($1/$5)

**SDK Exports**:
- `query()` - Main function
- `tool()` - Create custom tools
- `createSdkMcpServer()` - Create MCP server
