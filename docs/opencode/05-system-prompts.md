# OpenCode - System Prompts & Agent Guidelines

> **Guide to customizing OpenCode's behavior through AGENTS.md files and system prompts**

---

## Table of Contents

- [Overview](#overview)
- [AGENTS.md Files](#agentsmd-files)
- [System Prompt Hierarchy](#system-prompt-hierarchy)
- [Creating AGENTS.md](#creating-agentsmd)
- [Custom Instructions](#custom-instructions)
- [Best Practices](#best-practices)

---

## Overview

OpenCode allows deep customization of AI behavior through **system prompts** and **AGENTS.md** files. These mechanisms let you:

- Define coding standards and style guidelines
- Specify build/test commands
- Set architectural patterns
- Configure tool usage preferences
- Customize per-project or globally

---

## AGENTS.md Files

### What is AGENTS.md?

`AGENTS.md` is a special markdown file that contains instructions for AI agents working in your codebase. OpenCode automatically discovers and includes these files in every AI interaction.

### Discovery Order

OpenCode searches for agent guidelines in this order:

**1. Local Files** (searched upward from working directory):
- `AGENTS.md` - Generic agent instructions
- `CLAUDE.md` - Claude-specific instructions
- `CONTEXT.md` - Context and background

**2. Global Files** (user's home directory):
- `~/.opencode/AGENTS.md` - Global OpenCode instructions
- `~/.claude/CLAUDE.md` - Global Claude instructions

**3. Config Instructions** (from `.opencode/config.json`):
```json
{
  "instructions": [
    "docs/coding-standards.md",
    "~/my-global-rules.md",
    ".github/*.md"
  ]
}
```

### File Hierarchy

```
Project Root
├── AGENTS.md                    # ← Most specific (found first)
│
├── packages/
│   ├── opencode/
│   │   └── AGENTS.md            # ← Package-level rules
│   │
│   └── desktop/
│       └── AGENTS.md            # ← Component-specific rules
│
└── ...

~/.opencode/AGENTS.md            # ← Global (fallback)
~/.claude/CLAUDE.md              # ← Alternative global
```

**Priority**: Local files found first stop the search for that file type. This allows you to override global rules per-project.

---

## System Prompt Hierarchy

### Complete Prompt Stack

When OpenCode sends a prompt to an AI model, it assembles system prompts in this order:

```
1. Provider Header (optional spoofing)
   ↓
2. Provider-Specific Prompt (Anthropic/OpenAI/Gemini/etc.)
   ↓
3. Environment Information (directory, platform, date)
   ↓
4. Custom Instructions (AGENTS.md files)
   ↓
5. Agent-Specific Prompt (if using custom agent)
   ↓
6. User Override (if provided via --system flag)
```

### 1. Provider Header

**Purpose**: Some providers benefit from identity "spoofing" to improve behavior.

**Example** (Anthropic):
```
You are Claude, a large language model trained by Anthropic.
Knowledge cutoff: 2024-04
Current date: 2025-01-15
```

### 2. Provider-Specific Prompts

Different models need different prompting styles:

**Anthropic (Claude)**:
```markdown
You are an expert coding assistant.
- Use precise, technical language
- Provide complete, working code
- Follow best practices
- Explain complex concepts clearly
```

**OpenAI (GPT)**:
```markdown
You are an advanced AI coding assistant.
- Write production-quality code
- Consider edge cases
- Optimize for readability
- Document your approach
```

**Google (Gemini)**:
```markdown
You are a helpful coding AI.
- Focus on practical solutions
- Explain your reasoning
- Consider performance
- Write clean code
```

### 3. Environment Information

Automatically injected context:

```markdown
Here is some useful information about the environment you are running in:

<env>
 Working directory: /Users/me/projects/my-app
 Is directory a git repo: yes
 Platform: darwin
 Today's date: Saturday, January 15, 2025
</env>

<project>
 src/
   auth/
     login.ts
     middleware.ts
   user/
     service.ts
     model.ts
 tests/
   ...
</project>
```

### 4. Custom Instructions (AGENTS.md)

Your project-specific guidelines from discovered `AGENTS.md` files.

### 5. Agent Prompts

If using a custom agent (`.opencode/agent/build.md`):

```markdown
# Build Agent

You are a senior software engineer focused on building robust code.

## Guidelines
- Write tests for all new functions
- Use TypeScript strict mode
- Add JSDoc comments
- Handle errors explicitly
```

### 6. User Override

Command-line override:

```bash
opencode --system "Focus on performance optimization" "Optimize this function"
```

---

## Creating AGENTS.md

### Structure

A good `AGENTS.md` file typically includes:

1. **Build/Test Commands** - How to run the project
2. **Code Style** - Formatting, naming, patterns
3. **Architecture** - Key patterns and structures
4. **Dependencies** - Important libraries and their usage
5. **Special Rules** - Project-specific constraints

### Root Level Example

```markdown
# OpenCode Development Guidelines

## Build/Test Commands

- **Install**: `bun install`
- **Run**: `bun dev`
- **Typecheck**: `bun run typecheck`
- **Test**: `bun test`
- **Single test**: `bun test test/specific.test.ts`

## Code Style

- **Runtime**: Bun with TypeScript ESM modules
- **Imports**: Use relative imports for local modules
- **Types**: Zod schemas for validation
- **Naming**: camelCase for variables, PascalCase for classes
- **Error handling**: Use Result patterns, avoid throwing exceptions
- **File structure**: Namespace-based organization

## Important Guidelines

- Try to keep things in one function unless composable or reusable
- DO NOT do unnecessary destructuring of variables
- DO NOT use `else` statements unless necessary
- AVOID `try`/`catch` where possible
- AVOID using `any` type
- AVOID `let` statements
- PREFER single word variable names where possible
- Use as many bun apis as possible like Bun.file()

## Architecture

- **Tools**: Implement `Tool.Info` interface with `execute()` method
- **Context**: Pass `sessionID` in tool context
- **Validation**: All inputs validated with Zod schemas
- **Logging**: Use `Log.create({ service: "name" })` pattern
- **Storage**: Use `Storage` namespace for persistence

## Tool Calling

ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE. Example for 3 parallel file reads:

\```json
{
    "recipient_name": "multi_tool_use.parallel",
    "parameters": {
        "tool_uses": [
            {
                "recipient_name": "functions.read",
                "parameters": { "filePath": "path/to/file1.ts" }
            },
            {
                "recipient_name": "functions.read",
                "parameters": { "filePath": "path/to/file2.ts" }
            }
        ]
    }
}
\```
```

### Package-Level Example

```markdown
# Package: @opencode/desktop

## Build/Test Commands

- **Development**: `bun run dev` (starts Vite dev server on port 3000)
- **Build**: `bun run build` (production build)
- **Preview**: `bun run serve` (preview production build)
- **Validation**: Use `bun run typecheck` only
- **Testing**: Do not create or run automated tests

## Code Style

- **Framework**: SolidJS with TypeScript
- **Imports**: Use `@/` alias for src/ directory (e.g., `import Button from "@/ui/button"`)
- **Formatting**: Prettier configured with semicolons disabled, 120 character line width
- **Components**: Use function declarations, splitProps for component props
- **Types**: Define interfaces for component props, avoid `any` type
- **CSS**: TailwindCSS with custom CSS variables theme system
- **Naming**: PascalCase for components, camelCase for variables/functions, snake_case for file names
- **File Structure**: 
  - UI primitives in `/ui/`
  - Higher-level components in `/components/`
  - Pages in `/pages/`
  - Providers in `/providers/`

## Key Dependencies

- SolidJS, @solidjs/router, @kobalte/core (UI primitives)
- TailwindCSS 4.x with @tailwindcss/vite
- Custom theme system with CSS variables
```

### Minimal Example

For simple projects:

```markdown
# Project Guidelines

## Commands

- Run: `npm start`
- Test: `npm test`
- Build: `npm run build`

## Style

- TypeScript strict mode
- Prettier formatting
- ESLint rules enforced
- Tests required for new features
```

---

## Custom Instructions

### Via Configuration

Add glob patterns in `.opencode/config.json`:

```json
{
  "instructions": [
    "docs/coding-standards.md",
    "docs/architecture.md",
    "~/my-coding-style.md",
    ".github/**/*.md"
  ]
}
```

**Features**:
- Supports glob patterns (`**/*.md`)
- Supports `~/` home directory expansion
- Supports absolute paths
- Multiple files combined in order

### Via CLI

Override system prompt for single session:

```bash
# Add custom instruction
opencode --system "Focus on security" "Review this code"

# Longer instructions
opencode --system "$(cat security-checklist.md)" "Audit auth.ts"
```

### Via Agents

Create custom agents with specific instructions:

```bash
# Create agent
mkdir -p .opencode/agent
cat > .opencode/agent/security.md << 'EOF'
# Security Agent

Focus on security in all code reviews and implementations.

## Guidelines
- Check for SQL injection vulnerabilities
- Validate all user inputs
- Use parameterized queries
- Implement proper authentication
- Check CORS policies
- Review error messages (no sensitive data)
EOF

# Use agent
opencode --agent security "Review API endpoints"
```

---

## Best Practices

### 1. Keep It Concise

**Good** (20-30 lines):
```markdown
# Guidelines

## Commands
- Dev: `npm start`
- Test: `npm test`

## Style
- TypeScript strict
- Tests required
- Prettier formatting
```

**Bad** (too verbose):
```markdown
# Extremely Detailed Guidelines

## Introduction
This document describes in excruciating detail...

## History
This project was started in 2020...

## Philosophy
We believe that code should be...
[500 more lines]
```

### 2. Focus on Actionable Rules

**Good**:
```markdown
- ALWAYS write tests for new functions
- NEVER use `any` type
- USE Zod for validation
```

**Bad**:
```markdown
- It would be nice if we could maybe consider writing tests
- Try to avoid any type when possible
- Validation is important
```

### 3. Include Examples

**Good**:
```markdown
## Error Handling

Use Result pattern:

\```typescript
function parse(input: string): Result<Data, Error> {
  if (!valid(input)) {
    return Err(new ValidationError())
  }
  return Ok(parsed)
}
\```
```

**Bad**:
```markdown
## Error Handling

Handle errors properly.
```

### 4. Project-Specific Overrides

**Root** (`AGENTS.md`):
```markdown
# General project rules
- TypeScript strict mode
- Tests required
```

**Package** (`packages/legacy/AGENTS.md`):
```markdown
# Legacy package (overrides root rules)
- JavaScript (no TypeScript)
- Tests optional (old code)
- Maintain compatibility
```

### 5. Update Regularly

```markdown
# Guidelines (Last updated: 2025-01-15)

## Recent Changes
- 2025-01-15: Added Zod validation requirement
- 2025-01-10: Switched to Bun runtime
- 2025-01-05: Adopted SolidJS for UI
```

### 6. Include Command Examples

```markdown
## Common Tasks

\```bash
# Run specific test
bun test test/auth.test.ts

# Type check only
bun run typecheck

# Debug mode
DEBUG=* bun dev

# Build production
bun run build --mode production
\```
```

### 7. Specify Dependencies

```markdown
## Key Dependencies

- **Bun**: Runtime and package manager
- **Zod**: Schema validation (use for all inputs)
- **Hono**: Web framework (use for all APIs)
- **SolidJS**: UI framework (use for desktop app)

## DO NOT Use

- ❌ Express (use Hono instead)
- ❌ React (use SolidJS instead)
- ❌ class-validator (use Zod instead)
```

---

## Advanced Customization

### Multiple AGENTS.md Files

OpenCode supports discovering multiple types:

```
project/
├── AGENTS.md          # ← Generic instructions
├── CLAUDE.md          # ← Claude-specific
├── CONTEXT.md         # ← Background context
└── .opencode/
    └── config.json    # ← Additional via config
```

All discovered files are combined.

### Conditional Instructions

Use environment-aware instructions:

```markdown
# Development Guidelines

## Platform-Specific

### macOS
- Use `brew install` for dependencies

### Linux
- Use `apt install` or `pacman -S`

### Windows
- Use `scoop install` or `choco install`
```

### Integration with Existing Tools

Reference existing documentation:

```markdown
# Guidelines

See also:
- [Contributing Guide](CONTRIBUTING.md)
- [Architecture Docs](docs/architecture.md)
- [API Reference](docs/api.md)

Follow all guidelines in those documents.
```

---

## Troubleshooting

### AGENTS.md Not Being Used

**Check discovery**:
```bash
# List all found instructions
opencode debug config | jq '.instructions'
```

**Common issues**:
- File not in search path (must be in or above working directory)
- File has wrong name (must be exactly `AGENTS.md`, not `agents.md`)
- File not readable (check permissions)

### Instructions Conflicting

**Resolution order**:
1. Local files take precedence over global
2. Later instructions don't override earlier ones (they combine)
3. More specific rules should be in local files

**Solution**: Use clear, non-conflicting rules or override locally.

### Too Much Context

If your `AGENTS.md` is very long:

1. **Split into multiple files**:
   ```json
   {
     "instructions": [
       "docs/style.md",
       "docs/architecture.md",
       "docs/testing.md"
     ]
   }
   ```

2. **Keep only essential rules** in `AGENTS.md`

3. **Reference external docs** instead of including full text

---

## Summary

OpenCode's system prompt customization provides:

- **Automatic discovery** of AGENTS.md files
- **Hierarchical overrides** (local > global)
- **Provider-specific** prompting
- **Environment context** injection
- **Agent personas** for specialized tasks
- **CLI overrides** for one-off customization

Well-crafted `AGENTS.md` files ensure AI assistants understand your project's conventions, tools, and requirements, leading to higher-quality suggestions and faster development.

---

## Next Steps

- **[06-tool-system.md](./06-tool-system.md)** - Tool architecture
- **[13-configuration.md](./13-configuration.md)** - Configuration system
- **[24-development-guide.md](./24-development-guide.md)** - Development guide

