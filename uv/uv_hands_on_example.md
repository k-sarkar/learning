# Hands-On: Building a Real Workspace Example

This guide walks through creating a complete, working multi-package workspace from scratch.

## Scenario: Data Processing Application

We'll build a workspace with three packages:
- **core**: Data models and utilities
- **processor**: Data processing logic (depends on core)
- **api**: REST API to expose processing (depends on core & processor)

## Step 1: Create the Workspace

```bash
# Create and enter workspace directory
mkdir data-processing-app
cd data-processing-app

# Initialize as a workspace
uv init --workspace
```

This creates:
```
data-processing-app/
├── pyproject.toml      # Workspace configuration
├── .python-version
└── README.md
```

The workspace `pyproject.toml`:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "data-processing-app"

[tool.uv.workspace]
members = []  # We'll add members as we create them
```

## Step 2: Create the Core Package

```bash
# Create packages directory
mkdir -p packages

# Initialize core package
uv init --lib packages/core
```

This creates:
```
packages/core/
├── pyproject.toml
├── .python-version
└── src/
    └── core/
        └── __init__.py
```

Edit `packages/core/src/core/__init__.py`:
```python
"""Core utilities and data models."""

from .models import DataRecord, ProcessingResult
from .utils import validate_data, format_timestamp

__all__ = [
    "DataRecord",
    "ProcessingResult",
    "validate_data",
    "format_timestamp",
]
```

Create `packages/core/src/core/models.py`:
```python
"""Data models for the application."""

from dataclasses import dataclass
from datetime import datetime


@dataclass
class DataRecord:
    """Represents a single data record."""
    
    id: str
    value: float
    timestamp: datetime
    
    def __str__(self) -> str:
        return f"DataRecord(id={self.id}, value={self.value}, ts={self.timestamp})"


@dataclass
class ProcessingResult:
    """Result of processing a data record."""
    
    record_id: str
    original_value: float
    processed_value: float
    processing_time_ms: float
    
    def __str__(self) -> str:
        return (
            f"Result(id={self.record_id}, "
            f"original={self.original_value}, "
            f"processed={self.processed_value})"
        )
```

Create `packages/core/src/core/utils.py`:
```python
"""Utility functions."""

from datetime import datetime


def validate_data(value: float) -> bool:
    """Validate if a value is within acceptable range."""
    return -1000 <= value <= 1000


def format_timestamp(ts: datetime) -> str:
    """Format a timestamp for display."""
    return ts.strftime("%Y-%m-%d %H:%M:%S")
```

Edit `packages/core/pyproject.toml`:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "core"
version = "0.1.0"
description = "Core utilities and data models"
requires-python = ">=3.9"
dependencies = []

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
]

[tool.hatch.build.targets.wheel]
packages = ["src/core"]
```

## Step 3: Create the Processor Package

```bash
uv init --lib packages/processor
```

Edit `packages/processor/src/processor/__init__.py`:
```python
"""Data processing engine."""

from .engine import DataProcessor

__all__ = ["DataProcessor"]
```

Create `packages/processor/src/processor/engine.py`:
```python
"""Processing engine that depends on core."""

import time
from core.models import DataRecord, ProcessingResult
from core.utils import validate_data, format_timestamp


class DataProcessor:
    """Process data records."""
    
    def __init__(self, multiplier: float = 1.5):
        self.multiplier = multiplier
    
    def process(self, record: DataRecord) -> ProcessingResult:
        """Process a single data record."""
        if not validate_data(record.value):
            raise ValueError(f"Invalid value: {record.value}")
        
        start_time = time.time()
        
        # Simulate processing
        processed_value = record.value * self.multiplier
        processed_value = max(-1000, min(1000, processed_value))  # Clamp
        
        processing_time_ms = (time.time() - start_time) * 1000
        
        return ProcessingResult(
            record_id=record.id,
            original_value=record.value,
            processed_value=processed_value,
            processing_time_ms=processing_time_ms,
        )
    
    def batch_process(self, records: list[DataRecord]) -> list[ProcessingResult]:
        """Process multiple records."""
        return [self.process(record) for record in records]
```

Edit `packages/processor/pyproject.toml`:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "processor"
version = "0.1.0"
description = "Data processing engine"
requires-python = ">=3.9"
dependencies = [
    "core",  # ← Local package reference
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
]

[tool.hatch.build.targets.wheel]
packages = ["src/processor"]
```

## Step 4: Create the API Package

```bash
uv init --lib packages/api
```

Edit `packages/api/src/api/__init__.py`:
```python
"""REST API for data processing."""

from .server import create_app

__all__ = ["create_app"]
```

Create `packages/api/src/api/server.py`:
```python
"""FastAPI server implementation."""

from datetime import datetime
from fastapi import FastAPI, HTTPException
from core.models import DataRecord, ProcessingResult
from processor.engine import DataProcessor


def create_app() -> FastAPI:
    """Create and configure the FastAPI application."""
    
    app = FastAPI(title="Data Processing API", version="0.1.0")
    processor = DataProcessor(multiplier=2.0)
    
    @app.get("/health")
    def health():
        """Health check endpoint."""
        return {"status": "healthy"}
    
    @app.post("/process")
    def process_single(record_id: str, value: float):
        """Process a single data record."""
        try:
            record = DataRecord(
                id=record_id,
                value=value,
                timestamp=datetime.now(),
            )
            result = processor.process(record)
            
            return {
                "success": True,
                "record_id": result.record_id,
                "original_value": result.original_value,
                "processed_value": result.processed_value,
                "processing_time_ms": result.processing_time_ms,
            }
        except ValueError as e:
            raise HTTPException(status_code=400, detail=str(e))
    
    return app


