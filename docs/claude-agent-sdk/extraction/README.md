# Claude Agent SDK - Complete Extraction Documentation

**SDK Version**: 0.1.22

---

## 📚 Overview

This directory contains comprehensive extraction documentation for the Claude Agent SDK and Claude Code CLI. All information has been extracted from source code, TypeScript definitions, and implementation analysis.

**Target Audiences**:
- SDK developers building custom agents
- Claude Code users learning advanced features
- AI agents consuming documentation programmatically
- Technical writers and documentation contributors

---

## 📖 Complete Documentation Index

### ✅ Core Features (6 documents - COMPLETE)

#### 1. [Internal Constants & Implementations](./cli-internal-constants.md)

**What's Inside**:
- All tool limits (Read: 2000 lines, Bash: 30K output, PDF: 32MB)
- Timeout constants (Bash: 2 min, Hook: 5 sec)
- Agent colors (8 colors with deterministic assignment)
- Ultrathink configuration (31,999 tokens)
- Storage & cache limits (1000 files max)
- Performance characteristics (execution times, memory usage)
- Implementation patterns (read-before-write, edit uniqueness)

**Key Discoveries**:
- Silent output truncation (major gotcha!)
- Hook timeout limitations
- Bundle size implications (9.7MB)
- Token efficiency patterns

**Perfect For**: Understanding SDK limits, performance tuning, debugging edge cases

---

#### 2. [Hooks System Complete](./hooks-system-complete.md)

**What's Inside**:
- **All 9 Hook Events**: PreToolUse, PostToolUse, Notification, UserPromptSubmit, SessionStart, SessionEnd, Stop, SubagentStop, PreCompact
- Hook input/output structures for each event
- Hook execution lifecycle
- Permission override mechanism
- 5 advanced patterns (security, sanitization, logging, injection, async)
- 2 real-world examples (enterprise, dev workflow)
- Complete gotchas and best practices

**Key Discoveries**:
- PreToolUse can override all permissions
- Hook overrides bypass entire permission system
- 5-second timeout (strict enforcement)
- Async hooks for fire-and-forget

**Perfect For**: Building custom security, logging, monitoring, or workflow automation

---

#### 3. [Agents & Subagents Complete](./agents-subagents-complete.md)

**What's Inside**:
- **All 5 Built-in Agents**:
  - Explore (Haiku, isolated, 70-84% token savings)
  - general-purpose (Sonnet, forked, full tools)
  - statusline-setup (Sonnet, isolated, config only)
  - output-style-setup (Sonnet, isolated, style creation)
  - security-review (Sonnet, isolated, git-restricted)
- Agent definition structure (complete TypeScript schema)
- Context management (forked vs isolated)
- Agent color system (8 colors, deterministic)
- Async agent execution
- Custom agent creation (3 methods)
- Token optimization strategies
- Real-world patterns

**Key Discoveries**:
- Explore agent: Massive token savings (70-84%)
- security-review: Git-only bash commands
- Agent colors: Hash-based deterministic assignment
- Forked context: No token savings (actually adds overhead)

**Perfect For**: Building efficient multi-agent workflows, token optimization, agent architecture

---

#### 4. [Permissions System Complete](./permissions-system-complete.md)

**What's Inside**:
- **4 Permission Modes**: default, acceptEdits, bypassPermissions, plan
- **6-Level Resolution**: Session → Local → Project → User → Policy → Mode
- Permission rules (structure, matching, patterns)
- Permission updates (6 types)
- Permission suggestions mechanism
- Hook permission override
- Real-world patterns (CI/CD, dev, secure review)

**Key Discoveries**:
- First match wins (strict precedence order)
- bypassPermissions: Dangerous but useful for CI/CD
- Hook overrides bypass ALL rules (security consideration)
- Plan mode: Perfect for safe exploration

**Perfect For**: Security configuration, team workflows, CI/CD setup, permission troubleshooting

---

#### 5. [Skills System Complete](./skills-system-complete.md)

