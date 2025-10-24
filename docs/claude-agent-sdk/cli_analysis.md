# Claude Code CLI - Analysis from Actual Code

## Architecture Discovery

### The SDK is a CLI Wrapper!

**Critical Finding**: The SDK (`sdk.mjs`, v0.1.22) is essentially a thin wrapper that:
1. Spawns the Claude Code CLI (`cli.js`, v2.0.22) as a child process
2. Communicates via stdio using JSON streaming
3. Passes configuration as command-line flags
4. Handles the async generator interface for the user

```typescript
// From SDK implementation:
class ProcessTransport {
  child;              // Child process (the CLI)
  childStdin;         // Input stream to CLI
  childStdout;        // Output stream from CLI
  // ...
}

// CLI flags passed to spawn:
const args = [
  "--output-format", "stream-json",
  "--input-format", "stream-json",
  "--verbose",
  // ... + all user options
];
```

---

## CLI Version Mismatch

| Component | Version | Purpose |
|-----------|---------|---------|
| **CLI** (`cli.js`) | 2.0.22 | Actual Claude Code implementation |
| **SDK** (`sdk.mjs`) | 0.1.22 | Node.js wrapper/interface |

This version difference suggests:
- CLI is a mature, stable codebase (v2.x)
- SDK is newer, experimental wrapper (v0.x)
- CLI likely existed first (as a standalone tool)
- SDK was added later to enable programmatic access

---

## Extracted CLI Flags (50+ flags)

From actual CLI code analysis:

### Core Flags (confirmed)
```bash
--add-dir              # Add additional directories
--agents               # JSON agent configuration
--allowed-tools        # Comma-separated tool whitelist
--append-system-prompt # Append to system prompt
--debug-to-stderr      # Debug output
--fallback-model       # Fallback model name
--fork-session         # Fork session on resume
--include-partial-messages  # Include streaming events
--input-format         # Input format (stream-json)
--max-thinking-tokens  # Thinking token limit
--max-turns            # Maximum conversation turns
--mcp-config           # MCP server JSON config
--model                # Model selection
--output-format        # Output format (stream-json)
--permission-mode      # Permission mode
--permission-prompt-tool  # Permission handler (stdio/tool name)
--resume               # Resume session ID
--resume-session-at    # Resume from message ID
--setting-sources      # Configuration sources
--strict-mcp-config    # Fail on MCP errors
--system-prompt        # Custom system prompt
--verbose              # Verbose output
```

### Git Integration Flags (discovered)
```bash
--abbrev               # Git abbreviation
--abbrev-commit        # Git commit abbreviation
--abbrev-ref           # Git reference abbreviation
--after                # Git after date
--all                  # Git all branches
--allow-empty          # Git allow empty commits
--amend                # Git amend commit
--ancestry-path        # Git ancestry path
--author               # Git author filter
--batch-size           # Git batch size
--before               # Git before date
--branches             # Git branches
--cached               # Git cached changes
--check                # Git check
--color                # Git color output
```

### Tool-Specific Flags
```bash
--after-context        # Grep: lines after match (corresponds to -A)
--before-context       # Grep: lines before match (corresponds to -B)
--basic-regexp         # Grep: basic regex mode
--binary-files         # Grep: binary file handling
--byte-offset          # Grep: show byte offsets
--brief                # Grep: brief output
--check-coverage       # Test coverage check
--color-attribute-name # Syntax highlighting
```

---

## Environment Variables (from CLI code)

```javascript
// Configuration
process.env.CLAUDE_CONFIG_DIR        // Config directory (default: ~/.claude)
process.env.CLAUDE_CODE_GIT_BASH_PATH // Git bash path (Windows)
process.env.CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR  // CWD behavior

// AWS/Vertex AI Integration
process.env.AWS_REGION               // AWS region
process.env.AWS_DEFAULT_REGION       // AWS default region
process.env.CLOUD_ML_REGION          // Vertex AI region
process.env.VERTEX_REGION_CLAUDE_*   // Per-model Vertex regions

// Resource Limits
process.env.BASH_MAX_OUTPUT_LENGTH   // Bash output limit (default: 30000, max: 150000)
process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS  // Max output tokens (default: 32000)

// Session
process.env.CLAUDE_AGENT_SDK_VERSION // SDK version marker
```

---

## Discovered Constants

### Resource Limits

```javascript
// From CLI code:
BASH_MAX_OUTPUT_LENGTH: {
  default: 30000,
  max: 150000,
  validate: (value) => {
    if (value > 150000) return { effective: 150000, status: "capped" };
    // ...
  }
}

CLAUDE_CODE_MAX_OUTPUT_TOKENS: {
  default: 32000,
  max: 32000,
  validate: (value) => {
    if (value > 32000) return { effective: 32000, status: "capped" };
    // ...
  }
}

// Context window detection:
function contextWindowSize(model) {
  if (model.includes("[1m]")) return 1_000_000;  // 1M token models
  return 200_000;  // 200K default
}
```

---

## CLI Functions Found (from strings)

