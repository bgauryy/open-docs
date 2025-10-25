# OpenCode - LSP Integration

> **Language Server Protocol integration for real-time code intelligence**

---

## Overview

OpenCode integrates Language Server Protocol (LSP) to provide:
- **Real-time diagnostics** - Errors and warnings as you code
- **Hover information** - Type info and documentation
- **Auto-configuration** - Automatic language detection and LSP server management
- **Multi-language** - TypeScript, Python, Rust, Go, and more

**Files**:
- `lsp/server.ts` (964 lines, 30KB) - LSP server management
- `lsp/client.ts` - LSP client implementation
- `lsp/language.ts` - Language detection

---

## Architecture

### LSP Manager

```typescript
export namespace LSPServer {
  // Start LSP server for a language
  export function start(language: string, workspaceDir: string): Promise<void>
  
  // Get diagnostics for a file
  export function diagnostics(filePath: string): Promise<Diagnostic[]>
  
  // Get hover information
  export function hover(filePath: string, line: number, column: number): Promise<Hover>
  
  // Shutdown server
  export function shutdown(language: string): Promise<void>
}
```

### Supported Languages

- **TypeScript/JavaScript**: `typescript-language-server`
- **Python**: `pyright`
- **Rust**: `rust-analyzer`
- **Go**: `gopls`
- **JSON**: `vscode-json-languageserver`
- **HTML/CSS**: Built-in support

---

## Tool Integration

### lsp-diagnostics Tool

```typescript
{
  tool: "lsp-diagnostics",
  parameters: {
    filePath: "src/auth.ts"
  }
}
```

**Output**:
```
Diagnostics for src/auth.ts:

Line 23: Error - Type 'string | undefined' is not assignable to type 'string'
Line 45: Warning - Unused variable 'result'
```

### lsp-hover Tool

```typescript
{
  tool: "lsp-hover",
  parameters: {
    filePath: "src/auth.ts",
    line: 23,
    column: 10
  }
}
```

**Output**:
```
function authenticate(token: string): Promise<User>

Authenticates a user with the provided token.

Returns:
  Promise<User> - The authenticated user object
```

---

## Auto-Configuration

OpenCode automatically:
1. **Detects language** from file extension
2. **Installs LSP server** if not present
3. **Starts server** on first file access
4. **Caches results** for performance
5. **Restarts on crash** with exponential backoff

**Example Flow**:
```
User edits TypeScript file
    ↓
OpenCode detects .ts extension
    ↓
Checks for typescript-language-server
    ↓
Starts LSP server (if needed)
    ↓
Opens file in LSP
    ↓
Returns diagnostics to AI
```

---

## Configuration

**.opencode/config.json**:
```json
{
  "lsp": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "enabled": true
    },
    "python": {
      "command": "pyright-langserver",
      "args": ["--stdio"],
      "enabled": true
    }
  }
}
```

---

## Best Practices

**For AI Agents**:
- Check diagnostics after edits
- Use hover for type information
- Validate changes with LSP

**For Users**:
- Install language servers globally
- Configure per-language settings
- Enable/disable per project

---

For implementation details, see `packages/opencode/src/lsp/`.

