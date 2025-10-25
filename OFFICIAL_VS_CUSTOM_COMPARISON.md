# Official vs Custom Documentation - Content Comparison

**Date**: October 25, 2025
**Purpose**: Show exact differences between official and custom documentation

---

## How to Read This Document

This document compares **content** (not metadata) between official and custom docs:
- **Left Column**: What official docs cover
- **Right Column**: What custom docs cover
- **✅ Unique**: Content only in one source
- **⚠️ Overlap**: Content appears in both (minimal)

---

## 1. Configuration

| **Official: config.md (49KB)** | **Custom: 08-configuration.md (15KB)** |
|--------------------------------|----------------------------------------|
| **User-Facing Features** | **Implementation Details** |
| | |
| ✅ How to set model via CLI | ✅ `Config` struct definition in Rust |
| ✅ Environment variables list | ✅ Config loading hierarchy in code |
| ✅ Model provider configuration syntax | ✅ Platform-specific path resolution |
| ✅ MCP server configuration examples | ✅ `ConfigOverrides` struct |
| ✅ Profiles and how to use them | ✅ TOML parsing implementation |
| ✅ Network tuning options | ✅ Priority system internals |
| ✅ Reasoning effort/summary settings | ✅ Environment variable application logic |
| ✅ Azure provider example | ⚠️ Config file format (both have this) |
| ✅ Query parameters for providers | ⚠️ Environment variables (both have this) |
| ✅ Complete config reference table | |
| | |
| **Content Type**: What to configure | **Content Type**: How config system works |
| **Target Audience**: End users | **Target Audience**: Developers |

**Duplication**: ~10% (basic config file format concepts)
**Unique Value**: 90% - Custom shows implementation, official shows usage

---

## 2. Authentication

| **Official: authentication.md (3.3KB)** | **Custom: 13-authentication.md (16KB)** |
|-----------------------------------------|-----------------------------------------|
| **User Instructions** | **Implementation Architecture** |
| | |
| ✅ How to run `codex login` | ✅ `AuthManager` struct definition |
| ✅ API key setup instructions | ✅ OAuth2 device flow complete code |
| ✅ Headless machine workarounds | ✅ Token refresh logic implementation |
| ✅ SSH port forwarding guide | ✅ Auth storage with file permissions |
| ✅ Copying auth.json between machines | ✅ Refresh lock mechanism |
| ✅ VPS connection instructions | ✅ `AuthMode` enum variants |
| ⚠️ API key vs ChatGPT login options | ⚠️ Authentication methods overview |
| | ✅ Polling logic with retry mechanism |
| | ✅ Token expiry handling |
| | ✅ Security: Unix 0600 file mode |
| | ✅ Concurrent refresh prevention |
| | |
| **Content Type**: How to log in | **Content Type**: How auth system works |
| **Target Audience**: End users | **Target Audience**: Developers |

**Duplication**: ~5% (basic auth method concepts)
**Unique Value**: 95% - Completely different focus

---

## 3. Sandbox & Security

| **Official: sandbox.md (6.1KB)** | **Custom: 07-security-sandboxing.md (17KB)** |
|----------------------------------|-----------------------------------------------|
| **Usage Guide** | **Implementation Deep-Dive** |
| | |
| ✅ Sandbox modes explained (read-only, workspace-write, full-access) | ✅ Complete defense-in-depth architecture |
| ✅ Approval policies guide | ✅ Seatbelt .sbpl profile syntax |
| ✅ Defaults and recommendations | ✅ Landlock API usage in Rust |
| ✅ Common combinations table | ✅ `SandboxMode` enum implementation |
| ✅ `--full-auto` flag usage | ✅ Path validation logic |
| ✅ Platform differences overview | ✅ Network restriction implementation |
| ✅ Testing sandbox with CLI | ✅ Approval flow state machine |
| ⚠️ Sandbox modes (conceptual overlap) | ⚠️ Sandbox modes (implementation) |
| ⚠️ Approval policies (usage) | ⚠️ Approval policies (internals) |
| | ✅ macOS Seatbelt complete profile |
| | ✅ Linux Landlock complete code |
| | ✅ 5-layer security model |
| | ✅ Resource limits implementation |
| | ✅ Command safety checks |
| | ✅ Writable roots validation |
| | |
| **Content Type**: When and how to use | **Content Type**: How sandboxing works |
| **Target Audience**: End users | **Target Audience**: Security auditors, developers |

