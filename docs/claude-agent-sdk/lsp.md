# LSP Integration in Claude Code (v2.1.42)

> How Claude Code uses Language Server Protocol (LSP) for read-only code intelligence inside the agent runtime.

**Source (v2.1.42)**: `@anthropic-ai/claude-code/cli.js` (search: `var VSA = 'LSP'`, `name: VSA`, `lspServers: x.union([`)

---

## Table of Contents

1. [What This Is](#what-this-is)
2. [When the LSP Tool Is Enabled](#when-the-lsp-tool-is-enabled)
3. [Tool Interface (Input/Output)](#tool-interface-inputoutput)
4. [Supported Operations](#supported-operations)
5. [Configuring LSP Servers (Plugin Manifest)](#configuring-lsp-servers-plugin-manifest)
6. [Behavior, Permissions, and Validation](#behavior-permissions-and-validation)
7. [Troubleshooting](#troubleshooting)
8. [Examples](#examples)

---

## What This Is

Claude Code includes a built-in, **read-only** tool named `LSP` that can perform common code-intelligence queries (definition, references, hover, symbols, call hierarchy) by talking to configured Language Server Protocol servers.

This is separate from MCP:
- MCP extends Claude Code with external tools/resources.
- LSP provides editor-like code navigation over your local workspace using language servers.

---

## When the LSP Tool Is Enabled

The `LSP` tool is only available when Claude Code has at least one configured LSP server and at least one server is not in an error state.

**Source anchors**:
- Tool definition and enable check: search `name: VSA` and `isEnabled() {` near `var VSA = 'LSP'`.
- Enable logic looks for server configs and checks server state (search: `getAllServers()` and `state !== 'error'` near `name: VSA`).

---

## Tool Interface (Input/Output)

### Input schema

**Source (v2.1.42)**: search `(WFY = L6(() => x.strictObject({` in `@anthropic-ai/claude-code/cli.js`.

```ts
type LspInput = {
  operation:
    | 'goToDefinition'
    | 'findReferences'
    | 'hover'
    | 'documentSymbol'
    | 'workspaceSymbol'
    | 'goToImplementation'
    | 'prepareCallHierarchy'
    | 'incomingCalls'
    | 'outgoingCalls';
  filePath: string;     // absolute or relative path
  line: number;         // 1-based
  character: number;    // 1-based
};
```

### Output schema

**Source (v2.1.42)**: search `(GFY = L6(() => x.object({` in `@anthropic-ai/claude-code/cli.js`.

```ts
type LspOutput = {
  operation: LspInput['operation'];
  result: string;         // formatted text result
  filePath: string;
  resultCount?: number;   // optional counts when applicable
  fileCount?: number;
};
```

---

## Supported Operations

All operations are read-only.

- `goToDefinition`: locate the symbol definition at a cursor position.
- `findReferences`: list reference locations for a symbol.
- `hover`: show hover text/documentation at a position.
- `documentSymbol`: list symbols in a document (file).
- `workspaceSymbol`: search for symbols across the workspace.
- `goToImplementation`: find implementation(s) for an interface/abstract symbol.
- `prepareCallHierarchy`: compute call-hierarchy handles for a symbol.
- `incomingCalls` / `outgoingCalls`: expand call hierarchy.

**Source (v2.1.42)**: the operation enum appears in both the input and output schemas (search: `operation: x.enum([` near `WFY` and `GFY`).

---

## Configuring LSP Servers (Plugin Manifest)

Claude Code supports configuring LSP servers via plugin metadata.

### `lspServers` field

The `lspServers` field is a union that supports:
- A string path to a `.lsp.json` file (relative to the plugin root).
- An inline mapping of server name → server config.
- An array mixing paths and inline mappings.

**Source (v2.1.42)**: search `lspServers: x.union([` in `@anthropic-ai/claude-code/cli.js`.

### Server config shape

**Source (v2.1.42)**: search `(lH1 = L6(() => x.strictObject({` in `@anthropic-ai/claude-code/cli.js`.

Key fields include:
- `command` (required): binary to start the LSP server. Must not contain spaces; use `args`.
- `args` (optional): argument list for the server.
- `extensionToLanguage` (required): map file extensions to LSP language IDs (must be non-empty).
- `transport` (default `stdio`): `stdio` or `socket`.
- `env` (optional): environment variables for the server process.
- `initializationOptions` (optional): sent during initialization.
- `settings` (optional): sent via `workspace/didChangeConfiguration`.
- `workspaceFolder` (optional): workspace folder path.
- `startupTimeout` / `shutdownTimeout` / `restartOnCrash` / `maxRestarts` (optional).

### Implementation note on timeout/restart fields

Some optional fields may be accepted by the schema but not fully implemented (for example, warnings indicating that certain fields should be removed).

**Source (v2.1.42)**: search:
- `restartOnCrash is not yet implemented`
- `startupTimeout is not yet implemented`
- `shutdownTimeout is not yet implemented`

---

## Behavior, Permissions, and Validation

### Read-only and concurrency-safe

The `LSP` tool declares itself read-only and concurrency-safe.

**Source (v2.1.42)**: search `isReadOnly() { return !0; }` and `isConcurrencySafe() { return !0; }` near `name: VSA`.

### Input validation

Claude Code validates that `filePath` resolves to an existing file (with special-casing for UNC paths).

**Source (v2.1.42)**: search `File does not exist:` and `Path is not a file:` near `validateInput` in the `LSP` tool definition.

### Permissions

Even though the tool is read-only, Claude Code still performs a permission check in the tool pipeline.

**Source (v2.1.42)**: search `async checkPermissions(A, q)` near `name: VSA`.

---

## Troubleshooting

### Plugin LSP config errors

Common error paths include invalid LSP config, server start failures, crashes, request timeouts, and request failures.

**Source (v2.1.42)**: search:
- `lsp-config-invalid`
- `lsp-server-start-failed`
- `lsp-server-crashed`
- `lsp-request-timeout`
- `lsp-request-failed`

### “LSP tool is unavailable”

If no servers are configured, or all servers are in an error state, the tool remains disabled.

**Source (v2.1.42)**: search `getAllServers()` and `state !== 'error'` near the `LSP` tool `isEnabled()` implementation.

---

## Examples

### Go to definition

```ts
LSP({
  operation: 'goToDefinition',
  filePath: 'src/index.ts',
  line: 42,
  character: 10,
});
```

### Hover at a symbol

```ts
LSP({
  operation: 'hover',
  filePath: 'src/index.ts',
  line: 42,
  character: 10,
});
```
