---
description: Discover available Pants commands, goals, and targets in the current repository - show what you can do with Pants
user_invocable: true
triggers:
  - "pants commands"
  - "pants goals"
  - "what can pants do"
  - "available pants"
  - "list pants"
  - "pants help"
  - "show pants"
  - "discover pants"
---

# Pants Command Discovery

You help users discover what Pants commands and capabilities are available in their current repository. This skill analyzes the pants.toml configuration and reports on available goals, targets, and tools.

## Quick Discovery Commands

When a user wants to know what they can do with Pants:

### 1. List All Available Goals

```bash
pants goals
```

This shows all available commands like `lint`, `fmt`, `test`, `check`, etc.

### 2. Get Help on a Specific Goal

```bash
pants help <goal>
pants help lint
pants help test
```

### 3. List All Targets in the Repo

```bash
pants list ::
```

### 4. Show Target Details

```bash
pants peek <target>
pants peek src/python:lib
```

---

## Discovery Workflow

When discovering Pants capabilities:

### Step 1: Check if Pants is Configured

```bash
# Check for pants.toml
if [ -f pants.toml ]; then
    echo "Pants is configured"
    cat pants.toml | head -30
else
    echo "No pants.toml found - Pants not set up"
fi
```

### Step 2: Identify Loaded Backends

Read the `backend_packages` from pants.toml to understand available features:

```bash
grep -A 20 "backend_packages" pants.toml
```

### Step 3: List Available Goals

```bash
pants goals
```

### Step 4: Show Configured Tools

Based on backends, explain what tools are available.

---

## Goal Categories

### Code Quality

| Goal | Command | Description |
|------|---------|-------------|
| `lint` | `pants lint ::` | Run all configured linters |
| `fmt` | `pants fmt ::` | Format code with configured formatters |
| `check` | `pants check ::` | Run type checkers |
| `fix` | `pants fix ::` | Auto-fix issues |

### Testing

| Goal | Command | Description |
|------|---------|-------------|
| `test` | `pants test ::` | Run tests |
| `test --coverage` | `pants test --coverage ::` | Run with coverage |

### Building & Packaging

| Goal | Command | Description |
|------|---------|-------------|
| `package` | `pants package ::` | Build packages |
| `publish` | `pants publish ::` | Publish packages |
| `run` | `pants run src:app` | Run an executable target |

### Exploration

| Goal | Command | Description |
|------|---------|-------------|
| `list` | `pants list ::` | List all targets |
| `peek` | `pants peek target` | Show target details |
| `dependencies` | `pants dependencies target` | Show dependencies |
| `dependents` | `pants dependents target` | Show reverse dependencies |
| `paths` | `pants paths from to` | Show dependency path |
| `filedeps` | `pants filedeps target` | Show source files |
| `roots` | `pants roots` | Show source roots |

### Maintenance

| Goal | Command | Description |
|------|---------|-------------|
| `tailor` | `pants tailor ::` | Auto-generate BUILD files |
| `update-build-files` | `pants update-build-files ::` | Update BUILD file formatting |
| `export` | `pants export ::` | Export virtual environment |
| `generate-lockfiles` | `pants generate-lockfiles` | Update lock files |

---

## Backend-Specific Goals

### Python Baseline Plugin

If `python_baseline` is in backends:

| Goal | Command | Description |
|------|---------|-------------|
| `baseline-lint` | `pants baseline-lint ::` | Run Ruff linting |
| `baseline-fmt` | `pants baseline-fmt ::` | Run Ruff formatting |
| `baseline-typecheck` | `pants baseline-typecheck ::` | Run ty type checking |
| `baseline-test` | `pants baseline-test ::` | Run pytest with coverage |
| `baseline-audit` | `pants baseline-audit ::` | Run uv security audit |

### Docker Backend

If `pants.backend.docker` is in backends:

| Goal | Command | Description |
|------|---------|-------------|
| `package` | `pants package ::` | Build Docker images |
| `publish` | `pants publish ::` | Push to registry |
| `run` | `pants run image:target` | Run container |

### Shell Backend

If `pants.backend.shell` is in backends:

| Goal | Command | Description |
|------|---------|-------------|
| `lint` | `pants lint ::` | Run shellcheck |
| `fmt` | `pants fmt ::` | Format with shfmt |
| `run` | `pants run script.sh` | Execute shell script |

---

## Target Discovery

### List All Targets

```bash
pants list ::
```

### List Targets by Type

```bash
# Python targets
pants list --filter-target-type=python_source ::

# Test targets
pants list --filter-target-type=python_test ::

# Docker targets
pants list --filter-target-type=docker_image ::
```

### Show Target Definition

```bash
pants peek src/python:mylib
```

Output:
```json
{
  "address": "src/python:mylib",
  "target_type": "python_sources",
  "dependencies": ["3rdparty:requests"],
  "sources": ["**/*.py"]
}
```

---

## Configuration Discovery

### Show All Options for a Tool

```bash
pants help-advanced <tool>
pants help-advanced ruff
pants help-advanced mypy
pants help-advanced pytest
```

### Show Current Configuration

```bash
pants help-all | grep -A 5 "\[ruff\]"
```

### Validate Configuration

```bash
# Check for issues
pants lint --check ::
pants fmt --check ::
```

---

## Common Discovery Scenarios

### "What linters are configured?"

```bash
# Check backends
grep -E "(lint|ruff|flake8|pylint|black|isort)" pants.toml

# See what lint does
pants help lint
```

### "What can I run tests with?"

```bash
# Check if pytest is available
pants goals | grep test

# Show test options
pants help test
```

### "What targets exist in src/?"

```bash
pants list src/::
```

### "What depends on this target?"

```bash
pants dependents src/lib:mylib
```

### "Why is this target being built?"

```bash
pants paths src/lib:mylib src/app:main
```

---

## Quick Reference Card

```bash
# Goals
pants goals                    # List all goals
pants help <goal>              # Help for specific goal
pants help-all                 # All options

# Targets
pants list ::                  # List all targets
pants list dir/::              # List targets in directory
pants peek <target>            # Show target details

# Dependencies
pants dependencies <target>    # What does this depend on?
pants dependents <target>      # What depends on this?
pants paths <from> <to>        # Dependency path

# Files
pants filedeps <target>        # Source files for target
pants roots                    # Source roots

# Common Goals
pants lint ::                  # Lint everything
pants fmt ::                   # Format everything
pants check ::                 # Type check
pants test ::                  # Run tests
pants package ::               # Build packages
pants tailor ::                # Generate BUILD files
```

---

## Troubleshooting Discovery

### "pants: command not found"

Pants isn't installed. Run:
```bash
curl -sSL https://static.pantsbuild.org/setup/get-pants.sh | bash
```

### "No targets found"

Check:
1. BUILD files exist
2. Source patterns match files
3. Run `pants tailor ::` to auto-generate

### "Goal not found"

The backend isn't loaded. Add to `backend_packages` in pants.toml.

---

## Reference

- [Goal Reference](reference/goals.md)
- [Target Types](reference/target-types.md)
