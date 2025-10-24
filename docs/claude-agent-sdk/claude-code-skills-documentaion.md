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

- **Tool Name**: "Skill" (constant `sw` at line 2699)
- **File Format**: Markdown files named `SKILL.md` with YAML frontmatter
- **Discovery**: Automatically loaded from plugin directories and skill paths
- **Invocation**: Via Skill tool or as slash commands
- **Differentiation**: Marked with `isSkill: true` flag in metadata

---

## Skills Architecture

### Component Overview

```
┌─────────────────────────────────────────┐
│        Skill Tool (sw = "Skill")        │
│         (Line 2699-2750+)               │
└───────────┬─────────────────────────────┘
            │
            ├──> Input Validation (validateInput)
            ├──> Permission Checking (checkPermissions)
            └──> Skill Execution (call method)
                 │
                 ├──> Load Skill Definition (iM function)
                 ├──> Process Prompt (wo1 function)
                 └──> Return Messages & Context Modifiers
```

### Core Components

1. **Skill Constant** (Line 2699)
   ```javascript
   var sw="Skill"
   ```

2. **Skill Tool Object** (Lines 2750+)
   - `name`: sw ("Skill")
   - `inputSchema`: Zod schema for skill name
   - `outputSchema`: Success/failure result
   - `description`: Dynamic description based on skill name
   - `prompt`: Generates tool prompt with available skills
   - `validateInput`: Validates skill existence and configuration
   - `checkPermissions`: Handles permission system
   - `call`: Executes the skill

3. **SKILL.md Parser** (Line 2806+)
   - Function: `jwQ` (load skills from directory)
   - Reads `SKILL.md` files from directories
   - Parses frontmatter using `UD` function
   - Creates skill objects with metadata

---

## SKILL.md File Format

### File Structure

```markdown
---
name: skill-name
description: Brief description of what this skill does
when-to-use: When to invoke this skill
allowed-tools: tool1,tool2,tool3
argument-hint: [optional arguments]
version: 1.0.0
model: inherit|sonnet|opus|haiku
disable-model-invocation: false
---

# Skill Content

This is the prompt content that will be expanded when the skill is invoked.
It can contain instructions, code examples, or any other guidance.
```

### Frontmatter Fields

**Source**: Line 2806+ (jwQ function)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Display name (defaults to directory name) |
| `description` | string | Yes* | Brief description of the skill |
| `when-to-use` | string | No | Guidance on when to use this skill |
| `allowed-tools` | string (comma-separated) | No | Tools this skill can use |
| `argument-hint` | string | No | Hints for arguments |
| `version` | string | No | Skill version |
| `model` | string | No | Model preference: "inherit", "sonnet", "opus", "haiku" |
| `disable-model-invocation` | boolean | No | If true, skill cannot be invoked via Skill tool |

*If `description` is not provided, it falls back to using content or `when-to-use`

### Example SKILL.md

```markdown
---
name: pdf-analyzer
description: Analyze PDF documents and extract key information
when-to-use: When working with PDF files that need analysis or data extraction
allowed-tools: Read,Bash
model: sonnet
version: 1.0.0
---

# PDF Analysis Skill

## Instructions

1. First, check if the PDF file exists
2. Extract text content using appropriate tools
3. Analyze the structure and extract key information
4. Present findings in a structured format

## Base Directory

Base directory for this skill: ${baseDir}
```

---

## Skill Tool Implementation

### Tool Definition

**Location**: Lines 2750+ (`jo1` function)
**Reference**: Line 2738 shows the complete tool object structure

