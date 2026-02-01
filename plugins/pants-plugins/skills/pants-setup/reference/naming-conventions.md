# Pants Plugin Naming Conventions

Standard naming conventions for Pants plugins to ensure discoverability and consistency.

## Overview

| Component | Convention | Example |
|-----------|------------|---------|
| GitHub Repo | `pants-{name}` | `pants-baseline` |
| PyPI Package | `jaymd96-pants-{name}` | `jaymd96-pants-baseline` |
| Python Package | `pants_{name}` | `pants_baseline` |
| GitHub Labels | `pants-plugin`, `{name}` | `pants-plugin`, `baseline` |

## GitHub Repository

### Naming
- Pattern: `pants-{plugin-name}`
- Examples:
  - `pants-baseline` - Python baseline quality plugin
  - `pants-docker-lint` - Docker linting plugin
  - `pants-protobuf-gen` - Protobuf code generator

### Labels (Required)
Every Pants plugin repo should have these labels:

1. **`pants-plugin`** - Required for discoverability
2. **`{plugin-name}`** - The specific plugin name (e.g., `baseline`)
3. Optional: `python`, `linting`, `testing`, etc.

### Repository Setup

```bash
# Create repo with gh CLI
gh repo create pants-{name} --public --description "Pants plugin for {description}"

# Add labels
gh label create pants-plugin --color "0052CC" --description "Pants build system plugin"
gh label create {name} --color "1D76DB" --description "Plugin: {name}"

# Add topics (for GitHub search)
gh repo edit --add-topic pants-plugin --add-topic pants --add-topic {name}
```

## PyPI Package

### Naming
- Pattern: `jaymd96-pants-{plugin-name}`
- This namespacing prevents conflicts with other packages
- Examples:
  - `jaymd96-pants-baseline`
  - `jaymd96-pants-docker-lint`

### In pyproject.toml

```toml
[project]
name = "jaymd96-pants-{name}"
version = "0.1.0"
description = "Pants plugin for {description}"
keywords = [
    "pants",
    "plugin",
    "{name}",
    # ... other relevant keywords
]

[project.urls]
Homepage = "https://github.com/jaymd96/pants-{name}"
Repository = "https://github.com/jaymd96/pants-{name}.git"
```

## Python Package (Internal)

### Naming
- Pattern: `pants_{name}` (underscores, not hyphens)
- This is the importable package name
- Examples:
  - `pants_baseline`
  - `pants_docker_lint`

### Directory Structure

```
pants-{name}/                    # Repo name (hyphens)
├── src/
│   └── pants_{name}/           # Python package (underscores)
│       ├── __init__.py
│       ├── __about__.py
│       ├── register.py
│       └── ...
├── pyproject.toml
└── ...
```

### In pyproject.toml

```toml
[tool.hatch.build.targets.wheel]
packages = ["src/pants_{name}"]

[tool.hatch.version]
path = "src/pants_{name}/__about__.py"

[tool.ruff.lint.isort]
known-first-party = ["pants_{name}"]
```

## Backend Package Name

When users add your plugin to pants.toml:

```toml
[GLOBAL]
backend_packages = [
    "pants.backend.python",
    "pants_{name}",              # Your plugin's backend
]
plugins = [
    "jaymd96-pants-{name}==0.1.0",  # PyPI package name
]
```

## Complete Example: pants-baseline

| Component | Value |
|-----------|-------|
| GitHub Repo | `jaymd96/pants-baseline` |
| GitHub Labels | `pants-plugin`, `baseline` |
| PyPI Package | `jaymd96-pants-baseline` |
| Python Package | `pants_baseline` |
| Backend Name | `pants_baseline` |

### pyproject.toml

```toml
[project]
name = "jaymd96-pants-baseline"
version = "0.1.0"

[project.urls]
Homepage = "https://github.com/jaymd96/pants-baseline"
Repository = "https://github.com/jaymd96/pants-baseline.git"

[tool.hatch.build.targets.wheel]
packages = ["src/pants_baseline"]
```

### pants.toml (for users)

```toml
[GLOBAL]
backend_packages = ["pants_baseline"]
plugins = ["jaymd96-pants-baseline==0.1.0"]
```

## Quick Reference

When creating a new plugin called `foo`:

```bash
# 1. Create GitHub repo
gh repo create pants-foo --public

# 2. Add labels
gh label create pants-plugin
gh label create foo

# 3. Set up pyproject.toml
# name = "jaymd96-pants-foo"
# packages = ["src/pants_foo"]

# 4. Create Python package
mkdir -p src/pants_foo
touch src/pants_foo/__init__.py

# 5. Publish to PyPI
hatch build
hatch publish
```
