# Pants Plugin Development Gotchas (v2.30+)

Critical issues and their fixes when developing Pants plugins for version 2.30 and later.

---

## 1. Duplicate Scope Registration

**Problem:** Defining the same goal scope in multiple places causes a fatal error.

```
Scope `baseline-audit` claimed by ... was also claimed by ...
```

**Fix:** Ensure each `GoalSubsystem.name` is defined in exactly one place. Keep goals in `goals/` and rules in `rules/` - don't mix goal definitions into rule files.

```python
# WRONG - goal defined in rules.py
# rules/audit.py
class AuditGoal(Goal):
    subsystem_cls = AuditSubsystem  # Also defined in goals/audit.py!

# CORRECT - goals only in goals/
# goals/audit.py
class AuditGoal(Goal):
    subsystem_cls = AuditSubsystem

# rules/audit.py - only rules, no goal definitions
@rule
async def run_audit(...) -> AuditResult:
    ...
```

---

## 2. Scope Conflicts with Built-in Backends

**Problem:** Using common scope names like `ruff`, `ty`, `uv` conflicts with Pants' built-in backends.

```
Scope `ruff` claimed by pants_baseline.subsystems.ruff.RuffSubsystem,
was also claimed by pants.backend.python.lint.ruff.subsystem.Ruff
```

**Fix:** Namespace your subsystem scopes to avoid conflicts with current or future Pants backends.

```python
# WRONG
class RuffSubsystem(Subsystem):
    options_scope = "ruff"  # Conflicts with pants.backend.python.lint.ruff

# CORRECT
class RuffSubsystem(Subsystem):
    options_scope = "baseline-ruff"  # Namespaced to your plugin
```

Use prefixes like `baseline-ruff`, `baseline-ty`, `baseline-uv` instead of bare tool names.

---

## 3. Deprecated Get() Syntax - But Don't Use the New One Yet

**Problem:** The 3-argument `Get()` is deprecated and warns you to use dict syntax, but the dict syntax has type inference bugs.

```python
# Deprecated (but works)
result = await Get(FallibleProcessResult, Process, process)

# "New" syntax (broken - causes type inference errors)
result = await Get(FallibleProcessResult, {Process: process})
```

**Fix:** Stick with the deprecated 3-argument syntax for now until the dict syntax is fixed. Accept the deprecation warnings.

```python
# Use this (yes, it's deprecated, but it works)
result = await Get(FallibleProcessResult, Process, process)
digest = await Get(Digest, CreateDigest, create_digest)
```

---

## 4. Avoid `from __future__ import annotations` in Rule Files

**Problem:** PEP 563 deferred annotation evaluation stores types as strings, breaking Pants' runtime type inference.

```
Could not resolve type for `process` in module ...
```

**Fix:** Don't use `from __future__ import annotations` in files containing Pants rules. Use Python 3.10+ union syntax (`X | Y`) directly instead.

```python
# WRONG - breaks Pants type inference
from __future__ import annotations

@rule
async def my_rule(request: MyRequest) -> MyResult:
    ...

# CORRECT - use native syntax
@rule
async def my_rule(request: MyRequest) -> MyResult:
    ...

# For unions, use native Python 3.10+ syntax
def process(value: str | None) -> int | float:
    ...
```

---

## 5. Subsystems Don't Have a .rules() Method

**Problem:** Calling `Subsystem.rules()` in your `register.py` causes runtime errors.

```
cannot access local variable 'req' where it is not associated with a value
```

**Fix:** Don't call `.rules()` on Subsystem classes. Subsystems are auto-registered when used as parameters in rule functions.

```python
# WRONG
def rules():
    return [
        *collect_rules(my_rules),
        *MySubsystem.rules(),  # Subsystems don't have this!
    ]

# CORRECT
def rules():
    return [
        *collect_rules(my_rules),
        # Subsystems auto-register when used as rule parameters
    ]
```

Subsystems are automatically discovered and registered by the engine when they appear as parameters in your `@rule` functions.

---

## 6. Wheel Caching by Filename

**Problem:** When testing locally, Pants caches wheels by filename. If you rebuild without changing the version, you get the old cached version.

**Fix:** Bump the version number every time you rebuild for local testing, or clear the entire Pants cache:

```bash
# Option 1: Bump version in pyproject.toml before each rebuild
# version = "0.1.1" -> "0.1.2" -> "0.1.3"

# Option 2: Clear Pants cache
rm -rf ~/.cache/pants

# Option 3: Use a dev version with timestamp
# version = "0.1.0.dev20240115"
```

---

## Summary Checklist for Pants 2.30+ Plugins

Before publishing or testing your plugin:

- [ ] **Namespace all subsystem scopes** to avoid conflicts (e.g., `myplugin-ruff` not `ruff`)
- [ ] **Define each goal scope in exactly one place** (goals in `goals/`, rules in `rules/`)
- [ ] **Use 3-argument Get() syntax** (not dict syntax) - accept the deprecation warnings
- [ ] **Don't use `from __future__ import annotations`** in rule files
- [ ] **Don't call `.rules()` on Subsystem classes** - they auto-register
- [ ] **Bump version for each local test iteration** or clear `~/.cache/pants`
- [ ] **Test with `pants goals`** before publishing to verify registration

---

## Quick Diagnostic Commands

```bash
# Verify your plugin loads correctly
pants goals

# Check for scope conflicts
pants help-all 2>&1 | grep -i "scope.*claimed"

# View registered backends
pants backends

# Clear cache for fresh testing
rm -rf ~/.cache/pants
```
