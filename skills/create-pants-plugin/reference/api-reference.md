# Pants Plugin API Reference

Complete API documentation for Pants plugin development.

## Target API

### What is a Target?

A Target is addressable metadata describing code:

```python
# In BUILD file
python_sources(
    name="utils",
    sources=["*.py"],
    dependencies=[":models"],
)
# Address: Address("src/util", target_name="utils")
```

### Defining Fields

```python
from pants.engine.target import (
    StringField,
    IntField,
    BoolField,
    StringSequenceField,
    DictField,
)

class SourceField(StringField):
    alias = "source"
    help = "Path to a source file"

class TimeoutField(IntField):
    alias = "timeout"
    default = 60

class SkipField(BoolField):
    alias = "skip"
    default = False

class DependenciesField(StringSequenceField):
    alias = "dependencies"
    default = ()

class MetadataField(DictField):
    alias = "metadata"
```

### Defining Target Types

```python
from pants.engine.target import Target, COMMON_TARGET_FIELDS

class CustomTarget(Target):
    alias = "custom_target"
    help = "A custom target type"

    core_fields = (
        *COMMON_TARGET_FIELDS,  # name, tags, description
        SourceField,
        TimeoutField,
        MetadataField,
    )
```

### Using Targets in Rules

```python
from pants.engine.target import WrappedTarget

@rule
async def process_target(wrapped: WrappedTarget) -> Output:
    target = wrapped.target

    # Check if target has a field
    if target.has_field(TimeoutField):
        timeout = target[TimeoutField].value

    # Get field with default
    metadata = target.get(MetadataField) or MetadataField()

    return Output(...)
```

### Key Field Types

| Type | Description | Example |
|------|-------------|---------|
| `StringField` | Single string | `"value"` |
| `StringSequenceField` | List of strings | `["a", "b"]` |
| `IntField` | Integer | `60` |
| `BoolField` | Boolean | `True` |
| `DictField` | Dictionary | `{"key": "value"}` |
| `Dependencies` | Target dependencies | `[":other"]` |

## Rules API

### Rule Basics

```python
from pants.engine.rules import rule

@rule
async def my_rule(input1: Type1, input2: Type2) -> OutputType:
    """
    Constraints:
    - Must be async
    - Type hints required
    - NO side effects
    - Return single output type
    """
    result = await process(input1, input2)
    return OutputType(result)
```

### Regular Rules: @rule

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class CompiledCode:
    code: str
    exit_code: int

@rule
async def compile_code(sources: SourceFiles) -> CompiledCode:
    return CompiledCode(code="...", exit_code=0)
```

### Goal Rules: @goal_rule

```python
from pants.engine.goal import Goal, GoalSubsystem, goal_rule
from pants.engine.console import Console

class MyGoalSubsystem(GoalSubsystem):
    name = "my-goal"
    help = "Run my goal"

class MyGoal(Goal):
    subsystem_cls = MyGoalSubsystem

@goal_rule
async def run_my_goal(console: Console, targets: Targets) -> MyGoal:
    for target in targets:
        console.print_stdout(f"Processing {target.name}")
    return MyGoal(exit_code=0)
```

### Using Get() for Dependencies

```python
from pants.engine.rules import Get

@rule
async def process_target(target: Target) -> OutputType:
    # Call another rule
    compiled = await Get(CompiledCode, SourceFiles, sources)

    # Run a subprocess
    result = await Get(ProcessResult, Process, process)

    return OutputType(...)
```

### Subsystems: Configuration

```python
from pants.option.subsystem import Subsystem
from pants.option.option_types import StrOption, BoolOption, IntOption

class MyToolSubsystem(Subsystem):
    options_scope = "my-tool"

    version = StrOption(default="1.0", help="Tool version")
    enabled = BoolOption(default=True, help="Enable tool")
    timeout = IntOption(default=30, help="Timeout seconds")

@rule
async def process_with_config(subsystem: MyToolSubsystem) -> Output:
    if subsystem.enabled:
        # Use subsystem.version, subsystem.timeout
        pass
    return Output(...)
```

### Process: Running Subprocesses

```python
from pants.engine.process import Process, ProcessResult
from pants.engine.fs import Digest, CreateDigest, FileContent

