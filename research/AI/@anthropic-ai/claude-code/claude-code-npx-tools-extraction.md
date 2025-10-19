# Claude Code NPX Installation - Tools Deep Dive

## Installation Structure

```
//
├── cli.js (9.7MB) - Main bundled executable
├── sdk.mjs (533KB) - SDK module
├── sdk.d.ts - SDK TypeScript definitions
├── sdk-tools.d.ts - Tool input schema TypeScript definitions
├── package.json
├── README.md
├── LICENSE.md
├── yoga.wasm - Layout engine
├── node_modules/
└── vendor/
```

---

## Tool Schema Definitions (from sdk-tools.d.ts)

The tool input schemas are generated from JSON Schema and exposed as TypeScript types:

### Complete Type Union

```typescript
export type ToolInputSchemas =
  | AgentInput
  | BashInput
  | BashOutputInput
  | ExitPlanModeInput
  | FileEditInput
  | FileReadInput
  | FileWriteInput
  | GlobInput
  | GrepInput
  | KillShellInput
  | ListMcpResourcesInput
  | McpInput
  | NotebookEditInput
  | ReadMcpResourceInput
  | TodoWriteInput
  | WebFetchInput
  | WebSearchInput;
```

---

## Tool Descriptions (from cli.js bundle)

### 1. Read Tool (`FileReadInput`)

**Variable Name in Bundle:** `x8 = "Read"`

**Description String Variable:** `EOA`

**Full Description:**
```
Reads a file from the local filesystem. You can access any file directly by using this tool.
Assume this tool is able to read all files on the machine. If the User provides a path to a file assume that path is valid. It is okay to read a file that does not exist; an error will be returned.

Usage:
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to 2000 lines (BG1) starting from the beginning of the file
- You can optionally specify a line offset and limit (especially handy for long files), but it's recommended to read the whole file by not providing these parameters
- Any lines longer than 2000 characters (eE9) will be truncated
- Results are returned using cat -n format, with line numbers starting at 1
- This tool allows Claude Code to read images (eg PNG, JPG, etc). When reading an image file the contents are presented visually as Claude Code is a multimodal LLM.
- This tool can read PDF files (.pdf). PDFs are processed page by page, extracting both text and visual content for analysis.
- This tool can read Jupyter notebooks (.ipynb files) and returns all cells with their outputs, combining code, text, and visualizations.
- This tool can only read files, not directories. To read a directory, use an ls command via the Bash tool.
- You can call multiple tools in a single response. It is always better to speculatively read multiple potentially useful files in parallel.
- You will regularly be asked to read screenshots. If the user provides a path to a screenshot, ALWAYS use this tool to view the file at the path. This tool will work with all temporary file paths.
- If you read a file that exists but has empty contents you will receive a system reminder warning in place of file contents.
```

**Input Schema:**
```typescript
export interface FileReadInput {
  file_path: string;        // The absolute path to the file to read
  offset?: number;          // The line number to start reading from
  limit?: number;           // The number of lines to read
}
```

**Constants:**
- `BG1 = 2000` - Default line read limit
- `eE9 = 2000` - Character truncation per line
- `qOA = "Read a file from the local filesystem."` - Short description

**PDF Support:**
- Maximum PDF size: `UOA = 33554432` (32MB)
- Returns base64 encoded PDF for processing

---

### 2. Write Tool (`FileWriteInput`)

**Variable Name in Bundle:** `CJ = "Write"`

**Description String Variable:** `wOA`

**Full Description:**
```
Writes a file to the local filesystem.

Usage:
- This tool will overwrite the existing file if there is one at the provided path.
- If this is an existing file, you MUST use the Read tool first to read the file's contents. This tool will fail if you did not read the file first.
- ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.
- NEVER proactively create documentation files (*.md) or README files. Only create documentation files if explicitly requested by the User.
- Only use emojis if the user explicitly requests it. Avoid writing emojis to files unless asked.
```

**Input Schema:**
```typescript
export interface FileWriteInput {
  file_path: string;        // The absolute path to the file to write (must be absolute, not relative)
  content: string;          // The content to write to the file
}
```

---

### 3. Edit Tool (`FileEditInput`)

