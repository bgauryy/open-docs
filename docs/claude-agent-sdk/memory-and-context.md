# Memory and Context Management in Claude Agent SDK

> Understanding how Claude handles memory, sessions, and context persistence

---

## Table of Contents

1. [Understanding "Memory" in Claude](#understanding-memory-in-claude)
2. [Session Management (SDK)](#session-management-sdk)
3. [Session Management (CLI)](#session-management-cli)
4. [Manual Context Management](#manual-context-management)
5. [Best Practices](#best-practices)
6. [Common Patterns](#common-patterns)
7. [FAQ](#faq)

---

## Understanding "Memory" in Claude

### What Memory Is NOT

**Important Clarifications:**

❌ **No Automatic Memory System** - Claude Agent SDK does not have a built-in automatic memory feature that remembers things across different sessions without explicit session management.

❌ **No Automatic Markdown File Creation** - Claude does NOT automatically create markdown files to track project context. If you've heard this from others, they likely created these files manually or instructed Claude to create them.

❌ **No Claude.ai Project Integration** - There is NO connection between projects on claude.ai (the web interface) and the Claude Agent SDK or Claude Code CLI. They are separate systems:
- **claude.ai Projects**: Web-based chat interface with its own memory system
- **Claude Code "Projects"**: Simply refers to your local workspace/directory with configuration files

❌ **No Persistent Cross-Session Memory** - Each new session starts fresh unless you explicitly use the `resume` functionality.

### What Memory Actually IS

✅ **Session-Based Conversation History** - Within a single session, Claude maintains the full conversation history (messages, tool uses, results) in memory.

✅ **Session Resumption** - You can resume previous sessions by their session ID, bringing back the full conversation history.

✅ **Session Forking** - You can branch from a previous session to try alternative approaches while keeping the original history.

✅ **Configuration Persistence** - Settings can be stored at user, project (workspace), or local levels in configuration files.

✅ **Manual Context Files** - You can create and maintain context files (markdown, text, etc.) that Claude can read at the start of sessions.

### How Memory Works: The Architecture

```
┌─────────────────────────────────────────────────────────┐
│  New Session (Fresh Start)                              │
│  ┌───────────────────────────────────────────────────┐ │
│  │ No previous context                               │ │
│  │ Reads files you explicitly provide or ask about  │ │
│  │ Session ID: abc-123                               │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                          ↓
                 Messages accumulate
                          ↓
┌─────────────────────────────────────────────────────────┐
│  During Session                                          │
│  ┌───────────────────────────────────────────────────┐ │
│  │ Full conversation history in memory               │ │
│  │ All tool uses and results tracked                 │ │
│  │ Context window managed automatically              │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                          ↓
                 Session ends/pauses
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Resume Session (Continued)                             │
│  ┌───────────────────────────────────────────────────┐ │
│  │ Full history restored from session abc-123        │ │
│  │ Continues exactly where it left off              │ │
│  │ All context preserved                             │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Session vs Project vs Configuration

| Concept | What It Is | Scope |
|---------|-----------|-------|
| **Session** | A conversation with message history | Single conversation thread |
| **Project/Workspace** | Your local directory with code | Directory on your filesystem |
| **Project Settings** | Configuration stored in `.claudeconfig` | Workspace-specific settings |
| **User Settings** | Configuration in `~/.config/claude/` | All your projects |
| **Local Settings** | Configuration in `.claude/` (gitignored) | Project but not committed |

---

## Session Management (SDK)

### Basic Session Lifecycle

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

// 1. Start a new session
const session = query({
  prompt: "Analyze this codebase and create a summary",
  options: {
    cwd: '/path/to/project',
    model: 'claude-sonnet-4',
    maxTurns: 50
  }
});

let sessionId: string | undefined;

// 2. Iterate through messages and capture session ID
for await (const message of session) {
  if (message.type === 'assistant') {
    console.log(message.message.content);
    sessionId = message.session_id;
  } else if (message.type === 'result') {
    console.log('Session completed:', sessionId);
    console.log('Cost:', message.total_cost_usd);
  }
}
```

### Resuming Sessions

Resume a previous session to continue where you left off:

```typescript
// Resume the previous session
const continuedSession = query({
  prompt: "Now implement the changes we discussed",
  options: {
    cwd: '/path/to/project',
    resume: sessionId,  // Use the session ID from before
  }
});

for await (const message of continuedSession) {
  // Claude has full context from previous session
  if (message.type === 'assistant') {
    console.log(message.message.content);
  }
}
```

**Key Points:**
- Session ID is a UUID generated for each conversation
- Sessions are stored by the SDK/CLI (location varies by platform)
- Full message history is restored when resuming
- All tool uses, file edits, and context are preserved

### Forking Sessions

Create a branch from a session to try alternative approaches:

```typescript
// Fork creates a new session ID with copied history
const forkedSession = query({
  prompt: "Try a different approach to the same problem",
  options: {
    cwd: '/path/to/project',
    resume: sessionId,
    forkSession: true  // Creates new session with same history
  }
});
```

**Use Cases for Forking:**
- Try alternative implementations without losing original
- Experiment with different approaches
- Create branches for different team members
- Test risky changes before committing

### Resume from Specific Point

Resume from a specific message in the conversation:

```typescript
// Resume from a checkpoint in the conversation
const checkpointSession = query({
  prompt: "Continue from the refactoring step",
  options: {
    cwd: '/path/to/project',
    resume: sessionId,
    resumeSessionAt: messageId  // message.message.id from SDKAssistantMessage
  }
});
```

**When to Use:**
- Rollback to before a mistake
- Branch from an earlier decision point
- Skip problematic turns
- Resume from a known good state

### Session Hooks

Monitor session lifecycle with hooks:

```typescript
const session = query({
  prompt: "Long-running task",
  options: {
    hooks: {
      SessionStart: [{
        hooks: [async (input, toolUseId, { signal }) => {
          console.log('Session starting:', input.session_id);
          console.log('Source:', input.source); // 'startup' | 'resume' | 'clear' | 'compact'
          console.log('Transcript path:', input.transcript_path);
          return { continue: true };
        }]
      }],
      SessionEnd: [{
        hooks: [async (input, toolUseId, { signal }) => {
          console.log('Session ending:', input.session_id);
          console.log('Reason:', input.reason); // 'error' | 'max_turns' | 'stop' | etc.
          
          // Save session ID for later
          await saveSessionId(input.session_id);
          
          return { continue: true };
        }]
      }]
    }
  }
});
```

### Complete Session Management Example

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';
import fs from 'fs/promises';

interface SessionMetadata {
  id: string;
  startedAt: string;
  description: string;
  lastMessage?: string;
}

class SessionManager {
  private sessionsFile = '.claude-sessions.json';
  
  async saveSession(metadata: SessionMetadata) {
    const sessions = await this.loadSessions();
    sessions[metadata.id] = metadata;
    await fs.writeFile(this.sessionsFile, JSON.stringify(sessions, null, 2));
  }
  
  async loadSessions(): Promise<Record<string, SessionMetadata>> {
    try {
      const data = await fs.readFile(this.sessionsFile, 'utf-8');
      return JSON.parse(data);
    } catch {
      return {};
    }
  }
  
  async resumeLatest() {
    const sessions = await this.loadSessions();
    const latest = Object.values(sessions)
      .sort((a, b) => new Date(b.startedAt).getTime() - new Date(a.startedAt).getTime())[0];
    return latest?.id;
  }
}

// Usage
const manager = new SessionManager();

async function runSession(prompt: string, resumeId?: string) {
  const session = query({
    prompt,
    options: {
      cwd: process.cwd(),
      resume: resumeId,
      hooks: {
        SessionStart: [{
          hooks: [async (input) => {
            await manager.saveSession({
              id: input.session_id,
              startedAt: new Date().toISOString(),
              description: prompt.substring(0, 100)
            });
            return { continue: true };
          }]
        }]
      }
    }
  });
  
  for await (const message of session) {
    if (message.type === 'assistant' && message.message.content) {
      const text = message.message.content
        .filter(c => c.type === 'text')
        .map(c => c.text)
        .join('\n');
      console.log(text);
    }
  }
}

// Start new session
await runSession("Analyze the codebase");

// Resume latest session
const latestId = await manager.resumeLatest();
if (latestId) {
  await runSession("Continue with implementation", latestId);
}
```

---

## Session Management (CLI)

### Basic CLI Session Usage

```bash
# Start a new session
claude "Analyze this codebase"
# Session ID will be displayed: session-abc-123-...

# Resume a session
claude --resume session-abc-123-... "Continue the analysis"

# Fork a session (create branch)
claude --resume session-abc-123-... --fork-session "Try alternative approach"

# Resume from specific message
claude --resume session-abc-123-... --resume-session-at msg-456-... "Continue from checkpoint"
```

### Where Sessions Are Stored

Session data is stored in platform-specific locations:

**macOS/Linux:**
```
~/.local/share/claude/sessions/
  ├── session-abc-123.json
  ├── session-def-456.json
  └── ...
```

**Windows:**
```
%LOCALAPPDATA%\claude\sessions\
  ├── session-abc-123.json
  ├── session-def-456.json
  └── ...
```

### CLI Session Management Commands

```bash
# View session history (if supported by your CLI version)
claude --list-sessions

# Continue most recent session
claude --resume "$(claude --list-sessions | head -n1 | awk '{print $1}')" "Continue"

# Clear a session (if supported)
claude --clear-session session-abc-123

# Export session for debugging
cat ~/.local/share/claude/sessions/session-abc-123.json | jq .
```

### CLI Configuration for Sessions

Configure default session behavior in `.claudeconfig`:

```json
{
  "maxTurns": 100,
  "model": "claude-sonnet-4",
  "permissionMode": "acceptEdits",
  "additionalDirectories": [
    "./docs",
    "./context"
  ]
}
```

---

## Manual Context Management

Since Claude doesn't automatically create memory files, you need to manage context manually. Here are practical strategies.

### Strategy 1: Project Context File

Create a context file that Claude reads at the start of each session:

```markdown
<!-- PROJECT_CONTEXT.md -->
# Project: My Application

## Overview
This is a web application built with React and Node.js.

## Current Status
- Authentication system: ✅ Complete
- User dashboard: 🚧 In Progress
- API integration: ⏳ Not Started

## Architecture Decisions
1. Using JWT for authentication
2. PostgreSQL for database
3. Redis for caching

## Known Issues
- Performance issue with large data sets (see issue #42)
- Need to refactor auth middleware

## Next Steps
1. Complete user dashboard
2. Add API rate limiting
3. Write integration tests
```

**Usage:**
```typescript
const session = query({
  prompt: `Read PROJECT_CONTEXT.md first, then help me implement the user dashboard`,
  options: { cwd: process.cwd() }
});
```

### Strategy 2: Session Summary Files

After each major session, create a summary:

```bash
# At the end of a session, ask Claude to create a summary
claude "Create a summary of what we accomplished in SESSION_SUMMARY_2024-01-15.md"
```

```markdown
<!-- SESSION_SUMMARY_2024-01-15.md -->
# Session Summary - January 15, 2024

## Session ID
session-abc-123-def-456

## What We Did
1. Refactored authentication middleware
2. Added JWT refresh token logic
3. Updated tests for auth flow
4. Fixed bug in token expiration

## Files Modified
- src/middleware/auth.ts
- src/services/token.ts
- tests/auth.test.ts

## Decisions Made
- Use 15-minute access tokens
- Use 7-day refresh tokens
- Store refresh tokens in httpOnly cookies

## Next Session
- Add rate limiting to auth endpoints
- Write integration tests
- Update documentation
```

### Strategy 3: Using TodoWrite for Tracking

Use the built-in `TodoWrite` tool to track progress:

```typescript
const session = query({
  prompt: `
    Create a todo list for the user dashboard feature:
    - Design user profile component
    - Implement settings page
    - Add data visualization
    Then work on the first task.
  `,
  options: { cwd: process.cwd() }
});
```

Claude will use the `TodoWrite` tool to create trackable todos that persist in the UI/output.

### Strategy 4: Automated Context Updates

Use hooks to automatically update context after sessions:

```typescript
const session = query({
  prompt: "Implement user dashboard",
  options: {
    cwd: process.cwd(),
    hooks: {
      SessionEnd: [{
        hooks: [async (input) => {
          // Automatically create session summary
          const summarySession = query({
            prompt: `Create a summary of this session in SESSION_SUMMARY_${new Date().toISOString().split('T')[0]}.md. Include what was accomplished, files modified, and next steps.`,
            options: {
              cwd: process.cwd(),
              resume: input.session_id
            }
          });
          
          for await (const msg of summarySession) {
            // Let it run
          }
          
          return { continue: true };
        }]
      }]
    }
  }
});
```

### Strategy 5: Context Files in .claude/ Directory

Create a `.claude/` directory for context files:

```
project/
  ├── .claude/
  │   ├── context.md           # Main project context
  │   ├── architecture.md      # Architecture decisions
  │   ├── progress.md          # Progress tracking
  │   ├── sessions/            # Session summaries
  │   │   ├── 2024-01-15.md
  │   │   ├── 2024-01-16.md
  │   │   └── ...
  │   └── decisions/           # Decision logs
  │       ├── auth-strategy.md
  │       └── database-choice.md
  ├── .claudeconfig            # Claude configuration
  ├── src/
  └── ...
```

Add to `.gitignore` if you don't want to commit:
```
.claude/sessions/
```

### Template: Comprehensive Context File

```markdown
<!-- .claude/context.md -->
# Project Context: [Project Name]

## Quick Reference
- **Project Type**: [Web App / API / CLI / Library / etc.]
- **Tech Stack**: [React, Node.js, PostgreSQL, etc.]
- **Started**: [Date]
- **Team**: [Solo / Team members]

## Project Overview
[Brief description of what this project does]

## Current Status
**Phase**: [Planning / Development / Testing / Maintenance]

### Completed
- ✅ [Feature/task]
- ✅ [Feature/task]

### In Progress
- 🚧 [Feature/task]
- 🚧 [Feature/task]

### Planned
- ⏳ [Feature/task]
- ⏳ [Feature/task]

## Architecture
### Structure
```
[Directory structure or component diagram]
```

### Key Components
- **[Component Name]**: [Description]
- **[Component Name]**: [Description]

### Data Flow
[How data moves through the system]

## Important Decisions
1. **[Decision Topic]**: [What was decided and why]
2. **[Decision Topic]**: [What was decided and why]

## Known Issues
- [ ] [Issue description] (Priority: High/Med/Low)
- [ ] [Issue description] (Priority: High/Med/Low)

## Dependencies
- [Package name] - [Why we use it]
- [Package name] - [Why we use it]

## Environment
```bash
NODE_ENV=development
DATABASE_URL=postgresql://...
API_KEY=...
```

## Testing
- **Unit Tests**: [Location / Status]
- **Integration Tests**: [Location / Status]
- **E2E Tests**: [Location / Status]

## Deployment
- **Staging**: [URL / Status]
- **Production**: [URL / Status]

## Resources
- [Documentation link]
- [Design files]
- [API documentation]

## Recent Sessions
- [Date]: [What was accomplished] (Session: session-id)
- [Date]: [What was accomplished] (Session: session-id)

## Notes for Claude
[Specific instructions or context that helps Claude understand the project better]
- Prefer [pattern/style]
- Avoid [antipattern]
- Always [requirement]
```

---

## Best Practices

### 1. Always Capture Session IDs

```typescript
// Good: Save session ID for later
let sessionId: string;
for await (const message of session) {
  if (message.session_id) {
    sessionId = message.session_id;
    console.log('Session ID:', sessionId);
  }
}

// Better: Save to file
await fs.writeFile('.last-session', sessionId);

// Best: Track all sessions with metadata
await saveSessionMetadata({
  id: sessionId,
  timestamp: new Date().toISOString(),
  prompt: userPrompt,
  status: 'completed'
});
```

### 2. Create Context Files Early

Start every new project with context:

```bash
# First session with a new project
claude "Read PROJECT_CONTEXT.md if it exists. If not, help me create one based on our conversation about building a [description]. Then help me get started."
```

### 3. Update Context Regularly

Don't let context get stale:

```bash
# After major milestones
claude "Update PROJECT_CONTEXT.md with our progress. We completed [features] and decided to [decisions]."

# Weekly updates
claude "Review and update all files in .claude/ directory to reflect current state"
```

### 4. Use Descriptive Session Prompts

Help yourself remember what each session was about:

```typescript
// Bad: Vague prompt
query({ prompt: "help me" })

// Good: Descriptive prompt
query({ prompt: "Implement user authentication with JWT tokens" })

// Better: Descriptive with context
query({ prompt: "Continue authentication work from session-abc. Now add refresh token rotation." })
```

### 5. Leverage .claudeconfig

Store project-specific configuration:

```json
{
  "permissionMode": "acceptEdits",
  "additionalDirectories": [
    "./docs",
    "./.claude"
  ],
  "allowedTools": [
    "Read",
    "Write",
    "Edit",
    "Bash",
    "Grep",
    "Glob",
    "TodoWrite"
  ],
  "maxTurns": 100
}
```

This ensures consistent behavior across sessions.

### 6. Create a Session Workflow

Establish a routine:

```bash
# Start of session
1. claude "Read .claude/context.md and .claude/progress.md"
2. Work on tasks
3. End of session: claude "Update .claude/progress.md with what we accomplished"

# Or automate it:
# start-session.sh
#!/bin/bash
claude "Read .claude/context.md and .claude/progress.md, then: $1"

# end-session.sh
#!/bin/bash
claude --resume "$(cat .last-session)" "Update .claude/progress.md with what we accomplished in this session, then create a summary in .claude/sessions/$(date +%Y-%m-%d).md"
```

### 7. Use Forking for Experiments

Before trying risky changes:

```typescript
// Fork the session first
const experimentSession = query({
  prompt: "Try refactoring the database layer to use Prisma",
  options: {
    resume: sessionId,
    forkSession: true
  }
});

// Original session remains untouched
// Can resume from it if experiment fails
```

### 8. Document Decisions

Create decision logs:

```markdown
<!-- .claude/decisions/auth-strategy.md -->
# Decision: Authentication Strategy

## Date
2024-01-15

## Context
Need to implement user authentication for the application.

## Options Considered
1. Session-based with cookies
2. JWT tokens
3. OAuth only

## Decision
Using JWT tokens with refresh token rotation

## Rationale
- Stateless authentication
- Better for microservices
- Mobile app support
- Can still be secure with proper refresh token handling

## Consequences
- Need to implement token refresh logic
- Need secure storage for refresh tokens
- Added complexity vs session-based

## Implementation Notes
- Access tokens: 15 minutes
- Refresh tokens: 7 days
- Store refresh tokens in httpOnly cookies
```

---

## Common Patterns

### Pattern 1: Project README as Context Anchor

Keep your main README comprehensive and have Claude reference it:

```markdown
<!-- README.md -->
# My Project

## Quick Start
[Setup instructions]

## For Claude / AI Assistants
See [.claude/context.md](.claude/context.md) for project context and current status.

## [Rest of README]
```

Then in sessions:
```bash
claude "Read README.md and .claude/context.md, then help me with [task]"
```

### Pattern 2: Milestone Summaries

After each major milestone:

```typescript
// Create milestone summary
const milestone = query({
  prompt: `
    We just completed the authentication feature.
    Create a comprehensive summary in docs/milestones/auth-complete.md including:
    - What was implemented
    - All files created/modified
    - How to use it
    - Tests added
    - Known limitations
    - Next steps
  `,
  options: { cwd: process.cwd() }
});
```

### Pattern 3: Daily Progress Logs

Maintain a daily log:

```markdown
<!-- .claude/daily-log.md -->
# Daily Progress Log

## 2024-01-16
- Continued authentication work
- Implemented refresh token rotation
- Added tests for token expiration
- **Session**: session-abc-123
- **Next**: Add rate limiting

## 2024-01-15
- Started authentication feature
- Decided on JWT approach
- Implemented basic login/logout
- **Session**: session-xyz-789
- **Next**: Add refresh tokens
```

### Pattern 4: Architecture Decision Records (ADR)

Follow ADR pattern in `.claude/adr/`:

```markdown
<!-- .claude/adr/001-use-postgresql.md -->
# ADR 001: Use PostgreSQL as Primary Database

## Status
Accepted

## Context
Need to choose a database for the application. Requirements:
- Relational data model
- ACID compliance
- Good performance
- Production-ready

## Decision
Use PostgreSQL as the primary database.

## Consequences
### Positive
- Mature and reliable
- Excellent documentation
- Rich feature set (JSON, full-text search, etc.)
- Strong community support

### Negative
- More complex than SQLite for local development
- Requires separate service/container

## Implementation
- Use Prisma as ORM
- Docker for local development
- Managed PostgreSQL in production (RDS/Supabase)
```

### Pattern 5: Context-Aware Session Starter

Create a script that always provides context:

```bash
#!/bin/bash
# start-claude.sh

SESSION_FILE=".last-session"
CONTEXT_FILE=".claude/context.md"

if [ -f "$SESSION_FILE" ]; then
  LAST_SESSION=$(cat "$SESSION_FILE")
  echo "Last session: $LAST_SESSION"
  echo "Resume? (y/n)"
  read -r RESUME
  
  if [ "$RESUME" = "y" ]; then
    claude --resume "$LAST_SESSION" "$1"
  else
    claude "Read $CONTEXT_FILE first. Then: $1"
  fi
else
  claude "Read $CONTEXT_FILE if it exists. Then: $1"
fi
```

Usage:
```bash
./start-claude.sh "Help me implement user dashboard"
```

### Pattern 6: Automated Backup and Restore

Save session state periodically:

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';
import fs from 'fs/promises';

async function runWithBackup(prompt: string) {
  const backupFile = '.claude/session-backup.json';
  
  // Load last session if exists
  let lastSessionId: string | undefined;
  try {
    const backup = JSON.parse(await fs.readFile(backupFile, 'utf-8'));
    lastSessionId = backup.sessionId;
  } catch {
    // No backup
  }
  
  const session = query({
    prompt,
    options: {
      cwd: process.cwd(),
      resume: lastSessionId,
      hooks: {
        SessionStart: [{
          hooks: [async (input) => {
            // Save session ID
            await fs.writeFile(backupFile, JSON.stringify({
              sessionId: input.session_id,
              timestamp: new Date().toISOString(),
              prompt
            }, null, 2));
            return { continue: true };
          }]
        }]
      }
    }
  });
  
  for await (const message of session) {
    // Process messages
    if (message.type === 'assistant') {
      console.log(message.message.content);
    }
  }
}
```

---

## FAQ

### Q: Does Claude remember things between different projects?

**A:** No. Each project is completely independent. Session history is tied to a specific session ID, which is specific to a project directory. If you want to carry knowledge between projects, you need to:
1. Export context from one project
2. Import it into another project's context files
3. Or use the same `.claudeconfig` across projects

### Q: My friend says Claude automatically creates markdown files. Why doesn't mine?

**A:** Your friend likely:
1. Asked Claude to create these files (e.g., "create a project context file")
2. Manually created them themselves
3. Misremembers or is using a different tool

Claude Agent SDK and Claude Code CLI do **not** automatically create memory/context files. This must be done explicitly.

### Q: How do I connect my claude.ai project to Claude Code?

**A:** You cannot. They are separate systems:
- **claude.ai** has its own memory system tied to web conversations
- **Claude Code/Agent SDK** uses local session management

There is no integration between them. If you want to bring context from claude.ai to Claude Code:
1. Copy relevant conversation parts
2. Create a context file manually
3. Reference it in your prompts

### Q: Where is my session data stored?

**A:** Platform-specific locations:
- **macOS/Linux**: `~/.local/share/claude/sessions/`
- **Windows**: `%LOCALAPPDATA%\claude\sessions\`

Session files are JSON and contain the full message history.

### Q: How long are sessions kept?

**A:** Sessions are kept indefinitely on your local machine until you manually delete them. The SDK/CLI does not automatically clean up old sessions.

### Q: Can I share sessions with team members?

**A:** Technically yes, by copying session files, but:
1. Sessions contain full conversation history (may include sensitive info)
2. File paths may be absolute and not portable
3. Better to share context files and summaries instead

**Better approach:**
```bash
# Instead of sharing sessions, share context
claude "Create a handoff document in HANDOFF.md with everything the next person needs to know"
```

### Q: Does resuming a session cost tokens for the full history?

**A:** Yes. When you resume a session, the full conversation history is sent to Claude's API, which consumes input tokens. For long sessions:
1. Consider forking from a specific point
2. Use context compaction (automatic in SDK)
3. Summarize old context into files instead

### Q: What's the difference between `resume` and `continue`?

**A:**
- **`resume`**: Load a specific session by ID (can resume any previous session)
- **`continue`**: Continue the most recent turn (immediate continuation)

```typescript
// Resume specific session
query({ prompt: "...", options: { resume: "session-abc-123" } })

// Continue current/last turn
query({ prompt: "...", options: { continue: true } })
```

### Q: Can I edit session history?

**A:** Not directly through the SDK API, but:
1. You can fork and use `resumeSessionAt` to skip parts
2. Session files are JSON - you could manually edit (not recommended)
3. Better: Create a new session with summarized context

### Q: How do I prevent Claude from "forgetting" important context?

**A:**
1. **Put it in files**: Create context files that are read each session
2. **Use resume**: Don't start new sessions unnecessarily
3. **Repeat key info**: Include important context in your prompts
4. **Use additionalDirectories**: Make docs always accessible

```typescript
// Ensure Claude can always access context
const options = {
  cwd: process.cwd(),
  additionalDirectories: [
    './docs',
    './.claude',
    './context'
  ]
};
```

### Q: Should I commit `.claude/` to git?

**Depends:**
- **Commit**: Context files, architecture docs, decision logs
- **Don't commit**: Session files, personal notes, API keys

```gitignore
# .gitignore
.claude/sessions/
.claude/personal-notes.md
.last-session
```

### Q: Can I use Claude to maintain its own context?

**A:** Yes! This is a good pattern:

```typescript
// End of each major task
const contextUpdate = query({
  prompt: `
    Update .claude/context.md with:
    1. What we just accomplished
    2. New architecture decisions
    3. Updated status of features
    4. Any new issues discovered
    
    Keep the format consistent with the existing file.
  `,
  options: { resume: sessionId }
});
```

### Q: What happens when I hit the context window limit?

**A:** The SDK automatically handles context compaction:
1. Older messages are summarized
2. Important tool uses are preserved
3. Recent messages kept in full
4. A `PreCompact` hook fires (you can intercept)

You'll see better results by:
- Keeping sessions focused on specific tasks
- Using multiple sessions instead of one mega-session
- Saving important info to files

---

## Summary

**The Reality of Memory in Claude:**
- ✅ Session-based: Full history within a session
- ✅ Resumable: Can continue previous sessions
- ✅ Manual context: You create and maintain context files
- ❌ No automatic memory: Doesn't remember between sessions without resume
- ❌ No auto-files: Doesn't create tracking files automatically
- ❌ No web integration: Separate from claude.ai projects

**Best Approach:**
1. Use session resume for conversation continuity
2. Create context files for project knowledge
3. Establish a workflow for context updates
4. Use hooks to automate context management
5. Document decisions and architecture
6. Keep the README as the entry point

**Remember:** Claude is a powerful assistant, but memory management is your responsibility. Build good habits around context files and session management, and you'll have a smooth experience.

---

## Related Documentation

- [Comprehensive Guide - Session Management](./comprehensive-guide.md#session-management)
- [Hooks & Permissions](./hooks-permissions-complete.md)
- [Tool System - TodoWrite](./tools-complete.md#todowrite)
- [Implementation Gotchas](./implementation-gotchas.md)

