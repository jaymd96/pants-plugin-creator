# Pants Plugin File Templates

Complete file templates for Pants plugin development.

## pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "{plugin-name}"
version = "0.1.0"
description = "A custom Pants plugin for {purpose}"
readme = "README.md"
license = {text = "Apache-2.0"}
requires-python = ">=3.12,<4"
authors = [
    {name = "{author}", email = "{email}"},
]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Environment :: Plugins",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: Apache Software License",
    "Programming Language :: Python :: 3.12",
    "Programming Language :: Python :: 3.13",
]

[project.urls]
Homepage = "https://github.com/{org}/{plugin-name}"
Repository = "https://github.com/{org}/{plugin-name}.git"

[tool.hatch.version]
path = "src/{package}/version.py"
pattern = '__version__ = "(?P<version>[^"]+)"'

[tool.hatch.build.targets.wheel]
packages = ["src/{package}"]

[tool.hatch.envs.default]
dependencies = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "mypy>=1.0",
]

[tool.hatch.envs.default.scripts]
test = "pytest {args:tests}"
test-cov = "pytest --cov=src/{package} --cov-report=term-missing tests"
lint = [
    "black --check src tests",
    "isort --check-only src tests",
    "mypy src",
]
fmt = [
    "black src tests",
    "isort src tests",
]
all = ["fmt", "lint", "test-cov"]

# Publishing configuration
[tool.hatch.publish.index]
disable = false

[tool.hatch.publish.index.repos.main]
url = "https://upload.pypi.org/legacy/"

[tool.hatch.publish.index.repos.test]
url = "https://test.pypi.org/legacy/"

[tool.black]
line-length = 100
target-version = ["py312", "py313"]

[tool.isort]
profile = "black"
line_length = 100

[tool.mypy]
python_version = "3.13"
ignore_missing_imports = true

[tool.pytest.ini_options]
pythonpath = ["src"]
testpaths = ["tests"]
```

## src/{package}/version.py

```python
"""Version information for {package}."""

__version__ = "0.1.0"
__author__ = "{author} <{email}>"


def get_version() -> str:
    """Return the plugin version string."""
    return f"{plugin-name}/{__version__}"
```

## src/{package}/__init__.py

```python
"""A custom Pants plugin for {purpose}.

This plugin provides:
- Custom target types for {domain}
- Rules for {functionality}
- Goals for {commands}
"""

from {package}.version import __version__

__all__ = ["__version__"]
```

## src/{package}/register.py

```python
"""Register the plugin with Pants.

This file must be named register.py and define rules() and target_types().
Pants loads this module when the plugin is in backend_packages.
"""

from typing import Sequence

from pants.engine.rules import collect_rules

from {package} import rules as plugin_rules
from {package}.subsystem import PluginSubsystem
from {package}.targets import CustomTarget


def rules() -> Sequence:
    """Return all rules and subsystems provided by this plugin."""
    return [
        *collect_rules(plugin_rules),
        *PluginSubsystem.rules(),
    ]


def target_types() -> Sequence:
    """Return all custom target types provided by this plugin."""
    return [CustomTarget]
```

## src/{package}/subsystem.py

```python
"""Configuration options for the plugin."""

from pants.option.subsystem import Subsystem
from pants.option.option_types import BoolOption, IntOption, StrOption


class PluginSubsystem(Subsystem):
    """Configuration for {plugin-name}."""

    options_scope = "{scope}"
    help = "Options for {plugin-name}"

    enabled = BoolOption(
        default=True,
        help="Whether to enable this plugin.",
    )

    option_string = StrOption(
        default="default",
        help="A string configuration option.",
    )

    timeout_seconds = IntOption(
        default=30,
        help="Timeout for operations in seconds.",
    )
```

## src/{package}/targets.py

```python
"""Target types provided by the plugin."""

from pants.engine.target import (
    COMMON_TARGET_FIELDS,
    StringField,
    StringSequenceField,
    BoolField,
    Target,
)


