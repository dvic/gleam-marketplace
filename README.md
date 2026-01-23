# Gleam Everything - Comprehensive Claude Code Plugin for Gleam Development

A complete Claude Code plugin providing best practices, patterns, workflows, and expert guidance for Gleam development.

## Overview

This plugin equips Claude Code with comprehensive knowledge of Gleam programming, including:

- **Official conventions and patterns** from the Gleam team
- **Best practices** for web development, OTP, testing, and deployment
- **Specialized agents** for architecture, testing, OTP, and web development
- **Quick commands** for common workflows
- **Context-aware guidance** for different development scenarios
- **Up-to-date information** with links to authoritative sources

## Features

### 🎯 Agent Skills (Auto-Invoked)

Claude automatically uses these skills based on context:

- **gleam-web-development** - Wisp, Mist, Lustre, REST APIs, Parse→Process→Present pattern
- **gleam-otp-development** - Actors, supervisors, fault tolerance, concurrent applications
- **gleam-testing** - Unit tests, integration tests, `let assert` (NOT deprecated `should`)
- **gleam-deployment** - Docker, Fly.io, production deployment
- **gleam-package-development** - Creating and publishing packages to Hex.pm
- **gleam-javascript-interop** - FFI, NPM packages, browser APIs
- **gleam-erlang-interop** - BEAM integration, Erlang/Elixir libraries

### ⚡ Slash Commands

Quick workflows via slash commands:

**General:**
- `/gleam:new-project` - Create new project with best practices
- `/gleam:test` - Run tests with proper configuration
- `/gleam:format` - Format code
- `/gleam:build` - Build and handle errors
- `/gleam:add` - Add dependencies

**Backend (Wisp):**
- `/gleam:web-endpoint` - Create web API endpoint with Parse→Process→Present pattern
- `/gleam:otp-supervisor` - Create OTP supervisor

**Frontend (Lustre):**
- `/gleam:lustre-component` - Create reusable UI component
- `/gleam:lustre-page` - Create new page with Model-View-Update pattern

### 🤖 Specialized Agents

Delegate complex tasks to specialized agents:

- **gleam-architect** - Architecture and design decisions
- **gleam-test-specialist** - Comprehensive test coverage
- **gleam-otp-specialist** - Concurrent and distributed systems
- **gleam-web-specialist** - Web APIs and HTTP services

## Installation

### Recommended: Via Marketplace (Easiest)

Install directly from GitHub using Claude Code's plugin system:

```shell
# Add the marketplace
/plugin marketplace add github:renatillas/gleam-claude-plugin

# Install the plugin
/plugin install gleam@gleam
```

Or install directly without adding the marketplace:

```shell
/plugin install gleam@github:renatillas/gleam-claude-plugin
```

### Alternative: Manual Installation

#### Method 1: Clone and Link

```bash
# Clone this repository
git clone https://github.com/renatillas/gleam-claude-plugin.git
cd gleam-claude-plugin

# Link to Claude Code plugins directory
ln -s $(pwd) ~/.claude/plugins/gleam
```

#### Method 2: Direct Download

```bash
# Create plugins directory
mkdir -p ~/.claude/plugins

# Download and extract
cd ~/.claude/plugins
# Download from https://github.com/renatillas/gleam-claude-plugin
```

### Verify Installation

Check that Claude Code recognizes the plugin:

```bash
# Via CLI
claude plugin list

# Or in Claude Code
/plugin list
```

You should see `gleam` in the list.

### MCP Integration

This plugin includes the **gleam_packages MCP server** which provides:
- Search for Gleam packages on hex.pm
- Get package information and documentation
- Browse modules, functions, and types
- Search within packages

The MCP server is automatically configured via `.mcp.json` and connects to:
`https://gleam-package-mcp.renata-amutio.workers.dev/`

No additional setup required - Claude Code will automatically have access to Gleam package search and documentation!

### LSP Integration

This plugin includes **Gleam LSP (Language Server Protocol)** configuration which provides:
- Go to definition
- Find references
- Hover documentation
- Code diagnostics and errors
- Auto-completion

The LSP server is automatically configured via `.lsp.json`. Requires `gleam` to be installed and available in your PATH.

