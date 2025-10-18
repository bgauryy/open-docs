# Claude Agent SDK - Complete Prompts and System Instructions

Generated: 2025-10-18T16:33:15.210Z

---

## Primary System Prompts

These are the main system prompts used based on context:

### Default (Claude Code) (`mOA`)

```
You are Claude Code, Anthropic's official CLI for Claude.
```

### With SDK Context (`Nw9`)

```
You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.
```

### Agent Mode (`dOA`)

```
You are a Claude agent, built on Anthropic's Claude Agent SDK.
```

## Specialized Agent Prompts

Prompts for specific agent types and tasks:

### Bash Output Analyzer

```
You are analyzing output from a bash command to determine if it should be summarized.

Your task is to:
1. Determine if the output contains mostly repetitive logs, verbose build output, or other "log spew"
2. If it does, extract only the relevant information (errors, test results, completion status, etc.)
3. Consider the conversation context - if the user specifically asked to see detailed output, preserve it

You MUST output your response using XML tags in the following format:
<should_summarize>true/false</should_summarize>
<reason>reason for why you decided to summarize or not summarize the output</reason>
<summary>markdown summary as described below (only if should_summarize is true)</summary>

If should_summarize is true, include all three tags with a comprehensive summary.
If should_summarize is false, include only the first two tags and omit the summary tag.

Summary: The summary should be extremely comprehensive and detailed in markdown format. Especially consider the converstion context to determine what to focus on.
Freely copy parts of the output verbatim into the summary if you think it is relevant to the conversation context or what the user is asking for.
It's fine if the summary is verbose. The summary should contain the following sections: (Make sure to include all of these sections)
1. Overview: An overview of the output including the most interesting information summarized.
2. Detailed summary: An extremely detailed summary of the output.
3. Errors: List of relevant errors that were encountered. Inclu
```

### Conversation Summarizer

```
You are a helpful AI assistant tasked with summarizing conversations."],maxThinkingTokens:0,tools:[w6],signal:B.abortController.signal,options:{async getToolPermissionContext(){return(await B.getAppState()).toolPermissionContext},model:j3(),toolChoice:void 0,isNonInteractiveSession:B.options.isNonInteractiveSession,hasAppendSystemPrompt:B.options.hasAppendSystemPrompt,maxOutputTokensOverride:a$1,querySource:"compact",agents:B.options.agentDefinitions.activeAgents}})[Symbol.asyncIterator](),z=await H.next(),C=!1,q;while(!z.done){let g=z.value;if(!C&&g.type==="stream_event"&&g.event.type==="content_block_start"&&g.event.content_block.type==="text")C=!0,B.setStreamMode?.("responding");if(g.type==="stream_event"&&g.event.type==="content_block_delta"&&g.event.delta.type==="text_delta"){let t=g.event.delta.text.length;B.setResponseLength?.((d)=>d+t)}if(g.type===
```

### Git PR Comment Fetcher

```
You are an AI assistant integrated into a git-based version control system. Your task is to fetch and display comments from a GitHub pull request.

Follow these steps:

1. Use \
```

### Code Reviewer

```
You are an expert code reviewer. Follow these steps:

      1. If no PR number is provided in the args, use ${I9.name}("gh pr list") to show open PRs
      2. If a PR number is provided, use ${I9.name}("gh pr view <number>") to get PR details
      3. Use ${I9.name}("gh pr diff <number>") to get the diff
      4. Analyze the changes and provide a thorough code review that includes:
         - Overview of what the PR does
         - Analysis of code quality and style
         - Specific suggestions for improvements
         - Any potential issues or risks
      
      Keep your review concise but thorough. Focus on:
      - Code correctness
      - Following project conventions
      - Performance implications
      - Test coverage
      - Security considerations

      Format your review with clear sections and bullet points.

      PR number: ${A}
```

### Session Title Generator

```
You are coming up with a succinct title for a coding session based on the provided description. The title should be clear, concise, and accurately reflect the content of the coding task.
You should keep it short and simple, ideally no more than 4 words. Avoid using jargon or overly technical terms unless absolutely necessary. The title should be easy to understand for anyone reading it.
You should wrap the title in <title> XML tags. You MUST return your best attempt for the title.

For example:
<title>Fix login button not working on mobile</title>
<title>Update README with installation instructions</title>
<title>Improve performance of data processing script</title>
```

### General Task Agent

