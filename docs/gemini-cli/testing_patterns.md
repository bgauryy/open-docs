# Gemini CLI - Testing Patterns & Edge Cases Reference

**Generated:** 2024-10-24
**Purpose:** Documentation of test patterns, integration tests, and edge case handling

---

## Table of Contents

1. [Testing Overview](#testing-overview)
2. [Test Framework](#test-framework)
3. [Unit Testing Patterns](#unit-testing-patterns)
4. [Integration Testing](#integration-testing)
5. [Test Utilities](#test-utilities)
6. [Edge Cases](#edge-cases)
7. [Mock Patterns](#mock-patterns)
8. [Coverage](#coverage)

---

## Testing Overview

### Test Distribution

**Total Test Files:** 100+ test files across packages

**Test Frameworks:**
- **Vitest** - Primary test runner (all packages)
- **Node Test Runner** - Integration tests

**Test Types:**
1. **Unit Tests** - Individual functions/classes
2. **Integration Tests** - End-to-end scenarios
3. **Snapshot Tests** - UI component snapshots
4. **Property Tests** - Edge case generation

---

### Running Tests

```bash
# Root level (all packages)
npm test

# Specific package
cd packages/core
npm test

# Watch mode
npm run test:watch

# Coverage
npm run test:coverage

# Integration tests
npm run test:integration
```

---

## Test Framework

### Vitest Configuration

**Location:** `vitest.config.ts` (per package)

**Key Features:**
- TypeScript support out-of-the-box
- Fast parallel execution
- Hot module reloading
- Built-in coverage (via `@vitest/coverage-v8`)

---

### Test Structure

```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';

describe('MyComponent', () => {
  beforeEach(() => {
    // Setup
  });
  
  afterEach(() => {
    // Cleanup
  });
  
  it('should behave correctly', () => {
    // Arrange
    const input = ...;
    
    // Act
    const result = myFunction(input);
    
    // Assert
    expect(result).toBe(expected);
  });
});
```

---

## Unit Testing Patterns

### 1. Tool Testing

**Pattern:** Test tool parameter validation, execution, and error handling

**Example:** `packages/core/src/tools/write-file.test.ts`

```typescript
describe('WriteFileTool', () => {
  it('validates file path', async () => {
    const tool = new WriteFileTool(config);
    await expect(tool.call({
      file_path: '../../../etc/passwd',  // Path traversal attempt
      content: 'malicious'
    })).rejects.toThrow();
  });
  
  it('writes file successfully', async () => {
    const tool = new WriteFileTool(config);
    const result = await tool.call({
      file_path: '/tmp/test.txt',
      content: 'Hello, World!'
    });
    
    expect(result.error).toBeUndefined();
    expect(fs.existsSync('/tmp/test.txt')).toBe(true);
  });
});
```

---

### 2. Configuration Testing

**Pattern:** Test settings loading, merging, and validation

**Example:** `packages/cli/src/config/settings.test.ts`

```typescript
describe('Settings', () => {
  it('merges user and project settings', () => {
    const userSettings = { model: { name: 'gemini-1.5-pro' } };
    const projectSettings = { tools: { sandbox: true } };
    
    const merged = mergeSettings(userSettings, projectSettings);
    
    expect(merged.model.name).toBe('gemini-1.5-pro');
    expect(merged.tools.sandbox).toBe(true);
  });
  
  it('validates schema', () => {
    expect(() => {
      validateSettings({ invalid: 'setting' });
    }).toThrow();
  });
});
```

---

### 3. Command Testing

**Pattern:** Test slash commands, arguments, and completion

**Example:** `packages/cli/src/ui/commands/chatCommand.test.ts`

```typescript
describe('ChatCommand', () => {
  it('starts new chat', async () => {
    const context = createMockContext();
    await chatCommand.action(context, '');
    
    expect(context.ui.clearHistory).toHaveBeenCalled();
  });
});
```

---

## Integration Testing

### Integration Test Setup

**Location:** `integration-tests/`

**Purpose:** End-to-end tests of Gemini CLI

**Test Helper:** `integration-tests/test-helper.ts`

---

### Test Helper Utilities

```typescript
export async function runGeminiCLI(
  args: string[],
  options: {
    input?: string;
    env?: Record<string, string>;
    cwd?: string;
    timeout?: number;
  } = {}
): Promise<{
  stdout: string;
  stderr: string;
  exitCode: number;
}> {
  // Implementation
}
```

---

### Integration Test Patterns

#### 1. File Operations

**Test:** `integration-tests/file-system.test.ts`

```typescript
it('reads and writes files', async () => {
  const result = await runGeminiCLI(
    ['chat'],
    {
      input: 'Write "Hello" to test.txt',
      cwd: testDir
    }
  );
  
  expect(fs.readFileSync(`${testDir}/test.txt`, 'utf-8')).toBe('Hello');
});
```

---

#### 2. MCP Server Integration

**Test:** `integration-tests/simple-mcp-server.test.ts`

```typescript
it('connects to MCP server', async () => {
  const mcpServer = await startTestMCPServer();
  
  const result = await runGeminiCLI(
    ['chat'],
    {
      env: {
        MCP_SERVER_URL: mcpServer.url
      },
      input: 'Call test_tool'
    }
  );
  
  expect(result.stdout).toContain('Tool called successfully');
  
  await mcpServer.stop();
});
```

---

#### 3. Extension Installation

**Test:** `integration-tests/extensions-install.test.ts`

```typescript
it('installs extension from GitHub', async () => {
  const result = await runGeminiCLI(
    ['extensions', 'install', 'github:user/repo']
  );
  
  expect(result.exitCode).toBe(0);
  expect(result.stdout).toContain('Extension installed');
});
```

---

### Test MCP Server

**Location:** `integration-tests/test-mcp-server.ts`

**Purpose:** Minimal MCP server for integration tests

**Features:**
- Implements MCP protocol
- Provides test tools
- Configurable responses
- Easy start/stop

---

## Test Utilities

### Mock Factories

**Location:** `packages/core/src/test-utils/`

#### `createMockConfig()`

```typescript
export function createMockConfig(overrides?: Partial<Config>): Config {
  return {
    getTargetDir: vi.fn(() => '/test/dir'),
    getGeminiClient: vi.fn(() => mockGeminiClient),
    getToolRegistry: vi.fn(() => mockToolRegistry),
    ...overrides
  };
}
```

---

#### `createMockGeminiClient()`

```typescript
export function createMockGeminiClient(): GeminiClient {
  return {
    generateContent: vi.fn().mockResolvedValue({
      candidates: [{
        content: {
          parts: [{ text: 'Mock response' }]
        }
      }]
    }),
    // ... other methods
  };
}
```

---

### Test Fixtures

**Location:** `integration-tests/fixtures/`

**Common Fixtures:**

- **Test Projects** - Sample codebases for testing
- **Config Files** - Sample configurations
- **MCP Server Configs** - Test MCP server setups
- **Extension Manifests** - Sample extension files

---

### Snapshot Testing

**Used for:** UI components (React/Ink)

**Example:** `packages/cli/src/ui/components/ThemeDialog.test.tsx.snap`

```typescript
it('renders theme dialog', () => {
  const { lastFrame } = render(<ThemeDialog themes={mockThemes} />);
  expect(lastFrame()).toMatchSnapshot();
});
```

---

## Edge Cases

### 1. File System Edge Cases

#### Binary Files

**Test:** Verify tools handle binary files correctly

```typescript
it('rejects writing binary content as text', async () => {
  const binaryContent = Buffer.from([0x00, 0x01, 0x02]);
  await expect(writeFile('/test.bin', binaryContent.toString())).rejects.toThrow();
});
```

---

#### Path Traversal

**Test:** Prevent directory traversal attacks

```typescript
it('blocks path traversal', async () => {
  await expect(readFile('../../etc/passwd')).rejects.toThrow('Path traversal');
});
```

---

#### Special Characters in Filenames

**Test:** Handle Unicode, spaces, special chars

```typescript
it('handles special filenames', async () => {
  const filename = 'test file (1) [copy].txt';
  await writeFile(filename, 'content');
  expect(await readFile(filename)).toBe('content');
});
```

---

### 2. MCP Server Edge Cases

#### Server Disconnection

**Test:** `packages/core/src/tools/mcp-client.test.ts`

```typescript
it('handles server disconnect gracefully', async () => {
  const client = await connectMCPServer(config);
  await server.stop();  // Simulate disconnect
  
  await expect(client.callTool('test')).rejects.toThrow('Server disconnected');
});
```

---

#### Circular Schema References

**Test:** `integration-tests/mcp_server_cyclic_schema.test.ts`

```typescript
it('handles cyclic schema', async () => {
  const schema = {
    type: 'object',
    properties: {
      self: { $ref: '#' }  // Circular reference
    }
  };
  
  // Should not cause infinite loop
  const validated = validateSchema(schema);
  expect(validated).toBeDefined();
});
```

---

### 3. Compression Edge Cases

**Test:** `integration-tests/context-compress-interactive.test.ts`

#### Empty Conversation

```typescript
it('handles empty chat compression', async () => {
  const result = await compressChat([]);
  expect(result).toEqual([]);
});
```

---

#### Compression Failure

**Test:** `context-compress-interactive.compress-failure.json`

```typescript
it('recovers from compression failure', async () => {
  mockGeminiClient.generateContent.mockRejectedValue(new Error('API Error'));
  
  const result = await compressChat(longHistory);
  
  // Should return original history on failure
  expect(result).toEqual(longHistory);
});
```

---

### 4. Shell Command Edge Cases

#### Ctrl+C Handling

**Test:** `integration-tests/ctrl-c-exit.test.ts`

```typescript
it('exits on Ctrl+C', async () => {
  const proc = spawn('gemini', ['chat']);
  
  setTimeout(() => proc.kill('SIGINT'), 100);
  
  const exitCode = await waitForExit(proc);
  expect(exitCode).toBe(0);  // Graceful exit
});
```

---

#### Mixed Input (stdin + flags)

**Test:** `integration-tests/mixed-input-crash.test.ts`

```typescript
it('handles mixed input sources', async () => {
  const result = await runGeminiCLI(
    ['chat', '--model', 'gemini-1.5-pro'],
    { input: 'Hello' }
  );
  
  expect(result.exitCode).toBe(0);
});
```

---

### 5. Encoding Edge Cases

#### UTF-8 BOM

**Test:** `integration-tests/utf-bom-encoding.test.ts`

```typescript
it('handles UTF-8 BOM correctly', async () => {
  const withBOM = '\uFEFFHello';
  await writeFile('test.txt', withBOM);
  
  const content = await readFile('test.txt');
  expect(content).toBe('Hello');  // BOM stripped
});
```

---

## Mock Patterns

### 1. Mocking Gemini API

```typescript
const mockGenerateContent = vi.fn().mockResolvedValue({
  candidates: [{
    content: {
      parts: [{ text: 'Mocked response' }]
    }
  }]
});

vi.mock('@google/genai', () => ({
  GoogleGenerativeAI: vi.fn(() => ({
    getGenerativeModel: vi.fn(() => ({
      generateContent: mockGenerateContent
    }))
  }))
}));
```

---

### 2. Mocking File System

```typescript
vi.mock('node:fs', () => ({
  promises: {
    readFile: vi.fn(),
    writeFile: vi.fn(),
    stat: vi.fn()
  }
}));

beforeEach(() => {
  vi.mocked(fs.promises.readFile).mockResolvedValue('file content');
});
```

---

### 3. Mocking MCP Servers

```typescript
class MockMCPServer {
  tools = new Map<string, MCPTool>();
  
  addTool(name: string, handler: Function) {
    this.tools.set(name, { name, handler });
  }
  
  async handleRequest(request: MCPRequest) {
    const tool = this.tools.get(request.method);
    return tool?.handler(request.params);
  }
}
```

---

## Coverage

### Coverage Reports

**Command:** `npm run test:coverage`

**Output:** `coverage/` directory with HTML reports

**Minimum Targets:**
- **Statements:** 80%
- **Branches:** 75%
- **Functions:** 80%
- **Lines:** 80%

---

### Excluded from Coverage

- Test files (`*.test.ts`)
- Type definitions (`*.d.ts`)
- Build artifacts (`dist/`)
- Integration test fixtures

---

### Coverage Gaps (Known)

1. **Error Paths** - Some error handling branches untested
2. **Network Errors** - Transient network failures
3. **UI Components** - Ink components difficult to test
4. **Sandbox Execution** - Platform-specific code

---

## Testing Best Practices

### 1. Arrange-Act-Assert Pattern

```typescript
it('should work', () => {
  // Arrange
  const input = createTestInput();
  const expected = calculateExpected(input);
  
  // Act
  const result = functionUnderTest(input);
  
  // Assert
  expect(result).toEqual(expected);
});
```

---

### 2. Test Naming

**Convention:** `should [behavior] when [condition]`

```typescript
it('should return error when file not found', ...);
it('should merge configs when both exist', ...);
it('should skip disabled extensions', ...);
```

---

### 3. Isolation

- Each test is independent
- No shared state between tests
- Clean up resources in `afterEach`

---

### 4. Fast Tests

- Mock external dependencies (API, FS, network)
- Use in-memory storage
- Parallel execution when possible

---

This comprehensive testing reference documents test patterns, integration tests, edge cases, and mock strategies used in the Gemini CLI test suite.

