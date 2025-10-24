# Gemini CLI - IDE Integration Reference

**Generated:** 2024-10-24
**Purpose:** Complete documentation of IDE integration, protocol, and code assist features

**STATUS:** ⚠️ **PREVIEW** - VSCode extension is in preview

---

## Table of Contents

1. [IDE Integration Overview](#ide-integration-overview)
2. [VSCode Extension](#vscode-extension)
3. [IDE Protocol](#ide-protocol)
4. [Diff Management](#diff-management)
5. [Open Files Tracking](#open-files-tracking)
6. [IDE Client (Core)](#ide-client-core)
7. [Code Assist Features](#code-assist-features)
8. [Workspace Trust Integration](#workspace-trust-integration)
9. [Environment Synchronization](#environment-synchronization)
10. [IDE Detection](#ide-detection)

---

## IDE Integration Overview

### What is IDE Integration?

**IDE Integration** enables Gemini CLI to run as a companion to your IDE (VSCode, Zed, etc.) with deep integration:

- **Diff Previews** - Show file changes in IDE diff viewer
- **File Awareness** - Access currently open files
- **Workspace Context** - Understand project structure
- **Trust Integration** - Respect workspace trust settings
- **Environment Sync** - Share environment variables

---

### Supported IDEs

| IDE | Status | Package |
|-----|--------|---------|
| **VSCode** | ✅ Preview | `gemini-cli-vscode-ide-companion` |
| **Zed** | ⚠️ Alpha | Built-in integration (ACP) |
| **Others** | ❌ Planned | TBD |

---

### Architecture

```
┌───────────────────────────────────────────────────────────┐
│                     IDE (VSCode/Zed)                       │
│  - Extension/Plugin                                        │
│  - IDE Server (HTTP)                                       │
│  - Diff Manager                                            │
│  - Open Files Manager                                      │
└─────────────────────────┬─────────────────────────────────┘
                          ↓ HTTP / JSON-RPC
┌───────────────────────────────────────────────────────────┐
│                    Gemini CLI (Core)                       │
│  - IdeClient (packages/core/src/ide/ide-client.ts)        │
│  - IDE Protocol (JSON-RPC over HTTP)                      │
│  - Code Assist Integration                                │
└───────────────────────────────────────────────────────────┘
```

---

## VSCode Extension

### Extension Details

**Name:** Gemini CLI Companion  
**ID:** `Google.gemini-cli-vscode-ide-companion`  
**Publisher:** Google  
**Status:** Preview (`"preview": true`)  
**VSCode Version:** `^1.99.0`

---

### Installation

#### From Marketplace

```bash
# Search in VSCode Extensions view
# Or install from command line
code --install-extension Google.gemini-cli-vscode-ide-companion
```

---

#### From Source

```bash
cd packages/vscode-ide-companion
npm install
npm run package
code --install-extension gemini-cli-vscode-ide-companion-*.vsix
```

---

### Extension Activation

**Activation Event:** `onStartupFinished`

**What it does:**
- Starts immediately after VSCode starts
- No explicit activation command needed
- Always active in background

---

### Extension Commands

#### `gemini-cli.runGeminiCLI`

**Title:** "Gemini CLI: Run"

**Description:** Opens Gemini CLI terminal in current workspace folder

**Usage:**
1. Open Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`)
2. Type "Gemini CLI: Run"
3. Select workspace folder (if multiple)
4. Terminal opens with `gemini` command

---

#### `gemini.diff.accept`

**Title:** "Gemini CLI: Accept Diff"  
**Icon:** `$(check)`  
**Keybinding:** `Ctrl+S` / `Cmd+S` (when diff is visible)

**Description:** Accept proposed file changes from diff view

**When Available:** Only when `gemini.diff.isVisible` context is true

---

#### `gemini.diff.cancel`

**Title:** "Gemini CLI: Close Diff Editor"  
**Icon:** `$(close)`

**Description:** Close diff editor without accepting changes

**When Available:** Only when `gemini.diff.isVisible` context is true

---

#### `gemini-cli.showNotices`

**Title:** "Gemini CLI: View Third-Party Notices"

**Description:** Shows `NOTICES.txt` with third-party license information

---

### Extension Configuration

#### `gemini-cli.debug.logging.enabled`

**Type:** `boolean`  
**Default:** `false`

**Description:** Enable detailed logging for debugging the Gemini CLI Companion

**Usage:**

```json
// settings.json
{
  "gemini-cli.debug.logging.enabled": true
}
```

**Effect:** Logs output to "Gemini CLI IDE Companion" output channel

---

### Extension Lifecycle

#### Activation

From `extension.ts:activate()`:

```typescript
export async function activate(context: vscode.ExtensionContext) {
  // 1. Create output channel for logging
  logger = vscode.window.createOutputChannel('Gemini CLI IDE Companion');
  
  // 2. Check for extension updates
  await checkForUpdates(context, log, isManagedExtensionSurface);
  
  // 3. Initialize diff manager
  const diffContentProvider = new DiffContentProvider();
  const diffManager = new DiffManager(log, diffContentProvider);
  
  // 4. Register commands and providers
  context.subscriptions.push(
    vscode.workspace.registerTextDocumentContentProvider(DIFF_SCHEME, diffContentProvider),
    vscode.commands.registerCommand('gemini.diff.accept', ...),
    vscode.commands.registerCommand('gemini.diff.cancel', ...),
    // ... other commands
  );
  
  // 5. Start IDE server
  ideServer = new IDEServer(log, diffManager);
  await ideServer.start(context);
  
  // 6. Sync workspace environment
  ideServer.syncEnvVars();
  
  // 7. Show welcome message (first time only)
  if (!context.globalState.get(INFO_MESSAGE_SHOWN_KEY)) {
    vscode.window.showInformationMessage('Gemini CLI Companion extension successfully installed.');
    context.globalState.update(INFO_MESSAGE_SHOWN_KEY, true);
  }
}
```

---

#### Deactivation

```typescript
export async function deactivate(): Promise<void> {
  // 1. Stop IDE server
  if (ideServer) {
    await ideServer.stop();
  }
  
  // 2. Dispose logger
  if (logger) {
    logger.dispose();
  }
}
```

---

### Auto-Update Check

**Frequency:** On activation

**Behavior:**
1. Fetches latest version from VSCode Marketplace
2. Compares with installed version
3. Shows notification if update available
4. User can click to update

**Skipped for:** Managed extension surfaces (Firebase Studio, Cloud Shell)

---

## IDE Protocol

### Protocol Type

**Transport:** HTTP + JSON-RPC 2.0

**Location:** `packages/core/src/ide/types.ts`

---

### IDE Server

**Location:** `packages/vscode-ide-companion/src/ide-server.ts`

**Purpose:** HTTP server that exposes IDE functionality to Gemini CLI

---

### Server Configuration

**Port:** Auto-assigned, shared via environment variable `GEMINI_CLI_IDE_SERVER_PORT`

**Framework:** Express.js

**CORS:** Enabled (localhost only)

---

### Protocol Endpoints

#### POST `/diff`

**Purpose:** Open diff view in IDE

**Request:**

```typescript
interface IdeOpenDiffRequest {
  filePath: string;           // Absolute path to file
  proposedContent: string;    // New file content
}
```

**Response:**

```typescript
interface IdeOpenDiffResponse {
  diffId: string;             // Unique diff identifier
  status: 'opened';
}
```

**Behavior:**
1. Creates diff document with `gemini-diff:` scheme
2. Opens split diff view (original vs proposed)
3. User can edit in diff view
4. Returns diff ID for tracking

---

#### POST `/diff/:diffId/accept`

**Purpose:** Accept diff changes

**Request:** (empty body)

**Response:**

```typescript
interface IdeDiffAcceptedNotification {
  status: 'accepted';
  content: string;            // Final content (may be user-modified)
}
```

**Behavior:**
1. Reads content from diff view (may differ from original proposal)
2. Writes content to original file
3. Closes diff view
4. Returns final content

---

#### POST `/diff/:diffId/cancel`

**Purpose:** Cancel diff without accepting

**Request:** (empty body)

**Response:**

```typescript
interface IdeDiffClosedNotification {
  status: 'cancelled';
}
```

**Behavior:**
1. Closes diff view
2. Discards proposed changes

---

#### GET `/open-files`

**Purpose:** Get list of currently open files

**Response:**

```typescript
interface IdeOpenFilesResponse {
  files: string[];            // Array of absolute file paths
}
```

**Behavior:**
- Returns paths of all open text documents
- Excludes untitled documents
- Excludes non-file schemes (e.g., `gemini-diff:`)

---

#### GET `/workspace`

**Purpose:** Get workspace information

**Response:**

```typescript
interface IdeWorkspaceResponse {
  folders: Array<{
    uri: string;              // Workspace folder URI
    name: string;             // Folder name
  }>;
  isTrusted: boolean;         // Workspace trust status
}
```

---

#### POST `/log`

**Purpose:** Log message from Gemini CLI to IDE output

**Request:**

```typescript
interface IdeLogRequest {
  level: 'info' | 'warn' | 'error';
  message: string;
}
```

**Response:** `200 OK`

---

### JSON-RPC Notifications

**Protocol:** JSON-RPC 2.0 over HTTP

**Used for:** Async events (diff accepted, diff cancelled)

**Example:**

```json
{
  "jsonrpc": "2.0",
  "method": "diff/accepted",
  "params": {
    "diffId": "diff-123",
    "content": "final content here"
  }
}
```

---

## Diff Management

### DiffManager Class

**Location:** `packages/vscode-ide-companion/src/diff-manager.ts`

**Purpose:** Manage lifecycle of diff views

---

### Diff Lifecycle

```
Open Diff Request (from Gemini CLI)
    ↓
Create Diff Document (gemini-diff:<uuid>)
    ↓
Show Split Diff View
    ↓
User Reviews/Edits in Diff View
    ↓
┌─────────────────────────────┐
│  Accept  │  Cancel  │ Close │
└────┬─────┴─────┬────┴───┬───┘
     ↓           ↓        ↓
Accept Diff   Cancel    Cancel
(Write file) (Discard)  (Discard)
     ↓           ↓        ↓
Send Notification to Gemini CLI
     ↓
Close Diff View
```

---

### Diff Document

**Scheme:** `gemini-diff:`

**URI Format:** `gemini-diff:<uuid>?originalPath=<path>`

**Content Provider:** `DiffContentProvider`

**Storage:** In-memory Map

---

### DiffContentProvider

**Purpose:** Provide content for diff documents

**Implementation:**

```typescript
export class DiffContentProvider implements vscode.TextDocumentContentProvider {
  private content = new Map<string, string>();
  private onDidChangeEmitter = new vscode.EventEmitter<vscode.Uri>();
  
  get onDidChange(): vscode.Event<vscode.Uri> {
    return this.onDidChangeEmitter.event;
  }
  
  provideTextDocumentContent(uri: vscode.Uri): string {
    return this.content.get(uri.toString()) ?? '';
  }
  
  setContent(uri: vscode.Uri, content: string): void {
    this.content.set(uri.toString(), content);
    this.onDidChangeEmitter.fire(uri);  // Trigger refresh
  }
  
  deleteContent(uri: vscode.Uri): void {
    this.content.delete(uri.toString());
  }
}
```

---

### Diff State Tracking

```typescript
interface DiffInfo {
  originalFilePath: string;   // Absolute path to original file
  newContent: string;         // Proposed content
  rightDocUri: vscode.Uri;    // Diff document URI
}

private diffDocuments = new Map<string, DiffInfo>();
```

---

### Auto-Cleanup

**Trigger:** `onDidCloseTextDocument` event

**Behavior:**
- Detects when diff document is closed manually
- Sends cancellation notification to Gemini CLI
- Cleans up diff state

---

## Open Files Tracking

### OpenFilesManager Class

**Location:** `packages/vscode-ide-companion/src/open-files-manager.ts`

**Purpose:** Track which files are currently open in IDE

---

### Tracked Information

- **File Paths** - Absolute paths of open text documents
- **Document Changes** - Listen for open/close events
- **Active Editor** - Currently focused file

---

### Events

```typescript
// When document opened
vscode.workspace.onDidOpenTextDocument((doc) => {
  if (doc.uri.scheme === 'file') {
    openFiles.add(doc.uri.fsPath);
  }
});

// When document closed
vscode.workspace.onDidCloseTextDocument((doc) => {
  if (doc.uri.scheme === 'file') {
    openFiles.delete(doc.uri.fsPath);
  }
});
```

---

### Use Cases

1. **Context Awareness** - Gemini CLI knows what files you're working on
2. **Smart Suggestions** - Can reference open files in responses
3. **Targeted Edits** - Prefer editing files already open
4. **Navigation** - Jump to specific open files

---

## IDE Client (Core)

### IdeClient Class

**Location:** `packages/core/src/ide/ide-client.ts`

**Purpose:** Client library for communicating with IDE server

---

### Singleton Pattern

```typescript
export class IdeClient {
  private static instance?: IdeClient;
  
  static async getInstance(): Promise<IdeClient | undefined> {
    if (!IdeClient.instance) {
      const port = process.env['GEMINI_CLI_IDE_SERVER_PORT'];
      if (port) {
        IdeClient.instance = new IdeClient(`http://localhost:${port}`);
      }
    }
    return IdeClient.instance;
  }
}
```

---

### Key Methods

#### `openDiff()`

```typescript
async openDiff(filePath: string, proposedContent: string): Promise<IdeDiffResult> {
  const response = await fetch(`${this.baseUrl}/diff`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ filePath, proposedContent })
  });
  
  const { diffId } = await response.json();
  
  // Return promise that resolves when user accepts/cancels
  return new Promise((resolve, reject) => {
    this.pendingDiffs.set(diffId, { resolve, reject });
  });
}
```

---

#### `getOpenFiles()`

```typescript
async getOpenFiles(): Promise<string[]> {
  const response = await fetch(`${this.baseUrl}/open-files`);
  const data = await response.json();
  return data.files;
}
```

---

#### `getWorkspaceInfo()`

```typescript
async getWorkspaceInfo(): Promise<{ isTrusted: boolean; folders: WorkspaceFolder[] }> {
  const response = await fetch(`${this.baseUrl}/workspace`);
  return response.json();
}
```

---

#### `isDiffingEnabled()`

```typescript
isDiffingEnabled(): boolean {
  return process.env['GEMINI_CLI_IDE_SERVER_PORT'] !== undefined;
}
```

---

## Code Assist Features

### OAuth Integration

**Location:** `packages/core/src/code_assist/oauth2.ts`

**Purpose:** Authenticate with Google Cloud services for code assist APIs

**Flow:**
1. User initiates OAuth via browser
2. Authorization code received
3. Exchange for access token
4. Token stored securely
5. Token refreshed automatically

---

### Code Assist Types

**Location:** `packages/core/src/code_assist/types.ts`

**Features:**

1. **Code Completion** - AI-powered code suggestions
2. **Code Explanation** - Explain code snippets
3. **Code Generation** - Generate code from natural language
4. **Refactoring** - Suggest code improvements

---

### Integration Points

**Disabled by Default:** Code assist features are not fully integrated yet

**Future:** Will provide IDE-native code completion powered by Gemini

---

## Workspace Trust Integration

### Trust Detection

**Method:** Query IDE workspace trust status

**Endpoint:** `GET /workspace`

**Response Field:** `isTrusted: boolean`

---

### Trust Integration Flow

```
Gemini CLI Starts
    ↓
Check if IDE mode enabled (GEMINI_CLI_IDE_SERVER_PORT set)
    ↓ Yes
Query Workspace Trust from IDE
    ↓
┌────────────────────────────┐
│ Trusted?                   │
└──┬──────────────────┬──────┘
   ↓ Yes              ↓ No
Allow File          Prompt User
Operations         (If folder trust enabled)
```

---

### Trust Precedence

**IDE Trust** > **Local Trust File**

If IDE integration is enabled:
- IDE trust status takes precedence
- Local `trustedFolders.json` is ignored
- User cannot override IDE trust decision

---

## Environment Synchronization

### Workspace Environment Variables

**Purpose:** Share environment variables from VSCode to Gemini CLI terminal

**Implementation:** `ideServer.syncEnvVars()`

**When Synced:**
- On extension activation
- On workspace folder change (`onDidChangeWorkspaceFolders`)
- On workspace trust change (`onDidGrantWorkspaceTrust`)

---

### Synced Variables

**Environment Variables Set:**

```typescript
process.env['GEMINI_CLI_IDE_SERVER_PORT'] = serverPort;
process.env['GEMINI_CLI_IDE_WORKSPACE_PATH'] = workspacePath;
process.env['TERM_PROGRAM'] = 'vscode';  // Or 'zed' for Zed editor
```

---

### How Gemini CLI Detects IDE

```typescript
// In Gemini CLI
if (process.env['GEMINI_CLI_IDE_SERVER_PORT']) {
  // IDE mode enabled
  const ideClient = await IdeClient.getInstance();
}
```

---

## IDE Detection

### Automatic IDE Detection

**Location:** `packages/core/src/ide/detect-ide.ts`

**Method:** `detectIdeFromEnv()`

**Detection Criteria:**

```typescript
export function detectIdeFromEnv(): IdeInfo {
  if (process.env['TERM_PROGRAM'] === 'vscode') {
    return IDE_DEFINITIONS.vscode;
  }
  if (process.env['TERM_PROGRAM'] === 'Apple_Terminal') {
    return IDE_DEFINITIONS.terminal;
  }
  if (process.env['ZED']) {
    return IDE_DEFINITIONS.zed;
  }
  if (process.env['CLOUDSHELL_ENVIRONMENT']) {
    return IDE_DEFINITIONS.cloudshell;
  }
  // ... more checks
  
  return IDE_DEFINITIONS.unknown;
}
```

---

### Supported IDE Definitions

```typescript
const IDE_DEFINITIONS = {
  vscode: { name: 'vscode', displayName: 'Visual Studio Code' },
  zed: { name: 'zed', displayName: 'Zed' },
  terminal: { name: 'terminal', displayName: 'Terminal' },
  cloudshell: { name: 'cloudshell', displayName: 'Cloud Shell' },
  firebasestudio: { name: 'firebasestudio', displayName: 'Firebase Studio' },
  unknown: { name: 'unknown', displayName: 'Unknown' }
};
```

---

## IDE Integration Gotchas

### 1. Preview Status

**Issue:** VSCode extension is in preview (`"preview": true`)

**Effect:** APIs may change, features may be incomplete

**Recommendation:** Test thoroughly before production use

---

### 2. Port Auto-Assignment

**Issue:** IDE server port is auto-assigned

**Effect:** Port may change between sessions

**Mitigation:** Environment variable `GEMINI_CLI_IDE_SERVER_PORT` is always up-to-date

---

### 3. Diff Race Condition

**Issue:** File may be modified on disk while diff is open

**Status:** Known issue (TODO comment in code)

**Workaround:** Avoid editing files externally while diff is open

---

### 4. No Authentication

**Issue:** IDE server has no authentication

**Security:** Only listens on localhost, but still vulnerable to local attacks

**Recommendation:** Future versions should add authentication token

---

### 5. Diff Content May Be Modified

**Issue:** User can edit content in diff view before accepting

**Effect:** Final content may differ from AI-proposed content

**Behavior:** This is intentional - allows user refinement

---

### 6. Trust Integration May Block Operations

**Issue:** If IDE workspace is untrusted and folder trust is enabled, operations are blocked

**Mitigation:** Trust the workspace in IDE, or disable folder trust in Gemini CLI

---

### 7. Extension Auto-Update Check

**Issue:** Extension checks for updates on every activation

**Effect:** Network request on startup

**Skipped for:** Managed extension surfaces (Firebase Studio, Cloud Shell)

---

### 8. No Zed Extension (Yet)

**Issue:** Zed integration is alpha, no dedicated extension

**Status:** Uses built-in ACP (Agent Context Protocol) support

**Future:** May get dedicated extension like VSCode

---

This comprehensive reference documents all IDE integration features, protocol details, and implementation patterns for Gemini CLI's IDE companions.

