# Pants Engine Architecture

Deep dive into how the Pants build system works internally.

## Hybrid Rust + Python Design

Pants combines:
- **Rust Core**: File watching, process execution, caching, rule graph execution
- **Python API**: Plugin development, rule definitions, target definitions

This design provides:
1. Performance of native code for I/O and execution
2. Ergonomics of Python for plugin authors
3. Type safety through Python type hints

## The Rule Graph

### Declarative Execution

Rules declare what they need and produce:

```python
@rule
async def compile(source: SourceFile) -> CompiledObject:
    deps = await Get(Dependencies, SourceFile, source)
    ...
```

The engine:
1. Builds a DAG from rule signatures
2. Determines execution order automatically
3. Parallelizes independent operations
4. Caches results by input hash

### Rule Resolution

When you request `Get(OutputType, InputType, input)`:
1. Engine finds rule producing `OutputType` from `InputType`
2. Checks cache for existing result
3. Executes rule if cache miss
4. Stores result for future requests

## Content-Addressable Storage (CAS)

Files are stored by content hash, not path:

```
SHA256(file_content) -> storage_location
```

Benefits:
- **Deduplication**: Identical files stored once
- **Immutability**: Content cannot change without changing hash
- **Cache Sharing**: Same content = same hash = cache hit
- **Remote Caching**: Hash-based retrieval from remote stores

## Sandboxed Execution

Processes run in isolated environments:

```
Process Request:
  - argv: ["compiler", "input.c", "-o", "output.o"]
  - input_digest: <hash of input files>
  - output_files: ["output.o"]

Execution:
  1. Create temp directory
  2. Materialize input_digest files
  3. Strip environment
  4. Execute process
  5. Capture declared outputs
  6. Return output_digest
```

This ensures:
- **Reproducibility**: Same inputs = same outputs
- **Hermeticity**: No undeclared dependencies
- **Remote Execution**: Can run anywhere with same results

## Caching Layers

### Local Cache (~/.cache/pants/lmdb_store)

```
Cache Key = hash(
    rule_name,
    input_types,
    input_values,  # Including file content hashes
    env_vars       # Only explicitly declared ones
)
```

### Remote Cache (Optional)

Configure in pants.toml:
```toml
[GLOBAL]
remote_cache_read = true
remote_cache_write = true
remote_store_address = "grpc://cache.example.com:9092"
```

### Cache Invalidation

Cache invalidates when:
- File content changes (hash changes)
- Rule inputs change
- Declared environment variables change
- Tool version changes

Cache does NOT invalidate for:
- File timestamps
- File permissions (usually)
- Undeclared environment variables

## Concurrency Model

### Automatic Parallelization

```python
# These execute in parallel
results = await MultiGet(
    Get(Output, Input, inp) for inp in inputs
)
```

The engine:
1. Identifies independent subgraphs
2. Executes them concurrently
3. Manages thread pool for processes
4. Handles I/O asynchronously

### Process Execution Limits

Control parallelism:
```toml
[GLOBAL]
process_execution_local_parallelism = 8
```

## Memory Management

### Streaming Results

Large rule graphs don't require all results in memory:
- Results are stored in CAS
- Only lightweight `Digest` references held
- Content loaded on demand

### Garbage Collection

Old cache entries cleaned based on:
- Age
- Total cache size
- Access patterns

Configure:
```toml
[GLOBAL]
local_store_gc_max_size_bytes = 10000000000  # 10GB
```

## Remote Execution

For large codebases, distribute work:

```toml
[GLOBAL]
remote_execution = true
remote_execution_address = "grpc://exec.example.com:9092"
```

Requirements for remote-compatible rules:
1. All file access through Digest
2. All execution through Process
3. Deterministic outputs
4. No local filesystem access

## Introspection

### Visualize Rule Graph

```bash
pants --engine-visualize-to=graph.html my-goal ::
```

### Profile Execution

```bash
pants --stats-log my-goal ::
pants --time my-goal ::
```

### Debug Logging

```bash
pants -ldebug my-goal ::
pants -ltrace my-goal ::  # Very verbose
```
