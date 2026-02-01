# Pants Plugin Developer Guide

Best practices for writing production-grade Pants plugins.

## Core Architecture

### The Pants Engine

Pants is a Python-in-Rust execution engine that:

1. **Compiles** plugin code at startup
2. **Builds rule graph** from `@rule` definitions
3. **Executes rules lazily** (only when needed)
4. **Caches results** automatically (content-addressed)
5. **Parallelizes** execution automatically
6. **Supports remote execution** natively

### Key Differences from Other Build Systems

**Pants** (Rules Engine):
- Pure function composition
- Plugins must be pure (no side effects)
- Lazy evaluation of rules
- Immutable data (frozen dataclasses)
- Automatic caching & remote execution

**Gradle/Maven** (Task-based):
- Task-based execution model
- Plugins can have side effects
- Eager configuration
- Mutable domain objects

## Rules Engine Lifecycle

### Phase 1: Plugin Discovery (Startup)

```
pants.toml loaded
       ↓
[GLOBAL] backend_packages read
       ↓
Python imports your register.py
       ↓
rules() and target_types() called
       ↓
Rule graph compiled and validated
```

**Best Practice**: Make `rules()` fast. No I/O or expensive computation.

```python
# GOOD: Just collect the rules
def rules():
    return collect_rules()

# BAD: Expensive operations
def rules():
    return [*fetch_from_network(), *load_config()]  # NO!
```

### Phase 2: Target Discovery

When Pants runs:
1. Walks source directories for BUILD files
2. Parses BUILD files and creates Target objects
3. Validates against registered target types
4. Builds dependency graph

### Phase 3: Goal Execution

When user runs `pants my-goal`:
1. Engine finds `@goal_rule` for that goal
2. Determines what inputs it needs
3. Walks rule graph backwards
4. Executes rules in dependency order
5. Caches results

## Plugin Structure Best Practices

### Directory Organization

```
my-plugin/
├── src/my_plugin/
│   ├── __init__.py      # Package marker
│   ├── version.py       # Version string
│   ├── register.py      # Entry point (REQUIRED)
│   ├── subsystem.py     # Configuration
│   ├── targets.py       # Target types
│   ├── rules.py         # Business logic
│   └── goals.py         # User commands
├── tests/
│   ├── conftest.py      # Fixtures
│   └── unit/
└── pyproject.toml       # Hatch config
```

### Module Patterns

**register.py** - Always simple, just collects:
```python
def rules():
    return [*collect_rules(rules_module), *Subsystem.rules()]

def target_types():
    return [MyTarget]
```

**subsystem.py** - Configuration only:
```python
class MySubsystem(Subsystem):
    options_scope = "my-plugin"
    enabled = BoolOption(default=True)
```

**targets.py** - Data model only:
```python
class MyTarget(Target):
    alias = "my_target"
    core_fields = (*COMMON_TARGET_FIELDS, SourcesField)
```

**rules.py** - Pure business logic:
```python
@rule
async def process(target: Target) -> Output:
    # Pure logic only
    return Output(...)
```

## Performance & Caching

### Cache Keys

Rule outputs are cached based on:
- Input types and values
- File content hashes
- Environment variables (if declared)

### Determinism Requirements

For caching to work, rules must be deterministic:

```python
# GOOD: Deterministic
@rule
async def good_rule(target: Target) -> Output:
    return Output(name=target.name)

# BAD: Non-deterministic
@rule
async def bad_rule(target: Target) -> Output:
    return Output(timestamp=datetime.now())  # NO!
```

### Remote Execution

Rules automatically support remote execution if they:
- Use `Process` for subprocesses (hermetic)
- Don't access local state
- Have deterministic outputs

## Common Pitfalls

### 1. Missing Type Hints

```python
# BAD: Will fail at startup
@rule
async def my_rule(target):  # Missing type hints
    return Output()

# GOOD: Type hints required
@rule
async def my_rule(target: Target) -> Output:
    return Output()
```

### 2. Calling Rules Directly

```python
# BAD: Never call rules directly
@rule
async def bad_rule():
    result = other_rule()  # NO!

# GOOD: Use Get()
@rule
async def good_rule():
    result = await Get(Output, Input, input_instance)
```

### 3. Side Effects in Rules

```python
# BAD: Side effects
@rule
async def bad_rule():
    print("Processing...")  # NO!
    with open("file.txt") as f:  # NO!
        data = f.read()

# GOOD: Use Console and Process
@goal_rule
async def good_rule(console: Console):
    console.print_stdout("Processing...")
    result = await Get(ProcessResult, Process, process)
```

### 4. Non-Frozen Outputs

```python
# BAD: Mutable output
@dataclass
class BadOutput:  # Missing frozen=True
    value: str

# GOOD: Frozen dataclass
@dataclass(frozen=True)
class GoodOutput:
    value: str
```

### 5. Global State

```python
# BAD: Global state
_cache = {}  # NO!

@rule
async def bad_rule():
    _cache["key"] = "value"  # NO!

# GOOD: Use subsystems for configuration
@rule
async def good_rule(subsystem: MySubsystem):
    value = subsystem.option
```

## Testing & Quality

### RuleRunner Usage

```python
from pants.testutil.rule_runner import RuleRunner, QueryRule

@pytest.fixture
def rule_runner():
    return RuleRunner(
        rules=[*my_rules(), QueryRule(Output, [Input])],
        target_types=[MyTarget],
    )

def test_rule(rule_runner):
    rule_runner.write_files({
        "BUILD": "my_target(name='test')",
        "file.txt": "content",
    })
    result = rule_runner.request(Output, [Input(...)])
    assert result.exit_code == 0
```

### Integration Testing

```python
def test_goal(rule_runner):
    rule_runner.write_files({"BUILD": "my_target(name='test')"})
    result = rule_runner.run_goal(MyGoal, [":test"])
    assert result.exit_code == 0
```

## The 10 Commandments of Pants Plugins

1. **Rules shall be pure** - No side effects
2. **Outputs shall be frozen** - Immutable dataclasses
3. **Type hints shall be complete** - Every parameter and return
4. **Get() shall be thy interface** - Never call rules directly
5. **Process shall execute commands** - Hermetic subprocess execution
6. **Subsystems shall hold configuration** - Not global state
7. **Targets shall be immutable** - Metadata only
8. **Tests shall use RuleRunner** - Isolated testing
9. **register.py shall be simple** - Just collect rules
10. **Determinism shall be maintained** - Same inputs = same outputs

## Production Checklist

Before publishing:

- [ ] All tests pass: `hatch run test`
- [ ] Code formatted: `hatch run fmt`
- [ ] No lint issues: `hatch run lint`
- [ ] Type hints complete
- [ ] Frozen dataclasses for outputs
- [ ] No side effects in rules
- [ ] Subsystem options documented
- [ ] README.md complete
- [ ] Version updated

## Resources

- [Official Docs](https://www.pantsbuild.org/stable/docs/writing-plugins/overview)
- [Plugin Upgrade Guide](https://www.pantsbuild.org/stable/docs/writing-plugins/common-plugin-tasks/plugin-upgrade-guide)
- [Pants GitHub](https://github.com/pantsbuild/pants)
