# File Operations

**Path**: `packages/opencode/src/file`
**Type**: Data Layer
**File Count**: 6

## Description

File operations including fzf, ripgrep, and file watching.

## Purpose

The file component provides high-performance file operations for searching, finding, and monitoring files. It integrates with external tools like ripgrep and fzf for optimal performance.

## Key Features

- Fast file search with ripgrep
- Fuzzy file finding with fzf
- File system watching
- Glob pattern matching
- Content search
- Change detection

## Component Files

- `index.ts` - Main file operations API
- `ripgrep.ts` - Ripgrep integration
- `fzf.ts` - Fuzzy finder integration
- `watcher.ts` - File system watcher
- `glob.ts` - Glob pattern matching
- `utils.ts` - File utilities

## Dependencies

### Internal Dependencies
- Used by `packages/opencode/src/tool` (6 imports)

### External Dependencies
- `@parcel/watcher` - High-performance file watching
- `chokidar` - Fallback file watcher

## Usage

### Ripgrep Search

```typescript
import { File } from './file';

// Search for pattern
const matches = await File.ripgrep('TODO', {
  path: './src',
  type: 'ts',
  caseInsensitive: true
});

for (const match of matches) {
  console.log(`${match.file}:${match.line}: ${match.text}`);
}
```

### Fuzzy File Finding

```typescript
import { File } from './file';

// Fuzzy find files
const files = await File.fzf('index');
console.log('Matching files:', files);

// With directory filter
const srcFiles = await File.fzf('component', { cwd: './src' });
```

### File Watching

```typescript
import { File } from './file';

// Watch directory for changes
const watcher = File.watch('./src', (event) => {
  console.log(`${event.type}: ${event.path}`);

  if (event.type === 'update') {
    console.log('File modified:', event.path);
  } else if (event.type === 'create') {
    console.log('File created:', event.path);
  } else if (event.type === 'delete') {
    console.log('File deleted:', event.path);
  }
});

// Stop watching
watcher.close();
```

### Glob Pattern Matching

```typescript
import { File } from './file';

// Find TypeScript files
const tsFiles = await File.glob('**/*.ts', {
  ignore: ['node_modules/**', 'dist/**']
});

// Find test files
const testFiles = await File.glob('**/*.test.ts');
```

## Ripgrep Options

```typescript
interface RipgrepOptions {
  path?: string;              // Search path
  type?: string;              // File type (ts, js, py, etc.)
  caseInsensitive?: boolean;  // Case-insensitive search
  fixed?: boolean;            // Fixed string (not regex)
  maxMatches?: number;        // Max matches per file
  context?: number;           // Lines of context
  hidden?: boolean;           // Search hidden files
  followSymlinks?: boolean;   // Follow symlinks
}
```

## File Watcher Events

```typescript
interface WatchEvent {
  type: 'create' | 'update' | 'delete';
  path: string;
  timestamp: number;
}
```

## Performance Considerations

### Ripgrep
- Fastest search tool (faster than grep, ack, ag)
- Respects .gitignore by default
- Parallel search across cores
- Memory-efficient streaming

### File Watcher
- Uses native OS APIs (@parcel/watcher)
- Fallback to chokidar if needed
- Efficient change detection
- Batch change notifications

### Glob
- Fast pattern matching
- Supports negation patterns
- Directory traversal optimization

## Common Patterns

### Search and Read
```typescript
// Find files with pattern
const matches = await File.ripgrep('export function', {
  path: './src',
  type: 'ts'
});

// Read matching files
for (const match of matches) {
  const content = await File.read(match.file);
  console.log(content);
}
```

### Watch and Process
```typescript
const watcher = File.watch('./src', async (event) => {
  if (event.type === 'update' && event.path.endsWith('.ts')) {
    // Recompile TypeScript
    await compile(event.path);
  }
});
```

### Search with Context
```typescript
const matches = await File.ripgrep('TODO', {
  context: 3,  // 3 lines before and after
  path: './src'
});

for (const match of matches) {
  console.log('---');
  console.log(match.beforeContext.join('\n'));
  console.log('>>>', match.text);
  console.log(match.afterContext.join('\n'));
}
```

## Integration with Tools

The file component is used by multiple tools:

- **GrepTool** - Uses `File.ripgrep()`
- **GlobTool** - Uses `File.glob()`
- **ReadTool** - Uses `File.read()`
- **WriteTool** - Uses `File.write()`
- **WatchTool** - Uses `File.watch()`

## Error Handling

```typescript
try {
  const matches = await File.ripgrep('pattern', {
    path: './nonexistent'
  });
} catch (error) {
  if (error.code === 'ENOENT') {
    console.error('Directory not found');
  } else if (error.code === 'RIPGREP_NOT_FOUND') {
    console.error('ripgrep not installed');
  }
}
```

## Related Documentation

- [Tool Execution Flow](../flows/tool-execution-flow.md)
- [Grep Tool API](../api-reference.md#greptool)
- [Glob Tool API](../api-reference.md#globtool)
- [File Operations API](../api-reference.md#file-operations)
