# OpenCode - Flow Diagrams

> **Visual reference for understanding OpenCode's execution flows**

---

## Overview

This document provides ASCII flow diagrams for key OpenCode operations, helping visualize how different components interact.

---

## Session Creation Flow

```
User Command
    │
    ▼
┌──────────────────┐
│ CLI Entry Point  │
│ (index.ts)       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Parse Arguments  │
│ (yargs)          │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Load Config      │
│ Global + Project │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Create Session   │
│ Session.create() │
└────────┬─────────┘
         │
         ├─ Generate ID (ulid)
         ├─ Set provider/model
         ├─ Create directory
         ├─ Initialize state
         │
         ▼
┌──────────────────┐
│ Return Session   │
│ { id, provider } │
└──────────────────┘
```

---

## Prompt Processing Flow

```
User Input
    │
    ▼
┌─────────────────────┐
│ SessionPrompt       │
│ .prompt()           │
└──────────┬──────────┘
           │
           ▼
     ┌─────────────┐
     │ Check Lock  │ ◀─── Busy? Queue and wait
     └─────┬───────┘
           │ Free
           ▼
┌─────────────────────┐
│ Acquire Lock        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Build Context       │
├─────────────────────┤
│ - System prompts    │
│ - Environment info  │
│ - AGENTS.md files   │
│ - Message history   │
│ - Tool descriptions │
│ - LSP data          │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Check Token Limit   │
└──────────┬──────────┘
           │
           ├─ Overflow? ──▶ Compact Session
           │
           ▼
┌─────────────────────┐
│ Stream AI Request   │
│ (Vercel AI SDK)     │
└──────────┬──────────┘
           │
           ▼
     ┌──────────────────────┐
     │ Process Stream       │
     ├──────────────────────┤
     │ ├─ text-delta        │
     │ │   └─ Update part   │
     │ ├─ tool-call         │
     │ │   └─ Execute tool  │
     │ ├─ tool-result       │
     │ │   └─ Continue AI   │
     │ └─ finish            │
     │     └─ Calculate cost│
     └──────────┬───────────┘
                │
                ▼
     ┌──────────────────────┐
     │ Release Lock         │
     └──────────┬───────────┘
                │
                ▼
     ┌──────────────────────┐
     │ Process Queue        │
     │ (if pending)         │
     └──────────────────────┘
```

---

## Tool Execution Flow

```
AI Tool Request
    │
    ▼
┌──────────────────┐
│ Parse Tool Call  │
│ - name           │
│ - arguments      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Find Tool in     │
│ Registry         │
└────────┬─────────┘
         │ Found
         ▼
┌──────────────────┐
│ Validate Args    │
│ (Zod schema)     │
└────────┬─────────┘
         │ Valid
         ▼
┌──────────────────┐
│ Check Permission │
└────────┬─────────┘
         │
         ├─"allow"─▶ Continue
         ├─"deny"──▶ Return error
         └─"ask"───┐
                   ▼
         ┌──────────────────┐
         │ Show Approval UI │
         └────────┬─────────┘
                  │
                  ├─Approved──▶ Continue
                  └─Denied────▶ Return error
                  │
                  ▼
         ┌──────────────────┐
         │ Create Tool Part │
         │ (status: active) │
         └────────┬─────────┘
                  │
                  ▼
         ┌──────────────────┐
         │ Execute Tool     │
         │ with timeout     │
         └────────┬─────────┘
                  │
                  ├─Success─┐
                  │         ▼
                  │    ┌──────────────┐
                  │    │ Update part  │
                  │    │ (completed)  │
                  │    └──────┬───────┘
                  │           │
                  │           ▼
                  │    Return output to AI
                  │
                  └─Error───┐
                            ▼
                       ┌──────────────┐
                       │ Update part  │
                       │ (error)      │
                       └──────┬───────┘
                              │
                              ▼
                       Return error to AI
```

---

## ACP Protocol Flow

```
IDE (Zed/VS Code)
    │
    ▼
┌────────────────────┐
│ Initialize         │
│ JSON-RPC Request   │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ opencode acp       │
│ (stdio mode)       │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Return Capabilities│
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Create Session     │
│ Request            │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Session.create()   │
│ Map ACP ⟷ OpenCode│
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Return Session ID  │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Send Message       │
│ Request            │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ SessionPrompt      │
│ .prompt()          │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Stream Response    │
│ Buffer chunks      │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│ Return Complete    │
│ Response to IDE    │
└────────────────────┘
```

---

## Session Compaction Flow