class CustomSourcesField(StringSequenceField):
    """Source files for the custom target."""

    alias = "sources"
    help = "Source file(s) for this target."
    default = ()


class CustomProcessField(BoolField):
    """Whether to process this target."""

    alias = "process"
    help = "Whether to process this target."
    default = True


class CustomTarget(Target):
    """A custom target for {purpose}.

    Example:
        custom_target(
            name="my_target",
            sources=["*.txt"],
            process=True,
        )
    """

    alias = "custom_target"
    help = "A custom target type for {domain}"

    core_fields = (
        *COMMON_TARGET_FIELDS,
        CustomSourcesField,
        CustomProcessField,
    )
```

## src/{package}/rules.py

```python
"""Rules that implement the plugin's functionality."""

from dataclasses import dataclass
from typing import Sequence

from pants.engine.rules import collect_rules, rule, Get
from pants.engine.target import WrappedTarget
from pants.engine.process import Process, ProcessResult

from {package}.targets import CustomTarget, CustomSourcesField
from {package}.subsystem import PluginSubsystem


@dataclass(frozen=True)
class ProcessedOutput:
    """Result of processing a target."""

    target_name: str
    output: str
    exit_code: int


@rule
async def process_custom_target(
    wrapped: WrappedTarget,
    subsystem: PluginSubsystem,
) -> ProcessedOutput:
    """Process a custom target.

    Rules:
    - Must be async
    - Must have type hints
    - Must return frozen dataclass
    - Must NOT have side effects
    """
    target = wrapped.target

    if not isinstance(target, CustomTarget):
        return ProcessedOutput(target_name="", output="", exit_code=1)

    if not subsystem.enabled:
        return ProcessedOutput(
            target_name=target.name,
            output="Plugin disabled",
            exit_code=0,
        )

    sources = target[CustomSourcesField].value

    # Example: Run a process
    process = Process(
        argv=["echo", f"Processing {target.name} with {len(sources)} sources"],
        description=f"Processing {target.name}",
    )
    result = await Get(ProcessResult, Process, process)

    return ProcessedOutput(
        target_name=target.name,
        output=result.stdout.decode("utf-8"),
        exit_code=result.exit_code,
    )


def rules() -> Sequence:
    """Return all rules from this module."""
    return collect_rules()
```

## src/{package}/goals.py

```python
"""Goals provided by the plugin."""

from typing import Sequence

from pants.engine.goal import Goal, GoalSubsystem, goal_rule
from pants.engine.console import Console
from pants.engine.rules import collect_rules, Get
from pants.engine.target import Targets, WrappedTarget

from {package}.targets import CustomTarget
from {package}.rules import ProcessedOutput, process_custom_target
from {package}.subsystem import PluginSubsystem


class PluginGoalSubsystem(GoalSubsystem):
    """Subsystem for plugin goal options."""

    name = "{scope}"
    help = "Run {plugin-name} on specified targets."


class PluginGoal(Goal):
    """A goal provided by the plugin."""

    subsystem_cls = PluginGoalSubsystem


@goal_rule
async def run_plugin_goal(
    console: Console,
    targets: Targets,
    subsystem: PluginSubsystem,
) -> PluginGoal:
    """Execute the plugin goal.

    Invoked with: pants {scope} [targets]
    """
    if not subsystem.enabled:
        console.print_stdout("Plugin is disabled")
        return PluginGoal(exit_code=0)

    custom_targets = [t for t in targets if isinstance(t, CustomTarget)]

    if not custom_targets:
        console.print_stdout("No custom targets found")
        return PluginGoal(exit_code=0)

    exit_code = 0
    for target in custom_targets:
        wrapped = WrappedTarget(target)
        result = await Get(ProcessedOutput, WrappedTarget, wrapped)

        status = "OK" if result.exit_code == 0 else "FAIL"
        console.print_stdout(f"[{status}] {result.target_name}: {result.output.strip()}")

        if result.exit_code != 0:
            exit_code = result.exit_code

    return PluginGoal(exit_code=exit_code)


