# Complete Testing Reference

## RuleRunner Overview

RuleRunner creates an isolated Pants environment for testing rules, targets, and goals.

## Basic Setup

### Fixture

```python
import pytest
from pants.testutil.rule_runner import RuleRunner, QueryRule

from my_plugin.register import rules, target_types
from my_plugin.rules import MyOutput, MyInput

@pytest.fixture
def rule_runner() -> RuleRunner:
    return RuleRunner(
        rules=[
            *rules(),
            QueryRule(MyOutput, [MyInput]),
        ],
        target_types=target_types(),
    )
```

### QueryRule Explained

QueryRule tells RuleRunner what requests you'll make:

```python
# You plan to request: rule_runner.request(OutputType, [InputType(...)])
QueryRule(OutputType, [InputType])

# Multiple inputs
QueryRule(OutputType, [InputA, InputB])
```

## Writing Files

```python
def test_example(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="test", sources=["*.py"])',
        "main.py": "print('hello')",
        "lib.py": "def helper(): pass",
        "subdir/BUILD": 'my_target(name="sub")',
        "subdir/module.py": "# code here",
    })
```

## Requesting Rules

### Basic Request

```python
def test_my_rule(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({...})

    result = rule_runner.request(
        MyOutput,
        [MyInput(value="test")]
    )

    assert result.success is True
```

### Getting Targets

```python
from pants.engine.addresses import Address

def test_with_target(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "src/BUILD": 'my_target(name="lib", sources=["*.py"])',
        "src/main.py": "# code",
    })

    # Get a single target
    target = rule_runner.get_target(Address("src", target_name="lib"))

    assert target.alias == "my_target"
    assert target[SourcesField].value == ("*.py",)
```

## Setting Options

```python
def test_with_options(rule_runner: RuleRunner) -> None:
    rule_runner.set_options([
        "--my-tool-enabled=true",
        "--my-tool-config=config.json",
        "--my-tool-args=['--verbose', '--strict']",
    ])

    # Options affect subsequent requests
    result = rule_runner.request(...)
```

### Environment Variables

```python
def test_with_env(rule_runner: RuleRunner) -> None:
    rule_runner.set_options(
        [],
        env={"MY_VAR": "value", "OTHER_VAR": "other"},
    )
```

### Inherited Options

```python
def test_inheriting_options(rule_runner: RuleRunner) -> None:
    rule_runner.set_options(
        ["--my-tool-skip=false"],
        env_inherit={"PATH", "HOME"},  # Inherit from real env
    )
```

## Testing Goals

### Run Goal Rule

```python
def test_my_goal(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "src/BUILD": 'my_target(name="test")',
    })

    result = rule_runner.run_goal_rule(
        MyGoal,
        args=["src:test"],
    )

    assert result.exit_code == 0
    assert "expected output" in result.stdout
    assert result.stderr == ""
```

### With Global Args

```python
def test_goal_with_args(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({...})

    result = rule_runner.run_goal_rule(
        MyGoal,
        global_args=["--my-tool-verbose"],
        args=["src::"],
    )
```

## Testing Target Types

### Verify Field Values

```python
def test_target_fields(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": """
my_target(
    name="test",
    sources=["*.py"],
    timeout=30,
    custom_field="value",
)
        """,
    })

    target = rule_runner.get_target(Address("", target_name="test"))

    assert target[SourcesField].value == ("*.py",)
    assert target[TimeoutField].value == 30
    assert target[CustomField].value == "value"
```

### Verify Defaults

```python
def test_field_defaults(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="minimal")',
    })

    target = rule_runner.get_target(Address("", target_name="minimal"))

    assert target[TimeoutField].value == 60  # Default
    assert target[EnabledField].value is True  # Default
```

### Verify Required Fields

```python
def test_required_field_error(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="test")',  # Missing required field
    })

    with pytest.raises(Exception, match="required"):
        rule_runner.get_target(Address("", target_name="test"))
```

## Testing Process Execution

### Test Tool Runs

