# Comparison: Documentation vs NPX Installation

## Overview

This document compares the tool information from:
1. **`CLAUDE_CODE_INTERNAL_TOOLS_DEEP_DIVE.md`** - Documentation derived from system prompts and tool definitions
2. **NPX Installation** - Actual tool schemas and descriptions from the installed Claude Code CLI

---

## Key Findings

### ✅ Perfect Match Items

The following information **perfectly matches** between the documentation and the NPX installation:

1. **Tool Input Schemas** - All TypeScript interface definitions match exactly
2. **Tool Names** - All tool names are consistent
3. **Required/Optional Parameters** - Parameter requirements match
4. **Tool Descriptions** - Core functionality descriptions are identical
5. **Usage Instructions** - Detailed usage notes align perfectly

### 📝 Source of Truth

**NPX Installation Location:**
- Tool schemas: `/sdk-tools.d.ts` (auto-generated from JSON Schema)
- Tool descriptions: `/cli.js` (bundled, ~9.7MB)
- SDK definitions: `/sdk.d.ts`

**Documentation in `CLAUDE_CODE_INTERNAL_TOOLS_DEEP_DIVE.md`:**
- Extracted from system prompts visible to Claude during execution
- Represents the actual tool definitions Claude sees

### 🔍 Detailed Comparison by Tool

---

## 1. Read Tool

### Schema Match: ✅ PERFECT

**Documentation:**
```typescript
interface ReadTool {
  file_path: string;
  offset?: number;
  limit?: number;
}
```

**NPX (sdk-tools.d.ts):**
```typescript
export interface FileReadInput {
  file_path: string;
  offset?: number;
  limit?: number;
}
```

### Description Match: ✅ PERFECT

Both sources contain identical text:
```
Reads a file from the local filesystem. You can access any file directly by using this tool.
Assume this tool is able to read all files on the machine. If the User provides a path to a file assume that path is valid. It is okay to read a file that does not exist; an error will be returned.
```

### Constants Match: 

| Constant | Documentation | NPX cli.js | Match |
|----------|--------------|------------|-------|
| Default lines | 2000 | `BG1=2000` | ✅ |
| Char truncation | 2000 | `eE9=2000` | ✅ |
| PDF max size | 32MB | `UOA=33554432` | ✅ |

---

## 2. Write Tool

### Schema Match: ✅ PERFECT

**Documentation:**
```typescript
interface WriteTool {
  file_path: string;
  content: string;
}
```

**NPX:**
```typescript
export interface FileWriteInput {
  file_path: string;
  content: string;
}
```

### Description Match: ✅ PERFECT

Identical safety mechanisms documented:
- Must read file first (system enforced)
- Overwrites existing files completely
- Never create docs unless requested
- No emojis unless requested

---

## 3. Edit Tool

### Schema Match: ✅ PERFECT

**Documentation:**
```typescript
interface EditTool {
  file_path: string;
  old_string: string;
  new_string: string;
  replace_all?: boolean;
}
```

**NPX:**
```typescript
export interface FileEditInput {
  file_path: string;
  old_string: string;
  new_string: string;
  replace_all?: boolean;
}
```

### Description Match: ✅ PERFECT

From NPX cli.js:
```
Performs exact string replacements in files.

Usage:
- You must use your Read tool at least once in the conversation before editing
- Ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix
- The line number prefix format is: spaces + line number + tab
- The edit will FAIL if `old_string` is not unique in the file
- Use `replace_all` for replacing and renaming strings across the file
```

This matches the documentation exactly.

---

## 4. Glob Tool

### Schema Match: ✅ PERFECT

**Documentation:**
```typescript
interface GlobTool {
  pattern: string;
  path?: string;
}
```

**NPX:**
```typescript
export interface GlobInput {
  pattern: string;
  path?: string;
}
```

### Description Match: ✅ PERFECT

From NPX cli.js:
```
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
```

Matches documentation completely.

---

## 5. Grep Tool

### Schema Match: ✅ PERFECT

All parameters match including:
- `pattern`, `path`, `glob`, `output_mode`
- Context flags: `-A`, `-B`, `-C`, `-n`, `-i`
- `type`, `head_limit`, `multiline`

### Description Match: ✅ PERFECT

