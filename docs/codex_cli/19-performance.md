# Performance Tuning Guide

Optimize Codex for speed, cost, and quality through model selection, caching, context management, and configuration tuning.

---

## Performance Dimensions

### 1. Response Speed
How fast Codex generates responses.

**Factors:**
- Model selection (o3 vs gpt-5-codex vs gpt-5)
- Reasoning effort setting
- Prompt caching hit rate
- Network latency

### 2. Cost Efficiency
Total API cost per session.

**Factors:**
- Token usage (input + output + cached)
- Model pricing tier
- Compaction frequency
- Tool call count

### 3. Quality
Accuracy and usefulness of responses.

**Factors:**
- Model capability
- Reasoning effort/summary settings
- Context window utilization
- AGENTS.md guidance quality

**Trade-off:** Speed ↔ Cost ↔ Quality. Optimize based on your priorities.

---

## Model Selection

### Model Comparison

| Model | Speed | Cost | Quality | Best For |
|-------|-------|------|---------|----------|
| **gpt-5-codex** | Fast | $$ | Excellent | Daily coding tasks |
| **gpt-5** | Fast | $$ | Excellent | General tasks |
| **o3** | Slow | $$$$ | Best | Complex reasoning |
| **o4-mini** | Medium | $$ | Good | Cost-sensitive workloads |
| **gpt-4o** | Medium | $ | Good | Chat Completions API |

### Configuration

**In `config.toml`:**

```toml
# Fast and cost-effective
model = "gpt-5-codex"

# Maximum quality (slow)
model = "o3"
model_reasoning_effort = "high"
model_reasoning_summary = "detailed"

# Balanced
model = "gpt-5"
model_reasoning_effort = "medium"
```

**CLI Override:**

```bash
# Quick task with fast model
codex --model gpt-5-codex "Fix linting errors"

# Complex task with reasoning
codex --model o3 --config model_reasoning_effort="high" "Refactor for thread safety"
```

### Reasoning Settings (o3, o4-mini, codex models)

**Effort Levels:**
- `minimal` - Fastest, lowest quality reasoning
- `low` - Quick responses
- `medium` - Balanced (default)
- `high` - Thorough analysis (slowest)

**Summary Verbosity:**
- `none` - No reasoning shown
- `auto` - Model decides (default)
- `concise` - Brief summary
- `detailed` - Full reasoning trace

```toml
model = "o3"
model_reasoning_effort = "high"      # More thinking time
model_reasoning_summary = "detailed"  # Show full reasoning
```

**Trade-off:** Higher effort = better results but slower responses and higher cost.

---

## Prompt Caching

Codex automatically uses OpenAI's prompt caching to speed up repeated context.

### How It Works

1. **First Request:** Full context sent, no cache hit
2. **Subsequent Requests:** Repeated parts cached (system prompt, AGENTS.md, old messages)
3. **Cache Tokens:** Billed at ~50% discount

### Cache Efficiency

**High cache efficiency:**
- Long AGENTS.md files (reused every turn)
- Large repositories with stable context
- Long conversations (old messages cached)

**Low cache efficiency:**
- Short conversations
- Frequently changing AGENTS.md
- Small repositories

### Monitoring Cache Usage

Check `/status` in TUI or logs:

```
Turn completed
  Input tokens: 15,000
  Cached tokens: 12,000  ← 80% cache hit rate!
  Output tokens: 500
```

### Optimizing for Caching

1. **Stable AGENTS.md** - Avoid changing frequently
2. **Long system prompts** - More content to cache
3. **Keep sessions alive** - Cache persists within session
4. **Resume sessions** - Cache may persist across resumes

```toml
# Maximize cache lifespan
auto_compact_token_limit = 150000  # Delay compaction
```

**Note:** Compaction clears cache (old messages discarded).

---

## Context Window Management

### Token Budget

Models have limited context windows:

| Model | Context Window |
|-------|----------------|
| gpt-5-codex | 128K tokens |
| gpt-5 | 128K tokens |
| o3 | 128K tokens |
| gpt-4o | 128K tokens |

**1 token ≈ 4 characters** (approximate)

### Token Usage Breakdown

Typical Codex session (turn 5 of a conversation):

```
System prompt:         5,000 tokens
AGENTS.md:            3,000 tokens
Prior messages:      15,000 tokens (cached)
Tool definitions:     2,000 tokens
Current context:      5,000 tokens
─────────────────────────────────────
Total input:         30,000 tokens
Output:               2,000 tokens
```

### Compaction

When approaching context limit, Codex compresses old messages.

**Configuration:**

```toml
# Trigger compaction at 100K tokens (default: 90% of context window)
auto_compact_token_limit = 100000
```

**Manual Compaction:**

Use `/compact` in TUI (if available) or wait for automatic trigger.

**Compaction Strategy:**

1. Select oldest 50% of messages
2. Send compression request: "Summarize these messages in 2-3 sentences"
3. Replace old messages with summary
4. Continue conversation with reduced context

**Trade-off:** Compaction adds latency (extra API call) but extends conversation life.

