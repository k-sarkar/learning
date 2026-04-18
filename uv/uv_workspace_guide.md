# UV Workspace and Package Management Guide

## What is UV?

`uv` is a fast Python package installer and resolver written in Rust. It replaces `pip`, `pipenv`, and `poetry` with a unified tool that's significantly faster and more reliable. It supports:
- Package installation and dependency resolution
- Project scaffolding
- Workspace management (multiple packages in one repository)
- Virtual environment management
- Script running

## Installation

```bash
# On macOS
brew install uv

# On Linux/Windows or universal
curl -LsSf https://astral.sh/uv/install.sh | sh

# Verify installation
uv --version
```

---

## Part 1: Single Package Project

### Create a New Project

```bash
uv init my-project
cd my-project
```

This creates:
```
my-project/
├── pyproject.toml          # Project metadata and dependencies
├── README.md
├── .python-version         # Python version specification
├── .gitignore
└── src/
    └── my_project/
        └── __init__.py
```

### Understanding pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-project"
version = "0.1.0"
description = "Add your description here"
requires-python = ">=3.8"
dependencies = [
    "requests>=2.28.0",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "black>=23.0",
]

[tool.uv]
# UV specific configuration
```

### Adding Dependencies

```bash
# Add a regular dependency
uv add requests pandas

# Add a development dependency
uv add --dev pytest pytest-cov black ruff

# Add optional dependencies group
uv add --optional docs sphinx sphinx-rtd-theme

# Remove a dependency
uv remove requests
```

### Creating Virtual Environments

```bash
# Create a venv
uv venv

# Activate it (Linux/macOS)
source .venv/bin/activate

# Activate it (Windows)
.venv\Scripts\activate

# UV can run commands without activation
uv run python script.py
uv run pytest
```

---

## Part 2: Workspace Structure (Multi-Package)

### What is a Workspace?

A workspace allows you to manage multiple Python packages in a single repository. Packages can depend on each other, share dependencies, and are developed together.

### Create a Workspace

```bash
# Create workspace root directory
mkdir my-monorepo
cd my-monorepo

# Initialize workspace (create pyproject.toml at root)
uv init --workspace

# Create package members
uv init --lib packages/core
uv init --lib packages/cli
uv init --lib packages/server
```

---

## Part 3: Workspace Directory Structure

### Multi-Package Workspace Example

```
my-monorepo/
├── pyproject.toml                    # ← WORKSPACE ROOT
├── .python-version
├── .gitignore
├── README.md
├── uv.lock                           # Shared lockfile for entire workspace
│
├── packages/
│   │
│   ├── core/                         # Package 1: Core utilities
│   │   ├── pyproject.toml            # Package-specific config
│   │   ├── src/
│   │   │   └── core/
│   │   │       ├── __init__.py
│   │   │       ├── models.py
│   │   │       └── utils.py
│   │   └── tests/
│   │       ├── __init__.py
│   │       └── test_utils.py
│   │
│   ├── cli/                          # Package 2: Command-line interface
│   │   ├── pyproject.toml
│   │   ├── src/
│   │   │   └── cli/
│   │   │       ├── __init__.py
│   │   │       ├── main.py
│   │   │       └── commands/
│   │   │           ├── __init__.py
│   │   │           └── deploy.py
│   │   └── tests/
│   │       └── test_main.py
│   │
│   └── server/                       # Package 3: Server
│       ├── pyproject.toml
│       ├── src/
│       │   └── server/
│       │       ├── __init__.py
│       │       ├── app.py
│       │       └── routes/
│       │           ├── __init__.py
│       │           └── api.py
│       └── tests/
│           └── test_app.py
│
└── tools/                            # Optional: shared development tools
    ├── build.py
    └── lint.sh
```

### Workspace Root pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-monorepo"
version = "0.1.0"

[tool.uv.workspace]
# Define all members of the workspace
members = [
    "packages/core",
    "packages/cli",
    "packages/server",
]

# Shared dependencies (optional)
[tool.uv]
managed = true
```

### Package-Specific pyproject.toml (packages/cli/pyproject.toml)

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-cli"
version = "0.1.0"
description = "CLI package"
requires-python = ">=3.8"

# Depend on other packages in the workspace
dependencies = [
    "core",              # ← Local package reference
    "click>=8.0",
    "rich>=10.0",
]

[project.scripts]
my-cli = "cli.main:main"
```

### Package-Specific pyproject.toml (packages/core/pyproject.toml)

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "core"
version = "0.1.0"
description = "Core utilities"
requires-python = ">=3.8"
dependencies = [
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest>=7.0"]
```

---

## Part 4: Managing Workspace Dependencies

