---
name: create-pants-plugin
description: Create production-grade Pants build system plugins. Use when the user wants to create a Pants plugin, scaffold a Pants backend, or build custom build system functionality for Pants.
user-invocable: true
allowed-tools: Write, Edit, Read, Bash, Glob, Grep
---

# Pants Plugin Creator

Guide users through creating production-grade Pants build system plugins following official best practices.

## What This Skill Does

1. Gathers requirements through focused questions
2. Generates complete plugin structure using native file tools
3. Customizes targets, rules, and goals for specific use cases
4. Provides testing templates and guidance
5. Explains development and publishing workflow

## Workflow

### Phase 1: Requirements Gathering

Ask these questions (adapt based on responses):

**1. Plugin name**: What should your plugin be called?
- Must be lowercase with hyphens (e.g., `my-linter-plugin`)
- Will create package name with underscores (e.g., `my_linter_plugin`)

**2. Purpose**: What does your plugin do?
- Lint/format code (integrates with external linter)
- Generate code from definitions (protobuf, thrift, etc.)
- Build/compile custom assets
- Integrate with external tools
- Other custom processing

**3. Author info**: Name and email for pyproject.toml

**4. Organization**: GitHub org/username for repository URLs

**5. Custom targets**: Does your plugin need custom target types?
- If yes, ask what metadata users will specify in BUILD files

**6. Configuration**: What options should users configure in pants.toml?

### Phase 2: Generate Plugin Structure

Create the following directory structure:

```
{plugin-name}/
├── pyproject.toml
├── README.md
├── LICENSE
├── Makefile
├── .gitignore
├── src/{package}/
│   ├── __init__.py
│   ├── version.py
│   ├── register.py      # CRITICAL: Entry point
│   ├── subsystem.py     # Configuration
│   ├── targets.py       # Target definitions
│   ├── rules.py         # Business logic
│   └── goals.py         # User commands
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── __init__.py
│   │   └── test_subsystem.py
│   └── integration/
│       └── __init__.py
└── docs/
    └── index.md
```

### Phase 3: File Generation Templates

Use the templates from `reference/templates.md` for each file. Key files:

#### pyproject.toml
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "{plugin-name}"
version = "0.1.0"
description = "{description}"
requires-python = ">=3.11,<4"
authors = [{name = "{author}", email = "{email}"}]

[tool.hatch.build.targets.wheel]
packages = ["src/{package}"]

[tool.hatch.envs.default.scripts]
test = "pytest {args:tests}"
lint = ["black --check src tests", "mypy src"]
fmt = ["black src tests", "isort src tests"]
```

#### register.py (REQUIRED - Pants loads this)
```python
from pants.engine.rules import collect_rules
from {package} import rules as plugin_rules
from {package}.subsystem import PluginSubsystem
from {package}.targets import CustomTarget

def rules():
    return [*collect_rules(plugin_rules), *PluginSubsystem.rules()]

def target_types():
    return [CustomTarget]
```

#### subsystem.py (Configuration)
```python
from pants.option.subsystem import Subsystem
from pants.option.option_types import BoolOption, StrOption

class PluginSubsystem(Subsystem):
    options_scope = "{scope}"
    help = "Options for {plugin-name}"

    enabled = BoolOption(default=True, help="Whether to enable this plugin.")
```

#### targets.py (What data you need)
```python
from pants.engine.target import Target, COMMON_TARGET_FIELDS, StringSequenceField

class SourcesField(StringSequenceField):
    alias = "sources"
    help = "Source files for this target"
    default = ()

class CustomTarget(Target):
    alias = "custom_target"
    help = "A custom target type"
    core_fields = (*COMMON_TARGET_FIELDS, SourcesField)
```

#### rules.py (How you process data)
```python
from dataclasses import dataclass
from pants.engine.rules import rule, collect_rules

@dataclass(frozen=True)
class ProcessedOutput:
    target_name: str
    exit_code: int = 0

