# Pants Plugin Quick Start

Get your first Pants plugin running in 5 minutes.

## What is a Pants Plugin?

A Pants plugin extends the Pants build system with:
- **Custom target types** - Define new BUILD file syntax for your domain
- **Rules** - Pure async functions that process targets
- **Goals** - User-facing commands (`pants my-goal`)
- **Subsystems** - Configuration options in pants.toml

## Quick Start (5 minutes)

### 1. Generate Plugin Structure

Use the skill to generate a complete plugin, or manually create:

```bash
mkdir my-plugin && cd my-plugin
mkdir -p src/my_plugin tests/unit docs
```

### 2. Create pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-plugin"
version = "0.1.0"
requires-python = ">=3.11,<4"

[tool.hatch.build.targets.wheel]
packages = ["src/my_plugin"]

[tool.hatch.envs.default.scripts]
test = "pytest tests"
```

### 3. Create Entry Point (register.py)

```python
# src/my_plugin/register.py
from pants.engine.rules import collect_rules
from my_plugin import rules

def rules():
    return collect_rules(rules)

def target_types():
    return []
```

### 4. Set Up Development

```bash
hatch env create
hatch run test
```

### 5. Test in Another Repo

```toml
# pants.toml in test repo
[GLOBAL]
pythonpath = ["/path/to/my-plugin/src"]
backend_packages = ["my_plugin"]
```

## Architecture Overview

```
┌─────────────────────────────────────────┐
│        Pants Engine (Rust)              │
│  (Scheduling, caching, parallelism)     │
└──────────────────┬──────────────────────┘
                   │ loads
                   ↓
┌─────────────────────────────────────────┐
│      Your Plugin (Python Package)       │
├─────────────────────────────────────────┤
│ register.py                             │
│ ├── rules()        → Your @rules        │
│ ├── target_types() → Your Target types  │
│ └── subsystem_cls  → Your config        │
├─────────────────────────────────────────┤
│ targets.py        → Data model (what)   │
│ rules.py          → Logic (how)         │
│ goals.py          → Interface (why)     │
│ subsystem.py      → Configuration       │
└─────────────────────────────────────────┘
```

## Key Concepts

### Target
Metadata about code in BUILD files:
```python
my_target(name="utils", sources=["*.py"], timeout=30)
```

### Field
Single piece of target metadata:
```python
class TimeoutField(IntField):
    alias = "timeout"
    default = 60
```

### Rule
Pure async function: Input → Output
```python
@rule
async def process(target: Target) -> Result:
    return Result(...)
```

### Goal
User-facing command:
```python
@goal_rule
async def my_goal(console: Console) -> MyGoal:
    console.print_stdout("Hello!")
    return MyGoal(exit_code=0)
```

### Subsystem
Configuration in pants.toml:
```python
class MySubsystem(Subsystem):
    options_scope = "my-plugin"
    timeout = IntOption(default=30)
```

## File Structure

```
my-plugin/
├── pyproject.toml           # Hatch config
├── src/my_plugin/
│   ├── __init__.py
│   ├── version.py
│   ├── register.py          # Entry point (REQUIRED)
│   ├── targets.py           # Target definitions
│   ├── rules.py             # Business logic
│   ├── goals.py             # User commands
│   └── subsystem.py         # Configuration
├── tests/
│   ├── conftest.py
│   └── unit/
└── README.md
```

## Common Workflows

### Add a Linter
1. Define target with sources field
2. Create rule that runs linter process
3. Create goal to invoke linter

### Generate Code
1. Define target with input definition field
2. Create rule that runs codegen tool
3. Hook output into build graph

### Custom Build Step
1. Define target with required metadata
2. Create rule that processes the target
3. Create goal for user invocation

## Testing

Use RuleRunner for isolated testing:

```python
from pants.testutil.rule_runner import RuleRunner

def test_my_rule():
    runner = RuleRunner(rules=my_rules(), target_types=my_targets())
    runner.write_files({"BUILD": "my_target(name='test')"})
    result = runner.request(Output, [input])
    assert result.exit_code == 0
```

## Publishing

### To PyPI
```bash
hatch build
hatch publish
```

### In-Repo
```toml
[GLOBAL]
pythonpath = ["%(buildroot)s/plugins/my-plugin/src"]
backend_packages = ["my_plugin"]
```

## Debugging

```bash
# View targets
pants peek src::

# Debug logging
pants my-goal --debug-log-level=debug

# View rule graph
pants peek --graph-representation=dot
```

## Important Facts

1. **Plugin API is NOT stable** - May change between minor versions
2. **Rules must be pure** - No side effects, no print()
3. **Outputs must be frozen** - Use `@dataclass(frozen=True)`
4. **Use Get() for dependencies** - Never call rules directly
5. **Automatic caching** - Deterministic outputs enable caching

## Resources

- [Official Plugin Docs](https://www.pantsbuild.org/stable/docs/writing-plugins/overview)
- [Rules API](https://www.pantsbuild.org/stable/docs/writing-plugins/the-rules-api/concepts)
- [Target API](https://www.pantsbuild.org/stable/docs/writing-plugins/the-target-api/concepts)
- [Pants Slack](https://pantsbuild.slack.com)
