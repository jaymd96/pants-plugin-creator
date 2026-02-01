# Linter Plugin Pattern

Complete pattern for creating a Pants plugin that integrates an external linter.

## Overview

A linter plugin:
1. Defines targets with source files
2. Runs an external linting tool via Process
3. Reports lint results to the user
4. Provides a goal for users to invoke

## Complete Implementation

### targets.py

```python
"""Target types for the linter plugin."""

from pants.engine.target import (
    COMMON_TARGET_FIELDS,
    MultipleSourcesField,
    Target,
)


class LinterSourcesField(MultipleSourcesField):
    """Source files to lint."""

    default = ("*.sh",)  # Default file patterns
    help = "Shell script files to lint with shellcheck"


class ShellcheckTarget(Target):
    """A target for shell scripts to be linted.

    Example:
        shellcheck_sources(
            name="scripts",
            sources=["*.sh"],
        )
    """

    alias = "shellcheck_sources"
    help = "Shell script sources to lint with shellcheck"

    core_fields = (
        *COMMON_TARGET_FIELDS,
        LinterSourcesField,
    )
```

### subsystem.py

```python
"""Configuration for the linter."""

from pants.option.subsystem import Subsystem
from pants.option.option_types import BoolOption, StrOption, StrListOption


class ShellcheckSubsystem(Subsystem):
    """Configuration for shellcheck linter."""

    options_scope = "shellcheck"
    help = "The shellcheck linter for shell scripts"

    skip = BoolOption(
        default=False,
        help="Skip running shellcheck",
    )

    version = StrOption(
        default="0.9.0",
        help="Version of shellcheck to use",
    )

    args = StrListOption(
        help="Extra arguments to pass to shellcheck",
    )
```

### rules.py

```python
"""Rules for running the linter."""

from dataclasses import dataclass
from typing import Sequence

from pants.engine.rules import collect_rules, rule, Get
from pants.engine.target import FieldSet, Target
from pants.engine.process import Process, ProcessResult
from pants.engine.fs import Digest, MergeDigests, PathGlobs

from my_linter.targets import LinterSourcesField
from my_linter.subsystem import ShellcheckSubsystem


@dataclass(frozen=True)
class ShellcheckFieldSet(FieldSet):
    """FieldSet for targets that can be linted with shellcheck."""

    required_fields = (LinterSourcesField,)

    sources: LinterSourcesField


@dataclass(frozen=True)
class LintResult:
    """Result of running the linter on a target."""

    exit_code: int
    stdout: str
    stderr: str
    target_name: str


@rule
async def run_shellcheck(
    fieldset: ShellcheckFieldSet,
    subsystem: ShellcheckSubsystem,
) -> LintResult:
    """Run shellcheck on a target's source files."""

    if subsystem.skip:
        return LintResult(
            exit_code=0,
            stdout="Skipped",
            stderr="",
            target_name=str(fieldset.address),
        )

    # Get source files as a Digest
    sources_digest = await Get(
        Digest,
        PathGlobs(fieldset.sources.value),
    )

    # Build the command
    argv = ["shellcheck", *subsystem.args, *fieldset.sources.value]

    # Create the process
    process = Process(
        argv=argv,
        input_digest=sources_digest,
        description=f"Lint {fieldset.address} with shellcheck",
    )

    # Run the process
    result = await Get(ProcessResult, Process, process)

    return LintResult(
        exit_code=result.exit_code,
        stdout=result.stdout.decode("utf-8"),
        stderr=result.stderr.decode("utf-8"),
        target_name=str(fieldset.address),
    )


def rules() -> Sequence:
    return collect_rules()
```

### goals.py

```python
"""Goal for running the linter."""

from typing import Sequence

from pants.engine.goal import Goal, GoalSubsystem, goal_rule
from pants.engine.console import Console
from pants.engine.rules import collect_rules, Get, MultiGet
from pants.engine.target import Targets

from my_linter.rules import ShellcheckFieldSet, LintResult, run_shellcheck
from my_linter.subsystem import ShellcheckSubsystem


class ShellcheckGoalSubsystem(GoalSubsystem):
    """Options for the shellcheck goal."""

    name = "shellcheck"
    help = "Run shellcheck on shell script files"


class ShellcheckGoal(Goal):
    """Goal to run shellcheck linter."""

    subsystem_cls = ShellcheckGoalSubsystem


@goal_rule
async def run_shellcheck_goal(
    console: Console,
    targets: Targets,
    subsystem: ShellcheckSubsystem,
) -> ShellcheckGoal:
    """Run shellcheck on all applicable targets."""

    if subsystem.skip:
        console.print_stdout("Shellcheck skipped")
        return ShellcheckGoal(exit_code=0)

    # Filter to targets with the required fields
    fieldsets = [
        ShellcheckFieldSet.create(t)
        for t in targets
        if ShellcheckFieldSet.is_applicable(t)
    ]

    if not fieldsets:
        console.print_stdout("No shell script targets found")
        return ShellcheckGoal(exit_code=0)

    # Run linter on all targets in parallel
    results = await MultiGet(
        Get(LintResult, ShellcheckFieldSet, fs)
        for fs in fieldsets
    )

    # Report results
    exit_code = 0
    for result in results:
        if result.exit_code == 0:
            console.print_stdout(f"[OK] {result.target_name}")
        else:
            console.print_stderr(f"[FAIL] {result.target_name}")
            console.print_stderr(result.stdout)
            console.print_stderr(result.stderr)
            exit_code = 1

    # Summary
    passed = sum(1 for r in results if r.exit_code == 0)
    failed = len(results) - passed
    console.print_stdout(f"\nResults: {passed} passed, {failed} failed")

    return ShellcheckGoal(exit_code=exit_code)


def rules() -> Sequence:
    return collect_rules()
```

### register.py

```python
"""Register the linter plugin."""

from typing import Sequence

from pants.engine.rules import collect_rules

from my_linter import rules as lint_rules
from my_linter import goals as lint_goals
from my_linter.subsystem import ShellcheckSubsystem
from my_linter.targets import ShellcheckTarget


def rules() -> Sequence:
    return [
        *collect_rules(lint_rules),
        *collect_rules(lint_goals),
        *ShellcheckSubsystem.rules(),
    ]


def target_types() -> Sequence:
    return [ShellcheckTarget]
```

## Usage

### BUILD file

```python
shellcheck_sources(
    name="scripts",
    sources=["*.sh"],
)
```

### pants.toml

```toml
[GLOBAL]
backend_packages = ["my_linter"]

[shellcheck]
args = ["-x", "-e", "SC1091"]
```

### Run the linter

```bash
pants shellcheck src::
```

## Key Points

1. **FieldSet** filters targets to those with required fields
2. **Process** runs the external tool hermetically
3. **MultiGet** runs linting in parallel for all targets
4. **LintResult** is a frozen dataclass for caching
5. **Subsystem** provides configuration options

## Testing

```python
from pants.testutil.rule_runner import RuleRunner, QueryRule

def test_shellcheck(rule_runner):
    rule_runner.write_files({
        "BUILD": 'shellcheck_sources(name="test", sources=["test.sh"])',
        "test.sh": "#!/bin/bash\necho 'hello'",
    })

    result = rule_runner.run_goal(ShellcheckGoal, [":test"])
    assert result.exit_code == 0
```
