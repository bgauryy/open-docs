# Claude Code Skills - Comprehensive Technical Documentation

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Skills Architecture](#skills-architecture)
3. [SKILL.md File Format](#skillmd-file-format)
4. [Skill Tool Implementation](#skill-tool-implementation)
5. [Skill Loading Mechanisms](#skill-loading-mechanisms)
6. [Skill vs Slash Command](#skill-vs-slash-command)
7. [Permission System](#permission-system)
8. [Execution Flow](#execution-flow)
9. [Source Code References](#source-code-references)
10. [Best Practices](#best-practices)

---

## Executive Summary

**Skills** are specialized, reusable prompt-based workflows in Claude Code that provide domain-specific capabilities. They are defined in `SKILL.md` files with frontmatter metadata and can be invoked via the Skill tool or used directly as commands.

### Key Facts

- **Tool Name**: `Skill` (prompt-command execution tool)
- **Input**: `{ skill: string, args?: string }` (leading `/` in `skill` is accepted)
- **File Format**: `SKILL.md` (case-insensitive) with YAML frontmatter + Markdown body
- **Default Locations**:
  - Project: `.claude/skills/<skill-name>/SKILL.md`
  - Personal: `~/.claude/skills/<skill-name>/SKILL.md`
  - Policy-managed skills may also be loaded (platform-dependent)
- **Invocation Surfaces**:
  - REPL “slash commands”: typing `/<skill> [args]` (can be disabled with `--disable-slash-commands`)
  - Model tool-use: the model calls the `Skill` tool directly
- **Forked Execution**: `context: fork` runs the skill in a sub-agent and returns `status: "forked"` with `agentId` + `result`
- **Permissions**:
  - Skill invocation is governed by `Skill`-tool permission rules (allow/deny, supports `:*` wildcards)
  - `allowed-tools` in a skill can widen the in-memory tool allowlist during skill execution
- **Hooks**: Skills can include a `hooks` frontmatter block; hooks are registered when the skill is invoked

---

## Skills Architecture

### Component Overview

```
+------------------------------+
|         REPL / Model         |
| "/<skill> [args]" or tool    |
+--------------+---------------+
               |
               v
+------------------------------+
|          Skill Tool          |
| validateInput + permissions  |
+--------------+---------------+
               |
               v
+------------------------------+
|       Resolve Skill          |
| policy/user/project/plugins  |
+--------------+---------------+
               |
               v
+------------------------------+
|   Build Prompt + Context     |
| base dir + arg substitution  |
| optional hooks registration  |
+--------------+---------------+
               |
     +---------+----------+
     |                    |
     v                    v
Inline (main convo)     Fork (sub-agent)
returns newMessages     returns result string
```

### Core Components

1. **Skill Tool (`Skill`)**
   - `@anthropic-ai/claude-code/cli.js` (search: `st = {`, `validateInput`, `checkPermissions`)

2. **Skill tool prompt + listing/budget**
   - `@anthropic-ai/claude-code/cli.js` (search: `SLASH_COMMAND_TOOL_CHAR_BUDGET`, `Execute a skill within the main conversation`)

3. **On-disk skills loader**
   - `@anthropic-ai/claude-code/cli.js` (search: `Loading skills from:`, `conditional skills stored`, `Activated conditional skill`)

4. **Plugin prompt-command loader**
   - `@anthropic-ai/claude-code/cli.js` (search: `SKILL.md`, `Failed to load skills`)

5. **Slash command parsing**
   - `@anthropic-ai/claude-code/cli.js` (search: `function pQ4`)

6. **Argument substitution**
   - `@anthropic-ai/claude-code/cli.js` (search: `function X01`)

---

## SKILL.md File Format

### File Structure

```markdown
---
name: skill-name
description: Brief description of what this skill does
when_to_use: Use when the user asks for <X>. Include trigger phrases and examples.
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash(git:*)
argument-hint: "[optional arguments]"
arguments:
  - arg1
  - arg2
# Optional execution mode:
context: fork
agent: general-purpose
model: inherit|sonnet|opus|haiku
user-invocable: true
disable-model-invocation: false
paths: "src/** docs/**"
version: 1.0.0
---

# Skill Content

This is the prompt content that will be expanded when the skill is invoked.
It can contain instructions, code examples, or any other guidance.
```

### Frontmatter Fields

**Source (v2.1.42)**: `@anthropic-ai/claude-code/cli.js`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Display name (defaults to the directory name / command name) |
| `description` | string | No | Brief description (if omitted, Claude derives one from content) |
| `when_to_use` | string | No | Guidance + trigger phrases; helps the model decide when to invoke |
| `allowed-tools` | string \| string[] | No | Tool permission patterns the skill needs (kept minimal) |
| `argument-hint` | string | No | Short hint shown with the skill (e.g. `\"[pr_number] [--flag]\"`) |
| `arguments` | string \| string[] | No | Named args for `$name` substitutions in the body |
| `context` | string | No | `fork` to run in a sub-agent; omit for inline execution |
| `agent` | string | No | Agent type to use when `context: fork` |
| `model` | string | No | Model override (`inherit` or a model alias) |
| `version` | string | No | Optional version label |
| `user-invocable` | boolean | No | Whether to show the skill in user-facing lists (defaults to true) |
| `disable-model-invocation` | boolean | No | If true, the `Skill` tool will refuse to run this skill |
| `paths` | string | No | Conditional activation patterns; activates when touched files match |
| `hooks` | object | No | Hooks to register when the skill is invoked |

Notes:
- Use `when_to_use` (underscore), not `when-to-use` (dash).
- `SKILL.md` is detected case-insensitively (e.g. `skill.md` works).
- `hooks` uses the same hook event names and hook definition structure described in `hooks-permissions-complete.md`, but is loaded from a skill’s frontmatter and registered at invocation time.

### Example SKILL.md

```markdown
---
name: pdf-analyzer
description: Analyze PDF documents and extract key information
when_to_use: Use when the user needs help understanding or extracting information from a PDF. Examples: "summarize this pdf", "extract tables", "what are the key points?"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash(pdftotext:*)
argument-hint: "[path] [optional focus]"
arguments:
  - path
  - focus
model: sonnet
version: 1.0.0
---

# PDF Analysis Skill

## Instructions

1. First, check if the PDF file exists
2. Extract text content using appropriate tools
3. Analyze the structure and extract key information
4. Present findings in a structured format
```

Note: When Claude builds the prompt for an on-disk skill, it prepends a line like `Base directory for this skill: <absolute path>` above the skill body; there is no `${baseDir}` variable substitution.

---

## Skill Tool Implementation

### Tool Definition

The `Skill` tool is implemented in `@anthropic-ai/claude-code/cli.js` (search: `st = {`).

Key behaviors:
- `validateInput`: trims and normalizes the skill name (leading `/` allowed), resolves the skill registry, rejects skills with `disable-model-invocation`.
- `checkPermissions`: checks allow/deny rules for the `Skill` tool; supports exact matches and `:*` wildcards; otherwise asks and suggests rules to add.
- `call`: executes inline (injects `newMessages`) or forked (sub-agent run) depending on skill `context`.

### Input Schema

```json
{ "skill": "commit", "args": "-m \"Fix bug\"" }
```

### Output Schema

The tool returns one of two shapes:
- **Inline**: `{ success, commandName, allowedTools?, model?, status?: "inline" }` plus injected `newMessages`
- **Forked**: `{ success, commandName, status: "forked", agentId, result }`

---

## Skill Loading Mechanisms

### Directory Structure

Claude assembles a registry of prompt-based commands (skills and commands) from multiple sources.

**On-disk skill roots (v2.1.42)**
- **Policy-managed** skills: a managed `.claude/skills` directory (platform-dependent base path)
- **User** skills: `~/.claude/skills`
- **Project** skills: `.claude/skills` in the current repo and in other configured project roots
- **Additional discovery**: `.claude/skills` directories can be discovered by walking up from allowed working directories (deepest paths take priority)

Each on-disk skill is a directory containing `SKILL.md` (case-insensitive). The directory name becomes the skill name.

**Deduplication**
- Skills are deduplicated by real path to the underlying `SKILL.md` file; the same file loaded from multiple sources is only loaded once.

**Conditional skills (`paths`)**
- If a skill declares `paths`, it is stored as conditional and becomes active only after a touched file path matches one of the patterns.

**Plugin-provided skills/commands**
- Enabled plugins can ship prompt commands (`.md`) and skills (`SKILL.md` in subdirectories). These are namespaced with `:` segments (e.g. `my-plugin:pdf`).

**Legacy commands**
- `.claude/commands` is also loaded as a legacy prompt-command source (marked internally as deprecated).

### Skill Naming Convention

Common patterns you will see:
- **On-disk skills**: `.claude/skills/<name>/SKILL.md` → `name`
- **Plugin skills/commands**: `plugin-name:<path>:<name>` (colon-separated namespaces)
- **Legacy `.claude/commands`**: nested paths become `:` segments (e.g. `review:pr`)

### isSkillMode Flag

There is no single `isSkill: true` marker that users configure. Internally, the loaders treat `SKILL.md` as “skill mode” (e.g. prepending a base-directory line and using skill-style naming), and treat regular `.md` files as prompt commands.

---

## Skill vs Slash Command

### Fundamental Difference

In v2.1.42, a “skill” is a **prompt command definition** (usually from `SKILL.md`), while a “slash command” is a **REPL input surface** (`/<name> ...`) that triggers a prompt command.

| Aspect | Skill (definition) | Slash command (surface) |
|--------|---------------------|--------------------------|
| What it is | A prompt-based command loaded into the command registry | A user input syntax in the REPL (`/<command> [args]`) |
| Where it lives | Files (`SKILL.md`, `.md`) or plugins | Typed by the user at runtime |
| Executor | The `Skill` tool | Typically translated into a `Skill` tool invocation |
| Args | Defined by the tool input (`args`) and substituted into the prompt | Parsed from the remainder of the line after the command name |
| Disable switch | `disable-model-invocation` blocks tool execution | `--disable-slash-commands` disables parsing/listing of slash commands |

### Shared Characteristics

Most prompt commands (skills and commands) share these characteristics:
1. Defined in Markdown with YAML frontmatter
2. Can request tool permissions via `allowed-tools`
3. Can optionally set `model` and execution mode (`context: fork`)
4. Can be namespaced (e.g. `plugin-name:subdir:command`)

### How Slash Commands Are Parsed

In the interactive REPL, a line starting with `/` is parsed into:
- `commandName`: the first token after `/`
- `args`: the remaining text (may be empty)

The parser also supports an `(MCP)` marker immediately after the command name: `/<name> (MCP) ...`.

Slash command parsing/listing can be disabled with `--disable-slash-commands`. When disabled, `/...` is treated as normal user text rather than a command invocation.

**Where to look**
- Slash command parser: `@anthropic-ai/claude-code/cli.js` (search: `function pQ4`)
- Flag wiring: `@anthropic-ai/claude-code/cli.js` (search: `--disable-slash-commands`)
- REPL integration: `@anthropic-ai/claude-code/cli.js` (search: `disableSlashCommands`)

---

## Permission System

The `Skill` tool has its own permission checks that decide whether the skill invocation itself is allowed.

### Validation

Before permissions are evaluated, the tool validates:
- The skill name is non-empty (leading `/` is allowed and stripped for lookup).
- The skill exists in the resolved skill/command registry.
- The resolved command is prompt-based and does **not** have `disable-model-invocation: true`.

### Tool Permission Rules (Allow/Deny)

The tool consults `toolPermissionContext` for rules targeting the `Skill` tool:
- **Deny rules** are checked first. If any matching rule is found, execution is blocked.
- **Allow rules** are checked next. If any matching rule is found, execution proceeds.

Matching behavior:
- Rules match against the normalized skill name (leading `/` stripped).
- Rules can be exact (`commit`) or namespace wildcards (`my-plugin:*`).

### Ask + Suggestions

If neither allow nor deny rules match, the tool typically asks for approval and suggests adding allow rules to local settings for:
- `<skill>`
- `<skill>:*`

### Interaction with `allowed-tools`

The skill’s own `allowed-tools` frontmatter is separate from the “may I run this skill?” decision:
- It is used to widen the in-memory tool allowlist during skill execution (and for inline skills, via the returned `contextModifier`).
- It does not by itself prevent other tools; it is an allowlist addition.

**Where to look**
- `@anthropic-ai/claude-code/cli.js` (search: `validateInput`, `checkPermissions`, `:*`)

---

## Execution Flow

The `Skill` tool has two execution modes: **inline** (default) and **forked** (`context: fork`).

### 1) Invocation

Skills can be invoked by:
- The user, via REPL slash commands: `/<skill> [args]`
- The model, via tool use: `{"skill": "<skill>", "args": "<optional args>"}`

### 2) Resolution + Validation

The tool:
1. Trims the skill name and strips a leading `/` (if present).
2. Loads the current registry of prompt commands (on-disk skills, plugin commands/skills, legacy commands).
3. Rejects unknown skills and rejects `disable-model-invocation` skills.

### 3) Prompt Construction (inline and forked)

For prompt-based skills, Claude builds the prompt text by:
- Prepending a base directory line for on-disk skills: `Base directory for this skill: <absolute path>`
- Applying argument substitution (see below)
- Replacing `${CLAUDE_SESSION_ID}`
- Running the standard template expansion used for prompt commands

### 4) Inline Execution (default)

Inline skills execute within the main conversation:
- The tool returns `newMessages` to inject into the conversation (these messages carry the expanded prompt).
- The tool output includes optional `allowedTools` and `model`.
- The tool also returns a `contextModifier` that merges `allowedTools` into the in-memory permission context.

### 5) Forked Execution (`context: fork`)

Forked skills execute in a sub-agent:
- The tool runs the skill prompt in a forked agent context (optionally using the skill’s `agent` and `model`).
- The tool returns `{ status: "forked", agentId, result }`.
- Forked skills do not inject `newMessages` into the main conversation.

### 6) Hooks Registration (`hooks`)

If the resolved skill defines `hooks`, they are registered when the skill is invoked.
- Hooks can be configured per hook event name.
- Hooks with `once: true` are removed after the first run.

**Where to look**
- Tool execution and fork handling: `@anthropic-ai/claude-code/cli.js`
- Skill hook registration: `@anthropic-ai/claude-code/cli.js` (search: `function iW6`)

### Argument Substitution Details (`args`)

Arguments are provided as a raw string (`args`) and tokenized with shell-like quoting rules.

Supported substitutions in the skill body:
- `$ARGUMENTS` → the raw args string
- `$1`, `$2`, ... → positional tokens
- `$ARGUMENTS[0]`, `$ARGUMENTS[1]`, ... → positional tokens
- If `arguments:` is set in frontmatter (e.g. `arguments: [pr_number, message]`), then `$pr_number` maps to token 1, `$message` maps to token 2, etc.

If `args` is provided and none of the substitutions apply, Claude appends a line like:

```
ARGUMENTS: <args>
```

**Where to look**
- Substitution logic: `@anthropic-ai/claude-code/cli.js` (search: `function X01`)

---

## Source Code References

### Key Modules (v2.1.42)

- Skill tool (schemas, validation, permissions, inline vs fork): `@anthropic-ai/claude-code/cli.js` (search: `st = {`)
- Skill tool prompt + listing/char budget: `@anthropic-ai/claude-code/cli.js` (search: `SLASH_COMMAND_TOOL_CHAR_BUDGET`, `Execute a skill within the main conversation`)
- On-disk skills loader + conditional activation: `@anthropic-ai/claude-code/cli.js` (search: `Loading skills from:`)
- Plugin prompt-command loader (plugin commands + plugin skills): `@anthropic-ai/claude-code/cli.js` (search: `SKILL.md`, `Failed to load skills`)
- Slash command parsing + hook registration helper: `@anthropic-ai/claude-code/cli.js` (search: `function pQ4`, `function iW6`)
- Argument substitution: `@anthropic-ai/claude-code/cli.js` (search: `function X01`)

### Useful Search Strings

- `Unknown skill:`
- `Skill ${z} cannot be used with Skill tool due to disable-model-invocation`
- `Registered ${w} hooks from skill`
- `Loading skills from: managed=`

---

## Best Practices

### Creating Skills That Work Well

1. **Name for invocation**: The directory name is the invocation name for on-disk skills. Keep it short and stable.
2. **Write a strong `when_to_use`**: Start with “Use when…”, include trigger phrases and examples. This is what helps the model decide to invoke the skill.
3. **Keep `allowed-tools` minimal**: Use specific patterns (e.g. `Bash(git:*)`) rather than broad permissions.
4. **If the skill has args, make them explicit**:
   - Add `argument-hint` and `arguments`
   - Use `$ARGUMENTS`, `$1`, `$ARGUMENTS[0]`, or `$<name>` in the body
5. **Pick inline vs fork intentionally**:
   - Inline is best when the user wants to steer mid-process.
   - Fork (`context: fork`) is best for self-contained tasks that can run without mid-process user input.
6. **Use `user-invocable` and `paths` to reduce clutter**:
   - `user-invocable: false` hides a skill from user-facing lists.
   - `paths` makes a skill conditional so it is only activated when relevant files are touched.

### SKILL.md Template

```markdown
---
description: Brief one-line description
when_to_use: Use when ...
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash(git:*)
argument-hint: "[arg1] [arg2]"
arguments:
  - arg1
  - arg2
# Optional:
# context: fork
# agent: general-purpose
# model: inherit
---

# Skill Title

## Goal
What the workflow should accomplish and what “done” means.

## Steps
1. ...

## Notes
- Base directory is provided in the prompt as: `Base directory for this skill: <path>`
- Args are available as `$ARGUMENTS`, `$1`, `$ARGUMENTS[0]`, and `$<name>` (when `arguments` is set)
```

### Recommended Layouts

Project-scoped skills:
```
.claude/skills/<skill-name>/SKILL.md
```

Personal skills:
```
~/.claude/skills/<skill-name>/SKILL.md
```

### Performance / UX Considerations

- **Skill listing char budget**: the “available skills” listing is truncated to a character budget (default `16000`) and can be overridden via `SLASH_COMMAND_TOOL_CHAR_BUDGET`.
- **Concurrency**: the `Skill` tool is not concurrency-safe (do not assume multiple skills can run in parallel in the same thread).
- **Conditional skills**: prefer `paths` for repo-specific skills to keep irrelevant skills out of the model’s prompt until needed.

---

## Advanced Topics

### Conditional Skills (`paths`)

If a skill includes `paths`, it is stored as conditional and is activated only after a touched file path matches one of the patterns. This helps keep the “available skills” list small and relevant.

**Where to look**
- Conditional storage/activation: `@anthropic-ai/claude-code/cli.js` (search: `conditional skills stored`, `Activated conditional skill`)

### Plugin Namespacing and Permission Wildcards

Plugin prompt commands and skills are typically namespaced with `:` segments (e.g. `my-plugin:pdf`). Permission rules support `:*` wildcards (e.g. `my-plugin:*`) to allow or deny entire namespaces.

### Forked Skills (`context: fork`)

Forked skills are executed in a sub-agent and return a summarized result string instead of injecting `newMessages` into the main conversation. Use this for workflows that should not interleave with the user’s current turn-by-turn interaction.

### Disabling Slash Commands

`--disable-slash-commands` disables parsing/listing of REPL slash commands. It does not remove the underlying `Skill` tool from the runtime; it changes how user input is interpreted and which commands are exposed as slash commands.

---

## Troubleshooting

### Common Issues

1. **"Unknown skill" Error**
   - Verify the skill is in a loaded location (e.g. `.claude/skills/<name>/SKILL.md` or `~/.claude/skills/<name>/SKILL.md`)
   - If it’s a plugin skill, verify the plugin is enabled and you’re using the fully qualified name (e.g. `my-plugin:pdf`)
   - If the skill is conditional (`paths`), it may not be active until relevant files are touched

2. **"disable-model-invocation" Error**
   - Skill has `disable-model-invocation: true`
   - Remove the field (or set it to false) to allow running the skill via the `Skill` tool

3. **Permission Denied**
   - Check permission rules in settings
   - Look for deny rules matching skill name or pattern

4. **Skill Not Appearing**
   - The skill may be hidden via `user-invocable: false`
   - The skill may be conditional (`paths`) and not yet activated
   - The “available skills” listing is subject to a char budget; less relevant skills may be omitted from the list

5. **Slash commands not working**
   - Check whether `--disable-slash-commands` is enabled for the session
   - Ensure you are using `/<skill> ...` format (command name first token, args after)

---

## Conclusion

The Claude Code Skills system provides a powerful way to package and reuse specialized AI workflows. By understanding the SKILL.md format, loading mechanisms, and execution flow, you can create effective skills that enhance Claude Code's capabilities.

**Key Takeaways**:
- Skills are defined in `SKILL.md` with frontmatter that guides discovery (`when_to_use`), permissions (`allowed-tools`), and execution mode (`context: fork`).
- The `Skill` tool handles validation, permission checks, and execution (inline or forked).
- REPL slash commands (`/<skill> [args]`) are a user-facing surface that typically maps to invoking the `Skill` tool.
- Arguments can be substituted into the skill body via `$ARGUMENTS`, `$1`, `$ARGUMENTS[0]`, and `$<name>` (with `arguments`).
- Skills can register hooks at invocation time via `hooks` frontmatter, and can be made conditional with `paths`.
