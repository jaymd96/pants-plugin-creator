# Code Generation Plugin Pattern

Complete pattern for creating a Pants plugin that generates code from definition files.

## Overview

A codegen plugin:
1. Defines targets with definition files (protobuf, thrift, etc.)
2. Runs a code generator tool via Process
3. Outputs generated source files as a Digest
4. Integrates with the build graph

## Complete Implementation

### targets.py

```python
"""Target types for the codegen plugin."""

from pants.engine.target import (
    COMMON_TARGET_FIELDS,
    StringField,
    StringSequenceField,
    Target,
)


class ProtobufSourceField(StringField):
    """The protobuf definition file."""

    alias = "source"
    help = "Path to the .proto file"


class ProtobufDependenciesField(StringSequenceField):
    """Dependencies on other proto files."""

    alias = "proto_deps"
    help = "Other proto files this one imports"
    default = ()


class ProtobufLanguageField(StringField):
    """Target language for code generation."""

    alias = "language"
    help = "Target language: python, go, java"
    default = "python"


class ProtobufTarget(Target):
    """A protobuf definition file for code generation.

    Example:
        protobuf_source(
            name="messages",
            source="messages.proto",
            language="python",
        )
    """

    alias = "protobuf_source"
    help = "A protobuf file for code generation"

    core_fields = (
        *COMMON_TARGET_FIELDS,
        ProtobufSourceField,
        ProtobufDependenciesField,
        ProtobufLanguageField,
    )
```

### subsystem.py

```python
"""Configuration for the codegen plugin."""

from pants.option.subsystem import Subsystem
from pants.option.option_types import StrOption, StrListOption


class ProtobufSubsystem(Subsystem):
    """Configuration for protobuf code generation."""

    options_scope = "protobuf"
    help = "Protobuf code generation settings"

    protoc_version = StrOption(
        default="3.21.12",
        help="Version of protoc to use",
    )

    extra_args = StrListOption(
        help="Extra arguments to pass to protoc",
    )
```

### rules.py

```python
"""Rules for code generation."""

from dataclasses import dataclass
from typing import Sequence

from pants.engine.rules import collect_rules, rule, Get
from pants.engine.target import WrappedTarget
from pants.engine.process import Process, ProcessResult
from pants.engine.fs import (
    Digest,
    CreateDigest,
    FileContent,
    MergeDigests,
    PathGlobs,
    Snapshot,
)

from my_codegen.targets import (
    ProtobufTarget,
    ProtobufSourceField,
    ProtobufLanguageField,
)
from my_codegen.subsystem import ProtobufSubsystem


@dataclass(frozen=True)
class GeneratedCode:
    """Result of code generation."""

    output_digest: Digest
    source_files: tuple[str, ...]


@dataclass(frozen=True)
class CodegenRequest:
    """Request to generate code from a protobuf file."""

    source_path: str
    language: str
    output_dir: str


@rule
async def generate_protobuf(
    request: CodegenRequest,
    subsystem: ProtobufSubsystem,
) -> GeneratedCode:
    """Generate code from a protobuf file."""

    # Get the source file
    source_digest = await Get(Digest, PathGlobs([request.source_path]))

    # Determine output file extension based on language
    output_extensions = {
        "python": "_pb2.py",
        "go": ".pb.go",
        "java": ".java",
    }
    ext = output_extensions.get(request.language, "_pb2.py")

    # Build the protoc command
    argv = [
        "protoc",
        f"--{request.language}_out={request.output_dir}",
        *subsystem.extra_args,
        request.source_path,
    ]

    # Create the process
    process = Process(
        argv=argv,
        input_digest=source_digest,
        output_directories=(request.output_dir,),
        description=f"Generate {request.language} from {request.source_path}",
    )

    # Run protoc
    result = await Get(ProcessResult, Process, process)

    if result.exit_code != 0:
        raise Exception(f"protoc failed: {result.stderr.decode()}")

    # Get the generated files
    output_snapshot = await Get(
        Snapshot,
        Digest,
        result.output_digest,
    )

    return GeneratedCode(
        output_digest=result.output_digest,
        source_files=tuple(output_snapshot.files),
    )


@rule
async def generate_from_target(
    wrapped: WrappedTarget,
    subsystem: ProtobufSubsystem,
) -> GeneratedCode:
    """Generate code from a protobuf target."""

    target = wrapped.target

    if not isinstance(target, ProtobufTarget):
        return GeneratedCode(output_digest=Digest(), source_files=())

    source = target[ProtobufSourceField].value
    language = target[ProtobufLanguageField].value

    request = CodegenRequest(
        source_path=source,
        language=language,
        output_dir="generated",
    )

    return await Get(GeneratedCode, CodegenRequest, request)


def rules() -> Sequence:
    return collect_rules()
```

