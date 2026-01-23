# Plugin Structure - Gleam Claude Code Plugin

Official Claude Code plugin structure following [Claude Code Plugin Guidelines](https://code.claude.com/docs/en/plugins).

## Plugin Structure

```
gleam-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest (required)
│   └── marketplace.json         # Marketplace catalog
├── .mcp.json                    # MCP server configuration
├── .lsp.json                    # LSP server configuration
├── skills/                      # Agent Skills (auto-discovered)
│   ├── gleam-web-development/
│   │   └── SKILL.md
│   ├── gleam-otp-development/
│   │   └── SKILL.md
│   ├── gleam-testing/
│   │   └── SKILL.md
│   ├── gleam-deployment/
│   │   └── SKILL.md
│   ├── gleam-package-development/
│   │   └── SKILL.md
│   ├── gleam-javascript-interop/
│   │   └── SKILL.md
│   └── gleam-erlang-interop/
│       └── SKILL.md
├── commands/                    # Slash commands (auto-discovered)
│   ├── new-project.md
│   ├── test.md
│   ├── format.md
│   ├── build.md
│   ├── add-dependency.md
│   ├── web-endpoint.md
│   ├── lustre-component.md
│   ├── lustre-page.md
│   └── otp-supervisor.md
└── agents/                      # Custom agents (auto-discovered)
    ├── gleam-architect.md
    ├── gleam-test-specialist.md
    ├── gleam-otp-specialist.md
    └── gleam-web-specialist.md
```

## Component Details

### Plugin Manifest (`.claude-plugin/plugin.json`)

**Required fields:**
- `name` - Plugin identifier (becomes namespace for commands)
- `version` - Semantic version
- `description` - Brief description

**Optional fields:**
- `author` - Author information
- `homepage` - Plugin homepage URL
- `repository` - Source code repository
- `license` - License identifier
- `keywords` - Search keywords

Claude Code auto-discovers components from directory structure. No need to list commands, skills, or agents in the manifest.

### Marketplace Catalog (`.claude-plugin/marketplace.json`)

Enables distribution via plugin marketplaces. Defines:
- Marketplace name and owner
- Plugin source locations
- Version information

### MCP Servers (`.mcp.json`)

Configures Model Context Protocol servers at plugin root.

**Our configuration:**
```json
{
  "mcpServers": {
    "gleam_packages": {
      "transport": "http",
      "url": "https://gleam-package-mcp.renata-amutio.workers.dev/",
      "description": "Gleam package search and documentation"
    }
  }
}
```

Provides:
- Package search on hex.pm
- Package documentation
- Module and function lookup

### LSP Server (`.lsp.json`)

Configures Language Server Protocol for code intelligence.

**Our configuration:**
```json
{
  "gleam": {
    "command": "gleam",
    "args": ["lsp"],
    "extensionToLanguage": {
      ".gleam": "gleam"
    }
  }
}
```

Provides:
- Go to definition
- Find references
- Hover documentation
- Code diagnostics

Requires `gleam` to be installed in PATH.

### Skills (`skills/*/SKILL.md`)

Agent Skills are automatically invoked by Claude based on context.

**Structure:**
```
skills/
└── skill-name/
    └── SKILL.md
```

**SKILL.md format:**
```markdown
---
name: skill-name
description: When Claude should use this skill
---

# Skill Content

Instructions and examples...
```

**Our skills:**
- `gleam-web-development` - Wisp, Mist, Lustre, REST APIs
- `gleam-otp-development` - Actors, supervisors, fault tolerance
- `gleam-testing` - Tests with gleeunit and `let assert`
- `gleam-deployment` - Docker, Fly.io deployment
- `gleam-package-development` - Publishing to Hex.pm
- `gleam-javascript-interop` - FFI with JavaScript
- `gleam-erlang-interop` - FFI with Erlang/Elixir

Skills are model-invoked: Claude automatically uses them when the description matches the task.

### Commands (`commands/*.md`)

Slash commands that users invoke explicitly.

**Naming:** Files become commands with plugin namespace:
- `commands/test.md` → `/gleam:test`
- `commands/web-endpoint.md` → `/gleam:web-endpoint`

**Our commands:**
- `new-project` - Create new Gleam project
- `test` - Run tests
- `format` - Format code
- `build` - Build project
- `add` - Add dependencies
- `web-endpoint` - Create API endpoint (Wisp)
- `lustre-component` - Create UI component (Lustre)
- `lustre-page` - Create page (Lustre)
- `otp-supervisor` - Create OTP supervisor

### Agents (`agents/*.md`)

Specialized agents for complex tasks.

**Our agents:**
- `gleam-architect` - Architecture and design
- `gleam-test-specialist` - Comprehensive testing
- `gleam-otp-specialist` - Concurrent systems
- `gleam-web-specialist` - Web APIs and services

## Installation

### For Users

**Via marketplace:**
```shell
/plugin marketplace add github:renatillas/gleam-claude-plugin
/plugin install gleam@gleam
```

**Direct from GitHub:**
```shell
/plugin install gleam@github:renatillas/gleam-claude-plugin
```

**Manual:**
```bash
git clone https://github.com/renatillas/gleam-claude-plugin.git
cd gleam-claude-plugin
ln -s $(pwd) ~/.claude/plugins/gleam
```

### For Teams

Add to `.claude/settings.json`:
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

## Testing

### Validate Plugin

```bash
# From plugin directory
claude plugin validate .

# Or in Claude Code
/plugin validate .
```

### Local Testing

```bash
# Load plugin for testing
claude --plugin-dir ./gleam-claude-plugin

# Use commands
/gleam:test
/gleam:format

# Check help
/help
```

## Version Management

Update version in both files:
1. `.claude-plugin/plugin.json`
2. `.claude-plugin/marketplace.json`

Use semantic versioning:
- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes

Tag releases:
```bash
git tag v1.1.0
git push origin v1.1.0
```

## How Components Work

### Auto-Discovery

Claude Code automatically discovers:
- Commands in `commands/` directory
- Skills in `skills/*/SKILL.md` files
- Agents in `agents/` directory
- MCP servers in `.mcp.json`
- LSP servers in `.lsp.json`

No need to list these in `plugin.json`.

### Namespacing

All commands are namespaced with plugin name:
- Plugin name: `gleam`
- Command file: `test.md`
- Full command: `/gleam:test`

Prevents conflicts with other plugins.

### Skill Invocation

Skills are automatically invoked based on:
1. `description` field in frontmatter
2. Task context
3. User intent

Example: User asks "build a REST API" → Claude invokes `gleam-web-development` skill.

## References

- [Official Plugin Guide](https://code.claude.com/docs/en/plugins)
- [Plugin Reference](https://code.claude.com/docs/en/plugins-reference)
- [Skills Documentation](https://code.claude.com/docs/en/skills)
- [Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
