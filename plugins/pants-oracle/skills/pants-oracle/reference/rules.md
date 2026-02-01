# Complete Rules API Reference

## Rule Basics

### Simple Rule

```python
from pants.engine.rules import rule

@rule
async def my_rule(input: MyInput) -> MyOutput:
    """Process input and return output."""
    return MyOutput(result=process(input))
```

### Rule with Dependencies

```python
from pants.engine.rules import rule, Get

@rule
async def my_rule(input: MyInput) -> MyOutput:
    # Request another rule's output
    other = await Get(OtherOutput, OtherInput, input.to_other())
    return MyOutput(result=other.value)
```

### Parallel Requests

```python
from pants.engine.rules import rule, Get, MultiGet

@rule
async def my_rule(inputs: MyInputs) -> MyOutput:
    # Execute all in parallel
    results = await MultiGet(
        Get(ItemOutput, ItemInput, item)
        for item in inputs.items
    )
    return MyOutput(results=tuple(results))
```

## Rule Registration

### Using collect_rules

```python
# my_rules.py
from pants.engine.rules import collect_rules, rule

@rule
async def rule_a(...) -> OutputA:
    ...

@rule
async def rule_b(...) -> OutputB:
    ...

def rules():
    return collect_rules()
```

### In register.py

```python
# register.py
from my_plugin import rules as my_rules

def rules():
    return [
        *my_rules.rules(),
        # Additional rules
    ]
```

## Get Requests

### Basic Get

```python
# Get(OutputType, InputType, input_value)
result = await Get(CompiledCode, SourceFile, source)
```

### Get with Implicit Parameters

```python
# When input type has __defaults__ rule
result = await Get(Output, Input(...))  # Simplified form
```

### MultiGet

```python
# Parallel execution
results = await MultiGet(
    Get(Output, Input, item) for item in items
)

# With mixed types
a, b, c = await MultiGet(
    Get(OutputA, InputA, input_a),
    Get(OutputB, InputB, input_b),
    Get(OutputC, InputC, input_c),
)
```

## Union Rules

### Defining a Union

```python
from pants.engine.unions import union

@union
class FormatRequest:
    """Base class for format requests."""
    pass
```

### Implementing a Union Member

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class BlackFormatRequest(FormatRequest):
    field_set: PythonFieldSet
```

### Registering Union Rules

```python
from pants.engine.unions import UnionRule

def rules():
    return [
        *collect_rules(),
        UnionRule(FormatRequest, BlackFormatRequest),
    ]
```

### Requesting All Union Members

```python
from pants.engine.unions import UnionMembership

@rule
async def format_all(
    request: FormatRequest,
    union_membership: UnionMembership,
) -> FormatResult:
    request_types = union_membership.get(FormatRequest)
    results = await MultiGet(
        Get(FormatResult, FormatRequest, req_type(...))
        for req_type in request_types
    )
    ...
```

## Goal Rules

### Basic Goal

```python
from pants.engine.goal import Goal, GoalSubsystem
from pants.engine.rules import goal_rule

class MyGoalSubsystem(GoalSubsystem):
    name = "my-goal"
    help = "Does something useful."

class MyGoal(Goal):
    subsystem_cls = MyGoalSubsystem
    environment_behavior = Goal.EnvironmentBehavior.LOCAL_ONLY

@goal_rule
async def my_goal_rule(
    console: Console,
    targets: Targets,
) -> MyGoal:
    for target in targets:
        console.print_stdout(f"Processing: {target.address}")
    return MyGoal(exit_code=0)
```

### Goal with Options

```python
from pants.option.option_types import BoolOption, StrOption

class MyGoalSubsystem(GoalSubsystem):
    name = "my-goal"
    help = "Does something."

    verbose = BoolOption(
        default=False,
        help="Enable verbose output.",
    )

    output_file = StrOption(
        default=None,
        help="Write output to file.",
    )

@goal_rule
async def my_goal_rule(
    console: Console,
    subsystem: MyGoalSubsystem,
) -> MyGoal:
    if subsystem.verbose:
        console.print_stdout("Verbose mode enabled")
    ...
