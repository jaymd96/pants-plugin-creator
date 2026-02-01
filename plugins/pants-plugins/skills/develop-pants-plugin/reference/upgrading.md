# Upgrading Pants Plugins

Guide to upgrading plugins for new Pants versions.

## Before You Upgrade

### 1. Check Current Compatibility

```bash
# Check what Pants version you support
grep -r "pants" pyproject.toml

# Check current Pants version in use
pants --version
```

### 2. Read the Release Notes

- [Pants Release Notes](https://www.pantsbuild.org/stable/docs/releases)
- [Plugin Upgrade Guide](https://www.pantsbuild.org/stable/docs/writing-plugins/common-plugin-tasks/plugin-upgrade-guide)

### 3. Check Deprecation Warnings

```bash
# Run with deprecation warnings visible
pants --print-stacktrace my-goal ::
```

## Version Detection

### Check Pants Version in Code

```python
from pants.version import PANTS_SEMVER
from packaging.version import Version

# Simple version check
if PANTS_SEMVER >= Version("2.18.0"):
    # Use new API
    pass
else:
    # Use old API
    pass
```

### Conditional Imports

```python
from pants.version import PANTS_SEMVER
from packaging.version import Version

if PANTS_SEMVER >= Version("2.18.0"):
    from pants.engine.new_location import SomeClass
else:
    from pants.engine.old_location import SomeClass
```

### Feature Detection (Preferred)

```python
try:
    from pants.engine.new_module import NewFeature
    HAS_NEW_FEATURE = True
except ImportError:
    HAS_NEW_FEATURE = False
    from pants.engine.old_module import OldFeature as NewFeature
```

## Common Breaking Changes

### Import Path Changes

Pants frequently reorganizes modules. Watch for:

```python
# Old (2.17)
from pants.backend.python.util_rules.interpreter_constraints import InterpreterConstraints

# New (2.18+)
from pants.backend.python.subsystems.interpreter_constraints import InterpreterConstraints
```

**Solution**: Use try/except imports:

```python
try:
    from pants.backend.python.subsystems.interpreter_constraints import InterpreterConstraints
except ImportError:
    from pants.backend.python.util_rules.interpreter_constraints import InterpreterConstraints
```

### Type Signature Changes

```python
# Old: Single parameter
@rule
async def old_rule(target: Target) -> Output:
    pass

# New: Wrapped parameter
@rule
async def new_rule(wrapped: WrappedTarget) -> Output:
    target = wrapped.target
    pass
```

### Field Type Changes

```python
# Old
class OldSourcesField(StringSequenceField):
    alias = "sources"

# New: Use MultipleSourcesField for better glob support
class NewSourcesField(MultipleSourcesField):
    default = ("*.py",)
```

### Goal API Changes

```python
# Old style
class MyGoalSubsystem(GoalSubsystem):
    name = "my-goal"
    help = "..."

# Some versions changed to
class MyGoalSubsystem(GoalSubsystem):
    name = "my-goal"
    help = "..."
    # New required attributes
    required = True
```

### Process API Changes

```python
# Old
process = Process(
    argv=["tool"],
    input_digest=digest,
    description="Run tool",
)

# New: Additional optional parameters
process = Process(
    argv=["tool"],
    input_digest=digest,
    description="Run tool",
    output_files=("out.txt",),  # New in some versions
    cache_scope=ProcessCacheScope.PER_SESSION,  # New
)
```

## Upgrade Checklist

### Step 1: Update Dependencies

```toml
# pyproject.toml
[project]
requires-python = ">=3.12,<4"  # Match Pants requirements

[project.optional-dependencies]
dev = [
    "pantsbuild.pants>=2.28,<2.32",  # Pin to compatible range
]
```

### Step 2: Fix Import Errors

Run and check for import failures:

```bash
python -c "from my_plugin.register import rules, target_types"
```

Fix any import errors with version detection.

### Step 3: Fix Type Errors

Pants validates types at startup:

```bash
pants help my-goal
```

Fix any type validation errors.

### Step 4: Run Tests

```bash
hatch run test
```

Fix test failures.

### Step 5: Test Manually

```bash
# Test in a real project
pants my-goal src::
```

## Multi-Version Support

### Supporting Multiple Pants Versions

```python
# compat.py - Version compatibility layer

from pants.version import PANTS_SEMVER
from packaging.version import Version

# Version flags
PANTS_2_18_PLUS = PANTS_SEMVER >= Version("2.18.0")
PANTS_2_19_PLUS = PANTS_SEMVER >= Version("2.19.0")

# Conditional imports
if PANTS_2_18_PLUS:
    from pants.engine.new_module import Feature
else:
    from pants.engine.old_module import Feature

# Re-export
__all__ = ["Feature", "PANTS_2_18_PLUS", "PANTS_2_19_PLUS"]
```

### Use in Rules

```python
from my_plugin.compat import Feature, PANTS_2_18_PLUS

@rule
async def my_rule() -> Output:
    if PANTS_2_18_PLUS:
        # Use new behavior
        pass
    else:
        # Use old behavior
        pass
```

## Testing Multiple Versions

### CI Matrix

```yaml
# .github/workflows/test.yml
jobs:
  test:
    strategy:
      matrix:
        pants-version: ["2.17", "2.18", "2.19"]
    steps:
      - uses: actions/checkout@v3
      - name: Install Pants
        run: |
          pip install pantsbuild.pants==${{ matrix.pants-version }}.*
      - name: Test
        run: hatch run test
```

### Local Multi-Version Testing

```bash
# Test with specific version
pip install pantsbuild.pants==2.17.0
hatch run test

# Test with another version
pip install pantsbuild.pants==2.18.0
hatch run test
```

## Deprecation Handling

### Mark Deprecated Features

```python
import warnings

def deprecated_function():
    warnings.warn(
        "deprecated_function is deprecated, use new_function instead",
        DeprecationWarning,
        stacklevel=2,
    )
    # Old implementation
```

### Remove in Next Major Version

```python
from pants.version import PANTS_SEMVER
from packaging.version import Version

if PANTS_SEMVER >= Version("3.0.0"):
    # Removed deprecated feature
    raise ImportError("deprecated_module removed in Pants 3.0")
```

## Resources

- [Pants Plugin Upgrade Guide](https://www.pantsbuild.org/stable/docs/writing-plugins/common-plugin-tasks/plugin-upgrade-guide)
- [Pants Changelog](https://github.com/pantsbuild/pants/blob/main/CHANGELOG.md)
- [Pants API Reference](https://www.pantsbuild.org/stable/reference/)