Install Gleam:
```bash
# macOS
brew install gleam

# Linux
curl -sSL https://gleam.run/install.sh | sh

# Or download from https://gleam.run/getting-started/installing/
```

### Team Distribution

To require this plugin for your team, add it to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "gleam": {
      "source": {
        "source": "github",
        "repo": "renatillas/gleam-claude-plugin"
      }
    }
  },
  "enabledPlugins": {
    "gleam@gleam": true
  }
}
```

Team members will be prompted to install the plugin when they trust the project folder.

### Validation and Testing

Test the plugin locally before distribution:

```bash
# Validate plugin structure and configuration
claude plugin validate .

# Or from within Claude Code
/plugin validate .

# Test locally with --plugin-dir flag
claude --plugin-dir ./gleam-claude-plugin
```

## Usage

### For Project-Specific Setup

The plugin works automatically once installed. For project-specific customization, you can create a `CLAUDE.md` or `.claude/CLAUDE.md` in your Gleam project root with project-specific information.

### Using Slash Commands

After installation, all commands are available under the `gleam` namespace:

```shell
# General commands
/gleam:new-project my_app
/gleam:test
/gleam:format
/gleam:build
/gleam:add wisp

# Backend (Wisp) commands
/gleam:web-endpoint /api/users GET
/gleam:otp-supervisor worker_pool

# Frontend (Lustre) commands
/gleam:lustre-component button primary
/gleam:lustre-page products
```

View all available commands:

```shell
/help
```

### Agent Skills (Automatic)

Claude automatically invokes these skills based on context:

- **gleam-web-development** - Building web APIs, REST services, Wisp/Mist applications
- **gleam-otp-development** - Creating actors, supervision trees, concurrent applications
- **gleam-testing** - Writing tests with gleeunit, using `let assert`
- **gleam-deployment** - Deploying to Fly.io, Docker, production environments
- **gleam-package-development** - Creating and publishing packages to Hex.pm
- **gleam-javascript-interop** - Using FFI with JavaScript, browser APIs
- **gleam-erlang-interop** - Using FFI with Erlang/Elixir, BEAM libraries

Simply describe what you want to build, and Claude will use the appropriate skills:

```
"Help me build a REST API for user management"
→ Automatically uses gleam-web-development skill

"Create an actor-based worker pool"
→ Automatically uses gleam-otp-development skill
```

### Delegating to Specialized Agents

Request specialized agents for complex tasks:

```
"Let's design the architecture for this feature"
→ Uses Gleam Architect agent

"Write comprehensive tests for this module"
→ Uses Test Specialist agent

"Help me build an OTP supervision tree"
→ Uses OTP Specialist agent

