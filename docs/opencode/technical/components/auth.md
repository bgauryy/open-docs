# Authentication

**Path**: `packages/opencode/src/auth`
**Type**: Authentication
**File Count**: 2

## Description

Authentication system with GitHub Copilot integration.

## Purpose

The auth component handles user authentication for OpenCode. It integrates with GitHub Copilot for authentication and provides token-based auth for server mode.

## Key Features

- GitHub Copilot authentication
- Token-based authentication
- OAuth integration via OpenAuth
- Session token management
- Multi-provider support

## Component Files

- `index.ts` - Main authentication logic
- `copilot.ts` - GitHub Copilot integration

## External Dependencies

- `@openauthjs/openauth` - OAuth framework

## Usage

### CLI Authentication

```bash
# Login with GitHub Copilot
opencode auth login

# Logout
opencode auth logout

# List auth providers
opencode auth list
```

### Programmatic Authentication

```typescript
import { Auth } from './auth';

// Authenticate with GitHub Copilot
const token = await Auth.loginCopilot();

// Verify token
const isValid = await Auth.verifyToken(token);

// Get current user
const user = await Auth.getCurrentUser();
```

## Authentication Flow

1. User initiates login via CLI
2. Opens GitHub Copilot auth flow
3. User authorizes in browser
4. Token returned to CLI
5. Token stored securely
6. Token used for API requests

## Token Storage

Tokens are stored securely in:
- macOS: Keychain
- Linux: Secret Service / libsecret
- Windows: Credential Manager

## Server Mode Authentication

When running in server mode, authentication is required:

```bash
# Start server with auth
opencode serve --auth

# Clients must provide token
curl -H "Authorization: Bearer TOKEN" http://localhost:3000/api/session
```

## Related Documentation

- [Auth Command](../api-reference.md#authcommand)
- [Server Mode](../flows/serve-command-flow.md)
