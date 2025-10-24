# open-docs

> Creating comprehensive, high-quality documentation for complex technical projects that deserve better docs.

## What is this?

**open-docs** is a documentation initiative focused on producing thorough, well-structured technical documentation for open-source projects. Many powerful tools and frameworks have incomplete, scattered, or hard-to-navigate documentation. This project aims to fix that.

By combining automated research, code analysis, and systematic organization, we create comprehensive documentation that covers:

- **Undocumented features** - Capabilities that exist in the code but aren't officially documented
- **Architecture deep-dives** - Internal structure, design patterns, and implementation details
- **Complete API references** - Every tool, function, and interface with practical examples
- **Gotchas and edge cases** - Hidden issues, quirks, and important implementation notes
- **Real-world usage** - Practical guides based on actual code patterns and best practices

### Principles

- **Open Source Only** - All documentation is derived exclusively from publicly available open-source repositories and community resources
- **No Proprietary Content** - We never include private, confidential, or proprietary materials
- **Automated + Human-Curated** - Leveraging AI-powered research with human oversight for quality and accuracy
- **Community-Driven** - Built from and for the community

## 📚 Available Documentation

### 🚀 [Claude Agent SDK - Complete Documentation](docs/)

**Status**: ✅ **85% Complete** | 15 comprehensive documents | ~23,000+ lines | **100% source-verified**

#### Quick Start
- **[Quick Start Guide](docs/quick-start/quick-start-guide.md)** - Get started in 5 minutes with installation, first agent, and common workflows

#### For SDK Developers
- **[Complete SDK Developer Guide](docs/sdk-developer-guide/complete-sdk-guide.md)** (2,600+ lines) - Installation to deployment, building agents, custom tools, MCP integration, hooks, permissions, production deployment
- **[LLM Integration Patterns](docs/patterns/llm-integration-patterns.md)** (1,300+ lines) - Model selection, streaming, token management, extended thinking, cost optimization

#### Deep Dive / Internals
- **[Architecture & Internals](docs/internals/architecture-and-internals.md)** (2,800+ lines) - Complete architecture, core components, message flow, tool execution pipeline, permission resolution, hook engine, session management

#### Source-Verified Extraction Docs (from actual SDK code)
- **[Tools System Complete](docs/extraction/tools-system-complete.md)** (1,547 lines) - All 17 tools with TypeScript schemas, execution patterns, performance
- **[Hooks System Complete](docs/extraction/hooks-system-complete.md)** (963 lines) - All 9 hook events, patterns, examples
- **[Agents & Subagents Complete](docs/extraction/agents-subagents-complete.md)** (1,071 lines) - All 5 built-in agents, token optimization (70-84% savings)
- **[Permissions System Complete](docs/extraction/permissions-system-complete.md)** (922 lines) - 4 modes, 6-level resolution cascade
- **[MCP Integration Complete](docs/extraction/mcp-integration-complete.md)** (1,142 lines) - 4 transport types (SDK 30-60x faster!)
- **[Configuration Complete](docs/extraction/configuration-complete.md)** (843 lines) - 7-level configuration resolution
- **[Skills System Complete](docs/extraction/skills-system-complete.md)** (868 lines) - SKILL.md format, frontmatter, discovery
- **[Internal Constants](docs/extraction/cli-internal-constants.md)** (561 lines) - All limits, timeouts, performance benchmarks
- **[Type System Complete](docs/extraction/sdk-types-complete.md)** (909 lines) - All TypeScript types from source

#### Troubleshooting & API Reference
- **[Complete Troubleshooting Guide](docs/troubleshooting/complete-troubleshooting-guide.md)** (3,200+ lines) - Common errors, authentication issues, tool problems, permission errors, MCP issues, network errors, debugging techniques, error reference
- **[Complete API Reference](docs/api-reference/complete-api-reference.md)** (2,900+ lines) - External API endpoints, authentication flows (OAuth 2.0 with PKCE), network architecture, SDK public API, model reference, message protocol

#### 🔥 Hidden Features & Beta Features (NEW! - TODAY)
- **[Undocumented Features & Beta](docs/hidden-features/undocumented-features-and-beta.md)** (3,000+ lines) - 20+ undocumented features extracted from source: beta API features, hidden env variables, feature flags, hidden CLI commands, advanced prompt caching, debug features, power user tips

#### Features Documented
✅ All 17 tools | ✅ All 9 hooks | ✅ All 5 agents | ✅ All 4 MCP transports | ✅ Complete architecture | ✅ 200+ code examples | ✅ 80+ gotchas | ✅ 50+ patterns | ✅ 40+ performance benchmarks | ✅ Complete OAuth flow | ✅ Network diagrams | ✅ Error solutions

---

### [Codex CLI](docs/codex_cli/README.md)
Full documentation of the codex_cli project, including prompt processing, LLM integration, tool system, security, and MCP integration.

## 🚀 Getting Started

Each documentation set includes:
- A comprehensive README with navigation guide
- Multiple focused documents covering specific aspects
- Code examples and practical usage patterns
- Architecture diagrams and implementation details

Start by browsing the README for any documentation set above, or use the search functionality to find specific topics.

## 🔍 Recommended Tools

For intelligent exploration of this repository, we recommend using **Octocode MCP** - a smart AI code and GitHub research MCP server that creates better context for you and your AI agents.

### Why Octocode MCP?

**Octocode MCP** enhances your development environment by providing AI agents with powerful code research capabilities:

- **Smart Code Search** - Advanced pattern matching and semantic search across codebases
- **Structure Analysis** - Understand directory layouts, dependencies, and file organization
- **Content Exploration** - Efficient file reading with pagination and context extraction
- **Intelligent Navigation** - Quick discovery of relevant documentation and implementations
- **Cross-Reference Analysis** - Connect related concepts across multiple documentation sets (and actual repositories)

### Perfect for This Repository

When working with open-docs, Octocode MCP helps you:
- Quickly find specific topics across Claude Code, Claude Agent SDK, and Codex CLI docs
- Understand relationships between different projects and their implementations
- Search for patterns, APIs, and features across all documentation
- Navigate complex documentation structures efficiently
- Build comprehensive context for AI-assisted development

### Get Started

- **Repository:** [github.com/bgauryy/octocode-mcp](https://github.com/bgauryy/octocode-mcp)  
- **Demo:** [youtube.com/watch?v=S2pcEjHo6CM](https://www.youtube.com/watch?v=S2pcEjHo6CM)  
- **Setup:** Follow the README for installation with Claude Desktop, Cursor, or other MCP-compatible tools

## 💬 Request Documentation

Have a technical project that needs better documentation? Or do you want to better understand something about an existing project or documentation? We're always looking for new projects or topics to document.

**Open a request:** [github.com/bgauryy/open-docs/issues](https://github.com/bgauryy/open-docs/issues)

Please include:
- Project name and repository URL
- Why you think it needs better documentation
- Specific areas or features you'd like documented

## ⭐ Support

If you find this project useful, please consider:
- Giving it a ⭐ star on GitHub
- Sharing it with others who might benefit
- Contributing improvements or suggestions
- Requesting documentation for projects you care about

## ⚖️ Disclaimer

- This project is provided for **research and educational purposes only**
- All content is derived from publicly available open-source resources
- Do not use any information to cause harm, exploit systems, or violate laws or terms of service
- Independently verify all statements and use your own judgment
- You are responsible for how you use this material
- This project provides **no warranty** and is used **at your own risk**

---

*Built with AI-powered research and human curation to make technical documentation accessible to everyone.*