From NPX cli.js:
```
A powerful search tool built on ripgrep

Usage:
- ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command
- Supports full regex syntax (e.g., "log.*Error", "function\\s+\\w+")
- Output modes: "content" shows matching lines, "files_with_matches" shows only file paths (default), "count" shows match counts
- Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping
- Multiline matching: By default patterns match within single lines only
```

**Default output_mode:** ✅ Confirmed as `"files_with_matches"` in both sources

---

## 6. Bash Tool

### Schema Match: ✅ PERFECT

```typescript
export interface BashInput {
  command: string;
  timeout?: number;
  description?: string;
  run_in_background?: boolean;
}
```

### Operational Limits Match: 

| Limit | Documentation | NPX Implementation |
|-------|--------------|-------------------|
| Default timeout | 120000ms (2 min) | ✅ Confirmed |
| Max timeout | 600000ms (10 min) | ✅ Confirmed |
| Output truncation | 30000 chars | ✅ Confirmed |

### Description Match: ✅ PERFECT

From NPX cli.js:
```
Executes a given bash command in a persistent shell session with optional timeout, ensuring proper handling and security measures.

IMPORTANT: This tool is for terminal operations like git, npm, docker, etc. DO NOT use it for file operations (reading, writing, editing, searching, finding files) - use the specialized tools for this instead.
```

---

## 7. Task Tool (Agent)

### Schema Match: ✅ PERFECT

```typescript
export interface AgentInput {
  description: string;
  prompt: string;
  subagent_type: string;
}
```

### Agent Types Match: 

From NPX cli.js, confirmed agent types:
1. **general-purpose** - Tools: `*` (all)
2. **Explore** - Tools: `Glob, Grep, Read, Bash`
3. **statusline-setup** - Tools: `Read, Edit`
4. **output-style-setup** - Tools: `Read, Write, Edit, Glob, Grep`

### Thoroughness Levels (Explore): ✅ CONFIRMED

- "quick" - basic searches
- "medium" - moderate exploration
- "very thorough" - comprehensive analysis

### Agent Colors: ✅ DISCOVERED IN NPX

Found in cli.js bundle:
```javascript
const n61 = ["red","blue","green","yellow","purple","orange","pink","cyan"];
```

These are used for subagent identification with suffix `_FOR_SUBAGENTS_ONLY`.

---

## 8. TodoWrite Tool

### Schema Match: ✅ PERFECT

```typescript
export interface TodoWriteInput {
  todos: {
    content: string;
    status: "pending" | "in_progress" | "completed";
    activeForm: string;
  }[];
}
```

### Description Match: ✅ PERFECT

From NPX cli.js:
```
Use this tool to create and manage a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.

When to Use This Tool:
1. Complex multi-step tasks - When a task requires 3 or more distinct steps
2. Non-trivial and complex tasks
3. User explicitly requests todo list
4. User provides multiple tasks
5. After receiving new instructions
6. When you start working on a task - Mark it as in_progress BEFORE beginning work
7. After completing a task

Task Management:
- Update task status in real-time as you work
- Mark tasks complete IMMEDIATELY after finishing (don't batch completions)
- Exactly ONE task must be in_progress at any time (not less, not more)
```

---

## 9. AskUserQuestion Tool

### Schema Match: ✅ PERFECT

From NPX sdk-tools.d.ts, the schema includes:
- `questions` array (1-4 items)
- Each question has: `question`, `header`, `options`, `multiSelect`
- Options have: `label`, `description`
- `answers` object for collected responses

### Description Match: ✅ PERFECT

From NPX cli.js:
```
Use this tool when you need to ask the user questions during execution. This allows you to:
1. Gather user preferences or requirements
2. Clarify ambiguous instructions
3. Get decisions on implementation choices as you work
4. Offer choices to the user about what direction to take

Usage notes:
- Users will always be able to select "Other" to provide custom text input
- Use multiSelect: true to allow multiple answers to be selected for a question
```

---

## 10. ExitPlanMode Tool

### Schema Match: ✅ PERFECT

```typescript
export interface ExitPlanModeInput {
  plan: string;
}
```

### Usage Guidance Match: 

From NPX cli.js:
```
Use this tool when you are in plan mode and have finished presenting your plan and are ready to code

Handling Ambiguity in Plans:
1. Use the AskUserQuestion tool to clarify with the user
2. Ask about specific implementation choices
3. Clarify any assumptions that could affect the implementation
4. Only proceed with ExitPlanMode after resolving ambiguities
```