**What's Inside**:
- Skill file format (SKILL.md structure)
- Complete frontmatter schema (9 fields)
- Skill discovery (5 locations, precedence order)
- Skill execution flow
- Template variables ($ARGUMENTS)
- Skills vs slash commands comparison
- Built-in skills catalog
- Custom skill creation (step-by-step)
- 3 real-world examples (code review, test gen, API docs)

**Key Discoveries**:
- File name MUST be `SKILL.md` (case-sensitive, strict)
- Discovery from 5 locations (not hot-reloaded)
- Template variable: `{{$ARGUMENTS}}` only
- Skills more flexible than slash commands

**Perfect For**: Creating reusable workflows, team productivity, custom commands

---

#### 6. [Configuration Complete](./configuration-complete.md)

**What's Inside**:
- Settings file locations (6 levels)
- Settings resolution order (7 levels with CLI flags)
- Complete settings.json schema
- Environment variables (all core vars)
- CLI flags (complete reference)
- Programmatic options (SDK API)
- 3 configuration examples (team, personal, enterprise)

**Key Discoveries**:
- 7-level precedence: CLI → Session → Local → Project → User → Policy → Defaults
- Not hot-reloaded (restart required)
- Environment variable substitution: `${VAR_NAME}`
- CLI flags override EVERYTHING (including policy)

**Perfect For**: Team setup, enterprise deployment, configuration troubleshooting

---

## 🎯 Documentation Coverage

### ✅ Fully Documented Features

| Feature | Lines | Real-World Examples | Gotchas | Patterns |
|---------|-------|---------------------|---------|----------|
| **Internal Constants** | 561 | 5 | 15 | 3 |
| **Hooks System** | 963 | 2 | 5 | 5 |
| **Agents & Subagents** | 1,071 | 3 | 6 | 3 |
| **Permissions** | 922 | 4 | 6 | 4 |
| **Skills** | 868 | 3 | 6 | - |
| **Configuration** | 843 | 3 | 5 | - |
| **MCP Integration** | 1,142 | 3 | 6 | - |
| **Tool System** | 1,547 | 5 | 10 | 3 |
| **TOTAL** | **7,917** | **28** | **59** | **18** |

### 🔄 In Progress

- MCP Integration Complete
- Architecture Overview Complete
- Tool System Complete (all 17 tools)
- Type System Complete
- CLI Bundle Analysis

### 📋 Planned

- Quick Start Guide
- User Guide (CLI)
- SDK Developer Guide
- API Reference
- Troubleshooting Guide
- Examples Library

---

## 🚀 Quick Navigation

### By Use Case

**I want to...**

- **Understand SDK limits** → [Internal Constants](./cli-internal-constants.md)
- **Build custom security/logging** → [Hooks System](./hooks-system-complete.md)
- **Optimize token usage** → [Agents & Subagents](./agents-subagents-complete.md)
- **Configure permissions** → [Permissions System](./permissions-system-complete.md)
- **Create reusable commands** → [Skills System](./skills-system-complete.md)
- **Set up team/enterprise config** → [Configuration](./configuration-complete.md)

### By Role

**Claude Code User**:
1. Start: [Configuration](./configuration-complete.md)
2. Then: [Agents & Subagents](./agents-subagents-complete.md)
3. Then: [Skills System](./skills-system-complete.md)

**SDK Developer**:
1. Start: [Hooks System](./hooks-system-complete.md)
2. Then: [Permissions System](./permissions-system-complete.md)
3. Then: [Internal Constants](./cli-internal-constants.md)

**Enterprise Admin**:
1. Start: [Configuration](./configuration-complete.md)
2. Then: [Permissions System](./permissions-system-complete.md)
3. Then: [Agents & Subagents](./agents-subagents-complete.md)

**AI Agent/Bot**:
- All documents optimized for programmatic consumption
- Structured schemas included
- Type definitions provided
- Examples with expected outputs

---

## 🔍 Key Insights & Discoveries

### Critical Gotchas (Top 10)

1. **Silent Output Truncation**
   - Bash: 30,000 chars (no warning!)
   - Read: 2,000 chars per line (no ellipsis!)
   - **Workaround**: Redirect to file, then Read

