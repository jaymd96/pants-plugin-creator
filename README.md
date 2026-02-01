# Pants Plugin Creator

A Claude Code plugin for creating production-grade Pants build system plugins.

## Features

- **Guided Workflow**: Interactive discovery of plugin requirements
- **Complete Generation**: Creates full plugin structure with all necessary files
- **Best Practices**: Follows official Pants plugin development guidelines
- **Comprehensive Reference**: Bundled documentation for Target API, Rules API, and patterns
- **Pattern Templates**: Ready-to-use patterns for linters, code generators, and custom targets

## Installation

### Option 1: Install from GitHub (Recommended)

```bash
# Add the marketplace
/plugin marketplace add jaymd96/pants-plugin-creator

# Install the plugin
/plugin install pants-plugin-creator@pants-plugins
```

Users can then update anytime with:
```bash
/plugin marketplace update pants-plugins
```

### Option 2: Install from Local Directory

```bash
claude --plugin-dir /path/to/pants-plugin-creator
```

### Option 3: Install Globally

```bash
cp -r pants-plugin-creator ~/.claude/plugins/
```

## Usage

Invoke the skill by asking Claude to create a Pants plugin:

```
/create-pants-plugin
```

Or naturally:

```
"Create a Pants plugin that lints shell scripts with shellcheck"
"I need a plugin for generating code from protobuf definitions"
"Help me build a custom Pants backend for my documentation system"
```

## What Gets Generated

The skill creates a complete plugin structure:

```
my-plugin/
├── pyproject.toml           # Hatch-based build configuration
├── README.md                # Plugin documentation
├── LICENSE                  # Apache 2.0 license
├── Makefile                 # Development commands
├── .gitignore
├── src/my_plugin/
│   ├── __init__.py
│   ├── version.py
│   ├── register.py          # Plugin entry point (REQUIRED)
│   ├── subsystem.py         # Configuration options
│   ├── targets.py           # Custom target types
│   ├── rules.py             # Business logic
│   └── goals.py             # User commands
├── tests/
│   ├── conftest.py          # Pytest fixtures
│   └── unit/
└── docs/
```

## Reference Documentation

The skill includes comprehensive reference materials:

- **quickstart.md** - 5-minute introduction to Pants plugins
- **developer-guide.md** - Best practices, common pitfalls, production checklist
- **api-reference.md** - Complete Target API and Rules API documentation
- **templates.md** - File templates for all plugin components
- **patterns/** - Ready-to-use patterns:
  - `linter.md` - External linter integration
  - `codegen.md` - Code generation from definitions
  - `custom-target.md` - Custom BUILD file syntax

## Development Workflow

After generating a plugin:

```bash
cd my-plugin
hatch env create       # Set up development environment
hatch run test         # Run tests
hatch run fmt          # Format code
hatch run lint         # Check code quality
hatch run all          # Run everything
```

## Publishing

### To PyPI

```bash
hatch build
hatch publish
```

### As In-Repo Plugin

```toml
# pants.toml
[GLOBAL]
pythonpath = ["%(buildroot)s/plugins/my-plugin/src"]
backend_packages = ["my_plugin"]
```

## Key Concepts

The skill teaches core Pants concepts:

- **Targets**: Metadata about code in BUILD files
- **Fields**: Individual pieces of target metadata
- **Rules**: Pure async functions that process targets
- **Goals**: User-facing commands (`pants my-goal`)
- **Subsystems**: Configuration in pants.toml

## Requirements

- Claude Code CLI
- Python 3.11+
- Hatch (for plugin development)

## Resources

- [Official Pants Plugin Docs](https://www.pantsbuild.org/stable/docs/writing-plugins/overview)
- [Pants GitHub](https://github.com/pantsbuild/pants)
- [Pants Slack Community](https://pantsbuild.slack.com)

## License

Apache License 2.0