---

## 11. WebFetch Tool

### Schema Match: ✅ PERFECT

```typescript
export interface WebFetchInput {
  url: string;
  prompt: string;
}
```

### Description Match: ✅ PERFECT

From NPX cli.js:
```
Fetches content from a specified URL and processes it using an AI model
- Takes a URL and a prompt as input
- Fetches the URL content, converts HTML to markdown
- Processes the content with the prompt using a small, fast model
- Includes a self-cleaning 15-minute cache for faster responses
- HTTP URLs will be automatically upgraded to HTTPS
- When a URL redirects to a different host, the tool will inform you and provide the redirect URL

IMPORTANT: If an MCP-provided web fetch tool is available, prefer using that tool instead
```

---

## 12. WebSearch Tool

### Schema Match: ✅ PERFECT

```typescript
export interface WebSearchInput {
  query: string;
  allowed_domains?: string[];
  blocked_domains?: string[];
}
```

### Constraints Match: 

- Minimum query length: 2 characters (confirmed in JSON schema)
- Only available in US (documented)
- Must account for current date in queries (documented)

---

## 13. NotebookEdit Tool

### Schema Match: ✅ PERFECT

```typescript
export interface NotebookEditInput {
  notebook_path: string;
  cell_id?: string;
  new_source: string;
  cell_type?: "code" | "markdown";
  edit_mode?: "replace" | "insert" | "delete";
}
```

### Key Details: 

- Cell indexing: 0-indexed (confirmed)
- Edit modes: replace (default), insert, delete
- Absolute paths required

---

## 14. Background Process Tools

### BashOutput

```typescript
export interface BashOutputInput {
  bash_id: string;
  filter?: string;
}
```

**Behavior:** Returns ONLY new output since last check (verified in both sources)

### KillShell

```typescript
export interface KillShellInput {
  shell_id: string;
}
```

Both schemas match perfectly.

---

## 15. MCP Tools

### ListMcpResources

```typescript
export interface ListMcpResourcesInput {
  server?: string;
}
```

### ReadMcpResource

```typescript
export interface ReadMcpResourceInput {
  server: string;
  uri: string;
}
```

### IDE Tools (from MCP)

**getDiagnostics:**
```typescript
interface getDiagnosticsInput {
  uri?: string;
}
```

**executeCode:**
```typescript
interface executeCodeInput {
  code: string;
}
```

All match the documentation.

---

## SDK Information

### Deprecation Notice: ✅ CONFIRMED

From NPX sdk.d.ts:
```typescript
/**
 * @deprecated The Claude Code SDK is now the Claude Agent SDK!
 * Please install and use @anthropic-ai/claude-agent-sdk instead.
 * This SDK entrypoint will be going away on October 21.
 * See https://docs.claude.com/en/docs/claude-code/sdk/migration-guide
 */
export declare function query(...): Query;
```

### Hook Events: 

```typescript
export const HOOK_EVENTS = [
  "PreToolUse",
  "PostToolUse",
  "Notification",
  "UserPromptSubmit",
  "SessionStart",
  "SessionEnd",
  "Stop",
  "SubagentStop",
  "PreCompact"
] as const;
```

Matches documentation exactly.

### Permission Modes: 

```typescript
export type PermissionMode =
  | 'default'
  | 'acceptEdits'
  | 'bypassPermissions'
  | 'plan';
```

All four modes confirmed.

### MCP Server Types: 

- `stdio` - Command-line servers
- `sse` - Server-Sent Events
- `http` - HTTP servers
- `sdk` - In-process SDK servers

---

## Bundle Implementation Details

### Variable Name Obfuscation

The bundled cli.js uses minified variable names:

```javascript
x8 = "Read"
CJ = "Write"
R3 = "Edit"
wD = "Glob"
_F = "Grep"
W3 = "Task"
_4 = "Bash"
```

These are internal and don't affect external API.

### Template String Variables

Tool descriptions use template variables:

```javascript
BG1 = 2000  // Read line limit
eE9 = 2000  // Character truncation
UOA = 33554432  // PDF max size (32MB)
```

These are interpolated into the tool description strings at runtime.

---