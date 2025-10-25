# OpenCode - Authentication

> **Provider authentication and token management**

---

## Overview

OpenCode handles authentication for multiple AI providers:
- **API key management** - Secure storage and retrieval
- **GitHub Copilot auth** - Reuse Copilot tokens
- **OAuth flows** - For supported providers
- **Token validation** - Check credentials
- **Credential storage** - Secure local storage

**Files**:
- `auth/index.ts` - Main auth module
- `auth/github-copilot.ts` - GitHub Copilot integration
- `cli/cmd/auth.ts` - Auth command

---

## Provider Authentication

### Environment Variables

**Primary method** - Set API keys as environment variables:

```bash
# Anthropic (Claude)
export ANTHROPIC_API_KEY="sk-ant-..."

# OpenAI (GPT)
export OPENAI_API_KEY="sk-..."

# Google (Gemini)
export GOOGLE_API_KEY="..."

# AWS Bedrock
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
```

Add to shell profile (`~/.zshrc`, `~/.bashrc`):
```bash
# OpenCode API Keys
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."
```

---

## GitHub Copilot Authentication

**Reuse existing Copilot tokens** for Anthropic/OpenAI:

### Setup

```bash
# Check if Copilot token is available
opencode auth status

# Output:
# GitHub Copilot: ✓ Authenticated
# Anthropic: ✓ (via Copilot)
# OpenAI: ✓ (via Copilot)
```

### How It Works

```typescript
// From auth/github-copilot.ts
export async function getCopilotToken(): Promise<string | null> {
  // Read from GitHub CLI config
  const ghConfig = await Bun.file(
    path.join(os.homedir(), ".config/github-copilot/hosts.json")
  ).json()
  
  return ghConfig.token
}

export async function getAnthropicToken(): Promise<string | null> {
  const copilotToken = await getCopilotToken()
  if (!copilotToken) return null
  
  // Exchange for Anthropic token
  const response = await fetch("https://api.githubcopilot.com/token", {
    headers: {
      "Authorization": `Bearer ${copilotToken}`,
      "Editor-Version": "vscode/1.85.0"
    }
  })
  
  const data = await response.json()
  return data.token
}
```

**Advantages**:
- No additional API keys needed
- Reuses existing Copilot subscription
- Automatic token refresh

**Limitations**:
- Requires active Copilot subscription
- May have usage limits
- Not all providers supported

---

## Auth Command

### Check Status

```bash
opencode auth status
```

Output:
```
Authentication Status:

Anthropic:
  ✓ Authenticated via ANTHROPIC_API_KEY
  User: user@example.com
  
OpenAI:
  ✓ Authenticated via GitHub Copilot
  
Google:
  ✗ Not authenticated
  Run: export GOOGLE_API_KEY="your-key"
  
AWS Bedrock:
  ✗ Not authenticated
  Configure AWS credentials
```

### Login

```bash
# Provider-specific login
opencode auth login anthropic

# Opens browser for OAuth or prompts for API key
```

### Logout

```bash
# Remove stored credentials
opencode auth logout anthropic
```

---

## Credential Storage

### Secure Storage

**Location**: `~/.opencode/credentials.json`

```json
{
  "anthropic": {
    "apiKey": "sk-ant-...",
    "source": "manual"
  },
  "openai": {
    "apiKey": "sk-...",
    "source": "github-copilot",
    "expiresAt": 1234567890
  }
}
```

**Security**:
- File permissions: 0600 (owner read/write only)
- Encrypted at rest (macOS Keychain, Linux Secret Service)
- Never logged or transmitted except to provider APIs

### Access Credentials

```typescript
import { Auth } from "./auth"

// Get all credentials
const creds = await Auth.all()

// Get specific provider
const anthropicKey = await Auth.get("anthropic")

// Set credential
await Auth.set("anthropic", {
  apiKey: "sk-ant-...",
  source: "manual"
})

// Remove credential
await Auth.remove("anthropic")
```

---

## Token Validation

### Check Validity

```typescript
async function validateToken(
  provider: string,
  apiKey: string
): Promise<boolean> {
  try {
    // Make test API call
    const response = await fetch(`${provider}/v1/models`, {
      headers: { "Authorization": `Bearer ${apiKey}` }
    })
    
    return response.ok
  } catch {
    return false
  }
}
```

### Auto-Refresh

For providers supporting token refresh:

```typescript
async function getValidToken(provider: string): Promise<string> {
  const cred = await Auth.get(provider)
  
  // Check expiration
  if (cred.expiresAt && Date.now() > cred.expiresAt) {
    // Refresh token
    const newToken = await refreshToken(provider, cred.refreshToken)
    await Auth.set(provider, {
      ...cred,
      apiKey: newToken.access_token,
      expiresAt: Date.now() + newToken.expires_in * 1000
    })
    return newToken.access_token
  }
  
  return cred.apiKey
}
```

---

## Multi-Account Support

### Named Profiles

```bash
# Set profile
export OPENCODE_PROFILE="work"

# Different credentials per profile
~/.opencode/profiles/
├── work/
│   └── credentials.json
├── personal/
│   └── credentials.json
└── client/
    └── credentials.json
```

**Usage**:
```bash
# Work profile
OPENCODE_PROFILE=work opencode "Task"

# Personal profile
OPENCODE_PROFILE=personal opencode "Task"
```

---

## Best Practices

**Security**:
- Never commit API keys to Git
- Use environment variables
- Set restrictive file permissions
- Rotate keys periodically
- Monitor usage for anomalies

**Organization**:
- Use separate keys per project
- Document key sources
- Set up billing alerts
- Track usage per key

**Development**:
- Use test/sandbox keys
- Don't share keys
- Revoke compromised keys immediately
- Use short-lived tokens when possible

**Production**:
- Use secrets management (AWS Secrets Manager, Vault)
- Implement key rotation
- Monitor API usage
- Set rate limits

---

## Troubleshooting

### "No API key found"

```bash
# Check environment variables
env | grep API_KEY

# Check stored credentials
opencode auth status

# Set manually
export ANTHROPIC_API_KEY="sk-ant-..."
```

### "Invalid API key"

```bash
# Validate key format
# Anthropic: sk-ant-...
# OpenAI: sk-...
# Google: ...

# Test key
opencode auth test anthropic
```

### "GitHub Copilot not found"

```bash
# Check Copilot installation
gh copilot --version

# Authenticate with GitHub
gh auth login
gh copilot setup
```

---

For implementation, see `packages/opencode/src/auth/`.