```

## Console Output

```python
from pants.engine.console import Console

@goal_rule
async def my_goal(console: Console) -> MyGoal:
    # Standard output
    console.print_stdout("Normal message")
    console.print_stderr("Error message")

    # Colored output (respects --colors flag)
    console.print_stdout(console.red("Error!"))
    console.print_stdout(console.green("Success!"))
    console.print_stdout(console.blue("Info"))
    console.print_stdout(console.yellow("Warning"))
    console.print_stdout(console.cyan("Debug"))
    console.print_stdout(console.magenta("Special"))

    return MyGoal(exit_code=0)
```

## Side Effects

### Writing Files

```python
from pants.engine.fs import Workspace

@goal_rule
async def my_goal(
    workspace: Workspace,
) -> MyGoal:
    # Only in goal rules!
    workspace.write_digest(output_digest, path_prefix="output/")
    return MyGoal(exit_code=0)
```

### Interactive Processes

```python
from pants.engine.process import InteractiveProcess, InteractiveProcessResult

@goal_rule
async def my_goal() -> MyGoal:
    result = await Get(
        InteractiveProcessResult,
        InteractiveProcess(
            argv=["vim", "file.txt"],
            run_in_workspace=True,
        ),
    )
    return MyGoal(exit_code=result.exit_code)
```

## Logging

```python
import logging

logger = logging.getLogger(__name__)

@rule
async def my_rule(input: Input) -> Output:
    logger.info("Processing %s", input)
    logger.debug("Detailed info: %s", details)
    logger.warning("Something unexpected: %s", issue)
    logger.error("Failed to process: %s", error)
    return Output(...)
```

Control log level:
```bash
pants -ldebug my-goal ::
pants -ltrace my-goal ::  # Very verbose
```

## Process Execution

### Basic Process

```python
from pants.engine.process import Process, ProcessResult

@rule
async def compile(source: Source) -> Compiled:
    process = Process(
        argv=["compiler", source.path],
        input_digest=source.digest,
        output_files=("output.o",),
        description=f"Compiling {source.path}",
    )

    result = await Get(ProcessResult, Process, process)
    return Compiled(
        digest=result.output_digest,
        stdout=result.stdout.decode(),
    )
```

### Fallible Process

```python
from pants.engine.process import FallibleProcessResult

@rule
async def lint(sources: Sources) -> LintResult:
    process = Process(
        argv=["linter", *sources.files],
        input_digest=sources.digest,
        description="Linting",
    )

    # Doesn't raise on non-zero exit
    result = await Get(FallibleProcessResult, Process, process)

    return LintResult(
        exit_code=result.exit_code,
        stdout=result.stdout.decode(),
        stderr=result.stderr.decode(),
    )
```

### Process with Environment

```python
from pants.engine.environment import EnvironmentRequest, EnvironmentVars

@rule
async def my_rule() -> Output:
    env = await Get(
        EnvironmentVars,
        EnvironmentRequest(["PATH", "HOME", "MY_VAR"]),
    )

    process = Process(
        argv=["tool"],
        env=dict(env),
        ...
    )
    ...
```

## Data Classes

### Frozen Dataclasses (Required)

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class MyInput:
    path: str
    config: str | None = None

@dataclass(frozen=True)
class MyOutput:
    digest: Digest
    files: tuple[str, ...]  # Use tuple, not list
```

### With Validation

```python
@dataclass(frozen=True)
class ValidatedInput:
    value: int

    def __post_init__(self) -> None:
        if self.value < 0:
            raise ValueError("value must be non-negative")
```

## Best Practices

### DO

- Use `@dataclass(frozen=True)` for all rule inputs/outputs
- Use `tuple` instead of `list` for sequences
- Use `MultiGet` for parallel operations
- Keep rules pure (no side effects except in goal rules)
- Use descriptive `description` in Process

### DON'T

- Access filesystem directly (use Digest/PathGlobs)
- Use mutable data structures
- Include timestamps or random values in outputs
- Use `os.environ` (use EnvironmentRequest)
- Perform I/O in non-goal rules