---

## Tool Execution Performance

### Tool Latency

Typical tool execution times:

- **`read_file`**: 1-5ms
- **`list_dir`**: 5-20ms
- **`grep_files`**: 10-100ms (depends on scope)
- **`local_shell`**: 200ms-120s (depends on command)
- **MCP tools**: Varies (configurable timeout)

### Optimization Tips

#### 1. Batch Read Operations

**Bad:**
```
read_file("a.txt")
read_file("b.txt")
read_file("c.txt")
```

**Good:**
```
// Model requests all three in parallel
[read_file("a.txt"), read_file("b.txt"), read_file("c.txt")]
```

Codex executes parallel read operations concurrently.

#### 2. Limit Grep Scope

**Bad:**
```
grep_files(pattern="TODO", path="/")  # Searches entire filesystem
```

**Good:**
```
grep_files(pattern="TODO", path="./src")  # Scoped to project
```

#### 3. Tune MCP Timeouts

```toml
[mcp_servers.slow_server]
command = "python"
args = ["server.py"]
tool_timeout_sec = 300  # 5 minutes for slow operations
```

**Default:** 60 seconds

#### 4. Use Faster Shell Commands

**Bad:**
```bash
find . -name "*.ts" | wc -l  # Slow recursive search
```

**Good:**
```bash
rg --files --type ts | wc -l  # Fast with ripgrep
```

---

## Network Optimization

### Request Latency

**Factors:**
- Geographic distance to OpenAI API
- Network bandwidth
- TLS handshake overhead

### Reducing Latency

#### 1. Use HTTP/2 (Automatic)

Codex uses HTTP/2 by default for multiplexing and reduced overhead.

#### 2. Increase Timeouts (If Needed)

```toml
[model_providers.openai]
stream_idle_timeout_ms = 600000  # 10 minutes (default: 5 minutes)
```

Use for:
- Slow networks
- Large responses
- High-latency connections

#### 3. Retry Configuration

```toml
[model_providers.openai]
request_max_retries = 4     # HTTP request retries (default: 4)
stream_max_retries = 10      # SSE stream retries (default: 5)
```

Higher retries = more resilience to network issues.

---

## Session Management

### Resume vs. New Session

**Resume:**
- **Pro:** Preserves context, faster startup
- **Con:** Larger context (more input tokens)

**New Session:**
- **Pro:** Clean slate, smaller context
- **Con:** Loses history, must re-explain context

**Recommendation:** Resume for related tasks, new session for unrelated tasks.

### Compaction Frequency

**Aggressive Compaction:**
```toml
auto_compact_token_limit = 50000  # Compact early
```
- **Pro:** Keeps context small, cheaper per turn
- **Con:** Frequent API calls, may lose important context

**Conservative Compaction:**
```toml
auto_compact_token_limit = 150000  # Compact late
```
- **Pro:** Retains more history, better continuity
- **Con:** Higher token cost, longer prompts

**Recommendation:** Default (90% of context window) works well.

### Fork vs. Continue

**Fork (backtrack):**
- Creates new branch from previous message
- Keeps original conversation intact
- **Cost:** Same as continuing (shares history)

**Continue:**
- Adds to existing conversation
- **Cost:** Grows context over time

**Recommendation:** Fork to try alternative approaches without starting over.

---

## MCP Server Performance

### Startup Time

**Slow startup:**
```toml
[mcp_servers.heavy_server]
command = "python"
args = ["server.py"]
startup_timeout_sec = 30  # Default: 10s
```

Increase timeout for:
- Docker containers
- Servers with slow initialization
- Remote HTTP servers

### Tool Execution Time

```toml
[mcp_servers.slow_tool_server]
tool_timeout_sec = 180  # 3 minutes
```

Increase for:
- Database queries
- API calls to external services
- Heavy computation

### Connection Pooling (HTTP Servers)

For Streamable HTTP MCP servers:

```javascript
// In your MCP server
const transport = new HttpServerTransport({
  keepAlive: true,
  keepAliveTimeout: 30000,  // 30 seconds
});
```

Reuses connections for multiple tool calls.

---

## Monitoring Performance

### TUI Status

Press `/status` to see:

```
Model: gpt-5-codex
Approval: on-request
Sandbox: workspace-write

Last turn:
  Input tokens: 25,000 (15,000 cached)
  Output tokens: 1,500
  Duration: 3.2s

MCP Servers:
  ✓ docs (connected, 15ms startup)
  ✓ database (connected, 1.2s startup)
```

### Log Analysis

Enable debug logging:

```bash
export RUST_LOG=codex_core=debug
codex
```

Check logs:

```bash
tail -F ~/.codex/log/codex-tui.log
```

**Look for:**
- Long tool execution times
- MCP server connection failures
- Repeated API retries
- Compaction frequency

### API Usage Tracking

**Via `history.jsonl`:**