```javascript
ym = {
  name: sw,  // "Skill"
  inputSchema: W$8,  // Zod schema for {command: string}
  outputSchema: J$8,  // Zod schema for {success: boolean, commandName: string}

  description: async({command: A}) => `Execute skill: ${A}`,

  prompt: async() => q$Q(),  // Generates tool prompt

  userFacingName: () => sw,

  isConcurrencySafe: () => !1,  // false - not safe for concurrent execution

  isEnabled: () => !0,  // true - always enabled

  isReadOnly: () => !1,  // false - can modify state

  // Validation logic...
  async validateInput({command: A}, B) { /* ... */ },

  // Permission checking...
  async checkPermissions({command: A}, B) { /* ... */ },

  // Execution logic...
  async *call({command: A}, B) { /* ... */ },

  // Rendering methods...
  mapToolResultToToolResultBlockParam(A, B) { /* ... */ },
  renderToolResultMessage: Mo1,
  renderToolUseMessage: Oo1,
  renderToolUseProgressMessage: Ro1,
  renderToolUseRejectedMessage: To1,
  renderToolUseErrorMessage: Po1
}
```

### Input Schema

**Location**: Line 2750+

```javascript
W$8 = x.object({
  command: x.string().describe('The skill name (no arguments). E.g., "pdf" or "xlsx"')
})
```

### Output Schema

```javascript
J$8 = x.object({
  success: x.boolean().describe("Whether the skill is valid"),
  commandName: x.string().describe("The name of the skill")
})
```

---

## Skill Loading Mechanisms

### Directory Structure

Skills are loaded from these locations:

1. **Plugin Default Skills Path**: `plugin.skillsPath`
2. **Plugin Custom Skills Paths**: `plugin.skillsPaths[]`
3. **User/Project Settings**: Via configuration

### Loading Process

**Function**: `jwQ` (Line 2806+)

```javascript
async function jwQ(A, B, Q, Z) {
  let G = L1(), Y = [];

  // Check for single SKILL.md at root
  let W = d51(A, "SKILL.md");
  if (G.existsSync(W)) {
    // Load single skill from root
    let I = G.readFileSync(W, {encoding:"utf-8"}),
        {frontmatter:X, content:F} = UD(I),
        V = `${B}:${c51(A)}`,
        K = {filePath:W, baseDir:vm(W), frontmatter:X, content:F},
        D = oz1(V, K, Q, Z, !0, {isSkillMode:!0});
    if (D) Y.push(D);
    return Y;
  }

  // Scan subdirectories for SKILL.md files
  let J = G.readdirSync(A);
  for (let I of J) {
    if (!I.isDirectory() && !I.isSymbolicLink()) continue;
    let X = d51(A, I.name),
        F = d51(X, "SKILL.md");
    if (G.existsSync(F)) {
      // Load skill from subdirectory
      let V = G.readFileSync(F, {encoding:"utf-8"}),
          {frontmatter:K, content:D} = UD(V),
          H = `${B}:${I.name}`,
          z = {filePath:F, baseDir:vm(F), frontmatter:K, content:D},
          C = oz1(H, z, Q, Z, !0, {isSkillMode:!0});
      if (C) Y.push(C);
    }
  }

  return Y;
}
```

### Skill Naming Convention

**Pattern**: `{pluginName}:{skillDirectory}` or `{pluginName}:{skillName}`

Examples:
- `ms-office-suite:pdf`
- `my-plugin:data-analysis`
- Single skill: `my-plugin:skill-name`

### isSkillMode Flag

**Location**: Line 2806+

When creating a skill object, the function passes `{isSkillMode:!0}` (true) to differentiate skills from regular commands:

```javascript
D = oz1(H, z, Q, Z, !0, {isSkillMode:!0});
```

This sets the `isSkill` property on the command object.

---

## Skill vs Slash Command

### Fundamental Difference

| Aspect | Skills | Slash Commands |
|--------|--------|----------------|
| File Name | `SKILL.md` | Any `.md` file |
| Directory | Subdirectories or root | Direct .md files |
| isSkill Flag | `true` | Not set |
| Tool | "Skill" tool | "SlashCommand" tool |
| Invocation | Via Skill tool OR `/skillname` | Via SlashCommand tool OR `/commandname` |
| Display | Shown in `<available_skills>` | Shown in Available Commands |

### Shared Characteristics

Both skills and slash commands:
1. Are defined in Markdown files with frontmatter
2. Support the same frontmatter fields
3. Can be invoked as prompt-based commands
4. Support `allowed-tools`, `model`, `disable-model-invocation`
5. Are loaded from plugin directories

