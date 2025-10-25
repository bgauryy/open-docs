# OpenCode - TUI Implementation

> **Terminal User Interface built with Go and Bubble Tea**

---

## Overview

OpenCode's TUI provides a rich terminal interface:
- **Split panes** - Code and chat side-by-side
- **Real-time updates** - Streaming AI responses
- **File browser** - Navigate project files
- **Session management** - Switch between conversations
- **Keyboard shortcuts** - Efficient navigation

**Tech Stack**:
- **Language**: Go
- **Framework**: Bubble Tea (TUI framework)
- **Server Communication**: Stainless SDK (HTTP client)
- **Binary**: Bundled with OpenCode

---

## Starting TUI

```bash
opencode tui

# Or simply
opencode
```

**What happens**:
1. Starts OpenCode server (if not running)
2. Downloads/updates TUI binary (if needed)
3. Launches TUI connected to server
4. Opens current directory as project

---

## Interface Layout

```
┌────────────────────────────────────────────────────────────────┐
│ OpenCode - /projects/my-app                     [Ctrl+Q] Quit │
├──────────────────────┬─────────────────────────────────────────┤
│                      │                                         │
│  Files               │  Chat: session_abc123                  │
│                      │                                         │
│  📁 src/             │  You: Add error handling                │
│    📄 auth.ts        │                                         │
│    📄 user.ts        │  Assistant: I'll add try-catch...      │
│    📄 db.ts          │                                         │
│  📁 tests/           │  [read] auth.ts                         │
│    📄 auth.test.ts   │  00001| export function auth() {        │
│                      │  00002|   // ...                        │
│  📁 docs/            │  00003| }                               │
│                      │                                         │
│                      │  [edit] auth.ts                         │
│                      │  - Added try-catch block                │
│                      │                                         │
│  [Tab] Files         │  [bash] npm test                        │
│  [Enter] Open        │  ✓ All tests passed                     │
│                      │                                         │
│                      │  Done!                                  │
│                      │                                         │
│                      │  > _                                    │
│                      │                                         │
│                      │  [Ctrl+N] New | [Ctrl+L] List          │
└──────────────────────┴─────────────────────────────────────────┘
```

---

## Key Features

### Split Panes

- **Left**: File browser or session list
- **Right**: Active chat conversation
- **Resizable**: Drag to adjust width
- **Toggle**: Hide/show left pane

### Streaming Responses

Real-time text appears as AI generates:
```
Assistant: I'll add err█
Assistant: I'll add error hand█
Assistant: I'll add error handling t█
Assistant: I'll add error handling to that function.
```

### Tool Execution

Visual feedback for tools:
```
[read] Reading auth.ts...
[edit] Editing auth.ts...
  - Line 23: Added try block
  - Line 45: Added catch block
[bash] Running npm test...
  ✓ Test suite passed
```

### Session Management

- **New session**: Ctrl+N
- **List sessions**: Ctrl+L
- **Switch session**: Select from list
- **Archive session**: Delete from list

---

## Keyboard Shortcuts

### Navigation

- `Tab` - Switch between panes
- `↑/↓` - Scroll up/down
- `PgUp/PgDn` - Page up/down
- `Home/End` - Jump to start/end
- `Ctrl+F` - Search

### Sessions

- `Ctrl+N` - New session
- `Ctrl+L` - List sessions
- `Ctrl+W` - Close session
- `Ctrl+R` - Rename session

### Actions

- `Enter` - Send message / Open file
- `Esc` - Cancel input
- `Ctrl+C` - Interrupt AI
- `Ctrl+Q` - Quit

### Files

- `Ctrl+O` - Open file browser
- `Ctrl+P` - Quick file search
- `Enter` - Open file in editor
- `Space` - Preview file

---

## Configuration

**.opencode/config.json**:
```json
{
  "tui": {
    "theme": "dark" | "light",
    "splitRatio": 0.3,
    "showLineNumbers": true,
    "autoScroll": true,
    "keyBindings": {
      "quit": "ctrl+q",
      "newSession": "ctrl+n"
    }
  }
}
```

---

## TUI Command Options

```bash
opencode tui [options]

Options:
  --port PORT          Server port (default: auto)
  --no-browser         Don't open browser for auth
  --theme THEME        Color theme (dark/light)
```

---

## Server Communication

### Architecture

```
TUI (Go)
    │
    ├─ HTTP Client (Stainless SDK)
    │     │
    │     ▼
    │  OpenCode Server (TypeScript)
    │     │
    │     ├─ Sessions
    │     ├─ Messages
    │     └─ Files
    │
    └─ Bubble Tea
          │
          ├─ Rendering
          ├─ Input handling
          └─ State management
```

### API Calls

```go
// Create session
session, err := client.Sessions.Create(ctx, SessionCreateParams{
    Provider: "anthropic",
    Model: "claude-3-5-sonnet",
})

// Send message
stream := client.Sessions.Messages.Stream(ctx, session.ID, MessageParams{
    Parts: []Part{{Type: "text", Text: input}},
})

for stream.Next() {
    chunk := stream.Current()
    renderChunk(chunk)
}
```

---

## Auto-Update

TUI binary auto-updates on launch:

```
Checking for updates...
  Current: v0.1.5
  Latest: v0.1.6
Downloading update... [████████████] 100%
Update complete! Restarting...
```

**Locations**:
- macOS: `~/.opencode/bin/opencode-tui-darwin-arm64`
- Linux: `~/.opencode/bin/opencode-tui-linux-amd64`
- Windows: `~/.opencode/bin/opencode-tui-windows-amd64.exe`

---

## Development

### Building TUI

```bash
cd packages/tui
go build -o opencode-tui
```

### Running Locally

```bash
# Start server
bun dev

# Run TUI (in another terminal)
./opencode-tui --port 3000
```

### Generating Client SDK

When server API changes:

```bash
# Generate new Stainless SDK
bun run generate-sdk

# Update TUI imports
cd packages/tui
go get -u github.com/opencode/client-go
```

---

## Best Practices

**Usage**:
- Use TUI for long coding sessions
- Keep multiple sessions open
- Use keyboard shortcuts for efficiency
- Preview files before opening

**Performance**:
- Close unused sessions
- Clear old history
- Monitor resource usage

---

For implementation, see `packages/tui/`.

