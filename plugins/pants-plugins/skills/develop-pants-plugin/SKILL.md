---
name: develop-pants-plugin
description: Develop and modify existing Pants plugins. Use when adding targets, rules, goals to existing plugins, debugging plugin issues, writing tests, or upgrading plugins for new Pants versions.
user-invocable: true
allowed-tools: Write, Edit, Read, Bash, Glob, Grep
---

# Pants Plugin Development Assistant

Help developers work with existing Pants plugins - add features, debug issues, write tests, and upgrade for compatibility.

## When to Use This Skill

- Adding new targets, fields, rules, or goals to an existing plugin
- Debugging plugin issues (rules not firing, caching problems, type errors)
- Writing or improving tests with RuleRunner
- Upgrading a plugin for a new Pants version
- Understanding how an existing plugin works
- Performance optimization and caching improvements

## Workflow

### Phase 1: Understand the Existing Plugin

First, explore the plugin structure to understand what exists:

1. **Find the entry point**: Look for `register.py` - this shows all registered rules and targets
2. **Map the components**:
   - `targets.py` - What targets and fields exist?
   - `rules.py` - What rules process those targets?
   - `goals.py` - What user commands are available?
   - `subsystem.py` - What configuration options exist?
3. **Check tests**: Look at `tests/` to understand expected behavior
4. **Review dependencies**: Check `pyproject.toml` for Pants version compatibility

### Phase 2: Common Development Tasks

#### Adding a New Field to an Existing Target

```python
# In targets.py - add the field class
class NewField(StringField):
    alias = "new_field"
    help = "Description of what this field does"
    default = "default_value"

# Update the target's core_fields
class ExistingTarget(Target):
    core_fields = (
        *COMMON_TARGET_FIELDS,
        ExistingField,
        NewField,  # Add here
    )
```

#### Adding a New Target Type

```python
# In targets.py
class NewTarget(Target):
    alias = "new_target"
    help = "What this target represents"
    core_fields = (
        *COMMON_TARGET_FIELDS,
        SourcesField,
        # Add your fields
    )

# In register.py - add to target_types()
def target_types():
    return [ExistingTarget, NewTarget]
```

#### Adding a New Rule

```python
# In rules.py
from dataclasses import dataclass

@dataclass(frozen=True)
class NewOutput:
    """Always use frozen dataclass for rule outputs."""
    result: str
    exit_code: int

@rule
async def new_rule(
    target: WrappedTarget,
    subsystem: PluginSubsystem,
) -> NewOutput:
    """
    Rules must:
    - Be async
    - Have complete type hints
    - Return frozen dataclass
    - Have no side effects
    """
    # Implementation
    return NewOutput(result="done", exit_code=0)

# Make sure collect_rules() picks it up
def rules():
    return collect_rules()
```

#### Adding a New Goal

```python
# In goals.py
class NewGoalSubsystem(GoalSubsystem):
    name = "new-goal"
    help = "What this goal does"

class NewGoal(Goal):
    subsystem_cls = NewGoalSubsystem

@goal_rule
async def run_new_goal(
    console: Console,
    targets: Targets,
) -> NewGoal:
    # Implementation
    console.print_stdout("Processing...")
    return NewGoal(exit_code=0)
```

#### Adding Configuration Options

```python
# In subsystem.py
class PluginSubsystem(Subsystem):
    options_scope = "my-plugin"

    # Add new options
    new_option = StrOption(
        default="value",
        help="What this option controls",
    )

    new_list_option = StrListOption(
        help="List of values",
    )

    new_bool = BoolOption(
        default=False,
        help="Enable/disable something",
    )
```

### Phase 3: Debugging Common Issues

#### Rule Not Being Called

1. **Check registration**: Is the rule in `collect_rules()`?
2. **Check types**: Do input/output types match what's expected?
3. **Check target filtering**: Is the target type correct?

```python
# Debug: Add logging
import logging
logger = logging.getLogger(__name__)

@rule
async def my_rule(target: WrappedTarget) -> Output:
    logger.debug(f"Processing target: {target.target.address}")
    # ...
```

Run with debug logging:
```bash
pants --debug my-goal ::
```

#### Caching Issues

Rules must be deterministic. Check for:
- Timestamps or random values in outputs
- Reading files outside the sandbox
- Environment variables not declared