```python
def test_external_tool(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'lint_target(name="test", sources=["*.py"])',
        "bad_code.py": "x=1",  # Style violation
    })

    result = rule_runner.run_goal_rule(LintGoal, args=[":test"])

    assert result.exit_code == 1  # Lint failure
    assert "style" in result.stdout.lower()
```

### Test Error Handling

```python
def test_process_failure_handling(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="test", sources=["invalid.py"])',
        "invalid.py": "this is not valid python!!!",
    })

    result = rule_runner.run_goal_rule(MyGoal, args=[":test"])

    assert result.exit_code != 0
    assert "error" in result.stderr.lower()
```

## Testing Dependencies

### Transitive Dependencies

```python
def test_transitive_deps(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "lib/BUILD": 'my_target(name="lib")',
        "app/BUILD": """
my_target(
    name="app",
    dependencies=["lib:lib"],
)
        """,
    })

    result = rule_runner.run_goal_rule(MyGoal, args=["app:app"])

    assert "lib" in result.stdout
    assert "app" in result.stdout
```

## Test Utilities

### Create Subsystem

```python
from pants.testutil.option_util import create_subsystem

def test_subsystem_logic() -> None:
    subsystem = create_subsystem(
        MySubsystem,
        config="custom.conf",
        timeout=120,
    )

    assert subsystem.config == "custom.conf"
    assert subsystem.timeout == 120
```

### Mock Console

```python
from pants.testutil.option_util import mock_console

def test_with_mock_console() -> None:
    with mock_console() as (console, stdio_reader):
        console.print_stdout("Hello")
        console.print_stderr("Error")

    assert stdio_reader.stdout == "Hello\n"
    assert stdio_reader.stderr == "Error\n"
```

## Test Organization

### Directory Structure

```
tests/
├── conftest.py          # Shared fixtures
├── unit/
│   ├── __init__.py
│   ├── test_targets.py
│   ├── test_fields.py
│   └── test_rules.py
└── integration/
    ├── __init__.py
    ├── test_goals.py
    └── test_end_to_end.py
```

### Shared conftest.py

```python
"""Shared test fixtures."""
import pytest
from pants.testutil.rule_runner import RuleRunner

from my_plugin.register import rules, target_types

@pytest.fixture
def rule_runner() -> RuleRunner:
    return RuleRunner(
        rules=rules(),
        target_types=target_types(),
    )

@pytest.fixture
def rule_runner_with_defaults(rule_runner: RuleRunner) -> RuleRunner:
    rule_runner.set_options([
        "--my-tool-enabled=true",
    ])
    return rule_runner
```

### Helper Functions

```python
def create_project(
    rule_runner: RuleRunner,
    name: str,
    sources: dict[str, str] | None = None,
) -> Address:
    """Create a test project with standard structure."""
    files = {
        f"{name}/BUILD": f'my_target(name="{name}", sources=["*.py"])',
    }
    if sources:
        for filename, content in sources.items():
            files[f"{name}/{filename}"] = content
    else:
        files[f"{name}/main.py"] = "# default"

    rule_runner.write_files(files)
    return Address(name, target_name=name)
```

## Running Tests

```bash
# All tests
hatch run test

# Specific file
hatch run pytest tests/unit/test_rules.py -v

# Specific test
hatch run pytest tests/unit/test_rules.py::test_my_rule -v

# With coverage
hatch run test-cov

# Only unit tests
hatch run pytest tests/unit/ -v

# Only integration tests
hatch run pytest tests/integration/ -v

# Show print statements
hatch run pytest -s tests/

# Stop on first failure
hatch run pytest -x tests/
```

## Debugging Tests

### Print Debug Info

```python
def test_debug(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({...})

    # Print all targets
    all_targets = rule_runner.request(AllTargets, [])
    for t in all_targets:
        print(f"Target: {t.address}, type: {type(t).__name__}")

    # Print file contents
    snapshot = rule_runner.request(Snapshot, [PathGlobs(["**/*"])])
    print(f"Files: {snapshot.files}")
```

### Verbose RuleRunner

```python
@pytest.fixture
def rule_runner() -> RuleRunner:
    return RuleRunner(
        rules=rules(),
        target_types=target_types(),
        # Enable debug logging
        extra_opts=["--level=debug"],
    )
```
