# Codex CLI - System Prompts Internals

> **📚 Official User Guide**: For custom user prompts (slash commands), see [Official prompts.md](../../context/codex/docs/prompts.md)
>
> **🎯 This Document**: Focuses on **internal system prompts** - the base instructions Codex uses, not user custom prompts.

---

## Quick Links

- **User Guide**: `/context/codex/docs/prompts.md` - How to create custom slash commands
- **This Doc**: Internal system prompt architecture for developers
- **Note**: This doc covers system prompts, not user custom prompts

--- & Instructions

## Table of Contents
- [Base System Prompts](#base-system-prompts)
- [AGENTS.md Discovery](#agentsmd-discovery)
- [Custom User Prompts](#custom-user-prompts)
- [Prompt Hierarchy](#prompt-hierarchy)
- [Configuration Options](#configuration-options)

---

## Base System Prompts

### Location and Format

**Directory**: `codex-rs/core/`
**Format**: Markdown files embedded at compile time

### Primary Prompts

#### 1. prompt.md (Regular Mode)

**Size**: 11KB, 311 lines
**Purpose**: Main coding agent instructions
**Location**: `codex-rs/core/prompt.md`

**Key Sections**:

1. **Identity & Capabilities**
   ```markdown
   You are a coding agent running in the Codex CLI, a terminal-based 
   coding assistant. You are expected to be precise, safe, and helpful.
   ```

2. **Personality Guidelines**
   ```markdown
   Your default personality and tone is concise, direct, and friendly. 
   You communicate efficiently, always keeping the user clearly informed 
   about ongoing actions without unnecessary detail.
   ```

3. **AGENTS.md Specification**
   - Scope rules for project documentation
   - Priority and override rules
   - When to check for AGENTS.md files

4. **Responsiveness**
   - Preamble message guidelines
   - Progress update frequency
   - Communication style examples

5. **Planning System**
   - When to use `update_plan` tool
   - High-quality vs low-quality plan examples
   - Plan structure guidelines

6. **Task Execution**
   - Autonomous operation philosophy
   - Code quality guidelines
   - File editing constraints
   - Git workflow rules

7. **Sandbox & Approvals**
   - Filesystem modes: read-only, workspace-write, danger-full-access
   - Network sandboxing policies
   - Approval modes: untrusted, on-failure, on-request, never
   - When to request escalated permissions

8. **Validation**
   - Testing philosophy
   - When to run tests vs suggest them
   - Linting and formatting approach

9. **Final Answer Formatting**
   - Section headers (Title Case, `**wrapped**`)
   - Bullet point guidelines
   - Monospace formatting rules
   - File reference format
   - Tone and brevity requirements

**Example Excerpt**:

```markdown
## Task execution

You are a coding agent. Please keep going until the query is completely 
resolved, before ending your turn and yielding back to the user. Only 
terminate your turn when you are sure that the problem is solved.

You MUST adhere to the following criteria when solving queries:

- Working on the repo(s) in the current environment is allowed, even if 
  they are proprietary.
- Use the `apply_patch` tool to edit files
- Do not attempt to fix unrelated bugs or broken tests
- NEVER add copyright or license headers unless specifically requested
- Do not `git commit` your changes unless explicitly requested
```

#### 2. review_prompt.md (Review Mode)

**Size**: 2.4KB, 87 lines
**Purpose**: Code review guidelines
**Location**: `codex-rs/core/review_prompt.md`

**Key Sections**:

1. **Review Philosophy**
   ```markdown
   You are acting as a reviewer for a proposed code change made by 
   another engineer. Below are guidelines for determining whether the 
   original author would appreciate the issue being flagged.
   ```

2. **What Constitutes a Bug**
   - Impact criteria (accuracy, performance, security, maintainability)
   - Discreteness and actionability
   - Rigor appropriate to codebase
   - Was introduced in this change (not pre-existing)

3. **Finding Severity Levels**
   - P0: Drop everything, blocking release
   - P1: Urgent, next cycle
   - P2: Normal, fix eventually
   - P3: Low, nice to have

4. **Output Format**
   ```json
   {
     "findings": [
       {
         "title": "≤80 chars, imperative",
         "body": "Markdown explaining why",
         "confidence_score": 0.0-1.0,
         "priority": 0-3,
         "code_location": {
           "absolute_file_path": "...",
           "line_range": {"start": 10, "end": 15}
         }
       }
     ],
     "overall_correctness": "patch is correct" | "patch is incorrect"
   }
   ```

#### 3. gpt_5_codex_prompt.md (GPT-5 Specific)

**Size**: 3.7KB, 107 lines
**Purpose**: GPT-5 model-specific instructions
**Location**: `codex-rs/core/gpt_5_codex_prompt.md`

**Differences from base prompt**:

1. **More concise tone requirements**
   ```markdown
   Default: be very concise; friendly coding teammate tone.
   ```

2. **Explicit escalation workflow**
   ```markdown
   When requesting approval to execute a command:
   - Provide `with_escalated_permissions` parameter
   - Include 1 sentence explanation in justification parameter
   ```

3. **Condensed formatting rules**
   - Shorter section on file references
   - Combined tool guidelines
   - Streamlined examples

### Loading System Prompts

**Implementation**: `core/src/client_common.rs`

```rust
pub const PROMPT: &str = include_str!("../prompt.md");
pub const REVIEW_PROMPT: &str = include_str!("../review_prompt.md");
pub const GPT5_PROMPT: &str = include_str!("../gpt_5_codex_prompt.md");

pub fn get_system_prompt(
    task_kind: TaskKind,
    model_family: &ModelFamily,
) -> &'static str {
    match task_kind {
        TaskKind::Regular => {
            if model_family.uses_gpt5_prompt {
                GPT5_PROMPT
            } else {
                PROMPT
            }
        }
        TaskKind::Review => REVIEW_PROMPT,
        TaskKind::Compact => PROMPT, // Same as regular
    }
}
```

---

## AGENTS.md Discovery

### File Discovery Algorithm

**Implementation**: `core/src/project_doc.rs`

```rust
pub fn discover_project_doc_paths(config: &Config) -> io::Result<Vec<PathBuf>> {
    let mut dir = config.cwd.clone();
    
    // Build chain from cwd upwards
    let mut chain: Vec<PathBuf> = vec![dir.clone()];
    let mut git_root: Option<PathBuf> = None;
    
    // Walk up to git root
    let mut cursor = dir;
    while let Some(parent) = cursor.parent() {
        if cursor.join(".git").exists() {
            git_root = Some(cursor.clone());
            break;
        }
        chain.push(parent.to_path_buf());
        cursor = parent.to_path_buf();
    }
    
    // Search from git root down to cwd
    let search_dirs = if let Some(root) = git_root {
        chain.iter().rev()
            .skip_while(|p| *p != &root)
            .collect()
    } else {
        vec![config.cwd.clone()]
    };
    
    // Find AGENTS.md in each directory
    let mut found = Vec::new();
    for dir in search_dirs {
        for name in candidate_filenames(config) {
            let candidate = dir.join(name);
            if candidate.exists() && candidate.is_file() {
                found.push(candidate);
                break; // Only one per directory
            }
        }
    }
    
    Ok(found)
}

fn candidate_filenames(config: &Config) -> Vec<&str> {
    let mut names = vec!["AGENTS.override.md", "AGENTS.md"];
    names.extend(config.project_doc_fallback_filenames.iter().map(|s| s.as_str()));
    names
}
```

### Priority Order

1. **AGENTS.override.md** - Local override (not committed to git)
2. **AGENTS.md** - Standard project documentation
3. **Fallback filenames** - Configured in `project_doc_fallback_filenames`

### Example Directory Structure

```
/project/                       # Git root
├── .git/
├── AGENTS.md                   ← Loaded (root level)
│   └── "Use 2-space indentation"
│
├── backend/
│   ├── AGENTS.md               ← Loaded (backend specific)
│   │   └── "Always use async/await"
│   │
│   └── services/
│       └── auth/               # Current working directory
│           └── AGENTS.override.md  ← Loaded (most specific, overrides all)
│               └── "Use JWT for auth"
```

**Result**: All three files concatenated in order (root → backend → auth)

### Content Assembly

```rust
pub async fn get_user_instructions(config: &Config) -> Option<String> {
    let paths = discover_project_doc_paths(config).ok()?;
    let mut parts: Vec<String> = Vec::new();
    
    for path in paths {
        let content = tokio::fs::read_to_string(&path).await.ok()?;
        if !content.trim().is_empty() {
            parts.push(content);
        }
    }
    
    if parts.is_empty() {
        return config.user_instructions.clone();
    }
    
    let project_doc = parts.join("\n\n");
    
    match &config.user_instructions {
        Some(base) => Some(format!(
            "{}\n\n--- project-doc ---\n\n{}",
            base,
            project_doc
        )),
        None => Some(project_doc),
    }
}
```

### Configuration Options

```yaml
# ~/.codex/config.yaml
project_doc_max_bytes: 32768  # Max 32KB combined

project_doc_fallback_filenames:
  - ".codex.md"
  - "CONTRIBUTING.md"
```

**Environment Variable**:
```bash
export CODEX_DISABLE_PROJECT_DOC=1  # Disable AGENTS.md loading
```

**CLI Flag**:
```bash
codex --no-project-doc "your prompt"
```

### Example AGENTS.md

```markdown
# Project: MyApp Backend

## Architecture
- Microservices architecture
- Each service in `services/` directory
- Shared code in `common/`

## Code Style
- Use 2-space indentation
- Prefer async/await over callbacks
- TypeScript strict mode enabled

## Testing
- Jest for unit tests
- Supertest for API tests
- Tests live next to source: `*.test.ts`
- Run with: `npm test`

## Database
- PostgreSQL via Prisma ORM
- Migrations in `prisma/migrations/`
- Never write raw SQL, use Prisma

## API Design
- REST endpoints follow `/api/v1/:resource` pattern
- Use HTTP status codes correctly
- Always return JSON with `{success, data, error}` shape

## Security
- All routes require authentication unless in `PUBLIC_ROUTES`
- Use bcrypt for password hashing (10 rounds)
- JWT tokens expire in 7 days
```

---

## Custom User Prompts

### Directory Structure

```
~/.codex/prompts/
├── perf.md           # Performance review
├── security.md       # Security audit
├── test.md           # Add tests
├── refactor.md       # Refactoring
└── docs.md           # Add documentation
```

### Prompt File Format

**Basic Format**:

```markdown
Review this code for performance issues.
Look for:
- O(n²) or worse algorithms
- Unnecessary allocations
- Missing caching opportunities
- Inefficient data structures
```

**With Frontmatter**:

```markdown
---
description: "Performance review and optimization"
argument-hint: "[file_path]"
---

Review the file at $1 for performance issues.
Consider:
- Time complexity of algorithms
- Space complexity and allocations
- Caching opportunities
- Database query efficiency
```

### Variable Substitution

- `$1`, `$2`, `$3`, ... - Positional arguments
- `$ARGUMENTS` - All arguments concatenated

**Example Usage**:

```bash
# In terminal
/perf src/api/users.ts

# Expands to:
# Review the file at src/api/users.ts for performance issues...
```

### Discovery and Loading

**Implementation**: `core/src/custom_prompts.rs`

```rust
pub async fn discover_prompts_in(dir: &Path) -> Vec<CustomPrompt> {
    let mut prompts = Vec::new();
    let mut entries = fs::read_dir(dir).await.ok()?;
    
    while let Some(entry) = entries.next_entry().await.ok()? {
        let path = entry.path();
        
        // Only .md files
        if path.extension()? != "md" {
            continue;
        }
        
        let name = path.file_stem()?.to_str()?.to_string();
        let content = fs::read_to_string(&path).await.ok()?;
        
        // Parse frontmatter
        let (description, argument_hint, body) = parse_frontmatter(&content);
        
        prompts.push(CustomPrompt {
            name,
            path,
            content: body,
            description,
            argument_hint,
        });
    }
    
    prompts.sort_by(|a, b| a.name.cmp(&b.name));
    prompts
}
```

### Frontmatter Parsing

```rust
fn parse_frontmatter(content: &str) -> (Option<String>, Option<String>, String) {
    if !content.starts_with("---\n") {
        return (None, None, content.to_string());
    }
    
    let mut lines = content.lines();
    lines.next(); // Skip opening ---
    
    let mut desc = None;
    let mut hint = None;
    let mut body_start = 0;
    
    for line in lines {
        if line.trim() == "---" {
            break;
        }
        
        if let Some((key, value)) = line.split_once(':') {
            let key = key.trim().to_lowercase();
            let value = value.trim().trim_matches('"').trim_matches('\'');
            
            match key.as_str() {
                "description" => desc = Some(value.to_string()),
                "argument-hint" | "argument_hint" => hint = Some(value.to_string()),
                _ => {}
            }
        }
        
        body_start += line.len() + 1;
    }
    
    let body = &content[body_start..];
    (desc, hint, body.to_string())
}
```

---

## Prompt Hierarchy

### Priority Rules

**Highest to Lowest Priority**:

1. **Direct user instructions** in current message
2. **Developer messages** (environment context)
3. **AGENTS.override.md** (most specific, not in git)
4. **AGENTS.md** (more specific directory)
5. **AGENTS.md** (less specific directory)
6. **AGENTS.md** (git root)
7. **Config `user_instructions`**
8. **Base system prompt** (prompt.md)

### Conflict Resolution

From `prompt.md`:

```markdown
## AGENTS.md spec

- Instructions in AGENTS.md files:
  - More-deeply-nested AGENTS.md files take precedence in the case of 
    conflicting instructions.
  - Direct system/developer/user instructions (as part of a prompt) 
    take precedence over AGENTS.md instructions.
```

**Example**:

```
Root AGENTS.md:     "Use 4-space indentation"
Backend AGENTS.md:  "Use 2-space indentation"
Current prompt:     "Use tabs for indentation"

Result: Tabs (direct instruction wins)
```

### Scope Rules

```markdown
- The scope of an AGENTS.md file is the entire directory tree rooted 
  at the folder that contains it.
- For every file you touch in the final patch, you must obey 
  instructions in any AGENTS.md file whose scope includes that file.
```

**Example**:

```
/project/
├── AGENTS.md (scope: entire project)
└── frontend/
    ├── AGENTS.md (scope: frontend/ only)
    └── components/
        └── Button.tsx

When editing Button.tsx:
- Apply instructions from /project/AGENTS.md
- Apply instructions from /project/frontend/AGENTS.md
- If conflict, frontend/AGENTS.md wins
```

---

## Configuration Options

### Config File Settings

```yaml
# ~/.codex/config.yaml

# Base user instructions (always included)
user_instructions: |
  Always write clear commit messages.
  Prefer functional programming patterns.

# Project documentation settings
project_doc_max_bytes: 32768  # 32KB limit

# Fallback filenames (checked if AGENTS.md missing)
project_doc_fallback_filenames:
  - ".codex.md"
  - "CONTRIBUTING.md"
  - "DEVELOPMENT.md"
```

### Environment Variables

```bash
# Disable AGENTS.md loading completely
export CODEX_DISABLE_PROJECT_DOC=1

# Change prompts directory
export CODEX_PROMPTS_DIR="$HOME/my-prompts"
```

### CLI Flags

```bash
# Disable project docs for this session
codex --no-project-doc "your prompt"

# Use specific model (may select different base prompt)
codex --model gpt-4 "your prompt"
```

---

## Best Practices

### Writing AGENTS.md

1. **Be Specific**
   ```markdown
   ❌ "Write good code"
   ✅ "Use 2-space indentation. Prefer const over let."
   ```

2. **Provide Context**
   ```markdown
   ## Architecture
   This is a monorepo. Each package is independent.
   
   ## Testing
   Run tests with: npm test
   All tests should pass before committing.
   ```

3. **Include Commands**
   ```markdown
   ## Building
   - Development: npm run dev
   - Production: npm run build
   - Output: dist/
   ```

4. **Scope Appropriately**
   - Root AGENTS.md: Project-wide conventions
   - Module AGENTS.md: Module-specific details
   - Don't repeat root instructions

### Writing Custom Prompts

1. **Use Frontmatter**
   ```markdown
   ---
   description: "Security audit"
   argument-hint: "[scope: api|auth|all]"
   ---
   ```

2. **Support Arguments**
   ```markdown
   Audit $1 for security issues.
   Focus on: $ARGUMENTS
   ```

3. **Be Clear and Actionable**
   ```markdown
   Review for:
   - SQL injection vulnerabilities
   - XSS attack vectors
   - CSRF protection
   - Authentication bypass
   ```

---

## Related Documentation

- [03-prompt-processing.md](./03-prompt-processing.md) - How prompts are assembled
- [08-configuration.md](./08-configuration.md) - All config options
- [15-code-reference.md](./15-code-reference.md) - Implementation details

