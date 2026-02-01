# Testing Pants Plugins

Comprehensive guide to testing Pants plugins with RuleRunner.

## RuleRunner Basics

RuleRunner creates an isolated Pants environment for testing.

### Setup

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
            # QueryRule tells RuleRunner what you want to request
            QueryRule(MyOutput, [MyInput]),
        ],
        target_types=target_types(),
    )
```

### Basic Test

```python
def test_my_rule(rule_runner: RuleRunner) -> None:
    # Create test files
    rule_runner.write_files({
        "BUILD": """
my_target(
    name="test",
    sources=["*.txt"],
)
        """,
        "file.txt": "test content",
    })

    # Request the rule output
    input_obj = MyInput(value="test")
    result = rule_runner.request(MyOutput, [input_obj])

    # Assert
    assert result.exit_code == 0
    assert "expected" in result.output
```

## Testing Different Components

### Testing Rules

```python
from pants.engine.target import WrappedTarget

def test_process_target_rule(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "src/BUILD": 'my_target(name="example", sources=["*.py"])',
        "src/main.py": "print('hello')",
    })

    # Get the target
    target = rule_runner.get_target(Address("src", target_name="example"))
    wrapped = WrappedTarget(target)

    # Request the rule
    result = rule_runner.request(ProcessedOutput, [wrapped])

    assert result.target_name == "example"
    assert result.exit_code == 0
```

### Testing Goals

```python
def test_my_goal(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "src/BUILD": 'my_target(name="test")',
    })

    # Run the goal
    result = rule_runner.run_goal(MyGoal, ["src:test"])

    assert result.exit_code == 0
    assert "Processing" in result.stdout
```

### Testing with Multiple Targets

```python
def test_multiple_targets(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "a/BUILD": 'my_target(name="a")',
        "b/BUILD": 'my_target(name="b")',
        "c/BUILD": 'other_target(name="c")',  # Different type
    })

    # Run on all targets
    result = rule_runner.run_goal(MyGoal, ["::"])

    # Should process only my_target types
    assert "a" in result.stdout
    assert "b" in result.stdout
    # c should be skipped (different type)
```

### Testing Subsystem Options

```python
def test_with_options(rule_runner: RuleRunner) -> None:
    # Set subsystem options
    rule_runner.set_options([
        "--my-plugin-enabled=true",
        "--my-plugin-timeout=60",
        "--my-plugin-extra-args=['--verbose', '--strict']",
    ])

    rule_runner.write_files({
        "BUILD": 'my_target(name="test")',
    })

    result = rule_runner.run_goal(MyGoal, [":test"])
    assert result.exit_code == 0
```

### Testing with Environment Variables

```python
def test_with_env_vars(rule_runner: RuleRunner) -> None:
    rule_runner.set_options([], env={"MY_VAR": "value"})

    rule_runner.write_files({
        "BUILD": 'my_target(name="test")',
    })

    result = rule_runner.run_goal(MyGoal, [":test"])
    assert result.exit_code == 0
```

## Testing Target Types

### Test Target Creation

```python
from pants.engine.addresses import Address

def test_target_fields(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": """
my_target(
    name="test",
    sources=["*.txt"],
    custom_field="custom_value",
    timeout=30,
)
        """,
    })

    target = rule_runner.get_target(Address("", target_name="test"))

    # Check target type
    assert isinstance(target, MyTarget)

    # Check field values
    assert target[SourcesField].value == ("*.txt",)
    assert target[CustomField].value == "custom_value"
    assert target[TimeoutField].value == 30
```

### Test Field Defaults

```python
def test_field_defaults(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="test")',  # Minimal, use defaults
    })

    target = rule_runner.get_target(Address("", target_name="test"))

    # Check defaults applied
    assert target[TimeoutField].value == 60  # Default
    assert target[EnabledField].value is True  # Default
```

### Test Required Fields

```python
def test_required_field_missing(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="test")',  # Missing required field
    })

    with pytest.raises(Exception, match="required"):
        rule_runner.get_target(Address("", target_name="test"))
```

## Testing Process Execution

### Test External Tool

```python
def test_runs_external_tool(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'linter_target(name="test", sources=["*.py"])',
        "code.py": "x=1",  # Code to lint
    })

    result = rule_runner.run_goal(LintGoal, [":test"])

    # Tool should have run
    assert result.exit_code == 0
    # Or check for expected output
    assert "linting" in result.stdout.lower()
```

### Test Process Failure Handling

```python
def test_process_failure(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="test", sources=["bad.py"])',
        "bad.py": "this is not valid python!!!",
    })

    result = rule_runner.run_goal(MyGoal, [":test"])

    # Should fail gracefully
    assert result.exit_code != 0
    assert "error" in result.stderr.lower()
```

## Testing Dependencies

### Test Transitive Dependencies

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

    result = rule_runner.run_goal(MyGoal, ["app:app"])

    # Should process both app and its dependency
    assert "lib" in result.stdout
    assert "app" in result.stdout
```

## Fixtures and Utilities

### Common Fixtures

```python
@pytest.fixture
def rule_runner() -> RuleRunner:
    """Standard rule runner for most tests."""
    return RuleRunner(
        rules=[*rules(), QueryRule(Output, [Input])],
        target_types=target_types(),
    )


@pytest.fixture
def rule_runner_with_options() -> RuleRunner:
    """Rule runner with common options set."""
    runner = RuleRunner(
        rules=[*rules(), QueryRule(Output, [Input])],
        target_types=target_types(),
    )
    runner.set_options([
        "--my-plugin-enabled=true",
        "--my-plugin-verbose=true",
    ])
    return runner
```

### Helper Functions

```python
def create_test_project(rule_runner: RuleRunner, name: str) -> Address:
    """Create a standard test project structure."""
    rule_runner.write_files({
        f"{name}/BUILD": f'my_target(name="{name}", sources=["*.py"])',
        f"{name}/main.py": "print('hello')",
        f"{name}/util.py": "def helper(): pass",
    })
    return Address(name, target_name=name)


def test_with_helper(rule_runner: RuleRunner) -> None:
    addr = create_test_project(rule_runner, "myproject")
    result = rule_runner.run_goal(MyGoal, [str(addr)])
    assert result.exit_code == 0
```

## Test Organization

### Directory Structure

```
tests/
├── conftest.py          # Shared fixtures
├── unit/
│   ├── __init__.py
│   ├── test_targets.py  # Target type tests
│   ├── test_rules.py    # Rule tests
│   └── test_subsystem.py
└── integration/
    ├── __init__.py
    └── test_goal.py     # End-to-end goal tests
```

### conftest.py

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
```

## Running Tests

```bash
# Run all tests
hatch run test

# Run specific test file
hatch run pytest tests/unit/test_rules.py -v

# Run specific test
hatch run pytest tests/unit/test_rules.py::test_my_rule -v

# Run with coverage
hatch run test-cov

# Run only unit tests
hatch run pytest tests/unit/ -v

# Run only integration tests
hatch run pytest tests/integration/ -v
```