@rule
async def process_target(wrapped: WrappedTarget) -> ProcessedOutput:
    target = wrapped.target
    return ProcessedOutput(target_name=target.name, exit_code=0)

def rules():
    return collect_rules()
```

#### goals.py (User-facing commands)
```python
from pants.engine.goal import Goal, GoalSubsystem, goal_rule
from pants.engine.console import Console

class PluginGoal(Goal):
    subsystem_cls = PluginGoalSubsystem

@goal_rule
async def run_plugin(console: Console, targets: Targets) -> PluginGoal:
    console.print_stdout(f"Processing {len(targets)} targets")
    return PluginGoal(exit_code=0)
```

### Phase 4: Customization Based on Plugin Type

#### For Linter Plugins
Read `reference/patterns/linter.md` for complete pattern:
- Define FieldSet with SourcesField
- Use Process to invoke external linter tool
- Return LintResult with exit_code and output

#### For Codegen Plugins
Read `reference/patterns/codegen.md` for complete pattern:
- Define input target with definition field
- Use CreateDigest for output files
- Return CodegenResult with Digest

#### For Custom Target Plugins
Read `reference/patterns/custom-target.md` for complete pattern:
- Define Field classes for each metadata piece
- Create Target with core_fields tuple
- Register in target_types()

### Phase 5: Testing Guidance

Generate test files using RuleRunner:

```python
from pants.testutil.rule_runner import RuleRunner, QueryRule

@pytest.fixture
def rule_runner():
    from {package}.register import rules, target_types
    return RuleRunner(rules=rules(), target_types=target_types())

def test_process_target(rule_runner):
    rule_runner.write_files({
        "BUILD": 'custom_target(name="test", sources=["*.txt"])',
        "file.txt": "content",
    })
    # Test your rules here
```

### Phase 6: Development Workflow

After generating the plugin, instruct the user:

```bash
cd {plugin-name}
hatch env create          # Create dev environment
hatch run test            # Run tests
hatch run fmt             # Format code
hatch run lint            # Check code quality
```

### Phase 7: Publishing Options

**Option A: Publish to PyPI**
```bash
hatch build
hatch publish
```

Users install with:
```toml
[GLOBAL]
plugins = ["{plugin-name}==1.0.0"]
backend_packages = ["{package}"]
```

**Option B: In-repo plugin**
```toml
[GLOBAL]
pythonpath = ["%(buildroot)s/pants-plugins/{plugin-name}/src"]
backend_packages = ["{package}"]
```

## Critical Reminders

When generating plugin code, always ensure:

1. **Rules must be pure** - No side effects, no print(), no direct file I/O
2. **Use frozen dataclasses** - All rule outputs: `@dataclass(frozen=True)`
3. **Type hints required** - All parameters and returns must be typed
4. **Use Get() for dependencies** - Never call rules directly
5. **Deterministic outputs** - Same inputs must produce same outputs
6. **Use Process for subprocesses** - Hermetic execution via the engine

## Reference Documentation

For detailed information, read these supporting files:

- `reference/quickstart.md` - 5-minute intro to Pants plugins
- `reference/api-reference.md` - Complete API documentation (Target API, Rules API)
- `reference/developer-guide.md` - Best practices, caching, common pitfalls
- `reference/templates.md` - Complete file templates with full content
- `reference/patterns/linter.md` - Complete linter plugin pattern
- `reference/patterns/codegen.md` - Complete code generation pattern
- `reference/patterns/custom-target.md` - Custom target type pattern

## Example Session

User: "I want to create a Pants plugin that runs shellcheck on shell scripts"

Response:
1. Ask for plugin name, author, org
2. Generate plugin structure with:
   - Target: `shellcheck_sources` with `sources` field
   - Rule: Runs shellcheck process on source files
   - Goal: `shellcheck` command for users
   - Subsystem: Configuration for shellcheck version, args
3. Generate tests
4. Provide development instructions