**Duplication**: ~15% (basic concepts of modes and policies)
**Unique Value**: 85% - Custom goes deep into OS-level implementation

---

## 4. Prompts vs System Prompts

| **Official: prompts.md (2.9KB)** | **Custom: 05-system-prompts.md (16KB)** |
|----------------------------------|------------------------------------------|
| **Custom User Prompts (Slash Commands)** | **Internal System Prompts** |
| | |
| ✅ How to create custom slash commands | ✅ Base system prompt structure |
| ✅ Markdown file format | ✅ `prompt.md` (regular mode) internals |
| ✅ Frontmatter syntax (description, argument-hint) | ✅ `prompt.exec.md` (exec mode) internals |
| ✅ Placeholders: $1-$9, $ARGUMENTS | ✅ AGENTS.md discovery mechanism |
| ✅ Named placeholders: $FILE, $TICKET_ID | ✅ System prompt composition |
| ✅ File location: `~/.codex/prompts/` | ✅ Prompt caching optimization |
| ✅ Running prompts via `/prompts:<name>` | ✅ Context window management |
| ✅ Example: Draft PR helper | ✅ Token budget allocation |
| | ✅ Personality guidelines in system prompt |
| | ✅ Responsiveness rules |
| | ✅ Tool usage policies embedded |
| | ✅ Prompt hierarchy and merging |
| | |
| **Content Type**: User extensibility | **Content Type**: Internal instructions |
| **Target Audience**: Users creating commands | **Target Audience**: Developers understanding system |

**Duplication**: ~0% - Completely different topics!
**Unique Value**: 100% - No overlap at all

---

## 5. MCP

| **Official: advanced.md (5.7KB, MCP section)** | **Custom: 14-mcp-integration.md (16KB)** |
|------------------------------------------------|-------------------------------------------|
| **Usage and Setup** | **Implementation Architecture** |
| | |
| ✅ MCP client configuration syntax | ✅ `McpConnectionManager` structure |
| ✅ MCP server mode usage (`codex mcp-server`) | ✅ Connection lifecycle management |
| ✅ MCP inspector quickstart | ✅ Tool/resource discovery protocol |
| ✅ `codex` tool parameters | ✅ Message routing internals |
| ✅ `codex-reply` tool parameters | ✅ Server process spawning |
| ✅ Timeout configuration tip | ✅ Stdio/SSE transport handling |
| ✅ Example: tic-tac-toe demo | ✅ Tool registration in registry |
| ⚠️ What MCP is (conceptual) | ⚠️ MCP protocol overview |
| | ✅ Connection state management |
| | ✅ Error handling and reconnection |
| | ✅ Resource caching strategies |
| | ✅ Tool call serialization |
| | |
| **Also in Custom: 18-mcp-development.md** | |
| | ✅ Building custom MCP servers guide |
| | ✅ Testing MCP servers |
| | ✅ Debugging strategies |
| | |
| **Content Type**: How to use MCP | **Content Type**: How MCP works internally |
| **Target Audience**: Users adding MCP | **Target Audience**: Developers, MCP server builders |

**Duplication**: ~10% (basic MCP concept explanation)
**Unique Value**: 90% - Custom goes deep into implementation

---

## 6. Execution Modes

| **Official: exec.md (5.2KB)** | **Custom: N/A** |
|-------------------------------|-----------------|
| **Non-Interactive Mode Guide** | |
| | |
| ✅ `codex exec` usage | ❌ Not covered in custom docs |
| ✅ Default output mode | ❌ Not covered in custom docs |
| ✅ JSON output mode (`--json`) | ❌ Not covered in custom docs |
| ✅ Structured output (`--output-schema`) | ❌ Not covered in custom docs |
| ✅ Git repository requirement | ❌ Not covered in custom docs |
| ✅ Resuming sessions (`exec resume`) | ❌ Not covered in custom docs |
| ✅ Authentication with `CODEX_API_KEY` | ❌ Not covered in custom docs |
| | |
| **Content Type**: Non-interactive usage | **Content Type**: N/A |
| **Target Audience**: CI/CD users | **Target Audience**: N/A |

