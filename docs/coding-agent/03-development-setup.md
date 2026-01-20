# Development Setup

This guide covers setting up the pi-coding-agent project for development, including prerequisites, installation, building, testing, and recommended development workflows.

## Prerequisites

### Node.js Version

The project requires **Node.js 20.0.0 or later**.

```json
// From package.json
"engines": {
  "node": ">=20.0.0"
}
```

Check your Node.js version:

```bash
node --version
```

### Additional Requirements

- **npm** (comes with Node.js)
- **Git** for version control
- **Bun 1.0+** (optional, for building standalone binaries)
- **Bash shell** (for running tools and tests)

## Installation from Source

### Clone the Repository

The project is part of a monorepo:

```bash
git clone https://github.com/badlogic/pi-mono.git
cd pi-mono
```

### Install Dependencies

Install all dependencies from the monorepo root:

```bash
npm install
```

Build all packages in the monorepo:

```bash
npm run build
```

### Navigate to the Coding Agent Package

```bash
cd packages/coding-agent
```

## Available npm Scripts

The project provides several npm scripts for development and building:

### Build Scripts

| Script | Command | Description |
|--------|---------|-------------|
| `clean` | `shx rm -rf dist` | Remove the `dist/` directory |
| `build` | `tsgo -p tsconfig.build.json && ...` | Compile TypeScript, copy assets |
| `build:binary` | `npm run build && bun build --compile ...` | Build standalone executable with Bun |
| `prepublishOnly` | `npm run clean && npm run build` | Auto-runs before npm publish |

### Test Scripts

| Script | Command | Description |
|--------|---------|-------------|
| `test` | `vitest --run` | Run all tests once (non-watch mode) |

### Asset Management

The build process includes two asset-copying steps:

1. **copy-assets**: Copies JSON theme files and HTML export templates to `dist/`
2. **copy-binary-assets**: Copies additional assets needed for standalone binaries (themes, export templates, WASM files, docs, examples)

## Building the Project

### Development Build

Compile TypeScript to JavaScript:

```bash
npm run build
```

This command:
1. Compiles TypeScript using `tsgo` with `tsconfig.build.json`
2. Sets execute permissions on `dist/cli.js`
3. Copies theme JSON files to `dist/modes/interactive/theme/`
4. Copies HTML export templates to `dist/core/export-html/`

**TypeScript Configuration** (`tsconfig.build.json`):

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.d.ts", "src/**/*.d.ts"]
}
```

The configuration:
- Extends the monorepo base config
- Outputs compiled JavaScript to `dist/`
- Includes all TypeScript files in `src/`
- Excludes existing type definitions

### Production Build

For a clean build from scratch:

```bash
npm run clean
npm run build
```

### Building Standalone Binaries

To create platform-specific executables (requires [Bun](https://bun.sh) 1.0+):

```bash
npm run build:binary
```

This creates a single-file executable at `dist/pi` that bundles:
- Compiled JavaScript code
- Node.js runtime
- All dependencies
- Theme files and export templates
- WebAssembly modules (photon-node for image processing)
- Documentation and examples

The binary can be distributed without requiring Node.js installation on target systems.

**Platform-specific builds** are handled by GitHub Actions for releases:
- macOS Apple Silicon (`pi-darwin-arm64.tar.gz`)
- macOS Intel (`pi-darwin-x64.tar.gz`)
- Linux x64 (`pi-linux-x64.tar.gz`)
- Linux ARM64 (`pi-linux-arm64.tar.gz`)
- Windows x64 (`pi-windows-x64.zip`)

## Running Tests

### Test Framework

The project uses **Vitest** for unit and integration testing.

Run all tests:

```bash
npm test
```

This executes tests in non-watch mode (runs once and exits).

### Test Structure

Tests are located alongside source files with the `.test.ts` suffix:

```
src/
  core/
    compaction/
      compaction.test.ts
    tools/
      edit.test.ts
      truncate.test.ts
```

### Writing Tests

Example test file structure:

```typescript
import { describe, it, expect } from 'vitest';
import { myFunction } from './my-module.js';

describe('myFunction', () => {
  it('should do something', () => {
    const result = myFunction('input');
    expect(result).toBe('expected');
  });
});
```

## Development Workflow

### Local Development Loop

1. Make changes to source files in `src/`
2. Build the project: `npm run build`
3. Test your changes: `npm test`
4. Run the CLI locally: `node dist/cli.js` or `./dist/cli.js`

### Using the Local Build

After building, run the CLI directly:

```bash
# Run from the package directory
./dist/cli.js

# Or use npm link to install globally
npm link
pi  # Now available globally
```

### Watching for Changes

Vitest doesn't have a watch script configured by default, but you can run it manually:

```bash
npx vitest
```

This will watch for file changes and re-run tests automatically.

## Terminal Setup

For the best development experience, configure your terminal to support the [Kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/).

### Recommended Terminals

- **Kitty** (Linux, macOS): Works out of the box
- **iTerm2** (macOS): Works out of the box
- **Ghostty** (macOS): Requires configuration
- **WezTerm** (cross-platform): Requires configuration

### Ghostty Configuration

Add to `~/.config/ghostty/config`:

```
keybind = alt+backspace=text:\x1b\x7f
keybind = shift+enter=text:\n
```

### WezTerm Configuration

Create `~/.wezterm.lua`:

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.enable_kitty_keyboard = true
return config
```

