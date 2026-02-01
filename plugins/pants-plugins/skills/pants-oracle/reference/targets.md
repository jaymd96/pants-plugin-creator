# Complete Target API Reference

## Target Definition

### Basic Structure

```python
from pants.engine.target import (
    Target,
    COMMON_TARGET_FIELDS,
    Dependencies,
    SingleSourceField,
    MultipleSourcesField,
)

class MyTarget(Target):
    alias = "my_target"  # Name in BUILD files
    core_fields = (
        *COMMON_TARGET_FIELDS,  # tags, description
        Dependencies,
        MySourcesField,
        MyCustomField,
    )
    help = "Description shown in `pants help my_target`."
```

### Registration

```python
# register.py
def target_types():
    return [MyTarget]
```

## Field Types

### String Fields

```python
from pants.engine.target import StringField

class MyStringField(StringField):
    alias = "my_field"
    help = "A string value."
    default = "default_value"  # Or required = True

# With validation
class ModeField(StringField):
    alias = "mode"
    valid_choices = ("debug", "release", "test")
    default = "debug"
```

### Boolean Fields

```python
from pants.engine.target import BoolField

class EnabledField(BoolField):
    alias = "enabled"
    default = True
    help = "Whether this target is enabled."
```

### Integer/Float Fields

```python
from pants.engine.target import IntField, FloatField

class TimeoutField(IntField):
    alias = "timeout"
    default = 60
    help = "Timeout in seconds."

class ThresholdField(FloatField):
    alias = "threshold"
    default = 0.8
    help = "Coverage threshold."
```

### Sequence Fields

```python
from pants.engine.target import StringSequenceField

class TagsField(StringSequenceField):
    alias = "extra_tags"
    default = ()
    help = "Additional tags."
```

### Dictionary Fields

```python
from pants.engine.target import DictStringToStringField

class EnvField(DictStringToStringField):
    alias = "env"
    help = "Environment variables."
```

### Source Fields

```python
from pants.engine.target import SingleSourceField, MultipleSourcesField

# Single file
class MySourceField(SingleSourceField):
    expected_file_extensions = (".proto",)

# Multiple files with globs
class MySourcesField(MultipleSourcesField):
    default = ("*.py", "!*_test.py")
    expected_file_extensions = (".py",)
```

## Custom Validation

```python
from pants.engine.target import StringField, InvalidFieldException

class VersionField(StringField):
    alias = "version"

    @classmethod
    def compute_value(
        cls,
        raw_value: str | None,
        address: Address,
    ) -> str | None:
        value = super().compute_value(raw_value, address)
        if value and not value.startswith("v"):
            raise InvalidFieldException(
                f"Version must start with 'v', got {value!r}"
            )
        return value
```

## Accessing Fields

### Direct Access

```python
# Raises KeyError if field not registered
value = target[MyField].value

# Safe access
field = target.get(MyField)
if field is not None:
    value = field.value
```

### Field Existence

```python
if target.has_field(MyField):
    ...

if target.has_fields([FieldA, FieldB]):
    ...
```

### Field Inheritance

Fields use class inheritance for polymorphism:

```python
class BaseSourceField(MultipleSourcesField):
    pass

class PythonSourceField(BaseSourceField):
    expected_file_extensions = (".py",)

class JavaSourceField(BaseSourceField):
    expected_file_extensions = (".java",)

# Both match has_field(BaseSourceField)
```

## Address API

### Construction

```python
from pants.engine.addresses import Address

# Directory target
addr = Address("src/python", target_name="lib")  # src/python:lib

# File target
addr = Address(
    "src/python",
    target_name="app",
    relative_file_name="main.py"
)  # src/python/main.py:app

# Default target name (matches directory)
addr = Address("src/python")  # src/python:python
```

### Properties

```python
addr.spec              # "src/python:lib"
addr.spec_path         # "src/python"
addr.target_name       # "lib"
addr.relative_file_path  # "main.py" or None
str(addr)              # "src/python:lib"
```

## Resolving Targets

### From User Specs

```python
@rule
async def my_rule(targets: Targets) -> Output:
    for target in targets:
        ...
```

### All Targets

```python
from pants.engine.target import AllTargets

@rule
async def my_rule(all_targets: AllTargets) -> Output:
    for target in all_targets:
        ...
```

### By Address

```python
from pants.engine.target import WrappedTarget, WrappedTargetRequest

wrapped = await Get(
    WrappedTarget,
    WrappedTargetRequest(address, description_of_origin="my rule"),
)
target = wrapped.target
```

### Transitive Dependencies

```python
from pants.engine.target import (
    TransitiveTargets,
    TransitiveTargetsRequest,
)

transitive = await Get(
    TransitiveTargets,
    TransitiveTargetsRequest([target.address]),
)

# Direct dependencies
for dep in transitive.roots:
    ...

# All transitive dependencies
for dep in transitive.closure:
    ...
```

## FieldSet Pattern

### Definition

```python
from dataclasses import dataclass
from pants.engine.target import FieldSet

@dataclass(frozen=True)
class MyFieldSet(FieldSet):
    required_fields = (SourcesField,)  # Target must have these

    # Fields to extract
    sources: SourcesField
    dependencies: Dependencies
    timeout: TimeoutField  # Gets default if not required

    @classmethod
    def opt_out(cls, tgt: Target) -> bool:
        """Optionally skip targets even if they have required fields."""
        return tgt.get(SkipField).value is True
```

### Usage in Rules

```python
@rule
async def process_target(field_set: MyFieldSet) -> Output:
    sources = await Get(
        SourceFiles,
        SourceFilesRequest([field_set.sources]),
    )
    ...
```

### With Lint/Test/Fmt Goals

```python
class MyLinterRequest(LintTargetsRequest):
    field_set_type = MyFieldSet
    tool_subsystem = MyLinterSubsystem
```

## Source Hydration

Sources need async hydration:

```python
from pants.core.util_rules.source_files import (
    SourceFiles,
    SourceFilesRequest,
)

# Single source field
sources = await Get(
    SourceFiles,
    SourceFilesRequest([target[SourcesField]]),
)

# Multiple targets
sources = await Get(
    SourceFiles,
    SourceFilesRequest(
        [t[SourcesField] for t in targets],
        for_sources_types=[PythonSourceField],
        enable_codegen=True,  # Include generated sources
    ),
)

# Access files
for path in sources.files:
    ...

# Get digest for process input
digest = sources.snapshot.digest
```
