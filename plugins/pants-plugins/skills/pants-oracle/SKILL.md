---
description: Expert knowledge base for all Pants build system questions - architecture, rules, targets, fields, caching, processes, testing, and plugin development
user_invocable: true
triggers:
  - "pants"
  - "how does pants"
  - "what is a pants"
  - "pants build"
  - "pants rule"
  - "pants target"
  - "pants caching"
  - "BUILD file"
---

# Pants Oracle

You are an expert on the Pants build system. Use this comprehensive knowledge base to answer any question about Pants architecture, APIs, rules, targets, caching, and plugin development.

## Quick Reference Index

| Topic | See Section |
|-------|-------------|
| Architecture | [Engine Architecture](#engine-architecture) |
| Targets & Fields | [Target API](#target-api) |
| Rules | [Rules API](#rules-api) |
| File Operations | [File System](#file-system-operations) |
| Process Execution | [Running Processes](#running-processes) |
| Caching | [Caching System](#caching-system) |
| Testing | [Testing Plugins](#testing-plugins) |
| Goals | [Goal Rules](#goal-rules) |
| Subsystems | [Options & Subsystems](#options-and-subsystems) |

---

## Engine Architecture

### Hybrid Rust + Python Design

Pants uses a **Rust engine** for performance-critical operations (file watching, process execution, caching) with a **Python API** for plugin development.

```
User Request → Rule Graph Resolution → Parallel Execution → Cached Results
```

### Key Principles

1. **Declarative Rule Graph**: Rules declare inputs/outputs; engine determines execution order
2. **Content-Addressable Storage**: Files stored by hash, enabling deduplication
3. **Hermetic Execution**: Processes run in sandboxes with explicit inputs only
4. **Automatic Parallelization**: Independent rules execute concurrently
5. **Fine-Grained Caching**: Results cached by input hash

### Rule Graph

```python
@rule
async def rule_a(input: InputA) -> OutputA:
    # Request another rule's output
    result_b = await Get(OutputB, InputB(...))
    return OutputA(...)
```

The engine builds a DAG from rule signatures and resolves dependencies automatically.

---

## Target API

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Target** | Metadata about code in BUILD files |
| **Field** | Individual piece of target metadata |
| **Address** | Unique identifier (`path:name` or `path/file.py:name`) |
| **FieldSet** | Typed field requirements for rules |

### Target Definition

```python
from pants.engine.target import (
    Target, COMMON_TARGET_FIELDS, Dependencies,
    SingleSourceField, StringField
)

class CustomField(StringField):
    alias = "custom_field"
    help = "A custom field."
    default = "default_value"

class MyTarget(Target):
    alias = "my_target"
    core_fields = (
        *COMMON_TARGET_FIELDS,
        Dependencies,
        SingleSourceField,
        CustomField,
    )
    help = "A custom target type."
```

### Field Types

| Type | Use Case | Value Type |
|------|----------|------------|
| `StringField` | Single string | `str` |
| `BoolField` | Boolean flag | `bool` |
| `IntField` | Integer (rejects floats) | `int` |
| `FloatField` | Decimal (rejects ints) | `float` |
| `StringSequenceField` | List of strings | `tuple[str, ...]` |
| `DictStringToStringField` | String dict | `FrozenDict[str, str]` |
| `TriBoolField` | True/False/None | `bool \| None` |
| `SingleSourceField` | One source file | `str` |
| `MultipleSourcesField` | Multiple sources | `tuple[str, ...]` |

### Accessing Field Values

```python
# Direct access (raises if missing)
value = target[MyField].value

# Safe access with default
field = target.get(MyField)
value = field.value if field else "default"

# Check existence
if target.has_field(MyField):
    ...
```

### Addresses

```python
from pants.engine.addresses import Address

# Construct addresses
addr = Address("src/python", target_name="lib")  # src/python:lib
addr = Address("src", relative_file_name="app.py")  # src/app.py

# Get string representation
spec = str(addr)  # or addr.spec
```

### FieldSet Pattern

```python
from dataclasses import dataclass
from pants.engine.target import FieldSet

@dataclass(frozen=True)
class MyFieldSet(FieldSet):
    required_fields = (SourcesField,)  # Filters targets

    sources: SourcesField
    dependencies: Dependencies  # Optional fields get defaults
```

---

## Rules API

### Rule Basics

Rules are **pure async functions** decorated with `@rule`. They:
- Declare typed inputs and outputs
- Cannot have side effects (no I/O, no global state)
- Request other rules via `Get()` and `MultiGet()`

```python
from pants.engine.rules import rule, Get, MultiGet

@rule
async def my_rule(input: MyInput) -> MyOutput:
    # Request single result
    result = await Get(OtherOutput, OtherInput, other_input)

    # Request multiple in parallel
    results = await MultiGet(
        Get(Output, Input, inp) for inp in inputs
    )

    return MyOutput(...)
```

### Rule Registration

```python
# register.py
from pants.engine.rules import collect_rules
from my_plugin import rules as rules_module

def rules():
    return [
        *collect_rules(rules_module),
    ]

def target_types():
    return [MyTarget]
```

### Union Rules

For extensible plugin points:

```python
from pants.engine.unions import UnionRule, union

@union
class LintRequest:
    pass

@dataclass(frozen=True)
class MyLintRequest(LintRequest):
    field_set: MyFieldSet

# Register the union
UnionRule(LintRequest, MyLintRequest)
```

---

## File System Operations

### Digest

A `Digest` is a lightweight reference to files in content-addressable storage.

```python
from pants.engine.fs import (
    Digest, Snapshot, PathGlobs, CreateDigest,
    DigestContents, MergeDigests, FileContent
)
```

### Reading Files

```python
# Read files matching patterns
snapshot = await Get(Snapshot, PathGlobs(["src/**/*.py"]))
print(snapshot.files)  # Sorted tuple of paths

# Read file contents (loads into memory)
contents = await Get(DigestContents, Digest, digest)
for file_content in contents:
    print(file_content.path, file_content.content)
```

### Creating Files

```python
# Create files programmatically
digest = await Get(
    Digest,
    CreateDigest([
        FileContent("output.txt", b"content here"),
        FileContent("script.sh", b"#!/bin/bash\necho hi", is_executable=True),
    ])
)
```

### Merging Digests

```python
# Combine multiple digests
merged = await Get(Digest, MergeDigests([digest1, digest2, digest3]))
```

### Path Manipulation

```python
from pants.engine.fs import AddPrefix, RemovePrefix, DigestSubset

# Add prefix to all paths
prefixed = await Get(Digest, AddPrefix(digest, "subdir"))

# Remove prefix
unprefixed = await Get(Digest, RemovePrefix(digest, "subdir"))

# Extract subset
subset = await Get(Digest, DigestSubset(digest, PathGlobs(["*.py"])))
```

---

## Running Processes

### Process Class

```python
from pants.engine.process import Process, ProcessResult

process = Process(
    argv=["tool", "--flag", "arg"],
    input_digest=input_files,
    description="Running tool",
    output_files=("output.txt",),
    output_directories=("results/",),
    env={"VAR": "value"},
    timeout_seconds=300,
)

result = await Get(ProcessResult, Process, process)
print(result.stdout.decode())
print(result.stderr.decode())
output_digest = result.output_digest
```

### Key Properties

| Property | Description |
|----------|-------------|
| `argv` | Command and arguments |
| `input_digest` | Files available in sandbox |
| `output_files` | Files to capture from sandbox |
| `output_directories` | Directories to capture |
| `env` | Environment variables |
| `description` | Shown in UI during execution |
| `timeout_seconds` | Process timeout |
| `cache_scope` | Caching behavior |

### Sandboxing

Processes run in isolated temp directories:
- Only files in `input_digest` are available
- Environment is stripped (explicit `env` only)
- Output must be declared to capture

---

## Caching System

### How Caching Works

Cache keys are computed from:
1. Rule name
2. All input values (hashed)
3. File content hashes (not timestamps)
4. Declared environment variables

```
Cache Key = hash(rule_name, input_types, input_values, file_hashes, env_vars)
```

### Ensuring Cacheability

**Rule outputs must be:**
- Frozen dataclasses (`@dataclass(frozen=True)`)
- Deterministic (no timestamps, random values, set iteration)
- Use tuples instead of lists

```python
# GOOD: Cacheable
@dataclass(frozen=True)
class GoodOutput:
    items: tuple[str, ...]  # Immutable

# BAD: Not cacheable
@dataclass
class BadOutput:
    items: list[str]  # Mutable
    timestamp: float  # Non-deterministic
```

### Process Caching

Processes cache when `exit_code == 0` by default. Control with `ProcessCacheScope`:

```python
from pants.engine.process import ProcessCacheScope

Process(
    ...,
    cache_scope=ProcessCacheScope.PER_SESSION,  # Don't persist
)
```

---

## Goal Rules

### Creating a Goal

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
async def my_goal(
    console: Console,
    targets: Targets,
    subsystem: MyGoalSubsystem,
) -> MyGoal:
    for target in targets:
        console.print_stdout(f"Processing {target.address}")
    return MyGoal(exit_code=0)
```

### Console Output

```python
from pants.engine.console import Console

@goal_rule
async def my_goal(console: Console) -> MyGoal:
    console.print_stdout("Normal output")
    console.print_stderr("Error output")
    console.print_stdout(console.red("Colored text"))
    return MyGoal(exit_code=0)
```

---

## Options and Subsystems

### Creating a Subsystem

```python
from pants.option.option_types import (
    StrOption, BoolOption, IntOption, StrListOption
)
from pants.option.subsystem import Subsystem

class MySubsystem(Subsystem):
    options_scope = "my-tool"
    help = "Configuration for my tool."

    config = StrOption(
        default=None,
        help="Path to config file.",
    )

    skip = BoolOption(
        default=False,
        help="Skip running this tool.",
    )

    args = StrListOption(
        default=[],
        help="Additional arguments.",
    )

    timeout = IntOption(
        default=60,
        advanced=True,
        help="Timeout in seconds.",
    )
```

### Using in Rules

```python
@rule
async def my_rule(subsystem: MySubsystem) -> Output:
    if subsystem.skip:
        return Output.skip()

    args = ["tool"]
    if subsystem.config:
        args.extend(["--config", subsystem.config])
    args.extend(subsystem.args)

    ...
```

### Option Types

| Type | pants.toml Example |
|------|-------------------|
| `StrOption` | `config = "path/to/config"` |
| `BoolOption` | `skip = true` |
| `IntOption` | `timeout = 120` |
| `FloatOption` | `threshold = 0.8` |
| `StrListOption` | `args = ["--flag", "value"]` |
| `EnumOption` | `mode = "strict"` |
| `DictOption` | `env = {"KEY": "value"}` |

---

## Testing Plugins

### RuleRunner Setup

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

### Writing Tests

```python
def test_my_rule(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "BUILD": 'my_target(name="test", sources=["*.py"])',
        "main.py": "print('hello')",
    })

    # Set options
    rule_runner.set_options(["--my-tool-enabled=true"])

    # Request rule output
    result = rule_runner.request(MyOutput, [MyInput(...)])

    assert result.exit_code == 0
```

### Testing Goals

```python
def test_goal(rule_runner: RuleRunner) -> None:
    rule_runner.write_files({
        "src/BUILD": 'my_target(name="test")',
    })

    result = rule_runner.run_goal_rule(MyGoal, args=["src:test"])

    assert result.exit_code == 0
    assert "expected output" in result.stdout
```

---

## Common Patterns

### Linter Pattern

```python
@dataclass(frozen=True)
class MyLinterFieldSet(FieldSet):
    required_fields = (SourcesField,)
    sources: SourcesField

class MyLinterRequest(LintTargetsRequest):
    field_set_type = MyLinterFieldSet
    tool_subsystem = MyLinterSubsystem

@rule
async def run_linter(
    request: MyLinterRequest.Batch,
    tool: MyLinterSubsystem,
) -> LintResult:
    sources = await Get(
        SourceFiles,
        SourceFilesRequest(fs.sources for fs in request.elements),
    )

    process = Process(
        argv=["linter", *sources.files],
        input_digest=sources.snapshot.digest,
        description="Linting...",
    )

    result = await Get(FallibleProcessResult, Process, process)
    return LintResult.from_fallible_process_result(result)
```

### Code Generator Pattern

```python
class GenerateFromProto(GenerateSourcesRequest):
    input = ProtoSourceField
    output = PythonSourceField

@rule
async def generate_python_from_proto(
    request: GenerateFromProto,
) -> GeneratedSources:
    # Get proto files
    sources = await Get(
        HydratedSources,
        HydrateSourcesRequest(request.protocol_sources),
    )

    # Run protoc
    result = await Get(
        ProcessResult,
        Process(argv=["protoc", "--python_out=.", *sources.files], ...),
    )

    return GeneratedSources(result.output_digest)
```

---

## Reference Documentation

For detailed information, see:
- [reference/architecture.md](reference/architecture.md) - Engine internals
- [reference/targets.md](reference/targets.md) - Complete Target API
- [reference/rules.md](reference/rules.md) - Complete Rules API
- [reference/testing.md](reference/testing.md) - Testing patterns

## External Resources

- [Official Pants Documentation](https://www.pantsbuild.org/stable/docs)
- [Pants GitHub Repository](https://github.com/pantsbuild/pants)
- [Pants Community Slack](https://pantsbuild.slack.com)
