# Codex CLI - LLM Integration

## Table of Contents
- [Model Client Architecture](#model-client-architecture)
- [Supported APIs](#supported-apis)
- [Provider Abstraction](#provider-abstraction)
- [Streaming Response Handling](#streaming-response-handling)
- [Token Management](#token-management)

---

## Model Client Architecture

### ModelClient Structure

**Location**: `core/src/client.rs`

```rust
pub struct ModelClient {
    config: Arc<Config>,
    auth_manager: Option<Arc<AuthManager>>,
    otel_event_manager: OtelEventManager,
    client: reqwest::Client,
    provider: ModelProviderInfo,
    conversation_id: ConversationId,
    effort: Option<ReasoningEffortConfig>,
    summary: ReasoningSummaryConfig,
}
```

### Key Responsibilities

1. **API Communication**: HTTP requests to LLM providers
2. **Protocol Adaptation**: Supports multiple wire protocols
3. **Streaming Management**: Handles SSE event streams
4. **Error Handling**: Retries, rate limiting, usage limits
5. **Token Tracking**: Monitors usage and context windows

### Initialization

```rust
impl ModelClient {
    pub fn new(
        config: Arc<Config>,
        auth_manager: Option<Arc<AuthManager>>,
        otel_event_manager: OtelEventManager,
        provider: ModelProviderInfo,
        effort: Option<ReasoningEffortConfig>,
        summary: ReasoningSummaryConfig,
        conversation_id: ConversationId,
    ) -> Self {
        let client = create_client(); // HTTP client with timeout
        Self { config, auth_manager, client, provider, ... }
    }
}
```

---

## Supported APIs

### Wire Protocol Abstraction

```rust
pub enum WireApi {
    Responses,  // OpenAI Responses API (preferred)
    Chat,       // OpenAI Chat Completions API
}
```

### 1. Responses API

**Endpoint**: `https://api.openai.com/v1/responses`

**Features**:
- Native streaming support
- Function calling with parallel execution
- Reasoning summaries (for o1/o3/o4 models)
- Usage tracking with cached tokens

**Request Format**:

```rust
#[derive(Serialize)]
struct ResponsesApiRequest {
    model: String,
    messages: Vec<ResponseInputItem>,
    tools: Option<Vec<Tool>>,
    stream: bool,
    reasoning_effort: Option<ReasoningEffort>,  // "low" | "medium" | "high"
    reasoning_summary: ReasoningSummary,        // "auto" | "concise" | "detailed"
}
```

**Event Stream**:

```
event: response.created
data: {"id":"resp_123","model":"o4-mini"}

event: response.item.delta
data: {"delta":{"type":"message","content":"Let me help"}}

event: response.reasoning.delta
data: {"delta":{"summary":"Analyzing the code..."}}

event: response.function_call.delta
data: {"delta":{"name":"shell","arguments":"{\"cmd\":\"ls\"}"}}

event: response.completed
data: {"id":"resp_123","usage":{"input_tokens":100,"output_tokens":50}}
```

**Implementation**:

```rust
async fn stream_responses(
    &self,
    prompt: &Prompt,
    task_kind: TaskKind,
) -> Result<ResponseStream> {
    let url = format!("{}/responses", self.provider.base_url);
    let request = build_responses_request(prompt, &self.config, task_kind);
    
    let response = self.client
        .post(&url)
        .header("Authorization", bearer_token(&self.auth_manager)?)
        .json(&request)
        .send()
        .await?;
    
    let stream = response
        .bytes_stream()
        .eventsource()
        .map(parse_responses_event);
    
    Ok(Box::pin(stream))
}
```

### 2. Chat Completions API

**Endpoint**: `https://api.openai.com/v1/chat/completions`

**Features**:
- Widely compatible (most providers support this)
- Function calling
- Streaming delta responses

**Request Format**:

```rust
#[derive(Serialize)]
struct ChatCompletionsRequest {
    model: String,
    messages: Vec<ChatMessage>,
    tools: Option<Vec<Tool>>,
    stream: bool,
    temperature: Option<f32>,
    max_tokens: Option<i32>,
}
```

**Event Stream**:

```
data: {"id":"chatcmpl-123","choices":[{"delta":{"role":"assistant"}}]}

data: {"id":"chatcmpl-123","choices":[{"delta":{"content":"Let me"}}]}

data: {"id":"chatcmpl-123","choices":[{"delta":{"tool_calls":[{"function":{"name":"shell"}}]}}]}

data: [DONE]
```

**Implementation**:

```rust
// core/src/chat_completions.rs
pub async fn stream_chat_completions(
    prompt: &Prompt,
    model_family: &ModelFamily,
    client: &reqwest::Client,
    provider: &ModelProviderInfo,
    otel: &OtelEventManager,
) -> Result<ResponseStream> {
    let url = format!("{}/chat/completions", provider.base_url);
    let request = build_chat_request(prompt, model_family);
    
    let response = client
        .post(&url)
        .header("Authorization", bearer_token()?)
        .json(&request)
        .send()
        .await?;
    
    // Parse and aggregate chunks
    let stream = response
        .bytes_stream()
        .eventsource()
        .aggregate_chat_deltas()  // Combine fragmented tool calls
        .map(convert_to_response_event);
    
    Ok(Box::pin(stream))
}
```

---

## Provider Abstraction

### ModelProviderInfo

**Location**: `core/src/model_provider_info.rs`

```rust
pub struct ModelProviderInfo {
    pub name: String,
    pub base_url: String,
    pub wire_api: WireApi,
    pub auth_mode: AuthMode,
    pub supports_vision: bool,
    pub supports_streaming: bool,
}
```

### Supported Providers

| Provider | Base URL | Wire API | Auth |
|----------|----------|----------|------|
| OpenAI | `https://api.openai.com/v1` | Responses/Chat | API Key |
| Azure OpenAI | `https://{deployment}.openai.azure.com/openai` | Chat | API Key |
| Google Gemini | `https://generativelanguage.googleapis.com/v1beta/openai` | Chat | API Key |
| Ollama | `http://localhost:11434/v1` | Chat | None |
| Mistral | `https://api.mistral.ai/v1` | Chat | API Key |
| DeepSeek | `https://api.deepseek.com` | Chat | API Key |
| xAI | `https://api.x.ai/v1` | Chat | API Key |
| Groq | `https://api.groq.com/openai/v1` | Chat | API Key |
| ArceeAI | `https://conductor.arcee.ai/v1` | Chat | API Key |

### Configuration

```yaml
# ~/.codex/config.yaml
provider: openai
providers:
  openai:
    name: OpenAI
    baseURL: https://api.openai.com/v1
    envKey: OPENAI_API_KEY
  
  ollama:
    name: Ollama
    baseURL: http://localhost:11434/v1
    envKey: OLLAMA_API_KEY  # Optional
```

### Selection Logic

```rust
// core/src/config.rs
pub fn resolve_provider(config: &Config) -> ModelProviderInfo {
    let provider_name = config.provider.as_deref().unwrap_or("openai");
    let provider_config = config.providers.get(provider_name)?;
    
    ModelProviderInfo {
        name: provider_config.name.clone(),
        base_url: provider_config.base_url.clone(),
        wire_api: detect_wire_api(&provider_config),
        ...
    }
}
```

---

## Streaming Response Handling

### Response Stream Type

```rust
pub type ResponseStream = Pin<Box<dyn Stream<Item = Result<ResponseEvent>> + Send>>;
```

### Event Processing

```rust
pub enum ResponseEvent {
    ResponseCreated { id: String, model: String },
    Delta { delta: String },
    ReasoningDelta { summary: String },
    FunctionCall { 
        id: String,
        name: String,
        arguments: String,
    },
    FunctionCallOutput {
        id: String,
        output: String,
    },
    Completed {
        id: String,
        usage: TokenUsage,
    },
    Error {
        code: String,
        message: String,
        r#type: Option<String>,
    },
}
```

### Stream Processing Loop

**Location**: `core/src/codex.rs`

```rust
async fn process_response_stream(
    &mut self,
    mut stream: ResponseStream,
) -> Result<()> {
    while let Some(event) = stream.next().await {
        match event? {
            ResponseEvent::Delta { delta } => {
                // Stream text to UI
                self.tx_event.send(Event::AgentMessageDelta {
                    delta: delta.clone()
                }).await?;
                
                // Accumulate for history
                self.current_turn.append_text(&delta);
            }
            
            ResponseEvent::ReasoningDelta { summary } => {
                // o1/o3/o4 reasoning summaries
                self.tx_event.send(Event::AgentReasoningDelta {
                    delta: summary.clone()
                }).await?;
            }
            
            ResponseEvent::FunctionCall { id, name, arguments } => {
                // Parse and execute tool
                let result = self.execute_tool(&name, &arguments).await?;
                
                // Send result back to model
                self.submit_tool_result(id, result).await?;
            }
            
            ResponseEvent::Completed { id, usage } => {
                // Update token tracking
                self.tx_event.send(Event::TokenCount { usage }).await?;
                break;
            }
            
            ResponseEvent::Error { code, message, .. } => {
                return Err(CodexErr::ApiError { code, message });
            }
        }
    }
    Ok(())
}
```

### Aggregating Fragmented Responses

Chat Completions API may fragment function calls:

```rust
// core/src/chat_completions.rs
pub trait AggregateStreamExt: Stream {
    fn aggregate_chat_deltas(self) -> AggregatedStream<Self>
    where
        Self: Sized;
}

impl<S> AggregateStreamExt for S where S: Stream {
    fn aggregate_chat_deltas(self) -> AggregatedStream<Self> {
        AggregatedStream {
            inner: self,
            pending_calls: HashMap::new(),
        }
    }
}
```

This combines fragmented chunks:

```
Chunk 1: {"tool_calls":[{"index":0,"function":{"name":"sh"}}]}
Chunk 2: {"tool_calls":[{"index":0,"function":{"arguments":"ell"}}]}
Chunk 3: {"tool_calls":[{"index":0,"function":{"arguments":"\",\"c"}}]}
         ↓
Aggregated: shell with arguments "cmd": "..."
```

---

## Token Management

### Usage Tracking

```rust
pub struct TokenUsage {
    pub input_tokens: i64,
    pub output_tokens: i64,
    pub cached_tokens: i64,  // Prompt caching savings
}

impl TokenUsage {
    pub fn total(&self) -> i64 {
        self.input_tokens + self.output_tokens
    }
    
    pub fn effective_input(&self) -> i64 {
        self.input_tokens - self.cached_tokens
    }
}
```

### Context Window Management

```rust
impl ModelClient {
    pub fn get_model_context_window(&self) -> Option<i64> {
        let pct = self.config.model_family.effective_context_window_percent;
        self.config
            .model_context_window
            .or_else(|| {
                get_model_info(&self.config.model_family)
                    .map(|info| info.context_window)
            })
            .map(|w| w.saturating_mul(pct) / 100)
    }
    
    pub fn get_auto_compact_token_limit(&self) -> Option<i64> {
        self.config.model_auto_compact_token_limit.or_else(|| {
            get_model_info(&self.config.model_family)
                .and_then(|info| info.auto_compact_token_limit)
        })
    }
}
```

### Model Information Database

**Location**: `core/src/openai_model_info.rs`

```rust
pub struct ModelInfo {
    pub context_window: i64,
    pub auto_compact_token_limit: Option<i64>,
    pub supports_vision: bool,
    pub supports_function_calling: bool,
}

pub fn get_model_info(model_family: &ModelFamily) -> Option<ModelInfo> {
    match model_family.model_name.as_str() {
        "gpt-4" | "gpt-4-turbo" => Some(ModelInfo {
            context_window: 128_000,
            auto_compact_token_limit: Some(100_000),
            supports_vision: true,
            supports_function_calling: true,
        }),
        "o1-preview" | "o1-mini" => Some(ModelInfo {
            context_window: 128_000,
            auto_compact_token_limit: Some(100_000),
            supports_vision: false,
            supports_function_calling: true,
        }),
        // ...
    }
}
```

### Auto-Compaction Trigger

When approaching token limit:

```rust
// core/src/codex.rs
async fn check_history_compaction(&mut self) -> Result<()> {
    let current_tokens = self.estimate_token_count();
    let limit = self.model_client.get_auto_compact_token_limit()?;
    
    if current_tokens > limit {
        // Trigger compaction
        let compacted = build_compacted_history(
            &self.history,
            &self.model_client
        ).await?;
        
        self.history = compacted;
        self.tx_event.send(Event::HistoryCompacted).await?;
    }
    
    Ok(())
}
```

---

## Error Handling

### Retry Logic

```rust
// core/src/client.rs
async fn stream_with_retry(
    &self,
    prompt: &Prompt,
    max_retries: usize,
) -> Result<ResponseStream> {
    let mut attempt = 0;
    
    loop {
        match self.stream(prompt).await {
            Ok(stream) => return Ok(stream),
            Err(e) if is_retryable(&e) && attempt < max_retries => {
                attempt += 1;
                let delay = backoff(attempt);
                tokio::time::sleep(delay).await;
                continue;
            }
            Err(e) => return Err(e),
        }
    }
}

fn is_retryable(error: &CodexErr) -> bool {
    matches!(error,
        CodexErr::NetworkError { .. } |
        CodexErr::RateLimited { .. } |
        CodexErr::ServerError { code: 500..=599, .. }
    )
}
```

### Rate Limiting

```rust
pub struct RateLimitSnapshot {
    pub window: RateLimitWindow,  // "minute" | "hour" | "day"
    pub remaining: i64,
    pub limit: i64,
    pub resets_at: DateTime<Utc>,
}

// Emitted as events
Event::RateLimitWarning { snapshot }
```

### Usage Limit Errors

```rust
// core/src/error.rs
pub enum CodexErr {
    UsageLimitReached {
        plan_type: PlanType,  // "free" | "plus" | "team"
        resets_at: i64,       // Unix timestamp
    },
    // ...
}
```

---

## Performance Optimizations

### Connection Pooling

```rust
// core/src/default_client.rs
pub fn create_client() -> reqwest::Client {
    reqwest::Client::builder()
        .pool_max_idle_per_host(10)
        .pool_idle_timeout(Duration::from_secs(90))
        .timeout(Duration::from_secs(60))
        .build()
        .expect("HTTP client")
}
```

### Streaming Buffer Management

Responses are processed incrementally to avoid memory spikes:

```rust
let stream = response
    .bytes_stream()
    .map(|chunk| {
        // Process small chunks immediately
        parse_sse_chunk(&chunk)
    });
```

### Prompt Caching

For compatible models (Claude, GPT-4):
- System prompts are cached
- Long AGENTS.md files benefit from caching
- Reduces effective input tokens

---

## Related Documentation

- [03-prompt-processing.md](./03-prompt-processing.md) - How prompts are built
- [05-system-prompts.md](./05-system-prompts.md) - Prompt content
- [13-authentication.md](./13-authentication.md) - Auth mechanisms

