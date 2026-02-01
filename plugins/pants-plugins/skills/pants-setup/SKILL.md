---
description: Setup and manage Pants build system in repositories - add baseline plugins, discover packages, configure pants.toml, search for plugins, and housekeeping tasks
user_invocable: true
triggers:
  - "setup pants"
  - "add pants"
  - "install pants plugin"
  - "pants baseline"
  - "configure pants"
  - "pants.toml"
  - "add python baseline"
  - "remove pants"
  - "pants packages"
  - "pants housekeeping"
  - "search pants plugins"
  - "find pants plugins"
  - "list pants plugins"
---

# Pants Setup & Housekeeping

You help users set up and manage the Pants build system in their repositories. This includes adding plugins, configuring pants.toml, discovering available packages, and general housekeeping tasks.

## Capabilities

### 1. Add Python Baseline Plugin

Add the opinionated Python code quality baseline to a repo:

```toml
# In pants.toml
[GLOBAL]
backend_packages = [
    "pants.backend.python",
    "pants_baseline",
]
plugins = ["jaymd96-pants-baseline==0.1.0"]

[python-baseline]
python_version = "3.11"
line_length = 120
coverage_threshold = 80
```

### 2. Search for Available Plugins

Search GitHub and PyPI for jaymd96 Pants plugins:

```bash
# Search GitHub repos with pants-plugin label
gh repo list jaymd96 --topic pants-plugin --json name,description

# Check PyPI for a specific plugin
curl -s "https://pypi.org/pypi/jaymd96-pants-baseline/json" | jq '.info.version'

# List all installed pants plugins
pip list | grep jaymd96-pants
```

### 3. Discover Built-in Backend Packages

Help users discover Pants backend packages and third-party plugins:

**Built-in Backend Packages:**
| Package | Purpose |
|---------|---------|
| `pants.backend.python` | Python support |
| `pants.backend.python.lint.ruff` | Ruff linting |
| `pants.backend.python.typecheck.mypy` | MyPy type checking |
| `pants.backend.docker` | Docker support |
| `pants.backend.shell` | Shell script support |
| `pants.backend.experimental.python` | Experimental Python features |

### 4. Remove/Disable Packages

Help users clean up unused packages from pants.toml.

### 5. General Housekeeping

- Validate pants.toml configuration
- Check for outdated plugins
- Optimize backend package selection
- Set up recommended defaults

---

## Workflow

When a user wants to set up or manage Pants:

### Step 1: Assess Current State

Check if Pants is already set up in the repo:

```bash
# Check for pants.toml
ls pants.toml 2>/dev/null && echo "Pants is configured" || echo "No pants.toml found"

# Check Pants version if installed
pants --version 2>/dev/null || echo "Pants not installed"
```

### Step 2: Determine User Intent

Ask what they want to do:
- **New Setup**: Create pants.toml from scratch
- **Add Plugin**: Add a specific plugin to existing setup
- **Remove Plugin**: Remove unused plugins
- **Discover**: Show available packages
- **Validate**: Check current configuration

### Step 3: Execute

Based on intent, perform the appropriate action.

---

## Adding Python Baseline

When adding the Python baseline plugin:

### 1. Check Prerequisites

```bash
# Ensure pants.toml exists
if [ ! -f pants.toml ]; then
    echo "Creating pants.toml..."
fi
```

### 2. Create/Update pants.toml

```toml
[GLOBAL]
pants_version = "2.20.0"
backend_packages = [
    "pants.backend.python",
    "pants_baseline",
]
plugins = [
    "jaymd96-pants-baseline==0.1.0",
]

[python]
interpreter_constraints = ["CPython>=3.11,<4"]

# Python Baseline Configuration
[python-baseline]
enabled = true
python_version = "3.11"
line_length = 120
src_roots = ["src"]
test_roots = ["tests"]
coverage_threshold = 80
strict_mode = true

# Ruff Configuration
[ruff]
select = ["E", "F", "I", "B", "UP", "C4", "SIM", "RUF"]
ignore = ["E501"]
quote_style = "double"
fix = true

# ty Configuration (Astral's type checker)
[ty]
strict = true
output_format = "text"

# uv Configuration
[uv]
audit_enabled = true
```

### 3. Create BUILD File

```python
# BUILD (in project root)
baseline_python_project(
    name="project",
    sources=["src/**/*.py"],
    test_sources=["tests/**/*.py"],
    python_version="3.11",
    line_length=120,
    strict=True,
    coverage_threshold=80,
)
```

### 4. Verify Setup

```bash
# Test the setup
pants goals
pants baseline-lint ::
```

---

## Available Backend Packages