# Create input files
input_digest = await Get(
    Digest,
    CreateDigest([FileContent("input.txt", b"content")]),
)

# Define process
process = Process(
    argv=["my-tool", "--option", "input.txt"],
    input_digest=input_digest,
    description="Run my tool",
    env={"KEY": "value"},
    timeout_seconds=30,
)

# Execute
result = await Get(ProcessResult, Process, process)

# Access results
if result.exit_code == 0:
    output = result.stdout.decode("utf-8")
    files = result.output_digest
else:
    errors = result.stderr.decode("utf-8")
```

### FieldSets: Filtering Targets

```python
from pants.engine.target import FieldSet

class MyToolFieldSet(FieldSet):
    required_fields = (SourcesField, TimeoutField)

    sources: SourcesField
    timeout: TimeoutField

@rule
async def process_fieldsets(fieldsets: FieldSets) -> Output:
    for fieldset in fieldsets:
        sources = fieldset.sources.value
    return Output(...)
```

## File I/O Patterns

### Reading Files

```python
from pants.engine.fs import Digest, PathGlobs

@rule
async def read_files(target: Target) -> Output:
    sources_field = target[SourcesField]
    source_digest = await Get(Digest, PathGlobs(sources_field.value))
    return Output(...)
```

### Writing Files

```python
from pants.engine.fs import CreateDigest, FileContent

@rule
async def generate_files() -> Digest:
    digest = await Get(
        Digest,
        CreateDigest([
            FileContent("output.txt", b"content"),
            FileContent("dir/file.py", b"code"),
        ]),
    )
    return digest
```

## Testing Patterns

### Testing a Rule

```python
from pants.testutil.rule_runner import RuleRunner, QueryRule

@pytest.fixture
def rule_runner():
    return RuleRunner(
        rules=[*my_rules(), QueryRule(Output, [Input])],
        target_types=[MyTarget],
    )

def test_my_rule(rule_runner):
    rule_runner.write_files({
        "BUILD": 'my_target(name="test", sources=["*.txt"])',
        "file.txt": "content",
    })

    input_obj = Input(...)
    result = rule_runner.request(Output, [input_obj])
    assert result.field == expected
```

### Testing a Goal

```python
def test_my_goal(rule_runner):
    rule_runner.write_files({"BUILD": "my_target(name='test')"})

    result = rule_runner.run_goal(MyGoal, [":test"])
    assert result.exit_code == 0
```

## Key Imports

```python
# Rules
from pants.engine.rules import rule, goal_rule, collect_rules, Get

# Targets
from pants.engine.target import (
    Target, Field, FieldSet, Targets, WrappedTarget,
    StringField, IntField, BoolField, StringSequenceField,
    COMMON_TARGET_FIELDS,
)

# Goals
from pants.engine.goal import Goal, GoalSubsystem

# Subsystems
from pants.option.subsystem import Subsystem
from pants.option.option_types import StrOption, BoolOption, IntOption

# Process
from pants.engine.process import Process, ProcessResult

# File system
from pants.engine.fs import Digest, PathGlobs, CreateDigest, FileContent

# Console
from pants.engine.console import Console

# Addresses
from pants.engine.addresses import Address, Addresses

# Testing
from pants.testutil.rule_runner import RuleRunner, QueryRule
```

## Debugging Tips

### View Rule Graph
```bash
pants peek --graph-representation=dot src/my_plugin
```

### Debug Logging
```bash
pants my-goal --debug-log-level=debug 2>&1 | grep -i "my_plugin"
```

### Inspect Targets
```bash
pants peek src::
pants peek src:specific-target
```

## Version Compatibility

Handle API changes:

```python
from pants.version import PANTS_SEMVER
from packaging.version import Version

if PANTS_SEMVER < Version("2.15.0"):
    from old_api import rules
else:
    from new_api import rules
```

## Best Practices

1. **Keep rules pure** - No I/O, no side effects
2. **Use frozen dataclasses** - Enables caching
3. **Document everything** - Rules, fields, subsystems
4. **Test thoroughly** - Use RuleRunner
5. **Type hint everything** - Required by engine
6. **Use subsystems** - Not global state
7. **Handle errors gracefully** - Good error messages
8. **Make outputs deterministic** - Same inputs = same outputs
