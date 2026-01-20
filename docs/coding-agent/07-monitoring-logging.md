# Monitoring and Logging

This document covers debugging, monitoring, and troubleshooting mechanisms in pi-coding-agent.

## Table of Contents

- [Debug Logging](#debug-logging)
- [Event Bus and Session Events](#event-bus-and-session-events)
- [Debugging Authentication Issues](#debugging-authentication-issues)
- [Debugging Terminal Issues](#debugging-terminal-issues)
- [Debugging Extension Issues](#debugging-extension-issues)
- [Context Usage and Token Monitoring](#context-usage-and-token-monitoring)
- [Troubleshooting Common Problems](#troubleshooting-common-problems)
- [Error Types and Patterns](#error-types-and-patterns)
- [Session Inspection](#session-inspection)

## Debug Logging

### Debug Log File

Pi writes debug logs to `~/.pi/agent/pi-coding-agent-debug.log`. This file contains internal error messages and diagnostic information.

**Location function:**

```typescript
// src/config.ts
export function getDebugLogPath(): string {
  return join(getAgentDir(), `${APP_NAME}-debug.log`);
}
```

Debug logs capture errors that occur during:
- Extension loading failures
- Tool execution errors
- Session file I/O issues
- Model API errors
- Internal runtime exceptions

### Console Logging

The codebase uses `console.error()` for error reporting throughout the application. Key logging locations:

- **Event bus errors:** When extension event handlers throw errors (src/core/event-bus.ts)
- **Extension runner errors:** When extension lifecycle events fail (src/core/extensions/runner.ts)
- **Model resolution errors:** When API keys are missing or invalid (src/core/model-resolver.ts)
- **File processing errors:** When reading or processing input files fails (src/cli/file-processor.ts)

Example from event bus:

```typescript
const safeHandler = async (data: unknown) => {
  try {
    await handler(data);
  } catch (err) {
    console.error(`Event handler error (${channel}):`, err);
  }
};
```

**Example debug output:**
```
Event handler error (tool_call): Error: Failed to validate tool parameters
    at validateToolInput (src/core/extensions/wrapper.ts:45)
    at async toolExecute (src/core/extensions/wrapper.ts:78)
```

## Event Bus and Session Events

### Event Bus Architecture

The event bus provides a pub/sub mechanism for monitoring session lifecycle. Located in `src/core/event-bus.ts`:

```typescript
export interface EventBus {
  emit(channel: string, data: unknown): void;
  on(channel: string, handler: (data: unknown) => void): () => void;
}
```

**Key features:**
- Asynchronous event handlers with error isolation
- Event handlers that throw errors are caught and logged without affecting other handlers
- Cleanup via returned unsubscribe function
- Extensions use this to react to session changes

### Session Event Types

The following session events are emitted during the agent lifecycle (defined in `src/core/extensions/types.ts`):

#### Lifecycle Events

| Event | When Fired | Purpose |
|-------|-----------|---------|
| `session_start` | Initial session load | Notify extensions of new session |
| `session_shutdown` | Exit (Ctrl+C, Ctrl+D, SIGTERM) | Cleanup and state persistence |
| `session_before_switch` | Before `/new` or `/resume` | Allow cancellation of session switch |
| `session_switch` | After session switch | React to new session context |
| `session_before_fork` | Before `/fork` | Allow cancellation or modification |
| `session_fork` | After fork completes | React to new forked session |
| `session_before_compact` | Before compaction | Customize or cancel compaction |
| `session_compact` | After compaction completes | React to compacted context |
| `session_before_tree` | Before `/tree` navigation | Customize or cancel tree navigation |
| `session_tree` | After tree navigation | React to new branch position |

#### Agent Events

| Event | When Fired | Purpose |
|-------|-----------|---------|
| `before_agent_start` | After user prompt, before LLM call | Inject messages or modify system prompt |
| `agent_start` | Start of agent processing | Track agent activity |
| `agent_end` | Agent finishes processing | Monitor completion |
| `turn_start` | Start of each LLM turn | Track turn-level activity |
| `turn_end` | End of each LLM turn | Collect turn results |
| `context` | Before each LLM call | Modify message context |

#### Tool Events

| Event | When Fired | Purpose |
|-------|-----------|---------|
| `tool_call` | Before tool execution | Block or monitor tool usage |
| `tool_result` | After tool execution | Modify or inspect results |

#### Input Events

| Event | When Fired | Purpose |
|-------|-----------|---------|
| `input` | User submits input | Transform or intercept input |
| `user_bash` | User executes `!` or `!!` | Intercept bash commands |

#### Model Events

| Event | When Fired | Purpose |
|-------|-----------|---------|
| `model_select` | Model changes (manual or restore) | React to model switches |

### Monitoring Example

Extensions can subscribe to events to monitor session activity:

```typescript
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    console.log(`Tool called: ${event.toolName}`, event.input);
  });

  pi.on("session_compact", async (event, ctx) => {
    console.log(`Compaction: ${event.compactionEntry.tokensBefore} tokens`);
  });
}
```

## Debugging Authentication Issues

### API Key Resolution Order

API keys are resolved in the following priority order (src/core/auth-storage.ts):

1. **Runtime overrides:** Set via `setRuntimeApiKey()` (highest priority)
2. **Auth file credentials:** From `~/.pi/agent/auth.json`
3. **Environment variables:** `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.
4. **Fallback resolver:** Custom fallback function (if configured)

### OAuth Authentication

OAuth credentials are stored in `auth.json` with the following structure:

```json
{
  "anthropic": {
    "type": "oauth",
    "access": "access_token_here",
    "refresh": "refresh_token_here",
    "expires": 1234567890000
  }
}
```

**Token refresh:** OAuth tokens are automatically refreshed when expired. File locking ensures safe concurrent access during refresh operations.

### Common Authentication Errors

**Port 1455 in use:**
- **Cause:** OAuth callback server cannot bind to port
- **Solution:** Close conflicting process or paste auth code/URL when prompted

**Token expired / refresh failed:**
- **Cause:** OAuth refresh token is invalid or expired
- **Solution:** Run `/login` again for the provider

**Usage limits (429):**
- **Cause:** Rate limit exceeded
- **Solution:** Wait for reset window (pi displays retry time)

**No API key error:**
- **Cause:** No valid credentials found for selected model
- **Solution:** Set environment variable, add to auth.json, or run `/login`

### Checking Authentication Status

```typescript
// Check if provider has authentication
const hasAuth = authStorage.hasAuth("anthropic");

// Get API key (with refresh for OAuth)
const apiKey = await authStorage.getApiKey("anthropic");

// Check if using OAuth (vs API key)
const isOAuth = authStorage.getOAuthCredential("anthropic");
```

## Debugging Terminal Issues

### Keyboard Support

Pi requires terminal emulators that support the Kitty keyboard protocol for features like `Shift+Enter` (multiline input).

**Supported terminals:**
- kitty
- wezterm
- iTerm2 (experimental)
- ghostty

**Workarounds for unsupported terminals:**

VS Code Integrated Terminal:
```json
{
  "key": "shift+enter",
  "command": "workbench.action.terminal.sendSequence",
  "args": { "text": "\\u001b[13;2u" },
  "when": "terminalFocus"
}
```

Windows Terminal:
```json
{
  "actions": [{
    "command": { "action": "sendInput", "input": "\\u001b[13;2u" },
    "keys": "shift+enter"
  }]
}
```

### Shell Configuration

**Windows:** Pi requires bash. Checked locations (in order):
1. Custom path from `~/.pi/agent/settings.json`
2. Git Bash (`C:\Program Files\Git\bin\bash.exe`)
3. `bash.exe` on PATH (Cygwin, MSYS2, WSL)

**Alias expansion:** Pi runs bash in non-interactive mode (`bash -c`), which doesn't expand aliases by default. Enable via settings:

```json
{
  "shellCommandPrefix": "shopt -s expand_aliases\neval \"$(grep '^alias ' ~/.zshrc)\""
}
```

## Debugging Extension Issues

### Extension Loading

Extensions are loaded via jiti (Just-In-Time TypeScript compilation). Loading errors are captured and returned in the `LoadExtensionsResult`:

```typescript
interface LoadExtensionsResult {
  extensions: Extension[];
  errors: Array<{ path: string; error: string }>;
  runtime: ExtensionRuntime;
}
```

**Common loading errors:**

1. **Module not found:**
   - **Cause:** Extension imports a missing npm package
   - **Solution:** Add `package.json` next to extension and run `npm install`

2. **Extension does not export a valid factory function:**
   - **Cause:** Missing or incorrect `export default function(pi: ExtensionAPI)`
   - **Solution:** Ensure extension exports a default function

3. **Failed to load extension: [error]:**
   - **Cause:** Runtime error during extension initialization
   - **Solution:** Check extension code for syntax errors or uncaught exceptions

### Extension Error Handling

Extension event handlers are wrapped with error handling to prevent one extension from breaking others:

```typescript
try {
  await handler(data);
} catch (err) {
  console.error(`Event handler error (${channel}):`, err);
}
```

**Tool execution errors:**
- Tool `execute()` errors are caught and returned to the LLM with `isError: true`
- The LLM sees the error message and can react accordingly
- Extension tools should catch and return structured errors rather than throwing

**Event handler errors:**
- Logged to console but don't crash the application
- Other extensions continue to receive events
- `tool_call` handler errors result in tool blocking (fail-safe)

### Extension Runtime Not Initialized

Extensions that call action methods during loading will see this error:

```typescript
const notInitialized = () => {
  throw new Error("Extension runtime not initialized. Action methods cannot be called during extension loading.");
};
```

**Solution:** Only call action methods (`sendMessage`, `setSessionName`, etc.) from event handlers, not at the module level during loading.

## Context Usage and Token Monitoring

### Context Usage API

Extensions can monitor token usage via `ctx.getContextUsage()`:

```typescript
interface ContextUsage {
  tokens: number;      // Total context tokens used
  contextWindow: number;  // Model's max context window
  percentage: number;  // Usage as percentage (0-100)
}
```

This returns the most recent context usage from the last LLM call, combining:
- **Last assistant message usage:** If available (most accurate)
- **Estimated tokens:** For messages after the last assistant response

### Session Statistics

Session statistics are available through the session manager:

```typescript
const entries = ctx.sessionManager.getEntries();
const messageCount = entries.filter(e => e.type === "message").length;
const compactionCount = entries.filter(e => e.type === "compaction").length;
```

### Compaction Monitoring

Track compaction events to understand context management:

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;
  console.log(`Compaction triggered:`);
  console.log(`  Tokens before: ${preparation.tokensBefore}`);
  console.log(`  Messages to summarize: ${preparation.messagesToSummarize.length}`);
  console.log(`  First kept entry: ${preparation.firstKeptEntryId}`);
});

pi.on("session_compact", async (event, ctx) => {
  console.log(`Compaction completed`);
  console.log(`  Summary length: ${event.compactionEntry.summary.length} chars`);
  console.log(`  From extension: ${event.fromExtension}`);
});
```

## Troubleshooting Common Problems

### Session File Issues

**Session not saving:**
- Check `~/.pi/agent/sessions/<project-path>/` directory permissions
- Verify no file locking issues (session files use JSONL append-only writes)
- Use `--no-session` to test without persistence

**Example error:**
```
Error: EACCES: permission denied, open '/Users/example/.pi/agent/sessions/--path--to--project/2026-01-20.jsonl'
```

**Solution:**
```bash
chmod -R 755 ~/.pi/agent/sessions/
```

**Session corruption:**
- Each line must be valid JSON
- Use a JSON validator on individual lines
- Session version mismatches are auto-migrated (v1 → v2 → v3)

**Example validation:**
```bash
# Check for invalid JSON lines
cat session.jsonl | while read line; do echo "$line" | jq . > /dev/null || echo "Invalid JSON"; done

# Validate session structure
cat session.jsonl | head -1 | jq '.type == "session"'
```

### Tool Execution Issues

**Bash tool timeout:**
- Default timeout is 120 seconds (configurable via tool options)
- Use `signal` parameter to check for cancellation
- Large output is truncated (50KB/2000 lines default)

**Example timeout scenario:**
```
Tool: bash
Input: { command: "npm install" }
Result: Command timed out after 120000ms
Exit code: undefined
```

**Solution:** Increase timeout or run command in background with tmux

**File operations failing:**
- Check working directory with `/settings`
- Verify file paths are relative to cwd or absolute
- Check file permissions

**Example debug scenario:**
```
Tool: read
Input: { path: "src/config.ts" }
Error: ENOENT: no such file or directory

Debug steps:
1. Check cwd: /session shows working directory
2. Verify path: ls src/ to list files
3. Check spelling: grep -r "config" to find similar files
```

### Model and Provider Issues

**Model not found:**
- Check model ID matches exactly (case-sensitive)
- Verify provider is available: `ctx.modelRegistry.getAll()`
- Ensure API key is configured for provider

**Example error:**
```
Error: Model not found: claude-3-sonnet-20240229
Available models: claude-3-5-sonnet-20241022, claude-opus-4-20250514

Hint: Check model ID spelling or use /model to select from available models
```

**Context overflow:**
- Enable auto-compaction in settings
- Manually compact with `/compact`
- Check context usage with `ctx.getContextUsage()`
- Reduce `keepRecentTokens` setting for more aggressive compaction

**Example overflow scenario:**
```
Warning: Context usage at 95% (191,456 / 200,000 tokens)
Recommendation: Run /compact to summarize older messages

After compaction:
Context usage: 45% (90,000 / 200,000 tokens)
Summary: Previous conversation covered API design and database schema
```

### Performance Issues

**Slow extension loading:**
- Extensions with heavy dependencies take time to load via jiti
- Consider pre-compiling extensions with npm packages
- Check for blocking I/O during extension initialization

**Example diagnosis:**
```bash
# Time extension loading
time pi --extension ./my-extension.ts --print "test"

# Check extension dependencies
ls -lh my-extension/node_modules/

# Profile with Node.js
node --prof dist/cli.js --extension ./my-extension.ts
node --prof-process isolate-*.log > profile.txt
```

**Memory usage:**
- Session files are loaded entirely into memory
- Very large sessions (>100MB JSONL) may cause issues
- Use compaction to reduce session size

**Memory profiling:**
```bash
# Check session file size
du -h ~/.pi/agent/sessions/*/2026-*.jsonl

# Monitor memory during execution
node --max-old-space-size=4096 dist/cli.js

# Profile heap usage
node --heap-prof dist/cli.js
```

## Error Types and Patterns

### Custom Error Classes

Pi uses standard Error classes with descriptive messages rather than custom error types.

Common error patterns:

```typescript
// File not found
throw new Error(`File not found: ${path}`);

// Invalid configuration
throw new Error(`Invalid setting: ${key}`);

// API errors
throw new Error(`Failed to call ${provider} API: ${statusCode} ${statusText}`);

// Tool blocked
return { block: true, reason: "Not allowed" };
```

### Error Handling Patterns

**Graceful degradation:**
```typescript
try {
  const result = await riskyOperation();
  return result;
} catch (err) {
  console.error("Operation failed:", err);
  return fallbackValue;
}
```

**User-facing errors:**
```typescript
if (!apiKey) {
  ctx.ui.notify("No API key configured", "error");
  return;
}
```

**Tool result errors:**
```typescript
return {
  content: [{ type: "text", text: `Error: ${err.message}` }],
  details: { error: err.message },
  isError: true
};
```

## Session Inspection

### Viewing Session Files

Session files are JSONL (JSON Lines) format at `~/.pi/agent/sessions/<project>/`:

```bash
# View session entries
cat ~/.pi/agent/sessions/--path--to--project/2024-*.jsonl

# Count entries by type
cat session.jsonl | jq -r '.type' | sort | uniq -c

# Extract all user messages
cat session.jsonl | jq 'select(.type=="message" and .message.role=="user")'

# Show compaction summaries
cat session.jsonl | jq 'select(.type=="compaction") | .summary'
```

### Programmatic Session Access

```typescript
// Get current session entries
const entries = ctx.sessionManager.getEntries();

// Walk current branch (from leaf to root)
const branch = ctx.sessionManager.getBranch();

// Get specific entry by ID
const entry = ctx.sessionManager.getEntry("a1b2c3d4");

// Get tree structure
const tree = ctx.sessionManager.getTree();

// Get session header (metadata)
const header = ctx.sessionManager.getHeader();
```

### Tree Navigation Inspection

```typescript
// Get current leaf position
const leafId = ctx.sessionManager.getLeafId();

// Get children of an entry (for branching analysis)
const children = ctx.sessionManager.getChildren(parentId);

// Get labels (bookmarks)
const label = ctx.sessionManager.getLabel(entryId);
```

## Real Debug Output Examples

This section shows actual log formats from the codebase to help with debugging.

### Session Lifecycle Logs

**Session start (interactive mode):**
```
✓ New session started
```

**Extension loading:**
```
✓ chalk-logger extension loaded
```

**Event handler errors:**
```
Event handler error (tool_call): Error: Failed to validate tool parameters
    at validateToolInput (src/core/extensions/wrapper.ts:45)
    at async toolExecute (src/core/extensions/wrapper.ts:78)
```

### Tool Execution Logs

**Tool call example (from chalk-logger extension):**
```
[chalk-logger] Tool: bash
[chalk-logger] Tool: read
[chalk-logger] Tool: edit
```

**Tool result tracking:**
```typescript
pi.on("tool_result", async (event, ctx) => {
  console.log(`Tool ${event.toolName} completed`);
  console.log(`Result:`, event.result);
});
```

### Model API Call Logs

**Model selection tracking:**
```
[model_select] claude-3-5-sonnet-20241022 → claude-opus-4-20250514 (manual)
[model_select] claude-opus-4-20250514 → claude-3-5-sonnet-20241022 (restore)
```

**Model change with status update:**
```typescript
pi.on("model_select", async (event, ctx) => {
  const model = event.model;
  const prev = event.previousModelId || "none";
  const next = model.id;
  const source = event.source; // "manual", "restore", "cycle"

  console.log(`[model_select] ${prev} → ${next} (${source})`);
});
```

### Extension Event Logs

**Complete monitoring extension example:**
```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import chalk from "chalk";

export default function (pi: ExtensionAPI) {
  console.log(`${chalk.green("✓")} ${chalk.bold("chalk-logger extension loaded")}`);

  pi.on("agent_start", async () => {
    console.log(`${chalk.blue("[chalk-logger]")} Agent starting`);
  });

  pi.on("tool_call", async (event) => {
    console.log(`${chalk.yellow("[chalk-logger]")} Tool: ${chalk.cyan(event.toolName)}`);
    return undefined;
  });

  pi.on("agent_end", async (event) => {
    const count = event.messages.length;
    console.log(`${chalk.green("[chalk-logger]")} Done with ${chalk.bold(String(count))} messages`);
  });
}
```

This extension outputs:
```
✓ chalk-logger extension loaded
[chalk-logger] Agent starting
[chalk-logger] Tool: bash
[chalk-logger] Tool: read
[chalk-logger] Done with 5 messages
```

## Monitoring Dashboard Concept

You can build real-time monitoring dashboards using RPC mode. The agent streams events as JSON lines to stdout, allowing external tools to track performance and usage.

### RPC Event Stream

**Starting the agent in RPC mode:**
```bash
pi --mode rpc --provider anthropic --model claude-sonnet-4-20250514
```

**Protocol overview:**
- Commands sent to stdin as JSON lines
- Responses and events streamed to stdout as JSON lines
- All events are real-time, allowing live monitoring

### Subscribing to Events

RPC mode streams all agent events automatically. Parse stdout to track:

**Agent lifecycle events:**
```json
{"type": "agent_start"}
{"type": "turn_start", "turnIndex": 0, "timestamp": 1705842000000}
{"type": "turn_end", "turnIndex": 0, "message": {...}, "toolResults": [...]}
{"type": "agent_end", "messages": [...]}
```

**Tool execution events:**
```json
{"type": "tool_call_start", "toolName": "bash", "toolCallId": "toolu_123"}
{"type": "tool_call_end", "toolName": "bash", "toolCallId": "toolu_123"}
```

**Message events:**
```json
{"type": "message_start", "message": {"role": "user", "content": [...]}}
{"type": "message_delta", "delta": {...}}
{"type": "message_end", "message": {"role": "assistant", "content": [...]}}
```

### Tracking Token Usage

**Get current state (includes usage):**
```json
{"type": "get_state"}
```

**Response includes model and context info:**
```json
{
  "type": "response",
  "command": "get_state",
  "success": true,
  "data": {
    "model": {
      "id": "claude-sonnet-4-20250514",
      "contextWindow": 200000,
      "maxTokens": 8192
    },
    "messageCount": 15,
    "isStreaming": false
  }
}
```

**Track usage from assistant messages:**

Assistant messages include usage data in their `usage` field:
```typescript
interface AssistantUsage {
  input: number;        // Input tokens used
  output: number;       // Output tokens generated
  cacheRead: number;    // Cache read tokens (prompt caching)
  cacheWrite: number;   // Cache write tokens
  totalTokens?: number; // Total tokens (input + output)
}
```

**Example monitoring code:**
```typescript
let totalInput = 0;
let totalOutput = 0;
let totalCacheRead = 0;
let totalCacheWrite = 0;

// Parse each event line
process.on('message_end', (event) => {
  if (event.message.role === 'assistant' && event.message.usage) {
    const usage = event.message.usage;
    totalInput += usage.input || 0;
    totalOutput += usage.output || 0;
    totalCacheRead += usage.cacheRead || 0;
    totalCacheWrite += usage.cacheWrite || 0;

    console.log(`Total usage: ${totalInput + totalOutput} tokens`);
    console.log(`Cache efficiency: ${totalCacheRead} read, ${totalCacheWrite} written`);
  }
});
```

### Measuring Response Times

**Track turn duration:**
```typescript
let turnStartTime: number;

process.on('turn_start', (event) => {
  turnStartTime = event.timestamp;
});

process.on('turn_end', (event) => {
  const duration = Date.now() - turnStartTime;
  console.log(`Turn ${event.turnIndex} completed in ${duration}ms`);
});
```

**Track agent run duration:**
```typescript
let agentStartTime: number;

process.on('agent_start', () => {
  agentStartTime = Date.now();
});

process.on('agent_end', (event) => {
  const duration = Date.now() - agentStartTime;
  const messageCount = event.messages.length;
  console.log(`Agent completed ${messageCount} messages in ${duration}ms`);
  console.log(`Average: ${(duration / messageCount).toFixed(0)}ms per message`);
});
```

### Building a Monitoring Dashboard

**Example Node.js monitoring script:**
```typescript
import { spawn } from 'child_process';
import * as readline from 'readline';

interface MonitoringStats {
  totalTokens: number;
  totalCost: number;
  messageCount: number;
  toolCallCount: number;
  avgResponseTime: number;
  contextUsage: number;
}

class AgentMonitor {
  private stats: MonitoringStats = {
    totalTokens: 0,
    totalCost: 0,
    messageCount: 0,
    toolCallCount: 0,
    avgResponseTime: 0,
    contextUsage: 0,
  };

  start() {
    const agent = spawn('pi', ['--mode', 'rpc', '--provider', 'anthropic']);

    const rl = readline.createInterface({
      input: agent.stdout,
      crlfDelay: Infinity,
    });

    rl.on('line', (line) => {
      try {
        const event = JSON.parse(line);
        this.handleEvent(event);
      } catch (err) {
        console.error('Failed to parse event:', err);
      }
    });

    // Send initial prompt
    agent.stdin.write(JSON.stringify({
      type: 'prompt',
      message: 'Hello, please help me debug this issue',
    }) + '\n');
  }

  private handleEvent(event: any) {
    switch (event.type) {
      case 'message_end':
        if (event.message.role === 'assistant') {
          this.stats.messageCount++;
          if (event.message.usage) {
            this.stats.totalTokens += event.message.usage.totalTokens || 0;
          }
        }
        break;

      case 'tool_call_start':
        this.stats.toolCallCount++;
        break;

      case 'turn_end':
        // Update dashboard display
        this.updateDashboard();
        break;
    }
  }

  private updateDashboard() {
    console.clear();
    console.log('=== Agent Monitoring Dashboard ===');
    console.log(`Messages: ${this.stats.messageCount}`);
    console.log(`Tool Calls: ${this.stats.toolCallCount}`);
    console.log(`Total Tokens: ${this.stats.totalTokens.toLocaleString()}`);
    console.log(`Context Usage: ${this.stats.contextUsage}%`);
  }
}

new AgentMonitor().start();
```

## Performance Metrics

### Tracking Context Usage

**Get context usage in extensions:**
```typescript
pi.on("turn_end", async (_event, ctx) => {
  const usage = ctx.getContextUsage();

  if (usage) {
    console.log(`Context: ${usage.tokens}/${usage.contextWindow} tokens (${usage.percent.toFixed(1)}%)`);
    console.log(`From usage: ${usage.usageTokens}, estimated: ${usage.trailingTokens}`);
  }
});
```

**Context usage interface:**
```typescript
interface ContextUsage {
  tokens: number;         // Total context tokens used
  contextWindow: number;  // Model's max context window
  percent: number;        // Usage as percentage (0-100)
  usageTokens: number;    // Tokens from last assistant usage
  trailingTokens: number; // Estimated tokens for messages after last usage
}
```

**Context calculation logic:**

The agent uses the most recent assistant message usage when available, then estimates tokens for any messages added after that point. This provides accurate context tracking without calling the tokenization API for every message.

```typescript
// From src/core/compaction/compaction.ts
export function estimateContextTokens(messages: AgentMessage[]): ContextUsageEstimate {
  const lastAssistantIndex = findLastAssistantWithUsage(messages);

  if (lastAssistantIndex >= 0) {
    const lastAssistant = messages[lastAssistantIndex] as AssistantMessage;
    const usageTokens = calculateContextTokens(lastAssistant.usage);

    // Estimate tokens for messages after the last assistant response
    const trailingMessages = messages.slice(lastAssistantIndex + 1);
    const trailingTokens = estimateTokens(trailingMessages);

    return {
      tokens: usageTokens + trailingTokens,
      usageTokens,
      trailingTokens,
    };
  }

  // No assistant usage available - estimate all messages
  return {
    tokens: estimateTokens(messages),
    usageTokens: 0,
    trailingTokens: estimateTokens(messages),
  };
}
```

### Session Size Monitoring

**Track session growth:**
```typescript
pi.on("turn_end", async (_event, ctx) => {
  const entries = ctx.sessionManager.getEntries();
  const messages = entries.filter(e => e.type === "message");
  const compactions = entries.filter(e => e.type === "compaction");

  console.log(`Session size: ${entries.length} entries`);
  console.log(`Messages: ${messages.length}, Compactions: ${compactions.length}`);
});
```

**Get session statistics:**
```typescript
const entries = ctx.sessionManager.getEntries();
const stats = {
  userMessages: entries.filter(e =>
    e.type === "message" && e.message.role === "user"
  ).length,
  assistantMessages: entries.filter(e =>
    e.type === "message" && e.message.role === "assistant"
  ).length,
  toolResults: entries.filter(e =>
    e.type === "message" && e.message.role === "toolResult"
  ).length,
  compactions: entries.filter(e => e.type === "compaction").length,
};
```

**Monitor session file size:**
```bash
# Check session file size
du -h ~/.pi/agent/sessions/*/2026-*.jsonl

# Watch session growth in real-time
watch -n 5 'du -h ~/.pi/agent/sessions/*/2026-01-20.jsonl'
```

### Compaction Triggers

**Monitor when compaction would trigger:**
```typescript
pi.on("turn_end", async (_event, ctx) => {
  const usage = ctx.getContextUsage();
  if (!usage) return;

  // Get compaction settings (from settings.json or defaults)
  const reserveTokens = 40000; // Default reserve
  const threshold = usage.contextWindow - reserveTokens;

  if (usage.tokens > threshold) {
    console.warn(`Compaction threshold reached: ${usage.tokens}/${threshold}`);
    console.warn(`Consider running /compact or enable auto-compaction`);
  }
});
```

**Automatic compaction trigger:**
```typescript
// From src/core/compaction/compaction.ts
export function shouldCompact(
  contextTokens: number,
  contextWindow: number,
  settings: CompactionSettings
): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

**Example: Trigger compaction at 80% usage:**
```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

const COMPACT_THRESHOLD_TOKENS = 100000; // 80% of 128K context

export default function (pi: ExtensionAPI) {
  pi.on("turn_end", async (_event, ctx) => {
    const usage = ctx.getContextUsage();

    if (!usage || usage.tokens <= COMPACT_THRESHOLD_TOKENS) {
      return;
    }

    console.log(`Auto-compacting: ${usage.tokens} tokens`);
    await ctx.compact();
  });
}
```

## Health Checks

### Verifying API Connectivity

**Check if model is available:**
```typescript
const model = ctx.model;
if (!model) {
  console.error("No model selected");
  return;
}

console.log(`Current model: ${model.id}`);
console.log(`Provider: ${model.provider}`);
console.log(`Context window: ${model.contextWindow}`);
console.log(`Max output: ${model.maxTokens}`);
```

**List available models:**
```typescript
// Get models that have authentication configured
const availableModels = await ctx.modelRegistry.getAvailable();

console.log(`Available models: ${availableModels.length}`);
availableModels.forEach(model => {
  console.log(`- ${model.provider}/${model.id} (${model.contextWindow} tokens)`);
});
```

**Test model resolution:**
```typescript
// Try to resolve a model pattern
const models = ctx.modelRegistry.getAll();
const pattern = "sonnet";

const matches = models.filter(m =>
  m.id.toLowerCase().includes(pattern.toLowerCase()) ||
  m.name?.toLowerCase().includes(pattern.toLowerCase())
);

if (matches.length === 0) {
  console.error(`No models match pattern: ${pattern}`);
} else {
  console.log(`Found ${matches.length} matches:`);
  matches.forEach(m => console.log(`- ${m.id}`));
}
```

### Extension Load Status

**Check which extensions loaded successfully:**

Extensions that fail to load produce errors captured during initialization. The CLI displays these on startup:

```
Failed to load extension: /path/to/extension.ts
Error: Cannot find module 'missing-package'
```

**Track loaded extensions:**
```typescript
// Extensions can announce themselves on load
export default function (pi: ExtensionAPI) {
  console.log("✓ my-extension loaded successfully");

  // Verify dependencies
  try {
    require('some-dependency');
    console.log("✓ Dependencies available");
  } catch (err) {
    console.error("✗ Missing dependency:", err.message);
  }
}
```

**Runtime extension verification:**
```typescript
// Test if extension features work
pi.on("session_start", async (_event, ctx) => {
  try {
    // Try extension-specific operations
    const result = await ctx.someExtensionMethod();
    console.log("✓ Extension operational");
  } catch (err) {
    console.error("✗ Extension error:", err);
  }
});
```

### Model Availability

**Check authentication status:**

The agent checks for API keys in this order:
1. Runtime overrides (set via extension)
2. Auth file (`~/.pi/agent/auth.json`)
3. Environment variables (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.)

**Verify credentials are available:**
```typescript
// From model registry
const hasAuth = authStorage.hasAuth("anthropic");

if (!hasAuth) {
  console.error("No authentication for anthropic");
  console.error("Run: pi login anthropic");
}
```

**Check OAuth token expiration:**
```typescript
// OAuth tokens stored in auth.json have expiration
const credential = authStorage.getOAuthCredential("anthropic");

if (credential) {
  const expiresAt = new Date(credential.expires);
  const now = new Date();

  if (expiresAt < now) {
    console.warn("OAuth token expired, will attempt refresh");
  } else {
    const hoursLeft = (expiresAt.getTime() - now.getTime()) / (1000 * 60 * 60);
    console.log(`Token expires in ${hoursLeft.toFixed(1)} hours`);
  }
}
```

**Monitor API errors:**
```typescript
pi.on("agent_end", async (event, ctx) => {
  const lastMessage = event.messages[event.messages.length - 1];

  if (lastMessage?.role === "assistant" && lastMessage.stopReason === "error") {
    const error = lastMessage.errorMessage || "Unknown error";

    // Check for common API errors
    if (error.includes("authentication") || error.includes("api_key")) {
      console.error("Authentication error - check API key");
    } else if (error.includes("rate_limit") || error.includes("429")) {
      console.error("Rate limit exceeded");
    } else if (error.includes("overloaded") || error.includes("529")) {
      console.warn("API overloaded - will retry");
    } else {
      console.error("API error:", error);
    }
  }
});
```

## End-to-End Debugging Workflow

This section walks through debugging a failing prompt from start to finish.

### Scenario: Agent Not Responding to Prompt

**Problem:** You send a prompt, but the agent doesn't respond or behaves unexpectedly.

### Step 1: Enable Debug Logging

**Check the debug log file:**
```bash
tail -f ~/.pi/agent/pi-coding-agent-debug.log
```

This file captures:
- Extension loading failures
- Tool execution errors
- Session file I/O issues
- Model API errors
- Internal runtime exceptions

**What to look for:**
- Error messages with stack traces
- Failed API calls with status codes
- File system errors (ENOENT, EACCES)
- Extension initialization errors

### Step 2: Check Session State

**Verify session loaded correctly:**

Run in interactive mode and use `/session` command to inspect:
```
> /session
Session: /Users/example/.pi/agent/sessions/--path--to--project/2026-01-20.jsonl
Entries: 25
Current branch: 25 entries
```

**Programmatically check session:**
```typescript
const entries = ctx.sessionManager.getEntries();
const header = ctx.sessionManager.getHeader();

console.log(`Session ID: ${header.id}`);
console.log(`Total entries: ${entries.length}`);
console.log(`Current leaf: ${ctx.sessionManager.getLeafId()}`);

// Check for corruption
entries.forEach((entry, i) => {
  if (!entry.id || !entry.type || !entry.timestamp) {
    console.error(`Invalid entry at index ${i}:`, entry);
  }
});
```

**Check context usage:**
```typescript
const usage = ctx.getContextUsage();

if (usage) {
  console.log(`Context: ${usage.tokens}/${usage.contextWindow}`);

  if (usage.percent > 95) {
    console.warn("Context nearly full - may cause issues");
    console.warn("Run /compact to free up space");
  }
}
```

### Step 3: Inspect Tool Execution

**Monitor all tool calls:**
```typescript
export default function (pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    console.log(`\n=== Tool Call: ${event.toolName} ===`);
    console.log(`ID: ${event.toolCallId}`);
    console.log(`Input:`, JSON.stringify(event.input, null, 2));
    return undefined; // Allow tool to execute
  });

  pi.on("tool_result", async (event, ctx) => {
    console.log(`\n=== Tool Result: ${event.toolName} ===`);
    console.log(`Success: ${!event.result.isError}`);

    if (event.result.isError) {
      console.error(`Error:`, event.result.content);
    } else {
      console.log(`Content length: ${JSON.stringify(event.result.content).length}`);
    }
  });
}
```

**Check for tool blocking:**
```typescript
pi.on("tool_call", async (event, ctx) => {
  // Example: Block dangerous operations
  if (event.toolName === "bash" && event.input.command.includes("rm -rf")) {
    console.warn(`Blocked dangerous command: ${event.input.command}`);
    return { block: true, reason: "Dangerous command blocked" };
  }

  return undefined; // Allow execution
});
```

**Verify tool output:**
```bash
# Check for truncated output
cat ~/.pi/agent/pi-coding-agent-debug.log | grep -A 5 "truncated"

# Check for timeout errors
cat ~/.pi/agent/pi-coding-agent-debug.log | grep "timeout"
```

### Step 4: Verify Model Response

**Check model selection:**
```typescript
const model = ctx.model;

if (!model) {
  console.error("No model selected!");
  console.error("Available models:", ctx.modelRegistry.getAll().map(m => m.id));
} else {
  console.log(`Using model: ${model.id}`);
  console.log(`Context window: ${model.contextWindow}`);
  console.log(`Supports thinking: ${model.reasoning}`);
}
```

**Monitor model API responses:**
```typescript
pi.on("turn_end", async (event, ctx) => {
  const message = event.message;

  if (message.role === "assistant") {
    console.log(`\n=== Assistant Response ===`);
    console.log(`Stop reason: ${message.stopReason}`);

    if (message.usage) {
      console.log(`Usage: ${message.usage.input + message.usage.output} tokens`);
    }

    if (message.stopReason === "error") {
      console.error(`Error: ${message.errorMessage}`);
    } else if (message.stopReason === "max_tokens") {
      console.warn(`Response truncated at max tokens`);
    }
  }
});
```

**Check for context overflow:**
```typescript
pi.on("agent_end", async (event, ctx) => {
  const lastMessage = event.messages[event.messages.length - 1];

  if (lastMessage?.role === "assistant" && lastMessage.errorMessage) {
    const error = lastMessage.errorMessage;

    if (error.includes("context") && error.includes("too long")) {
      console.error("Context overflow detected!");

      const usage = ctx.getContextUsage();
      if (usage) {
        console.error(`Context: ${usage.tokens}/${usage.contextWindow} tokens`);
      }

      console.error("Solution: Run /compact or enable auto-compaction");
    }
  }
});
```

### Step 5: Review Extension Hooks

**Verify extensions aren't interfering:**
```typescript
export default function (pi: ExtensionAPI) {
  // Log all extension interactions

  pi.on("before_agent_start", async (event, ctx) => {
    console.log("Extension: before_agent_start");
    // Check if messages are being modified
    console.log(`Messages to send: ${event.messages.length}`);
  });

  pi.on("context", async (event, ctx) => {
    console.log("Extension: context event");
    console.log(`Message count: ${event.messages.length}`);

    // Verify no extension is removing messages
    if (event.messages.length === 0) {
      console.error("Extension cleared all messages!");
    }
  });

  pi.on("tool_call", async (event, ctx) => {
    console.log(`Extension: tool_call ${event.toolName}`);
    // Make sure no extension is blocking all tools
    return undefined;
  });
}
```

**Check for extension errors:**
```bash
# Extensions log errors to console
pi --extension ./debug-extension.ts 2>&1 | grep -i error

# Common extension errors:
# - "Extension runtime not initialized" = calling actions during load
# - "Event handler error" = exception in event handler
# - "Failed to load extension" = syntax error or missing dependency
```

### Complete Debug Extension Example

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

/**
 * Complete debugging extension that logs all relevant events
 * Usage: pi --extension ./debug.ts
 */
export default function (pi: ExtensionAPI) {
  console.log("✓ Debug extension loaded\n");

  // Session lifecycle
  pi.on("session_start", async (_event, ctx) => {
    console.log("=== SESSION START ===");
    console.log(`Working directory: ${ctx.cwd}`);
    console.log(`Model: ${ctx.model?.id || "none"}`);

    const usage = ctx.getContextUsage();
    if (usage) {
      console.log(`Context: ${usage.tokens}/${usage.contextWindow} (${usage.percent.toFixed(1)}%)`);
    }
    console.log();
  });

  // Agent lifecycle
  pi.on("agent_start", async (_event, ctx) => {
    console.log(">>> Agent starting");
  });

  pi.on("turn_start", async (event, ctx) => {
    console.log(`>>> Turn ${event.turnIndex} starting`);
  });

  pi.on("turn_end", async (event, ctx) => {
    console.log(`<<< Turn ${event.turnIndex} complete`);

    if (event.message.role === "assistant") {
      const msg = event.message;
      console.log(`    Stop reason: ${msg.stopReason}`);

      if (msg.usage) {
        const total = msg.usage.input + msg.usage.output;
        console.log(`    Tokens: ${total} (${msg.usage.input} in, ${msg.usage.output} out)`);
      }

      if (msg.stopReason === "error") {
        console.error(`    ERROR: ${msg.errorMessage}`);
      }
    }
    console.log();
  });

  pi.on("agent_end", async (event, ctx) => {
    console.log(`<<< Agent complete (${event.messages.length} messages)\n`);
  });

  // Tool execution
  pi.on("tool_call", async (event, ctx) => {
    console.log(`[TOOL CALL] ${event.toolName}`);
    console.log(`    ID: ${event.toolCallId.slice(0, 16)}...`);

    const inputStr = JSON.stringify(event.input);
    const preview = inputStr.length > 100 ? inputStr.slice(0, 100) + "..." : inputStr;
    console.log(`    Input: ${preview}`);

    return undefined;
  });

  pi.on("tool_result", async (event, ctx) => {
    console.log(`[TOOL RESULT] ${event.toolName}`);

    if (event.result.isError) {
      console.error(`    ERROR: ${JSON.stringify(event.result.content)}`);
    } else {
      const contentStr = JSON.stringify(event.result.content);
      console.log(`    Success (${contentStr.length} chars)`);
    }
    console.log();
  });

  // Context monitoring
  pi.on("turn_end", async (_event, ctx) => {
    const usage = ctx.getContextUsage();

    if (usage && usage.percent > 80) {
      console.warn(`⚠️  Context usage: ${usage.percent.toFixed(1)}%`);

      if (usage.percent > 95) {
        console.warn(`⚠️  Context nearly full - consider compaction\n`);
      }
    }
  });

  // Compaction tracking
  pi.on("session_before_compact", async (event, ctx) => {
    console.log("=== COMPACTION STARTING ===");
    console.log(`Tokens before: ${event.preparation.tokensBefore}`);
    console.log(`Messages to summarize: ${event.preparation.messagesToSummarize.length}`);
  });

  pi.on("session_compact", async (event, ctx) => {
    console.log("=== COMPACTION COMPLETE ===");
    console.log(`Summary: ${event.compactionEntry.summary.slice(0, 100)}...`);
    console.log(`Triggered by: ${event.fromExtension ? "extension" : "auto/manual"}\n`);
  });

  // Error tracking
  pi.on("agent_end", async (event, ctx) => {
    const lastMsg = event.messages[event.messages.length - 1];

    if (lastMsg?.role === "assistant" && lastMsg.stopReason === "error") {
      console.error("=== ERROR DETECTED ===");
      console.error(`Message: ${lastMsg.errorMessage}`);
      console.error(`Model: ${ctx.model?.id}`);

      const usage = ctx.getContextUsage();
      if (usage) {
        console.error(`Context: ${usage.tokens}/${usage.contextWindow} tokens`);
      }
      console.error();
    }
  });
}
```

**Expected output from debug extension:**
```
✓ Debug extension loaded

=== SESSION START ===
Working directory: /Users/example/project
Model: claude-sonnet-4-20250514
Context: 1250/200000 (0.6%)

>>> Agent starting
>>> Turn 0 starting
[TOOL CALL] bash
    ID: toolu_012yuiPP1VAf...
    Input: {"command":"ls -la"}
[TOOL RESULT] bash
    Success (523 chars)

<<< Turn 0 complete
    Stop reason: tool_use
    Tokens: 2150 (1850 in, 300 out)

>>> Turn 1 starting
<<< Turn 1 complete
    Stop reason: end_turn
    Tokens: 2450 (2100 in, 350 out)

<<< Agent complete (5 messages)
```

This comprehensive debugging approach helps identify issues at every level of the agent's operation, from session loading to model API responses.