def rules() -> Sequence:
    """Return all rules from this module."""
    return collect_rules()
```

## tests/conftest.py

```python
"""Pytest configuration and shared fixtures."""

import pytest

from pants.testutil.rule_runner import RuleRunner


@pytest.fixture
def rule_runner() -> RuleRunner:
    """Create a RuleRunner for testing plugin rules."""
    from {package}.register import rules, target_types

    return RuleRunner(
        rules=rules(),
        target_types=target_types(),
    )
```

## tests/unit/test_subsystem.py

```python
"""Tests for plugin subsystem configuration."""

import pytest

from {package}.subsystem import PluginSubsystem


class TestPluginSubsystem:
    """Test cases for PluginSubsystem."""

    def test_default_options(self) -> None:
        """Test that subsystem has correct defaults."""
        # Note: In real tests, use RuleRunner to create subsystem
        # This is a simplified example
        assert PluginSubsystem.options_scope == "{scope}"
```

## Makefile

```makefile
.PHONY: help install test lint fmt clean

help:
	@echo "Available commands:"
	@echo "  make install    - Set up development environment"
	@echo "  make test       - Run tests"
	@echo "  make lint       - Check code quality"
	@echo "  make fmt        - Format code"
	@echo "  make clean      - Remove build artifacts"

install:
	hatch env create

test:
	hatch run test

lint:
	hatch run lint

fmt:
	hatch run fmt

clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type d -name .pytest_cache -exec rm -rf {} +
	find . -type d -name *.egg-info -exec rm -rf {} +
	rm -rf build/ dist/ .hatch/
```

## .gitignore

```
# Hatch
.hatch/

# Python
__pycache__/
*.py[cod]
*.egg-info/
dist/
build/

# Testing
.pytest_cache/
.coverage
htmlcov/

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
```

## README.md

```markdown
# {plugin-name}

A custom Pants plugin for {purpose}.

## Installation

### From PyPI

```bash
pip install {plugin-name}
```

### In pants.toml

```toml
[GLOBAL]
plugins = ["{plugin-name}==0.1.0"]
backend_packages = ["{package}"]
```

## Usage

### Define targets

```python
# BUILD
custom_target(
    name="my_target",
    sources=["*.txt"],
)
```

### Run the plugin

```bash
pants {scope} src::
```

## Configuration

```toml
# pants.toml
[{scope}]
enabled = true
timeout_seconds = 60
```

## Development

```bash
hatch env create
hatch run test
hatch run fmt
hatch run lint
```

## License

Apache License 2.0
```

## Publishing to PyPI

### Setup PyPI Credentials

Create `~/.pypirc` or use environment variables:

```bash
# Option 1: Environment variables (recommended for CI)
export HATCH_INDEX_USER=__token__
export HATCH_INDEX_AUTH=pypi-your-api-token-here

# Option 2: ~/.pypirc file
cat > ~/.pypirc << 'EOF'
[pypi]
username = __token__
password = pypi-your-api-token-here

[testpypi]
username = __token__
password = pypi-your-test-token-here
EOF
```

### Publishing Workflow

```bash
# 1. Update version in version.py
# 2. Run all checks
hatch run all

# 3. Build the package
hatch build

# 4. Test publish to TestPyPI first
hatch publish -r test

# 5. Publish to PyPI
hatch publish

# Or publish to specific repo
hatch publish -r main
```

### Version Bumping

```bash
# Show current version
hatch version

# Bump version
hatch version minor  # 0.1.0 -> 0.2.0
hatch version patch  # 0.1.0 -> 0.1.1
hatch version major  # 0.1.0 -> 1.0.0
```

---

## LICENSE

```
Apache License
Version 2.0, January 2004
http://www.apache.org/licenses/

Copyright {year} {author}

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
