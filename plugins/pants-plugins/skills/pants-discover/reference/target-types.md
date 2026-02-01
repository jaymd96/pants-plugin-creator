# Target Types Reference

Common target types available in Pants.

## Python Targets

### python_source
A single Python source file.

```python
python_source(
    name="app",
    source="app.py",
    dependencies=["src/lib:util"],
)
```

### python_sources
Multiple Python source files.

```python
python_sources(
    name="lib",
    sources=["**/*.py"],
    dependencies=[],
)
```

### python_test
A single Python test file.

```python
python_test(
    name="test_app",
    source="test_app.py",
    dependencies=["src:app"],
)
```

### python_tests
Multiple Python test files.

```python
python_tests(
    name="tests",
    sources=["test_*.py"],
    dependencies=["src:lib"],
)
```

### python_requirement
A third-party Python dependency.

```python
python_requirement(
    name="requests",
    requirements=["requests>=2.28.0"],
)
```

### python_requirements
Multiple requirements from a file.

```python
python_requirements(
    name="reqs",
    source="requirements.txt",
)
```

### python_distribution
A distributable Python package.

```python
python_distribution(
    name="mypackage",
    dependencies=["src:lib"],
    provides=python_artifact(
        name="mypackage",
        version="1.0.0",
    ),
)
```

### pex_binary
A PEX executable.

```python
pex_binary(
    name="app",
    entry_point="app.py",
    dependencies=["src:lib"],
)
```

## Python Baseline Targets

### baseline_python_project
A Python project with quality baseline checks.

```python
baseline_python_project(
    name="project",
    sources=["src/**/*.py"],
    test_sources=["tests/**/*.py"],
    python_version="3.13",
    line_length=120,
    strict=True,
    coverage_threshold=80,
)
```

## Docker Targets

### docker_image
A Docker image.

```python
docker_image(
    name="app",
    source="Dockerfile",
    dependencies=["src:app"],
    repository="myrepo/myapp",
)
```

## Shell Targets

### shell_source
A shell script.

```python
shell_source(
    name="deploy",
    source="deploy.sh",
)
```

### shell_sources
Multiple shell scripts.

```python
shell_sources(
    name="scripts",
    sources=["*.sh"],
)
```

### shunit2_test
A shell test using shunit2.

```python
shunit2_test(
    name="test_deploy",
    source="test_deploy.sh",
    dependencies=[":deploy"],
)
```

## Resource Targets

### resource
A single resource file.

```python
resource(
    name="config",
    source="config.yaml",
)
```

### resources
Multiple resource files.

```python
resources(
    name="data",
    sources=["**/*.json", "**/*.yaml"],
)
```

### file
A single file (any type).

```python
file(
    name="readme",
    source="README.md",
)
```

### files
Multiple files.

```python
files(
    name="docs",
    sources=["*.md", "*.txt"],
)
```

## Common Fields

Most targets support these fields:

| Field | Description |
|-------|-------------|
| `name` | Target name (required for explicit targets) |
| `source` / `sources` | Source file(s) |
| `dependencies` | Explicit dependencies |
| `tags` | Tags for filtering |
| `description` | Documentation |

## Address Syntax

Targets are referenced by address:

```
path/to/dir:target_name     # Explicit target
path/to/dir                  # Default target in dir
path/to/dir:                 # Same as above
path/to/dir::                # All targets recursively
path/to/file.py              # File target
```

Examples:
```bash
pants list src/python::      # All targets under src/python
pants test tests:unit        # Specific target
pants run src/app.py         # File target
```

## Auto-Generated Targets

Pants can auto-generate BUILD files with `tailor`:

```bash
pants tailor ::
```

This creates targets for discovered files based on conventions.
