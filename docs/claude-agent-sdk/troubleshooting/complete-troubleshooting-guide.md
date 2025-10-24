# Claude Agent SDK - Complete Troubleshooting & Debugging Guide

**SDK Version**: 0.1.22
**Source**: SDK source code and implementation analysis

---

## Table of Contents

1. [Common Errors & Solutions](#common-errors--solutions)
2. [Authentication Issues](#authentication-issues)
3. [Tool Execution Problems](#tool-execution-problems)
4. [Permission Errors](#permission-errors)
5. [MCP Server Issues](#mcp-server-issues)
6. [Performance Problems](#performance-problems)
7. [Network & API Errors](#network--api-errors)
8. [Debugging Techniques](#debugging-techniques)
9. [Error Reference](#error-reference)
10. [Diagnostic Tools](#diagnostic-tools)

---

## Common Errors & Solutions

### 1. "Must read file before editing"

**Error**:
```
Error: Must read /path/to/file.ts before editing. Use FileRead tool first.
```

**Cause**: SDK enforces read-before-write for safety

**Solution**:
```typescript
// ❌ Wrong - Edit without reading
const result = await query({
  prompt: "Change line 10 in app.ts"
});

// ✅ Correct - Read first, then edit
const result = await query({
  prompt: "Read app.ts, then change line 10"
});
```

**Why**: Prevents accidental file overwrites. The SDK tracks which files have been read in the current session.

**Workaround**: Use `FileRead` explicitly first
```typescript
// In your agent logic, ensure read happens first
const readResult = await fileTool.read({ file_path: 'app.ts' });
// Now edit is allowed
const editResult = await fileTool.edit({ 
  file_path: 'app.ts',
  old_string: 'const x = 1',
  new_string: 'const x = 2'
});
```

---

### 2. "old_string must be unique"

**Error**:
```
Error: old_string appears 3 times. Must be unique or use replace_all: true
```

**Cause**: `FileEdit` requires unique match for safety

**Solution**:
```typescript
// ❌ Wrong - old_string appears multiple times
{
  file_path: 'app.ts',
  old_string: 'import React',  // Appears 10 times!
  new_string: 'import React, { useState }'
}

// ✅ Solution 1: Make old_string more specific
{
  file_path: 'app.ts',
  old_string: 'import React from "react";\nimport { Component } from "react";',
  new_string: 'import React, { Component, useState } from "react";'
}

// ✅ Solution 2: Use replace_all flag
{
  file_path: 'app.ts',
  old_string: 'console.log',
  new_string: 'logger.debug',
  replace_all: true  // Replace ALL occurrences
}
```

**Best Practice**: Include enough context to make match unique

---

### 3. "Output truncated silently"

**Problem**: Bash output or file content appears cut off

**Cause**: Built-in limits:
- **Bash output**: 30,000 characters max
- **FileRead per line**: 2,000 characters max
- **FileRead default**: 2,000 lines max

**Solution**:
```typescript
// For large files - use offset/limit
{
  file_path: 'large-file.txt',
  offset: 2000,    // Skip first 2000 lines
  limit: 2000      // Read next 2000 lines
}

// For long bash output - use BashOutput tool
// Returns structured output without truncation
const result = await bashOutputTool.execute({
  command: 'find . -name "*.ts"'
});

// Or split into smaller chunks
await bashTool.execute({
  command: 'ls | head -n 100'  // First 100
});
await bashTool.execute({
  command: 'ls | tail -n +101 | head -n 100'  // Next 100
});
```

**Detection**: Look for suspiciously round numbers (exactly 2000 lines, exactly 30000 chars)

---

### 4. "Hook timeout"

**Error**:
```
Warning: Hook timed out after 5000ms
```

**Cause**: Hook took longer than 5 seconds (hardcoded limit)

**Solution**:
```typescript
// ❌ Wrong - Slow operation in hook
const slowHook = async (input) => {
  // This takes 10 seconds!
  await slowDatabaseQuery();
  return { decision: 'approve' };
};

// ✅ Solution 1: Make hook fast
const fastHook = async (input) => {
  // Quick validation only
  if (input.tool_name === 'Bash' && input.tool_input.command.includes('rm -rf')) {
    return { decision: 'block' };
  }
  return { decision: 'approve' };
};

// ✅ Solution 2: Use async hook for logging
const asyncHook = async (input) => {
  // Return immediately
  setTimeout(() => {
    // Log asynchronously (fire-and-forget)
    logToDatabase(input);
  }, 0);
  
  return { 
    async: true  // Tells SDK to not wait
  };
};
```

**Best Practice**: Keep hooks under 1 second. Use async: true for logging/metrics.

---

### 5. "Permission denied"

**Error**:
```
Error: Permission denied: tool 'Bash' not allowed
```

**Cause**: Permission resolution blocked the tool

**Solution**:
```typescript
// Check permission mode
const result = await query({
  prompt: "Run git status",
  options: {
    permissionMode: 'default'  // May block Bash
  }
});

// Solution 1: Use acceptEdits mode (more permissive)
const result = await query({
  prompt: "Run git status",
  options: {
    permissionMode: 'acceptEdits'  // Auto-approves file edits + bash
  }
});

// Solution 2: Custom permission handler
const result = await query({
  prompt: "Run git status",
  options: {
    canUseTool: async (toolName, input) => {
      if (toolName === 'Bash' && input.command.startsWith('git ')) {
        return { behavior: 'allow' };  // Allow git commands
      }
      return { behavior: 'deny' };
    }
  }
});

// Solution 3: Add permission rule to settings.json
// ~/.claude/settings.json
{
  "permissions": {
    "rules": [
      {
        "tool": "Bash",
        "command": "git *",
        "behavior": "allow"
      }
    ]
  }
}
```

---

### 6. "MCP server connection timeout"

**Error**:
```
Error: MCP server 'my-server' connection timeout after 30s
```

**Cause**: MCP server took too long to start or respond

**Solution**:
```typescript
// ❌ Wrong - Server takes too long to start
{
  type: 'stdio',
  command: 'python',
  args: ['slow-server.py']  // Takes 45 seconds to start!
}

// ✅ Solution 1: Optimize server startup
// Make sure your MCP server starts quickly (<30s)

// ✅ Solution 2: Pre-start server
// Start server before calling query()
const serverProcess = spawn('python', ['server.py']);
await new Promise(resolve => setTimeout(resolve, 5000)); // Wait for startup

const result = await query({
  prompt: "Use custom tool",
  options: {
    mcpServers: {
      'custom': {
        type: 'http',  // Connect to already-running server
        url: 'http://localhost:8080'
      }
    }
  }
});

// ✅ Solution 3: Use SDK transport (in-process, no timeout)
const server = createSdkMcpServer({
  name: 'fast-server',
  tools: [myTool]
});

const result = await query({
  prompt: "Use custom tool",
  options: {
    mcpServers: {
      'custom': server  // Instant connection!
    }
  }
});
```

---

### 7. "TodoWrite constraint violation"

**Error**:
```
Error: Can only mark ONE todo as in_progress
```

**Cause**: Tried to set multiple todos to `in_progress` status

**Solution**:
```typescript
// ❌ Wrong - Multiple in_progress
{
  todos: [
    { id: '1', status: 'in_progress', content: 'Task 1' },
    { id: '2', status: 'in_progress', content: 'Task 2' }  // ERROR!
  ]
}

// ✅ Correct - Only one in_progress
{
  todos: [
    { id: '1', status: 'in_progress', content: 'Task 1' },
    { id: '2', status: 'pending', content: 'Task 2' }  // OK
  ]
}

// When starting new task, complete the old one
{
  todos: [
    { id: '1', status: 'completed', content: 'Task 1' },
    { id: '2', status: 'in_progress', content: 'Task 2' }
  ]
}
```

---

### 8. "Bash command timeout"

**Error**:
```
Error: Command timed out after 120000ms
```

**Cause**: Bash command exceeded default 2-minute timeout

**Solution**:
```typescript
// ❌ Wrong - Long-running command with default timeout
{
  command: 'npm install'  // Takes 5 minutes!
}

// ✅ Solution 1: Increase timeout (max 10 minutes)
{
  command: 'npm install',
  timeout: 600000  // 10 minutes (maximum allowed)
}

// ✅ Solution 2: Break into smaller commands
{
  command: 'npm install --prefer-offline'  // Faster
}

// ✅ Solution 3: Run in background (for very long tasks)
{
  command: 'nohup npm install > install.log 2>&1 &'
}

// Then check status later
{
  command: 'tail install.log'
}
```

---

## Authentication Issues

### API Key Not Found

**Error**:
```
Error: No API key found. Set ANTHROPIC_API_KEY environment variable.
```

**Cause**: SDK can't locate API key

**Resolution Order** (SDK checks in this order):
```
1. options.apiKey (explicit parameter)
   ↓
2. ANTHROPIC_API_KEY environment variable
   ↓
3. ~/.claude/settings.json (user config)
   ↓
4. .claude/settings.json (project config)
   ↓
5. API key from OAuth flow (if authenticated)
```

**Solutions**:

```bash
# Solution 1: Environment variable (recommended for dev)
export ANTHROPIC_API_KEY="sk-ant-api03-your-key-here"

# Solution 2: User settings (recommended for personal use)
mkdir -p ~/.claude
echo '{"apiKey": "sk-ant-api03-your-key-here"}' > ~/.claude/settings.json

# Solution 3: Project settings (recommended for teams)
mkdir -p .claude
echo '{"apiKey": "sk-ant-api03-your-key-here"}' > .claude/settings.json

# Solution 4: OAuth (recommended for enterprises)
npx @anthropic-ai/claude-agent-sdk auth login
```

```typescript
// Solution 5: Explicit in code (NOT recommended for production)
const result = await query({
  prompt: "Hello",
  options: {
    apiKey: "sk-ant-api03-your-key-here"  // Visible in code!
  }
});
```

---

### Invalid API Key

**Error**:
```
Error: 401 Unauthorized - Invalid API key
```

**Checks**:
1. ✓ Key starts with `sk-ant-api03-` (correct format)
2. ✓ No extra spaces or newlines
3. ✓ Key is active (check console.anthropic.com)
4. ✓ Correct environment (prod vs staging)

**Solution**:
```bash
# Verify key format
echo $ANTHROPIC_API_KEY | cat -A  # Check for hidden chars

# Test key directly
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-3-5-sonnet-20241022",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello"}]
  }'

# If 401, key is invalid - generate new one at:
# https://console.anthropic.com/settings/keys
```

---

### Rate Limit Exceeded

**Error**:
```
Error: 429 Too Many Requests - Rate limit exceeded
```

**Cause**: Too many requests to Anthropic API

**Solution**:
```typescript
// Implement retry with exponential backoff
async function queryWithRetry(prompt: string, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await query({ prompt });
    } catch (error) {
      if (error.status === 429) {
        // Extract retry-after header
        const retryAfter = error.response?.headers['retry-after'] || 60;
        const delay = (retryAfter * 1000) * Math.pow(2, attempt);
        
        console.warn(`Rate limited. Retrying after ${delay}ms...`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries exceeded');
}
```

**Prevention**:
```typescript
// Use token budget to prevent rate limits
class RateLimitedAgent {
  private requestCount = 0;
  private windowStart = Date.now();
  private readonly MAX_REQUESTS_PER_MINUTE = 50;
  
  async execute(prompt: string) {
    // Check rate limit
    const now = Date.now();
    if (now - this.windowStart > 60000) {
      this.requestCount = 0;
      this.windowStart = now;
    }
    
    if (this.requestCount >= this.MAX_REQUESTS_PER_MINUTE) {
      const waitTime = 60000 - (now - this.windowStart);
      console.warn(`Rate limit reached. Waiting ${waitTime}ms...`);
      await new Promise(resolve => setTimeout(resolve, waitTime));
      this.requestCount = 0;
      this.windowStart = Date.now();
    }
    
    this.requestCount++;
    return await query({ prompt });
  }
}
```

---

## Tool Execution Problems

### Tool Not Found

**Error**:
```
Error: Tool 'CustomTool' not found
```

**Causes & Solutions**:

```typescript
// Cause 1: Tool not in allowed list
{
  allowedTools: ['FileRead', 'FileWrite'],  // CustomTool missing!
  mcpServers: {
    'custom': createSdkMcpServer({
      tools: [customTool]
    })
  }
}

// Solution: Don't use allowedTools with MCP tools (they're auto-allowed)
{
  mcpServers: {
    'custom': createSdkMcpServer({
      tools: [customTool]  // Automatically available
    })
  }
}

// Cause 2: Tool in disallowed list
{
  disallowedTools: ['CustomTool'],  // Blocks the tool!
  mcpServers: { /* ... */ }
}

// Solution: Remove from disallowed list
{
  disallowedTools: [],  // Allow all
  mcpServers: { /* ... */ }
}

// Cause 3: MCP server failed to start
// Check server status:
const status = await queryInstance.mcpServerStatus();
console.log(status);  // Check for 'failed' or 'needs-auth'
```

---

### Tool Execution Hangs

**Problem**: Tool never returns, execution hangs forever

**Common Causes**:

1. **Bash command waiting for input**
```bash
# ❌ Hangs - waiting for password
sudo apt install package

# ✅ Fixed - non-interactive
sudo -n apt install package || echo "Need sudo"
```

2. **Custom tool async issue**
```typescript
// ❌ Wrong - Promise never resolves
const brokenTool = tool('broken', 'Broken tool', {}, async () => {
  someAsyncOperation();  // Forgot to await!
  return { content: [{ type: 'text', text: 'Done' }] };
});

// ✅ Correct - Properly await
const fixedTool = tool('fixed', 'Fixed tool', {}, async () => {
  await someAsyncOperation();  // Properly awaited
  return { content: [{ type: 'text', text: 'Done' }] };
});
```

3. **MCP server deadlock**
```typescript
// Tool calls another MCP tool in same server (deadlock!)
// Solution: Use separate servers or SDK transport
```

**Debugging**:
```typescript
// Add timeout wrapper
async function executeWithTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number
): Promise<T> {
  const timeout = new Promise<never>((_, reject) => {
    setTimeout(() => reject(new Error('Timeout')), timeoutMs);
  });
  return Promise.race([promise, timeout]);
}

// Use with query
const result = await executeWithTimeout(
  query({ prompt: "Task" }),
  300000  // 5 minute timeout
);
```

---

## Permission Errors

### Permission Denied (Unexpected)

**Problem**: Tool blocked despite permission mode

**Debug Permission Resolution**:

```typescript
// Enable verbose logging
process.env.DEBUG = 'claude:permissions';

// Check effective permissions
const result = await query({
  prompt: "What permissions do I have?",
  options: {
    permissionMode: 'acceptEdits',
    canUseTool: async (toolName, input, context) => {
      // Log all permission checks
      console.log('Permission check:', {
        tool: toolName,
        input,
        suggestions: context.suggestions
      });
      
      // Let default handler decide
      return { behavior: 'allow' };
    }
  }
});
```

**Check Configuration Cascade**:
```bash
# 1. Session rules (highest priority)
# Stored in memory, cleared on restart

# 2. Local config (.claude/ in current directory)
cat .claude/settings.json

# 3. Project config (.claude/ in git root)
cd $(git rev-parse --show-toplevel)
cat .claude/settings.json

# 4. User config (~/.claude/)
cat ~/.claude/settings.json

# 5. Policy config (enterprise)
# Managed by IT, check with admin

# 6. Permission mode (fallback)
# Default: ask user
# acceptEdits: auto-approve file operations
# bypassPermissions: approve everything (DANGEROUS!)
```

---

## MCP Server Issues

### Server Failed to Start

**Error**:
```
MCP server 'my-server' status: failed
```

**Diagnostic Steps**:

```typescript
// 1. Check server status
const status = await queryInstance.mcpServerStatus();
for (const server of status) {
  if (server.status === 'failed') {
    console.error(`Server ${server.name} failed!`);
  }
}

// 2. Test server manually
// For stdio server:
const { spawn } = require('child_process');
const proc = spawn('npx', ['-y', 'my-mcp-server']);
proc.stdout.on('data', data => console.log('stdout:', data.toString()));
proc.stderr.on('data', data => console.error('stderr:', data.toString()));

// 3. Check server logs
// Most MCP servers log to stderr
// Set LOG_LEVEL=debug in env
{
  type: 'stdio',
  command: 'npx',
  args: ['-y', 'my-mcp-server'],
  env: {
    LOG_LEVEL: 'debug'
  }
}

// 4. Verify server is MCP-compatible
// Must implement MCP protocol 1.0
// Check server's package.json or docs
```

**Common Issues**:

```typescript
// Issue 1: Wrong command
{
  command: 'my-server'  // Not in PATH!
}
// Fix: Use full path or npx
{
  command: 'npx',
  args: ['-y', '@my/mcp-server']
}

// Issue 2: Missing dependencies
// Server requires Python but it's not installed
// Fix: Install dependencies first

// Issue 3: Port conflict (HTTP/SSE servers)
{
  type: 'http',
  url: 'http://localhost:8080'  // Port already in use!
}
// Fix: Use different port or kill existing process
```

---

## Performance Problems

### Slow Response Times

**Problem**: Queries take minutes instead of seconds

**Diagnostic**:

```typescript
// Measure performance
const start = Date.now();
const result = await query({ prompt: "Task" });

for await (const message of result) {
  if (message.type === 'result') {
    console.log('Total time:', Date.now() - start, 'ms');
    console.log('API time:', message.duration_api_ms, 'ms');
    console.log('Turns:', message.num_turns);
    console.log('Tokens:', message.usage.input_tokens + message.usage.output_tokens);
  }
}
```

**Common Causes & Solutions**:

```typescript
// 1. Too many turns
{
  maxTurns: 50  // Taking 50 turns!
}
// Solution: Reduce maxTurns or use more specific prompts
{
  maxTurns: 5  // Limit turns
}

// 2. Using Opus for everything
{
  model: 'opus'  // Slowest model!
}
// Solution: Use Sonnet or Haiku for non-critical tasks
{
  model: 'sonnet'  // 2-3x faster
}

// 3. Large context without caching
// Solution: Enable prompt caching
{
  systemPrompt: largeCodebase,  // Will be cached
  model: 'sonnet'
}

// 4. Slow MCP servers
// Diagnostic: Check which server is slow
const status = await queryInstance.mcpServerStatus();
// Solution: Use SDK transport for custom tools (faster)

// 5. Extended thinking enabled
{
  maxThinkingTokens: 31999  // Adds latency!
}
// Solution: Only use for complex problems
{
  maxThinkingTokens: 0  // Disable for simple tasks
}
```

---

### High Token Usage

**Problem**: Burning through token budget quickly

**Analysis**:

```typescript
for await (const message of result) {
  if (message.type === 'result') {
    console.log('Usage breakdown:');
    console.log('  Input tokens:', message.usage.input_tokens);
    console.log('  Output tokens:', message.usage.output_tokens);
    console.log('  Cache write:', message.usage.cache_creation_input_tokens);
    console.log('  Cache read:', message.usage.cache_read_input_tokens);
    console.log('Total cost: $', message.total_cost_usd);
    
    // Per-model breakdown
    for (const [model, usage] of Object.entries(message.modelUsage)) {
      console.log(`${model}:`, usage.inputTokens + usage.outputTokens, 'tokens');
    }
  }
}
```

**Optimization Strategies**:

```typescript
// 1. Use Explore agent (70-84% savings)
{
  agents: {
    'Explore': {
      description: 'Fast discovery',
      tools: ['Glob', 'Grep', 'Read'],
      prompt: 'Find files quickly',
      model: 'haiku'
    }
  }
}

// 2. Enable prompt caching
{
  systemPrompt: largeStaticContext  // Cached after first use
}

// 3. Use Haiku for simple tasks
{
  model: 'haiku'  // 5x cheaper than Opus
}

// 4. Limit file reads
// Don't read entire codebase
// Use Grep to find specific content

// 5. Use isolated context for agents
{
  agents: {
    'Explore': {
      // ... config ...
      // Isolated = doesn't inherit conversation context
    }
  }
}
```

---

## Network & API Errors

### Connection Timeout

**Error**:
```
Error: ETIMEDOUT - Connection to api.anthropic.com timed out
```

**Solutions**:

```bash
# 1. Check network connectivity
ping api.anthropic.com
curl -I https://api.anthropic.com

# 2. Check proxy settings
echo $HTTP_PROXY
echo $HTTPS_PROXY

# 3. Configure proxy if needed
export HTTPS_PROXY=http://proxy.company.com:8080
export HTTP_PROXY=http://proxy.company.com:8080

# 4. Check firewall
# Ensure outbound HTTPS (443) is allowed to:
# - api.anthropic.com
# - console.anthropic.com
```

```typescript
// 5. Increase timeout (if network is slow)
// Currently not configurable in SDK
// Workaround: Use AbortController
const controller = new AbortController();
setTimeout(() => controller.abort(), 60000); // 1 minute

const result = await query({
  prompt: "Task",
  options: {
    abortController: controller
  }
});
```

---

### SSL/TLS Errors

**Error**:
```
Error: UNABLE_TO_VERIFY_LEAF_SIGNATURE
```

**Cause**: Corporate proxy with SSL inspection

**Solution**:

```bash
# Option 1: Add corporate CA certificate
export NODE_EXTRA_CA_CERTS=/path/to/corporate-ca.pem

# Option 2: Disable SSL verification (NOT RECOMMENDED for production)
export NODE_TLS_REJECT_UNAUTHORIZED=0
```

---

## Debugging Techniques

### Enable Debug Logging

```bash
# All SDK debug logs
export DEBUG=claude:*

# Specific subsystems
export DEBUG=claude:permissions
export DEBUG=claude:hooks
export DEBUG=claude:mcp
export DEBUG=claude:tools

# Multiple subsystems
export DEBUG=claude:permissions,claude:hooks
```

### Inspect Streaming Messages

```typescript
const result = await query({
  prompt: "Debug test",
  options: {
    includePartialMessages: true  // Include streaming chunks
  }
});

for await (const message of result) {
  console.log('\n--- Message ---');
  console.log('Type:', message.type);
  console.log('Content:', JSON.stringify(message, null, 2));
}
```

### Session Inspection

```typescript
// Examine session state
const result = await query({ prompt: "Test" });

for await (const message of result) {
  if (message.type === 'system' && message.subtype === 'init') {
    console.log('Session ID:', message.session_id);
    console.log('CWD:', message.cwd);
    console.log('Tools:', message.tools);
    console.log('Agents:', message.agents);
    console.log('MCP Servers:', message.mcp_servers);
    console.log('API Key Source:', message.apiKeySource);
  }
}
```

### Hook Debugging

```typescript
const debugHook: HookCallback = async (input, toolUseID, context) => {
  console.log('\n=== HOOK DEBUG ===');
  console.log('Event:', input.hook_event_name);
  console.log('Tool:', input.tool_name);
  console.log('Input:', input.tool_input);
  console.log('Session:', input.session_id);
  console.log('Permission Mode:', input.permission_mode);
  
  if (input.hook_event_name === 'PreToolUse') {
    console.log('Suggestions:', context.suggestions);
  }
  
  return { decision: 'approve' };
};

const result = await query({
  prompt: "Test",
  options: {
    hooks: {
      PreToolUse: [{ hooks: [debugHook] }],
      PostToolUse: [{ hooks: [debugHook] }]
    }
  }
});
```

### Performance Profiling

```typescript
class ProfiledAgent {
  private metrics = new Map<string, number[]>();
  
  recordDuration(name: string, duration: number) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }
    this.metrics.get(name)!.push(duration);
  }
  
  async profiledQuery(prompt: string) {
    const start = Date.now();
    
    const result = await query({ prompt });
    
    let toolCount = 0;
    const toolTimes = new Map<string, number[]>();
    
    for await (const message of result) {
      if (message.type === 'assistant') {
        const toolUses = message.message.content.filter(
          c => c.type === 'tool_use'
        );
        toolCount += toolUses.length;
      } else if (message.type === 'result') {
        const totalTime = Date.now() - start;
        
        console.log('\n=== PERFORMANCE PROFILE ===');
        console.log('Total time:', totalTime, 'ms');
        console.log('API time:', message.duration_api_ms, 'ms');
        console.log('Overhead:', totalTime - message.duration_api_ms, 'ms');
        console.log('Turns:', message.num_turns);
        console.log('Tools used:', toolCount);
        console.log('Tokens:', message.usage.input_tokens + message.usage.output_tokens);
        console.log('Cost: $', message.total_cost_usd.toFixed(4));
      }
    }
  }
}
```

---

## Error Reference

### Error Codes

| Code | Error | Cause | Solution |
|------|-------|-------|----------|
| **401** | Unauthorized | Invalid/missing API key | Check API key |
| **403** | Forbidden | API key lacks permission | Verify key has `user:inference` scope |
| **429** | Rate Limited | Too many requests | Implement retry with backoff |
| **500** | Server Error | Anthropic API issue | Retry request |
| **503** | Service Unavailable | API temporarily down | Wait and retry |
| **ETIMEDOUT** | Network Timeout | Network connectivity | Check internet connection |
| **ECONNREFUSED** | Connection Refused | Service not reachable | Check firewall/proxy |
| **ENOTFOUND** | DNS Error | Can't resolve hostname | Check DNS settings |

### SDK-Specific Errors

| Error | Meaning | Fix |
|-------|---------|-----|
| `Must read file before editing` | Read-before-write enforcement | Read file first |
| `old_string must be unique` | Edit ambiguity | Make old_string more specific or use replace_all |
| `Hook timeout` | Hook took >5s | Optimize hook or use async |
| `Tool not found` | Tool not available | Check MCP server or allowedTools |
| `Permission denied` | Permission blocked tool | Adjust permission mode or rules |
| `MCP server timeout` | Server didn't start | Check server command/config |
| `Only ONE in_progress allowed` | TodoWrite constraint | Complete previous todo first |
| `Bash timeout` | Command took >2min | Increase timeout or split command |

---

## Diagnostic Tools

### Health Check Script

```typescript
async function healthCheck() {
  console.log('=== Claude Agent SDK Health Check ===\n');
  
  // 1. Check API key
  const apiKey = process.env.ANTHROPIC_API_KEY;
  console.log('1. API Key:', apiKey ? '✓ Found' : '✗ Missing');
  
  // 2. Test connection
  try {
    const testQuery = await query({
      prompt: "Hello",
      options: { model: 'haiku' }
    });
    
    let success = false;
    for await (const msg of testQuery) {
      if (msg.type === 'result') {
        success = msg.subtype === 'success';
      }
    }
    console.log('2. API Connection:', success ? '✓ Working' : '✗ Failed');
  } catch (error) {
    console.log('2. API Connection: ✗ Failed -', error.message);
  }
  
  // 3. Check MCP servers
  try {
    const status = await queryInstance.mcpServerStatus();
    console.log('3. MCP Servers:');
    for (const server of status) {
      console.log(`   - ${server.name}: ${server.status}`);
    }
  } catch (error) {
    console.log('3. MCP Servers: ✗ Not tested');
  }
  
  // 4. Check ripgrep (for Grep tool)
  try {
    const { execSync } = require('child_process');
    execSync('rg --version', { stdio: 'ignore' });
    console.log('4. ripgrep: ✓ Available');
  } catch {
    console.log('4. ripgrep: ✗ Missing (Grep tool may not work)');
  }
  
  // 5. Check permissions
  console.log('5. Permissions: Check settings files');
  console.log('   - User: ~/.claude/settings.json');
  console.log('   - Project: .claude/settings.json');
}

await healthCheck();
```

### Debug Environment

```typescript
function printDebugInfo() {
  console.log('=== Debug Information ===\n');
  console.log('Node version:', process.version);
  console.log('Platform:', process.platform);
  console.log('Arch:', process.arch);
  console.log('CWD:', process.cwd());
  console.log('HOME:', process.env.HOME);
  console.log('PATH:', process.env.PATH);
  console.log('\nEnvironment:');
  console.log('  ANTHROPIC_API_KEY:', process.env.ANTHROPIC_API_KEY ? 'Set' : 'Not set');
  console.log('  DEBUG:', process.env.DEBUG || 'Not set');
  console.log('  NODE_ENV:', process.env.NODE_ENV || 'Not set');
  console.log('  HTTP_PROXY:', process.env.HTTP_PROXY || 'Not set');
  console.log('  HTTPS_PROXY:', process.env.HTTPS_PROXY || 'Not set');
}
```

---

## Summary

### Quick Diagnostic Checklist

When things go wrong:

- [ ] Check API key is set and valid
- [ ] Enable DEBUG logging
- [ ] Check network connectivity
- [ ] Verify permission mode
- [ ] Check MCP server status
- [ ] Review recent code changes
- [ ] Check for silent truncation
- [ ] Verify hook timeouts
- [ ] Check session state
- [ ] Review error messages carefully

### Common Fixes

| Problem | Quick Fix |
|---------|-----------|
| Authentication | `export ANTHROPIC_API_KEY=sk-ant-...` |
| Permission denied | `permissionMode: 'acceptEdits'` |
| MCP timeout | Use SDK transport instead of stdio |
| Hook timeout | Make hook <1s or use async:true |
| Rate limit | Implement exponential backoff |
| Slow performance | Use Haiku + Explore agent |
| High token cost | Enable caching + use Haiku |
| Edit fails | Read file first + unique old_string |

### Getting Help

1. **Enable debug logs**: `export DEBUG=claude:*`
2. **Run health check**: Use script above
3. **Check SDK version**: `npm list @anthropic-ai/claude-agent-sdk`
4. **Search existing issues**: [GitHub Issues](https://github.com/anthropics/claude-code/issues)
5. **Ask on Discord**: [Anthropic Discord](https://discord.gg/anthropic)
6. **Report bugs**: Include debug logs + reproduction steps

