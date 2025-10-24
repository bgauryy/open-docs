# Gemini CLI - Deep Reverse Engineering Audit

**Generated:** October 24, 2025  
**Codebase:** `/Users/guybary/Documents/octocode-local-files/context/gemini-cli`  
**Total Files Analyzed:** 459+ TypeScript files across 5 packages  
**Purpose:** Comprehensive reference documentation for developers, researchers, super users, and security auditors

---

## Overview

This directory contains **11 comprehensive reference documents** resulting from a deep reverse engineering audit of the `gemini-cli` codebase. All undocumented features, tools, behaviors, gotchas, and internal mechanisms have been extracted and documented.

---

## Document Index

### 1. Architecture Overview
**File:** [`01_ARCHITECTURE_OVERVIEW.md`](./01_ARCHITECTURE_OVERVIEW.md)

**Contents:**
- System architecture and package structure
- Core components (5 packages: core, cli, a2a-server, vscode-ide-companion, test-utils)
- Data flow and execution model
- Technology stack
- Entry points and module dependencies

**Use for:** Understanding overall system design and package relationships

---

### 2. Tools Reference
**File:** [`02_TOOLS_REFERENCE.md`](./02_TOOLS_REFERENCE.md)

**Contents:**
- Complete catalog of 15+ built-in tools
- Tool parameters, schemas, and behaviors
- Tool confirmation system
- MCP tool integration
- Undocumented tool features
- Tool gotchas and limitations

**Use for:** Tool development, debugging tool issues, understanding tool capabilities

---

### 3. Prompts Reference
**File:** [`03_PROMPTS_REFERENCE.md`](./03_PROMPTS_REFERENCE.md)

**Contents:**
- Core system prompt structure
- Prompt components and builders
- Dynamic prompt features
- Compression prompts
- MCP prompts integration
- Agent prompts
- Prompt customization via `GEMINI_SYSTEM_MD`

**Use for:** Understanding AI behavior, customizing prompts, debugging responses

---

### 4. Configuration Reference
**File:** [`04_CONFIGURATION_REFERENCE.md`](./04_CONFIGURATION_REFERENCE.md)

**Contents:**
- Complete settings schema (100+ settings)
- Configuration file locations and precedence
- Environment variables (30+ documented)
- Configuration layers and merging strategies
- Keyboard shortcuts (40+ bindings)
- Hidden/undocumented settings

**Use for:** Configuring Gemini CLI, troubleshooting configuration issues

---

### 5. Security & Permissions
**File:** [`05_SECURITY_PERMISSIONS.md`](./05_SECURITY_PERMISSIONS.md)

**Contents:**
- Security architecture (4 layers)
- Trust system and folder trust
- Sandbox execution (Docker/Podman/Seatbelt)
- Tool confirmation system
- File system security
- Authentication & authorization
- Security environment variables
- Approval modes

**Use for:** Security audits, deployment planning, permission configuration

---

### 6. Hidden Features & Experimental
**File:** [`06_HIDDEN_FEATURES.md`](./06_HIDDEN_FEATURES.md)

**Contents:**
- Experimental features (extension management, model router, codebase investigator)
- Hidden tools (`write_todos`, `smart_edit`)
- Undocumented settings (30+)
- Deprecated features
- Development TODOs (70+ files)
- Feature flags (15+ environment variables)
- Alpha/Beta features

**Use for:** Discovering undocumented features, understanding experimental APIs

---

### 7. CLI Commands & UI
**File:** [`07_CLI_COMMANDS_UI.md`](./07_CLI_COMMANDS_UI.md)

**Contents:**
- 29+ slash commands with subcommands
- Core commands (`/help`, `/chat`, `/settings`, etc.)
- MCP commands (`/mcp list`, `/mcp auth`, etc.)
- Keyboard shortcuts (complete reference)
- UI components (10+ major components)
- Dialog types
- Hidden commands (`/corgi` Easter egg!)

**Use for:** Command reference, UI customization, keyboard navigation

---

### 8. A2A Agent System
**File:** [`08_A2A_AGENT_SYSTEM.md`](./08_A2A_AGENT_SYSTEM.md)

**Contents:**
- A2A (Agent-to-Agent) server architecture
- Task system and lifecycle
- CoderAgentExecutor
- HTTP API endpoints
- Persistence layer (GCS)
- Event system
- Deployment guides

**Status:** ⚠️ EXPERIMENTAL - APIs may change

**Use for:** Deploying Gemini CLI as a service, agent orchestration

---

### 9. IDE Integration
**File:** [`09_IDE_INTEGRATION.md`](./09_IDE_INTEGRATION.md)

**Contents:**
- VSCode extension reference
- IDE protocol (JSON-RPC over HTTP)
- Diff management system
- Open files tracking
- IdeClient API
- Code assist features
- Workspace trust integration
- Environment synchronization

**Status:** ⚠️ PREVIEW - VSCode extension in preview

**Use for:** IDE extension development, debugging IDE integration

---

### 10. Testing Patterns
**File:** [`10_TESTING_PATTERNS.md`](./10_TESTING_PATTERNS.md)

**Contents:**
- Test framework (Vitest)
- Unit testing patterns
- Integration testing (100+ tests)
- Test utilities and mocks
- Edge cases and boundary conditions
- Coverage reporting
- Testing best practices

**Use for:** Writing tests, understanding edge cases, debugging test failures

---

### 11. Gotchas & Limitations
**File:** [`11_GOTCHAS_LIMITATIONS.md`](./11_GOTCHAS_LIMITATIONS.md)