**Duplication**: 0% - Custom docs don't cover `codex exec`
**Gap Identified**: Custom docs should add exec mode implementation details

---

## 7. Getting Started

| **Official: getting-started.md (5.8KB)** | **Custom: 01-overview.md (6.9KB)** |
|------------------------------------------|-------------------------------------|
| **User Onboarding** | **System Overview** |
| | |
| ✅ CLI usage table | ✅ High-level architecture diagram |
| ✅ Resuming sessions guide | ✅ Core components list |
| ✅ Running with prompt as input | ✅ Tool system overview |
| ✅ Example prompts (7 examples) | ✅ Conversation flow |
| ✅ AGENTS.md usage | ✅ State persistence overview |
| ✅ Tips & shortcuts | ⚠️ Basic command usage |
| ✅ `@` file search | |
| ✅ Esc-Esc to edit previous | |
| ✅ `--cd` flag usage | |
| ✅ `--add-dir` flag | |
| ✅ Shell completions | |
| ✅ Image input | |
| | |
| **Content Type**: Getting started quickly | **Content Type**: Understanding the system |
| **Target Audience**: New users | **Target Audience**: Developers |

**Duplication**: ~20% (basic command concepts)
**Unique Value**: 80% - Different purposes

---

## 8. Topics ONLY in Custom Docs

These topics exist **only** in custom documentation:

### Architecture & Implementation
| Document | Size | Content |
|----------|------|---------|
| **02-architecture.md** | 14KB | Complete system architecture, component interactions, event loops |
| **03-prompt-processing.md** | 16KB | Prompt lifecycle, caching, context management, token optimization |
| **04-llm-integration.md** | 14KB | Provider implementations, streaming, client architecture |
| **09-state-management.md** | 17KB | Session persistence, state recovery, conversation history |
| **10-implementation.md** | 18KB | Entry points, main loops, async patterns, core logic |
| **11-tool-implementations.md** | 18KB | Individual tool code, execution patterns, approval flows |
| **12-ui-layer.md** | 18KB | Terminal rendering, TUI components, event handling |
| **15-code-reference.md** | 14KB | Source file index, module map, navigation guide |

### Advanced Topics
| Document | Size | Content |
|----------|------|---------|
| **16-hidden-features.md** | 17KB | Undocumented commands, experimental flags, internal tools |
| **17-cli-reference.md** | 13KB | Complete CLI command catalog with internals |
| **18-mcp-development.md** | 19KB | Building custom MCP servers, testing, debugging |
| **19-performance.md** | 14KB | Optimization techniques, cost reduction, speed tuning |
| **20-state-management-practical.md** | 11KB | Practical state patterns and examples |
| **21-tool-system-practical.md** | 11KB | Practical tool patterns and examples |

**Total Unique Content**: ~215KB of developer-focused technical documentation

---

## 9. Topics ONLY in Official Docs

These topics exist **only** in official documentation:

| Document | Size | Content |
|----------|------|---------|
| **exec.md** | 5.2KB | Non-interactive `codex exec` mode - usage guide |
| **faq.md** | 2.8KB | Frequently asked questions |
| **install.md** | 1.5KB | Installation instructions |
| **contributing.md** | 5.0KB | How to contribute to Codex |
| **release_management.md** | 1.8KB | Release process |
| **CLA.md** | 2.0KB | Contributor License Agreement |

**Total User-Facing Only**: ~18.3KB of official user/contributor documentation

---

## Summary Matrix

