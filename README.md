# Pants Plugin Marketplace

A Claude Code marketplace with plugins for Pants build system development.

## Available Plugins

### 1. pants-plugin-creator
**Create new Pants plugins from scratch**

- Guided workflow for requirements gathering
- Generates complete plugin structure with all files
- Includes pyproject.toml, targets, rules, goals, tests
- Ready-to-use patterns for linters, code generators, custom targets

### 2. pants-plugin-dev
**Develop and modify existing Pants plugins**

- Add new targets, fields, rules, goals to existing plugins
- Debug common issues (rules not firing, caching problems, type errors)
- Write and improve tests with RuleRunner
- Upgrade plugins for new Pants versions
- Performance optimization guidance

### 3. pants-oracle
**Comprehensive Pants knowledge base**

- Expert answers to any Pants question
- Complete API reference for targets, fields, rules
- Engine architecture and caching internals
- Testing patterns with RuleRunner
- Best practices and common pitfalls

## Installation

```bash
# Add the marketplace
/plugin marketplace add jaymd96/pants-plugin-creator

# Install one or both plugins
/plugin install pants-plugin-creator@pants-plugins
/plugin install pants-plugin-dev@pants-plugins
/plugin install pants-oracle@pants-plugins
```

Update anytime with:
```bash
/plugin marketplace update pants-plugins
```

## Usage

### Creating a New Plugin

```
/pants-plugin-creator:create-pants-plugin
```

Or ask naturally:
- "Create a Pants plugin that lints shell scripts"
- "I need a plugin for generating code from protobuf"

### Developing an Existing Plugin

```
/pants-plugin-dev:develop-pants-plugin
```

Or ask naturally:
- "Add a new target type to my plugin"
- "Debug why my rule isn't being called"
- "Write tests for my Pants plugin"
- "Upgrade my plugin for Pants 2.19"

### Pants Knowledge Base (Oracle)

```
/pants-oracle:pants-oracle
```

Or ask naturally:
- "How does Pants caching work?"
- "What's the difference between Digest and Snapshot?"
- "How do I create a custom field type?"
- "Explain the rule graph"

## What pants-plugin-creator Generates

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

Both plugins include comprehensive references:

**pants-plugin-creator:**
- `quickstart.md` - 5-minute intro
- `api-reference.md` - Target & Rules API
- `templates.md` - File templates
- `patterns/` - Linter, codegen, custom target patterns

**pants-plugin-dev:**
- `debugging.md` - Troubleshooting guide
- `testing.md` - RuleRunner patterns
- `upgrading.md` - Version migration
- `performance.md` - Optimization techniques

**pants-oracle:**
- `architecture.md` - Engine internals and design
- `targets.md` - Complete Target API reference
- `rules.md` - Complete Rules API reference
- `testing.md` - Testing patterns and utilities

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
