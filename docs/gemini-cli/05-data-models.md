# Data Models

## Overview

This document describes the key data structures, types, and schemas used throughout Gemini CLI. Understanding these models is essential for working with the codebase and extending its functionality.

## Configuration Models

### ConfigParameters Interface

The main configuration interface that controls all runtime behavior.

```typescript
interface ConfigParameters {
  // Authentication
  apiKey?: string;
  oauth?: OAuthConfig;
  authType?: AuthType;
  
  // Model Configuration
  model?: string;
  temperature?: number;
  maxOutputTokens?: number;
  
  // Tool Configuration
  coreTools?: string[];
  allowedTools?: string[];
  excludeTools?: string[];
  mcpEnabled?: boolean;
  extensionsEnabled?: boolean;
  
  // Directory Configuration
  targetDir?: string;
  includeDirectories?: string[];
  
  // Sandbox Configuration
  sandbox?: SandboxConfig;
  
  // Feature Flags
  enableAgents?: boolean;
  skillsSupport?: boolean;
  enableHooks?: boolean;
  previewFeatures?: boolean;
  
  // UI Configuration
  accessibility?: AccessibilitySettings;
  output?: OutputSettings;
  
  // Telemetry
  telemetrySettings?: TelemetrySettings;
  usageStatisticsEnabled?: boolean;
  
  // Advanced
  policyEngineConfig?: PolicyEngineConfig;
  modelConfigServiceConfig?: ModelConfigServiceConfig;
  experiments?: Experiments;
}
```

### SandboxConfig

Configuration for safe command execution.

```typescript
interface SandboxConfig {
  enabled: boolean;
  type: 'docker' | 'podman' | 'none';
  image?: string;
  mountPoints?: MountPoint[];
  networkMode?: 'none' | 'host' | 'bridge';
}

interface MountPoint {
  source: string;
  target: string;
  readonly: boolean;
}
```

### AccessibilitySettings

```typescript
interface AccessibilitySettings {
  screenReaderFriendly: boolean;
  highContrast: boolean;
  reducedMotion: boolean;
}
```

### OutputSettings

```typescript
interface OutputSettings {
  format: 'text' | 'json' | 'stream-json';
  color: boolean;
  verbose: boolean;
}
```

## Chat and Conversation Models

### Content Structure

Based on the Gemini API content format:

```typescript
interface Content {
  role: 'user' | 'model' | 'function';
  parts: Part[];
}

interface Part {
  text?: string;
  inlineData?: InlineData;
  functionCall?: FunctionCall;
  functionResponse?: FunctionResponse;
}

interface InlineData {
  mimeType: string;
  data: string; // Base64 encoded
}

interface FunctionCall {
  name: string;
  args: Record<string, unknown>;
}

interface FunctionResponse {
  name: string;
  response: {
    content: unknown;
  };
}
```

### Session State

```typescript
interface SessionState {
  sessionId: string;
  history: Content[];
  checkpoint?: CheckpointData;
  metadata: SessionMetadata;
}

interface SessionMetadata {
  createdAt: string;
  lastUpdatedAt: string;
  model: string;
  tokenCount: number;
}

interface CheckpointData {
  id: string;
  name?: string;
  timestamp: string;
  historyLength: number;
}
```

### ResumedSessionData

Data structure for resuming saved sessions.

```typescript
interface ResumedSessionData {
  sessionId: string;
  checkpoint: CheckpointData;
  history: Content[];
  systemInstruction?: string;
}
```

## Tool Models

### Tool Definition

```typescript
interface ToolDefinition {
  name: string;
  description: string;
  parameters: ToolParameterSchema;
  requiresConfirmation: boolean;
  serverName?: string; // For MCP tools
}

type ToolParameterSchema = z.ZodType<unknown>;
```

### ToolInvocation

Represents a tool execution request.

```typescript
interface ToolInvocation<TParams, TResult> {
  toolName: string;
  params: TParams;
  signal: AbortSignal;
  
  execute(): Promise<TResult>;
  getConfirmationDetails?(): Promise<ToolCallConfirmationDetails | false>;
}
```

### ToolResult

Standard tool execution result.

```typescript
interface ToolResult {
  success: boolean;
  output?: string;
  error?: string;
  metadata?: Record<string, unknown>;
}
```

### ToolCallConfirmationDetails

Information presented to user for confirmation.

```typescript
interface ToolCallConfirmationDetails {
  tool: string;
  params: Record<string, unknown>;
  description: string;
  risk: 'low' | 'medium' | 'high';
  serverName?: string;
  onConfirm: (outcome: ToolConfirmationOutcome) => Promise<void>;
}

type ToolConfirmationOutcome = 
  | { approved: true }
  | { approved: false; reason: string };
```

### ToolLocation

For tools that reference file locations.

```typescript
interface ToolLocation {
  path: string;
  startLine?: number;
  endLine?: number;
  startColumn?: number;
  endColumn?: number;
}
```

## Agent Models

### AgentSpec

Agent specification and configuration.

```typescript
interface AgentSpec {
  name: string;
  description: string;
  systemInstruction: string;
  tools?: string[];
  model?: string;
  temperature?: number;
  outputSchema?: z.ZodType<unknown>;
}
```

### AgentLoadError

Error when agent loading fails.

```typescript
class AgentLoadError extends Error {
  constructor(
    public agentName: string,
    public reason: string,
    public cause?: Error
  ) {
    super(`Failed to load agent '${agentName}': ${reason}`);
    this.name = 'AgentLoadError';
  }
}
```

### Agent Invocation

```typescript
interface AgentInvocationParams {
  agent: string;
  task: string;
  context?: Record<string, unknown>;
}

interface AgentInvocationResult<T> {
  success: boolean;
  output?: T;
  error?: string;
  toolCalls?: ToolCall[];
}
```

