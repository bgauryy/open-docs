# LSP (Language Server Protocol)

**Path**: `packages/opencode/src/lsp`
**Type**: Protocol Layer
**File Count**: 4

## Description

Language Server Protocol client/server implementation for code intelligence.

## Purpose

The LSP component provides code intelligence features like diagnostics, code completion, go-to-definition, and refactoring through the Language Server Protocol. It enables OpenCode to understand code semantics beyond simple text manipulation.

## Key Features

- LSP client implementation
- LSP server capabilities
- Code diagnostics
- Symbol navigation
- Code completion (future)
- Refactoring support (future)

## Component Files

- `client.ts` - LSP client implementation
- `server.ts` - LSP server wrapper
- `diagnostics.ts` - Diagnostic handling
- `types.ts` - LSP type definitions

## Dependencies

### Internal Dependencies
- Used by `packages/opencode/src/tool` (2 imports)

### External Dependencies
- `vscode-languageserver-protocol` - LSP types and utilities
- `vscode-languageclient` - LSP client

## Usage

### Get Diagnostics

```typescript
import { LSP } from './lsp';

// Initialize LSP client
const lsp = await LSP.connect({
  language: 'typescript',
  rootPath: process.cwd()
});

// Get all diagnostics
const diagnostics = await lsp.getDiagnostics();

for (const diagnostic of diagnostics) {
  console.log(`${diagnostic.uri}:${diagnostic.range.start.line + 1}`);
  console.log(`  ${diagnostic.severity}: ${diagnostic.message}`);
}

// Get diagnostics for specific file
const fileDiagnostics = await lsp.getDiagnostics({
  uri: 'file:///path/to/file.ts'
});
```

### Symbol Navigation

```typescript
// Get document symbols
const symbols = await lsp.getDocumentSymbols({
  uri: 'file:///path/to/file.ts'
});

for (const symbol of symbols) {
  console.log(`${symbol.name} (${symbol.kind})`);
}

// Go to definition
const definition = await lsp.getDefinition({
  uri: 'file:///path/to/file.ts',
  position: { line: 10, character: 5 }
});
```

### Code Actions

```typescript
// Get available code actions
const actions = await lsp.getCodeActions({
  uri: 'file:///path/to/file.ts',
  range: {
    start: { line: 10, character: 0 },
    end: { line: 10, character: 20 }
  }
});

// Apply code action
await lsp.applyCodeAction(actions[0]);
```

## Diagnostic Types

```typescript
interface Diagnostic {
  uri: string;
  range: {
    start: { line: number; character: number };
    end: { line: number; character: number };
  };
  severity: 'error' | 'warning' | 'info' | 'hint';
  message: string;
  source?: string;
  code?: string | number;
  relatedInformation?: DiagnosticRelatedInformation[];
}
```

## Supported Languages

The LSP component can connect to language servers for:

- TypeScript/JavaScript (typescript-language-server)
- Python (pyright, pylsp)
- Go (gopls)
- Rust (rust-analyzer)
- Java (jdtls)
- C/C++ (clangd)
- And any LSP-compliant language server

## Language Server Configuration

```json
{
  "lsp": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "initializationOptions": {
        "preferences": {
          "includeInlayParameterNameHints": "all"
        }
      }
    },
    "python": {
      "command": "pyright-langserver",
      "args": ["--stdio"]
    }
  }
}
```

## Integration with Tools

The LSP component is used by:

- **LspDiagnosticTool** - Get diagnostics
- **LspDefinitionTool** - Go to definition (future)
- **LspReferencesTool** - Find references (future)
- **LspRename Tool** - Rename symbol (future)

## Common Use Cases

### Find Type Errors

```typescript
const lsp = await LSP.connect({ language: 'typescript' });
const diagnostics = await lsp.getDiagnostics();

const errors = diagnostics.filter(d => d.severity === 'error');
console.log(`Found ${errors.length} type errors`);

for (const error of errors) {
  console.log(`${error.uri}:${error.range.start.line + 1}`);
  console.log(`  ${error.message}`);
}
```

### Code Analysis

```typescript
// Get all symbols in project
const symbols = await lsp.getWorkspaceSymbols({ query: '' });

// Find unused exports
const unusedExports = symbols.filter(s =>
  s.kind === 'Function' && !s.references
);

// Find complex functions
const complexFunctions = symbols.filter(s =>
  s.kind === 'Function' && s.complexity > 10
);
```

### Automated Refactoring

```typescript
// Find all diagnostics
const diagnostics = await lsp.getDiagnostics();

// Get code actions for each diagnostic
for (const diagnostic of diagnostics) {
  const actions = await lsp.getCodeActions({
    uri: diagnostic.uri,
    range: diagnostic.range,
    diagnostics: [diagnostic]
  });

  // Apply quick fixes
  const quickFixes = actions.filter(a => a.kind === 'quickfix');
  for (const fix of quickFixes) {
    await lsp.applyCodeAction(fix);
  }
}
```

## Performance Considerations

- LSP servers are persistent processes
- Initial indexing can be slow for large projects
- Diagnostics are cached and updated incrementally
- Symbol search uses indexed data
- Connection pooling for multiple files

## Error Handling

```typescript
try {
  const diagnostics = await lsp.getDiagnostics();
} catch (error) {
  if (error.code === 'SERVER_NOT_STARTED') {
    console.error('Language server not running');
    await lsp.start();
  } else if (error.code === 'TIMEOUT') {
    console.error('Request timed out');
  }
}
```

## Future Capabilities

Planned LSP features:

- Code completion suggestions
- Hover information
- Signature help
- Rename symbol
- Find references
- Code formatting
- Organize imports
- Extract function/variable

## Related Documentation

- [LSP Diagnostic Tool API](../api-reference.md#lspdiagnostictool)
- [Tool Execution Flow](../flows/tool-execution-flow.md)