2. **Read-Before-Write Enforcement**
   - Edit/Write tools REQUIRE prior Read in same session
   - No bypass mechanism
   - Session-scoped, not persisted

3. **Edit Tool Uniqueness**
   - `old_string` must be unique in file
   - Exact match (whitespace-sensitive)
   - **Workaround**: Use `replace_all: true` or add context

4. **Hook Timeout (5 seconds)**
   - Strict enforcement
   - Hook killed after timeout
   - No retry mechanism
   - **Best Practice**: Keep hooks <1 second

5. **TodoWrite Single In-Progress**
   - Exactly ONE task must be `in_progress`
   - Strictly enforced
   - No parallel task tracking

6. **Permission Hook Override**
   - Hooks can bypass ALL permission rules
   - Security consideration
   - Use with caution

7. **Skills Not Hot-Reloaded**
   - Changing SKILL.md requires restart
   - No automatic reload

8. **Agent Context Forking**
   - Forked context: NO token savings (adds overhead!)
   - Isolated context: 70-84% savings

9. **Configuration Not Hot-Reloaded**
   - settings.json changes require restart
   - Session settings cleared on exit

10. **CLI Flags Override Everything**
    - Even policy settings
    - Can bypass security (be careful!)

### Token Optimization Strategies

| Strategy | Savings | Use Case |
|----------|---------|----------|
| **Explore Agent** | 70-84% | Codebase discovery |
| **Isolated Context** | 70% | Independent tasks |
| **Haiku Model** | 40-60% | Simple operations |
| **Parallel Async Agents** | N/A (time) | Long-running tasks |
| **Forked Context** | 0% (overhead!) | Context-aware tasks |

### Performance Benchmarks

| Operation | Typical Time | Notes |
|-----------|-------------|-------|
| Read (cached) | 1-5ms | Very fast |
| Read (first time) | 10-50ms | File I/O |
| Write | 5-20ms | Atomic |
| Edit | 10-30ms | Read + validate + write |
| Glob | 50-500ms | Pattern complexity |
| Grep | 100-2000ms | Fast (ripgrep) |
| Bash | Variable | Command-dependent |
| Task (sync, Explore) | 5-15s | Haiku, isolated |
| Task (sync, general) | 30-120s | Sonnet, forked |
| Task (async) | ~100ms | Returns immediately |

---

## 📊 Documentation Statistics

### Extraction Phase 1

- **Documents Created**: 6 core feature docs
- **Total Lines**: 5,228 lines
- **Total Words**: ~47,000 words
- **Code Examples**: 50+ examples
- **Real-World Patterns**: 20 patterns
- **Gotchas Documented**: 43 critical issues

---

## 🎯 Next Steps

### Phase 1 (Extraction) - In Progress

Remaining documents to create:
1. MCP Integration Complete
2. Architecture Overview Complete
3. Tool System Complete (all 17 tools)
4. Type System Complete
5. CLI Bundle Analysis

### Phase 2 (Verification) - Not Started

- Cross-reference all docs against source code
- Verify all code examples compile
- Check consistency across documents
- Validate version numbers and API references

### Phase 3 (User Documentation) - Not Started

- Quick Start Guide (installation, first agent, config)
- User Guide (getting started → advanced)
- SDK Developer Guide (concepts → deployment)
- Examples Library (basic → real-world)
- Troubleshooting Guide (errors → solutions)

---

## 🤝 Contributing

These documents are extraction-only (source code analysis). For user-facing documentation:
- See `docs/user-guide/` (TBD)
- See `docs/sdk-guide/` (TBD)
- See `docs/examples/` (TBD)

---

## 📝 Version History

| Version | Date | Changes |
|---------|------|---------|
| **1.0** | 2025-10-24 | Initial extraction: 6 core documents |
| 0.1 | 2025-10-24 | Project start, plan created |

---

## 📖 Related Documentation

- [Complete Documentation Plan](../COMPREHENSIVE-DOCUMENTATION-PLAN.md)
- [Documentation Progress](../DOCUMENTATION-PROGRESS.md)
- [Main README](../../README.md)

---

**SDK Version**: 0.1.22

