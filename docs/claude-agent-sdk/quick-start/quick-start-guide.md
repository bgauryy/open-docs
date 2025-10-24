# Claude Agent SDK & Claude Code - Quick Start Guide

**Welcome!** This guide will get you up and running with Claude Agent SDK and Claude Code in less than 10 minutes.

---

## Table of Contents

1. [Installation](#installation)
2. [Setup & Configuration](#setup--configuration)
3. [First Agent in 5 Minutes](#first-agent-in-5-minutes)
4. [Basic Usage Examples](#basic-usage-examples)
5. [Common Workflows](#common-workflows)
6. [Troubleshooting](#troubleshooting)
7. [Next Steps](#next-steps)

---

## Installation

### Prerequisites

- **Node.js**: Version 18.0.0 or higher
- **npm, yarn, pnpm, or bun**: Any package manager
- **Anthropic API Key**: Get one at [console.anthropic.com](https://console.anthropic.com)

### Quick Install

Choose your package manager:

```bash
# npm
npm install -g @anthropic-ai/claude-agent-sdk

# yarn
yarn global add @anthropic-ai/claude-agent-sdk

# pnpm
pnpm add -g @anthropic-ai/claude-agent-sdk

# bun
bun add -g @anthropic-ai/claude-agent-sdk
```

### Verify Installation

```bash
claude-code --version
# Should output: claude-code/0.1.22 or similar
```

---

## Setup & Configuration

### Step 1: Set Your API Key

**Option A: Environment Variable** (Recommended)

```bash
# Add to ~/.bashrc, ~/.zshrc, or ~/.bash_profile
export ANTHROPIC_API_KEY="sk-ant-your-key-here"

# Or set for current session only
export ANTHROPIC_API_KEY="sk-ant-your-key-here"
```

**Option B: Configuration File**

Create `~/.claude/settings.json`:

```json
{
  "apiKey": "sk-ant-your-key-here"
}
```

### Step 2: Test the Connection

```bash
claude-code
```

You should see:
```
Claude Code v0.1.22
Type your message or '/help' for commands

>
```

Type `/help` to see available commands, or try:
```
> What can you help me with?
```

---

## First Agent in 5 Minutes

### Example 1: File Analysis

```bash
> Read the package.json file and summarize the project dependencies
```

**What happens**:
1. Agent uses `FileRead` tool to read package.json
2. Analyzes dependencies and devDependencies
3. Provides a clear summary with categories

### Example 2: Code Search

```bash
> Find all TODO comments in the src directory
```

**What happens**:
1. Agent uses `Grep` tool with pattern "TODO"
2. Searches through src/ directory
3. Lists all files containing TODO comments with line numbers

### Example 3: File Editing

```bash
> In src/config.ts, change the port from 3000 to 8080
```

**What happens**:
1. Agent reads src/config.ts first (required!)
2. Uses `FileEdit` to replace the port value
3. Confirms the change was made

### Example 4: Command Execution

```bash
> Run npm test and show me the results
```

**What happens**:
1. Agent uses `Bash` tool to execute npm test
2. Waits for command to complete
3. Shows test results

---

## Basic Usage Examples

### Working with Files

**Read a File**:
```
> Show me the contents of src/app.ts
```

**Create a New File**:
```
> Create a new file called src/utils/helper.ts with a function that formats dates
```

**Edit an Existing File**:
```
> In src/app.ts, add error handling to the main function
```

**Search for Code**:
```
> Find all functions that use the 'fetch' API
```

**Find Files**:
```
> Find all TypeScript files in the src directory
```

### Running Commands

**Install Dependencies**:
```
> Install the lodash package
```

**Run Tests**:
```
> Run the test suite
```

**Git Operations**:
```
> Show me the git status
> Create a new branch called feature/new-api
> Commit the changes with message "Add new API endpoint"
```

**Build Project**:
```
> Build the project for production
```

### Using Agents

**Quick Codebase Exploration** (70-84% faster!):
```
> Use the Explore agent to find all authentication-related code
```

**Code Review**:
```
> Use the security-review agent to check my latest changes
```

**Complex Refactoring**:
```
> Refactor the authentication system to use JWT tokens instead of sessions
```

---

## Common Workflows

### Workflow 1: New Feature Development

```bash
# 1. Create branch
> Create a new git branch called feature/user-auth

# 2. Create files
> Create a new file src/auth/login.ts with a basic login function

# 3. Implement feature
> Add JWT token generation to the login function

# 4. Add tests
> Create tests for the login function in tests/auth/login.test.ts

# 5. Run tests
> Run the tests

# 6. Commit changes
> Commit all changes with message "Add user authentication"
```

### Workflow 2: Bug Investigation

```bash
# 1. Find the bug
> Find all files that handle user authentication

# 2. Read the code
> Show me the contents of src/auth/middleware.ts

# 3. Analyze
> Explain what this authentication middleware does and identify potential issues

# 4. Fix
> Fix the race condition in the token validation

# 5. Test
> Run the auth tests

# 6. Commit
> Commit the fix with message "Fix authentication race condition"
```

### Workflow 3: Code Refactoring

```bash
# 1. Explore current structure
> Use the Explore agent to find all database access code

# 2. Plan refactoring
> /plan Create a plan to refactor database access into a repository pattern

# 3. Review plan and approve
> Yes, proceed with the plan

# 4. Execute refactoring
> Implement the first step of the plan

# 5. Test after each step
> Run tests to verify nothing broke

# 6. Commit frequently
> Commit the changes
```

---

## Troubleshooting

### Common Issues

#### Issue: "API key not found"

**Solution**:
```bash
# Set the API key
export ANTHROPIC_API_KEY="sk-ant-your-key-here"

# Or add to ~/.zshrc or ~/.bashrc
echo 'export ANTHROPIC_API_KEY="sk-ant-your-key-here"' >> ~/.zshrc
source ~/.zshrc
```

#### Issue: "Permission denied" when editing files

**Solution**:
```bash
# Use acceptEdits mode for rapid development
claude-code --permission-mode acceptEdits

# Or approve each edit when prompted
```

#### Issue: "File not found" errors

**Solution**:
- Use absolute paths instead of relative paths
- Or be in the correct directory
```bash
cd /path/to/your/project
claude-code
```

#### Issue: Bash command output truncated

**Solution**:
```
> Run npm test and save output to test-results.txt
> Show me the contents of test-results.txt
```

#### Issue: Edit tool says "old_string not found"

**Solution**:
- Read the file first
- Make sure old_string matches exactly (including whitespace)
- Add more context to make it unique

```
> Read src/app.ts first

> Now edit src/app.ts, replacing:
  function calculate() {
    return 42;
  }
with:
  function calculate() {
    return 1337;
  }
```

###  Performance Tips

**Use Explore Agent for Discovery** (70-84% faster!):
```bash
# Instead of:
> Find all React components

# Use:
> Use the Explore agent to find all React components
```

**Background Commands for Long Operations**:
```bash
> Run npm install in the background
# Continue working...
> Check the output of the background install
```

**Batch File Operations**:
```bash
> Read all TypeScript files in src/ and check for unused imports
# Better than reading one at a time
```

---

## Next Steps

### Learn More

**Core Features**:
- [Agents & Subagents](../extraction/agents-subagents-complete.md) - 70-84% token savings!
- [Hooks System](../extraction/hooks-system-complete.md) - Custom workflows
- [Permissions](../extraction/permissions-system-complete.md) - Security control
- [Skills](../extraction/skills-system-complete.md) - Reusable commands
- [MCP Integration](../extraction/mcp-integration-complete.md) - External tools

**Advanced Topics**:
- [Tool System](../extraction/tools-system-complete.md) - All 17 tools
- [Configuration](../extraction/configuration-complete.md) - Advanced setup
- [Internal Constants](../extraction/cli-internal-constants.md) - Limits & gotchas

### Try These Examples

**1. Code Review Workflow**:
```bash
> Review the changes in my current branch compared to main
> Focus on security issues and potential bugs
```

**2. Documentation Generation**:
```bash
> Generate API documentation for all exported functions in src/api/
```

**3. Test Generation**:
```bash
> Generate unit tests for src/utils/validation.ts with edge cases
```

**4. Refactoring**:
```bash
> Refactor src/legacy/ to use modern async/await instead of callbacks
```

**5. Codebase Analysis**:
```bash
> Use the Explore agent to give me an overview of the project structure
> Identify the main entry points and dependencies
```

---

## Quick Reference Card

### Essential Commands

```bash
# Start Claude Code
claude-code

# With specific options
claude-code --permission-mode acceptEdits
claude-code --model opus
claude-code --cwd /path/to/project

# Resume a session
claude-code --resume <session-id>
```

### Built-in Slash Commands

```bash
/help           # Show all commands
/plan           # Enter planning mode
/exit-plan      # Exit planning mode
/compact        # Compress conversation
/clear          # Clear conversation
/resume         # Resume previous session
/explore        # Use Explore agent
```

### Permission Modes

```bash
default              # Safe (asks for risky operations)
acceptEdits          # Auto-approve file edits
bypassPermissions    # Auto-approve everything (⚠️ dangerous!)
plan                 # Planning only (no execution)
```

### Agent Types

```bash
Explore              # Fast codebase discovery (Haiku)
general-purpose      # Complex tasks (Sonnet, all tools)
security-review      # Security audit (git-restricted)
```

---

## Support & Resources

**Documentation**:
- [Complete Documentation Index](../extraction/README.md)
- [Official SDK Docs](https://docs.claude.com/en/docs/claude-code)

**Community**:
- [Discord Server](https://discord.gg/anthropic)
- [GitHub Issues](https://github.com/anthropics/claude-code/issues)

**Contact**:
- Email: support@anthropic.com
- Twitter: @AnthropicAI

---

**Quick Start Complete!** You're now ready to use Claude Agent SDK and Claude Code effectively.

**Recommended Next Reading**:
1. [Agents & Subagents Guide](../extraction/agents-subagents-complete.md) - Learn about token savings
2. [Tool System Reference](../extraction/tools-system-complete.md) - Master all 17 tools
3. [Permissions Guide](../extraction/permissions-system-complete.md) - Configure security

**Happy Coding! 🚀**

