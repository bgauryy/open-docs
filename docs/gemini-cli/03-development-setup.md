# Development Setup Guide

## Prerequisites

Before setting up the development environment, ensure you have:

- **Node.js 20+**: Required for ES modules and modern JavaScript features
- **npm 10+**: For workspace support
- **Git**: For version control
- **A code editor**: VS Code recommended (with companion extension)

## Initial Setup

### 1. Clone the Repository

```bash
git clone https://github.com/google-gemini/gemini-cli.git
cd gemini-cli
```

### 2. Install Dependencies

```bash
npm ci
```

This installs all dependencies across all workspaces defined in the monorepo.

### 3. Build the Project

```bash
npm run build
```

This builds all packages in the correct order, respecting workspace dependencies.

### 4. Run the CLI in Development Mode

```bash
npm run start
```

Or with debug output:

```bash
npm run debug
```

## Environment Configuration

### Required Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `GEMINI_API_KEY` | API key authentication | `AIza...` |
| `GOOGLE_API_KEY` | Vertex AI authentication | `AIza...` |
| `GOOGLE_GENAI_USE_VERTEXAI` | Enable Vertex AI | `true` |
| `GOOGLE_CLOUD_PROJECT` | GCP project ID | `my-project` |

### Optional Configuration

| Variable | Purpose | Default |
|----------|---------|---------|
| `DEBUG` | Enable debug logging | `false` |
| `NO_COLOR` | Disable colored output | `false` |
| `GEMINI_SANDBOX` | Sandbox mode | `false` |

## Available npm Scripts

### Build Commands

| Script | Description |
|--------|-------------|
| `npm run build` | Build all packages |
| `npm run build:packages` | Build packages only |
| `npm run build:sandbox` | Build sandbox Docker image |
| `npm run build:vscode` | Build VS Code extension |
| `npm run build:all` | Build everything |
| `npm run bundle` | Create production bundle |

### Development Commands

| Script | Description |
|--------|-------------|
| `npm run start` | Run in development mode |
| `npm run debug` | Run with debugger attached |
| `npm run build-and-start` | Build then run |

### Testing Commands

| Script | Description |
|--------|-------------|
| `npm run test` | Run all tests |
| `npm run test:ci` | Run tests in CI mode |
| `npm run test:scripts` | Run script tests |
| `npm run test:e2e` | Run end-to-end tests |
| `npm run test:integration:all` | Run all integration tests |

### Quality Commands

| Script | Description |
|--------|-------------|
| `npm run lint` | Lint with ESLint |
| `npm run lint:fix` | Fix linting errors |
| `npm run format` | Format with Prettier |
| `npm run typecheck` | TypeScript type checking |
| `npm run preflight` | Full quality check |

## Monorepo Structure

The project uses npm workspaces for monorepo management:

```
gemini-cli/
├── packages/
│   ├── cli/           # @google/gemini-cli
│   ├── core/          # @google/gemini-cli-core
│   ├── a2a-server/    # @google/gemini-cli-a2a-server
│   ├── test-utils/    # @google/gemini-cli-test-utils
│   └── vscode-ide-companion/
├── integration-tests/
├── evals/
├── scripts/
├── docs/
└── package.json       # Root workspace config
```

### Working with Workspaces

Run a command in a specific workspace:

```bash
# Run tests in core package
npm test -w @google/gemini-cli-core

# Build CLI package only
npm run build -w @google/gemini-cli
```

Run tests in a specific file:

```bash
# IMPORTANT: Path is relative to workspace root
npm test -w @google/gemini-cli-core -- src/routing/modelRouterService.test.ts
```

### Inter-Package Dependencies

```
@google/gemini-cli
    └── @google/gemini-cli-core
            └── @google/gemini-cli-test-utils
```

## Testing Framework

### Vitest Configuration

The project uses Vitest for testing. Each workspace has its own `vitest.config.ts`:

```typescript
// packages/core/vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./test-setup.ts'],
  },
});
```