if __name__ == "__main__":
    import uvicorn
    app = create_app()
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Edit `packages/api/pyproject.toml`:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "api"
version = "0.1.0"
description = "REST API for data processing"
requires-python = ">=3.9"
dependencies = [
    "core",       # ← Local packages
    "processor",  # ← Local packages
    "fastapi>=0.100.0",
    "uvicorn>=0.23.0",
]

[project.scripts]
start-api = "api.server:main"

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "httpx>=0.24.0",
]

[tool.hatch.build.targets.wheel]
packages = ["src/api"]
```

## Step 5: Update Workspace Configuration

Edit the root `pyproject.toml`:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "data-processing-app"
version = "0.1.0"
description = "Multi-package data processing application"

[tool.uv.workspace]
members = [
    "packages/core",
    "packages/processor",
    "packages/api",
]
```

## Step 6: Set Up the Workspace

```bash
# From workspace root (data-processing-app/)
uv sync --all-extras

# This will:
# 1. Resolve dependencies across all packages
# 2. Install core, processor, api locally (editable mode)
# 3. Install all external dependencies
# 4. Create uv.lock file
```

## Step 7: Create Tests

Create `packages/core/tests/test_models.py`:
```python
"""Tests for core models."""

import pytest
from datetime import datetime
from core.models import DataRecord, ProcessingResult


def test_data_record_creation():
    """Test creating a DataRecord."""
    ts = datetime.now()
    record = DataRecord(id="test-1", value=42.5, timestamp=ts)
    
    assert record.id == "test-1"
    assert record.value == 42.5
    assert record.timestamp == ts


def test_processing_result_creation():
    """Test creating a ProcessingResult."""
    result = ProcessingResult(
        record_id="test-1",
        original_value=10.0,
        processed_value=20.0,
        processing_time_ms=1.5,
    )
    
    assert result.record_id == "test-1"
    assert result.processed_value == 20.0
```

Create `packages/processor/tests/test_engine.py`:
```python
"""Tests for the processing engine."""

import pytest
from datetime import datetime
from core.models import DataRecord
from processor.engine import DataProcessor


def test_processor_initialization():
    """Test initializing the processor."""
    processor = DataProcessor(multiplier=2.0)
    assert processor.multiplier == 2.0


def test_process_single_record():
    """Test processing a single record."""
    processor = DataProcessor(multiplier=2.0)
    record = DataRecord(
        id="test-1",
        value=10.0,
        timestamp=datetime.now(),
    )
    
    result = processor.process(record)
    
    assert result.record_id == "test-1"
    assert result.original_value == 10.0
    assert result.processed_value == 20.0


def test_process_invalid_record():
    """Test that invalid records raise errors."""
    processor = DataProcessor()
    record = DataRecord(
        id="test-1",
        value=10000.0,  # Out of range
        timestamp=datetime.now(),
    )
    
    with pytest.raises(ValueError):
        processor.process(record)
```

## Step 8: Run Tests

```bash
# From workspace root
uv sync --dev

# Run all tests
uv run pytest

# Run tests for a specific package
uv run -p packages/core pytest

# Run with coverage
uv run pytest --cov

# Run tests in watch mode
uv run pytest --watch
```

## Step 9: Run the Application

```bash
# From workspace root
uv sync  # Make sure everything is installed

# Run the API server
uv run -p packages/api python -m api.server

# In another terminal, test the API
curl http://localhost:8000/health
curl -X POST "http://localhost:8000/process?record_id=test-1&value=42.5"
```

## Step 10: Build and Package

```bash
# Build all packages
uv build

# Or build specific packages
uv build packages/core
uv build packages/processor
uv build packages/api

# Outputs appear in packages/*/dist/
```

## Final Workspace Structure

```
data-processing-app/
├── pyproject.toml              # Workspace root
├── uv.lock                     # Shared lockfile
├── README.md
│
├── packages/
│   │
│   ├── core/                   # No dependencies
│   │   ├── pyproject.toml
│   │   ├── src/core/
│   │   │   ├── __init__.py
│   │   │   ├── models.py
│   │   │   └── utils.py
│   │   └── tests/
│   │       └── test_models.py
│   │
│   ├── processor/              # Depends on: core
│   │   ├── pyproject.toml
│   │   ├── src/processor/
│   │   │   ├── __init__.py
│   │   │   └── engine.py
│   │   └── tests/
│   │       └── test_engine.py
│   │
│   └── api/                    # Depends on: core, processor
│       ├── pyproject.toml
│       ├── src/api/
│       │   ├── __init__.py
│       │   └── server.py
│       └── tests/
│           └── test_api.py
```

## Key Concepts Recap

1. **Workspace Root**: Single `pyproject.toml` at root with `[tool.uv.workspace]` section
2. **Members**: Each package is a member defined in the workspace config
3. **Local Imports**: Packages can import each other by name (e.g., `from core.models import DataRecord`)
4. **Shared Lock**: Single `uv.lock` ensures consistency across all packages
5. **Dependency Resolution**: UV resolves dependencies across all packages together
6. **Isolated Testing**: Test each package independently or all together

This is a production-ready pattern you can scale to large monorepos!