```
You are an agent for Claude Code, Anthropic's official CLI for Claude. Given the user's message, you should use the tools available to complete the task. Do what has been asked; nothing more, nothing less. When you complete the task simply respond with a detailed writeup.

Your strengths:
- Searching for code, configurations, and patterns across large codebases
- Analyzing multiple files to understand system architecture
- Investigating complex questions that require exploring many files
- Performing multi-step research tasks

Guidelines:
- For file searches: Use Grep or Glob when you need to search broadly. Use Read when you know the specific file path.
- For analysis: Start broad and narrow down. Use multiple search strategies if the first doesn't yield results.
- Be thorough: Check multiple locations, consider different naming conventions, look for related files.
- NEVER create files unless they're absolutely necessary for achieving your goal. ALWAYS prefer editing an existing file to creating a new one.
- NEVER proactively create documentation files (*.md) or README files. Only create documentation files if explicitly requested.
- In your final response always share relevant file names and code snippets. Any file paths you return in your response MUST be absolute. Do NOT use relative paths.
- For clear communication, avoid using emojis.
```

### File Search Specialist

```
You are a file search specialist for Claude Code, Anthropic's official CLI for Claude. You excel at thoroughly navigating and exploring codebases.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

Guidelines:
- Use ${ND} for broad file pattern matching
- Use ${bF} for searching file contents with regex
- Use ${x8} when you know the specific file path you need to read
- Use ${q4} for file operations like copying, moving, or listing directory contents
- Adapt your search approach based on the thoroughness level specified by the caller
- Return file paths as absolute paths in your final response
- For clear communication, avoid using emojis
- Do not create any files, or run bash commands that modify the user's system state in any way

Complete the user's search request efficiently and report your findings clearly.
```

### Agent Architect

```
You are an elite AI agent architect specializing in crafting high-performance agent configurations. Your expertise lies in translating user requirements into precisely-tuned agent specifications that maximize effectiveness and reliability.

**Important Context**: You may have access to project-specific instructions from CLAUDE.md files and other context that may include coding standards, project structure, and custom requirements. Consider this context when creating agents to ensure they align with the project's established patterns and practices.

When a user describes what they want an agent to do, you will:

1. **Extract Core Intent**: Identify the fundamental purpose, key responsibilities, and success criteria for the agent. Look for both explicit requirements and implicit needs. Consider any project-specific context from CLAUDE.md files. For agents that are meant to review code, you should assume that the user is asking to review recently written code and not the whole codebase, unless the user has explicitly instructed you otherwise.

2. **Design Expert Persona**: Create a compelling expert identity that embodies deep domain knowledge relevant to the task. The persona should inspire confidence and guide the agent's decision-making approach.

3. **Architect Comprehensive Instructions**: Develop a system prompt that:
   - Establishes clear behavioral boundaries and operational parameters
   - Provides specific methodologies and best practices for task execution
   - Anticipates edge cases and provides guidance for handling them
   - Incorporates any specific requirements or preferences mentioned by the user
   - Defines output format expectations when relevant
   - Aligns with project-specific coding standards and patterns from CLAUDE.md

4. **Optimize for Performance**: Include:
   - Decision-making frameworks appropriate to the domain
   - Quality control mechanisms and self-verification steps
   - Efficient workflow patterns
   - Clear escalation or fallback strategies

5. **Create Identifier**: Design a conc
```

### Status Line Setup Agent

```
You are a status line setup agent for Claude Code. Your job is to create or update the statusLine command in the user's Claude Code settings.

When asked to convert the user's shell PS1 configuration, follow these steps:
1. Read the user's shell configuration files in this order of preference:
   - ~/.zshrc
   - ~/.bashrc  
   - ~/.bash_profile
   - ~/.profile

2. Extract the PS1 value using this regex pattern: /(?:^|\\n)\\s*(?:export\\s+)?PS1\\s*=\\s*["']([^"']+)["']/m

3. Convert PS1 escape sequences to shell commands:
   - \\u → $(whoami)
   - \\h → $(hostname -s)  
   - \\H → $(hostname)
   - \\w → $(pwd)
   - \\W → $(basename "$(pwd)")
   - \\$ → $
   - \\n → \\n
   - \\t → $(date +%H:%M:%S)
   - \\d → $(date "+%a %b %d")
   - \\@ → $(date +%I:%M%p)
   - \\# → #
   - \\! → !

4. When using ANSI color codes, be sure to use \
```

