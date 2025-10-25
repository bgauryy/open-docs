# OpenCode - SDKs & Integrations

> **Official SDKs, plugins, and integration guides**

---

## Overview

OpenCode provides multiple integration options:
- **Go SDK** - Native Go client
- **JavaScript SDK** - TypeScript/JavaScript client
- **Plugin system** - Extend with custom functionality
- **GitHub Actions** - CI/CD integration
- **Slack integration** - Team collaboration

**Packages**: `packages/sdk/`, `packages/plugin/`, `packages/slack/`, `github/`

---

## Go SDK

**Location**: `packages/sdk/go`

### Installation

```bash
go get github.com/sst/opencode/sdk/go
```

### Usage

```go
package main

import (
    "context"
    "fmt"
    "github.com/sst/opencode/sdk/go"
)

func main() {
    client := opencode.NewClient("http://localhost:8080")
    
    // Create session
    session, err := client.Sessions.Create(context.Background(), &opencode.SessionCreateParams{
        Provider: "anthropic",
        Model: "claude-3-5-sonnet",
    })
    if err != nil {
        panic(err)
    }
    
    // Send message
    stream := client.Sessions.Messages.Stream(context.Background(), session.ID, &opencode.MessageParams{
        Parts: []opencode.Part{
            {Type: "text", Text: "Add error handling"},
        },
    })
    
    for stream.Next() {
        chunk := stream.Current()
        fmt.Printf("Type: %s\n", chunk.Type)
    }
}
```

### Features

- Full API coverage
- Streaming support
- Type-safe
- Context support
- Error handling

---

## JavaScript SDK

**Location**: `packages/sdk/js`

### Installation

```bash
npm install @opencode/sdk
# or
bun add @opencode/sdk
```

### Usage

```typescript
import { OpenCodeClient } from '@opencode/sdk'

const client = new OpenCodeClient('http://localhost:8080')

// Create session
const session = await client.sessions.create({
  provider: 'anthropic',
  model: 'claude-3-5-sonnet'
})

// Send message
for await (const chunk of client.sessions.message(session.id, {
  text: 'Add error handling'
})) {
  console.log('Type:', chunk.type)
  
  if (chunk.type === 'text-delta') {
    process.stdout.write(chunk.text)
  }
  
  if (chunk.type === 'tool-call') {
    console.log('Tool:', chunk.tool, chunk.args)
  }
  
  if (chunk.type === 'complete') {
    console.log('Done! Tokens:', chunk.tokens)
  }
}
```

### Features

- Promise-based
- Async iterators
- TypeScript types
- Browser compatible
- Node.js compatible

---

## Plugin System

**Location**: `packages/plugin`

### Creating Plugins

**Structure**:
```
my-plugin/
├── package.json
├── index.ts
└── README.md
```

**index.ts**:
```typescript
import { Plugin } from '@opencode-ai/plugin'

export const plugin: Plugin = {
  name: "my-plugin",
  version: "1.0.0",
  
  // Custom tools
  tool: {
    mytool: {
      description: "My custom tool",
      args: {
        input: {
          type: "string",
          description: "Input parameter"
        }
      },
      async execute(args, ctx) {
        return `Processed: ${args.input}`
      }
    }
  },
  
  // Lifecycle hooks
  async onLoad() {
    console.log("Plugin loaded")
  },
  
  async onUnload() {
    console.log("Plugin unloaded")
  }
}
```

### Installing Plugins

```bash
# Install via npm
npm install my-opencode-plugin

# Or add to .opencode/plugins/
cd .opencode/plugins
git clone https://github.com/user/plugin.git
```

**Configuration** (`.opencode/config.json`):
```json
{
  "plugins": [
    "my-opencode-plugin",
    "./custom-plugin"
  ]
}
```

### Plugin API

