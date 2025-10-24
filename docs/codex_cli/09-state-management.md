# Codex CLI - State Management

## Table of Contents
- [Session State](#session-state)
- [Turn State](#turn-state)
- [Conversation History](#conversation-history)
- [History Compaction](#history-compaction)
- [Diff Tracking](#diff-tracking)

---

## Session State

### SessionState Structure

**Location**: `core/src/state/mod.rs`

```rust
pub struct SessionState {
    pub session_id: SessionId,
    pub conversation_id: ConversationId,
    pub services: SessionServices,
    pub active_turn: Option<ActiveTurn>,
    pub history: ConversationHistory,
    pub config: Arc<Config>,
}

pub struct SessionServices {
    pub tool_router: ToolRouter,
    pub model_client: ModelClient,
    pub mcp_manager: McpConnectionManager,
    pub diff_tracker: TurnDiffTracker,
}
```

### Session Lifecycle

```
┌─────────────────────────────────────┐
│  1. Session Initialization          │
│     • Generate session ID            │
│     • Load configuration             │
│     • Initialize services            │
│     • Connect to MCP servers         │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  2. Active Session                   │
│     • Process user turns             │
│     • Execute tools                  │
│     • Track changes                  │
│     • Maintain history               │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│  3. Session Termination              │
│     • Save conversation history      │
│     • Close MCP connections          │
│     • Clean up resources             │
└─────────────────────────────────────┘
```

### Session Types

```rust
pub enum SessionSource {
    Interactive,      // TUI mode
    NonInteractive,   // Exec/quiet mode
    Resume(SessionId), // Resumed session
    Mcp,              // MCP server mode
}
```

---

## Turn State

### ActiveTurn Structure

**Location**: `core/src/state/turn.rs`

```rust
pub struct ActiveTurn {
    pub turn_id: TurnId,
    pub started_at: Instant,
    pub user_input: UserInput,
    pub accumulated_response: String,
    pub reasoning_summary: Option<String>,
    pub tool_calls: Vec<ToolCall>,
    pub status: TurnStatus,
}

pub enum TurnStatus {
    Processing,
    AwaitingApproval { tool_call: ToolCall },
    AwaitingToolResult { tool_call_id: String },
    Completed,
    Failed { error: String },
    Cancelled,
}
```

### Turn Lifecycle

```
User Submits Input
      │
      ▼
┌──────────────────────┐
│ Create ActiveTurn    │
│  • Generate turn ID  │
│  • Record input      │
│  • Set status        │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│ Process Response     │
│  • Stream deltas     │
│  • Accumulate text   │
│  • Detect tool calls │
└──────┬───────────────┘
       │
       ├──→ Text Delta
       │    └─→ Append to accumulated_response
       │
       ├──→ Reasoning Delta
       │    └─→ Update reasoning_summary
       │
       └──→ Tool Call
            └─→ Execute → Get Result → Continue
       │
       ▼
┌──────────────────────┐
│ Complete Turn        │
│  • Save to history   │
│  • Track diffs       │
│  • Emit event        │
└──────────────────────┘
```

### Turn Management

**Implementation**: `core/src/codex.rs`

```rust
impl Codex {
    async fn start_turn(&mut self, input: UserInput) -> Result<TurnId> {
        let turn_id = TurnId::new();
        
        self.active_turn = Some(ActiveTurn {
            turn_id,
            started_at: Instant::now(),
            user_input: input,
            accumulated_response: String::new(),
            reasoning_summary: None,
            tool_calls: Vec::new(),
            status: TurnStatus::Processing,
        });
        
        self.tx_event.send(Event::TaskStarted {
            turn_id,
        }).await?;
        
        Ok(turn_id)
    }
    
    async fn complete_turn(&mut self) -> Result<()> {
        if let Some(turn) = self.active_turn.take() {
            // Save to history
            self.history.add_turn(turn.clone());
            
            // Track file changes
            self.diff_tracker.finalize_turn();
            
            // Emit completion event
            self.tx_event.send(Event::TaskCompleted {
                turn_id: turn.turn_id,
                duration: turn.started_at.elapsed(),
            }).await?;
        }
        
        Ok(())
    }
}
```

---

## Conversation History

### ConversationHistory Structure

**Location**: `core/src/conversation_history.rs`

```rust
pub struct ConversationHistory {
    turns: Vec<CompletedTurn>,
    max_turns: usize,
    total_tokens: i64,
}

pub struct CompletedTurn {
    pub turn_id: TurnId,
    pub messages: Vec<Message>,
    pub tool_calls: Vec<ToolCallRecord>,
    pub token_usage: TokenUsage,
    pub timestamp: DateTime<Utc>,
}

pub enum Message {
    User {
        content: String,
        attachments: Vec<Attachment>,
    },
    Assistant {
        content: String,
        reasoning: Option<String>,
    },
    Tool {
        call_id: String,
        name: String,
        result: String,
    },
}
```

### History Operations

```rust
impl ConversationHistory {
    pub fn add_turn(&mut self, turn: CompletedTurn) {
        self.turns.push(turn.clone());
        self.total_tokens += turn.token_usage.total();
        
        // Prune if exceeds max
        if self.turns.len() > self.max_turns {
            self.turns.remove(0);
        }
    }
    
    pub fn get_messages_for_api(&self) -> Vec<Message> {
        let mut messages = Vec::new();
        
        for turn in &self.turns {
            messages.extend(turn.messages.clone());
        }
        
        messages
    }
    
    pub fn estimate_tokens(&self) -> i64 {
        self.total_tokens
    }
    
    pub fn clear(&mut self) {
        self.turns.clear();
        self.total_tokens = 0;
    }
}
```

### History Persistence

**Location**: `core/src/conversation_manager.rs`

```rust
pub struct ConversationManager {
    storage_dir: PathBuf,
}

impl ConversationManager {
    pub async fn save_conversation(
        &self,
        conversation_id: ConversationId,
        history: &ConversationHistory,
    ) -> Result<()> {
        let file_path = self.storage_dir
            .join(format!("{}.json", conversation_id));
        
        let json = serde_json::to_string_pretty(history)?;
        tokio::fs::write(&file_path, json).await?;
        
        Ok(())
    }
    
    pub async fn load_conversation(
        &self,
        conversation_id: ConversationId,
    ) -> Result<ConversationHistory> {
        let file_path = self.storage_dir
            .join(format!("{}.json", conversation_id));
        
        let json = tokio::fs::read_to_string(&file_path).await?;
        let history = serde_json::from_str(&json)?;
        
        Ok(history)
    }
    
    pub async fn list_conversations(&self) -> Result<Vec<ConversationId>> {
        let mut conversations = Vec::new();
        let mut entries = tokio::fs::read_dir(&self.storage_dir).await?;
        
        while let Some(entry) = entries.next_entry().await? {
            if let Some(id) = parse_conversation_id(&entry.file_name()) {
                conversations.push(id);
            }
        }
        
        Ok(conversations)
    }
}
```

### History Configuration

```yaml
# ~/.codex/config.yaml
history:
  maxSize: 1000              # Max turns to keep
  saveHistory: true          # Whether to persist
  sensitivePatterns:         # Filter these from history
    - "password"
    - "api_key"
    - "secret"
```

---

## History Compaction

### Why Compaction?

When conversation history approaches model's context window:
- Old turns are summarized
- Recent turns kept verbatim
- Tool results always preserved
- System prompt never touched

### Compaction Strategy

**Location**: `core/src/codex/compact.rs`

```rust
pub async fn build_compacted_history(
    original_history: &[Message],
    model_client: &ModelClient,
) -> Result<Vec<Message>> {
    let context_window = model_client.get_model_context_window()?;
    let current_tokens = estimate_tokens(original_history);
    let target_tokens = (context_window as f64 * 0.7) as i64; // 70% of limit
    
    if current_tokens <= target_tokens {
        return Ok(original_history.to_vec());
    }
    
    // 1. Keep system prompt
    let system_msg = original_history.first().cloned();
    
    // 2. Separate into old and recent
    let (old, recent) = split_history(original_history, 5); // Keep last 5 turns
    
    // 3. Summarize old turns
    let summary = summarize_turns(&old, model_client).await?;
    
    // 4. Combine: system + summary + recent
    let mut compacted = vec![system_msg.unwrap()];
    compacted.push(Message::User {
        content: format!("Previous conversation summary:\n{}", summary),
        attachments: vec![],
    });
    compacted.extend(recent);
    
    Ok(compacted)
}

async fn summarize_turns(
    turns: &[Message],
    model_client: &ModelClient,
) -> Result<String> {
    let prompt = format!(
        "Summarize this conversation, preserving key decisions and context:\n\n{}",
        format_messages(turns)
    );
    
    let summary_request = build_summary_request(&prompt);
    let response = model_client.complete(summary_request).await?;
    
    Ok(response.content)
}
```

### Compaction Trigger

```rust
impl Codex {
    async fn check_compaction_needed(&self) -> bool {
        let current = self.history.estimate_tokens();
        let limit = self.model_client
            .get_auto_compact_token_limit()
            .unwrap_or(100_000);
        
        current > limit
    }
    
    async fn trigger_compaction(&mut self) -> Result<()> {
        let messages = self.history.get_messages_for_api();
        
        let compacted = build_compacted_history(
            &messages,
            &self.model_client,
        ).await?;
        
        self.history = ConversationHistory::from_messages(compacted);
        
        self.tx_event.send(Event::HistoryCompacted {
            old_token_count: messages.len(),
            new_token_count: compacted.len(),
        }).await?;
        
        Ok(())
    }
}
```

### Compaction for o-series Models

For o1/o3/o4 models (compact mode):

```rust
// core/src/tasks/compact.rs
pub struct CompactTask {
    base_messages: Vec<Message>,
}

impl CompactTask {
    pub fn build_prompt(&self, user_input: &str) -> Prompt {
        // Collect only user messages (model self-reflects)
        let user_messages = collect_user_messages(&self.base_messages);
        
        let summary = format!(
            "Previous requests:\n{}",
            user_messages.join("\n")
        );
        
        Prompt {
            messages: vec![
                Message::System { content: COMPACT_SYSTEM_PROMPT },
                Message::User { content: summary },
                Message::User { content: user_input },
            ],
            tools: self.tools.clone(),
        }
    }
}
```

---

## Diff Tracking

### TurnDiffTracker Structure

**Location**: `core/src/turn_diff_tracker.rs`

```rust
pub struct TurnDiffTracker {
    pub files_modified: HashMap<PathBuf, FileDiff>,
    pub files_created: HashSet<PathBuf>,
    pub files_deleted: HashSet<PathBuf>,
}

pub struct FileDiff {
    pub path: PathBuf,
    pub original_content: Option<String>,
    pub current_content: String,
    pub hunks: Vec<DiffHunk>,
}

pub struct DiffHunk {
    pub old_start: usize,
    pub old_count: usize,
    pub new_start: usize,
    pub new_count: usize,
    pub lines: Vec<DiffLine>,
}

pub enum DiffLine {
    Context(String),
    Addition(String),
    Deletion(String),
}
```

### Tracking File Changes

```rust
impl TurnDiffTracker {
    pub fn record_file_modification(
        &mut self,
        path: PathBuf,
        original: String,
        modified: String,
    ) -> Result<()> {
        let hunks = compute_diff(&original, &modified);
        
        self.files_modified.insert(
            path.clone(),
            FileDiff {
                path,
                original_content: Some(original),
                current_content: modified,
                hunks,
            },
        );
        
        Ok(())
    }
    
    pub fn record_file_creation(&mut self, path: PathBuf) {
        self.files_created.insert(path);
    }
    
    pub fn record_file_deletion(&mut self, path: PathBuf) {
        self.files_deleted.insert(path);
    }
    
    pub fn get_summary(&self) -> DiffSummary {
        DiffSummary {
            files_modified: self.files_modified.len(),
            files_created: self.files_created.len(),
            files_deleted: self.files_deleted.len(),
            total_additions: self.count_additions(),
            total_deletions: self.count_deletions(),
        }
    }
    
    pub fn generate_unified_diff(&self) -> String {
        let mut diff = String::new();
        
        for (path, file_diff) in &self.files_modified {
            diff.push_str(&format!("diff --git a/{} b/{}\n", path.display(), path.display()));
            diff.push_str(&format_hunks(&file_diff.hunks));
        }
        
        diff
    }
}
```

### Emitting Diff Events

```rust
impl Codex {
    async fn emit_turn_diff(&self) -> Result<()> {
        let summary = self.diff_tracker.get_summary();
        
        self.tx_event.send(Event::TurnDiff {
            summary,
            files_modified: self.diff_tracker.files_modified.keys().cloned().collect(),
            files_created: self.diff_tracker.files_created.iter().cloned().collect(),
            files_deleted: self.diff_tracker.files_deleted.iter().cloned().collect(),
        }).await?;
        
        Ok(())
    }
}
```

### UI Visualization

Diff tracker enables UI features:
- Real-time file change list
- Unified diff viewer
- Git-style patch display
- Summary statistics

```
╭─────────────────────────────────────╮
│ Changes This Turn                   │
├─────────────────────────────────────┤
│ Modified:                           │
│   • src/main.rs         (+12, -3)   │
│   • src/utils.rs        (+5, -0)    │
│                                     │
│ Created:                            │
│   • tests/integration.rs            │
│                                     │
│ Total: 17 additions, 3 deletions    │
╰─────────────────────────────────────╯
```

---

## State Persistence

### Session Resume

**Location**: `core/src/conversation_manager.rs`

```rust
pub async fn resume_session(
    conversation_id: ConversationId,
) -> Result<Codex> {
    // 1. Load conversation history
    let history = conversation_manager.load_conversation(conversation_id).await?;
    
    // 2. Restore config
    let config = load_config_for_session(conversation_id).await?;
    
    // 3. Create new Codex instance with history
    let codex = Codex::spawn_with_history(
        config,
        history,
        conversation_id,
    ).await?;
    
    Ok(codex)
}
```

### State Serialization

```rust
#[derive(Serialize, Deserialize)]
pub struct SerializedSession {
    pub session_id: SessionId,
    pub conversation_id: ConversationId,
    pub history: ConversationHistory,
    pub config_snapshot: ConfigSnapshot,
    pub created_at: DateTime<Utc>,
    pub last_updated: DateTime<Utc>,
}
```

---

## Best Practices

### Managing History

1. **Regular Cleanup**: Remove old sessions periodically
2. **Sensitive Data**: Configure `sensitivePatterns` to filter secrets
3. **Size Limits**: Set appropriate `maxSize` for your workflow
4. **Compaction**: Trust auto-compaction, but monitor token usage

### Diff Tracking

1. **Review Changes**: Always review diff before committing
2. **Incremental Commits**: Commit after each logical change
3. **Backup**: Use git to backup before `full-auto` mode
4. **Track Progress**: Use diff stats to measure productivity

### Session Management

1. **Naming**: Use descriptive conversation IDs
2. **Resume**: Resume instead of starting fresh when possible
3. **Clean State**: Clear history when switching contexts
4. **Export**: Export important conversations for documentation

---

## Related Documentation

- [03-prompt-processing.md](./03-prompt-processing.md) - How history is used in prompts
- [04-llm-integration.md](./04-llm-integration.md) - Token management
- [10-implementation.md](./10-implementation.md) - Implementation details

