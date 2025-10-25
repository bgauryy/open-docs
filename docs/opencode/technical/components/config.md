# Configuration

**Path**: `packages/opencode/src/config`
**Type**: Configuration
**File Count**: 2

## Description

Configuration management for OpenCode settings.

## Purpose

The config component manages all configuration for OpenCode, including user preferences, AI provider settings, tool configuration, and project-specific settings.

## Key Features

- Hierarchical configuration (global, project, command-line)
- JSON-based configuration files
- Environment variable support
- Configuration validation with Zod
- Default values
- Configuration merging

## Component Files

- `index.ts` - Configuration loader and manager
- `schema.ts` - Configuration schema validation

## Configuration Hierarchy

Configuration is loaded in order of precedence (highest to lowest):

1. Command-line arguments
2. Environment variables
3. Project config (`.opencode/config.json`)
4. User config (`~/.opencode/config.json`)
5. Default values

## Configuration File Format

### Global Config (`~/.opencode/config.json`)

```json
{
  "defaultModel": "claude-3-sonnet",
  "provider": {
    "anthropic": {
      "apiKey": "${ANTHROPIC_API_KEY}"
    },
    "openai": {
      "apiKey": "${OPENAI_API_KEY}"
    }
  },
  "tools": {
    "enabled": ["bash", "edit", "read", "write", "grep", "glob"],
    "bash": {
      "timeout": 120000,
      "allowDangerous": false
    }
  },
  "session": {
    "saveHistory": true,
    "maxHistorySize": 100
  }
}
```

### Project Config (`.opencode/config.json`)

```json
{
  "defaultModel": "claude-3-opus",
  "tools": {
    "enabled": ["bash", "edit", "read", "write"],
    "bash": {
      "workingDir": "./workspace",
      "env": {
        "NODE_ENV": "development"
      }
    }
  },
  "ignore": [
    "node_modules",
    ".git",
    "dist"
  ]
}
```

## External Dependencies

- `zod` - Schema validation

## Usage

### CLI Configuration

```bash
# View current configuration
opencode config get

# Set configuration value
opencode config set defaultModel claude-3-opus

# Get specific value
opencode config get defaultModel

# Reset to defaults
opencode config reset

# Edit config in editor
opencode config edit
```

### Programmatic Configuration

```typescript
import { Config } from './config';

// Load configuration
const config = await Config.load();

// Get value
const model = config.get('defaultModel');

// Set value
config.set('defaultModel', 'claude-3-opus');

// Save configuration
await config.save();

// Merge configuration
const merged = Config.merge(userConfig, projectConfig, cliArgs);
```

## Configuration Schema

```typescript
interface OpenCodeConfig {
  // AI Provider Settings
  defaultModel: string;
  provider: {
    anthropic?: {
      apiKey: string;
      baseUrl?: string;
    };
    openai?: {
      apiKey: string;
      baseUrl?: string;
    };
    google?: {
      apiKey: string;
      project?: string;
    };
  };

  // Tool Configuration
  tools: {
    enabled: string[];
    [toolName: string]: ToolConfig;
  };

  // Session Settings
  session: {
    saveHistory: boolean;
    maxHistorySize: number;
    autoSave: boolean;
  };

  // File System
  ignore: string[];
  include: string[];

  // Server Settings
  server?: {
    port: number;
    host: string;
    auth: boolean;
  };
}
```

## Environment Variables

OpenCode supports environment variable substitution:

```json
{
  "provider": {
    "anthropic": {
      "apiKey": "${ANTHROPIC_API_KEY}"
    }
  }
}
```

Supported environment variables:
- `ANTHROPIC_API_KEY` - Anthropic API key
- `OPENAI_API_KEY` - OpenAI API key
- `GOOGLE_API_KEY` - Google API key
- `OPENCODE_MODEL` - Default model
- `OPENCODE_CONFIG` - Custom config file path

## Validation

Configuration is validated using Zod schemas:

```typescript
import { z } from 'zod';

const configSchema = z.object({
  defaultModel: z.string(),
  provider: z.object({
    anthropic: z.object({
      apiKey: z.string(),
      baseUrl: z.string().optional()
    }).optional()
  }),
  // ... more schema definitions
});

// Validate configuration
const validated = configSchema.parse(rawConfig);
```

## Default Configuration

```typescript
const defaultConfig = {
  defaultModel: 'claude-3-sonnet',
  tools: {
    enabled: ['bash', 'edit', 'read', 'write', 'grep', 'glob'],
    bash: {
      timeout: 120000,
      allowDangerous: false
    }
  },
  session: {
    saveHistory: true,
    maxHistorySize: 100,
    autoSave: true
  },
  ignore: [
    'node_modules',
    '.git',
    'dist',
    'build',
    '.next',
    '.turbo'
  ]
};
```

## Configuration Examples

### Minimal Configuration
```json
{
  "defaultModel": "claude-3-sonnet"
}
```

### Development Configuration
```json
{
  "defaultModel": "claude-3-opus",
  "tools": {
    "enabled": ["bash", "edit", "read", "write", "grep", "glob", "lsp_diagnostics"],
    "bash": {
      "timeout": 300000,
      "workingDir": "./src"
    }
  },
  "session": {
    "saveHistory": true,
    "maxHistorySize": 200
  }
}
```

### Production Configuration
```json
{
  "defaultModel": "claude-3-sonnet",
  "tools": {
    "enabled": ["read", "grep", "glob"],
    "bash": {
      "timeout": 60000,
      "allowDangerous": false
    }
  },
  "session": {
    "saveHistory": false
  }
}
```

## Related Documentation

- [CLI Bootstrap](./cli.md#bootstrap-process)
- [Overview](../01-overview.md)
