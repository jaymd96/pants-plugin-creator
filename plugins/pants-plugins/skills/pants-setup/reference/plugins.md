# Third-Party Plugin Installation

Guide to discovering, installing, and managing third-party Pants plugins.

## Installing Plugins

Add plugins to the `plugins` list in pants.toml:

```toml
[GLOBAL]
plugins = [
    "pants-python-baseline==0.1.0",
    "another-plugin>=1.0,<2.0",
]
```

## Version Pinning

**Always pin plugin versions** to avoid unexpected breakage:

```toml
# GOOD - Pinned version
plugins = ["pants-python-baseline==0.1.0"]

# OK - Version range
plugins = ["pants-python-baseline>=0.1.0,<1.0"]

# BAD - No version (will use latest)
plugins = ["pants-python-baseline"]
```

## Notable Plugins

### Python Quality

| Plugin | Description | Install |
|--------|-------------|---------|
| `pants-python-baseline` | Astral-based quality baseline (Ruff, ty, uv) | `pants-python-baseline==0.1.0` |

### Finding More Plugins

1. **PyPI Search**:
   ```bash
   pip index versions pants-  # List pants-related packages
   ```

2. **GitHub Topics**:
   - https://github.com/topics/pants-plugin
   - https://github.com/topics/pants-build

3. **Pants Community**:
   - Pants Slack: https://pantsbuild.slack.com
   - Pants GitHub Discussions

## Creating Your Own Plugin

See the `create-pants-plugin` skill:
```
/pants-plugins:create-pants-plugin
```

## Plugin Compatibility

Plugins must be compatible with your Pants version. Check:

1. Plugin documentation for supported versions
2. Plugin's `setup.py` or `pyproject.toml` for dependencies
3. Changelog for breaking changes

## Upgrading Plugins

1. Check for new versions:
   ```bash
   pip index versions pants-python-baseline
   ```

2. Update pants.toml:
   ```toml
   plugins = ["pants-python-baseline==0.2.0"]  # Updated
   ```

3. Test thoroughly:
   ```bash
   pants lint ::
   pants test ::
   ```

## Removing Plugins

1. Remove from `plugins` list
2. Remove related `backend_packages` entries
3. Remove configuration sections
4. Update BUILD files if they use plugin targets

```toml
# Before
[GLOBAL]
plugins = ["pants-python-baseline==0.1.0"]
backend_packages = ["python_baseline"]

[python-baseline]
enabled = true

# After
[GLOBAL]
plugins = []
backend_packages = []

# Remove [python-baseline] section entirely
```

## Troubleshooting

### Plugin Not Found

```
ERROR: Could not find pants-xyz
```

Check:
- Plugin name spelling
- Plugin is published to PyPI
- Network connectivity

### Version Conflict

```
ERROR: pants-xyz requires pants>=2.19, but pants 2.18 is installed
```

Either:
- Upgrade Pants version
- Use an older plugin version
- Check if plugin has a compatible release

### Import Error

```
ERROR: No module named 'xyz'
```

Check:
- Plugin is in `plugins` list
- Backend package is in `backend_packages`
- Restart Pants daemon: `pants kill`
