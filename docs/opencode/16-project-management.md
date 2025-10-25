# OpenCode - Project Management

> **Multi-project and workspace support**

---

## Overview

OpenCode supports managing multiple projects simultaneously:
- **Isolated sessions** - Separate conversations per project
- **Per-project config** - Independent settings
- **Multiple worktrees** - Git worktree support
- **Project API** - REST-like interface

**Files**:
- `project/instance.ts` - Project instance management
- `project/project.ts` - Project state
- `project/state.ts` - State management
- `specs/project.md` - API specification

---

## Project Concepts

### Instance

A running OpenCode instance tied to a specific directory:
```typescript
{
  directory: "/path/to/project",
  worktree: "/path/to/git/root",
  vcs: "git" | "none"
}
```

### Project

Logical grouping of related work:
```typescript
{
  id: "proj_abc123",
  name: "my-app",
  path: "/path/to/project",
  sessions: ["session_1", "session_2"],
  config: {...}
}
```

---

## Project API

### Create Project

```bash
POST /projects
{
  "name": "my-app",
  "path": "/Users/me/projects/my-app"
}
```

Response:
```json
{
  "id": "proj_abc123",
  "name": "my-app",
  "path": "/Users/me/projects/my-app",
  "created": 1234567890
}
```

### List Projects

```bash
GET /projects
```

Response:
```json
{
  "projects": [
    {
      "id": "proj_abc123",
      "name": "my-app",
      "sessions": 3,
      "lastActive": 1234567890
    }
  ]
}
```

### Get Project

```bash
GET /projects/:id
```

### Delete Project

```bash
DELETE /projects/:id
```

---

## Multi-Worktree Support

### Git Worktrees

OpenCode detects Git worktrees automatically:

```bash
# Main repo
cd /projects/my-app
opencode "Work on main"

# Worktree for feature
cd /projects/my-app-feature
opencode "Work on feature"
```

Both share the same Git root but have isolated OpenCode sessions.

---

## Session Per Project

Each project maintains its own sessions:

```
Project: my-app
├── session_1 (active)
│   └── Messages: "Add auth", "Fix bug"
├── session_2 (archived)
│   └── Messages: "Refactor"
└── session_3 (active)
    └── Messages: "Add tests"

Project: my-lib
├── session_4 (active)
    └── Messages: "Update API"
```

---

## Project Configuration

### Directory Structure

```
my-app/
├── .opencode/
│   ├── config.json          # Project config
│   ├── agent/               # Project agents
│   └── tool/                # Project tools
├── src/
└── tests/
```

### Config Inheritance

```
Global (~/.opencode/config.json)
    ↓ (merge)
Project (.opencode/config.json)
    ↓ (merge)
CLI Flags
    ↓
Final Config
```

---

## Use Cases

### Multiple Projects

```bash
# Terminal 1: Work on backend
cd ~/projects/backend
opencode tui

# Terminal 2: Work on frontend
cd ~/projects/frontend
opencode tui

# Terminal 3: Work on mobile
cd ~/projects/mobile
opencode tui
```

Each has isolated:
- Sessions
- Configuration
- History
- State

### Monorepo

```bash
# Root level
cd ~/monorepo
opencode "Overview changes"

# Package 1
cd ~/monorepo/packages/api
opencode "Add endpoint"

# Package 2
cd ~/monorepo/packages/web
opencode "Update UI"
```

Shares Git root but separate OpenCode instances.

---

## Implementation

### Instance State

```typescript
export namespace Instance {
  export let directory: string
  export let worktree: string
  export let project: ProjectInfo
  
  export function state<T>(
    init: () => T | Promise<T>,
    cleanup?: (state: T) => Promise<void>
  ): () => Promise<T>
}
```

**Usage**:
```typescript
const myState = Instance.state(() => {
  return { data: new Map() }
})

// Access state (per-instance)
const state = await myState()
state.data.set(key, value)
```

### State Isolation

State is isolated per instance (directory):

```typescript
// Instance 1: /projects/app1
const state1 = await myState()  // { data: Map {} }

// Instance 2: /projects/app2
const state2 = await myState()  // { data: Map {} } (different)
```

---

## Best Practices

**Organization**:
- One OpenCode instance per project
- Use separate sessions for separate tasks
- Archive old sessions

**Configuration**:
- Set project-specific settings in `.opencode/`
- Use global config for personal preferences
- Document project-specific agents/tools

**Workflows**:
- Use TUI for long-running projects
- Use CLI for quick tasks
- Use server for multiple clients

---

For implementation, see `packages/opencode/src/project/` and spec at `specs/project.md`.