### Core Python
| Package | Description |
|---------|-------------|
| `pants.backend.python` | Core Python support |
| `pants.backend.python.mixed_interpreter_constraints` | Multiple Python versions |

### Linting & Formatting
| Package | Description |
|---------|-------------|
| `pants.backend.python.lint.ruff` | Ruff linter (recommended) |
| `pants.backend.python.lint.flake8` | Flake8 linter |
| `pants.backend.python.lint.pylint` | Pylint linter |
| `pants.backend.python.lint.black` | Black formatter |
| `pants.backend.python.lint.isort` | Import sorter |
| `pants.backend.python.lint.bandit` | Security linter |

### Type Checking
| Package | Description |
|---------|-------------|
| `pants.backend.python.typecheck.mypy` | MyPy type checker |
| `pants.backend.python.typecheck.pyright` | Pyright type checker |

### Testing
| Package | Description |
|---------|-------------|
| `pants.backend.python.pytest` | pytest (included in core) |

### Packaging
| Package | Description |
|---------|-------------|
| `pants.backend.python.packaging.pyoxidizer` | PyOxidizer binaries |
| `pants.backend.experimental.python.packaging.pyoxidizer` | Experimental PyOxidizer |

### Other Languages
| Package | Description |
|---------|-------------|
| `pants.backend.docker` | Docker support |
| `pants.backend.shell` | Shell scripts |
| `pants.backend.javascript` | JavaScript/TypeScript |
| `pants.backend.go` | Go support |
| `pants.backend.java` | Java support |
| `pants.backend.scala` | Scala support |

---

## Third-Party Plugins (jaymd96)

### Available Plugins

| Plugin | GitHub | PyPI | Description |
|--------|--------|------|-------------|
| baseline | [pants-baseline](https://github.com/jaymd96/pants-baseline) | `jaymd96-pants-baseline` | Python quality baseline (Ruff, ty, uv, pytest) |

### Searching for Plugins

**Search GitHub:**
```bash
# List all jaymd96 pants plugins
gh repo list jaymd96 --topic pants-plugin --json name,description

# Search with more details
gh repo list jaymd96 --topic pants-plugin --json name,description,url --jq '.[] | "\(.name): \(.description)"'
```

**Search PyPI:**
```bash
# Check if a plugin exists
curl -s "https://pypi.org/pypi/jaymd96-pants-{name}/json" | jq '.info | {name, version}'

# Check available versions
curl -s "https://pypi.org/pypi/jaymd96-pants-baseline/json" | jq '.releases | keys'
```

### Plugin Naming Conventions

When creating or searching for plugins, follow these conventions:

| Component | Convention | Example |
|-----------|------------|---------|
| GitHub Repo | `pants-{name}` | `pants-baseline` |
| PyPI Package | `jaymd96-pants-{name}` | `jaymd96-pants-baseline` |
| Python Package | `pants_{name}` | `pants_baseline` |
| GitHub Labels | `pants-plugin`, `{name}` | `pants-plugin`, `baseline` |

See [Naming Conventions](reference/naming-conventions.md) for full details.

---

## Removing Packages

To remove a package:

1. **Remove from backend_packages** in pants.toml
2. **Remove from plugins** if it's a third-party plugin
3. **Remove related configuration sections**
4. **Clean up BUILD files** that use removed target types

Example - removing Flake8 after switching to Ruff:

```toml
# Before
[GLOBAL]
backend_packages = [
    "pants.backend.python",
    "pants.backend.python.lint.flake8",  # REMOVE
    "pants.backend.python.lint.ruff",
]

# After
[GLOBAL]
backend_packages = [
    "pants.backend.python",
    "pants.backend.python.lint.ruff",
]
```

---

## Validation Checklist

When validating a Pants setup:

- [ ] `pants_version` is set and recent
- [ ] `backend_packages` includes all needed backends
- [ ] `plugins` are versioned (avoid floating versions)
- [ ] `interpreter_constraints` match project requirements
- [ ] Source roots are correctly configured
- [ ] No duplicate or conflicting backends

```bash
# Validate configuration
pants help-all | head -20

# Check for issues
pants lint :: --check

# Verify targets are discovered
pants list ::
```

---

## Quick Setup Commands

```bash
# Initialize new Pants repo
pants # Downloads Pants if not installed

# Show current configuration
pants help-advanced global

# List all available goals
pants goals

# Show backends in use
grep -A 20 "backend_packages" pants.toml
```

---

## Reference

- [pants.toml Configuration](reference/pants-toml.md)
- [Available Backends](reference/backends.md)
- [Plugin Installation](reference/plugins.md)
- [Search for Plugins](reference/search-plugins.md)
- [Naming Conventions](reference/naming-conventions.md)