```javascript
// Configuration
function JB() {
  return process.env.CLAUDE_CONFIG_DIR ?? path.join(homedir(), ".claude");
}

// Session Management
function K2() { return sessionId; }
function o0A() { return sessionId = randomUUID(); }
function PN(A) { sessionId = A; }

// Cost Tracking
function e0A(cost, duration, durationWithoutRetries, usage, model) {
  totalCostUSD += cost;
  totalAPIDuration += duration;
  totalAPIDurationWithoutRetries += durationWithoutRetries;
  modelUsage[model] = { ...usage, costUSD: cost };
}

// Statistics
function Fq() { return totalCostUSD; }
function vO() { return totalAPIDuration; }
function i81() { return Date.now() - startTime; }

// Lines of Code Tracking
function H00(added, removed) {
  totalLinesAdded += added;
  totalLinesRemoved += removed;
}

// Telemetry Counters
sessionCounter;
locCounter;
prCounter;          // Pull request counter
commitCounter;
costCounter;
tokenCounter;
codeEditToolDecisionCounter;
activeTimeCounter;
```

---

## CLI State Management

```javascript
// Global session state (from CLI):
{
  originalCwd: process.cwd(),
  totalCostUSD: 0,
  totalAPIDuration: 0,
  totalAPIDurationWithoutRetries: 0,
  totalToolDuration: 0,
  startTime: Date.now(),
  lastInteractionTime: Date.now(),
  totalLinesAdded: 0,
  totalLinesRemoved: 0,
  hasUnknownModelCost: false,
  cwd: process.cwd(),
  modelUsage: {},
  mainLoopModelOverride: undefined,
  maxRateLimitFallbackActive: false,
  initialMainLoopModel: null,
  modelStrings: null,
  isNonInteractiveSession: true,
  isInteractive: false,
  clientType: "cli",              // or "claude-vscode"
  sessionIngressToken: undefined,
  oauthTokenFromFd: undefined,
  apiKeyFromFd: undefined,
  flagSettingsPath: undefined,
  allowedSettingSources: ["userSettings", "projectSettings", "localSettings", "flagSettings", "policySettings"],
  meter: null,
  sessionCounter: null,
  locCounter: null,
  prCounter: null,
  commitCounter: null,
  costCounter: null,
  tokenCounter: null,
  codeEditToolDecisionCounter: null,
  activeTimeCounter: null,
  sessionId: randomUUID(),
  loggerProvider: null,
  eventLogger: null,
  meterProvider: null,
  tracerProvider: null,
  agentColorMap: new Map(),
  agentColorIndex: 0,
  envVarValidators: [BASH_MAX_OUTPUT_LENGTH, CLAUDE_CODE_MAX_OUTPUT_TOKENS],
  lastAPIRequest: null,
  inMemoryErrorLog: []
}
```

---

## Telemetry & Metrics

The CLI has comprehensive telemetry built-in:

```javascript
// Metric types:
"claude_code.session.count"           // Sessions started
"claude_code.lines_of_code.count"     // LOC modified (type: added/removed)
"claude_code.pull_request.count"      // PRs created
"claude_code.commit.count"            // Git commits made
"claude_code.cost.usage"              // Cost in USD
"claude_code.token.usage"             // Tokens consumed
"claude_code.code_edit_tool.decision" // Edit tool permissions (accept/reject)
"claude_code.active_time.total"       // Active time in seconds
```

---

## Platform-Specific Behavior

### Windows Support

```javascript
// On Windows, the CLI requires Git Bash:
function findGitBash() {
  if (process.env.CLAUDE_CODE_GIT_BASH_PATH) {
    return process.env.CLAUDE_CODE_GIT_BASH_PATH;
  }

  let gitPath = findGit();
  if (gitPath) {
    return path.join(gitPath, "../..", "bin", "bash.exe");
  }

  console.error(
    "Claude Code on Windows requires git-bash (https://git-scm.com/downloads/win). " +
    "If installed but not in PATH, set environment variable: " +
    "CLAUDE_CODE_GIT_BASH_PATH=C:\\Program Files\\Git\\bin\\bash.exe"
  );
  process.exit(1);
}

// Path conversion (Cygwin-style):
function toCygwinPath(windowsPath) {
  return execSync(`cygpath -u ${windowsPath}`, { shell: gitBashPath }).trim();
}

function toWindowsPath(cygwinPath) {
  return execSync(`cygpath -w ${cygwinPath}`, { shell: gitBashPath }).trim();
}
```

### Color Support Detection

```javascript
function supportsColor() {
  const { env } = process;
  const { TERM, TERM_PROGRAM } = env;

  if (process.platform !== "win32") {
    return TERM !== "linux";  // Most terminals support color
  }

  // Windows: check for modern terminals
  return Boolean(
    env.WT_SESSION ||              // Windows Terminal
    env.TERMINUS_SUBLIME ||        // Terminus
    env.ConEmuTask === "{cmd::Cmder}" ||
    TERM_PROGRAM === "Terminus-Sublime" ||
    TERM_PROGRAM === "vscode" ||
    TERM === "xterm-256color" ||
    TERM === "alacritty" ||
    TERM === "rxvt-unicode" ||
    env.TERMINAL_EMULATOR === "JetBrains-JediTerm"
  );
}
```