```typescript
interface Plugin {
  name: string
  version: string
  
  // Custom tools
  tool?: Record<string, ToolDefinition>
  
  // Custom agents
  agent?: Record<string, AgentDefinition>
  
  // Lifecycle hooks
  onLoad?(): Promise<void>
  onUnload?(): Promise<void>
  
  // Session hooks
  onSessionCreate?(session: Session): Promise<void>
  onSessionMessage?(session: Session, message: Message): Promise<void>
}
```

---

## GitHub Actions

**Location**: `github/`

### Workflow Integration

**.github/workflows/opencode.yml**:
```yaml
name: OpenCode Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup OpenCode
        uses: sst/opencode-action@v1
        with:
          provider: anthropic
          model: claude-3-5-sonnet
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      
      - name: Review PR
        run: |
          opencode run "Review this PR for:
          - Code quality
          - Security issues
          - Performance concerns
          - Test coverage"
      
      - name: Post Results
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs')
            const review = fs.readFileSync('review.md', 'utf8')
            
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: review
            })
```

### Action Inputs

```yaml
with:
  provider: "anthropic"        # AI provider
  model: "claude-3-5-sonnet"   # Model to use
  config: ".opencode/config.json"  # Config file
  working-directory: "."       # Working directory
```

---

## Slack Integration

**Location**: `packages/slack`

### Setup

```bash
# Install Slack package
bun install @opencode/slack

# Configure
opencode slack setup
```

**Interactive setup prompts**:
1. Slack App OAuth token
2. Channel to post in
3. Notification preferences

### Configuration

**.opencode/config.json**:
```json
{
  "slack": {
    "token": "${SLACK_TOKEN}",
    "channel": "#dev-ai",
    "notifications": {
      "sessionStart": true,
      "sessionComplete": true,
      "errors": true
    }
  }
}
```

### Usage

```typescript
import { SlackIntegration } from '@opencode/slack'

const slack = new SlackIntegration({
  token: process.env.SLACK_TOKEN,
  channel: "#dev-ai"
})

// Notify on session start
slack.notify({
  type: "session_start",
  session: session.id,
  message: "Started refactoring auth module"
})

// Share results
slack.share({
  type: "session_complete",
  session: session.id,
  summary: "Added error handling to auth.ts",
  files: ["auth.ts"],
  tokens: 1234,
  cost: 0.05
})
```

---

## IDE Extensions

### VS Code

**Marketplace**: Search "OpenCode"

**Features**:
- Inline code assistance
- Chat panel
- File context
- Diff preview

**Configuration** (`.vscode/settings.json`):
```json
{
  "opencode.enabled": true,
  "opencode.provider": "anthropic",
  "opencode.model": "claude-3-5-sonnet",
  "opencode.serverUrl": "http://localhost:8080"
}
```

### Zed

**Built-in ACP support**:

**Configuration** (`~/.config/zed/settings.json`):
```json
{
  "agents": {
    "opencode": {
      "command": "opencode",
      "args": ["acp"],
      "enabled": true
    }
  }
}
```

---

## Custom Integrations

### HTTP API

```typescript
// Custom integration
class MyService {
  async processCode(code: string) {
    const response = await fetch('http://localhost:8080/sessions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        provider: 'anthropic',
        model: 'claude-3-5-sonnet'
      })
    })
    
    const session = await response.json()
    
    // Send message
    const stream = await fetch(
      `http://localhost:8080/sessions/${session.id}/messages`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          parts: [{ type: 'text', text: `Improve this:\n${code}` }]
        })
      }
    )
    
    // Process stream...
  }
}
```

---

## Best Practices

**SDK Usage**:
- Handle errors gracefully
- Implement timeouts
- Use streaming for long responses
- Close connections properly

**Plugin Development**:
- Keep plugins focused
- Document thoroughly
- Test extensively
- Handle errors

**Integration**:
- Validate inputs
- Implement retries
- Log operations
- Monitor usage

---

For implementations, see `packages/sdk/`, `packages/plugin/`, and `github/`.