### Git History Analyzer

```
You are an expert at analyzing git history. Given a list of files and their modification counts, return exactly five filenames that are frequently modified and represent core application logic (not auto-generated files, dependencies, or configuration). Make sure filenames are diverse, not all in the same folder, and are a mix of user and other users. Return only the filenames' basenames (without the path) separated by newlines with no explanation."],userPrompt:A,signal:new AbortController().signal,options:{querySource:"example_commands_frequently_modified",agents:[],isNonInteractiveSession:!1,hasAppendSystemPrompt:!1}})).message.content[0];if(!G||G.type!=="text")return[];let Y=G.text.trim().split(
```


---

## TypeScript Type Definitions

Complete type definitions from the SDK:

### SDK Main Types (`sdk.d.ts`)

```typescript
import type { Options as BaseOptions, Query, SDKUserMessage } from './sdkTypes.js';
export type AgentDefinition = {
    description: string;
    tools?: string[];
    prompt: string;
    model?: 'sonnet' | 'opus' | 'haiku' | 'inherit';
};
export type SettingSource = 'user' | 'project' | 'local';
export type Options = Omit<BaseOptions, 'customSystemPrompt' | 'appendSystemPrompt'> & {
    agents?: Record<string, AgentDefinition>;
    settingSources?: SettingSource[];
    systemPrompt?: string | {
        type: 'preset';
        preset: 'claude_code';
        append?: string;
    };
};
export declare function query(_params: {
    prompt: string | AsyncIterable<SDKUserMessage>;
    options?: Options;
}): Query;
export type { NonNullableUsage, ModelUsage, ApiKeySource, ConfigScope, McpStdioServerConfig, McpSSEServerConfig, McpHttpServerConfig, McpSdkServerConfig, McpSdkServerConfigWithInstance, McpServerConfig, McpServerConfigForProcessTransport, PermissionBehavior, PermissionUpdate, PermissionResult, PermissionRuleValue, CanUseTool, HookEvent, HookCallback, HookCallbackMatcher, BaseHookInput, PreToolUseHookInput, PostToolUseHookInput, NotificationHookInput, UserPromptSubmitHookInput, SessionStartHookInput, StopHookInput, SubagentStopHookInput, PreCompactHookInput, ExitReason, SessionEndHookInput, HookInput, AsyncHookJSONOutput, SyncHookJSONOutput, HookJSONOutput, PermissionMode, SlashCommand, ModelInfo, McpServerStatus, SDKMessageBase, SDKUserMessage, SDKUserMessageReplay, SDKAssistantMessage, SDKPermissionDenial, SDKResultMessage, SDKSystemMessage, SDKPartialAssistantMessage, SDKCompactBoundaryMessage, SDKMessage, Query, } from './sdkTypes.js';
export { HOOK_EVENTS, EXIT_REASONS, tool, createSdkMcpServer, AbortError, } from './sdkTypes.js';

```

### SDK Types (`sdkTypes.d.ts`)

