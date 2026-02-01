# Pants Plugins for Claude Code

A Claude Code plugin with comprehensive tools for Pants build system development.

## Installation

```bash
# Add the marketplace
/plugin marketplace add jaymd96/pants-plugin-creator

# Install the plugin
/plugin install pants-plugins@pants-plugins
```

Update anytime with:
```bash
/plugin marketplace update pants-plugins
```

## Available Skills

### 1. create-pants-plugin
**Create new Pants plugins from scratch**

```
/pants-plugins:create-pants-plugin
```

Or ask naturally:
- "Create a Pants plugin that lints shell scripts"
- "I need a plugin for generating code from protobuf"

Features:
- Guided workflow for requirements gathering
- Generates complete plugin structure with all files
- Includes pyproject.toml, targets, rules, goals, tests
- Ready-to-use patterns for linters, code generators, custom targets

### 2. develop-pants-plugin
**Develop and modify existing Pants plugins**

```
/pants-plugins:develop-pants-plugin
```

Or ask naturally:
- "Add a new target type to my plugin"
- "Debug why my rule isn't being called"
- "Write tests for my Pants plugin"
- "Upgrade my plugin for Pants 2.19"

Features:
- Add new targets, fields, rules, goals to existing plugins
- Debug common issues (rules not firing, caching problems, type errors)
- Write and improve tests with RuleRunner
- Upgrade plugins for new Pants versions
- Performance optimization guidance

### 3. pants-oracle
**Comprehensive Pants knowledge base**

```
/pants-plugins:pants-oracle
```

Or ask naturally:
- "How does Pants caching work?"
- "What's the difference between Digest and Snapshot?"
- "How do I create a custom field type?"
- "Explain the rule graph"

Features:
- Expert answers to any Pants question
- Complete API reference for targets, fields, rules
- Engine architecture and caching internals
- Testing patterns with RuleRunner
- Best practices and common pitfalls

### 4. pants-setup
**Setup and manage Pants in repositories**

```
/pants-plugins:pants-setup
```

Or ask naturally:
- "Setup Pants in this repo"
- "Add the Python baseline plugin"
- "Search for available Pants plugins"
- "What Pants packages are available?"
- "Remove unused Pants backends"
- "Configure pants.toml"

Features:
- Add Python baseline plugin (Ruff, ty, uv, pytest)
- **Search for plugins** on GitHub (`gh repo list jaymd96 --topic pants-plugin`)
- **Search PyPI** for installable packages (`jaymd96-pants-*`)
- Discover available backend packages
- Configure pants.toml with best practices
- Add/remove third-party plugins
- **Naming conventions** for creating new plugins
- General Pants housekeeping

### 5. pants-discover
**Discover available Pants commands in your repo**

```
/pants-plugins:pants-discover
```

Or ask naturally:
- "What Pants commands can I run?"
- "Show me available Pants goals"
- "List all targets in this repo"
- "What does `pants lint` do?"

Features:
- List all available goals and commands
- Show what backends are configured
- Explain goal-specific options
- List and explore targets
- Dependency exploration commands

## What create-pants-plugin Generates

```
my-plugin/
├── pyproject.toml           # Hatch-based build config
├── src/my_plugin/
│   ├── register.py          # Plugin entry point
│   ├── targets.py           # Target definitions
│   ├── rules.py             # Business logic
│   ├── goals.py             # User commands
│   └── subsystem.py         # Configuration
├── tests/
│   ├── conftest.py
│   └── unit/
├── README.md
├── Makefile
└── .gitignore
```

## Reference Documentation

The plugin includes comprehensive reference docs:

**create-pants-plugin:**
- `quickstart.md` - 5-minute intro
- `api-reference.md` - Target & Rules API
- `templates.md` - File templates
- `patterns/` - Linter, codegen, custom target patterns

**develop-pants-plugin:**
- `debugging.md` - Troubleshooting guide
- `testing.md` - RuleRunner patterns
- `upgrading.md` - Version migration
- `performance.md` - Optimization techniques

**pants-oracle:**
- `architecture.md` - Engine internals and design
- `targets.md` - Complete Target API reference
- `rules.md` - Complete Rules API reference
- `testing.md` - Testing patterns and utilities

**pants-setup:**
- `pants-toml.md` - Complete pants.toml configuration reference
- `backends.md` - All available backend packages
- `plugins.md` - Third-party plugin installation guide
- `search-plugins.md` - Search GitHub/PyPI for plugins
- `naming-conventions.md` - Plugin naming standards

**pants-discover:**
- `goals.md` - Complete goals reference
- `target-types.md` - All target types reference

## Development Workflow

```bash
cd my-plugin
hatch env create       # Set up environment
hatch run test         # Run tests
hatch run fmt          # Format code
hatch run lint         # Check quality
hatch run all          # Everything
```

## Publishing to PyPI

```bash
hatch build
hatch publish
```

## Key Pants Concepts

- **Targets**: Metadata about code in BUILD files
- **Fields**: Individual pieces of target metadata
- **Rules**: Pure async functions (no side effects!)
- **Goals**: User commands (`pants my-goal`)
- **Subsystems**: Configuration in pants.toml

## Resources

- [Official Pants Plugin Docs](https://www.pantsbuild.org/stable/docs/writing-plugins/overview)
- [Pants GitHub](https://github.com/pantsbuild/pants)
- [Pants Slack](https://pantsbuild.slack.com)

## License

Apache License 2.0