### goals.py

```python
"""Goal for running code generation."""

from typing import Sequence

from pants.engine.goal import Goal, GoalSubsystem, goal_rule
from pants.engine.console import Console
from pants.engine.rules import collect_rules, Get, MultiGet
from pants.engine.target import Targets, WrappedTarget
from pants.engine.fs import Workspace

from my_codegen.targets import ProtobufTarget
from my_codegen.rules import GeneratedCode, generate_from_target


class CodegenGoalSubsystem(GoalSubsystem):
    """Options for the codegen goal."""

    name = "protobuf-gen"
    help = "Generate code from protobuf definitions"


class CodegenGoal(Goal):
    """Goal to generate code from protobuf."""

    subsystem_cls = CodegenGoalSubsystem


@goal_rule
async def run_codegen_goal(
    console: Console,
    targets: Targets,
    workspace: Workspace,
) -> CodegenGoal:
    """Generate code from all protobuf targets."""

    # Filter to protobuf targets
    proto_targets = [t for t in targets if isinstance(t, ProtobufTarget)]

    if not proto_targets:
        console.print_stdout("No protobuf targets found")
        return CodegenGoal(exit_code=0)

    # Generate code for all targets in parallel
    results = await MultiGet(
        Get(GeneratedCode, WrappedTarget, WrappedTarget(t))
        for t in proto_targets
    )

    # Report results and write files
    total_files = 0
    for target, result in zip(proto_targets, results):
        if result.source_files:
            console.print_stdout(f"Generated from {target.name}:")
            for f in result.source_files:
                console.print_stdout(f"  - {f}")
            total_files += len(result.source_files)

            # Write generated files to workspace
            workspace.write_digest(result.output_digest)

    console.print_stdout(f"\nTotal: {total_files} files generated")

    return CodegenGoal(exit_code=0)


def rules() -> Sequence:
    return collect_rules()
```

### register.py

```python
"""Register the codegen plugin."""

from typing import Sequence

from pants.engine.rules import collect_rules

from my_codegen import rules as codegen_rules
from my_codegen import goals as codegen_goals
from my_codegen.subsystem import ProtobufSubsystem
from my_codegen.targets import ProtobufTarget


def rules() -> Sequence:
    return [
        *collect_rules(codegen_rules),
        *collect_rules(codegen_goals),
        *ProtobufSubsystem.rules(),
    ]


def target_types() -> Sequence:
    return [ProtobufTarget]
```

## Usage

### BUILD file

```python
protobuf_source(
    name="messages",
    source="messages.proto",
    language="python",
)
```

### pants.toml

```toml
[GLOBAL]
backend_packages = ["my_codegen"]

[protobuf]
extra_args = ["--experimental_allow_proto3_optional"]
```

### Generate code

```bash
pants protobuf-gen src/protos::
```

## Key Points

1. **Digest** represents file content in the engine
2. **output_directories** in Process captures generated files
3. **Workspace.write_digest** writes files to the filesystem
4. **CodegenRequest** allows parameterized code generation
5. Generated code is cached based on input content hash

## Testing

```python
from pants.testutil.rule_runner import RuleRunner, QueryRule

def test_codegen(rule_runner):
    rule_runner.write_files({
        "BUILD": '''
protobuf_source(
    name="test",
    source="test.proto",
    language="python",
)
        ''',
        "test.proto": '''
syntax = "proto3";
message TestMessage {
    string name = 1;
}
        ''',
    })

    result = rule_runner.run_goal(CodegenGoal, [":test"])
    assert result.exit_code == 0
```

## Integration with Build Graph

To integrate generated code with the build:

```python
@rule
async def inject_generated_sources(
    request: PythonSourcesGeneratorRequest,
) -> GeneratedSources:
    """Inject generated protobuf sources into Python targets."""

    codegen_result = await Get(GeneratedCode, CodegenRequest, ...)

    return GeneratedSources(codegen_result.output_digest)
```

This allows generated code to be automatically included in dependent targets.