```typescript
import type { MessageParam as APIUserMessage } from '@anthropic-ai/sdk/resources';
import type { BetaMessage as APIAssistantMessage, BetaUsage as Usage, BetaRawMessageStreamEvent as RawMessageStreamEvent } from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs';
import type { UUID } from 'crypto';
import type { CallToolResult } from '@modelcontextprotocol/sdk/types.js';
import { type McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { type z, type ZodRawShape, type ZodObject } from 'zod';
export type NonNullableUsage = {
    [K in keyof Usage]: NonNullable<Usage[K]>;
};
export type ModelUsage = {
    inputTokens: number;
    outputTokens: number;
    cacheReadInputTokens: number;
    cacheCreationInputTokens: number;
    webSearchRequests: number;
    costUSD: number;
    contextWindow: number;
};
export type ApiKeySource = 'user' | 'project' | 'org' | 'temporary';
export type ConfigScope = 'local' | 'user' | 'project';
export type McpStdioServerConfig = {
    type?: 'stdio';
    command: string;
    args?: string[];
    env?: Record<string, string>;
};
export type McpSSEServerConfig = {
    type: 'sse';
    url: string;
    headers?: Record<string, string>;
};
export type McpHttpServerConfig = {
    type: 'http';
    url: string;
    headers?: Record<string, string>;
};
export type McpSdkServerConfig = {
    type: 'sdk';
    name: string;
};
export type McpSdkServerConfigWithInstance = McpSdkServerConfig & {
    instance: McpServer;
};
export type McpServerConfig = McpStdioServerConfig | McpSSEServerConfig | McpHttpServerConfig | McpSdkServerConfigWithInstance;
export type McpServerConfigForProcessTransport = McpStdioServerConfig | McpSSEServerConfig | McpHttpServerConfig | McpSdkServerConfig;
type PermissionUpdateDestination = 'userSettings' | 'projectSettings' | 'localSettings' | 'session';
export type PermissionBehavior = 'allow' | 'deny' | 'ask';
export type PermissionUpdate = {
    type: 'addRules';
    rules: PermissionRuleValue[];
    behavior: PermissionBehavior;
    destination: PermissionUpdateDestination;
} | {
    type: 'replaceRules';
    rules: PermissionRuleValue[];
    behavior: PermissionBehavior;
    destination: PermissionUpdateDestination;
} | {
    type: 'removeRules';
    rules: PermissionRuleValue[];
    behavior: PermissionBehavior;
    destination: PermissionUpdateDestination;
} | {
    type: 'setMode';
    mode: PermissionMode;
    destination: PermissionUpdateDestination;
} | {
    type: 'addDirectories';
    directories: string[];
    destination: PermissionUpdateDestination;
} | {
    type: 'removeDirectories';
    directories: string[];
    destination: PermissionUpdateDestination;
};
export type PermissionResult = {
    behavior: 'allow';
    /**
     * Updated tool input to use, if any changes are needed.
     *
     * For example if the user was given the option to update the tool use
     * input before approving, then this would be the updated input which
     * would be executed by the tool.
     */
    updatedInput: Record<string, unknown>;
    /**
     * Permissions updates to be applied as part of accepting this tool use.
     *
     * Typically this is used as part of the 'always allow' flow and these
     * permission updates are from the `suggestions` field from the
     * CanUseTool callback.
     *
     * It is recommended that you use these suggestions rather than
     * attempting to re-derive them from the tool use input, as the
     * suggestions may include other permission changes such as adding
     * directories or incorporate complex tool-use logic such as bash
     * commands.
     */
    updatedPermissions?: PermissionUpdate[];
} | {
    behavior: 'deny';
    /**
     * Message indicating the reason for denial, or guidance of what the
     * model should do instead.
     */
    message: string;
    /**
     * If true, interrupt execution and do not continue.
     *
     * Typically this should be set to true when the user says 'no' with no
     * further guidance. Leave unset or false if the user provides guidance
     * which the model should incorporate and continue.
     */
    interrupt?: boolean;
};
export type PermissionRuleValue = {
    toolName: string;
    ruleContent?: string;
};
export type CanUseTool = (toolName: string, input: Record<string, unknown>, options: {
    /** Signaled if the operation should be aborted. */
    signal: AbortSignal;
    /**
     * Suggestions for updating permissions so that the user will not be
     * prompted again for this tool during this session.
     *
     * Typically if presenting the user an option 'always allow' or similar,
     * then this full set of suggestions should be returned as the
     * `updatedPermissions` in the PermissionResult.
     */
    suggestions?: PermissionUpdate[];
}) => Promise<PermissionResult>;
export declare const HOOK_EVENTS: readonly ["PreToolUse", "PostToolUse", "Notification", "UserPromptSubmit", "SessionStart", "SessionEnd", "Stop", "SubagentStop", "PreCompact"];
export type HookEvent = (typeof HOOK_EVENTS)[number];
export type HookCallback = (input: HookInput, toolUseID: string | undefined, options: {
    signal: AbortSignal;
}) => Promise<HookJSONOutput>;
export interface HookCallbackMatcher {
    matcher?: string;
    hooks: HookCallback[];
}
export type BaseHookInput = {
    session_id: string;
    transcript_path: string;
    cwd: string;
    permission_mode?: string;
};
export type PreToolUseHookInput = BaseHookInput & {
    hook_event_name: 'PreToolUse';
    tool_name: string;
    tool_input: unknown;
};
export type PostToolUseHookInput = BaseHookInput & {
    hook_event_name: 'PostToolUse';
    tool_name: string;
    tool_input: unknown;
    tool_response: unknown;
};
export type NotificationHookInput = BaseHookInput & {
    hook_event_name: 'Notification';
    message: string;
    title?: string;
};
export type UserPromptSubmitHookInput = BaseHookInput & {
    hook_event_name: 'UserPromptSubmit';
    prompt: string;
};
export type SessionStartHookInput = BaseHookInput & {
    hook_event_name: 'SessionStart';
    source: 'startup' | 'resume' | 'clear' | 'compact';
};
export type StopHookInput = BaseHookInput & {
    hook_event_name: 'Stop';
    stop_hook_active: boolean;
};
export type SubagentStopHookInput = BaseHookInput & {
    hook_event_name: 'SubagentStop';
    stop_hook_active: boolean;
};
export type PreCompactHookInput = BaseHookInput & {
    hook_event_name: 'PreCompact';
    trigger: 'manual' | 'auto';
    custom_instructions: string | null;
};
export declare const EXIT_REASONS: string[];
export type ExitReason = (typeof EXIT_REASONS)[number];
export type SessionEndHookInput = BaseHookInput & {
    hook_event_name: 'SessionEnd';
    reason: ExitReason;
};
export type HookInput = PreToolUseHookInput | PostToolUseHookInput | NotificationHookInput | UserPromptSubmitHookInput | SessionStartHookInput | SessionEndHookInput | StopHookInput | SubagentStopHookInput | PreCompactHookInput;
export type AsyncHookJSONOutput = {
    async: true;
    asyncTimeout?: number;
};
export type SyncHookJSONOutput = {
    continue?: boolean;
    suppressOutput?: boolean;
    stopReason?: string;
    decision?: 'approve' | 'block';
    systemMessage?: string;
    reason?: string;
    hookSpecificOutput?: {
        hookEventName: 'PreToolUse';
        permissionDecision?: 'allow' | 'deny' | 'ask';
        permissionDecisionReason?: string;
        updatedInput?: Record<string, unknown>;
    } | {
        hookEventName: 'UserPromptSubmit';
        additionalContext?: string;
    } | {
        hookEventName: 'SessionStart';
        additionalContext?: string;
    } | {
        hookEventName: 'PostToolUse';
        additionalContext?: string;
    };
};
export type HookJSONOutput = AsyncHookJSONOutput | SyncHookJSONOutput;
export type Options = {
    abortController?: AbortController;
    additionalDirectories?: string[];
    allowedTools?: string[];
    appendSystemPrompt?: string;
    canUseTool?: Can
... (truncated for brevity)

```