**Contents:**
- 30+ documented gotchas
- Critical security issues (3)
- Configuration gotchas (10+)
- Tool limitations (5+)
- MCP gotchas (5+)
- IDE integration limitations (5+)
- Platform-specific issues
- Workarounds and solutions matrix

**Use for:** Troubleshooting, avoiding common pitfalls, security hardening

---

## Quick Navigation

### By Role

#### **Developers**
→ Start with: Architecture Overview → Tools Reference → Configuration Reference

#### **Researchers**
→ Start with: Hidden Features → Prompts Reference → Testing Patterns

#### **Super Users**
→ Start with: CLI Commands & UI → Configuration Reference → Gotchas & Limitations

#### **Security Auditors**
→ Start with: Security & Permissions → Gotchas & Limitations → Architecture Overview

#### **Extension Developers**
→ Start with: Tools Reference → IDE Integration → A2A Agent System

---

### By Topic

#### **Configuration & Setup**
- Configuration Reference (#4)
- Security & Permissions (#5)
- CLI Commands & UI (#7)

#### **Tool Development**
- Tools Reference (#2)
- Architecture Overview (#1)
- Testing Patterns (#10)

#### **Advanced Features**
- Hidden Features (#6)
- A2A Agent System (#8)
- IDE Integration (#9)

#### **Debugging & Troubleshooting**
- Gotchas & Limitations (#11)
- Testing Patterns (#10)
- Security & Permissions (#5)

---

## Key Findings Summary

### Architecture
- **5 packages:** core, cli, a2a-server, vscode-ide-companion, test-utils
- **459+ TypeScript files**
- **Monorepo structure** with shared core
- **Event-driven** execution model

### Tools
- **15+ built-in tools** (read, write, search, shell, etc.)
- **MCP integration** for external tools
- **Tool confirmation system** with 3 approval modes
- **Hidden tools:** `write_todos`, `smart_edit`

### Configuration
- **100+ settings** across 15+ categories
- **3 configuration layers:** defaults → user → project → env vars
- **30+ environment variables**
- **40+ keyboard shortcuts**

### Security
- **4-layer security model:** trust → confirmation → sandbox → filesystem
- **3 sandbox types:** Docker, Podman, Seatbelt (macOS)
- **Folder trust system** (experimental)
- **No workspace boundary enforcement** (use sandbox!)

### Experimental Features
- **Extension management** (enabled by default, but experimental)
- **Model router** (auto-selects best model)
- **Codebase investigator** agent
- **A2A server** (BETA)
- **VSCode extension** (PREVIEW)

### Critical Gotchas
1. **No workspace boundary** - Can access entire filesystem
2. **YOLO mode** - Disables all confirmations (dangerous!)
3. **Ignored files not protected** - `.gitignore` doesn't prevent access
4. **A2A server no auth** - HTTP API has no authentication

---

## Statistics

- **Documents Generated:** 11
- **Total Pages:** ~150+ pages (estimated)
- **Total Word Count:** ~50,000+ words
- **Files Analyzed:** 459+ TypeScript files
- **Packages Covered:** 5 (100%)
- **Settings Documented:** 100+
- **Environment Variables:** 30+
- **Keyboard Shortcuts:** 40+
- **Slash Commands:** 29+
- **Tools Documented:** 15+
- **Gotchas Identified:** 30+
- **Test Files Analyzed:** 100+

---

## Methodology

### Code Analysis Techniques
1. **Static Code Analysis** - Read all TypeScript source files
2. **Schema Extraction** - Analyzed Zod schemas for complete settings
3. **Type Analysis** - Extracted TypeScript interfaces and types
4. **Pattern Matching** - Searched for TODO/FIXME/HACK comments
5. **Test Analysis** - Examined test files for edge cases
6. **Integration Test Review** - Analyzed end-to-end test scenarios

### Tools Used
- `local_ripgrep` - Pattern searching across codebase
- `local_find_files` - File discovery and filtering
- `local_fetch_content` - Targeted file reading
- `local_view_structure` - Directory structure mapping
- Manual code review - Deep analysis of critical files

---

## Disclaimer

**Status:** This audit is based on the codebase as of **October 24, 2025**.

**Experimental Features:** Many features are marked as experimental (A2A server, VSCode extension, etc.) and APIs may change.

**Completeness:** While comprehensive, this audit may not cover every edge case or undocumented behavior. Always refer to official documentation for production deployments.

**Security:** Security findings are for informational purposes. Always conduct professional security audits for production systems.

---

## How to Use This Audit

1. **Start with README** (this file) for overview
2. **Navigate by role** using the Quick Navigation section
3. **Reference specific documents** as needed
4. **Use document search** (Cmd+F / Ctrl+F) to find specific topics
5. **Check Gotchas & Limitations** before production deployment
6. **Refer to Configuration Reference** for all settings
7. **Consult Security & Permissions** for secure deployment

---

## Contributing

If you discover additional undocumented features, gotchas, or limitations:

1. Document the finding with context
2. Note the file/location in codebase
3. Provide reproduction steps if applicable
4. Submit as feedback to the Gemini CLI team

---

## License

This audit documentation is provided as-is for reference purposes. The original Gemini CLI codebase is licensed under Apache 2.0 (see LICENSE in the gemini-cli repository).

---

**Audit Completed:** October 24, 2025  
**Generated by:** Deep Code Analysis AI Agent  
**Codebase Version:** gemini-cli (latest as of audit date)

For questions or issues with this audit, please refer to the Gemini CLI repository: https://github.com/google-gemini/gemini-cli