### Adding Dependencies to a Package

```bash
# Add to a specific package
uv add -p packages/cli click rich

# Add development dependency to a package
uv add -p packages/core --dev pytest

# Add optional dependency group
uv add -p packages/core --optional dev pytest pytest-cov
```

### Dependency Resolution

When you run `uv sync` or add dependencies:
- UV resolves dependencies **across all workspace packages**
- Creates a **single shared lockfile** (`uv.lock`) for deterministic installs
- Packages can depend on each other using local references

### Syncing the Entire Workspace

```bash
# Install all dependencies for all packages
uv sync

# Install with dev dependencies
uv sync --dev

# Install specific optional groups
uv sync --extra dev
```

---

## Part 5: Practical Workflow

### Running Commands in a Workspace

```bash
# Run tests for a specific package
uv run -p packages/core pytest

# Run tests for all packages
uv run pytest

# Run a script from the CLI package
uv run -p packages/cli python -m cli.main

# Run a console script defined in pyproject.toml
uv run my-cli --help
```

### Building and Publishing

```bash
# Build a specific package
uv build packages/core

# Outputs to packages/core/dist/
# - core-0.1.0.tar.gz
# - core-0.1.0-py3-none-any.whl

# Publish to PyPI
# First build
uv build packages/core

# Then upload
pip install twine
twine upload packages/core/dist/*
```

### Example: Using Core in CLI

**packages/core/src/core/models.py:**
```python
from pydantic import BaseModel

class Config(BaseModel):
    name: str
    version: str
    
    def display(self):
        return f"{self.name} v{self.version}"
```

**packages/cli/src/cli/main.py:**
```python
from core.models import Config
import click

@click.command()
def main():
    config = Config(name="MyApp", version="1.0")
    click.echo(config.display())

if __name__ == "__main__":
    main()
```

When you run `uv run -p packages/cli python src/cli/main.py`, the CLI package can import from `core` directly because UV manages the local path resolution.

---

## Part 6: Best Practices

### Directory Organization

**Option 1: Packages Subdirectory** (recommended for workspaces)
```
workspace/
├── pyproject.toml
├── packages/
│   ├── package-a/
│   └── package-b/
```

**Option 2: Root-level Packages**
```
workspace/
├── pyproject.toml
├── package-a/
├── package-b/
```

**Option 3: Src Layout** (recommended for quality)
```
workspace/
├── pyproject.toml
├── packages/
│   └── my-package/
│       ├── src/
│       │   └── my_package/
│       │       └── __init__.py
│       └── tests/
```

### Dependency Best Practices

```toml
# ✅ Good: Specific version constraints
dependencies = [
    "requests>=2.28.0,<3.0",
    "pydantic>=2.0,<3.0",
]

# ❌ Avoid: Unpinned versions
dependencies = [
    "requests",
    "pydantic",
]

# ✅ Good: Development dependencies separate
[project.optional-dependencies]
dev = ["pytest>=7.0", "black>=23.0"]
```

### Virtual Environment Strategy

```bash
# One shared venv for entire workspace (recommended)
uv venv
source .venv/bin/activate
uv sync --all-extras

# Or use UV's direct execution (no activation needed)
uv run pytest
uv run -p packages/cli python script.py
```

---

## Part 7: Common Commands Reference

```bash
# Initialization
uv init                          # Create new project
uv init --workspace              # Create workspace root
uv init --lib                    # Create library package

# Dependencies
uv add package-name              # Add dependency
uv add --dev pytest              # Add dev dependency
uv remove package-name           # Remove dependency
uv pip list                      # List installed packages

# Managing packages
uv sync                          # Install workspace dependencies
uv sync --dev                    # Include dev dependencies
uv tree                          # Show dependency tree

# Running
uv run python script.py          # Run with installed dependencies
uv run -p package pytest         # Run in specific package
uv run --no-sync pytest          # Run without syncing first

# Building
uv build                         # Build current project
uv build -p package             # Build specific package

# Virtual environment
uv venv                          # Create venv
uv venv --python 3.9            # Create with specific Python version
uv venv --upgrade                # Upgrade venv
```

---

## Summary

| Feature | Single Package | Workspace |
|---------|---|---|
| Root pyproject.toml | 1 file | 1 file (at root) |
| Package pyproject.toml | 1 file | 1 per package |
| Lockfile | `uv.lock` | Shared `uv.lock` |
| Dependency resolution | Per package | Across all packages |
| Local imports | N/A | Package name in dependencies |
| Best for | Simple projects | Monorepos, related packages |

Use `uv` to manage both! It handles single packages seamlessly and scales to complex workspaces.