```
Token Overflow Detected
    │
    ▼
┌──────────────────────┐
│ SessionCompaction    │
│ .run()               │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Get All Messages     │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Strategy 1:          │
│ Prune Old Tool       │
│ Outputs              │
└────────┬─────────────┘
         │
         ├─ Keep user messages
         ├─ Keep recent tool calls
         ├─ Remove old outputs
         │
         ▼
┌──────────────────────┐
│ Still Overflow?      │
└────────┬─────────────┘
         │ Yes
         ▼
┌──────────────────────┐
│ Strategy 2:          │
│ Summarize Old        │
│ Conversation         │
└────────┬─────────────┘
         │
         ├─ Group old messages
         ├─ Send to AI for summary
         ├─ Replace with summary
         │
         ▼
┌──────────────────────┐
│ Mark Messages as     │
│ Compacted            │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Verify Token Count   │
│ Under Limit          │
└──────────────────────┘
```

---

## LSP Integration Flow

```
File Edited
    │
    ▼
┌──────────────────────┐
│ LSP.touchFile()      │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Detect Language      │
│ from Extension       │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ LSP Server Running?  │
└────────┬─────────────┘
         │ No
         ▼
┌──────────────────────┐
│ Start LSP Server     │
│ for Language         │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Initialize Server    │
│ with Workspace       │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Open Document        │
│ in LSP               │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Request Diagnostics  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Cache Results        │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Return to Tool       │
│ (lsp-diagnostics)    │
└──────────────────────┘
```

---

## MCP Server Integration Flow

```
OpenCode Startup
    │
    ▼
┌──────────────────────┐
│ Load Config          │
│ (.opencode/config)   │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Find MCP Servers     │
│ in config.mcp        │
└────────┬─────────────┘
         │
         ▼
     ┌─────────────────┐
     │ For each server │
     └────────┬────────┘
              │
              ▼
     ┌──────────────────────┐
     │ Spawn Server Process │
     │ (command + args)     │
     └────────┬─────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ Connect via          │
     │ stdio/IPC            │
     └────────┬─────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ Request Tools List   │
     └────────┬─────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ Register Tools in    │
     │ Tool Registry        │
     └────────┬─────────────┘
              │
              ▼
     ┌──────────────────────┐
     │ Tools Available      │
     │ to AI Agent          │
     └──────────────────────┘

When AI Calls MCP Tool:
    │
    ▼
┌──────────────────────┐
│ Route to MCP Server  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Send JSON-RPC        │
│ Request              │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Receive Response     │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Return to AI         │
└──────────────────────┘
```

---

## Configuration Loading Flow

```
OpenCode Start
    │
    ▼
┌──────────────────────┐
│ Load Global Config   │
│ ~/.opencode/config   │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Find Project Configs │
│ (walk up directory)  │
└────────┬─────────────┘
         │
         ▼
     ┌───────────────────┐
     │ For each found:   │
     │ .opencode/config  │
     └────────┬──────────┘
              │
              ▼
     ┌──────────────────────┐
     │ Deep Merge           │
     │ (project overrides)  │
     └────────┬─────────────┘
              │
              ▼
┌──────────────────────┐
│ Check CLI Flags      │
│ (highest priority)   │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Check ENV Vars       │
│ (OPENCODE_*)         │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Final Merged Config  │
└──────────────────────┘

Config Priority (high to low):
1. CLI flags
2. Environment variables
3. Project config (closest)
4. Project config (ancestors)
5. Global config
```

---

## File Tool Flow (Read Example)

```
AI: "Read auth.ts"
    │
    ▼
┌──────────────────────┐
│ Parse Tool Call      │
│ tool: read           │
│ args: {              │
│   filePath: auth.ts  │
│ }                    │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Resolve Path         │
│ (relative to cwd)    │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Validate in          │
│ Workspace            │
└────────┬─────────────┘
         │ Valid
         ▼
┌──────────────────────┐
│ Check File Exists    │
└────────┬─────────────┘
         │ Exists
         ▼
┌──────────────────────┐
│ Detect if Binary     │
└────────┬─────────────┘
         │ Text File
         ▼
┌──────────────────────┐
│ Read Content         │
│ (Bun.file().text())  │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Split into Lines     │
│ Apply offset/limit   │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Format with Line     │
│ Numbers              │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Touch in LSP         │
│ (track access)       │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│ Return to AI         │
│ <file>               │
│ 00001| content       │
│ 00002| ...           │
│ </file>              │
└──────────────────────┘
```

---

## Summary

These flow diagrams illustrate:
- **Session lifecycle** - Creation to completion
- **Prompt processing** - Context assembly and streaming
- **Tool execution** - Permission to result
- **Integrations** - ACP, LSP, MCP protocols
- **Configuration** - Multi-level merging
- **File operations** - Safe file access

Use these diagrams to understand how components interact and data flows through OpenCode.

---

**Tip**: When debugging, trace your issue through the relevant flow diagram to identify which component might be responsible.