```bash
# Total tokens used
jq -s 'map(.input_tokens // 0) | add' ~/.codex/history.jsonl

# Cost estimation (gpt-5-codex: $3/1M input, $15/1M output)
jq -s '
  (map(.input_tokens // 0) | add) * 3 / 1000000 +
  (map(.output_tokens // 0) | add) * 15 / 1000000
' ~/.codex/history.jsonl
```

**Via OpenAI Dashboard:**

https://platform.openai.com/usage

---

## Cost Optimization

### 1. Choose Cost-Effective Models

| Task Type | Recommended Model | Why |
|-----------|------------------|-----|
| Simple edits | gpt-5-codex | Fast, accurate, good value |
| Complex reasoning | o3 (medium effort) | Balance quality & cost |
| Batch analysis | gpt-4o | Cheaper Chat Completions |
| Long conversations | gpt-5-codex | Good caching efficiency |

### 2. Minimize Tool Calls

**Guidance in AGENTS.md:**

```markdown
## Tool Usage Guidelines

- Read multiple files in parallel when needed
- Use grep_files to find code before reading specific files
- Batch shell commands (e.g., `npm test && npm build`)
- Avoid redundant reads (remember file contents from earlier)
```

### 3. Reduce Output Verbosity

```toml
model = "gpt-5"
model_verbosity = "low"  # Shorter responses
```

**Trade-off:** Lower verbosity = less detailed explanations.

### 4. Compact Aggressively

```toml
auto_compact_token_limit = 60000  # Compact at 60K tokens
```

**Trade-off:** More compaction API calls, but lower per-turn cost.

---

## Quality Optimization

### 1. Provide Rich Context

**Good AGENTS.md:**

```markdown
# Project Context

This is a Rust web service using actix-web and PostgreSQL.

## Architecture

- `src/handlers/` - HTTP request handlers
- `src/db/` - Database models and queries
- `src/auth/` - Authentication middleware

## Coding Standards

- Use `async fn` for handlers
- Database queries should use prepared statements
- All errors should implement `std::error::Error`
```

**Impact:** Better responses, fewer clarifying questions, more accurate code.

### 2. Use Higher Reasoning Effort

For complex tasks:

```bash
codex --model o3 --config model_reasoning_effort="high" "Refactor for thread safety"
```

**Impact:** Deeper analysis, better solutions, but slower & more expensive.

### 3. Enable Reasoning Summaries

```toml
model = "o3"
model_reasoning_summary = "detailed"
```

See the model's thinking process (helpful for debugging responses).

### 4. Iterative Refinement

Rather than one massive prompt, iterate:

```bash
# Turn 1
codex "Draft a refactoring plan for the auth module"

# Turn 2 (after reviewing plan)
"Implement step 1: Extract authentication logic to a trait"

# Turn 3
"Add tests for the new AuthService trait"
```

**Impact:** Better quality through incremental improvements.

---

## Benchmark Scenarios

### Scenario 1: Quick Fix (Optimize for Speed)

**Task:** Fix a linting error

**Configuration:**
```toml
model = "gpt-5-codex"
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

**Command:**
```bash
codex --full-auto "Fix all linting errors in src/"
```

**Expected:** 5-15 seconds

---

### Scenario 2: Code Review (Optimize for Quality)

**Task:** Review PR for security issues

**Configuration:**
```toml
model = "o3"
model_reasoning_effort = "high"
model_reasoning_summary = "detailed"
sandbox_mode = "read-only"
```

**Command:**
```bash
codex --model o3 --config model_reasoning_effort="high" "Review changes for security vulnerabilities"
```

**Expected:** 30-120 seconds (thorough analysis)

---

### Scenario 3: Large Refactoring (Optimize for Cost)

**Task:** Refactor entire module

**Configuration:**
```toml
model = "gpt-5-codex"
auto_compact_token_limit = 60000  # Aggressive compaction
approval_policy = "on-request"
```

**Command:**
```bash
codex --full-auto "Refactor auth module to use dependency injection"
```

**Expected:** Multiple turns, 2-5 minutes total

---

## Advanced Techniques

### 1. Parallel Sessions

Run multiple Codex instances for independent tasks:

```bash
# Terminal 1
codex --cd frontend "Update API client"

# Terminal 2
codex --cd backend "Add new endpoint"
```

**Pro:** Faster completion for multi-part projects

**Con:** No shared context between sessions

### 2. Pre-warming MCP Servers

If using slow-starting MCP servers:

```bash
# Start servers in advance
codex mcp list  # Triggers server connections

# Then use Codex
codex "Use the docs server to search for authentication"
```

### 3. Custom Reasoning Prompts

**Advanced:** Customize via `GEMINI_SYSTEM_MD` (experimental)

```bash
export GEMINI_SYSTEM_MD="$HOME/.codex/custom-prompt.md"
codex
```

Add domain-specific reasoning guidance.

---

## References

- [Config Reference](./config.md) - All tunable parameters
- [Model Selection](./faq.md#which-models-are-supported) - Model capabilities
- [Compaction Internals](./state-management.md#compaction) - How compression works
- [Tool Performance](./tool-system.md#tool-performance) - Tool-specific optimization
