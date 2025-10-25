# OpenCode - Desktop Application

> **Cross-platform desktop app built with SolidJS and Tauri**

---

## Overview

OpenCode Desktop provides a native GUI experience:
- **Native windows** - True desktop application
- **SolidJS UI** - Reactive, performant interface
- **Same features** as TUI - Full functionality
- **Cross-platform** - macOS, Linux, Windows

**Tech Stack**:
- **Framework**: SolidJS
- **Desktop Runtime**: Tauri (or Electron alternative)
- **Styling**: TailwindCSS 4.x
- **Build**: Vite

**Package**: `packages/desktop`

---

## Installation

```bash
# Download from website
open https://opencode.ai/download

# Or build from source
cd packages/desktop
bun install
bun run build
```

---

## Architecture

```
Desktop App
├── SolidJS (UI)
│   ├── Components
│   ├── Pages
│   ├── Providers (state)
│   └── UI Primitives
│
├── Tauri (Native)
│   ├── Window management
│   ├── File system access
│   └── IPC bridge
│
└── OpenCode Server
    └── Same HTTP API as CLI
```

---

## Development

### Setup

```bash
cd packages/desktop
bun install
```

### Run Dev Server

```bash
bun run dev
```

Opens at http://localhost:3000

### Build

```bash
bun run build
```

Outputs to `dist/`

### Type Check

```bash
bun run typecheck
```

---

## Code Style

From `packages/desktop/AGENTS.md`:

**Framework**: SolidJS with TypeScript

**Imports**: Use `@/` alias
```typescript
import Button from "@/ui/button"
import { useSession } from "@/providers/session"
```

**Components**: Function declarations with splitProps
```typescript
function MyComponent(props: MyComponentProps) {
  const [local, others] = splitProps(props, ["value"])
  return <div {...others}>{local.value}</div>
}
```

**Styling**: TailwindCSS with custom theme
```tsx
<div class="bg-background text-foreground p-4 rounded-lg">
  <Button variant="primary">Click me</Button>
</div>
```

**File Structure**:
- `/ui/` - UI primitives (Button, Input, etc.)
- `/components/` - Higher-level components
- `/pages/` - Page components
- `/providers/` - State management

**Naming**:
- PascalCase for components
- camelCase for variables/functions
- snake_case for file names

---

## Project Structure

```
packages/desktop/
├── src/
│   ├── ui/                 # UI primitives
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   └── card.tsx
│   │
│   ├── components/         # App components
│   │   ├── chat/
│   │   │   ├── message.tsx
│   │   │   ├── input.tsx
│   │   │   └── history.tsx
│   │   ├── editor/
│   │   │   ├── code_view.tsx
│   │   │   └── diff_view.tsx
│   │   └── sidebar/
│   │       ├── file_tree.tsx
│   │       └── sessions.tsx
│   │
│   ├── pages/              # Page views
│   │   ├── home.tsx
│   │   ├── chat.tsx
│   │   └── settings.tsx
│   │
│   ├── providers/          # State providers
│   │   ├── session.tsx
│   │   ├── theme.tsx
│   │   └── auth.tsx
│   │
│   ├── lib/                # Utilities
│   │   ├── api.ts
│   │   └── storage.ts
│   │
│   └── app.tsx            # Root component
│
├── public/                # Static assets
├── index.html
├── vite.config.ts
├── tailwind.config.ts
└── package.json
```

---

## Key Components

### Chat Interface

```tsx
function ChatPage() {
  const [session] = useSession()
  const [messages] = createResource(() => fetchMessages(session.id))
  
  return (
    <div class="flex flex-col h-screen">
      <ChatHeader session={session} />
      
      <div class="flex-1 overflow-auto">
        <For each={messages()}>
          {(msg) => <Message message={msg} />}
        </For>
      </div>
      
      <ChatInput onSubmit={sendMessage} />
    </div>
  )
}
```

### File Browser

```tsx
function FileBrowser() {
  const [files] = createResource(fetchProjectFiles)
  
  return (
    <div class="w-64 border-r">
      <FileTree
        files={files()}
        onSelect={(file) => openFile(file)}
      />
    </div>
  )
}
```

### Settings Panel

```tsx
function SettingsPage() {
  const [config, setConfig] = createStore(loadConfig())
  
  return (
    <div class="p-6">
      <h1 class="text-2xl mb-4">Settings</h1>
      
      <Section title="Provider">
        <Select
          value={config.provider}
          onChange={(v) => setConfig("provider", v)}
          options={["anthropic", "openai", "google"]}
        />
      </Section>
      
      <Section title="Model">
        <Select
          value={config.model}
          onChange={(v) => setConfig("model", v)}
          options={availableModels()}
        />
      </Section>
    </div>
  )
}
```

---

## State Management

### Session Provider

```tsx
const SessionContext = createContext()

export function SessionProvider(props) {
  const [session, setSession] = createSignal(null)
  const [messages, setMessages] = createSignal([])
  
  const value = {
    session,
    messages,
    createSession: async (opts) => {
      const s = await api.sessions.create(opts)
      setSession(s)
    },
    sendMessage: async (text) => {
      const msg = await api.sessions.message(session().id, { text })
      setMessages([...messages(), msg])
    }
  }
  
  return (
    <SessionContext.Provider value={value}>
      {props.children}
    </SessionContext.Provider>
  )
}

export const useSession = () => useContext(SessionContext)
```

---

## Styling

### TailwindCSS 4.x

**Configuration** (`tailwind.config.ts`):
```typescript
export default {
  content: ["./src/**/*.{ts,tsx}"],
  theme: {
    extend: {
      colors: {
        background: "var(--background)",
        foreground: "var(--foreground)",
        primary: "var(--primary)",
        secondary: "var(--secondary)",
      }
    }
  }
}
```

### CSS Variables

**Theme system** (`src/theme.css`):
```css
:root {
  --background: 255 255 255;
  --foreground: 0 0 0;
  --primary: 59 130 246;
  --secondary: 148 163 184;
}

[data-theme="dark"] {
  --background: 0 0 0;
  --foreground: 255 255 255;
  --primary: 96 165 250;
  --secondary: 71 85 105;
}
```

---

## Best Practices

**Components**:
- Keep components small and focused
- Use splitProps for prop forwarding
- Leverage SolidJS reactivity
- Avoid unnecessary memo/effects

**Performance**:
- Use `<For>` for lists
- Use `<Show>` for conditionals
- Lazy load routes
- Optimize re-renders

**Styling**:
- Use Tailwind utilities
- Define custom CSS variables for theme
- Follow consistent spacing
- Ensure accessibility

---

For implementation, see `packages/desktop/` and AGENTS.md for detailed guidelines.

