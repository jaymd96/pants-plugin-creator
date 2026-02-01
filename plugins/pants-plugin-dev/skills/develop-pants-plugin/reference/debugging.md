# Debugging Pants Plugins

Comprehensive guide to debugging Pants plugin issues.

## Debug Logging

### Enable Debug Output

```bash
# Full debug output
pants --debug my-goal ::

# Filter to your plugin
pants my-goal :: 2>&1 | grep -i "my_plugin\|rule"

# Specific log level
pants --level=debug my-goal ::
```

### Add Logging to Rules

```python
import logging

logger = logging.getLogger(__name__)

@rule
async def my_rule(target: WrappedTarget) -> Output:
    logger.debug(f"Processing: {target.target.address}")
    logger.info(f"Found {len(sources)} sources")
    logger.warning(f"Deprecated field used")
    logger.error(f"Failed to process")
    return Output(...)
```

## Common Issues and Solutions

### 1. Rule Not Being Called

**Symptoms**: Goal runs but your rule never executes

**Debugging Steps**:

```python
# 1. Verify rule is registered
# In register.py
def rules():
    all_rules = [*collect_rules(my_rules_module)]
    print(f"Registered rules: {all_rules}")  # Temporary debug
    return all_rules

# 2. Check if target matches
@rule
async def my_rule(target: WrappedTarget) -> Output:
    print(f"Rule called for: {target.target}")  # Should print
    if not isinstance(target.target, MyTargetType):
        print("Target type mismatch!")
    return Output(...)

# 3. Verify rule graph connection
# Run: pants peek --graph-representation=dot src/my_plugin
```

**Common Causes**:
- Rule not in `collect_rules()` scope
- Input type doesn't match what engine provides
- Missing `QueryRule` in tests

### 2. Type Errors at Startup

**Symptoms**: Pants fails immediately with type validation errors

```
Engine traceback:
  Rule `my_rule` has invalid type hints
```

**Fixes**:

```python
# Missing return type
@rule
async def bad(target: Target):  # ERROR: no return type
    pass

@rule
async def good(target: Target) -> Output:  # CORRECT
    return Output()

# Wrong parameter type
@rule
async def bad(target: Target) -> Output:  # ERROR: use WrappedTarget
    pass

@rule
async def good(target: WrappedTarget) -> Output:  # CORRECT
    return Output()

# Non-frozen dataclass
@dataclass
class BadOutput:  # ERROR: must be frozen
    value: str

@dataclass(frozen=True)
class GoodOutput:  # CORRECT
    value: str
```

### 3. Caching Not Working

**Symptoms**: Rule runs every time, even with same inputs

**Check determinism**:

```python
# BAD: Non-deterministic outputs
@rule
async def bad_rule() -> Output:
    return Output(
        timestamp=time.time(),      # Changes!
        random_id=uuid.uuid4(),     # Changes!
        env_var=os.environ["VAR"],  # May change!
    )

# GOOD: Deterministic outputs
@rule
async def good_rule(input: Input) -> Output:
    return Output(
        value=input.value,          # Derived from input
        hash=hashlib.sha256(data),  # Content-based
    )
```

**Check input types are hashable**:

```python
# Inputs must be frozen/hashable
@dataclass(frozen=True)  # Required!
class MyInput:
    value: str
    items: tuple[str, ...]  # Use tuple, not list
```

### 4. Process Execution Failures

**Symptoms**: External tool fails or behaves unexpectedly

```python
@rule
async def run_tool() -> Output:
    process = Process(
        argv=["my-tool", "--verbose"],  # Add verbose flag
        input_digest=digest,
        description="Running my-tool",
        # Debug: capture all output
        output_files=("stdout.log", "stderr.log"),
    )

    result = await Get(ProcessResult, Process, process)

    # Debug output
    print(f"Exit code: {result.exit_code}")
    print(f"Stdout: {result.stdout.decode()}")
    print(f"Stderr: {result.stderr.decode()}")

    if result.exit_code != 0:
        raise Exception(
            f"Tool failed:\n"
            f"stdout: {result.stdout.decode()}\n"
            f"stderr: {result.stderr.decode()}"
        )

    return Output(...)
```

### 5. Missing Files in Process Sandbox

**Symptoms**: Process can't find input files

```python
# Ensure files are in input_digest
source_digest = await Get(Digest, PathGlobs(["src/**/*.py"]))

process = Process(
    argv=["tool", "src/file.py"],
    input_digest=source_digest,  # Files must be here!
    description="Process files",
)
```

**Debug sandbox contents**:

```python
# List what's in the digest
snapshot = await Get(Snapshot, Digest, source_digest)
print(f"Files in sandbox: {snapshot.files}")
```

### 6. Dependency Resolution Issues

**Symptoms**: Can't find other targets, dependency graph wrong

```python
from pants.engine.target import TransitiveTargets, TransitiveTargetsRequest

@rule
async def process_with_deps(target: Target) -> Output:
    # Get all transitive dependencies
    transitive = await Get(
        TransitiveTargets,
        TransitiveTargetsRequest([target.address]),
    )

    print(f"Direct deps: {transitive.roots}")
    print(f"All deps: {transitive.closure}")

    return Output(...)
```

## Inspect Commands

### View Targets

```bash
# See all targets in directory
pants peek src::

# See specific target with all fields
pants peek src:my-target

# See dependencies
pants dependencies src:my-target

# See dependents (what depends on this)
pants dependents src:my-target
```

### View Rule Graph

```bash
# List backends/plugins
pants backends

# Export rule graph as DOT
pants peek --graph-representation=dot > graph.dot
dot -Tpng graph.dot -o graph.png
```

### Validate Plugin

```bash
# Check plugin loads without errors
pants help my-goal

# List available goals
pants help goals

# Check target types
pants help targets
```

## Testing Debug Techniques

### Isolated Rule Testing

```python
def test_rule_debug(rule_runner):
    # Enable debug output in tests
    rule_runner.set_options(["--debug"])

    rule_runner.write_files({
        "BUILD": "my_target(name='test')",
    })

    # Print what we're testing
    print(f"Targets: {rule_runner.get_target(Address('', target_name='test'))}")

    result = rule_runner.request(Output, [input])
    print(f"Result: {result}")

    assert result.exit_code == 0
```

### Debug Fixture

```python
@pytest.fixture
def debug_rule_runner():
    runner = RuleRunner(
        rules=rules(),
        target_types=target_types(),
    )
    runner.set_options([
        "--debug",
        "--my-plugin-verbose=true",
    ])
    return runner
```

## Environment Issues

### Check Pants Version

```python
from pants.version import PANTS_SEMVER

print(f"Pants version: {PANTS_SEMVER}")
```

### Check Plugin is Loaded

```python
# In register.py - temporary debug
def rules():
    import sys
    print(f"Plugin loaded! Python: {sys.version}", file=sys.stderr)
    return collect_rules()
```

## Remote Debugging

For complex issues, attach a debugger:

```python
@rule
async def my_rule() -> Output:
    import debugpy
    debugpy.listen(5678)
    debugpy.wait_for_client()
    debugpy.breakpoint()

    # Your code here
    return Output(...)
```

Then connect VS Code or PyCharm debugger to port 5678.