"Create a web API endpoint for users"
→ Uses Web Specialist agent
```

## Key Principles

This plugin emphasizes:

### 1. **Link to Authoritative Sources**
Rather than duplicating content, we link to official documentation:
- [Gleam Official Docs](https://gleam.run/)
- [Gleam Conventions](https://github.com/gleam-lang/website/blob/patterns/documentation/conventions-patterns-anti-patterns.djot)
- [Package Documentation](https://hexdocs.pm/)

### 2. **Mandatory Best Practices**
Enforce critical rules:
- Use `Result` for errors, never `Option`
- Use `let assert`, NOT deprecated `should` module
- Libraries NEVER panic
- Always use qualified imports
- All public functions need type annotations

### 3. **Up-to-Date Information**
Based on:
- Gleam v1.10 (Happy Holidays 2025 release)
- Latest package versions
- Current ecosystem tools
- Recent blog posts and articles

### 4. **Practical Workflows**
Focus on real development tasks:
- Creating projects
- Writing tests
- Building web APIs
- Deploying applications
- Publishing packages

## Primary Sources

This plugin references these authoritative sources:

### Official Documentation
- [Gleam Website](https://gleam.run/)
- [Writing Gleam](https://gleam.run/writing-gleam/)
- [Gleam FAQ](https://gleam.run/frequently-asked-questions/)
- [Conventions & Patterns](https://github.com/gleam-lang/website/blob/patterns/documentation/conventions-patterns-anti-patterns.djot)
- [Language Tour](https://tour.gleam.run/)
- [Deployment Guide](https://gleam.run/deployment/fly/)

### Package Documentation
- [Gleam Standard Library](https://hexdocs.pm/gleam_stdlib/)
- [Gleam OTP](https://hexdocs.pm/gleam_otp/)
- [Wisp](https://hexdocs.pm/wisp/)
- [Mist](https://hexdocs.pm/mist/)
- [Lustre](https://hexdocs.pm/lustre/)
- [Gleeunit](https://hexdocs.pm/gleeunit/)

### Community Resources
- [Awesome Gleam](https://github.com/gleam-lang/awesome-gleam)
- [Gleam Test Timeouts](https://gearsco.de/blog/gleam-test-timeouts/)
- [Bit Array Syntax](https://gearsco.de/blog/bit-array-syntax/)
- [Gleam OTP: Using Supervisors](https://vpgleam.substack.com/p/gleam-otp-using-supervisors)

### Recent Updates
- [Gleam v1.10 Release Notes](https://gleam.run/news/the-happy-holidays-2025-release/)
- [Gleam Roadmap](https://gleam.run/roadmap/)
- [Gleam News](https://gleam.run/news/)

## Contributing

To contribute to this plugin:

1. **Submit issues** for corrections or improvements
2. **Update links** if sources change
3. **Add new resources** with proper attribution
4. **Keep examples** practical and current
5. **Follow the link-focused approach** - reference authoritative sources

## Philosophy

### Link-Focused, Not Content-Duplicating

This plugin links to authoritative sources rather than duplicating their content. Benefits:

- **Always current** - Sources update independently
- **Authoritative** - Official documentation is the source of truth
- **Maintainable** - Less content to keep synchronized
- **Respectful** - Proper attribution to original authors

### Comprehensive, Not Overwhelming

Provide:
- **Critical rules** that must be followed
- **Common patterns** for frequent tasks
- **Links to details** for deeper exploration
- **Practical examples** for real scenarios

### Prescriptive for Best Practices

Where Gleam has established conventions, this plugin is prescriptive:
- **MANDATORY** - Must always follow
- **FORBIDDEN** - Must never do
- **RECOMMENDED** - Should generally follow
- **DISCOURAGED** - Should generally avoid

## Project Structure

```
gleam-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest
│   └── marketplace.json         # Marketplace catalog
├── .mcp.json                    # MCP servers (gleam_packages)
├── .lsp.json                    # LSP server (Gleam LSP)
├── skills/                      # Agent Skills (auto-invoked)
│   ├── gleam-web-development/SKILL.md
│   ├── gleam-otp-development/SKILL.md
│   ├── gleam-testing/SKILL.md
│   ├── gleam-deployment/SKILL.md
│   ├── gleam-package-development/SKILL.md
│   ├── gleam-javascript-interop/SKILL.md
│   └── gleam-erlang-interop/SKILL.md
├── commands/                    # Slash commands
│   ├── new-project.md           # /gleam:new-project
│   ├── test.md                  # /gleam:test
│   ├── format.md                # /gleam:format
│   ├── build.md                 # /gleam:build
│   ├── add-dependency.md        # /gleam:add
│   ├── web-endpoint.md          # /gleam:web-endpoint
│   ├── lustre-component.md      # /gleam:lustre-component
│   ├── lustre-page.md           # /gleam:lustre-page
│   └── otp-supervisor.md        # /gleam:otp-supervisor
├── agents/                      # Specialized agents
│   ├── gleam-architect.md
│   ├── gleam-test-specialist.md
│   ├── gleam-otp-specialist.md
│   └── gleam-web-specialist.md
├── README.md
└── PLUGIN_STRUCTURE.md
```

## Requirements

- Claude Code CLI
- Gleam v1.10+ installed
- For Erlang target: Erlang/OTP 27+
- For JavaScript target: Node.js 18+

## License

[Specify your license]

## Acknowledgments

This plugin is built on the excellent work of:
- The Gleam core team
- The Gleam community
- Package maintainers
- Documentation contributors

All linked resources retain their original licenses and attributions.

---

**Built for Claude Code** | **Inspired by [everything-claude-code](https://github.com/affaan-m/everything-claude-code)**

For questions or issues, please visit the repository.
