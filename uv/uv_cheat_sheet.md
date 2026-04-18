# UV Quick Reference Cheat Sheet

## Setup & Initialization

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create single package
uv init my-project
uv init --lib my-library

# Create workspace
uv init --workspace
mkdir packages
uv init --lib packages/package-a
uv init --lib packages/package-b
```

## Dependencies Management

```bash
# Add to current/specified package
uv add requests numpy            # Regular dependencies
uv add --dev pytest black        # Dev dependencies
uv add --optional docs sphinx    # Optional group
uv add -p packages/api fastapi   # To specific package

# Remove
uv remove requests
uv remove -p packages/api fastapi

# List dependencies
uv pip list
uv tree                          # Dependency tree

# Update lockfile
uv lock
uv lock --upgrade                # Upgrade all
uv sync                          # Sync venv with lock
```

## Virtual Environments

```bash
# Create venv
uv venv
uv venv --python 3.9
uv venv .venv-test

# Activate
source .venv/bin/activate        # Linux/macOS
.venv\Scripts\activate           # Windows

# Or use uv run (no activation needed)
uv run python script.py
uv run pytest
```

## Running Commands

```bash
# In current package
uv run python script.py
uv run pytest
uv run black .

# In specific package
uv run -p packages/core pytest
uv run -p packages/api uvicorn api.server:app

# Without syncing first
uv run --no-sync python script.py

# With custom Python version
uv run --python 3.9 script.py
```

## Workspace Configuration

### Root pyproject.toml
```toml
[tool.uv.workspace]
members = [
    "packages/core",
    "packages/processor",
    "packages/api",
]
```

### Package Dependencies
```toml
[project]
name = "my-api"
dependencies = [
    "core",           # ← Local package
    "fastapi>=0.100",
    "uvicorn>=0.23",
]
```

## Building & Publishing

```bash
# Build
uv build                    # Current package
uv build packages/core      # Specific package
uv build --wheel            # Only wheel
uv build --sdist            # Only source dist

# Publish to PyPI
pip install twine
twine upload dist/*

# Check build
uv build --check
```

## Project Structure Patterns

### Single Package
```
project/
├── pyproject.toml
├── src/my_package/
└── tests/
```

### Workspace (Recommended)
```
workspace/
├── pyproject.toml          # [tool.uv.workspace]
├── packages/
│   ├── core/
│   │   ├── pyproject.toml
│   │   └── src/core/
│   └── api/
│       ├── pyproject.toml
│       └── src/api/
```

## Common pyproject.toml Sections

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-project"
version = "0.1.0"
description = "..."
requires-python = ">=3.9"
dependencies = [
    "requests>=2.28.0",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=7.0", "black>=23.0"]
docs = ["sphinx>=5.0"]

[project.scripts]
my-cli = "my_package.cli:main"

[tool.uv.workspace]
members = ["packages/a", "packages/b"]

[tool.uv]
# UV-specific settings
```

## Dependency Version Constraints

```python
requests              # Any version
requests>=2.28       # At least 2.28
requests==2.28.1     # Exact version
requests>=2.28,<3.0  # Range (recommended)
requests[extra]      # With extras
```

## Debugging

```bash
# See full dependency tree
uv tree --depth 10

# Check which package provides a module
python -c "import requests; print(requests.__file__)"

# View resolved dependencies
uv pip show requests

# Validate pyproject.toml
uv build --check

# See what uv would do (dry run)
uv sync --dry-run
```

## Environment Variables

```bash
# Force reinstall
uv sync --force-reinstall

# Use offline mode
uv sync --offline

# Custom Python path
UV_PYTHON_PREFER_PYPY=1 uv sync

# Verbose output
uv sync -v
uv sync -vv
```

## Tips & Tricks

```bash
# Run command in all workspace packages
for pkg in packages/*/; do
  echo "Running in $pkg"
  uv run -p "$pkg" pytest
done

# Format and lint everything
uv run black .
uv run ruff check --fix .

# Run tests with coverage
uv run pytest --cov --cov-report=html

# Generate requirements.txt
uv pip compile pyproject.toml -o requirements.txt

# Upgrade all dependencies safely
uv lock --upgrade

# Interactive Python with workspace imports
uv run python
# Now you can import local packages directly
>>> from core.models import DataRecord
```

## Performance Tips

- Use **UV** instead of pip/pipenv for 10-20x speed improvements
- **Group dependencies**: Keep dev/test deps in optional groups
- **Use lockfile**: Always commit `uv.lock` for reproducible installs
- **Local packages**: Development is faster with local workspace packages
- **Cache**: UV automatically caches wheels and downloads

## Common Errors & Solutions

```
Error: "No such file or directory: 'pyproject.toml'"
→ Run from directory containing pyproject.toml

Error: "failed to find a Python installation"
→ uv venv --python 3.9  (specify version)

Error: "cannot import local package"
→ Make sure pyproject.toml has [tool.uv.workspace] members

Error: "Package X not found"
→ Add with: uv add -p packages/api package-name

ImportError: "No module named 'core'"
→ Run uv sync to install local packages in development mode
```

## Migration Guide

### From pip + requirements.txt
```bash
uv init
uv add $(cat requirements.txt | tr '\n' ' ')
rm requirements.txt
```

### From poetry
```bash
# Poetry still works, but uv is faster
# Copy dependencies to new pyproject.toml and use uv instead
uv init --lib
uv add $(grep "^[a-z]" pyproject.toml | cut -d' ' -f1 | tr '\n' ' ')
```

### From pipenv
```bash
# Export from pipenv
pipenv requirements > requirements.txt
pipenv requirements --dev > requirements-dev.txt

# Create with uv
uv init
uv add $(cat requirements.txt | tr '\n' ' ')
uv add --dev $(cat requirements-dev.txt | tr '\n' ' ')
```

## More Information

- Official docs: https://docs.astral.sh/uv/
- GitHub: https://github.com/astral-sh/uv
- Workspace docs: https://docs.astral.sh/uv/concepts/workspaces/
