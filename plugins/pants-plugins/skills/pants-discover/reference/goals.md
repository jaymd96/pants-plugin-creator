# Pants Goals Reference

Complete reference of all built-in Pants goals.

## Core Goals

### lint
Run all configured linters.

```bash
pants lint ::                    # Lint everything
pants lint src/::                # Lint specific directory
pants lint --only=ruff ::        # Run only specific linter
pants lint --skip=flake8 ::      # Skip specific linter
```

### fmt
Format code with configured formatters.

```bash
pants fmt ::                     # Format everything
pants fmt --only=black ::        # Format with specific tool
pants fmt --check ::             # Check without modifying (CI mode)
```

### check
Run type checkers.

```bash
pants check ::                   # Type check everything
pants check --only=mypy ::       # Use specific checker
```

### test
Run tests.

```bash
pants test ::                    # Run all tests
pants test tests/::              # Run tests in directory
pants test --coverage ::         # With coverage
pants test -- -k "test_name"     # Pass args to pytest
pants test --force ::            # Ignore cache
```

### fix
Auto-fix issues that linters can fix.

```bash
pants fix ::                     # Fix all auto-fixable issues
pants fix --only=ruff ::         # Fix with specific tool
```

### run
Run an executable target.

```bash
pants run src:app                # Run application
pants run src:app -- arg1 arg2   # With arguments
```

### package
Build distributable packages.

```bash
pants package ::                 # Package everything
pants package src:dist           # Package specific target
```

### publish
Publish packages to registries.

```bash
pants publish ::                 # Publish everything
pants publish --dry-run ::       # Preview without publishing
```

## Exploration Goals

### list
List targets matching a pattern.

```bash
pants list ::                    # All targets
pants list src/::                # Targets in src/
pants list --filter-target-type=python_test ::  # Only tests
```

### peek
Show detailed target information.

```bash
pants peek src:lib               # JSON output of target
pants peek --include-dep-rules src:lib  # Include dependency rules
```

### dependencies
Show what a target depends on.

```bash
pants dependencies src:lib       # Direct dependencies
pants dependencies --transitive src:lib  # All dependencies
```

### dependents
Show what depends on a target.

```bash
pants dependents src:lib         # Direct dependents
pants dependents --transitive src:lib  # All dependents
```

### paths
Find dependency paths between targets.

```bash
pants paths src:lib src:app      # Path from lib to app
```

### filedeps
Show source files for a target.

```bash
pants filedeps src:lib           # Files for target
pants filedeps --transitive src:lib  # Include dependencies
```

### roots
Show configured source roots.

```bash
pants roots                      # List all source roots
```

## Maintenance Goals

### tailor
Auto-generate BUILD files.

```bash
pants tailor ::                  # Generate for whole repo
pants tailor src/::              # Generate for directory
pants tailor --check ::          # Check what would be generated
```

### update-build-files
Format and update BUILD files.

```bash
pants update-build-files ::      # Update all BUILD files
pants update-build-files --check ::  # Check mode
```

### export
Export a virtual environment.

```bash
pants export ::                  # Export venv
pants export --resolve=python-default  # Specific resolve
```

### generate-lockfiles
Generate/update dependency lock files.

```bash
pants generate-lockfiles         # All lockfiles
pants generate-lockfiles --resolve=python-default  # Specific
```

## Utility Goals

### help
Show help for goals and options.

```bash
pants help                       # General help
pants help lint                  # Help for lint goal
pants help-advanced lint         # All options for lint
pants help-all                   # All available options
```

### goals
List all available goals.

```bash
pants goals                      # List all goals
```

### version
Show Pants version.

```bash
pants version                    # Show version
```

### repl
Start a Python REPL with dependencies.

```bash
pants repl src:lib               # REPL with lib's dependencies
```

## Docker Goals (with docker backend)

### package (for docker)
Build Docker images.

```bash
pants package image:app          # Build image
```

### publish (for docker)
Push images to registry.

```bash
pants publish image:app          # Push to registry
```

### run (for docker)
Run a container.

```bash
pants run image:app              # Run container
```

## Python Baseline Goals (with baseline plugin)

### baseline-lint
Run Ruff linting with baseline config.

```bash
pants baseline-lint ::
```

### baseline-fmt
Run Ruff formatting with baseline config.

```bash
pants baseline-fmt ::
```

### baseline-typecheck
Run ty type checking.

```bash
pants baseline-typecheck ::
```

### baseline-test
Run pytest with coverage thresholds.

```bash
pants baseline-test ::
```

### baseline-audit
Run uv security audit.

```bash
pants baseline-audit ::
```

## Goal Options

Most goals support these common options:

| Option | Description |
|--------|-------------|
| `--only=<tool>` | Run only specified tool |
| `--skip=<tool>` | Skip specified tool |
| `--check` | Check mode (no modifications) |
| `--force` | Ignore cache |
| `--changed-since=<ref>` | Only targets changed since git ref |
| `--tag=<tag>` | Only targets with tag |

Example:
```bash
pants lint --only=ruff --changed-since=main ::
```
