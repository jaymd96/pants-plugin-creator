# Searching for Pants Plugins

Commands and methods to discover available Pants plugins.

## Search GitHub (jaymd96 Plugins)

### Using gh CLI

```bash
# List all pants plugins by jaymd96
gh repo list jaymd96 --topic pants-plugin --json name,description,url

# Search for specific plugin
gh search repos "pants-" --owner jaymd96 --topic pants-plugin

# Get details of a specific plugin
gh repo view jaymd96/pants-baseline

# List with more details
gh repo list jaymd96 --topic pants-plugin --json name,description,url,stargazersCount --jq '.[] | "\(.name): \(.description)"'
```

### Using curl/API

```bash
# Search GitHub API for pants plugins
curl -s "https://api.github.com/users/jaymd96/repos" | \
  jq '.[] | select(.name | startswith("pants-")) | {name, description, html_url}'

# Search by topic
curl -s "https://api.github.com/search/repositories?q=topic:pants-plugin+user:jaymd96" | \
  jq '.items[] | {name, description, html_url}'
```

### Quick One-Liner

```bash
# List all jaymd96 pants plugins (names only)
gh repo list jaymd96 --topic pants-plugin --json name --jq '.[].name'
```

## Search PyPI (jaymd96 Packages)

### Using pip

```bash
# Search for jaymd96 pants packages (if pip search works)
pip index versions jaymd96-pants-baseline

# List installed pants plugins
pip list | grep jaymd96-pants
```

### Using curl/API

```bash
# Check if a specific package exists
curl -s "https://pypi.org/pypi/jaymd96-pants-baseline/json" | jq '.info | {name, version, summary}'

# Get all versions
curl -s "https://pypi.org/pypi/jaymd96-pants-baseline/json" | jq '.releases | keys'
```

### Using uv (faster)

```bash
# Check package info
uv pip show jaymd96-pants-baseline

# Search PyPI (if available)
uv search jaymd96-pants
```

## Combined Search Script

Save this as `search-pants-plugins.sh`:

```bash
#!/bin/bash
# Search for available jaymd96 Pants plugins

echo "=== GitHub Repositories ==="
gh repo list jaymd96 --topic pants-plugin --json name,description --jq '.[] | "  \(.name): \(.description)"' 2>/dev/null || \
  echo "  (gh CLI not available, install with: brew install gh)"

echo ""
echo "=== Known PyPI Packages ==="
# List of known packages to check
packages=(
  "jaymd96-pants-baseline"
  # Add more as they're published
)

for pkg in "${packages[@]}"; do
  version=$(curl -s "https://pypi.org/pypi/$pkg/json" 2>/dev/null | jq -r '.info.version' 2>/dev/null)
  if [ "$version" != "null" ] && [ -n "$version" ]; then
    echo "  $pkg ($version)"
  fi
done

echo ""
echo "=== Installation ==="
echo "  pip install jaymd96-pants-{plugin-name}"
echo "  # or"
echo "  uv pip install jaymd96-pants-{plugin-name}"
```

## Available Plugins (Known)

| Plugin | GitHub | PyPI | Description |
|--------|--------|------|-------------|
| baseline | [pants-baseline](https://github.com/jaymd96/pants-baseline) | `jaymd96-pants-baseline` | Python quality baseline (Ruff, ty, uv, pytest) |

## Installing a Plugin

Once you find a plugin:

### 1. Add to pants.toml

```toml
[GLOBAL]
plugins = [
    "jaymd96-pants-baseline==0.1.0",
]
backend_packages = [
    "pants_baseline",
]
```

### 2. Or install directly for testing

```bash
# Using pip
pip install jaymd96-pants-baseline

# Using uv (faster)
uv pip install jaymd96-pants-baseline
```

## Checking Plugin Compatibility

Before installing, check compatibility:

```bash
# Check Python version requirements
curl -s "https://pypi.org/pypi/jaymd96-pants-baseline/json" | \
  jq '.info.requires_python'

# Check dependencies
curl -s "https://pypi.org/pypi/jaymd96-pants-baseline/json" | \
  jq '.info.requires_dist'
```

## Contributing a Plugin

To add your plugin to the ecosystem:

1. Follow [naming conventions](naming-conventions.md)
2. Add `pants-plugin` label to GitHub repo
3. Publish to PyPI with `jaymd96-pants-` prefix
4. Submit PR to add to the "Available Plugins" list

## Troubleshooting

### Plugin Not Found

```bash
# Check if package exists on PyPI
curl -I "https://pypi.org/pypi/jaymd96-pants-{name}/json"
# 200 = exists, 404 = not found

# Check GitHub repo
gh repo view jaymd96/pants-{name}
```

### Version Issues

```bash
# List all available versions
curl -s "https://pypi.org/pypi/jaymd96-pants-baseline/json" | \
  jq '.releases | keys | reverse | .[:5]'
```