### VS Code Integrated Terminal

Add to `keybindings.json` for `Shift+Enter` support:

```json
{
  "key": "shift+enter",
  "command": "workbench.action.terminal.sendSequence",
  "args": { "text": "\u001b[13;2u" },
  "when": "terminalFocus"
}
```

## Setting Up API Keys

### For Testing

Set environment variables for the AI providers you want to test:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export OPENAI_API_KEY=sk-...
export GEMINI_API_KEY=...
```

### Using auth.json

Create `~/.pi/agent/auth.json` for persistent credentials:

```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-..." },
  "openai": { "type": "api_key", "key": "sk-..." },
  "google": { "type": "api_key", "key": "..." }
}
```

### OAuth Providers

For OAuth-based providers (Anthropic Claude Pro, GitHub Copilot, Google Gemini CLI):

```bash
./dist/cli.js
/login  # Select provider and authorize
```

OAuth credentials are stored in `~/.pi/agent/auth.json` after successful authentication.

## Developing and Testing Extensions

### Extension Locations

Extensions are TypeScript files that extend agent behavior:

- **Global**: `~/.pi/agent/extensions/*.ts`
- **Project**: `.pi/extensions/*.ts`
- **CLI flag**: `--extension path/to/extension.ts`

### Creating a Test Extension

Create `test-extension.ts`:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  console.log("Extension loaded!");

  pi.registerCommand("test", {
    description: "Test command",
    handler: async (args, ctx) => {
      ctx.ui.notify("Extension works!", "info");
    },
  });
}
```

### Loading Extensions

Load your extension during development:

```bash
./dist/cli.js --extension ./test-extension.ts
```

Then use it:

```
/test
```

### Extension Dependencies

Extensions can have their own dependencies. Create a `package.json` next to your extension:

```json
{
  "dependencies": {
    "axios": "^1.0.0"
  }
}
```

Run `npm install` in that directory, and imports will be resolved via [jiti](https://github.com/unjs/jiti):

```typescript
import axios from 'axios';

export default function (pi: ExtensionAPI) {
  // Use axios...
}
```

### Extension Development Tips

1. **Use TypeScript**: Extensions are TypeScript files with full type checking
2. **Hot Reload**: Restart the agent to reload extensions (no hot reload)
3. **Error Handling**: Extension errors are logged but don't crash the agent
4. **Debugging**: Use `console.log()` for debugging (output visible in terminal)
5. **Test in Print Mode**: Use `--print` flag for faster iteration without TUI

Example development workflow:

```bash
# Edit extension
vim my-extension.ts

# Test in print mode
./dist/cli.js -p --extension ./my-extension.ts "test message"

# Test in interactive mode
./dist/cli.js --extension ./my-extension.ts
```

## IDE Setup Recommendations

### Visual Studio Code

Install recommended extensions:

- **ESLint**: For linting TypeScript/JavaScript
- **Prettier**: For code formatting
- **TypeScript**: Built-in support

Recommended `settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "typescript.tsdk": "node_modules/typescript/lib"
}
```

### TypeScript Language Server

The project uses TypeScript 5.7.3. Your IDE should automatically detect the `tsconfig.json` files.

### Debugging Configuration

#### VS Code Launch Configuration

Create `.vscode/launch.json` for debugging:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Pi CLI",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/dist/cli.js",
      "args": [],
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "sourceMaps": true,
      "console": "integratedTerminal"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Pi with Extension",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/dist/cli.js",
      "args": ["--extension", "./test-extension.ts"],
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "sourceMaps": true,
      "console": "integratedTerminal"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Tests",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["test"],
      "skipFiles": ["<node_internals>/**"],
      "console": "integratedTerminal"
    }
  ]
}
```

#### Debugging Tips

**Breakpoint Debugging:**
- Set breakpoints in TypeScript source files
- Use F5 to start debugging session
- Use Debug Console to inspect variables and execute expressions

**Console Debugging:**
- Add `console.log()` statements in source code
- Rebuild with `npm run build`
- Run CLI to see output

**Extension Debugging:**
- Extensions load via jiti, so set breakpoints in extension TypeScript files
- Use `--extension` flag to load specific extension
- Extension errors logged to console and debug log

**Tool Debugging:**
- Tools execute in separate context
- Add logging in tool `execute()` function
- Check tool result details for structured debugging info

**Session Debugging:**
- Inspect session files at `~/.pi/agent/sessions/`
- Use `jq` to parse JSONL files
- Check `~/.pi/agent/pi-coding-agent-debug.log` for errors

## Troubleshooting

### Build Errors

**Issue**: `Cannot find module` errors after build

**Solution**: Ensure all dependencies are installed:

```bash
cd ../../  # Go to monorepo root
npm install
npm run build
cd packages/coding-agent
npm run build
```

### Test Failures

**Issue**: Tests fail with file path errors

**Solution**: Tests expect to run from the package directory:

```bash
cd packages/coding-agent
npm test
```

### Extension Loading Errors

**Issue**: Extension fails to load with TypeScript errors

**Solution**: Ensure your extension exports a default function:

```typescript
export default function (pi: ExtensionAPI) {
  // Extension code
}
```

### Binary Build Errors

**Issue**: `bun: command not found`

**Solution**: Install Bun:

```bash
curl -fsSL https://bun.sh/install | bash
```

---

## Complete Development Workflow Example

This section demonstrates a complete end-to-end workflow from cloning the repository to testing and debugging changes.

### Step 1: Clone and Setup

```bash
# Clone the monorepo
git clone https://github.com/badlogic/pi-mono.git
cd pi-mono

# Install all dependencies from the monorepo root
npm install

# Build all packages
npm run build

# Navigate to the coding agent package
cd packages/coding-agent

# Verify the build
node dist/cli.js --version
```

### Step 2: Make Code Changes

Let's say you want to add a new feature to the Read tool. Here's a realistic workflow:

```bash
# Create a new feature branch
git checkout -b feature/add-file-metadata

# Open the project in your IDE
code .

# Make changes to the source file
# Edit: src/core/tools/read.ts
# Add functionality to include file metadata (size, modified date)
```

Example change to `read.ts`:

```typescript
// Add to the ReadToolDetails interface
interface ReadToolDetails {
  truncation?: TruncationResult;
  metadata?: {
    size: number;
    modified: Date;
    permissions: string;
  };
}

// In the execute function, add metadata collection
const stats = await fs.stat(absolutePath);
const metadata = {
  size: stats.size,
  modified: stats.mtime,
  permissions: stats.mode.toString(8).slice(-3),
};
```

### Step 3: Test Changes

```bash
# Build the changes
npm run build

# Run the test suite to ensure nothing broke
npm test

# Test specific functionality
npm test -- src/core/tools/read.test.ts

# Test the CLI manually with the new feature
./dist/cli.js -p "Read the package.json file and show its metadata"
```

### Step 4: Debug Issues

If you encounter issues, use these debugging strategies:

**Console Debugging:**

```typescript
// Add debug logging in your code
console.log('[DEBUG] File metadata:', metadata);
console.log('[DEBUG] Absolute path:', absolutePath);
```

**VS Code Debugger:**

1. Set breakpoints in the TypeScript source file
2. Use the "Debug Pi CLI" launch configuration
3. Press F5 to start debugging
4. Step through code with F10/F11

**Check debug logs:**

```bash
# Enable debug mode
PI_TIMING=1 ./dist/cli.js

# View the debug log
cat ~/.pi/agent/pi-coding-agent-debug.log
```

**Test in isolation:**

```typescript
// Create a test file: test-read.ts
import { createReadTool } from './dist/core/tools/read.js';

const tool = createReadTool(process.cwd());
const result = await tool.execute('test-1', { path: 'package.json' }, undefined, undefined);
console.log(result);
```

Run it:

```bash
node test-read.ts
```

### Step 5: Build and Verify

```bash
# Clean build from scratch
npm run clean
npm run build

# Run full test suite
npm test

# Verify the CLI works correctly
./dist/cli.js -p "Test the new feature"

# Build a standalone binary (optional)
npm run build:binary
./dist/pi --version
```

### Step 6: Create a Pull Request

```bash
# Stage your changes
git add src/core/tools/read.ts
git add src/core/tools/read.test.ts

# Commit with a descriptive message
git commit -m "feat(read): add file metadata to read tool results

- Include file size, modification date, and permissions
- Update ReadToolDetails interface
- Add tests for metadata collection"

# Push to your fork
git push origin feature/add-file-metadata

# Create a PR on GitHub
# Visit: https://github.com/badlogic/pi-mono/compare
```

### Common Development Tasks

**Watch mode for rapid iteration:**

```bash
# Terminal 1: Watch TypeScript compilation
npx tsc -p tsconfig.build.json --watch

# Terminal 2: Run tests in watch mode
npx vitest

# Terminal 3: Test your changes
./dist/cli.js -p "Your prompt here"
```

**Testing extensions:**

```bash
# Create a test extension
cat > test-ext.ts << 'EOF'
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("hello", {
    description: "Test command",
    handler: async (args, ctx) => {
      ctx.ui.notify("Hello from test extension!", "info");
    },
  });
}
EOF

# Test the extension
./dist/cli.js --extension ./test-ext.ts

# In the CLI, run:
# /hello
```

**Performance profiling:**

```bash
# Enable timing instrumentation
PI_TIMING=1 ./dist/cli.js

# Check performance metrics in the debug log
grep -E "took|ms" ~/.pi/agent/pi-coding-agent-debug.log
```

---

## Next Steps

- **Read the API Reference** ([04-api-reference.md](./04-api-reference.md)) to understand the SDK
- **Explore Examples** in `examples/` directory for SDK usage patterns
- **Review Extensions** in `examples/extensions/` for extension patterns
- **Check Documentation** in `docs/` for advanced topics (compaction, skills, RPC mode)
