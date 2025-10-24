# Claude Agent SDK - Skills System Complete Reference

**SDK Version**: 0.1.22
**Source**: SDK documentation and implementation patterns

---

## Table of Contents

1. [Overview](#overview)
2. [Skill File Format](#skill-file-format)
3. [Skill Discovery](#skill-discovery)
4. [Skill Execution](#skill-execution)
5. [Frontmatter Schema](#frontmatter-schema)
6. [Template Variables](#template-variables)
7. [Skills vs Slash Commands](#skills-vs-slash-commands)
8. [Built-in Skills](#built-in-skills)
9. [Custom Skill Creation](#custom-skill-creation)
10. [Real-World Examples](#real-world-examples)
11. [Gotchas & Best Practices](#gotchas--best-practices)

---

## Overview

The Skills System allows you to create reusable, parameterized prompts that can be invoked as commands. Skills are:

- **Markdown-based**: Stored as `SKILL.md` files
- **Discoverable**: Auto-loaded from specific directories
- **Parameterized**: Accept arguments via `$ARGUMENTS`
- **Agent-aware**: Can specify which agent to use
- **Tool-restricted**: Can limit available tools

### Key Concepts

```
Skill File (SKILL.md)
     ↓
Discovery (scanned from directories)
     ↓
Registration (added to available skills)
     ↓
Invocation (/skill-name or Skill tool)
     ↓
Execution (agent runs with skill prompt)
```

**Benefits**:
- Reusable prompts for common tasks
- Consistent workflows across team
- Reduced typing (parameterized templates)
- Agent specialization per skill
- Tool restriction for safety

---

## Skill File Format

### Basic Structure

```markdown
---
name: skill-name
description: Short description of what this skill does
agent: Explore
allowed-tools: Glob, Grep, Read
when_to_use: When you need to find something in the codebase
---

# Skill Prompt Content

This is the actual prompt that will be sent to the agent.

## Instructions

1. Do this
2. Then do that
3. Return results in this format

## Example Usage

Find {{$ARGUMENTS}} in the codebase.
```

**Components**:
1. **Frontmatter** (YAML between `---`): Metadata and configuration
2. **Prompt Body** (Markdown): Instructions for the agent
3. **Template Variables**: `{{$ARGUMENTS}}` for dynamic content

---

## Skill Discovery

### Discovery Paths

Skills are discovered from multiple locations (in order of precedence):

```
1. Session directory (temporary)
   - Not typically used for skills

2. Local directory (cwd/.claude/)
   - Project-specific, gitignored
   - For personal/experimental skills

3. Project directory (git root/.claude/)
   - Shared team skills
   - Committed to repository

4. User directory (~/.claude/)
   - Personal skills across all projects
   - User-specific workflows

5. Policy directory (enterprise)
   - Organization-wide skills
   - Enforced standards

6. Plugin directories
   - Plugin-provided skills
   - Third-party skills
```

### Discovery Algorithm

```typescript
// Simplified discovery logic
function discoverSkills(): Skill[] {
  const skills: Skill[] = [];
  
  // Scan each directory
  for (const dir of SKILL_DIRECTORIES) {
    // Find all SKILL.md files
    const skillFiles = findFiles(dir, "**/SKILL.md");
    
    for (const file of skillFiles) {
      // Parse frontmatter
      const skill = parseSkillFile(file);
      
      // Validate
      if (validateSkill(skill)) {
        // Register (first found wins for duplicates)
        if (!skills.find(s => s.name === skill.name)) {
          skills.push(skill);
        }
      }
    }
  }
  
  return skills;
}
```

### Discovery Rules

1. **File Name**: Must be exactly `SKILL.md` (case-sensitive)
2. **Directory Structure**: Can be nested (e.g., `skills/search/SKILL.md`)
3. **Name Conflicts**: First discovered wins (by precedence order)
4. **Hot Reload**: Changes to skill files typically require restart

---

## Skill Execution

### Invocation Methods

#### 1. Via Skill Tool

```typescript
Skill({
  skill_name: "quick-search",
  arguments: "authentication functions"
})
```

#### 2. Via Slash Command

```bash
/quick-search authentication functions
```

#### 3. Via Natural Language (if enabled)

```
> Use the quick-search skill to find authentication functions
```

### Execution Flow

```
1. Skill Invocation
     ↓
2. Skill Lookup (by name)
     ↓
3. Template Variable Substitution ($ARGUMENTS)
     ↓
4. Agent Selection (from frontmatter or default)
     ↓
5. Tool Restriction (from allowed-tools)
     ↓
6. Agent Execution (Task tool with processed prompt)
     ↓
7. Result Return (agent output)
```

### Execution Example

**Skill File**: `~/.claude/skills/search/SKILL.md`
```markdown
---
name: quick-search
agent: Explore
allowed-tools: Glob, Grep, Read
---

Find all files containing {{$ARGUMENTS}} and show me the relevant code sections.

Return in this format:
- File: path/to/file.ts
  Lines: 45-67
  Match: [relevant code snippet]
```

**Invocation**:
```bash
/quick-search "authentication"
```

**Actual Execution** (behind the scenes):
```typescript
Task({
  agent_type: "Explore",
  prompt: `Find all files containing "authentication" and show me the relevant code sections.

Return in this format:
- File: path/to/file.ts
  Lines: 45-67
  Match: [relevant code snippet]`,
  expected_output: "Formatted file list with code snippets",
  allowed_tools: ["Glob", "Grep", "Read"]
})
```

---

## Frontmatter Schema

### Complete Schema

```yaml
---
# Required Fields
name: skill-name                    # Unique identifier (used in commands)
description: What this skill does   # Short description (for discovery)

# Optional Fields
agent: Explore                      # Agent to use (default: general-purpose)
allowed-tools: Glob, Grep, Read     # Comma-separated tool list
when_to_use: When to use this skill # Guidance for AI/users
version: 1.0.0                      # Skill version (semantic versioning)
argument-hint: <search pattern>     # Hint for argument format
disable-model-invocation: false     # If true, just returns prompt text
---
```

### Field Descriptions

**name** (required):
- Unique identifier for the skill
- Used in slash commands: `/name`
- Convention: lowercase with hyphens
- Examples: `quick-search`, `review-pr`, `generate-tests`

**description** (required):
- Short, human-readable description
- Shown in skill list
- Used for skill discovery
- Max ~80 characters recommended

**agent** (optional):
- Agent to execute skill with
- Options: `Explore`, `general-purpose`, `security-review`, custom agents
- Default: `general-purpose` (if not specified)
- Affects: model, tools, context

**allowed-tools** (optional):
- Comma-separated list of allowed tools
- Overrides agent's default tools
- Examples: `Glob, Grep, Read` or `*` (all tools)
- Empty = inherit from agent

**when_to_use** (optional):
- Guidance for when to use this skill
- Helps AI select appropriate skill
- Also useful for documentation
- Example: "When you need to search large codebases quickly"

**version** (optional):
- Semantic version number
- Used for skill updates and compatibility
- Format: `major.minor.patch`
- Example: `1.2.3`

**argument-hint** (optional):
- Hint for expected argument format
- Shown in usage instructions
- Examples: `<file pattern>`, `<branch name>`, `<search query>`
- Displayed as: `/skill-name <argument-hint>`

**disable-model-invocation** (optional):
- If `true`: Returns prompt text without executing
- If `false` (default): Executes skill normally
- Use case: Template generation, debugging
- Rarely used

---

## Template Variables

### Available Variables

Currently, the primary template variable is:

**$ARGUMENTS**:
- Replaced with arguments passed to skill
- Syntax: `{{$ARGUMENTS}}` or `$ARGUMENTS`
- Trimmed whitespace
- Can appear multiple times

**Example**:
```markdown
---
name: find-and-analyze
---

Step 1: Find files matching {{$ARGUMENTS}}
Step 2: Analyze the contents of {{$ARGUMENTS}}
Step 3: Summarize findings for {{$ARGUMENTS}}
```

**Invocation**:
```bash
/find-and-analyze "*.test.ts"
```

**Result**:
```markdown
Step 1: Find files matching "*.test.ts"
Step 2: Analyze the contents of "*.test.ts"
Step 3: Summarize findings for "*.test.ts"
```

### Future Variables (Potential)

While not currently implemented, potential future variables:

```markdown
{{$USER}}          - Current user
{{$CWD}}           - Current working directory
{{$DATE}}          - Current date
{{$SESSION_ID}}    - Current session ID
{{$GIT_BRANCH}}    - Current git branch
{{$PROJECT_NAME}}  - Project name
```

**Note**: These are speculative and not part of current SDK.

---

## Skills vs Slash Commands

### Comparison

| Feature | Skills | Slash Commands |
|---------|--------|----------------|
| **Definition** | Markdown files (SKILL.md) | Code-defined in SDK |
| **Customization** | User/project/plugin | Built-in only |
| **Arguments** | Template-based | Structured parameters |
| **Discovery** | File system scan | Hardcoded |
| **Distribution** | Files (git, plugins) | SDK update required |
| **Agent** | Configurable | Fixed per command |
| **Tools** | Configurable | Fixed per command |

### When to Use Skills

✅ **Use Skills When**:
- Reusable prompts across projects
- Team-shared workflows
- Parameterized templates
- Agent specialization needed
- Tool restriction desired
- Distribution via files/git

### When to Use Slash Commands

✅ **Use Slash Commands When**:
- Built-in functionality sufficient
- Standard operations (e.g., `/compact`, `/help`)
- Complex logic required (not just prompt)
- Tight SDK integration needed

---

## Built-in Skills

The SDK may include some built-in skills. Common examples:

### Example: Explore Skill

```markdown
---
name: explore
description: Quick codebase exploration
agent: Explore
allowed-tools: Glob, Grep, Read, Bash
when_to_use: When you need to quickly find files or code patterns
argument-hint: <search query or pattern>
---

Explore the codebase for: {{$ARGUMENTS}}

Use these steps:
1. Use Glob to find relevant files
2. Use Grep to search within files
3. Use Read to examine promising matches
4. Summarize findings with file paths and line numbers

Be thorough but concise.
```

**Usage**:
```bash
/explore "API endpoints"
/explore "authentication logic"
/explore "*.test.ts files with failing tests"
```

---

## Custom Skill Creation

### Step-by-Step Guide

**Step 1: Choose Location**

```bash
# User-level (personal skills)
mkdir -p ~/.claude/skills/

# Project-level (team skills)
mkdir -p ./.claude/skills/
```

**Step 2: Create SKILL.md**

```bash
cd ~/.claude/skills/
mkdir my-skill
cd my-skill
touch SKILL.md
```

**Step 3: Write Skill**

```markdown
---
name: my-skill
description: Does something useful
agent: Explore
allowed-tools: Glob, Grep, Read
when_to_use: When you need X
version: 1.0.0
argument-hint: <what to look for>
---

# My Skill Instructions

You are tasked with: {{$ARGUMENTS}}

## Steps

1. First, do A
2. Then, do B
3. Finally, do C

## Output Format

Return results as:
- Result 1
- Result 2
- Result 3
```

**Step 4: Test Skill**

```bash
# Restart claude-code to load skills
claude-code

# Test invocation
> /my-skill test arguments

# Verify output
```

---

## Real-World Examples

### Example 1: Code Review Skill

**File**: `.claude/skills/review-changes/SKILL.md`

```markdown
---
name: review-changes
description: Review code changes in current branch
agent: security-review
allowed-tools: Bash(git*), Read, Grep
when_to_use: Before creating PR or committing
version: 1.0.0
argument-hint: <base branch, default: main>
---

# Code Review

Review all changes in current branch compared to {{$ARGUMENTS}}.

## Review Checklist

1. **Security**:
   - Check for exposed secrets
   - Validate input sanitization
   - Check for SQL injection risks

2. **Quality**:
   - Check for TODO/FIXME comments
   - Validate error handling
   - Check test coverage

3. **Style**:
   - Consistent formatting
   - Proper naming conventions
   - Adequate comments

## Output Format

### Security Issues
- [None found] or [List issues]

### Quality Issues
- [None found] or [List issues]

### Style Issues
- [None found] or [List issues]

### Summary
Overall assessment: PASS/FAIL
Recommendations: [...]
```

**Usage**:
```bash
/review-changes main
/review-changes develop
```

---

### Example 2: Test Generation Skill

**File**: `~/.claude/skills/generate-tests/SKILL.md`

```markdown
---
name: generate-tests
description: Generate unit tests for specified file
agent: general-purpose
allowed-tools: Read, Write, Glob, Grep
when_to_use: When you need tests for new or untested code
version: 1.0.0
argument-hint: <file path>
---

# Generate Unit Tests

Generate comprehensive unit tests for: {{$ARGUMENTS}}

## Steps

1. Read the source file
2. Identify all functions/methods
3. For each function:
   - Generate happy path test
   - Generate edge case tests
   - Generate error case tests
4. Write tests to corresponding test file

## Test File Location

If source is: `src/auth/login.ts`
Create: `tests/auth/login.test.ts`

## Test Framework

Use Jest/Vitest syntax:
```typescript
describe('functionName', () => {
  it('should do X when Y', () => {
    // Test code
  });
});
```

## Coverage Target

Aim for 80%+ code coverage.
```

**Usage**:
```bash
/generate-tests src/auth/login.ts
/generate-tests src/utils/format.ts
```

---

### Example 3: API Documentation Skill

**File**: `.claude/skills/document-api/SKILL.md`

```markdown
---
name: document-api
description: Generate API documentation for endpoints
agent: general-purpose
allowed-tools: Read, Write, Grep, Glob
when_to_use: When you need to document API endpoints
version: 1.0.0
argument-hint: <route pattern, e.g., /api/users*>
---

# API Documentation Generator

Generate comprehensive API documentation for: {{$ARGUMENTS}}

## Steps

1. Find all route definitions matching pattern
2. For each endpoint, extract:
   - HTTP method (GET, POST, etc.)
   - Path
   - Parameters (path, query, body)
   - Response format
   - Error codes
   - Authentication requirements

## Output Format

### Endpoint: [METHOD] /path/to/endpoint

**Description**: [What this endpoint does]

**Authentication**: [Required/Optional/None]

**Parameters**:
- `param1` (type): Description
- `param2` (type): Description

**Request Body** (if applicable):
```json
{
  "field": "value"
}
```

**Response** (200 OK):
```json
{
  "result": "value"
}
```

**Errors**:
- `400 Bad Request`: [When this occurs]
- `401 Unauthorized`: [When this occurs]
- `404 Not Found`: [When this occurs]

**Example**:
```bash
curl -X METHOD https://api.example.com/path \
  -H "Authorization: Bearer token" \
  -d '{"field": "value"}'
```

---

[Repeat for each endpoint]

## Documentation File

Write to: `docs/api/[section].md`
```

**Usage**:
```bash
/document-api "/api/users*"
/document-api "/api/auth/*"
```

---

## Gotchas & Best Practices

### Gotchas

1. **File Name Must Be Exactly SKILL.md**:
   ```bash
   # ❌ Wrong
   skill.md
   Skill.md
   SKILL.MD
   my-skill.md
   
   # ✅ Correct
   SKILL.md
   ```

2. **Frontmatter Must Be Valid YAML**:
   ```markdown
   ---
   name: my-skill
   description: This is broken because no quotes with colon
   # ❌ Error: YAML parsing fails
   ---
   ```
   
   ```markdown
   ---
   name: my-skill
   description: "This works: quotes protect special chars"
   # ✅ Valid
   ---
   ```

3. **Name Conflicts - First Found Wins**:
   ```
   ~/.claude/skills/search/SKILL.md (name: search)
   ./.claude/skills/search/SKILL.md (name: search)
   
   # User skill wins (higher precedence)
   # Project skill ignored
   ```

4. **Skills Not Hot-Reloaded**:
   - Changing SKILL.md requires restart
   - No automatic reload on file change

5. **$ARGUMENTS is String Only**:
   - No structured arguments (objects, arrays)
   - All arguments passed as single string
   - Parse within prompt if needed

6. **Tool Restrictions Strictly Enforced**:
   ```markdown
   ---
   allowed-tools: Read, Grep
   ---
   
   # Skill tries to use Write
   # ❌ Error: Tool "Write" not allowed
   ```

### Best Practices

**1. Use Descriptive Names**:
```markdown
# ✅ Good
name: review-security-changes
name: generate-api-docs
name: find-unused-imports

# ❌ Bad
name: skill1
name: do-stuff
name: x
```

**2. Provide Clear Instructions**:
```markdown
---
name: my-skill
---

# Clear Structure

## What to Do
[Explain the task]

## How to Do It
1. Step 1
2. Step 2
3. Step 3

## Expected Output
[Format specification]

## Example
[Show example output]
```

**3. Specify Output Format**:
```markdown
Return results in this format:

```json
{
  "findings": [],
  "summary": "",
  "recommendation": ""
}
```

Or use markdown:

### Findings
- Item 1
- Item 2

### Summary
[Text]
```

**4. Use Appropriate Agent**:
```markdown
# ✅ Good
---
name: quick-search
agent: Explore     # Fast, isolated, file operations only
---

---
name: complex-refactor
agent: general-purpose   # Full tools, context-aware
---
```

**5. Restrict Tools for Safety**:
```markdown
# ✅ Good - read-only operations
---
name: analyze-code
allowed-tools: Read, Grep, Glob
---

# ⚠️ Careful - full access
---
name: auto-fix
allowed-tools: "*"    # All tools (risky!)
---
```

**6. Version Your Skills**:
```markdown
---
name: my-skill
version: 1.0.0    # Initial version
---

# Later, when updated:
---
name: my-skill
version: 1.1.0    # Minor update
---
```

**7. Test with Different Arguments**:
```bash
# Test edge cases
/my-skill ""                    # Empty
/my-skill "simple"              # Single word
/my-skill "multiple words"      # Spaces
/my-skill "special * chars ?"   # Special characters
```

**8. Document Argument Format**:
```markdown
---
name: my-skill
argument-hint: "<file-pattern> [--option]"
description: Process files matching pattern
---

Usage: /my-skill "src/**/*.ts"
Usage: /my-skill "*.js" --recursive
```

---

## Summary

### Skill System Overview

| Component | Purpose |
|-----------|---------|
| **SKILL.md** | File format for skill definitions |
| **Frontmatter** | YAML configuration (name, agent, tools) |
| **Prompt Body** | Instructions for agent |
| **$ARGUMENTS** | Template variable for dynamic content |
| **Discovery** | Auto-load from multiple directories |
| **Execution** | Invoked via Skill tool or slash commands |

### Skill Creation Checklist

- [ ] Choose location (user/project/.claude/skills/)
- [ ] Create directory and SKILL.md file
- [ ] Write valid YAML frontmatter
- [ ] Set unique name (lowercase-with-hyphens)
- [ ] Write clear description
- [ ] Choose appropriate agent
- [ ] Restrict tools (if needed)
- [ ] Write clear prompt with steps
- [ ] Use $ARGUMENTS template variable
- [ ] Specify output format
- [ ] Test with various arguments
- [ ] Document usage and examples
- [ ] Version your skill
- [ ] Share with team (if project skill)

### Key Takeaways

- ✅ Skills are reusable, parameterized prompts
- ✅ Stored as `SKILL.md` files with YAML frontmatter
- ✅ Discovered from multiple locations (user, project, plugins)
- ✅ Can specify agent and tool restrictions
- ✅ Invoked via `/skill-name` or Skill tool
- ✅ Template variable: `{{$ARGUMENTS}}` for dynamic content
- ⚠️ File name must be exactly `SKILL.md`
- ⚠️ Frontmatter must be valid YAML
- ⚠️ Changes require restart (not hot-reloaded)
- ⚠️ Name conflicts: first found wins