### Writing Tests

Tests are co-located with source files:

```
src/
├── tools/
│   ├── shell.ts
│   └── shell.test.ts
```

#### Test Structure

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

describe('ShellTool', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should execute commands', async () => {
    const tool = new ShellTool(mockConfig);
    const result = await tool.execute('echo hello');
    expect(result).toContain('hello');
  });
});
```

#### Mocking Patterns

**Module Mocking:**
```typescript
vi.mock('os', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    homedir: vi.fn(() => '/mock/home'),
  };
});
```

**Function Mocking:**
```typescript
const mockFn = vi.fn();
mockFn.mockResolvedValue({ data: 'test' });
```

**Spying:**
```typescript
const spy = vi.spyOn(object, 'method');
spy.mockImplementation(() => 'mocked');
```

### React/Ink Component Testing

```typescript
import { render } from 'ink-testing-library';
import { ChatMessage } from './ChatMessage';

it('renders message content', () => {
  const { lastFrame } = render(
    <ChatMessage content="Hello" role="user" />
  );
  expect(lastFrame()).toContain('Hello');
});
```

## Debugging

### VS Code Configuration

The project includes VS Code launch configurations in `.vscode/launch.json`:

```json
{
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug CLI",
      "program": "${workspaceFolder}/scripts/start.js",
      "env": {
        "DEBUG": "1"
      }
    }
  ]
}
```

### Debug Mode

```bash
# Start with Node.js inspector
npm run debug

# Then attach debugger in VS Code or Chrome DevTools
```

### Logging

Enable debug logging:

```bash
DEBUG=1 npm run start
```

## Preflight Check

The preflight check is a comprehensive validation that must pass before submitting changes:

```bash
npm run preflight
```

This runs:
1. `npm run clean` - Clean previous builds
2. `npm ci` - Fresh dependency install
3. `npm run format` - Code formatting
4. `npm run build` - Full build
5. `npm run lint:ci` - Linting
6. `npm run typecheck` - Type checking
7. `npm run test:ci` - All tests

### Why Preflight?

- Ensures code quality before commits
- Catches issues early in development
- Validates full build chain
- Required for CI/CD success

## Code Style Guidelines

### TypeScript Preferences

From `GEMINI.md`:

1. **Prefer plain objects over classes**
   ```typescript
   // Preferred
   interface User {
     name: string;
     email: string;
   }
   
   // Instead of
   class User {
     constructor(public name: string, public email: string) {}
   }
   ```

2. **Use ES module exports for encapsulation**
   ```typescript
   // Private to module
   function helper() { }
   
   // Public API
   export function publicFunction() {
     return helper();
   }
   ```

3. **Prefer `unknown` over `any`**
   ```typescript
   // Preferred
   function process(value: unknown) {
     if (typeof value === 'string') {
       return value.toUpperCase();
     }
   }
   
   // Avoid
   function process(value: any) {
     return value.toUpperCase(); // Runtime error risk
   }
   ```

### Linting Rules

ESLint is configured with:
- TypeScript-ESLint rules
- React/React Hooks rules
- Import ordering
- Prettier integration

## Common Development Tasks

### Adding a New Tool

1. Create tool file in `packages/core/src/tools/`
2. Extend `BaseDeclarativeTool`
3. Register in `ToolRegistry`
4. Add tests

### Adding a CLI Command

1. Create command in `packages/cli/src/commands/`
2. Export as `CommandModule` from yargs
3. Register in command router

### Adding a Dependency

```bash
# Add to specific workspace
npm install lodash -w @google/gemini-cli-core

# Add as dev dependency
npm install -D @types/lodash -w @google/gemini-cli-core
```

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Module not found | Run `npm run build` |
| Type errors | Run `npm run typecheck` |
| Test failures | Check mock setup |
| Build failures | Try `npm run clean && npm ci` |

### Clean Rebuild

```bash
npm run clean
npm ci
npm run build
```
