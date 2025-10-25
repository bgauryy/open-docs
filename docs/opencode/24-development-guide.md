# OpenCode - Development Guide

> **Complete guide for contributing to OpenCode**

---

## Overview

This guide covers:
- Setting up development environment
- Building from source
- Testing and debugging
- Code style and conventions
- Contribution workflow

---

## Prerequisites

### Required

- **Bun** 1.3.0+ - Runtime and package manager
- **Go** 1.24.0+ - For TUI
- **Git** - Version control
- **Node.js** 22+ - Alternative runtime (some packages require >=22)

### Recommended

- **VS Code** or **Zed** - IDE
- **GitHub CLI** (`gh`) - For Copilot auth
- **Ripgrep** (`rg`) - Fast search
- **FZF** - Fuzzy finder

---

## Setup

### Clone Repository

```bash
git clone https://github.com/sst/opencode.git
cd opencode
```

### Install Dependencies

```bash
bun install
```

This installs dependencies for all workspace packages.

### Build

```bash
# Build all packages
bun run build

# Build specific package
cd packages/opencode
bun run build
```

---

## Development Workflow

### Running OpenCode

**From source**:
```bash
cd packages/opencode
bun dev
```

**With debugging**:
```bash
DEBUG=* bun dev
```

**Specific modules**:
```bash
DEBUG=opencode:session,opencode:tool bun dev
```

### Testing

```bash
# Run all tests
bun test

# Run specific test file
bun test test/session.test.ts

# Watch mode
bun test --watch

# Coverage
bun test --coverage
```

### Type Checking

```bash
# Check types
bun run typecheck

# Watch mode
bun run typecheck --watch
```

### Linting

```bash
# Run linter
bun run lint

# Fix automatically
bun run lint --fix
```

---

## Code Style

From **AGENTS.md**:

### General Principles

```typescript
// ✅ Good: Single function, clear purpose
function processUser(user: User) {
  if (!user.active) return null
  return formatUser(user)
}

// ❌ Bad: Unnecessary destructuring
function processUser(user: User) {
  const { id, name, email, active } = user
  // ... only uses active
}
```

### Avoid Patterns

**No `else` statements** (when possible):
```typescript
// ✅ Good: Early return
function getStatus(user: User) {
  if (!user.active) return "inactive"
  if (user.banned) return "banned"
  return "active"
}

// ❌ Bad: Nested else
function getStatus(user: User) {
  if (user.active) {
    return "active"
  } else {
    if (user.banned) {
      return "banned"
    } else {
      return "inactive"
    }
  }
}
```

**Avoid `try/catch`** (when possible):
```typescript
// ✅ Good: Result type
function parseJSON(text: string): Result<Data, Error> {
  const data = JSON.parse(text)
  if (!validate(data)) {
    return Err(new ValidationError())
  }
  return Ok(data)
}

// ❌ Bad: Unnecessary try/catch
function parseJSON(text: string) {
  try {
    return JSON.parse(text)
  } catch (error) {
    return null  // Loses error information
  }
}
```

**No `any` type**:
```typescript
// ✅ Good: Proper types
function process<T>(data: T): Result<T, Error>

// ❌ Bad: Any type
function process(data: any): any
```

### Naming

```typescript
// ✅ Single-word variables (when clear)
const user = await getUser()
const msg = await getMessage()
const result = await process()

// Use longer names when needed for clarity
const userFromDatabase = await db.users.find()
const messageWithParts = await buildMessage()
```

### Bun APIs

```typescript
// ✅ Prefer Bun APIs
const content = await Bun.file("path.ts").text()
await Bun.write("path.ts", content)
const exists = await Bun.file("path.ts").exists()

// ❌ Avoid Node.js fs when Bun works
import fs from "fs/promises"
const content = await fs.readFile("path.ts", "utf8")
```

---

## Architecture Patterns

### Namespace Organization

```typescript
// Group related functionality in namespaces
export namespace Session {
  export interface Info {...}
  
  export async function create(params: CreateParams): Promise<Info> {...}
  export async function get(id: string): Promise<Info> {...}
  export async function remove(id: string): Promise<void> {...}
}

// Usage
const session = await Session.create({...})
```

### State Management

```typescript
// Per-instance state
export const state = Instance.state(() => {
  return {
    sessions: new Map<string, Session>(),
    cache: new Map<string, any>()
  }
})

// Usage
const s = await state()
s.sessions.set(id, session)
```

