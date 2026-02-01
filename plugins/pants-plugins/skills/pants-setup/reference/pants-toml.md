# pants.toml Configuration Reference

Complete reference for configuring Pants via pants.toml.

## File Location

`pants.toml` must be in the repository root (same directory as the `.git` folder).

## Complete Example

```toml
[GLOBAL]
# Pants version to use (required)
pants_version = "2.30.1"

# Backend packages to load
backend_packages = [
    # Core Python
    "pants.backend.python",

    # Linting (choose one or more)
    "pants.backend.python.lint.ruff",

    # Type checking (choose one)
    "pants.backend.python.typecheck.mypy",

    # Other languages (as needed)
    "pants.backend.docker",
    "pants.backend.shell",
]

# Third-party plugins
plugins = [
    "jaymd96-pants-baseline==0.1.0",
]

# Build root (usually current directory)
build_root = "."

# Parallelism
process_execution_local_parallelism = "%(num_cpus)s"

# Colors in output
colors = true

# Remote caching (optional)
# remote_cache_read = true
# remote_cache_write = true
# remote_store_address = "grpc://cache.example.com:9092"

[anonymous-telemetry]
enabled = false

# ============================================================================
# PYTHON CONFIGURATION
# ============================================================================

[python]
# Python interpreter constraints
interpreter_constraints = ["CPython>=3.13,<4"]

# Enable lockfile generation
enable_resolves = true

# Default resolve name
default_resolve = "python-default"

# Resolve definitions
[python.resolves]
python-default = "3rdparty/python/default.lock"

[python-infer]
# Automatic dependency inference
use_rust_parser = true
unowned_dependency_behavior = "warning"

# ============================================================================
# TOOL-SPECIFIC CONFIGURATION
# ============================================================================

[ruff]
# Ruff linting configuration
config = "pyproject.toml"  # Use pyproject.toml for detailed config

[mypy]
# MyPy configuration
config = "pyproject.toml"
args = ["--strict"]

[pytest]
# pytest configuration
args = ["-v", "--tb=short"]

[coverage-py]
# Coverage configuration
report = ["term", "html"]
fail_under = 80

# ============================================================================
# SOURCE ROOTS
# ============================================================================

[source]
# Define source roots
root_patterns = [
    "src",
    "src/python",
    "tests",
]

# ============================================================================
# DOCKER CONFIGURATION (if using)
# ============================================================================

[docker]
# Docker configuration
default_repository = "{directory}/{name}"

# ============================================================================
# SHELL CONFIGURATION (if using)
# ============================================================================

[shell-setup]
# Shell configuration
dependency_inference = true
```

## Section Reference

### [GLOBAL]

| Option | Type | Description |
|--------|------|-------------|
| `pants_version` | string | Required. Pants version to use |
| `backend_packages` | list | Backend packages to load |
| `plugins` | list | Third-party plugins to install |
| `build_root` | string | Root directory for builds |
| `colors` | bool | Enable colored output |
| `process_execution_local_parallelism` | string | Parallel process count |

### [python]

| Option | Type | Description |
|--------|------|-------------|
| `interpreter_constraints` | list | Python version constraints |
| `enable_resolves` | bool | Enable lockfile support |
| `default_resolve` | string | Default resolve name |

### [source]

| Option | Type | Description |
|--------|------|-------------|
| `root_patterns` | list | Directories that are source roots |

## Environment Variables

Override any option via environment:

```bash
# Format: PANTS_<SECTION>_<OPTION>
export PANTS_PYTHON_INTERPRETER_CONSTRAINTS="['CPython>=3.12']"
export PANTS_GLOBAL_COLORS=false
```

## Command Line Override

Override any option via CLI:

```bash
pants --python-interpreter-constraints="['CPython>=3.12']" test ::
pants --no-colors lint ::
```