### Code Evidence

**Skills Tool Name** (Line 2699):
```javascript
var sw="Skill";
```

**Slash Command Tool Name** (Line 2755+):
```javascript
var Sj="SlashCommand";
```

**Progress Message Difference** (Line ~2690+):
```javascript
// For skills:
I = A.isSkill ? `The "${A.userFacingName()}" skill is ${A.progressMessage}`
              : `${A.userFacingName()}is ${A.progressMessage}…`

// Command name format:
X = A.isSkill ? `<command-name>${A.userFacingName()}</command-name>`
              : `<command-name>/${A.userFacingName()}</command-name>`
```

---

## Permission System

### Validation Flow

**Location**: Lines 2750+ (validateInput method)

```javascript
async validateInput({command: A}, B) {
  let Q = A.trim();

  // 1. Check for empty command
  if (!Q) return {result:!1, message:`Invalid skill format: ${A}`, errorCode:1};

  // 2. Remove leading slash if present
  let Z = Q.startsWith("/") ? Q.substring(1) : Q,
      G = await UH(); // Get all commands

  // 3. Check if skill exists
  if (!Ys(Z, G)) return {result:!1, message:`Unknown skill: ${Z}`, errorCode:2};

  // 4. Load skill definition
  let Y = iM(Z, G);
  if (!Y) return {result:!1, message:`Could not load skill: ${Z}`, errorCode:3};

  // 5. Check disable-model-invocation flag
  if (Y.disableModelInvocation)
    return {result:!1, message:`Skill ${Z} cannot be used with ${sw} tool due to disable-model-invocation`, errorCode:4};

  // 6. Ensure it's a prompt-based skill
  if (Y.type !== "prompt")
    return {result:!1, message:`Skill ${Z} is not a prompt-based skill`, errorCode:5};

  return {result:!0};
}
```

### Permission Checking

**Location**: Lines 2750+ (checkPermissions method)

```javascript
async checkPermissions({command: A}, B) {
  let Q = A.trim(),
      Z = Q.startsWith("/") ? Q.substring(1) : Q,
      Y = (await B.getAppState()).toolPermissionContext,
      W = await UH(),
      J = iM(Z, W);

  // Pattern matching function for wildcard permissions
  I = (K) => {
    if (K === A) return !0;
    if (K.endsWith(":*")) {
      let D = K.slice(0,-2);
      return A.startsWith(D);
    }
    return !1;
  };

  // Check deny rules
  let X = Pz(Y, ym, "deny");
  for (let[K,D] of X.entries())
    if (I(K)) return {behavior:"deny", message:"Skill execution blocked by permission rules",
                       decisionReason:{type:"rule", rule:D}};

  // Check allow rules
  let F = Pz(Y, ym, "allow");
  for (let[K,D] of F.entries())
    if (I(K)) return {behavior:"allow", updatedInput:{command:A},
                       decisionReason:{type:"rule", rule:D}};

  // Default: ask user
  let V = [{type:"addRules", rules:[{toolName:sw, ruleContent:A}],
            behavior:"allow", destination:"localSettings"}];

  return {behavior:"ask", message:`Execute skill: ${Z}`,
          decisionReason:void 0, suggestions:V, metadata:{command:J}};
}
```

### Permission Wildcards

Skills support wildcard permissions:
- `skill-name` - Exact match
- `plugin:*` - All skills from a plugin
- Leading slash handling: `/skillname` → `skillname`

---

## Execution Flow

### Skill Invocation

**Location**: Lines 2750+ (call method)