### Result Types

```typescript
type Result<T, E> = 
  | { ok: true, value: T }
  | { ok: false, error: E }

function process(): Result<Data, Error> {
  if (!valid) {
    return { ok: false, error: new Error("Invalid") }
  }
  return { ok: true, value: data }
}
```

---

## Testing Patterns

### Unit Tests

```typescript
import { describe, it, expect } from "bun:test"

describe("Session", () => {
  it("creates session with default values", async () => {
    const session = await Session.create({})
    
    expect(session.id).toStartWith("session_")
    expect(session.provider).toBe("anthropic")
  })
  
  it("rejects invalid parameters", async () => {
    await expect(
      Session.create({ provider: "invalid" })
    ).rejects.toThrow()
  })
})
```

### Integration Tests

```typescript
describe("Session integration", () => {
  let sessionID: string
  
  beforeEach(async () => {
    sessionID = await Session.create({})
  })
  
  afterEach(async () => {
    await Session.remove(sessionID)
  })
  
  it("processes messages", async () => {
    const msg = await SessionPrompt.prompt({
      sessionID,
      parts: [{ type: "text", text: "Hello" }]
    })
    
    expect(msg.parts).toHaveLength(2) // User + assistant
  })
})
```

---

## Debugging

### Enable Debug Logging

```bash
# All modules
DEBUG=* bun dev

# Specific modules
DEBUG=opencode:session,opencode:tool bun dev

# Pattern matching
DEBUG=opencode:* bun dev
```

### VS Code Debug Config

**.vscode/launch.json**:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug OpenCode",
      "runtimeExecutable": "bun",
      "runtimeArgs": ["run", "dev"],
      "cwd": "${workspaceFolder}/packages/opencode",
      "env": {
        "DEBUG": "*"
      }
    }
  ]
}
```

---

## Contribution Workflow

### 1. Fork and Branch

```bash
# Fork on GitHub
gh repo fork sst/opencode --clone

# Create branch
git checkout -b feature/my-feature
```

### 2. Make Changes

```bash
# Edit code
vim packages/opencode/src/...

# Test
bun test

# Type check
bun run typecheck
```

### 3. Commit

```bash
git add .
git commit -m "Add feature: description"
```

**Commit message format**:
```
<type>: <description>

[optional body]

[optional footer]
```

**Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

**Examples**:
```
feat: add support for Gemini provider
fix: handle session lock timeout correctly
docs: update installation instructions
```

### 4. Push and PR

```bash
# Push
git push origin feature/my-feature

# Create PR
gh pr create --title "Add my feature" --body "Description"
```

**PR Guidelines**:
- Clear title and description
- Link related issues
- Include tests
- Update documentation
- Follow code style

---

## Common Tasks

### Adding a New Tool

1. Create `packages/opencode/src/tool/mytool.ts`:
```typescript
export const MyTool = Tool.define("mytool", {
  description: "...",
  parameters: z.object({...}),
  async execute(args, ctx) {...}
})
```

2. Add to registry:
```typescript
// tool/registry.ts
import { MyTool } from "./mytool"

export async function all() {
  return [
    // ...
    MyTool,
  ]
}
```

3. Add tests:
```typescript
// test/tool/mytool.test.ts
describe("MyTool", () => {...})
```

### Adding a Provider

1. Create provider module
2. Implement Provider.Info interface
3. Register in provider registry
4. Add authentication support
5. Update documentation

### Updating Server API

1. Edit `server/server.ts`
2. Generate TypeScript SDK:
```bash
bun run generate-sdk
```
3. Generate Go SDK (for TUI):
```bash
cd packages/tui
go generate
```

---

## Release Process

1. Update version in `package.json`
2. Update CHANGELOG.md
3. Create Git tag
4. Push tag
5. GitHub Actions builds and releases

---

## Resources

- **Codebase**: https://github.com/sst/opencode
- **Discussions**: GitHub Discussions
- **Issues**: GitHub Issues
- **Discord**: https://opencode.ai/discord

---

## Getting Help

**Questions**:
- Discord #development channel
- GitHub Discussions

**Bugs**:
- GitHub Issues with reproducible example

**Security**:
- Email security@opencode.ai
- Do not open public issues for security vulnerabilities

---

**Thank you for contributing to OpenCode!** 🎉

