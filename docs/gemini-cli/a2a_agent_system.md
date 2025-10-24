# Gemini CLI - A2A (Agent-to-Agent) Server Reference

**Generated:** 2024-10-24
**Purpose:** Complete documentation of the A2A server architecture, agent system, and routing

**STATUS:** ⚠️ **EXPERIMENTAL** - All code in this package is under active development

---

## Table of Contents

1. [A2A Server Overview](#a2a-server-overview)
2. [Architecture](#architecture)
3. [Task System](#task-system)
4. [Agent Executor](#agent-executor)
5. [HTTP API](#http-api)
6. [Persistence Layer](#persistence-layer)
7. [Configuration](#configuration)
8. [Command Registry](#command-registry)
9. [Event System](#event-system)
10. [Deployment](#deployment)

---

## A2A Server Overview

### What is A2A?

**A2A (Agent-to-Agent)** is a server implementation that allows Gemini CLI to function as a remotely-accessible AI agent service.

**Purpose:**
- Expose Gemini CLI capabilities via HTTP API
- Enable agent-to-agent communication
- Support multi-user, multi-task execution
- Provide persistent task storage

**Status:** BETA (experimental, APIs may change)

---

### Use Cases

1. **Remote Agent Service**
   - Run Gemini CLI as a backend service
   - Access from multiple clients
   - Centralized execution environment

2. **Agent Orchestration**
   - Connect multiple agents
   - Delegate subtasks to specialized agents
   - Coordinate complex workflows

3. **Cloud Deployment**
   - Deploy on cloud platforms (GCP, AWS, Azure)
   - Scale horizontally with multiple instances
   - Persistent task storage (GCS)

4. **IDE Integration**
   - Backend for IDE extensions
   - Shared agent across multiple IDEs/users
   - Centralized configuration management

---

## Architecture

### Package Structure

```
packages/a2a-server/
├── src/
│   ├── agent/              # Agent execution engine
│   │   ├── executor.ts     # CoderAgentExecutor
│   │   └── task.ts         # Task class
│   ├── commands/           # Command registry
│   │   ├── command-registry.ts
│   │   └── list-extensions.ts
│   ├── config/             # Configuration
│   │   ├── config.ts
│   │   ├── extension.ts
│   │   └── settings.ts
│   ├── http/               # HTTP server
│   │   ├── app.ts          # Express app
│   │   ├── endpoints.ts    # API routes (test)
│   │   ├── server.ts       # Server startup
│   │   └── requestStorage.ts
│   ├── persistence/        # Storage layer
│   │   └── gcs.ts          # Google Cloud Storage
│   ├── utils/              # Utilities
│   │   ├── executor_utils.ts
│   │   ├── logger.ts
│   │   └── testing_utils.ts
│   ├── types.ts            # Type definitions
│   └── index.ts            # Entry point
├── README.md
└── package.json
```

---

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      A2A HTTP Server                         │
│  (Express.js - Port: CODER_AGENT_PORT)                      │
└─────────────────────────┬───────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│              CoderAgentExecutor                              │
│  - Task management                                           │
│  - Multi-task execution                                      │
│  - Event dispatching                                         │
└─────────────────────────┬───────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│                    Task (per request)                        │
│  - Task state (running, completed, error)                   │
│  - Tool scheduler                                            │
│  - Event bus                                                 │
│  - Config & Gemini Client                                   │
└─────────────────────────┬───────────────────────────────────┘
                          ↓
┌───────────────────────────────────────────────────────────┬─┐
│              Core Gemini CLI Components                    │ │
│  - CoreToolScheduler                                       │ │
│  - ToolRegistry                                            │ │
│  - GeminiClient                                            │ │
│  - FileSystemService                                       │ │
└────────────────────────────────────────────────────────────┘ │
                          ↓                                     │
┌─────────────────────────────────────────────────────────────┐
│              Persistence Layer (Optional)                    │
│  - TaskStore (GCS or in-memory)                             │
│  - Save task state                                          │
│  - Load task history                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Task System

### Task Class

**Location:** `packages/a2a-server/src/agent/task.ts`

**Purpose:** Represents a single agent execution session

---

### Task Properties

```typescript
export class Task {
  id: string;                   // Unique task identifier
  contextId: string;            // Context/session identifier
  scheduler: CoreToolScheduler; // Tool execution scheduler
  config: Config;               // Configuration
  geminiClient: GeminiClient;   // Gemini API client
  pendingToolConfirmationDetails: Map<string, ToolCallConfirmationDetails>;
  taskState: TaskState;         // running | completed | error
  eventBus?: ExecutionEventBus; // Event bus for streaming
  completedToolCalls: CompletedToolCall[];
  skipFinalTrueAfterInlineEdit: boolean;
  
  // Private fields
  private pendingToolCalls: Map<string, string>;
  private toolCompletionPromise?: Promise<void>;
  private toolCompletionNotifier?: { resolve: () => void; reject: () => void };
}
```

---

### Task States

```typescript
// TaskState is imported from '@a2a-js/sdk'
enum TaskState {
  SUBMITTED = 'submitted',       // Task created, not yet executing
  WORKING = 'working',           // Task actively executing
  INPUT_REQUIRED = 'input-required',  // Waiting for user input
  COMPLETED = 'completed',       // Task finished successfully
  FAILED = 'failed',             // Task encountered error
  CANCELED = 'canceled',         // Task was canceled
}
```

---

### Task Lifecycle

```
Create Task
    ↓
Initialize (Config, Gemini Client, Tool Scheduler)
    ↓
Execute (Process user message + tool calls)
    ↓
┌─────────────────────────────────┐
│  Tool Call Loop                 │
│  1. AI requests tool            │
│  2. Schedule tool execution     │
│  3. Wait for tool completion    │
│  4. Return result to AI         │
│  5. Repeat if AI requests more  │
└─────────────────────────────────┘
    ↓
Complete (Save state, persist to storage)
```

---

### Task Events

**Event Types:**

```typescript
export enum CoderAgentEvent {
  ToolCallConfirmationEvent = 'tool-call-confirmation',
  ToolCallUpdateEvent = 'tool-call-update',
  TextContentEvent = 'text-content',
  StateChangeEvent = 'state-change',
  StateAgentSettingsEvent = 'agent-settings',
  ThoughtEvent = 'thought',
  // Note: ThoughtSummary is a type, not an event
}
```

---

#### STATE_CHANGE Event

**When:** Task state transitions (running → completed)

**Payload:**

```typescript
interface StateChange {
  type: CoderAgentEvent.STATE_CHANGE;
  state: TaskState;
  timestamp: string;
}
```

---

#### TOOL_CALL_UPDATE Event

**When:** Tool execution progress

**Payload:**

```typescript
interface ToolCallUpdate {
  type: CoderAgentEvent.TOOL_CALL_UPDATE;
  toolCallId: string;
  status: 'requested' | 'executing' | 'completed' | 'error';
  toolName: string;
  result?: string;
  error?: string;
}
```

---

#### TEXT_CONTENT Event

**When:** AI generates text response

**Payload:**

```typescript
interface TextContent {
  type: CoderAgentEvent.TEXT_CONTENT;
  text: string;
}
```

---

#### THOUGHT Event

**When:** AI internal reasoning (thinking models)

**Payload:**

```typescript
interface Thought {
  type: CoderAgentEvent.THOUGHT;
  thought: string;
}
```

---

#### THOUGHT_SUMMARY Event

**When:** Summary of extended thinking

**Payload:**

```typescript
interface ThoughtSummary {
  type: CoderAgentEvent.THOUGHT_SUMMARY;
  summary: string;
}
```

---

## Agent Executor

### CoderAgentExecutor Class

**Location:** `packages/a2a-server/src/agent/executor.ts`

**Purpose:** Manages multiple concurrent tasks

---

### Executor Architecture

```typescript
export class CoderAgentExecutor implements AgentExecutor {
  private tasks: Map<string, TaskWrapper>;
  private executingTasks: Set<string>;
  
  constructor(private taskStore?: TaskStore) {}
  
  // Core methods
  async getConfig(agentSettings, taskId): Promise<Config>
  async createTask(taskId, contextId, agentSettings): Promise<Task>
  async executeTask(task, userMessage, eventCallback): Promise<void>
  async getTask(taskId): Promise<SDKTask>
  async listTasks(): Promise<SDKTask[]>
}
```

---

### TaskWrapper

**Purpose:** Wraps `Task` with metadata

```typescript
interface TaskWrapper {
  task: Task;
  agentSettings: AgentSettings;
}
```

---

### Executor Responsibilities

1. **Task Management**
   - Create tasks
   - Track active tasks
   - Store completed tasks
   - List all tasks

2. **Configuration**
   - Load workspace settings
   - Load extensions
   - Initialize Config for each task

3. **Execution**
   - Execute task with user message
   - Stream events to client
   - Handle tool confirmations
   - Persist task state

4. **Persistence**
   - Save task state to TaskStore
   - Load task history
   - Restore previous sessions

---

### Multi-Task Execution

**Concurrency Model:**

```typescript
private executingTasks = new Set<string>();

async executeTask(task, userMessage, eventCallback) {
  const taskId = task.id;
  
  // Track executing task
  this.executingTasks.add(taskId);
  
  try {
    // Execute task
    await task.execute(userMessage, eventCallback);
  } finally {
    // Remove from executing set
    this.executingTasks.delete(taskId);
  }
}
```

**Behavior:**
- Multiple tasks can execute concurrently
- No explicit limit on concurrent tasks
- Each task has independent state

---

## HTTP API

### Server Setup

**Location:** `packages/a2a-server/src/http/server.ts`

**Port:** Configured via `CODER_AGENT_PORT` environment variable (default: `41242`)

**Framework:** Express.js

---

### Endpoints

**Note:** Endpoint details are tested in `endpoints.test.ts` but not fully documented. Based on test patterns:

#### POST `/tasks`

Create new task

**Request:**

```json
{
  "contextId": "context-123",
  "agentSettings": {
    "workspaceRoot": "/path/to/workspace",
    "model": "gemini-2.0-flash-exp"
  }
}
```

**Response:**

```json
{
  "taskId": "task-456",
  "state": "running"
}
```

---

#### POST `/tasks/:taskId/execute`

Execute task with user message

**Request:**

```json
{
  "message": "Write a Python function to sort a list"
}
```

**Response:** Server-Sent Events (SSE) stream

**Event Stream:**

```
event: text_content
data: {"type": "text_content", "text": "I'll write a sort function..."}

event: tool_call_update
data: {"type": "tool_call_update", "toolCallId": "call-1", "status": "requested", "toolName": "write_file"}

event: tool_call_update
data: {"type": "tool_call_update", "toolCallId": "call-1", "status": "completed", "result": "..."}

event: state_change
data: {"type": "state_change", "state": "completed", "timestamp": "2025-10-24T12:00:00Z"}
```

---

#### GET `/tasks/:taskId`

Get task details

**Response:**

```json
{
  "id": "task-456",
  "contextId": "context-123",
  "state": "completed",
  "timestamp": "2025-10-24T12:00:00Z",
  "metadata": {},
  "history": [...],
  "artifacts": [...]
}
```

---

#### GET `/tasks`

List all tasks

**Response:**

```json
{
  "tasks": [
    {
      "id": "task-456",
      "contextId": "context-123",
      "state": "completed"
    },
    ...
  ]
}
```

---

### Request Storage

**Location:** `packages/a2a-server/src/http/requestStorage.ts`

**Purpose:** AsyncLocalStorage for request-scoped data

**Usage:** Store request context across async operations

```typescript
export const requestStorage = new AsyncLocalStorage<{
  requestId: string;
  taskId?: string;
}>();
```

---

## Persistence Layer

### TaskStore Interface

**Location:** `packages/a2a-server/src/persistence/gcs.ts`

**Purpose:** Persist task state for recovery and history

---

### GCS Implementation

**Storage:** Google Cloud Storage

**Configuration:**

```typescript
interface TaskStoreConfig {
  bucket: string;              // GCS bucket name
  projectId: string;           // GCP project ID
  credentials?: string;        // Path to service account key (optional)
}
```

**Environment Variables:**

- `GOOGLE_CLOUD_PROJECT` - GCP project ID
- `GOOGLE_APPLICATION_CREDENTIALS` - Service account key path
- `A2A_GCS_BUCKET` - GCS bucket name (default: `gemini-cli-a2a-tasks`)

---

### Storage Schema

**Bucket Structure:**

```
<bucket>/
  tasks/
    <taskId>/
      metadata.json       # Task metadata
      history.json        # Conversation history
      state.json          # Current state
      artifacts/          # Generated files
```

---

### Persistence Operations

#### Save Task

```typescript
async saveTask(task: SDKTask): Promise<void> {
  const taskPath = `tasks/${task.id}`;
  
  // Save metadata
  await bucket.file(`${taskPath}/metadata.json`).save(
    JSON.stringify(task.metadata)
  );
  
  // Save history
  await bucket.file(`${taskPath}/history.json`).save(
    JSON.stringify(task.history)
  );
  
  // Save state
  await bucket.file(`${taskPath}/state.json`).save(
    JSON.stringify({
      state: task.state,
      timestamp: task.timestamp
    })
  );
}
```

---

#### Load Task

```typescript
async loadTask(taskId: string): Promise<SDKTask | null> {
  const taskPath = `tasks/${taskId}`;
  
  try {
    const metadata = await bucket.file(`${taskPath}/metadata.json`).download();
    const history = await bucket.file(`${taskPath}/history.json`).download();
    const state = await bucket.file(`${taskPath}/state.json`).download();
    
    return {
      id: taskId,
      metadata: JSON.parse(metadata[0].toString()),
      history: JSON.parse(history[0].toString()),
      state: JSON.parse(state[0].toString()).state,
      timestamp: JSON.parse(state[0].toString()).timestamp,
      artifacts: []
    };
  } catch (error) {
    return null;  // Task not found
  }
}
```

---

#### List Tasks

```typescript
async listTasks(): Promise<SDKTask[]> {
  const [files] = await bucket.getFiles({ prefix: 'tasks/' });
  
  const taskIds = new Set<string>();
  for (const file of files) {
    const parts = file.name.split('/');
    if (parts.length >= 2) {
      taskIds.add(parts[1]);  // tasks/<taskId>/...
    }
  }
  
  const tasks: SDKTask[] = [];
  for (const taskId of taskIds) {
    const task = await this.loadTask(taskId);
    if (task) tasks.push(task);
  }
  
  return tasks;
}
```

---

## Configuration

### Agent Settings

**Location:** `packages/a2a-server/src/config/settings.ts`

**Purpose:** Load configuration for each task

```typescript
interface AgentSettings {
  workspaceRoot: string;     // Workspace directory
  model?: string;            // Gemini model (optional)
  apiKey?: string;           // API key (optional)
  // Other settings from regular Gemini CLI
}
```

---

### Configuration Loading

```typescript
async function loadConfig(
  settings: Settings,
  extensions: Extension[],
  taskId: string
): Promise<Config> {
  const config = await loadConfigFromSettings(settings);
  
  // Load extensions for this task
  for (const extension of extensions) {
    await loadExtension(config, extension);
  }
  
  // Set task-specific context
  config.setTaskId(taskId);
  
  return config;
}
```

---

### Settings Scope

**Per-Task Configuration:**

Each task gets its own `Config` instance:

```typescript
const config = await this.getConfig(agentSettings, taskId);
```

**Shared Configuration:**

- Workspace settings (from `.gemini/config.yaml`)
- User settings (from `~/.config/gemini-cli/config.yaml`)
- Extensions (installed globally)

**Isolated State:**

- Tool registry
- File system service
- Gemini client session

---

## Command Registry

### Purpose

Register custom commands for A2A agent

**Location:** `packages/a2a-server/src/commands/command-registry.ts`

---

### Command Interface

```typescript
interface AgentCommand {
  name: string;
  description: string;
  execute(context: CommandContext, args: string[]): Promise<CommandResult>;
}
```

---

### Built-in Commands

#### `list-extensions`

**Location:** `packages/a2a-server/src/commands/list-extensions.ts`

**Purpose:** List installed extensions

**Usage:**

```typescript
const result = await commandRegistry.execute('list-extensions', []);
```

**Response:**

```json
{
  "extensions": [
    {
      "name": "my-extension",
      "version": "1.0.0",
      "enabled": true
    }
  ]
}
```

---

### Registering Custom Commands

```typescript
const commandRegistry = new CommandRegistry();

commandRegistry.register({
  name: 'my-command',
  description: 'Custom command',
  execute: async (context, args) => {
    // Implementation
    return { success: true, output: 'Command executed' };
  }
});
```

---

## Event System

### Event Bus

**Purpose:** Stream task events to client via HTTP

**Interface:**

```typescript
interface ExecutionEventBus {
  emit(event: CoderAgentMessage): void;
  on(eventType: CoderAgentEvent, handler: (data: any) => void): void;
  off(eventType: CoderAgentEvent, handler: (data: any) => void): void;
}
```

---

### Event Flow

```
Task.execute()
    ↓
Emit Event (state_change, tool_call_update, text_content)
    ↓
EventBus.emit()
    ↓
HTTP SSE Stream
    ↓
Client Receives Event
```

---

### Event Handler Example

```typescript
const eventBus = new EventBus();

eventBus.on(CoderAgentEvent.TEXT_CONTENT, (data) => {
  console.log('AI Response:', data.text);
});

eventBus.on(CoderAgentEvent.TOOL_CALL_UPDATE, (data) => {
  console.log('Tool Call:', data.toolName, data.status);
});

await task.execute(userMessage, (event) => {
  eventBus.emit(event);
});
```

---

## Deployment

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `CODER_AGENT_PORT` | HTTP server port | `41242` |
| `GOOGLE_CLOUD_PROJECT` | GCP project ID (for GCS) | - |
| `GOOGLE_APPLICATION_CREDENTIALS` | Service account key | - |
| `A2A_GCS_BUCKET` | GCS bucket name | `gemini-cli-a2a-tasks` |
| `GEMINI_API_KEY` | Gemini API key | - |
| `NODE_ENV` | Node environment | `production` |

---

### Running A2A Server

#### Local Development

```bash
cd packages/a2a-server
npm install
npm run build

export CODER_AGENT_PORT=8080
export GEMINI_API_KEY=your-api-key
npm start
```

---

#### Docker Deployment

```dockerfile
FROM node:20

WORKDIR /app
COPY packages/a2a-server ./packages/a2a-server
COPY packages/core ./packages/core

RUN cd packages/a2a-server && npm install && npm run build

ENV CODER_AGENT_PORT=8080
EXPOSE 8080

CMD ["node", "packages/a2a-server/dist/index.js"]
```

---

#### Cloud Run (GCP)

```bash
# Build and push container
gcloud builds submit --tag gcr.io/PROJECT_ID/a2a-server

# Deploy to Cloud Run
gcloud run deploy a2a-server \
  --image gcr.io/PROJECT_ID/a2a-server \
  --platform managed \
  --region us-central1 \
  --set-env-vars GEMINI_API_KEY=your-api-key,GOOGLE_CLOUD_PROJECT=PROJECT_ID
```

---

### Health Check Endpoint

**Suggested Implementation:**

```typescript
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    uptime: process.uptime(),
    activeTasks: executor.getExecutingTaskCount(),
    totalTasks: executor.getTotalTaskCount()
  });
});
```

---

## A2A Gotchas & Limitations

### 1. Experimental Status

**Issue:** All A2A code is experimental, APIs may change

**Recommendation:** Pin versions, avoid production use without thorough testing

---

### 2. No Built-in Authentication

**Issue:** HTTP API has no authentication/authorization

**Mitigation:** 
- Deploy behind API Gateway with authentication
- Use Cloud IAP (Identity-Aware Proxy)
- Implement custom auth middleware

---

### 3. No Rate Limiting

**Issue:** No built-in rate limiting or quota management

**Mitigation:**
- Use API Gateway rate limits
- Implement custom rate limiting middleware
- Monitor API usage

---

### 4. Concurrent Task Limit

**Issue:** No explicit limit on concurrent tasks

**Effect:** Unbounded task execution could exhaust resources

**Mitigation:**
- Implement max concurrent tasks in executor
- Use queue system for task scheduling
- Monitor resource usage

---

### 5. GCS Persistence Only

**Issue:** Only GCS supported for persistence

**Alternatives:** 
- Implement custom `TaskStore` for other storage (S3, Azure Blob, PostgreSQL)
- Use in-memory storage (no persistence)

---

### 6. Tool Confirmation Handling

**Issue:** A2A server needs to handle tool confirmations remotely

**Current State:** Not fully implemented

**Workaround:** Configure `tools.autoAccept: true` (not recommended for production)

---

### 7. Session Management

**Issue:** No built-in session management or cleanup

**Effect:** Tasks persist indefinitely in memory

**Recommendation:**
- Implement task TTL (time-to-live)
- Periodic cleanup of completed tasks
- Persist to storage and evict from memory

---

### 8. Error Recovery

**Issue:** Limited error recovery for failed tasks

**Recommendation:**
- Implement retry logic
- Save checkpoints during execution
- Graceful degradation on errors

---

This comprehensive reference documents the A2A server architecture, APIs, and deployment patterns for the experimental A2A server package.