```javascript
async *call({command: A}, B) {
  // 1. Clean and parse command
  let Q = A.trim(),
      Z = Q.startsWith("/") ? Q.substring(1) : Q,
      G = await UH();

  // 2. Process skill command
  let Y = await wo1(Z, "", G, B);  // wo1 = process command function
  if (!Y.shouldQuery) throw Error("Command processing failed");

  // 3. Extract metadata
  let W = Y.allowedTools || [],
      J = Y.model,
      I = Y.maxThinkingTokens,
      X = Y.command.type === "prompt" && Y.command.isModeCommand === !0,
      F = fP().has(Z) ? Z : "custom";

  // 4. Track analytics
  Z1("tengu_skill_tool_invocation", {command_name:F, is_mode_command:X?1:0});

  // 5. Filter messages
  let V = Y.messages.filter((K) => K.type !== "progress");

  // 6. Yield result with context modifier
  yield {
    type: "result",
    data: {success:!0, commandName:Z},
    newMessages: V,
    contextModifier(K) {
      let D = K;

      // Add allowed tools to context
      if (W.length > 0) {
        D = {...D, async getAppState() {
          let H = await B.getAppState();
          return {...H, toolPermissionContext:{
            ...H.toolPermissionContext,
            alwaysAllowRules:{
              ...H.toolPermissionContext.alwaysAllowRules,
              command:[...new Set([...H.toolPermissionContext.alwaysAllowRules.command||[], ...W])]
            }
          }};
        }};
      }

      // Set model override
      if (J) D = {...D, options:{...D.options, mainLoopModel:J}};

      // Set thinking tokens limit
      if (I !== void 0) D = {...D, options:{...D.options, maxThinkingTokens:I}};

      return D;
    }
  };
}
```

### Message Processing

**Location**: Line ~2690+ (H$Q function)

```javascript
async function H$Q(A, B, Q, Z=[], G=[], Y, W) {
  // 1. Get skill prompt
  let J = await A.getPromptForCommand(B, Q);

  // 2. Create progress message
  let I = A.isSkill ? `The "${A.userFacingName()}" skill is ${A.progressMessage}`
                    : `${A.userFacingName()} is ${A.progressMessage}…`;

  // 3. Create command metadata
  let X = A.isSkill ? `<command-name>${A.userFacingName()}</command-name>`
                    : `<command-name>/${A.userFacingName()}</command-name>`;

  let F = [`<command-message>${I}</command-message>`, X,
           B ? `<command-args>${B}</command-args>` : null]
          .filter(Boolean).join(" ");

  // 4. Process allowed tools
  let K = x51(A.allowedTools ?? []);

  // 5. Combine messages
  let D = G.length>0 || Z.length>0 ? [...G, ...Z, ...J] : J;

  // 6. Calculate thinking tokens
  let H = m_([mA({content:D})], void 0, W);

  // 7. Get attachments
  let z = await Qf1(j51(J.filter((q)=>q.type===="text").map((q)=>q.text).join(" "),
                        Q, null, [], Q.messages, "repl_main_thread"));

  // 8. Construct final messages
  let C = [
    mA({content:F, autocheckpoint:Y}),
    mA({content:D, isMeta:!0}),
    ...z,
    ...(K.length || A.model ?
        [x3({type:"command_permissions", allowedTools:K,
             model:A.useSmallFastModel ? sF() : A.model})] : [])
  ];

  return {messages:C, shouldQuery:!0, allowedTools:K,
          maxThinkingTokens:H>0 ? H : void 0,
          model:A.useSmallFastModel ? sF() : A.model, command:A};
}
```

---

## Source Code References

### Key Line Numbers

| Component | Line(s) | Description |
|-----------|---------|-------------|
| Skill Constant | 2699 | `var sw="Skill"` |
| Skill Tool Definition | 2750+ | Complete tool object with all methods |
| SKILL.md Loading | 2806+ | `jwQ` function - loads skills from directories |
| Frontmatter Parsing | 2806+ | Uses `UD` function to parse YAML frontmatter |
| Skill Validation | 2750+ | `validateInput` method in tool object |
| Permission Checking | 2750+ | `checkPermissions` method |
| Skill Execution | 2750+ | `call` method (async generator) |
| Message Processing | ~2690+ | `H$Q` function |
| Skill vs Command Logic | ~2690+ | `isSkill` flag usage in message formatting |
| Plugin Loading | 2806+ | `En0` cached function for plugin skills |

### Function Call Chain

