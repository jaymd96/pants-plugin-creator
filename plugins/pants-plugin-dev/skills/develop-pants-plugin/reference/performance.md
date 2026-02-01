# Pants Plugin Performance

Guide to optimizing Pants plugin performance.

## Understanding Pants Caching

### How Caching Works

Pants caches rule outputs based on:
1. **Input types and values** - All rule parameters
2. **File content hashes** - Not timestamps
3. **Environment variables** - If explicitly declared

```python
@rule
async def cached_rule(input: Input) -> Output:
    # Same input = same output = cache hit
    return Output(value=process(input))
```

### Cache Keys

```
Cache Key = hash(
    rule_name,
    input_type_1_value,
    input_type_2_value,
    ...
    file_contents_hash,
    declared_env_vars
)
```

## Ensuring Cacheability

### Rule Output Requirements

```python
# GOOD: Frozen, deterministic
@dataclass(frozen=True)
class GoodOutput:
    value: str
    items: tuple[str, ...]  # Use tuple, not list

# BAD: Mutable, non-deterministic
@dataclass
class BadOutput:  # Not frozen!
    value: str
    items: list[str]  # Mutable!
    timestamp: float  # Non-deterministic!
```

### Deterministic Processing

```python
# BAD: Non-deterministic
@rule
async def bad_rule() -> Output:
    return Output(
        id=uuid.uuid4(),           # Random!
        time=datetime.now(),       # Changes!
        order=set(items),          # Unstable order!
    )

# GOOD: Deterministic
@rule
async def good_rule(input: Input) -> Output:
    return Output(
        id=hashlib.sha256(input.data).hexdigest(),  # Content-based
        items=tuple(sorted(items)),                  # Stable order
    )
```

### Environment Variables

```python
from pants.engine.environment import EnvironmentRequest, Environment

@rule
async def rule_with_env() -> Output:
    # Explicitly request env vars for cache key
    env = await Get(
        Environment,
        EnvironmentRequest(["MY_VAR", "OTHER_VAR"]),
    )

    my_var = env.get("MY_VAR", "default")
    return Output(value=my_var)
```

## Reducing Rule Calls

### Batch Processing

```python
# BAD: N separate rule calls
@goal_rule
async def bad_goal(targets: Targets) -> MyGoal:
    for target in targets:
        result = await Get(Output, Target, target)  # N calls!
    return MyGoal(exit_code=0)

# GOOD: Parallel batch processing
@goal_rule
async def good_goal(targets: Targets) -> MyGoal:
    results = await MultiGet(
        Get(Output, Target, t) for t in targets
    )  # Parallel!
    return MyGoal(exit_code=0)
```

### Avoid N+1 Patterns

```python
# BAD: N+1 pattern
@rule
async def bad_rule(targets: Targets) -> Output:
    all_deps = []
    for target in targets:
        deps = await Get(Dependencies, Target, target)  # N calls
        all_deps.extend(deps)
    return Output(deps=all_deps)

# GOOD: Single transitive request
@rule
async def good_rule(targets: Targets) -> Output:
    transitive = await Get(
        TransitiveTargets,
        TransitiveTargetsRequest([t.address for t in targets]),
    )  # Single call!
    return Output(deps=list(transitive.closure))
```

## Process Optimization

### Batch External Tool Calls

```python
# BAD: One process per file
@rule
async def bad_rule(files: list[str]) -> Output:
    results = []
    for f in files:
        result = await Get(
            ProcessResult,
            Process(argv=["tool", f], ...),
        )
        results.append(result)
    return Output(results=results)

# GOOD: Single process with all files
@rule
async def good_rule(files: list[str]) -> Output:
    result = await Get(
        ProcessResult,
        Process(
            argv=["tool", *files],  # All files at once
            ...
        ),
    )
    return Output(result=result)
```

### Minimize Process Input

```python
# BAD: Large input digest
@rule
async def bad_rule() -> Output:
    # Gets entire repo
    digest = await Get(Digest, PathGlobs(["**/*"]))
    process = Process(argv=["tool"], input_digest=digest, ...)
    return Output(...)

# GOOD: Minimal input digest
@rule
async def good_rule(fieldset: MyFieldSet) -> Output:
    # Only files we need
    digest = await Get(Digest, PathGlobs(fieldset.sources.value))
    process = Process(argv=["tool"], input_digest=digest, ...)
    return Output(...)
```

### Use Output Capture Wisely

```python
process = Process(
    argv=["tool"],
    input_digest=digest,
    # Only capture what you need
    output_files=("output.json",),  # Specific file
    # NOT output_directories=(".",)  # Everything!
)
```

## Memory Optimization

### Avoid Large In-Memory Data

```python
# BAD: Load everything into memory
@rule
async def bad_rule() -> Output:
    all_files = await Get(DigestContents, Digest, huge_digest)
    data = [f.content for f in all_files]  # All in memory!
    return Output(data=data)

# GOOD: Process incrementally or use Process
@rule
async def good_rule() -> Output:
    # Let external tool handle large data
    result = await Get(
        ProcessResult,
        Process(argv=["tool", "--process-files"], ...),
    )
    return Output(summary=result.stdout.decode())
```

### Use Generators Where Possible

```python
# In goal rules, yield results instead of collecting
@goal_rule
async def streaming_goal(
    console: Console,
    targets: Targets,
) -> MyGoal:
    for target in targets:
        result = await Get(Output, Target, target)
        console.print_stdout(result.summary)  # Print immediately
    return MyGoal(exit_code=0)
```

## Profiling

### Time Rule Execution

```python
import time

@rule
async def profiled_rule() -> Output:
    start = time.perf_counter()

    # Your logic
    result = await expensive_operation()

    elapsed = time.perf_counter() - start
    logger.info(f"Rule took {elapsed:.2f}s")

    return Output(result=result)
```

### Pants Built-in Profiling

```bash
# Profile goal execution
pants --stats-log my-goal ::

# Detailed timing
pants --time my-goal ::
```

## Remote Execution

### Ensure Rules are Remote-Compatible

```python
# Rules are remote-compatible if they:
# 1. Use Process for all external operations
# 2. Don't access local filesystem directly
# 3. Have deterministic outputs

@rule
async def remote_compatible_rule(input: Input) -> Output:
    # All file access through Digest
    digest = await Get(Digest, CreateDigest([...]))

    # All execution through Process
    result = await Get(
        ProcessResult,
        Process(
            argv=["tool"],
            input_digest=digest,
            description="Remote-safe operation",
        ),
    )

    return Output(output=result.stdout.decode())
```

### Avoid Local-Only Operations

```python
# BAD: Local filesystem access
@rule
async def bad_rule() -> Output:
    with open("/tmp/file.txt") as f:  # Local only!
        data = f.read()
    return Output(data=data)

# GOOD: Use Pants file system
@rule
async def good_rule() -> Output:
    digest = await Get(
        Digest,
        CreateDigest([FileContent("file.txt", b"data")]),
    )
    return Output(digest=digest)
```

## Summary: Performance Checklist

- [ ] Rule outputs are frozen dataclasses
- [ ] Outputs are deterministic (no timestamps, random values)
- [ ] Using `MultiGet` for parallel operations
- [ ] No N+1 query patterns
- [ ] External tools called with batched files
- [ ] Input digests are minimal
- [ ] No direct filesystem access
- [ ] Environment variables explicitly declared
- [ ] Large data processed externally or streamed