### Tool Input Schemas (`sdk-tools.d.ts`)

```typescript
/* eslint-disable */
/**
 * This file was automatically generated by json-schema-to-typescript.
 * DO NOT MODIFY IT BY HAND. Instead, modify the source JSONSchema file,
 * and run json-schema-to-typescript to regenerate this file.
 */

/**
 * JSON Schema definitions for Claude CLI tool inputs
 */
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

export interface AgentInput {
  /**
   * A short (3-5 word) description of the task
   */
  description: string;
  /**
   * The task for the agent to perform
   */
  prompt: string;
  /**
   * The type of specialized agent to use for this task
   */
  subagent_type: string;
}
export interface BashInput {
  /**
   * The command to execute
   */
  command: string;
  /**
   * Optional timeout in milliseconds (max 600000)
   */
  timeout?: number;
  /**
   * Clear, concise description of what this command does in 5-10 words, in active voice. Examples:
   * Input: ls
   * Output: List files in current directory
   *
   * Input: git status
   * Output: Show working tree status
   *
   * Input: npm install
   * Output: Install package dependencies
   *
   * Input: mkdir foo
   * Output: Create directory 'foo'
   */
  description?: string;
  /**
   * Set to true to run this command in the background. Use BashOutput to read the output later.
   */
  run_in_background?: boolean;
}
export interface BashOutputInput {
  /**
   * The ID of the background shell to retrieve output from
   */
  bash_id: string;
  /**
   * Optional regular expression to filter the output lines. Only lines matching this regex will be included in the result. Any lines that do not match will no longer be available to read.
   */
  filter?: string;
}
export interface ExitPlanModeInput {
  /**
   * The plan you came up with, that you want to run by the user for approval. Supports markdown. The plan should be pretty concise.
   */
  plan: string;
}
export interface FileEditInput {
  /**
   * The absolute path to the file to modify
   */
  file_path: string;
  /**
   * The text to replace
   */
  old_string: string;
  /**
   * The text to replace it with (must be different from old_string)
   */
  new_string: string;
  /**
   * Replace all occurences of old_string (default false)
   */
  replace_all?: boolean;
}
export interface FileReadInput {
  /**
   * The absolute path to the file to read
   */
  file_path: string;
  /**
   * The line number to start reading from. Only provide if the file is too large to read at once
   */
  offset?: number;
  /**
   * The number of lines to read. Only provide if the file is too large to read at once.
   */
  limit?: number;
}
export interface FileWriteInput {
  /**
   * The absolute path to the file to write (must be absolute, not relative)
   */
  file_path: string;
  /**
   * The content to write to the file
   */
  content: string;
}
export interface GlobInput {
  /**
   * The glob pattern to match files against
   */
  pattern: string;
  /**
   * The directory to search in. If not specified, the current working directory will be used. IMPORTANT: Omit this field to use the default directory. DO NOT enter "undefined" or "null" - simply omit it for the default behavior. Must be a valid directory path if provided.
   */
  path?: string;
}
export interface GrepInput {
  /**
   * The regular expression pattern to search for in file contents
   */
  pattern: string;
  /**
   * File or directory to search in (rg PATH). Defaults to current working directory.
   */
  path?: string;
  /**
   * Glob pattern to filter files (e.g. "*.js", "*.{ts,tsx}") - maps to rg --glob
   */
  glob?: string;
  /**
   * Output mode: "content" shows matching lines (supports -A/-B/-C context, -n line numbers, head_limit), "files_with_matches" shows file paths (supports head_limit), "count" shows match counts (supports head_limit). Defaults to "files_with_matches".
   */
  output_mode?: "content" | "files_with_matches" | "count";
  /**
   * Number of lines to show before each match (rg -B). Requires output_mode: "content", ignored otherwise.
   */
  "-B"?: number;
  /**
   * Number of lines to show after each match (rg -A). Requires output_mode: "content", ignored otherwise.
   */
  "-A"?: number;
  /**
   * Number of lines to show before and after each match (rg -C). Requires output_mode: "content", ignored otherwise.
   */
  "-C"?: number;
  /**
   * Show line numbers in output (rg -n). Requires output_mode: "content", ignored otherwise.
   */
  "-n"?: boolean;
  /**
   * Case insensitive search (rg -i)
   */
  "-i"?: boolean;
  /**
   * File type to search (rg --type). Common types: js, py, rust, go, java, etc. More efficient than include for standard file types.
   */
  type?: string;
  /**
   * Limit output to first N lines/entries, equivalent to "| head -N". Works across all output modes: content (limits output lines), files_with_matches (limits file paths), count (limits count entries). When unspecified, shows all results from ripgrep.
   */
  head_limit?: number;
  /**
   * Enable multiline mode where . matches newlines and patterns can span lines (rg -U --multiline-dotall). Default: false.
   */
  multiline?: boolean;
}
export interface KillShellInput {
  /**
   * The ID of the background shell to kill
   */
  shell_id: string;
}
export interface ListMcpResourcesInput {
  /**
   * Optional server name to filter resources by
   */
  server?: string;
}
export interface McpInput {
  [k: string]: unknown;
}
export interface NotebookEditInput {
  /**
   * The absolute path to the Jupyter notebook file to edit (must be absolute, not relative)
   */
  notebook_path: string;
  /**
   * The ID of the cell to edit. When inserting a new cell, the new cell will be inserted after the cell
... (truncated for brevity)

```


---

## Hook Event Types

The SDK supports the following hook events:

### PreToolUse
Triggered before a tool is used

### PostToolUse
Triggered after a tool has been used

### Notification
When notifications are sent

### UserPromptSubmit
When the user submits a prompt

### SessionStart
When a new session is started

### SessionEnd
When a session is ending

### Stop
Right before Claude concludes its response

### SubagentStop
Right before a subagent (Task tool call) concludes its response

### PreCompact
Before conversation compaction


---

## Usage Notes

1. **System Prompts**: Three variants are used based on context:
   - Default: Standard Claude Code CLI
   - SDK Context: When running within the Claude Agent SDK
   - Agent Mode: For custom agents built on the SDK

2. **Agent Definitions**: Custom agents can specify:
   - `description`: What the agent does
   - `prompt`: The system prompt for the agent
   - `tools`: List of available tools
   - `model`: Preferred model (sonnet/opus/haiku/inherit)

3. **Tool Permissions**: The SDK includes a permission system for tool use

4. **Hooks**: Allow custom behavior at different points in the conversation flow

