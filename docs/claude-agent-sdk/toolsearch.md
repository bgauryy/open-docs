# ToolSearch in Claude Code (v2.1.42)

> How Claude Code discovers and selects deferred tools at runtime.

**Source (v2.1.42)**: `@anthropic-ai/claude-code/cli.js` (search: `QW = 'ToolSearch'`, `name: QW`, `Query to find deferred tools`)

---

## Table of Contents

1. [What ToolSearch Is](#what-toolsearch-is)
2. [When to Use It](#when-to-use-it)
3. [Tool Interface (Input/Output)](#tool-interface-inputoutput)
4. [Query Syntax](#query-syntax)
5. [Behavior and Limits](#behavior-and-limits)
6. [Troubleshooting](#troubleshooting)
7. [Examples](#examples)

---

## What ToolSearch Is

Claude Code includes a tool named `ToolSearch` that helps the agent discover tools that are not immediately loaded into the active tool list (often referred to as “deferred tools”).

The result of ToolSearch is a list of matching tool names. In addition, a special `select:` query can be used to select a specific tool by name.

**Source (v2.1.42)**:
- Tool schema: search `(UH9 = L6(() => x.object({` and `(pH9 = L6(() => x.object({` near `name: QW`.
- Selection parsing: search `Y.match(/^select:(.+)$/i)` near `name: QW`.

---

## When to Use It

ToolSearch is most useful when:
- You know a tool exists (for example, provided by an MCP server or integration) but it is not available yet.
- The runtime prompts you to “load” or “select” a tool first.

In particular, Claude Code explicitly instructs that Chrome-browser tools must be loaded via ToolSearch before use.

**Source (v2.1.42)**: search `Before using any chrome browser tools, you MUST first load them using ToolSearch.` in `@anthropic-ai/claude-code/cli.js`.

---

## Tool Interface (Input/Output)

### Input schema

**Source (v2.1.42)**: search `(UH9 = L6(() => x.object({` in `@anthropic-ai/claude-code/cli.js`.

```ts
type ToolSearchInput = {
  query: string;          // keywords, or select:<tool_name>
  max_results?: number;   // default: 5
};
```

### Output schema

**Source (v2.1.42)**: search `(pH9 = L6(() => x.object({` in `@anthropic-ai/claude-code/cli.js`.

```ts
type ToolSearchOutput = {
  matches: string[];            // tool names
  query: string;
  total_deferred_tools: number;
};
```

---

## Query Syntax

ToolSearch supports two primary query modes:

### 1) Keyword search

Provide a keyword string and ToolSearch returns up to `max_results` matches from the deferred tool set.

**Source (v2.1.42)**: the schema description includes: `Use "select:<tool_name>" for direct selection, or keywords to search.`

### 2) Direct selection

Use:

```
select:<tool_name>
```

ToolSearch then attempts to find a deferred tool with `name === <tool_name>`.

**Source (v2.1.42)**: search `let O = Y.match(/^select:(.+)$/i);` and `w.find((j) => j.name === J)` near `name: QW`.

---

## Behavior and Limits

### Read-only and concurrency-safe

ToolSearch declares itself read-only and concurrency-safe.

**Source (v2.1.42)**: search `isReadOnly() { return !0; }` and `isConcurrencySafe() { return !0; }` near `name: QW`.

### Enablement and availability

ToolSearch can be disabled at runtime (for example, if the ToolSearch tool itself is not available due to tool restrictions).

**Source (v2.1.42)**: search `Tool search disabled: ToolSearchTool is not available` in `@anthropic-ai/claude-code/cli.js`.

### Result sizes

ToolSearch limits result block size and defaults `max_results` to 5.

**Source (v2.1.42)**: search `maxResultSizeChars: 1e5` near `name: QW` and `default(5)` in the ToolSearch input schema.

---

## Troubleshooting

### “select failed - tool not found”

If `select:<tool_name>` doesn’t match any deferred tool name, selection fails.

**Source (v2.1.42)**: search `ToolSearchTool: select failed - tool not found` in `@anthropic-ai/claude-code/cli.js`.

---

## Examples

### Keyword search

```ts
ToolSearch({ query: 'chrome' });
```

### Direct selection

```ts
ToolSearch({ query: 'select:mcp__claude-in-chrome__tabs_context_mcp' });
```