## MCP Models

### MCPServerConfig

Configuration for MCP servers.

```typescript
interface MCPServerConfig {
  command: string;
  args?: string[];
  env?: Record<string, string>;
  cwd?: string;
  timeout?: number;
  oauth?: MCPOAuthConfig;
}

interface MCPOAuthConfig {
  clientId: string;
  clientSecret?: string;
  scopes: string[];
  authorizationUrl: string;
  tokenUrl: string;
}
```

### MCP Tool Discovery

```typescript
interface DiscoveredMCPToolInfo {
  name: string;
  description: string;
  inputSchema: JSONSchema;
  serverName: string;
}
```

### MCP Resource

```typescript
interface MCPResource {
  uri: string;
  name: string;
  description?: string;
  mimeType?: string;
}
```

## Hook Models

### HookDefinition

```typescript
interface HookDefinition {
  name: string;
  command: string;
  args?: string[];
  env?: Record<string, string>;
  timeout?: number;
  continueOnError?: boolean;
}

type HookEventName = 
  | 'pre-prompt'
  | 'post-prompt'
  | 'pre-tool'
  | 'post-tool'
  | 'on-error'
  | 'on-start'
  | 'on-exit';
```

### Hook Event Payload

```typescript
interface HookEventPayload {
  eventName: HookEventName;
  sessionId: string;
  cwd: string;
  timestamp: string;
  data: Record<string, unknown>;
}

// Specific event payloads
interface PrePromptPayload extends HookEventPayload {
  eventName: 'pre-prompt';
  data: {
    prompt: string;
    history: Content[];
  };
}

interface PreToolPayload extends HookEventPayload {
  eventName: 'pre-tool';
  data: {
    tool: string;
    params: Record<string, unknown>;
  };
}
```

## Skill Models

### SkillDefinition

```typescript
interface SkillDefinition {
  name: string;
  description: string;
  version: string;
  activationCommand?: string;
  tools?: ToolDefinition[];
  prompts?: PromptTemplate[];
  resources?: ResourceDefinition[];
}
```

### SkillActivation

```typescript
interface SkillActivationParams {
  skillName: string;
  context?: Record<string, unknown>;
}

interface ActivatedSkill {
  name: string;
  tools: string[];
  active: boolean;
}
```

## Workspace Models

### WorkspaceContext

Information about the current working context.

```typescript
interface WorkspaceContext {
  rootPath: string;
  includedDirectories: string[];
  gitInfo?: GitInfo;
  geminiMdPaths: string[];
  projectType?: ProjectType;
}

interface GitInfo {
  isGitRepo: boolean;
  branch?: string;
  remoteUrl?: string;
  uncommittedChanges?: boolean;
}

type ProjectType = 
  | 'node'
  | 'python'
  | 'rust'
  | 'go'
  | 'java'
  | 'unknown';
```

## Routing Models

### RoutingContext

Context for model selection decisions.

```typescript
interface RoutingContext {
  promptLength: number;
  historyLength: number;
  toolsRequested: string[];
  userPreference?: string;
  availableModels: string[];
}
```

### RoutingDecision

```typescript
interface RoutingDecision {
  selectedModel: string;
  reason: string;
  fallbackModels: string[];
}
```

## Telemetry Models

### TelemetryEvent Base

```typescript
interface BaseTelemetryEvent {
  eventName: string;
  timestamp: string;
  sessionId: string;
  properties: Record<string, unknown>;
}
```

### Specific Event Types

```typescript
class AgentStartEvent extends BaseTelemetryEvent {
  eventName = 'agent_start';
  properties: {
    model: string;
    toolCount: number;
  };
}

class AgentFinishEvent extends BaseTelemetryEvent {
  eventName = 'agent_finish';
  properties: {
    success: boolean;
    duration: number;
    tokenCount: number;
  };
}

class ToolCallEvent extends BaseTelemetryEvent {
  eventName = 'tool_call';
  properties: {
    tool: string;
    success: boolean;
    duration: number;
  };
}
```

## Error Models

### Error Classification

```typescript
interface ClassifiedError {
  category: ErrorCategory;
  severity: ErrorSeverity;
  recoverable: boolean;
  userMessage: string;
  technicalDetails?: string;
}

type ErrorCategory = 
  | 'authentication'
  | 'rate_limit'
  | 'network'
  | 'validation'
  | 'execution'
  | 'internal';

type ErrorSeverity = 'warning' | 'error' | 'fatal';
```

## Validation Schemas

All major models use Zod for runtime validation:

```typescript
import { z } from 'zod';

// Example: Config validation
const ConfigSchema = z.object({
  model: z.string().optional(),
  temperature: z.number().min(0).max(2).optional(),
  maxOutputTokens: z.number().positive().optional(),
  // ... more fields
});

// Type inference
type Config = z.infer<typeof ConfigSchema>;

// Runtime validation
const config = ConfigSchema.parse(rawConfig);
```

## Serialization

### JSON Serialization

Most models serialize directly to JSON:

```typescript
const session: SessionState = { ... };
const json = JSON.stringify(session);
const restored = JSON.parse(json) as SessionState;
```

### Terminal Output Serialization

Special serialization for terminal content:

```typescript
interface AnsiOutput {
  lines: AnsiLine[];
}

interface AnsiLine {
  tokens: AnsiToken[];
}

interface AnsiToken {
  text: string;
  style?: AnsiStyle;
}

interface AnsiStyle {
  foreground?: string;
  background?: string;
  bold?: boolean;
  italic?: boolean;
  underline?: boolean;
}
```