```python
# BAD: Non-deterministic
@rule
async def bad_rule() -> Output:
    return Output(timestamp=time.time())  # Changes every run!

# GOOD: Deterministic
@rule
async def good_rule() -> Output:
    return Output(value="constant")
```

#### Type Errors at Startup

Pants validates the rule graph at startup. Common issues:

```python
# ERROR: Missing type hint
@rule
async def bad(target):  # Missing type hints!
    pass

# ERROR: Wrong return type
@rule
async def bad(target: Target) -> str:  # Must return dataclass
    return "result"

# CORRECT
@rule
async def good(target: WrappedTarget) -> Output:
    return Output(...)
```

#### Process Failures

```python
@rule
async def run_tool(request: ToolRequest) -> ToolResult:
    process = Process(
        argv=["tool", "--arg"],
        input_digest=request.digest,
        description="Running tool",
    )
    result = await Get(ProcessResult, Process, process)

    # Check for failures
    if result.exit_code != 0:
        raise ProcessExecutionFailure(
            result.exit_code,
            result.stdout.decode(),
            result.stderr.decode(),
        )

    return ToolResult(output=result.stdout.decode())
```

### Phase 4: Writing Tests

#### Basic Rule Test

```python
from pants.testutil.rule_runner import RuleRunner, QueryRule

@pytest.fixture
def rule_runner():
    return RuleRunner(
        rules=[
            *my_plugin_rules(),
            QueryRule(Output, [Input]),
        ],
        target_types=[MyTarget],
    )

def test_my_rule(rule_runner):
    rule_runner.write_files({
        "BUILD": 'my_target(name="test", sources=["*.txt"])',
        "file.txt": "content",
    })

    result = rule_runner.request(Output, [Input(...)])
    assert result.exit_code == 0
```

#### Testing Goals

```python
def test_goal(rule_runner):
    rule_runner.write_files({
        "BUILD": 'my_target(name="test")',
    })

    result = rule_runner.run_goal(MyGoal, [":test"])
    assert result.exit_code == 0
    assert "expected output" in result.stdout
```

#### Testing with Subsystem Options

```python
def test_with_options(rule_runner):
    rule_runner.set_options(["--my-plugin-option=value"])
    rule_runner.write_files({"BUILD": "..."})
    # Run test
```

### Phase 5: Upgrading for New Pants Versions

#### Check the Plugin Upgrade Guide

```bash
# Check current Pants version compatibility
grep -r "pants" pyproject.toml
```

#### Handle API Changes

```python
from pants.version import PANTS_SEMVER
from packaging.version import Version

if PANTS_SEMVER >= Version("2.18.0"):
    from pants.engine.new_module import NewClass
else:
    from pants.engine.old_module import OldClass as NewClass
```

#### Common Breaking Changes

- **Import path changes**: Module reorganization
- **Type signature changes**: New required parameters
- **Deprecated removals**: Check deprecation warnings
- **New required fields**: FieldSet or Target changes

### Phase 6: Performance Optimization

#### Batch Processing

```python
# BAD: N+1 rule calls
@rule
async def process_one(target: Target) -> Result:
    # Called once per target

# GOOD: Batch processing
@rule
async def process_batch(targets: Targets) -> BatchResult:
    results = await MultiGet(
        Get(SingleResult, Target, t) for t in targets
    )
    return BatchResult(results)
```

#### Minimize Process Calls

```python
# BAD: Multiple process calls
for file in files:
    await Get(ProcessResult, Process, Process(argv=["tool", file]))

# GOOD: Single process with multiple files
await Get(ProcessResult, Process, Process(argv=["tool", *files]))
```

## Reference Documentation

For detailed API information, see:

- `reference/debugging.md` - Comprehensive debugging guide
- `reference/testing.md` - Testing patterns and fixtures
- `reference/upgrading.md` - Version migration guide
- `reference/performance.md` - Optimization techniques

## Quick Commands

```bash
# Run specific tests
hatch run pytest tests/unit/test_specific.py -v

# Debug rule execution
pants --debug my-goal ::

# View target metadata
pants peek src:target-name

# Check what rules exist
pants backends | grep my_plugin

# Validate plugin loads
pants help my-goal
```
