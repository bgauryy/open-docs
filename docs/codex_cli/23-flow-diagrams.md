# Codex CLI - System Flow Diagrams

**Comprehensive visual guides to understanding Codex internal flows**

---

## Table of Contents
- [Prompt Processing Flow](#prompt-processing-flow)
- [Tool Execution Flow](#tool-execution-flow)
- [Session Lifecycle](#session-lifecycle)
- [Approval Flow](#approval-flow)
- [Authentication Flow](#authentication-flow)
- [Configuration Loading Flow](#configuration-loading-flow)
- [MCP Connection Flow](#mcp-connection-flow)
- [State Persistence Flow](#state-persistence-flow)
- [Sandbox Enforcement Flow](#sandbox-enforcement-flow)
- [Event Processing Flow](#event-processing-flow)

---

## Prompt Processing Flow

### Complete Lifecycle: User Input → Agent Response

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER SUBMITS PROMPT                          │
│                  "Refactor this function"                        │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. INPUT VALIDATION & PREPARATION                               │
│    ├─ Validate prompt not empty                                 │
│    ├─ Attach images if provided                                 │
│    ├─ Check @ file references                                   │
│    └─ Expand @ to full file paths                               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. CONTEXT GATHERING                                            │
│    ├─ Load AGENTS.md files (global + project)                   │
│    ├─ Check project documentation                               │
│    ├─ Load user instructions from config                        │
│    ├─ Check conversation history (if resuming)                  │
│    └─ Gather file contents for @ references                     │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. SYSTEM PROMPT COMPOSITION                                    │
│    ├─ Load base system prompt (prompt.md)                       │
│    ├─ Append AGENTS.md content (hierarchy order)                │
│    ├─ Add user instructions                                     │
│    ├─ Inject tool definitions                                   │
│    ├─ Add MCP tools/resources                                   │
│    └─ Set approval policy context                               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. TOKEN OPTIMIZATION                                           │
│    ├─ Calculate total token count                               │
│    ├─ Check against context window limit                        │
│    ├─ Apply prompt caching (if supported)                       │
│    ├─ Compress conversation history if needed                   │
│    └─ Warn if approaching limits                                │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 5. API REQUEST CONSTRUCTION                                     │
│    ├─ Build messages array                                      │
│    │  ├─ System message                                         │
│    │  ├─ Previous conversation (if any)                         │
│    │  └─ User message with attachments                          │
│    ├─ Add tools array                                           │
│    ├─ Set model parameters                                      │
│    │  ├─ Temperature                                            │
│    │  ├─ Max tokens                                             │
│    │  ├─ Reasoning effort (if o-series)                         │
│    │  └─ Response format (if structured output)                 │
│    └─ Add streaming configuration                               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 6. LLM API CALL (Streaming)                                     │
│    ├─ Send POST request to provider                             │
│    ├─ Establish SSE connection                                  │
│    └─ Begin receiving chunks                                    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 7. STREAM PROCESSING                                            │
│    │                                                             │
│    ├─ For each chunk received:                                  │
│    │  ├─ Parse SSE event                                        │
│    │  ├─ Extract delta                                          │
│    │  └─ Route to handler                                       │
│    │                                                             │
│    ├─ Text Delta:                                               │
│    │  ├─ Append to buffer                                       │
│    │  └─ Update UI in real-time                                 │
│    │                                                             │
│    ├─ Tool Call Delta:                                          │
│    │  ├─ Parse tool name                                        │
│    │  ├─ Accumulate arguments                                   │
│    │  └─ When complete → Route to Tool Execution Flow           │
│    │                                                             │
│    └─ Reasoning Delta (o-series):                               │
│       ├─ Accumulate reasoning                                   │
│       └─ Display summary when complete                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
      ┌───────────┴───────────┐
      │                       │
┌─────▼──────┐          ┌────▼─────────────────┐
│ Text Only  │          │ Tool Calls Present   │
│            │          │                      │
│ Skip to    │          │ Execute tools        │
│ step 9     │          │ (see Tool Flow)      │
└─────┬──────┘          └────┬─────────────────┘
      │                      │
      │                      │
      │                ┌─────▼─────────────────┐
      │                │ 8. TOOL RESULTS       │
      │                │    ├─ Collect outputs │
      │                │    ├─ Format results  │
      │                │    └─ Send to LLM     │
      │                └─────┬─────────────────┘
      │                      │
      │                ┌─────▼─────────────────┐
      │                │ Agent continues with  │
      │                │ tool results...       │
      │                │ (may call more tools) │
      │                └─────┬─────────────────┘
      └──────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 9. FINAL RESPONSE RECEIVED                                      │
│    ├─ Stream complete                                           │
│    ├─ Extract finish reason                                     │
│    ├─ Collect token usage                                       │
│    └─ Validate response (if structured output)                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 10. RESPONSE POST-PROCESSING                                    │
│     ├─ Parse markdown if present                                │
│     ├─ Extract code blocks                                      │
│     ├─ Identify file references                                 │
│     ├─ Format for display                                       │
│     └─ Update conversation history                              │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 11. STATE PERSISTENCE                                           │
│     ├─ Save conversation to session file                        │
│     ├─ Update session metadata                                  │
│     ├─ Cache computed context                                   │
│     └─ Log usage statistics                                     │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 12. DISPLAY TO USER                                             │
│     ├─ Render markdown                                          │
│     ├─ Show token usage                                         │
│     └─ Wait for next input                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tool Execution Flow

### From Tool Call → Result → Back to Agent

```
┌─────────────────────────────────────────────────────────────────┐
│            AGENT CALLS TOOL (from LLM response)                  │
│     {"name": "bash", "arguments": {"command": "ls -la"}}         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. TOOL CALL VALIDATION                                         │
│    ├─ Parse tool call JSON                                      │
│    ├─ Verify tool exists in registry                            │
│    ├─ Validate required arguments present                       │
│    └─ Check argument types match schema                         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. PERMISSION CHECK                                             │
│    ├─ Check sandbox mode (read-only/workspace-write/full)       │
│    ├─ Check approval policy (untrusted/on-failure/on-request)   │
│    ├─ Validate against command safety rules                     │
│    └─ Check if escalated permissions requested                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼───────┐    ┌──────▼───────────┐
│ Auto-Approved │    │ Approval Needed  │
│               │    │                  │
│ • Safe cmd    │    │ • Dangerous      │
│ • Within      │    │ • Outside scope  │
│   sandbox     │    │ • Escalation     │
└───────┬───────┘    └──────┬───────────┘
        │                   │
        │              ┌────▼─────────────────────┐
        │              │ 3. REQUEST APPROVAL      │
        │              │    ├─ Show command       │
        │              │    ├─ Show justification │
        │              │    ├─ Display risks      │
        │              │    └─ Wait for user      │
        │              └────┬─────────────────────┘
        │                   │
        │         ┌─────────┴──────────┐
        │         │                    │
        │    ┌────▼─────┐       ┌─────▼────┐
        │    │ Approved │       │ Denied   │
        │    └────┬─────┘       └─────┬────┘
        │         │                   │
        └─────────┘                   │
                  │                   │
┌─────────────────▼───────────────────┼─────────────────────────────┐
│ 4. PATH VALIDATION (for file tools) │                             │
│    ├─ Resolve relative paths        │                             │
│    ├─ Check within allowed roots    │                             │
│    ├─ Verify file exists (if read)  │                             │
│    └─ Check write permissions       │                             │
└─────────────────┬────────────────────┘                             │
                  │                                                  │
┌─────────────────▼───────────────────────────────────────────────┐ │
│ 5. SANDBOX PREPARATION (for bash tool)                          │ │
│    ├─ Determine sandbox mode                                    │ │
│    ├─ Generate sandbox profile                                  │ │
│    │  ├─ macOS: Seatbelt .sbpl                                  │ │
│    │  └─ Linux: Landlock rules                                  │ │
│    ├─ Set allowed paths                                         │ │
│    ├─ Set network policy                                        │ │
│    └─ Set resource limits (timeout, memory)                     │ │
└─────────────────┬───────────────────────────────────────────────┘ │
                  │                                                  │
┌─────────────────▼───────────────────────────────────────────────┐ │
│ 6. TOOL EXECUTION                                               │ │
│    │                                                             │ │
│    ├─ BASH TOOL:                                                │ │
│    │  ├─ Wrap command with sandbox                              │ │
│    │  │  └─ sandbox-exec (macOS) or landlock (Linux)            │ │
│    │  ├─ Spawn process                                          │ │
│    │  ├─ Stream stdout/stderr                                   │ │
│    │  ├─ Monitor for timeout                                    │ │
│    │  └─ Wait for completion                                    │ │
│    │                                                             │ │
│    ├─ READ TOOL:                                                │ │
│    │  ├─ Open file                                              │ │
│    │  ├─ Read specified lines (or all)                          │ │
│    │  ├─ Format with line numbers                               │ │
│    │  └─ Truncate if too long                                   │ │
│    │                                                             │ │
│    ├─ WRITE/EDIT TOOL:                                          │ │
│    │  ├─ Validate write permission                              │ │
│    │  ├─ Create backup if exists                                │ │
│    │  ├─ Perform operation                                      │ │
│    │  └─ Verify write succeeded                                 │ │
│    │                                                             │ │
│    └─ MCP TOOL:                                                 │ │
│       ├─ Route to MCP connection                                │ │
│       ├─ Send tool call via MCP protocol                        │ │
│       ├─ Wait for response                                      │ │
│       └─ Deserialize result                                     │ │
└─────────────────┬───────────────────────────────────────────────┘ │
                  │                                                  │
        ┌─────────┴─────────┐                                        │
        │                   │                                        │
┌───────▼────────┐   ┌──────▼────────┐                              │
│ Success        │   │ Failure       │                              │
│                │   │               │                              │
│ • Exit code 0  │   │ • Non-zero    │                              │
│ • Output       │   │ • Error msg   │◄─────────────────────────────┘
│   captured     │   │ • Stack trace │  (denied → error result)
└───────┬────────┘   └──────┬────────┘
        │                   │
        └─────────┬─────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 7. RESULT FORMATTING                                            │
│    ├─ Format output for LLM                                     │
│    ├─ Truncate if exceeds limit                                 │
│    ├─ Add metadata (exit code, duration)                        │
│    └─ Structure as tool result                                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 8. SEND RESULT TO AGENT                                         │
│    ├─ Add to conversation as tool result                        │
│    ├─ Associate with tool call ID                               │
│    └─ Trigger next agent turn                                   │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 9. AGENT PROCESSES RESULT                                       │
│    ├─ Read tool output                                          │
│    ├─ Decide next action                                        │
│    │  ├─ Call more tools                                        │
│    │  ├─ Ask clarifying questions                               │
│    │  └─ Provide final response                                 │
│    └─ Continue conversation                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Session Lifecycle

### From Start → Active → Persisted → Resume

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER STARTS CODEX                             │
│                  codex "initial prompt"                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. INITIALIZATION                                               │
│    ├─ Load configuration                                        │
│    ├─ Authenticate                                              │
│    ├─ Detect working directory                                  │
│    ├─ Check git repository                                      │
│    └─ Initialize logging                                        │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. SESSION CREATION                                             │
│    ├─ Generate session ID (UUID)                                │
│    ├─ Create session directory                                  │
│    │  └─ ~/.codex/sessions/{session_id}/                        │
│    ├─ Initialize conversation manager                           │
│    ├─ Create metadata                                           │
│    │  ├─ Created timestamp                                      │
│    │  ├─ Model used                                             │
│    │  ├─ Working directory                                      │
│    │  ├─ Approval policy                                        │
│    │  └─ Sandbox mode                                           │
│    └─ Set up event listeners                                    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. CONVERSATION LOOP (Active Session)                           │
│    │                                                             │
│    ├─ User sends prompt                                         │
│    ├─ Agent processes (see Prompt Flow)                         │
│    ├─ Tools may execute (see Tool Flow)                         │
│    ├─ Agent responds                                            │
│    │                                                             │
│    └─ After each turn:                                          │
│       ├─ Persist conversation state                             │
│       ├─ Update token usage                                     │
│       ├─ Save to session file                                   │
│       └─ Update last_modified timestamp                         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ (multiple turns...)
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐   ┌──────▼────────┐
│ User Exits     │   │ Error/Crash   │
└───────┬────────┘   └──────┬────────┘
        │                   │
        └─────────┬─────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. SESSION FINALIZATION                                         │
│    ├─ Final state save                                          │
│    ├─ Write complete conversation                               │
│    ├─ Save session metadata                                     │
│    │  ├─ Total turns                                            │
│    │  ├─ Total tokens used                                      │
│    │  ├─ Duration                                               │
│    │  └─ Exit reason                                            │
│    ├─ Close file handles                                        │
│    └─ Clean up temp files                                       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 5. PERSISTED STATE                                              │
│                                                                  │
│    ~/.codex/sessions/{session_id}/                              │
│    ├── session.json          # Metadata                         │
│    ├── conversation.json     # Full conversation                │
│    ├── state.json           # Cached state                      │
│    └── logs/                # Session logs                      │
│        └── events.log                                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ (later...)
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 6. SESSION RESUME                                               │
│    │                                                             │
│    ├─ User runs: codex resume {session_id}                      │
│    │                                                             │
│    ├─ Load session metadata                                     │
│    ├─ Validate session exists                                   │
│    ├─ Check compatibility                                       │
│    │  ├─ Model still available                                  │
│    │  └─ Config compatible                                      │
│    ├─ Restore conversation state                                │
│    │  ├─ Load conversation history                              │
│    │  ├─ Restore context                                        │
│    │  └─ Rebuild state                                          │
│    ├─ Display session info                                      │
│    │  ├─ Created date                                           │
│    │  ├─ Last modified                                          │
│    │  ├─ Number of turns                                        │
│    │  └─ Preview last messages                                  │
│    └─ Resume conversation loop (back to step 3)                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Approval Flow

### Decision Tree for Command/Edit Approvals

```
┌─────────────────────────────────────────────────────────────────┐
│           AGENT WANTS TO EXECUTE ACTION                          │
│       (Command, File Edit, Network Request)                      │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. CHECK APPROVAL POLICY                                        │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┼─────────┬─────────┬──────────┐
        │         │         │         │          │
┌───────▼──┐ ┌────▼────┐ ┌─▼────┐ ┌──▼──────┐ ┌─▼─────┐
│untrusted │ │on-failure│ │on-req│ │ never   │ │ yolo  │
│          │ │         │ │uest  │ │         │ │       │
└───────┬──┘ └────┬────┘ └─┬────┘ └──┬──────┘ └─┬─────┘
        │         │         │         │          │
        │         │         │         │          │
┌───────▼─────────▼─────────▼─────────▼──────────▼───────────────┐
│ 2. ACTION CLASSIFICATION                                        │
│    ├─ Command safety check                                      │
│    │  ├─ Read-only: ls, cat, git status                         │
│    │  ├─ Modify: npm install, git commit                        │
│    │  └─ Dangerous: rm -rf, sudo, curl (network)                │
│    ├─ File operation check                                      │
│    │  ├─ Read: Always safe                                      │
│    │  ├─ Write: Check path                                      │
│    │  └─ Delete: Always requires approval                       │
│    └─ Scope check                                               │
│       ├─ In workspace: Allowed in workspace-write               │
│       ├─ In temp: Usually allowed                               │
│       └─ Outside: Requires approval                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼─────────┐   ┌─────▼────────────────────┐
│ UNTRUSTED MODE  │   │ ON-FAILURE MODE          │
│                 │   │                          │
│ Safe commands:  │   │ Try in sandbox:          │
│ ├─ Auto-execute │   │ ├─ Execute sandboxed     │
│ └─ Show in UI   │   │ ├─ If succeeds: Done     │
│                 │   │ └─ If fails:             │
│ Unsafe:         │   │    ├─ Show error         │
│ ├─ Show modal   │   │    ├─ Offer to retry     │
│ ├─ Display cmd  │   │    │   unsandboxed        │
│ ├─ Show risks   │   │    └─ Require approval   │
│ └─ Wait for y/n │   └──────────────────────────┘
└───────┬─────────┘
        │
        │
┌───────▼──────────────────────────────────────────────────────┐
│ 3. APPROVAL UI (if needed)                                    │
│                                                               │
│    ╭──────────────────────────────────────────────────────╮  │
│    │ ⚠ Approval Required                                  │  │
│    ├──────────────────────────────────────────────────────┤  │
│    │                                                       │  │
│    │ Command: npm install express                         │  │
│    │                                                       │  │
│    │ Reason: Package installation requires network        │  │
│    │                                                       │  │
│    │ Risks:                                               │  │
│    │  • Downloads code from npmjs.com                     │  │
│    │  • Modifies package.json and lock file               │  │
│    │  • Adds files to node_modules/                       │  │
│    │                                                       │  │
│    │ [A]llow Once  [D]eny  [N]ever Ask  [E]xplain        │  │
│    ╰──────────────────────────────────────────────────────╯  │
└───────┬───────────────────────────────────────────────────────┘
        │
        │ User response
        │
        ┌─────┴──────┬──────────┬────────────┐
        │            │          │            │
┌───────▼─────┐ ┌────▼────┐ ┌──▼──────┐ ┌──▼────────┐
│ Allow Once  │ │ Deny    │ │Never Ask│ │ Explain   │
│             │ │         │ │(session)│ │           │
│ Execute     │ │ Cancel  │ │ Add to  │ │ Show more │
│ action      │ │ action  │ │ auto-   │ │ details   │
│             │ │         │ │ approve │ │ then loop │
└───────┬─────┘ └────┬────┘ └──┬──────┘ └───────────┘
        │            │         │
        │            │         └─→ Add to allow list
        │            │             Execute action
        │            │
        │            └─→ Return error to agent
        │                "User denied operation"
        │
        └─→ Execute action
            Return result to agent
```

---

## Authentication Flow

### OAuth2 Device Code Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER RUNS: codex login                       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. REQUEST DEVICE CODE                                          │
│    POST https://auth.openai.com/oauth/device/code               │
│    {                                                             │
│      "client_id": "codex-cli",                                   │
│      "scope": "openai"                                           │
│    }                                                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. RECEIVE DEVICE CODE                                          │
│    {                                                             │
│      "device_code": "ABC123...",                                 │
│      "user_code": "XYZW-1234",                                   │
│      "verification_uri": "https://platform.openai.com/device",   │
│      "expires_in": 900,                                          │
│      "interval": 5                                               │
│    }                                                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. DISPLAY TO USER                                              │
│                                                                  │
│    Please visit: https://platform.openai.com/device             │
│    Enter code: XYZW-1234                                         │
│                                                                  │
│    Waiting for authorization...                                 │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. USER ACTIONS IN BROWSER                                      │
│    ├─ Navigates to verification URI                             │
│    ├─ Enters user code: XYZW-1234                               │
│    ├─ Logs into OpenAI account                                  │
│    ├─ Reviews permissions requested                             │
│    └─ Clicks "Authorize"                                        │
└─────────────────────────────────────────────────────────────────┘
                  │
                  │ (Meanwhile, CLI is polling...)
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 5. POLLING FOR TOKEN (every 5 seconds)                          │
│    POST https://auth.openai.com/oauth/token                     │
│    {                                                             │
│      "grant_type": "urn:ietf:params:oauth:grant-type:device_code"│
│      "device_code": "ABC123...",                                 │
│      "client_id": "codex-cli"                                    │
│    }                                                             │
│                                                                  │
│    Responses:                                                    │
│    ├─ 400 + "authorization_pending" → Continue polling          │
│    ├─ 400 + "slow_down" → Increase interval                     │
│    ├─ 400 + "expired_token" → Start over                        │
│    ├─ 400 + "access_denied" → User denied                       │
│    └─ 200 → Authorization successful!                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 6. RECEIVE TOKENS                                               │
│    {                                                             │
│      "access_token": "eyJhbG...",                                │
│      "refresh_token": "def502...",                               │
│      "expires_in": 3600,                                         │
│      "token_type": "Bearer"                                      │
│    }                                                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 7. PERSIST TOKENS                                               │
│    ├─ Save to ~/.codex/auth.json                                │
│    ├─ Set file permissions to 0600 (Unix)                       │
│    ├─ Store access_token                                        │
│    ├─ Store refresh_token                                       │
│    ├─ Calculate expires_at timestamp                            │
│    └─ Encrypt sensitive data (optional)                         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 8. AUTHENTICATION COMPLETE                                      │
│    ✓ Successfully logged in!                                    │
└─────────────────────────────────────────────────────────────────┘


LATER: Using Access Token
═════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│ 9. USING TOKEN FOR API CALLS                                    │
│    ├─ Load access_token from auth.json                          │
│    ├─ Check if expired (compare expires_at with now)            │
│    └─ Add to Authorization header                               │
│       └─ "Authorization: Bearer eyJhbG..."                       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐   ┌──────▼─────────┐
│ Token Valid    │   │ Token Expired  │
│                │   │                │
│ Use directly   │   │ Refresh first  │
└────────────────┘   └──────┬─────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│ 10. TOKEN REFRESH                                               │
│     POST https://auth.openai.com/oauth/token                    │
│     {                                                            │
│       "grant_type": "refresh_token",                             │
│       "refresh_token": "def502...",                              │
│       "client_id": "codex-cli"                                   │
│     }                                                            │
│                                                                  │
│     Response:                                                    │
│     {                                                            │
│       "access_token": "eyJNew...",  (new token)                  │
│       "refresh_token": "abc123...",  (may be new)                │
│       "expires_in": 3600                                         │
│     }                                                            │
│                                                                  │
│     ├─ Update auth.json with new tokens                         │
│     ├─ Update expires_at timestamp                              │
│     └─ Continue with API call                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Configuration Loading Flow

### Multi-layer Configuration Resolution

```
┌─────────────────────────────────────────────────────────────────┐
│                      CODEX STARTS                                │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ LAYER 1: HARDCODED DEFAULTS                                     │
│                                                                  │
│ Config::default() {                                             │
│   model: "gpt-5-codex",                                          │
│   approval_policy: AskForApproval::Untrusted,                   │
│   sandbox_mode: SandboxMode::ReadOnly,                          │
│   model_context_window: None,  // Auto-detect                   │
│   // ... 50+ default values                                     │
│ }                                                                │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ LAYER 2: CONFIG FILE                                            │
│                                                                  │
│ 1. Find config file:                                            │
│    ├─ Check CODEX_HOME env var                                  │
│    ├─ Default: ~/.codex/config.toml                             │
│    └─ Platform-specific paths                                   │
│                                                                  │
│ 2. If exists:                                                   │
│    ├─ Parse TOML                                                │
│    ├─ Validate schema                                           │
│    └─ Merge with defaults                                       │
│       └─ File values override defaults                          │
│                                                                  │
│ 3. Example overrides:                                           │
│    [config.toml]                                                │
│    model = "gpt-5"                                               │
│    approval_policy = "on-request"                               │
│    sandbox_mode = "workspace-write"                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ LAYER 3: PROFILE (optional)                                     │
│                                                                  │
│ If --profile specified:                                         │
│                                                                  │
│ [config.toml]                                                   │
│ [profiles.production]                                           │
│   model = "gpt-5"                                                │
│   approval_policy = "never"                                     │
│   sandbox_mode = "workspace-write"                              │
│                                                                  │
│ codex --profile production                                      │
│                                                                  │
│ └─ Profile values override base config                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ LAYER 4: ENVIRONMENT VARIABLES                                  │
│                                                                  │
│ Check env vars:                                                 │
│ ├─ OPENAI_API_KEY → Set API key                                 │
│ ├─ CODEX_MODEL → Override model                                 │
│ ├─ CODEX_DISABLE_PROJECT_DOC → Set project_doc_max_bytes = 0    │
│ ├─ CODEX_HOME → Override config directory                       │
│ └─ DEBUG → Enable debug logging                                 │
│                                                                  │
│ Example:                                                         │
│ export CODEX_MODEL="gpt-5"                                       │
│                                                                  │
│ └─ Env values override profile/config/defaults                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ LAYER 5: CLI FLAGS (highest priority)                           │
│                                                                  │
│ Parse command line:                                             │
│ codex \\                                                         │
│   --model gpt-5 \\                                               │
│   --sandbox workspace-write \\                                   │
│   -a on-request \\                                               │
│   "your prompt"                                                 │
│                                                                  │
│ CLI flags override everything                                   │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ FINAL CONFIGURATION                                             │
│                                                                  │
│ Precedence (highest to lowest):                                 │
│ 1. CLI flags                                                    │
│ 2. Environment variables                                        │
│ 3. Profile (if specified)                                       │
│ 4. Config file                                                  │
│ 5. Hardcoded defaults                                           │
│                                                                  │
│ Example result:                                                 │
│ ├─ model: "gpt-5" (from CLI flag)                               │
│ ├─ approval_policy: OnRequest (from CLI flag)                   │
│ ├─ sandbox_mode: WorkspaceWrite (from CLI flag)                 │
│ ├─ api_key: "sk-..." (from env var)                             │
│ ├─ project_doc_max_bytes: 65536 (from config file)              │
│ └─ notify: true (from default)                                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ VALIDATION & POST-PROCESSING                                    │
│ ├─ Validate model exists                                        │
│ ├─ Check API key present                                        │
│ ├─ Verify paths exist                                           │
│ ├─ Resolve relative paths                                       │
│ └─ Apply platform-specific adjustments                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ CONFIGURATION READY                                             │
│ → Pass to Codex core for execution                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## MCP Connection Flow

### Connecting to MCP Servers

```
┌─────────────────────────────────────────────────────────────────┐
│                  CODEX STARTS WITH MCP CONFIG                    │
│                                                                  │
│ [config.toml]                                                   │
│ [mcp_servers.filesystem]                                        │
│   command = "npx"                                               │
│   args = ["-y", "@modelcontextprotocol/server-filesystem", "."] │
│                                                                  │
│ [mcp_servers.github]                                            │
│   command = "node"                                              │
│   args = ["./mcp-server.js"]                                    │
│   env = { GITHUB_TOKEN = "${GITHUB_TOKEN}" }                    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. MCP CONNECTION MANAGER INIT                                  │
│    ├─ Parse mcp_servers configuration                           │
│    ├─ Create connection pool                                    │
│    └─ For each server: Spawn connection                         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ (for each server...)
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. SERVER PROCESS SPAWN                                         │
│    ├─ Resolve command path (npx, node, python, etc.)            │
│    ├─ Prepare arguments                                         │
│    ├─ Substitute environment variables                          │
│    │  └─ ${GITHUB_TOKEN} → actual value                         │
│    ├─ Set up stdio pipes                                        │
│    │  ├─ stdin: to send messages                                │
│    │  └─ stdout: to receive messages                            │
│    ├─ Spawn subprocess                                          │
│    └─ Track process ID                                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. MCP HANDSHAKE                                                │
│    │                                                             │
│    ├─ Send initialize request:                                  │
│    │  {                                                          │
│    │    "jsonrpc": "2.0",                                        │
│    │    "id": 1,                                                 │
│    │    "method": "initialize",                                  │
│    │    "params": {                                              │
│    │      "protocolVersion": "1.0",                              │
│    │      "clientInfo": {                                        │
│    │        "name": "codex-cli",                                 │
│    │        "version": "1.0.0"                                   │
│    │      }                                                      │
│    │    }                                                        │
│    │  }                                                          │
│    │                                                             │
│    ├─ Wait for response (timeout 10s)                           │
│    │                                                             │
│    └─ Receive server capabilities:                              │
│       {                                                          │
│         "protocolVersion": "1.0",                                │
│         "serverInfo": {                                          │
│           "name": "filesystem-server",                           │
│           "version": "0.1.0"                                     │
│         },                                                       │
│         "capabilities": {                                        │
│           "tools": {},                                           │
│           "resources": {}                                        │
│         }                                                        │
│       }                                                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. DISCOVER TOOLS & RESOURCES                                   │
│    │                                                             │
│    ├─ Request tools list:                                       │
│    │  {                                                          │
│    │    "jsonrpc": "2.0",                                        │
│    │    "id": 2,                                                 │
│    │    "method": "tools/list"                                   │
│    │  }                                                          │
│    │                                                             │
│    │  Response:                                                  │
│    │  {                                                          │
│    │    "tools": [                                               │
│    │      {                                                      │
│    │        "name": "read_file",                                 │
│    │        "description": "Read file contents",                 │
│    │        "inputSchema": { ... }                               │
│    │      },                                                     │
│    │      {                                                      │
│    │        "name": "write_file",                                │
│    │        "description": "Write to file",                      │
│    │        "inputSchema": { ... }                               │
│    │      }                                                      │
│    │    ]                                                        │
│    │  }                                                          │
│    │                                                             │
│    ├─ Request resources list:                                   │
│    │  {                                                          │
│    │    "jsonrpc": "2.0",                                        │
│    │    "id": 3,                                                 │
│    │    "method": "resources/list"                               │
│    │  }                                                          │
│    │                                                             │
│    │  Response:                                                  │
│    │  {                                                          │
│    │    "resources": [                                           │
│    │      {                                                      │
│    │        "uri": "file:///README.md",                          │
│    │        "name": "Project README",                            │
│    │        "mimeType": "text/markdown"                          │
│    │      }                                                      │
│    │    ]                                                        │
│    │  }                                                          │
│    │                                                             │
│    └─ Register tools/resources in Codex tool registry           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 5. CONNECTION ACTIVE                                            │
│    ├─ Add to connection pool                                    │
│    ├─ Mark as healthy                                           │
│    ├─ Start heartbeat monitor                                   │
│    └─ Ready to receive tool calls                               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │ (during session...)
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 6. TOOL INVOCATION VIA MCP                                      │
│    │                                                             │
│    ├─ Agent calls MCP tool:                                     │
│    │  Tool call: {                                               │
│    │    "name": "mcp__filesystem__read_file",                    │
│    │    "arguments": {                                           │
│    │      "path": "src/main.rs"                                  │
│    │    }                                                        │
│    │  }                                                          │
│    │                                                             │
│    ├─ Route to MCP connection manager                           │
│    ├─ Find connection for "filesystem" server                   │
│    │                                                             │
│    ├─ Send tool call to server:                                 │
│    │  {                                                          │
│    │    "jsonrpc": "2.0",                                        │
│    │    "id": 100,                                               │
│    │    "method": "tools/call",                                  │
│    │    "params": {                                              │
│    │      "name": "read_file",                                   │
│    │      "arguments": {                                         │
│    │        "path": "src/main.rs"                                │
│    │      }                                                      │
│    │    }                                                        │
│    │  }                                                          │
│    │                                                             │
│    ├─ Wait for response                                         │
│    │                                                             │
│    └─ Receive result:                                           │
│       {                                                          │
│         "content": [                                             │
│           {                                                      │
│             "type": "text",                                      │
│             "text": "fn main() { ... }"                          │
│           }                                                      │
│         ]                                                        │
│       }                                                          │
│                                                                  │
│    └─ Return to agent as tool result                            │
└─────────────────────────────────────────────────────────────────┘


ERROR HANDLING:
═══════════════

┌─────────────────────────────────────────────────────────────────┐
│ CONNECTION FAILURES                                             │
│                                                                  │
│ ├─ Server not responding:                                       │
│ │  ├─ Retry connection (3 attempts)                             │
│ │  ├─ Exponential backoff                                       │
│ │  └─ Mark as unhealthy                                         │
│ │                                                                │
│ ├─ Server crashes:                                              │
│ │  ├─ Detect via process exit                                   │
│ │  ├─ Log error                                                 │
│ │  ├─ Remove from pool                                          │
│ │  └─ Optionally restart                                        │
│ │                                                                │
│ └─ Tool call timeout:                                           │
│    ├─ Default: 30s timeout                                      │
│    ├─ Return error to agent                                     │
│    └─ Connection still usable                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## State Persistence Flow

### How Conversations Are Saved and Restored

```
┌─────────────────────────────────────────────────────────────────┐
│               AFTER EACH TURN IN CONVERSATION                    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. CONVERSATION STATE SNAPSHOT                                  │
│    │                                                             │
│    ├─ Collect conversation data:                                │
│    │  ├─ All messages (user + assistant)                        │
│    │  ├─ All tool calls and results                             │
│    │  ├─ Metadata (timestamps, tokens)                          │
│    │  ├─ Model configuration used                               │
│    │  └─ Current turn number                                    │
│    │                                                             │
│    └─ Create snapshot structure:                                │
│       {                                                          │
│         "session_id": "abc123...",                               │
│         "created_at": "2025-01-15T10:30:00Z",                    │
│         "last_modified": "2025-01-15T11:45:00Z",                 │
│         "model": "gpt-5-codex",                                  │
│         "approval_policy": "on-request",                         │
│         "sandbox_mode": "workspace-write",                       │
│         "working_directory": "/path/to/project",                 │
│         "turns": [...],                                          │
│         "total_tokens": 15000,                                   │
│         "total_cost": 0.15                                       │
│       }                                                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. SERIALIZE TO JSON                                            │
│    ├─ Convert conversation to JSON                              │
│    ├─ Handle special types (dates, buffers)                     │
│    ├─ Compress large content (optional)                         │
│    └─ Pretty-print for readability                              │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. DETERMINE SAVE LOCATION                                      │
│    │                                                             │
│    ├─ Session directory:                                        │
│    │  ~/.codex/sessions/{session_id}/                           │
│    │                                                             │
│    ├─ Files to save:                                            │
│    │  ├─ session.json       # Metadata                          │
│    │  ├─ conversation.json  # Full conversation                 │
│    │  ├─ state.json        # Cached state                       │
│    │  └─ logs/             # Event logs                         │
│    │     └─ events.log                                          │
│    │                                                             │
│    └─ Ensure directory exists (create if not)                   │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. ATOMIC WRITE                                                 │
│    │                                                             │
│    ├─ Write to temporary file first:                            │
│    │  conversation.json.tmp                                     │
│    │                                                             │
│    ├─ Verify write succeeded                                    │
│    │  ├─ Check file size > 0                                    │
│    │  └─ Validate JSON parseable                                │
│    │                                                             │
│    ├─ Atomic rename:                                            │
│    │  conversation.json.tmp → conversation.json                 │
│    │  (OS guarantees atomicity)                                 │
│    │                                                             │
│    └─ Backup old version (optional):                            │
│       conversation.json.bak                                     │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 5. UPDATE SESSION INDEX                                         │
│    │                                                             │
│    ├─ Add to recent sessions list:                              │
│    │  ~/.codex/sessions/recent.json                             │
│    │  [                                                          │
│    │    {                                                        │
│    │      "id": "abc123...",                                     │
│    │      "created_at": "2025-01-15T10:30:00Z",                  │
│    │      "last_modified": "2025-01-15T11:45:00Z",               │
│    │      "preview": "First 100 chars of last message...",       │
│    │      "turns": 5,                                            │
│    │      "model": "gpt-5-codex"                                 │
│    │    }                                                        │
│    │  ]                                                          │
│    │                                                             │
│    └─ Keep last 100 sessions (configurable)                     │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 6. SAVE COMPLETE                                                │
│    └─ Ready for next turn or resume                             │
└─────────────────────────────────────────────────────────────────┘


RESTORATION FLOW:
═════════════════

┌─────────────────────────────────────────────────────────────────┐
│              USER RESUMES SESSION                                │
│         codex resume abc123 "continue"                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. LOCATE SESSION FILES                                         │
│    ├─ Find session directory                                    │
│    ├─ Check files exist                                         │
│    ├─ Validate not corrupted                                    │
│    └─ Check version compatibility                               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. LOAD SESSION METADATA                                        │
│    ├─ Read session.json                                         │
│    ├─ Check model still available                               │
│    ├─ Verify working directory exists                           │
│    └─ Load configuration used                                   │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. DESERIALIZE CONVERSATION                                     │
│    ├─ Read conversation.json                                    │
│    ├─ Parse JSON                                                │
│    ├─ Reconstruct message objects                               │
│    ├─ Rebuild conversation state                                │
│    └─ Validate data integrity                                   │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. RESTORE CONTEXT                                              │
│    ├─ Rebuild conversation history                              │
│    ├─ Restore tool registry state                               │
│    ├─ Re-establish MCP connections (if used)                    │
│    ├─ Verify cached context still valid                         │
│    └─ Load any cached computed state                            │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 5. SESSION RESUMED                                              │
│    ├─ Show session info to user                                 │
│    ├─ Display last few messages                                 │
│    ├─ Send new prompt to continue                               │
│    └─ Resume conversation loop                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Sandbox Enforcement Flow

### Platform-Specific Execution Isolation

```
┌─────────────────────────────────────────────────────────────────┐
│           AGENT WANTS TO RUN COMMAND                             │
│              command: "npm install"                              │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. CHECK SANDBOX MODE                                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┼─────────┬──────────────┐
        │         │         │              │
┌───────▼──┐ ┌────▼────┐ ┌─▼─────────┐ ┌──▼─────────────┐
│read-only │ │workspace│ │danger-full │ │Platform detect │
│          │ │-write   │ │-access     │ │                │
└───────┬──┘ └────┬────┘ └─┬─────────┘ └──┬─────────────┘
        │         │         │              │
        │         │         │      ┌───────┴────────┐
        │         │         │      │                │
        │         │         │  ┌───▼────┐      ┌────▼────┐
        │         │         │  │ macOS  │      │ Linux   │
        │         │         │  │        │      │         │
        │         │         │  │Seatbelt│      │Landlock │
        │         │         │  └───┬────┘      └────┬────┘
        │         │         │      │                │
        │         │         └──────┼────────────────┘
        │         │                │ (no sandbox)
        │         │                │
        └─────────┴────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. GENERATE SANDBOX PROFILE (macOS Seatbelt)                    │
│                                                                  │
│ let profile = format!(r#"                                        │
│   (version 1)                                                   │
│   (deny default)                                                │
│                                                                  │
│   ; Allow reading most files                                    │
│   (allow file-read*)                                            │
│                                                                  │
│   ; Restrict writing based on mode                              │
│   (allow file-write*                                            │
│       (subpath "{}")         ; workspace                        │
│       (subpath "{}")         ; temp dir                         │
│       (subpath "{}"))        ; ~/.codex                         │
│                                                                  │
│   ; Network policy                                              │
│   (deny network*)                                               │
│   (allow network-outbound                                       │
│       (remote tcp "api.openai.com:443"))                        │
│                                                                  │
│   ; Process operations                                          │
│   (allow process-exec                                           │
│       (subpath "/bin")                                          │
│       (subpath "/usr/bin"))                                     │
│                                                                  │
│   ; IPC restrictions                                            │
│   (deny ipc*)                                                   │
│ "#, workspace, tmpdir, codex_home);                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. WRAP COMMAND WITH SANDBOX (macOS)                            │
│                                                                  │
│ Original:  npm install                                          │
│                                                                  │
│ Wrapped:   /usr/bin/sandbox-exec \\                             │
│              -p "(version 1)(deny default)..." \\                │
│              npm install                                        │
│                                                                  │
│ Or with profile file:                                           │
│            /usr/bin/sandbox-exec \\                             │
│              -f /tmp/codex-sandbox-{pid}.sbpl \\                 │
│              npm install                                        │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. CONFIGURE LANDLOCK (Linux)                                   │
│                                                                  │
│ use landlock::{                                                 │
│     Access, AccessFs, Ruleset, RulesetAttr, ABI,                │
│ };                                                              │
│                                                                  │
│ let abi = ABI::V1;                                              │
│ let mut ruleset = Ruleset::new()                                │
│     .handle_access(AccessFs::from_all(abi))?                    │
│     .create()?;                                                 │
│                                                                  │
│ // Add allowed paths                                            │
│ ruleset = ruleset                                               │
│     .add_rule(                                                  │
│         AccessFs::from_read(abi)                                │
│             .union(AccessFs::from_write(abi)),                  │
│         workspace_path,                                         │
│     )?                                                          │
│     .add_rule(                                                  │
│         AccessFs::from_read(abi),                               │
│         "/usr",                                                 │
│     )?;                                                         │
│                                                                  │
│ // Restrict this process                                        │
│ ruleset.restrict_self()?;                                       │
│                                                                  │
│ // Now execute command (already sandboxed)                      │
│ exec_command("npm", &["install"])?;                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 5. SPAWN PROCESS                                                │
│    ├─ Create process with sandbox                               │
│    ├─ Set up stdio pipes                                        │
│    ├─ Set resource limits                                       │
│    │  ├─ CPU limit (if configured)                              │
│    │  ├─ Memory limit (if configured)                           │
│    │  └─ Timeout (default: 2 minutes)                           │
│    └─ Start process                                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 6. MONITOR EXECUTION                                            │
│    │                                                             │
│    ├─ Stream stdout/stderr                                      │
│    ├─ Check timeout every 100ms                                 │
│    ├─ Monitor for sandbox violations                            │
│    └─ Wait for completion                                       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐   ┌──────▼───────────────┐
│ Success        │   │ Sandbox Violation    │
│                │   │                      │
│ Exit code 0    │   │ Example errors:      │
│ Output OK      │   │ • File write denied  │
└───────┬────────┘   │ • Network blocked    │
        │            │ • Path forbidden     │
        │            └──────┬───────────────┘
        │                   │
        │                   │
┌───────▼───────────────────▼─────────────────────────────────────┐
│ 7. RETURN RESULT                                                │
│    ├─ Sandbox violations → error message                        │
│    ├─ Normal exit → output                                      │
│    └─ Send to agent                                             │
└─────────────────────────────────────────────────────────────────┘


SANDBOX VIOLATION HANDLING:
═══════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│ VIOLATION DETECTED                                              │
│                                                                  │
│ Example: "Operation not permitted: /etc/passwd"                 │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ Check Approval Policy                                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐   ┌──────▼──────────┐
│ on-failure     │   │ other modes     │
│                │   │                 │
│ Offer retry    │   │ Return error    │
│ without sandbox│   │ to agent        │
└───────┬────────┘   └─────────────────┘
        │
        │ User approves
        │
┌───────▼──────────────────────────────────────────────────────┐
│ Retry without sandbox (danger-full-access)                    │
└───────────────────────────────────────────────────────────────┘
```

---

## Event Processing Flow (Exec Mode)

### From Core Events → Output

```
┌─────────────────────────────────────────────────────────────────┐
│              CODEX EXEC STARTS                                   │
│    codex exec --json "analyze codebase"                         │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 1. DETERMINE OUTPUT MODE                                        │
│    ├─ Check --json flag                                         │
│    └─ Create appropriate processor                              │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼───────┐    ┌──────▼──────────────┐
│ Human Output  │    │ JSON Lines Output   │
│ Processor     │    │ Processor           │
└───────┬───────┘    └──────┬──────────────┘
        │                   │
        └─────────┬─────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 2. START SESSION                                                │
│    ├─ Send initial prompt to core                               │
│    ├─ Receive event stream                                      │
│    └─ Begin processing                                          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 3. EVENT STREAM LOOP                                            │
│    │                                                             │
│    └─ For each event:                                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  │
       EVENT: thread.started
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  (silent, save ID internally)                            │
│ JSON:   {"type":"thread.started","thread_id":"abc123"}          │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: turn.started
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  (silent)                                                │
│ JSON:   {"type":"turn.started"}                                 │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: item.started (reasoning)
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  → stderr: "Analyzing repository structure..."           │
│ JSON:   {"type":"item.started","item":{"id":"i1",               │
│            "type":"reasoning","text":"Analyzing..."}}            │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: item.completed (reasoning)
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  (silent, reasoning complete)                            │
│ JSON:   {"type":"item.completed","item":{"id":"i1",             │
│            "type":"reasoning","text":"Complete analysis"}}       │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: item.started (command_execution)
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  → stderr: "Running: ls -la"                             │
│ JSON:   {"type":"item.started","item":{"id":"i2",               │
│            "type":"command_execution","command":"ls -la",        │
│            "status":"in_progress"}}                              │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: item.updated (command_execution, streaming output)
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  → stderr: (partial output as it streams)                │
│ JSON:   {"type":"item.updated","item":{"id":"i2",               │
│            "aggregated_output":"total 24\ndrwxr..."}}            │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: item.completed (command_execution)
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  → stderr: "Command completed: exit code 0"              │
│ JSON:   {"type":"item.completed","item":{"id":"i2",             │
│            "exit_code":0,"status":"completed",                   │
│            "aggregated_output":"..."}}                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: item.completed (agent_message)
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  (save for final output)                                │
│         final_message = "Found 15 files in repository"          │
│ JSON:   {"type":"item.completed","item":{"id":"i3",             │
│            "type":"agent_message",                               │
│            "text":"Found 15 files in repository"}}               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
       EVENT: turn.completed
┌─────────────────▼───────────────────────────────────────────────┐
│ Human:  (silent)                                                │
│ JSON:   {"type":"turn.completed",                               │
│            "usage":{"input_tokens":1000,"output_tokens":50}}     │
└─────────────────┬───────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────────┐
│ 4. STREAM COMPLETE                                              │
│    └─ Finalize output                                           │
└─────────────────┬───────────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼───────┐    ┌──────▼──────────────┐
│ Human Mode    │    │ JSON Mode           │
│               │    │                     │
│ → stdout:     │    │ (already complete)  │
│ "Found 15     │    │ All events written  │
│  files in     │    │ to stdout           │
│  repository"  │    │                     │
│               │    │                     │
│ (clean final  │    │                     │
│  message only)│    │                     │
└───────────────┘    └─────────────────────┘
```

---

## Related Documentation

- **Official**: All official docs in `/context/codex/docs/`
- **Custom**:
  - [02-architecture.md](./02-architecture.md) - System architecture
  - [03-prompt-processing.md](./03-prompt-processing.md) - Prompt details
  - [06-tool-system.md](./06-tool-system.md) - Tool system architecture
  - [22-exec-mode-internals.md](./22-exec-mode-internals.md) - Exec mode implementation

---

**Last Updated**: October 25, 2025
**Purpose**: Visual reference for understanding Codex internal flows
**Note**: Diagrams are simplified for clarity. See source code for complete implementation.
