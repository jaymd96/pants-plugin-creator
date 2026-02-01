# Available Backend Packages

Complete list of Pants backend packages organized by category.

## Python

### Core
| Package | Description |
|---------|-------------|
| `pants.backend.python` | Core Python support (required for Python projects) |
| `pants.backend.python.mixed_interpreter_constraints` | Support for multiple Python versions |

### Linting & Formatting
| Package | Tool | Speed | Notes |
|---------|------|-------|-------|
| `pants.backend.python.lint.ruff` | Ruff | Fastest | Recommended - replaces Black, isort, Flake8 |
| `pants.backend.python.lint.black` | Black | Fast | Formatter only |
| `pants.backend.python.lint.isort` | isort | Fast | Import sorting only |
| `pants.backend.python.lint.flake8` | Flake8 | Medium | Legacy linter |
| `pants.backend.python.lint.pylint` | Pylint | Slow | Most comprehensive |
| `pants.backend.python.lint.bandit` | Bandit | Fast | Security-focused |
| `pants.backend.python.lint.docformatter` | docformatter | Fast | Docstring formatter |
| `pants.backend.python.lint.yapf` | YAPF | Medium | Alternative formatter |
| `pants.backend.python.lint.autoflake` | autoflake | Fast | Remove unused imports |
| `pants.backend.python.lint.pyupgrade` | pyupgrade | Fast | Upgrade Python syntax |

### Type Checking
| Package | Tool | Speed | Notes |
|---------|------|-------|-------|
| `pants.backend.python.typecheck.mypy` | MyPy | Slow | Reference implementation |
| `pants.backend.python.typecheck.pyright` | Pyright | Fast | Microsoft's type checker |

### Testing
| Package | Description |
|---------|-------------|
| (included in core) | pytest support is built into `pants.backend.python` |

### Packaging
| Package | Description |
|---------|-------------|
| `pants.backend.python.packaging.pyoxidizer` | Build standalone executables |
| `pants.backend.experimental.python.packaging.pyoxidizer` | Experimental PyOxidizer features |

### Framework Support
| Package | Description |
|---------|-------------|
| `pants.backend.python.framework.django` | Django support |
| `pants.backend.python.framework.stevedore` | Stevedore plugin discovery |

## Docker

| Package | Description |
|---------|-------------|
| `pants.backend.docker` | Docker image building |
| `pants.backend.docker.lint.hadolint` | Dockerfile linting |
| `pants.backend.experimental.docker` | Experimental Docker features |

## Shell

| Package | Description |
|---------|-------------|
| `pants.backend.shell` | Shell script support |
| `pants.backend.shell.lint.shellcheck` | Shell script linting |
| `pants.backend.shell.lint.shfmt` | Shell script formatting |

## JavaScript/TypeScript

| Package | Description |
|---------|-------------|
| `pants.backend.javascript` | JavaScript support |
| `pants.backend.javascript.lint.prettier` | Prettier formatting |

## Go

| Package | Description |
|---------|-------------|
| `pants.backend.go` | Go support |
| `pants.backend.go.lint.golangci_lint` | Go linting |

## JVM (Java/Scala/Kotlin)

| Package | Description |
|---------|-------------|
| `pants.backend.java` | Java support |
| `pants.backend.scala` | Scala support |
| `pants.backend.kotlin` | Kotlin support |
| `pants.backend.jvm.lint.google_java_format` | Java formatting |

## Other

| Package | Description |
|---------|-------------|
| `pants.backend.adhoc` | Ad-hoc scripts and commands |
| `pants.backend.build_files.fmt.black` | Format BUILD files with Black |
| `pants.backend.build_files.fmt.yapf` | Format BUILD files with YAPF |
| `pants.backend.codegen.protobuf.python` | Protobuf code generation |
| `pants.backend.codegen.thrift.apache.python` | Thrift code generation |
| `pants.backend.url_handlers.s3` | S3 URL support |

## Experimental Packages

Experimental packages may change between releases:

| Package | Description |
|---------|-------------|
| `pants.backend.experimental.python` | Experimental Python features |
| `pants.backend.experimental.python.lint.ruff` | Experimental Ruff features |
| `pants.backend.experimental.docker` | Experimental Docker features |
| `pants.backend.experimental.go` | Experimental Go features |
| `pants.backend.experimental.java` | Experimental Java features |

## Recommended Combinations

### Minimal Python
```toml
backend_packages = [
    "pants.backend.python",
]
```

### Modern Python (Recommended)
```toml
backend_packages = [
    "pants.backend.python",
    "pants.backend.python.lint.ruff",
    "pants.backend.python.typecheck.mypy",
]
```

### Full Stack (Python + Docker + Shell)
```toml
backend_packages = [
    "pants.backend.python",
    "pants.backend.python.lint.ruff",
    "pants.backend.python.typecheck.mypy",
    "pants.backend.docker",
    "pants.backend.docker.lint.hadolint",
    "pants.backend.shell",
    "pants.backend.shell.lint.shellcheck",
]
```

### With Python Baseline Plugin
```toml
backend_packages = [
    "pants.backend.python",
    "python_baseline",  # Adds Ruff, ty, uv, pytest
]
plugins = [
    "pants-python-baseline==0.1.0",
]
```