**Variable Name in Bundle:** `R3 = "Edit"`

**Input Schema:**
```typescript
export interface FileEditInput {
  file_path: string;        // The absolute path to the file to modify
  old_string: string;       // The text to replace
  new_string: string;       // The text to replace it with (must be different from old_string)
  replace_all?: boolean;    // Replace all occurences of old_string (default false)
}
```

---

### 4. Glob Tool (`GlobInput`)

**Variable Name in Bundle:** `wD = "Glob"`

**Full Description:**
```
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead
- You can call multiple tools in a single response. It is always better to speculatively perform multiple searches in parallel.
```

**Input Schema:**
```typescript
export interface GlobInput {
  pattern: string;          // The glob pattern to match files against
  path?: string;           // The directory to search in (defaults to cwd)
}
```

---

### 5. Grep Tool (`GrepInput`)

**Variable Name in Bundle:** `_F = "Grep"`

**Full Description:**
```
A powerful search tool built on ripgrep

Usage:
- ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command. The Grep tool has been optimized for correct permissions and access.
- Supports full regex syntax (e.g., "log.*Error", "function\\s+\\w+")
- Filter files with glob parameter (e.g., "*.js", "**/*.tsx") or type parameter (e.g., "js", "py", "rust")
- Output modes: "content" shows matching lines, "files_with_matches" shows only file paths (default), "count" shows match counts
- Use Task tool for open-ended searches requiring multiple rounds
- Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping (use `interface\\{\\}` to find `interface{}` in Go code)
- Multiline matching: By default patterns match within single lines only. For cross-line patterns like `struct \\{[\\s\\S]*?field`, use `multiline: true`
```

**Input Schema:**
```typescript
export interface GrepInput {
  pattern: string;              // The regular expression pattern to search for in file contents
  path?: string;                // File or directory to search in (defaults to cwd)
  glob?: string;                // Glob pattern to filter files (e.g. "*.js", "*.{ts,tsx}")
  output_mode?: "content" | "files_with_matches" | "count";  // Default: "files_with_matches"
  "-B"?: number;                // Lines before match
  "-A"?: number;                // Lines after match
  "-C"?: number;                // Lines before AND after
  "-n"?: boolean;               // Show line numbers
  "-i"?: boolean;               // Case insensitive search
  type?: string;                // File type (js, py, rust, go, java, etc.)
  head_limit?: number;          // Limit output to first N lines/entries
  multiline?: boolean;          // Enable multiline mode (default: false)
}
```

---

### 6. Bash Tool (`BashInput`)

**Variable Name in Bundle:** `_4 = "Bash"` (inferred from context)

**Full Description:**
```
Executes a given bash command in a persistent shell session with optional timeout, ensuring proper handling and security measures.

IMPORTANT: This tool is for terminal operations like git, npm, docker, etc. DO NOT use it for file operations (reading, writing, editing, searching, finding files) - use the specialized tools for this instead.

Before executing the command, please follow these steps:

1. Directory Verification:
   - If the command will create new directories or files, first use `ls` to verify the parent directory exists and is the correct location
   - For example, before running "mkdir foo/bar", first use `ls foo` to check that "foo" exists and is the intended parent directory

2. Command Execution:
   - Always quote file paths that contain spaces with double quotes
   - After ensuring proper quoting, execute the command
   - Capture the output of the command

Usage notes:
  - The command argument is required
  - You can specify an optional timeout in milliseconds (up to 600000ms / 10 minutes). If not specified, commands will timeout after 120000ms (2 minutes)
  - It is very helpful if you write a clear, concise description of what this command does in 5-10 words
  - If the output exceeds 30000 characters, output will be truncated before being returned to you
  - You can use the `run_in_background` parameter to run the command in the background, which allows you to continue working while the command runs
  - Never use `run_in_background` to run 'sleep' as it will return immediately
  - Avoid using Bash with the `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, or `echo` commands, unless explicitly instructed
```

**Input Schema:**
```typescript
export interface BashInput {
  command: string;              // The command to execute
  timeout?: number;             // Optional timeout in milliseconds (max 600000)
  description?: string;         // Clear, concise description (5-10 words)
  run_in_background?: boolean;  // Set to true to run in background
}
```

---

### 7. Task Tool (`AgentInput`)

**Variable Name in Bundle:** `W3 = "Task"`

**Full Description:**
```
Launch a new agent to handle complex, multi-step tasks autonomously.

Available agent types and the tools they have access to:
- general-purpose: General-purpose agent for researching complex questions, searching for code, and executing multi-step tasks. When you are searching for a keyword or file and are not confident that you will find the right match in the first few tries use this agent to perform the search for you (Tools: *)
- Explore: Fast agent specialized for exploring codebases. Use this when you need to quickly find files by patterns (eg. "src/components/**/*.tsx"), search code for keywords (eg. "API endpoints"), or answer questions about the codebase (eg. "how do API endpoints work?"). When calling this agent, specify the desired thoroughness level: "quick" for basic searches, "medium" for moderate exploration, or "very thorough" for comprehensive analysis (Tools: Glob, Grep, Read, Bash)
- statusline-setup: Use this agent to configure the user's Claude Code status line setting (Tools: Read, Edit)
- output-style-setup: Use this agent to create a Claude Code output style (Tools: Read, Write, Edit, Glob, Grep)

When using the Task tool, you must specify a subagent_type parameter to select which agent type to use.

When NOT to use the Agent tool:
- If you want to read a specific file path, use the Read or Glob tool instead of the Agent tool, to find the match more quickly
- If you are searching for a specific class definition like "class Foo", use the Glob tool instead, to find the match more quickly
- If you are searching for code within a specific file or set of 2-3 files, use the Read tool instead of the Agent tool, to find the match more quickly
- Other tasks that are not related to the agent descriptions above

Usage notes:
- Launch multiple agents concurrently whenever possible, to maximize performance; to do that, use a single message with multiple tool uses
- When the agent is done, it will return a single message back to you. The result returned by the agent is not visible to the user. To show the user the result, you should send a text message back to the user with a concise summary of the result.
- For agents that run in the background, you will need to use AgentOutputTool to retrieve their results once they are done. You can continue to work while async agents run in the background - when you need their results to continue you can use AgentOutputTool in blocking mode to pause and wait for their results.
- Each agent invocation is stateless. You will not be able to send additional messages to the agent, nor will the agent be able to communicate with you outside of its final report. Therefore, your prompt should contain a highly detailed task description for the agent to perform autonomously and you should specify exactly what information the agent should return back to you in its final and only message to you.
- The agent's outputs should generally be trusted
- Clearly tell the agent whether you expect it to write code or just to do research (search, file reads, web fetches, etc.), since it is not aware of the user's intent
- If the agent description mentions that it should be used proactively, then you should try your best to use it without the user having to ask for it first. Use your judgement.
- If the user specifies that they want you to run agents "in parallel", you MUST send a single message with multiple Task tool use content blocks. For example, if you need to launch both a code-reviewer agent and a test-runner agent in parallel, send a single message with both tool calls.
```

**Input Schema:**
```typescript
export interface AgentInput {
  description: string;      // A short (3-5 word) description of the task
  prompt: string;           // The task for the agent to perform
  subagent_type: string;    // The type of specialized agent to use for this task
}
```

**Agent Colors (for subagents):**
```typescript
const n61 = ["red","blue","green","yellow","purple","orange","pink","cyan"];
const as1 = {
  red: "red_FOR_SUBAGENTS_ONLY",
  blue: "blue_FOR_SUBAGENTS_ONLY",
  green: "green_FOR_SUBAGENTS_ONLY",
  yellow: "yellow_FOR_SUBAGENTS_ONLY",
  purple: "purple_FOR_SUBAGENTS_ONLY",
  orange: "orange_FOR_SUBAGENTS_ONLY",
  pink: "pink_FOR_SUBAGENTS_ONLY",
  cyan: "cyan_FOR_SUBAGENTS_ONLY"
};
```

---

### 8. TodoWrite Tool (`TodoWriteInput`)

**Full Description:**
```
Use this tool to create and manage a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.
It also helps the user understand the progress of the task and overall progress of their requests.

When to Use This Tool:
1. Complex multi-step tasks - When a task requires 3 or more distinct steps or actions
2. Non-trivial and complex tasks - Tasks that require careful planning or multiple operations
3. User explicitly requests todo list - When the user directly asks you to use the todo list
4. User provides multiple tasks - When users provide a list of things to be done (numbered or comma-separated)
5. After receiving new instructions - Immediately capture user requirements as todos
6. When you start working on a task - Mark it as in_progress BEFORE beginning work. Ideally you should only have one todo as in_progress at a time
7. After completing a task - Mark it as completed and add any new follow-up tasks discovered during implementation

When NOT to Use This Tool:
- There is only a single, straightforward task
- The task is trivial and tracking it provides no organizational benefit
- The task can be completed in less than 3 trivial steps
- The task is purely conversational or informational

Task Management:
- Update task status in real-time as you work
- Mark tasks complete IMMEDIATELY after finishing (don't batch completions)
- Exactly ONE task must be in_progress at any time (not less, not more)
- Complete current tasks before starting new ones
- Remove tasks that are no longer relevant from the list entirely

Task Completion Requirements:
- ONLY mark a task as completed when you have FULLY accomplished it
- If you encounter errors, blockers, or cannot finish, keep the task as in_progress
- When blocked, create a new task describing what needs to be resolved
- Never mark a task as completed if: Tests are failing, Implementation is partial, You encountered unresolved errors, You couldn't find necessary files or dependencies

Task Breakdown:
- Create specific, actionable items
- Break complex tasks into smaller, manageable steps
- Use clear, descriptive task names
- Always provide both forms: content and activeForm
```

**Input Schema:**
```typescript
export interface TodoWriteInput {
  todos: {
    content: string;                                   // Imperative form: what needs to be done
    status: "pending" | "in_progress" | "completed";   // Task status
    activeForm: string;                                // Present continuous: what's being done
  }[];
}
```

---

### 9. BashOutput Tool (`BashOutputInput`)

**Input Schema:**
```typescript
export interface BashOutputInput {
  bash_id: string;        // The ID of the background shell to retrieve output from
  filter?: string;        // Optional regex to filter output lines
}
```

---

### 10. KillShell Tool (`KillShellInput`)

**Input Schema:**
```typescript
export interface KillShellInput {
  shell_id: string;       // The ID of the background shell to kill
}
```

---

### 11. ExitPlanMode Tool (`ExitPlanModeInput`)

**Input Schema:**
```typescript
export interface ExitPlanModeInput {
  plan: string;         // The plan (supports markdown, should be concise)
}
```

---

### 12. NotebookEdit Tool (`NotebookEditInput`)

**Input Schema:**
```typescript
export interface NotebookEditInput {
  notebook_path: string;                          // The absolute path to .ipynb file
  cell_id?: string;                              // The ID of the cell to edit
  new_source: string;                            // The new source for the cell
  cell_type?: "code" | "markdown";               // Cell type
  edit_mode?: "replace" | "insert" | "delete";   // Edit type (default: replace)
}
```

---

### 13. WebFetch Tool (`WebFetchInput`)

**Input Schema:**
```typescript
export interface WebFetchInput {
  url: string;            // The URL to fetch content from
  prompt: string;         // The prompt to run on the fetched content
}
```

---

### 14. WebSearch Tool (`WebSearchInput`)

**Input Schema:**
```typescript
export interface WebSearchInput {
  query: string;                  // The search query to use
  allowed_domains?: string[];     // Only include results from these domains
  blocked_domains?: string[];     // Never include results from these domains
}
```

---

### 15. MCP Tools

**ListMcpResourcesInput:**
```typescript
export interface ListMcpResourcesInput {
  server?: string;      // Optional server name to filter resources by
}
```

**ReadMcpResourceInput:**
```typescript
export interface ReadMcpResourceInput {
  server: string;       // The MCP server name
  uri: string;          // The resource URI to read
}
```

**McpInput:**
```typescript
export interface McpInput {
  [k: string]: unknown;   // Dynamic MCP tool inputs
}
```

---

## SDK Information (from sdk.d.ts)

### Key Types

**Message Types:**
```typescript
export type SDKMessage =
  | SDKAssistantMessage
  | SDKUserMessage
  | SDKUserMessageReplay
  | SDKResultMessage
  | SDKSystemMessage
  | SDKPartialAssistantMessage
  | SDKCompactBoundaryMessage
  | SDKHookResponseMessage;
```

**Hook Events:**
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

**Permission Modes:**
```typescript
export type PermissionMode =
  | 'default'
  | 'acceptEdits'
  | 'bypassPermissions'
  | 'plan';
```

**MCP Server Configs:**
```typescript
export type McpServerConfig =
  | McpStdioServerConfig    // { type?: 'stdio', command, args?, env? }
  | McpSSEServerConfig      // { type: 'sse', url, headers? }
  | McpHttpServerConfig     // { type: 'http', url, headers? }
  | McpSdkServerConfigWithInstance;  // { type: 'sdk', name, instance }
```

### Query Function

```typescript
/**
 * @deprecated The Claude Code SDK is now the Claude Agent SDK!
 * Please install and use @anthropic-ai/claude-agent-sdk instead.
 * This SDK entrypoint will be going away on October 21.
 * See https://docs.claude.com/en/docs/claude-code/sdk/migration-guide for migration instructions.
 */
export declare function query({
  prompt,
  options,
}: {
  prompt: string | AsyncIterable<SDKUserMessage>;
  options?: Options;
}): Query;
```

### SDK Options

```typescript
export type Options = {
  abortController?: AbortController;
  additionalDirectories?: string[];
  allowedTools?: string[];
  appendSystemPrompt?: string;
  canUseTool?: CanUseTool;
  continue?: boolean;
  customSystemPrompt?: string;
  cwd?: string;
  disallowedTools?: string[];
  env?: { [envVar: string]: string | undefined };
  executable?: 'bun' | 'deno' | 'node';
  executableArgs?: string[];
  extraArgs?: Record<string, string | null>;
  fallbackModel?: string;
  forkSession?: boolean;
  hooks?: Partial<Record<HookEvent, HookCallbackMatcher[]>>;
  includePartialMessages?: boolean;
  maxThinkingTokens?: number;
  maxTurns?: number;
  mcpServers?: Record<string, McpServerConfig>;
  model?: string;
  pathToClaudeCodeExecutable?: string;
  permissionMode?: PermissionMode;
  permissionPromptToolName?: string;
  resume?: string;
  resumeSessionAt?: string;
  stderr?: (data: string) => void;
  strictMcpConfig?: boolean;
};
```

---

## Key Constants and Limits

From the bundled code:

- **Read Tool:**
  - Default line limit: `BG1 = 2000`
  - Character truncation per line: `eE9 = 2000`
  - PDF max size: `UOA = 33554432` (32MB)

- **Bash Tool:**
  - Default timeout: 120000ms (2 minutes)
  - Maximum timeout: 600000ms (10 minutes)
  - Output truncation: 30000 characters

---

## Tool Name Mappings in Bundle

The bundled code uses minified variable names:

- `x8 = "Read"`
- `CJ = "Write"`
- `R3 = "Edit"`
- `wD = "Glob"`
- `_F = "Grep"`
- `W3 = "Task"`
- And others...

These are internal to the bundle and not exposed to users.

---

## Research Notes

- The CLI is a single 9.7MB bundled JavaScript file
- Tool descriptions are embedded as template strings with variable interpolation
- The TypeScript definitions are auto-generated from JSON Schema
- SDK is deprecated in favor of `@anthropic-ai/claude-agent-sdk`
- Tools use JSON Schema Draft 07
- All tool input schemas are validated against TypeScript interfaces

---

**Extraction Method:**
1. Located global npm installation: `npm list -g --depth=0`
2. Read TypeScript definitions from `sdk-tools.d.ts`
3. Extracted tool descriptions from bundled `cli.js` using string search
4. Cross-referenced with existing documentation in `CLAUDE_CODE_INTERNAL_TOOLS_DEEP_DIVE.md`

**Validation:**
- ✅ All tool input schemas match TypeScript definitions
- ✅ Tool descriptions extracted from actual bundle
- ✅ Constants and limits verified from source code
- ✅ SDK deprecation notice confirmed