```
Skill Tool Invocation
  └─> call() [Line 2750+]
      └─> wo1(skillName, args, commands, context)
          └─> H$Q(skill, args, context, ...)
              ├─> skill.getPromptForCommand()
              ├─> Process allowed-tools
              ├─> Calculate maxThinkingTokens
              └─> Return messages array
```

---

## Best Practices

### Creating Skills

1. **Use Descriptive Names**: Choose clear, action-oriented names
2. **Provide Good Descriptions**: Include both `description` and `when-to-use`
3. **Specify Allowed Tools**: List only the tools your skill needs
4. **Set Model Preferences**: Use `model: inherit` unless you have specific requirements
5. **Include Examples**: Add usage examples in the skill content
6. **Document Base Directory**: Reference `${baseDir}` for file operations

### SKILL.md Template

```markdown
---
name: my-skill
description: Brief one-line description
when-to-use: Specific scenarios when this should be invoked
allowed-tools: Read,Write,Bash,Grep
model: inherit
version: 1.0.0
---

# {Skill Name}

## Purpose
Detailed explanation of what this skill does.

## Usage
How to use this skill effectively.

## Instructions
Step-by-step instructions for the AI.

## Base Directory
Base directory for this skill: ${baseDir}

## Examples
Example scenarios and expected outputs.
```

### Skill Organization

1. **Directory Structure**:
   ```
   skills/
   ├── data-analysis/
   │   └── SKILL.md
   ├── code-review/
   │   └── SKILL.md
   └── testing/
       └── SKILL.md
   ```

2. **Plugin Integration**:
   - Place skills in `skillsPath` or `skillsPaths[]`
   - Use namespaced names: `{plugin}:{skill}`

3. **Version Control**:
   - Track SKILL.md files in git
   - Use semantic versioning
   - Document changes in skill content

### Performance Considerations

1. **Token Budget**: Skills are subject to a character budget (`SLASH_COMMAND_TOOL_CHAR_BUDGET` = 15000)
2. **Concurrent Execution**: Skills are NOT concurrency-safe (`isConcurrencySafe: false`)
3. **Caching**: Plugin skills are cached (see `En0` cached function)
4. **Lazy Loading**: Skills are loaded on-demand

---

## Advanced Topics

### Mode Commands vs Regular Skills

Some skills can be marked as "mode commands" with `isModeCommand: true`. These:
- Are shown in a prioritized section
- Represent structured workflows
- Get special UI treatment during permission requests

**Code Reference**: Line 2750+ - checks `Y.command.isModeCommand === !0`

### Disable Model Invocation

Skills with `disable-model-invocation: true` cannot be invoked via the Skill tool but can still be used directly as slash commands.

**Validation**: Line 2750+ - Error code 4

### Context Modifiers

Skills can modify the execution context by:
1. Adding tools to `alwaysAllowRules`
2. Overriding the model
3. Setting `maxThinkingTokens`

**Implementation**: Lines 2750+ in the `call` method's `contextModifier` function

---

## Troubleshooting

### Common Issues

1. **"Unknown skill" Error**
   - Verify SKILL.md exists in the correct directory
   - Check plugin is loaded and enabled
   - Verify skill name matches directory name

2. **"disable-model-invocation" Error**
   - Skill has `disable-model-invocation: true`
   - Use as slash command instead: `/skillname`

3. **Permission Denied**
   - Check permission rules in settings
   - Look for deny rules matching skill name or pattern

4. **Skill Not Appearing**
   - Verify plugin configuration
   - Check `skillsPath` or `skillsPaths[]` settings
   - Look for loading errors in logs

---

## Conclusion

The Claude Code Skills system provides a powerful way to package and reuse specialized AI workflows. By understanding the SKILL.md format, loading mechanisms, and execution flow, you can create effective skills that enhance Claude Code's capabilities.

**Key Takeaways**:
- Skills are defined in `SKILL.md` files with frontmatter metadata
- The Skill tool (`sw = "Skill"`) handles validation, permissions, and execution
- Skills can be invoked via the tool or as slash commands
- Skills support tool restrictions, model preferences, and permission control
- The `isSkill` flag differentiates skills from regular commands in the codebase
