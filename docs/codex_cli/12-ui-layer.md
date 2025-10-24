# Codex CLI - UI Layer

## Table of Contents
- [Terminal UI Architecture](#terminal-ui-architecture)
- [Interactive Mode](#interactive-mode)
- [Non-Interactive Mode](#non-interactive-mode)
- [Event Handling](#event-handling)
- [Rendering Pipeline](#rendering-pipeline)
- [Approval Prompts](#approval-prompts)

---

## Terminal UI Architecture

### Technology: ratatui

**ratatui** is a Rust TUI (Text User Interface) framework that provides:
- Cross-platform terminal manipulation
- Widget-based rendering
- Event handling
- Layout management

**Location**: `codex-rs/tui/`

### TUI Structure

```
┌──────────────────────────────────────────────────────────┐
│                    Terminal Screen                        │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Header: Session Info, Model, Approval Mode       │  │
│  ├────────────────────────────────────────────────────┤  │
│  │                                                    │  │
│  │  Main Content Area:                               │  │
│  │  • Conversation history                           │  │
│  │  • Agent responses (streaming)                    │  │
│  │  • Tool execution output                          │  │
│  │  • Approval prompts                               │  │
│  │                                                    │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  Plan View (if active):                           │  │
│  │  ☐ Step 1: Description                            │  │
│  │  ☑ Step 2: Description                            │  │
│  │  ⚙ Step 3: Description                            │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  Status Bar: Token count, Duration, Status        │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  Input Prompt: > Enter your message...            │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### App State

**Location**: `tui/src/app.rs`

```rust
pub struct App {
    pub state: AppState,
    pub codex: Option<Codex>,
    pub config: Config,
    pub conversation_history: Vec<Turn>,
    pub current_input: String,
    pub mode: Mode,
    pub plan: Option<Plan>,
}

pub enum AppState {
    Initializing,
    Ready,
    Processing,
    AwaitingApproval,
    Error { message: String },
    Exiting,
}

pub enum Mode {
    Normal,         // Regular chat
    Input,          // User typing
    Approval,       // Approval prompt active
    PlanView,       // Viewing plan
}
```

---

## Interactive Mode

### Initialization

**Location**: `tui/src/main.rs`

```rust
pub async fn run_tui(cli: TuiCli, config_overrides: &CliConfigOverrides) -> ExitCode {
    // 1. Load configuration
    let config = Config::load_with_overrides(config_overrides)?;
    
    // 2. Initialize terminal
    let mut terminal = setup_terminal()?;
    
    // 3. Create app state
    let mut app = App::new(config);
    
    // 4. Spawn Codex
    let codex = Codex::spawn(CodexParams {
        config: app.config.clone(),
        initial_prompt: cli.prompt,
    }).await?;
    
    app.codex = Some(codex);
    
    // 5. Run main loop
    let exit_info = run_app_loop(&mut terminal, &mut app).await?;
    
    // 6. Cleanup
    restore_terminal(terminal)?;
    
    exit_info.exit_code
}
```

### Main Event Loop

```rust
async fn run_app_loop(
    terminal: &mut Terminal<CrosstermBackend<Stdout>>,
    app: &mut App,
) -> Result<AppExitInfo> {
    loop {
        // 1. Render UI
        terminal.draw(|f| render_app(f, app))?;
        
        // 2. Handle events
        tokio::select! {
            // Terminal events (keyboard, mouse)
            Some(event) = terminal_events.next() => {
                handle_terminal_event(app, event).await?;
            }
            
            // Codex events (responses, completions)
            Some(codex_event) = app.codex.as_mut()?.recv_event() => {
                handle_codex_event(app, codex_event).await?;
            }
            
            // Tick for animations
            _ = tick_interval.tick() => {
                app.tick();
            }
        }
        
        // 3. Check exit condition
        if matches!(app.state, AppState::Exiting) {
            break;
        }
    }
    
    Ok(app.get_exit_info())
}
```

### Keyboard Handling

```rust
async fn handle_terminal_event(app: &mut App, event: Event) -> Result<()> {
    match event {
        Event::Key(key_event) => {
            match app.mode {
                Mode::Normal => handle_normal_mode(app, key_event).await?,
                Mode::Input => handle_input_mode(app, key_event).await?,
                Mode::Approval => handle_approval_mode(app, key_event).await?,
                Mode::PlanView => handle_plan_view_mode(app, key_event).await?,
            }
        }
        Event::Resize(width, height) => {
            app.handle_resize(width, height);
        }
        _ => {}
    }
    Ok(())
}

async fn handle_normal_mode(app: &mut App, key: KeyEvent) -> Result<()> {
    match key.code {
        KeyCode::Char('q') if key.modifiers.contains(KeyModifiers::CONTROL) => {
            app.initiate_exit();
        }
        KeyCode::Char('c') if key.modifiers.contains(KeyModifiers::CONTROL) => {
            app.cancel_current_turn();
        }
        KeyCode::Char('i') | KeyCode::Enter => {
            app.enter_input_mode();
        }
        KeyCode::Char('p') => {
            app.toggle_plan_view();
        }
        KeyCode::Up => app.scroll_up(),
        KeyCode::Down => app.scroll_down(),
        _ => {}
    }
    Ok(())
}

async fn handle_input_mode(app: &mut App, key: KeyEvent) -> Result<()> {
    match key.code {
        KeyCode::Enter if !key.modifiers.contains(KeyModifiers::SHIFT) => {
            let input = app.current_input.clone();
            app.current_input.clear();
            app.mode = Mode::Normal;
            
            // Submit to Codex
            app.codex.as_ref()?.submit(Op::UserTurn {
                input: UserInput { text: input, files: vec![] },
                context: vec![],
            }).await?;
        }
        KeyCode::Char(c) => {
            app.current_input.push(c);
        }
        KeyCode::Backspace => {
            app.current_input.pop();
        }
        KeyCode::Esc => {
            app.mode = Mode::Normal;
        }
        _ => {}
    }
    Ok(())
}

async fn handle_approval_mode(app: &mut App, key: KeyEvent) -> Result<()> {
    match key.code {
        KeyCode::Char('y') | KeyCode::Char('Y') => {
            app.approve_current_request().await?;
            app.mode = Mode::Normal;
        }
        KeyCode::Char('n') | KeyCode::Char('N') => {
            app.deny_current_request().await?;
            app.mode = Mode::Normal;
        }
        _ => {}
    }
    Ok(())
}
```

---

## Non-Interactive Mode

### Exec Mode

**Location**: `codex-rs/exec/`

```rust
pub async fn run_exec_mode(
    cli: ExecCli,
    config_overrides: &CliConfigOverrides,
) -> ExitCode {
    // 1. Load config
    let config = Config::load_with_overrides(config_overrides)?;
    
    // 2. Spawn Codex
    let codex = Codex::spawn(CodexParams {
        config,
        initial_prompt: Some(cli.prompt.clone()),
    }).await?;
    
    // 3. Process without UI
    let result = process_non_interactive(codex, cli).await?;
    
    // 4. Output result
    if cli.json {
        println!("{}", serde_json::to_string_pretty(&result)?);
    } else {
        println!("{}", result.output);
    }
    
    result.exit_code
}

async fn process_non_interactive(
    mut codex: Codex,
    cli: ExecCli,
) -> Result<ExecResult> {
    let mut output = String::new();
    let mut error = None;
    
    // Process events without UI
    while let Ok(event) = codex.recv_event().await {
        match event {
            Event::AgentMessageDelta { delta } => {
                if !cli.quiet {
                    print!("{}", delta);
                    io::stdout().flush()?;
                }
                output.push_str(&delta);
            }
            
            Event::TaskCompleted { .. } => {
                break;
            }
            
            Event::ErrorEvent { message, .. } => {
                error = Some(message);
                break;
            }
            
            _ => {
                // Log other events if verbose
                if cli.verbose {
                    eprintln!("Event: {:?}", event);
                }
            }
        }
    }
    
    Ok(ExecResult {
        output,
        error,
        exit_code: if error.is_some() { ExitCode::FAILURE } else { ExitCode::SUCCESS },
    })
}
```

---

## Event Handling

### Event Processing

```rust
async fn handle_codex_event(app: &mut App, event: Event) -> Result<()> {
    match event {
        Event::SessionConfigured { .. } => {
            app.state = AppState::Ready;
        }
        
        Event::TaskStarted { turn_id } => {
            app.state = AppState::Processing;
            app.current_turn_id = Some(turn_id);
        }
        
        Event::AgentMessageDelta { delta } => {
            app.append_to_current_response(&delta);
        }
        
        Event::AgentReasoningDelta { delta } => {
            app.append_to_reasoning(&delta);
        }
        
        Event::ItemStarted { item } => {
            app.add_tool_call(item);
        }
        
        Event::ItemCompleted { item, output } => {
            app.complete_tool_call(item, output);
        }
        
        Event::ExecApprovalRequest { command, justification } => {
            app.state = AppState::AwaitingApproval;
            app.mode = Mode::Approval;
            app.current_approval = Some(ApprovalRequest::Exec {
                command,
                justification,
            });
        }
        
        Event::PlanUpdated { steps, explanation } => {
            app.plan = Some(Plan { steps, explanation });
        }
        
        Event::TurnDiff { summary, .. } => {
            app.update_diff_summary(summary);
        }
        
        Event::TokenCount { usage } => {
            app.update_token_count(usage);
        }
        
        Event::TaskCompleted { turn_id, duration } => {
            app.state = AppState::Ready;
            app.finalize_turn(turn_id, duration);
        }
        
        Event::ErrorEvent { message, .. } => {
            app.state = AppState::Error { message };
        }
        
        _ => {}
    }
    
    Ok(())
}
```

---

## Rendering Pipeline

### Main Render Function

```rust
fn render_app(f: &mut Frame, app: &App) {
    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Length(3),      // Header
            Constraint::Min(0),         // Main content
            Constraint::Length(5),      // Plan (if visible)
            Constraint::Length(1),      // Status bar
            Constraint::Length(3),      // Input area
        ])
        .split(f.size());
    
    render_header(f, chunks[0], app);
    render_content(f, chunks[1], app);
    
    if app.plan.is_some() && app.mode == Mode::PlanView {
        render_plan(f, chunks[2], app);
    }
    
    render_status_bar(f, chunks[3], app);
    render_input(f, chunks[4], app);
}
```

### Header Rendering

```rust
fn render_header(f: &mut Frame, area: Rect, app: &App) {
    let title = format!(
        " Codex | Model: {} | Mode: {} ",
        app.config.model,
        app.config.approval_mode
    );
    
    let header = Paragraph::new(title)
        .style(Style::default().fg(Color::Cyan).bold())
        .block(Block::default()
            .borders(Borders::ALL)
            .border_type(BorderType::Rounded));
    
    f.render_widget(header, area);
}
```

### Content Rendering

```rust
fn render_content(f: &mut Frame, area: Rect, app: &App) {
    let mut lines = Vec::new();
    
    // Render conversation history
    for turn in &app.conversation_history {
        // User message
        lines.push(Line::from(vec![
            Span::styled("❯ ", Style::default().fg(Color::Green).bold()),
            Span::raw(&turn.user_input),
        ]));
        
        // Agent response
        if !turn.agent_response.is_empty() {
            let response_lines: Vec<Line> = turn.agent_response
                .lines()
                .map(|line| Line::from(Span::raw(line)))
                .collect();
            lines.extend(response_lines);
        }
        
        // Tool calls
        for tool_call in &turn.tool_calls {
            lines.push(Line::from(vec![
                Span::styled("  🔧 ", Style::default().fg(Color::Yellow)),
                Span::styled(&tool_call.name, Style::default().fg(Color::Cyan)),
                Span::raw(": "),
                Span::styled(&tool_call.status, style_for_status(&tool_call.status)),
            ]));
        }
        
        lines.push(Line::from("")); // Blank line between turns
    }
    
    // Scroll to show latest content
    let scroll_offset = app.scroll_offset;
    
    let paragraph = Paragraph::new(lines)
        .block(Block::default()
            .borders(Borders::ALL)
            .title(" Conversation "))
        .scroll((scroll_offset as u16, 0))
        .wrap(Wrap { trim: false });
    
    f.render_widget(paragraph, area);
}
```

### Plan Rendering

```rust
fn render_plan(f: &mut Frame, area: Rect, app: &App) {
    if let Some(plan) = &app.plan {
        let mut lines = Vec::new();
        
        for step in &plan.steps {
            let (icon, color) = match step.status {
                PlanStatus::Pending => ("☐", Color::Gray),
                PlanStatus::InProgress => ("⚙", Color::Yellow),
                PlanStatus::Completed => ("☑", Color::Green),
                PlanStatus::Cancelled => ("✗", Color::Red),
            };
            
            lines.push(Line::from(vec![
                Span::styled(icon, Style::default().fg(color)),
                Span::raw(" "),
                Span::raw(&step.description),
            ]));
        }
        
        let paragraph = Paragraph::new(lines)
            .block(Block::default()
                .borders(Borders::ALL)
                .title(" Plan ")
                .border_style(Style::default().fg(Color::Cyan)));
        
        f.render_widget(paragraph, area);
    }
}
```

### Status Bar

```rust
fn render_status_bar(f: &mut Frame, area: Rect, app: &App) {
    let status_text = match &app.state {
        AppState::Initializing => "Initializing...".to_string(),
        AppState::Ready => "Ready".to_string(),
        AppState::Processing => "Processing...".to_string(),
        AppState::AwaitingApproval => "Awaiting approval (y/n)".to_string(),
        AppState::Error { message } => format!("Error: {}", message),
        AppState::Exiting => "Exiting...".to_string(),
    };
    
    let token_info = if let Some(usage) = &app.token_usage {
        format!(" | Tokens: {} in, {} out", usage.input_tokens, usage.output_tokens)
    } else {
        String::new()
    };
    
    let full_status = format!("{}{}", status_text, token_info);
    
    let status = Paragraph::new(full_status)
        .style(Style::default().fg(Color::White).bg(Color::DarkGray));
    
    f.render_widget(status, area);
}
```

---

## Approval Prompts

### Rendering Approval Request

```rust
fn render_approval_prompt(f: &mut Frame, area: Rect, app: &App) {
    if let Some(approval_req) = &app.current_approval {
        let prompt_text = match approval_req {
            ApprovalRequest::Exec { command, justification } => {
                format!(
                    "Approve command execution?\n\nCommand: {}\n\nReason: {}\n\nApprove? (y/n)",
                    command.join(" "),
                    justification.as_deref().unwrap_or("N/A")
                )
            }
            ApprovalRequest::ApplyPatch { file_path, hunks } => {
                format!(
                    "Approve file modification?\n\nFile: {}\nChanges: {} hunks\n\nApprove? (y/n)",
                    file_path.display(),
                    hunks
                )
            }
        };
        
        let popup = Paragraph::new(prompt_text)
            .block(Block::default()
                .borders(Borders::ALL)
                .border_type(BorderType::Double)
                .border_style(Style::default().fg(Color::Yellow))
                .title(" Approval Required "))
            .wrap(Wrap { trim: true });
        
        // Center popup
        let popup_area = centered_rect(60, 40, area);
        f.render_widget(Clear, popup_area); // Clear background
        f.render_widget(popup, popup_area);
    }
}

fn centered_rect(percent_x: u16, percent_y: u16, r: Rect) -> Rect {
    let popup_layout = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Percentage((100 - percent_y) / 2),
            Constraint::Percentage(percent_y),
            Constraint::Percentage((100 - percent_y) / 2),
        ])
        .split(r);
    
    Layout::default()
        .direction(Direction::Horizontal)
        .constraints([
            Constraint::Percentage((100 - percent_x) / 2),
            Constraint::Percentage(percent_x),
            Constraint::Percentage((100 - percent_x) / 2),
        ])
        .split(popup_layout[1])[1]
}
```

---

## Related Documentation

- [02-architecture.md](./02-architecture.md) - Overall architecture
- [10-implementation.md](./10-implementation.md) - Implementation details
- [15-code-reference.md](./15-code-reference.md) - Code reference

