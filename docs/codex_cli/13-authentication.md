# Codex CLI - Authentication (Implementation Details)

> **📚 Official User Guide**: For login instructions and setup, see [Official authentication.md](../../context/codex/docs/authentication.md)
>
> **🎯 This Document**: Focuses on internal authentication implementation, OAuth2 flow details, and token management for developers.

---

## Quick Links

- **User Guide**: `/context/codex/docs/authentication.md` - How to log in and authenticate
- **This Doc**: Implementation details for developers
- **Related**: [08-configuration.md](./08-configuration.md) - API key configuration

---

## Table of Contents
- [Auth Modes](#auth-modes)
- [AuthManager Implementation](#authmanager-implementation)
- [Token Storage and Refresh](#token-storage-and-refresh)
- [Security Considerations](#security-considerations)

---

## Auth Modes

### Overview

Codex supports multiple authentication methods:

1. **API Key** - Direct API key (most common)
2. **ChatGPT Authentication** - OAuth2 device code flow
3. **Session Tokens** - Cached authentication tokens

### 1. API Key Authentication

**Most Common Method**

**Setup**:
```bash
export OPENAI_API_KEY="sk-..."
```

Or in config:
```yaml
providers:
  openai:
    envKey: OPENAI_API_KEY
```

**Usage** (simplified representation):
```rust
// Conceptual implementation based on core/src/auth.rs
// Actual implementation may differ - see source for details
pub fn get_api_key(provider: &str) -> Result<String> {
    let env_key = format!("{}_API_KEY", provider.to_uppercase());
    env::var(&env_key)
        .or_else(|_| env::var("OPENAI_API_KEY")) // Fallback
        .map_err(|_| CodexErr::AuthFailed(format!("No API key found")))
}
```

**📝 Note**: Code examples in this document are simplified for clarity. See `codex-rs/core/src/auth.rs` for actual implementation.

### 2. ChatGPT Authentication

**OAuth2 Device Code Flow**

**Command**:
```bash
codex login
```

**Flow**:
```
1. Request Device Code
   ├─→ POST /oauth/device/code
   └─→ Returns: device_code, user_code, verification_uri

2. Display to User
   ├─→ "Visit: https://platform.openai.com/device"
   └─→ "Enter code: ABCD-1234"

3. Poll for Token
   ├─→ POST /oauth/token (every 5 seconds)
   └─→ Returns: access_token, refresh_token, expires_in

4. Store Tokens
   └─→ Save to ~/.codex/auth.json
```

### 3. Session Tokens

**Automatic Token Refresh**

When using ChatGPT auth, tokens are automatically refreshed:
- Access token expires: 1 hour
- Refresh token expires: 7 days
- Auto-refresh when access token expires

---

## AuthManager Implementation

### AuthManager Structure

**Location**: `core/src/auth.rs`

**Simplified Representation**:
```rust
// Conceptual structure - actual implementation may vary
pub struct AuthManager {
    auth_mode: AuthMode,
    storage: AuthStorage,
    refresh_lock: Arc<Mutex<()>>,
}

pub enum AuthMode {
    ApiKey { key: String },
    ChatGpt { tokens: ChatGptTokens },
    SessionToken { token: String },
}

pub struct ChatGptTokens {
    pub access_token: String,
    pub refresh_token: String,
    pub expires_at: DateTime<Utc>,
}
```

**Actual Files**:
- `codex-rs/core/src/auth.rs` - Main authentication logic
- `codex-rs/core/src/mcp/auth.rs` - MCP-specific authentication
- `codex-rs/core/src/token_data.rs` - Token handling

### Initialization

```rust
impl AuthManager {
    pub async fn initialize(config: &Config) -> Result<Self> {
        // 1. Try loading from storage
        if let Ok(stored_auth) = AuthStorage::load().await {
            return Ok(AuthManager {
                auth_mode: stored_auth.mode,
                storage: AuthStorage::new(),
                refresh_lock: Arc::new(Mutex::new(())),
            });
        }
        
        // 2. Try API key from environment
        if let Ok(api_key) = get_api_key(&config.provider) {
            return Ok(AuthManager {
                auth_mode: AuthMode::ApiKey { key: api_key },
                storage: AuthStorage::new(),
                refresh_lock: Arc::new(Mutex::new(())),
            });
        }
        
        // 3. No authentication found
        Err(CodexErr::AuthFailed(
            "No authentication found. Run 'codex login' or set API key.".into()
        ))
    }
}
```

### Getting Authorization Header

```rust
impl AuthManager {
    pub async fn get_auth_header(&self) -> Result<String> {
        match &self.auth_mode {
            AuthMode::ApiKey { key } => {
                Ok(format!("Bearer {}", key))
            }
            
            AuthMode::ChatGpt { tokens } => {
                // Check if expired
                if tokens.expires_at < Utc::now() {
                    // Refresh token
                    let new_tokens = self.refresh_access_token(&tokens.refresh_token).await?;
                    self.update_tokens(new_tokens).await?;
                    
                    Ok(format!("Bearer {}", new_tokens.access_token))
                } else {
                    Ok(format!("Bearer {}", tokens.access_token))
                }
            }
            
            AuthMode::SessionToken { token } => {
                Ok(format!("Bearer {}", token))
            }
        }
    }
}
```

---

## Token Storage and Refresh

### Auth Storage

**Location**: `core/src/auth.rs`

```rust
pub struct AuthStorage {
    storage_path: PathBuf,
}

#[derive(Serialize, Deserialize)]
struct StoredAuth {
    mode: AuthMode,
    created_at: DateTime<Utc>,
    last_refreshed: Option<DateTime<Utc>>,
}

impl AuthStorage {
    pub fn new() -> Self {
        let storage_path = codex_home()
            .expect("Could not find codex home")
            .join("auth.json");
        
        Self { storage_path }
    }
    
    pub async fn load(&self) -> Result<StoredAuth> {
        let contents = tokio::fs::read_to_string(&self.storage_path).await?;
        let stored: StoredAuth = serde_json::from_str(&contents)?;
        Ok(stored)
    }
    
    pub async fn save(&self, auth: &StoredAuth) -> Result<()> {
        // Ensure directory exists
        if let Some(parent) = self.storage_path.parent() {
            tokio::fs::create_dir_all(parent).await?;
        }
        
        // Write with restrictive permissions
        let contents = serde_json::to_string_pretty(auth)?;
        tokio::fs::write(&self.storage_path, contents).await?;
        
        // Set file permissions to 0600 (owner read/write only)
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            let mut perms = tokio::fs::metadata(&self.storage_path).await?.permissions();
            perms.set_mode(0o600);
            tokio::fs::set_permissions(&self.storage_path, perms).await?;
        }
        
        Ok(())
    }
    
    pub async fn clear(&self) -> Result<()> {
        if self.storage_path.exists() {
            tokio::fs::remove_file(&self.storage_path).await?;
        }
        Ok(())
    }
}
```

### Token Refresh

```rust
impl AuthManager {
    async fn refresh_access_token(&self, refresh_token: &str) -> Result<ChatGptTokens> {
        // Acquire lock to prevent concurrent refreshes
        let _lock = self.refresh_lock.lock().await;
        
        // Make refresh request
        let client = reqwest::Client::new();
        let response = client
            .post("https://auth.openai.com/oauth/token")
            .json(&serde_json::json!({
                "grant_type": "refresh_token",
                "refresh_token": refresh_token,
                "client_id": CODEX_CLIENT_ID,
            }))
            .send()
            .await?;
        
        if !response.status().is_success() {
            return Err(CodexErr::AuthFailed(
                "Token refresh failed. Please login again.".into()
            ));
        }
        
        let token_response: TokenResponse = response.json().await?;
        
        Ok(ChatGptTokens {
            access_token: token_response.access_token,
            refresh_token: token_response.refresh_token
                .unwrap_or_else(|| refresh_token.to_string()),
            expires_at: Utc::now() + Duration::seconds(token_response.expires_in),
        })
    }
    
    async fn update_tokens(&self, tokens: ChatGptTokens) -> Result<()> {
        // Update in-memory
        // (Would need RefCell or similar for mutation)
        
        // Persist to disk
        let stored = StoredAuth {
            mode: AuthMode::ChatGpt { tokens },
            created_at: Utc::now(),
            last_refreshed: Some(Utc::now()),
        };
        
        self.storage.save(&stored).await?;
        
        Ok(())
    }
}
```

---

## Login Commands

### Login Flow

**Location**: `cli/src/login.rs`

```rust
pub async fn run_login_with_device_code() -> Result<()> {
    println!("Logging in to OpenAI...\n");
    
    // 1. Request device code
    let device_code_response = request_device_code().await?;
    
    // 2. Display instructions
    println!("Please visit: {}", device_code_response.verification_uri);
    println!("Enter code: {}\n", device_code_response.user_code);
    
    // 3. Poll for authorization
    let tokens = poll_for_authorization(
        &device_code_response.device_code,
        device_code_response.interval,
    ).await?;
    
    // 4. Save tokens
    save_auth_tokens(tokens).await?;
    
    println!("\n✓ Successfully logged in!");
    Ok(())
}

async fn request_device_code() -> Result<DeviceCodeResponse> {
    let client = reqwest::Client::new();
    let response = client
        .post("https://auth.openai.com/oauth/device/code")
        .json(&serde_json::json!({
            "client_id": CODEX_CLIENT_ID,
            "scope": "openai",
        }))
        .send()
        .await?;
    
    let device_code_response: DeviceCodeResponse = response.json().await?;
    Ok(device_code_response)
}

async fn poll_for_authorization(
    device_code: &str,
    interval: u64,
) -> Result<ChatGptTokens> {
    let client = reqwest::Client::new();
    let mut attempts = 0;
    let max_attempts = 120; // 10 minutes
    
    loop {
        attempts += 1;
        if attempts > max_attempts {
            return Err(CodexErr::AuthFailed("Timeout waiting for authorization".into()));
        }
        
        tokio::time::sleep(Duration::from_secs(interval)).await;
        
        let response = client
            .post("https://auth.openai.com/oauth/token")
            .json(&serde_json::json!({
                "grant_type": "urn:ietf:params:oauth:grant-type:device_code",
                "device_code": device_code,
                "client_id": CODEX_CLIENT_ID,
            }))
            .send()
            .await?;
        
        match response.status().as_u16() {
            200 => {
                // Success!
                let token_response: TokenResponse = response.json().await?;
                return Ok(ChatGptTokens {
                    access_token: token_response.access_token,
                    refresh_token: token_response.refresh_token.unwrap(),
                    expires_at: Utc::now() + Duration::seconds(token_response.expires_in),
                });
            }
            400 => {
                // Check error type
                let error: OAuth2Error = response.json().await?;
                match error.error.as_str() {
                    "authorization_pending" => {
                        // Still waiting
                        print!(".");
                        io::stdout().flush()?;
                        continue;
                    }
                    "slow_down" => {
                        // Increase polling interval
                        tokio::time::sleep(Duration::from_secs(5)).await;
                        continue;
                    }
                    _ => {
                        return Err(CodexErr::AuthFailed(error.error_description));
                    }
                }
            }
            _ => {
                return Err(CodexErr::AuthFailed("Unexpected response".into()));
            }
        }
    }
}
```

### Login with API Key

```rust
pub async fn run_login_with_api_key(api_key: Option<String>) -> Result<()> {
    let key = if let Some(k) = api_key {
        k
    } else {
        // Read from stdin
        print!("Enter your OpenAI API key: ");
        io::stdout().flush()?;
        read_api_key_from_stdin()?
    };
    
    // Validate API key format
    if !key.starts_with("sk-") && !key.starts_with("sess-") {
        return Err(CodexErr::AuthFailed("Invalid API key format".into()));
    }
    
    // Test API key
    test_api_key(&key).await?;
    
    // Save
    let stored = StoredAuth {
        mode: AuthMode::ApiKey { key },
        created_at: Utc::now(),
        last_refreshed: None,
    };
    
    AuthStorage::new().save(&stored).await?;
    
    println!("✓ API key saved successfully!");
    Ok(())
}

async fn test_api_key(key: &str) -> Result<()> {
    let client = reqwest::Client::new();
    let response = client
        .get("https://api.openai.com/v1/models")
        .header("Authorization", format!("Bearer {}", key))
        .send()
        .await?;
    
    if !response.status().is_success() {
        return Err(CodexErr::AuthFailed("Invalid API key".into()));
    }
    
    Ok(())
}
```

### Logout

```rust
pub async fn run_logout() -> Result<()> {
    AuthStorage::new().clear().await?;
    println!("✓ Logged out successfully");
    Ok(())
}
```

### Login Status

```rust
pub async fn run_login_status() -> Result<()> {
    let storage = AuthStorage::new();
    
    match storage.load().await {
        Ok(stored) => {
            println!("Authentication Status: Logged in");
            
            match stored.mode {
                AuthMode::ApiKey { .. } => {
                    println!("Method: API Key");
                }
                AuthMode::ChatGpt { tokens } => {
                    println!("Method: ChatGPT OAuth");
                    println!("Expires: {}", tokens.expires_at.format("%Y-%m-%d %H:%M:%S"));
                    
                    if tokens.expires_at < Utc::now() {
                        println!("Status: Expired (will be refreshed automatically)");
                    } else {
                        println!("Status: Valid");
                    }
                }
                AuthMode::SessionToken { .. } => {
                    println!("Method: Session Token");
                }
            }
            
            println!("Last updated: {}", stored.created_at.format("%Y-%m-%d %H:%M:%S"));
        }
        Err(_) => {
            println!("Authentication Status: Not logged in");
            println!("\nTo login:");
            println!("  codex login                  # OAuth flow");
            println!("  codex login --api-key <KEY>  # API key");
            println!("  export OPENAI_API_KEY=<KEY>  # Environment variable");
        }
    }
    
    Ok(())
}
```

---

## Security Considerations

### File Permissions

Auth tokens stored with restrictive permissions:

```rust
#[cfg(unix)]
{
    use std::os::unix::fs::PermissionsExt;
    let mut perms = metadata.permissions();
    perms.set_mode(0o600); // Owner read/write only
    fs::set_permissions(path, perms)?;
}
```

### Token Exposure

Never log or display tokens:

```rust
impl fmt::Debug for ChatGptTokens {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("ChatGptTokens")
            .field("access_token", &"<redacted>")
            .field("refresh_token", &"<redacted>")
            .field("expires_at", &self.expires_at)
            .finish()
    }
}
```

### Environment Variables

Clear sensitive env vars after reading:

```rust
pub fn get_api_key_and_clear(provider: &str) -> Result<String> {
    let env_key = format!("{}_API_KEY", provider.to_uppercase());
    let key = env::var(&env_key)?;
    
    // Clear from environment
    env::remove_var(&env_key);
    
    Ok(key)
}
```

### HTTPS Only

All authentication requests use HTTPS:

```rust
const AUTH_BASE_URL: &str = "https://auth.openai.com";
const API_BASE_URL: &str = "https://api.openai.com/v1";

// Never allow insecure connections for auth
let client = reqwest::Client::builder()
    .https_only(true)
    .build()?;
```

---

## Best Practices

### For Users

1. **Prefer Environment Variables**: Set `OPENAI_API_KEY` per-session
2. **Use OAuth for Personal Use**: More secure than long-lived API keys
3. **Rotate Keys**: Change API keys periodically
4. **Check Permissions**: Verify `~/.codex/auth.json` is `0600`
5. **Logout on Shared Systems**: Run `codex logout` when done

### For Developers

1. **Never Log Tokens**: Redact in debug output
2. **Use HTTPS**: All auth requests over secure connections
3. **Validate Tokens**: Test before saving
4. **Handle Expiry**: Implement automatic refresh
5. **Secure Storage**: Restrictive file permissions

---

## Troubleshooting

### "No API key found"

**Solution**:
```bash
# Set environment variable
export OPENAI_API_KEY="sk-..."

# Or login
codex login

# Or specify in config
echo "providers:" >> ~/.codex/config.yaml
echo "  openai:" >> ~/.codex/config.yaml
echo "    envKey: OPENAI_API_KEY" >> ~/.codex/config.yaml
```

### "Token refresh failed"

**Solution**:
```bash
# Re-authenticate
codex logout
codex login
```

### "Invalid API key"

**Solution**:
```bash
# Verify key format (should start with sk- or sess-)
echo $OPENAI_API_KEY

# Test key manually
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

---

## Related Documentation

- [04-llm-integration.md](./04-llm-integration.md) - API communication
- [08-configuration.md](./08-configuration.md) - Configuration options
- [15-code-reference.md](./15-code-reference.md) - Code reference