| Topic | Official Docs | Custom Docs | Overlap | Unique Value |
|-------|---------------|-------------|---------|--------------|
| **Configuration** | 49KB | 15KB | 10% | Custom: Implementation details |
| **Authentication** | 3.3KB | 16KB | 5% | Custom: OAuth2 internals |
| **Sandbox** | 6.1KB | 17KB | 15% | Custom: Seatbelt/Landlock code |
| **Prompts** | 2.9KB (user) | 16KB (system) | 0% | Different topics entirely |
| **MCP** | 5.7KB | 16KB + 19KB | 10% | Custom: Implementation + development |
| **Exec Mode** | 5.2KB | 0KB | 0% | Official only |
| **Getting Started** | 5.8KB | 6.9KB | 20% | Different focuses |
| **Architecture** | 0KB | 14KB | 0% | Custom only |
| **Implementation** | 0KB | 18KB | 0% | Custom only |
| **Tool System** | 0KB | 20KB + 11KB | 0% | Custom only |
| **UI Layer** | 0KB | 18KB | 0% | Custom only |
| **State Management** | 0KB | 17KB + 11KB | 0% | Custom only |
| **Performance** | 0KB | 14KB | 0% | Custom only |
| **Hidden Features** | 0KB | 17KB | 0% | Custom only |
| **MCP Development** | 0KB | 19KB | 0% | Custom only |
| **Code Reference** | 0KB | 14KB | 0% | Custom only |

---

## Key Insights

### 1. Minimal Duplication
- **Overall Overlap**: 5-15% across documents that cover similar topics
- **Most Overlap**: Sandbox docs (15%) - but even here, official shows usage, custom shows implementation
- **Zero Overlap**: Prompts, Architecture, Implementation, Tool System, UI, State, Performance

### 2. Complementary Content

**Official Docs Excel At**:
- User onboarding
- Quick start guides
- Configuration syntax
- Usage examples
- CLI command reference
- Troubleshooting

**Custom Docs Excel At**:
- Internal architecture
- Rust code implementation
- System internals
- Security mechanisms
- Performance optimization
- Developer patterns

### 3. Target Audience Split

```
┌────────────────────────────────────────────────────────┐
│                    Documentation                        │
├────────────────────────────────────────────────────────┤
│                                                         │
│  Official Docs (13 files, 96KB)                        │
│  ├─ End Users         ████████████ 80%                 │
│  ├─ Contributors      ████ 15%                         │
│  └─ Developers        █ 5%                             │
│                                                         │
│  Custom Docs (23 files, 354KB)                         │
│  ├─ Developers        █████████████████ 90%            │
│  ├─ Security Auditors ████ 5%                          │
│  └─ End Users         █ 5%                             │
│                                                         │
└────────────────────────────────────────────────────────┘
```

### 4. Documentation Health

**Strengths**:
- ✅ Clear separation of concerns
- ✅ Minimal content duplication
- ✅ Complementary coverage
- ✅ Different depth levels

**Opportunities**:
- ⚠️ Add cross-references (now completed!)
- ⚠️ Custom docs should cover `codex exec` internals
- ⚠️ Verify all code examples against latest source
- ⚠️ Add more flow diagrams to custom docs

---

## Recommendations

### For Official Docs
✅ **No changes needed** - excellent user-facing documentation

### For Custom Docs
1. ✅ **Done**: Add cross-references to official docs
2. **Add**: Implementation details for `codex exec` mode
3. **Verify**: Code examples against latest source (ongoing)
4. **Enhance**: Add more flow diagrams for complex processes

### For Both
- Link to each other more prominently
- Update README.md with clear navigation
- Maintain this separation of concerns

---

## Conclusion

**The documentation ecosystems are highly complementary with minimal duplication.**

- **Quantified Overlap**: 5-15% in docs covering similar topics, 0% in most docs
- **Content Separation**: Official = usage, Custom = implementation
- **Target Separation**: Official = users, Custom = developers
- **Overall Health**: Excellent (95% unique content)

**Recommendation**: **Preserve both** - they serve different purposes and audiences.

---

**Analysis Date**: October 25, 2025
**Methodology**: Line-by-line content comparison, not automated diff
**Conclusion**: Minimal duplication, maximum value preservation recommended