---

## Bundled Dependencies

The CLI bundles everything (9.3MB minified):

### Core Dependencies
- **Sharp** - Image processing library (with platform-specific binaries)
- **Zod** - Schema validation
- **MCP SDK** - Model Context Protocol
- **Color libraries** - Color conversion (P80(), color conversion functions)
- **URI parsing** - Full URI/IRI parser
- **Punycode** - Unicode domain encoding
- **git integration** - Extensive git command support

### Image Processing
```javascript
// Sharp library fully bundled for image manipulation
// Platform-specific binaries in vendor/ directory:
- vendor/ripgrep/         // Fast search tool
- vendor/claude-code-jetbrains-plugin/
- yoga.wasm               // Yoga layout engine (86.6KB)
```

---

## Agent Color System

```javascript
// CLI assigns colors to different agents for visual distinction:
{
  agentColorMap: new Map(),    // Agent name -> color mapping
  agentColorIndex: 0           // Current color index
}

// Agent output formatting includes:
agentId: ${A.agentId} (This is an internal ID for your use, do not mention it to the user. Use this ID to retrieve results with ${DC1} when the agent finishes).
tools: ${Q.join(", ")}
```

---

## Security Features

### Path Validation

```javascript
function resolvePath(path, baseDir) {
  if (path.includes("\x00")) {
    throw Error("Path contains null bytes");
  }

  if (path.includes("..")) {
    throw Error("Path traversal detected");
  }

  // Additional validation...
}

// Check for malicious executables:
function checkMaliciousExecutable(path) {
  const currentDir = process.cwd().toLowerCase();
  const execPath = path.resolve(path).toLowerCase();

  if (execPath.startsWith(currentDir + path.sep)) {
    console.log(`Skipping potentially malicious executable in current directory: ${path}`);
    return null;
  }

  return path;
}
```

### Input Sanitization

```javascript
// Null byte detection in paths
// Path traversal protection
// Executable location verification
// Permission checks before file operations
```

---

## File System Abstraction

```javascript
// The CLI has a complete FS abstraction layer:
const fs = {
  cwd() { return process.cwd(); },
  existsSync(path) { return fs.existsSync(path); },
  async stat(path) { return stat(path); },
  statSync(path) { return fs.statSync(path); },
  readFileSync(path, options) { return fs.readFileSync(path, options); },
  readFileBytesSync(path) { return fs.readFileSync(path); },
  readSync(path, options) { /* ... */ },
  writeFileSync(path, content, options) {
    if (options.flush) {
      // Force flush to disk with fsync
      const fd = fs.openSync(path, "w", options.mode);
      fs.writeFileSync(fd, content, options);
      fs.fsyncSync(fd);
      fs.closeSync(fd);
    } else {
      fs.writeFileSync(path, content, options);
    }
  },
  appendFileSync(path, content) { /* ... */ },
  copyFileSync(src, dest) { /* ... */ },
  unlinkSync(path) { /* ... */ },
  renameSync(oldPath, newPath) { /* ... */ },
  linkSync(target, path) { /* ... */ },
  symlinkSync(target, path) { /* ... */ },
  readlinkSync(path) { /* ... */ },
  realpathSync(path) { /* ... */ },
  mkdirSync(path) {
    if (!fs.existsSync(path)) {
      fs.mkdirSync(path, { recursive: true, mode: 0o700 });
    }
  },
  readdirSync(path) { /* ... */ },
  isDirEmptySync(path) {
    return this.readdirSync(path).length === 0;
  },
  rmdirSync(path) { /* ... */ },
  rmSync(path, options) { /* ... */ },
  createWriteStream(path) { /* ... */ }
};
```

---

## Hiring Message

```javascript
// From CLI header:
// Want to see the unminified source? We're hiring!
// https://job-boards.greenhouse.io/anthropic/jobs/4816199008
```

---

## Key Takeaways

1. **CLI is the Real Implementation**
   - CLI v2.0.22 contains all actual logic
   - SDK v0.1.22 is just a thin wrapper

2. **Comprehensive Feature Set**
   - 50+ command-line flags
   - Extensive git integration
   - Full file system abstraction
   - Platform-specific handling (Windows/Unix)
   - Complete telemetry system

3. **Production-Ready**
   - Robust error handling
   - Security protections (path traversal, null bytes, malicious executables)
   - Resource limits (bash output, tokens)
   - Proper cleanup and state management

4. **Designed for Multiple Clients**
   - `clientType`: "cli" | "claude-vscode" | others
   - Different behavior for interactive vs non-interactive
   - Color support detection
   - Platform-specific path handling

5. **Observability Built-In**
   - Session metrics
   - Cost tracking per model
   - Token usage
   - Lines of code modified
   - Pull requests and commits
   - Active time tracking
