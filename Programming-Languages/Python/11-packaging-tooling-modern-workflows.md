---
title: "11 - Packaging, Tooling, Modern Workflows"
created: 2026-05-19
updated: 2026-05-19
tags: []
aliases: []
---

# 11 - Packaging, Tooling, Modern Workflows

[toc]

> **TL;DR:** Modern Python packaging is `pyproject.toml`-centric: one file declares build metadata, dependencies, tool configuration, and scripts. `uv` (Rust-based) has replaced `pip` + `venv` for day-to-day development — it is 10–100× faster. The canonical production stack is `uv` for package management, `ruff` for linting and formatting, `pyright` for type checking, `pytest` for testing, and `pre-commit` + GitHub Actions for CI enforcement.

## Vocabulary

**`pyproject.toml`**: The single PEP 517/518 configuration file for Python projects. Contains build backend declaration, project metadata, dependencies, and tool configuration (ruff, pyright, pytest, etc.).

---

**`pip`**: The traditional Python package installer. Downloads packages from PyPI and installs them. Slow dependency resolver in older versions; PEP 665 + pip 23+ is faster.

---

**`uv`**: A Rust-based Python package installer and virtual environment manager from Astral. Drop-in replacement for pip + venv. 10–100× faster due to parallel downloads and Rust-native dependency resolution.

---

**`poetry`**: A higher-level dependency manager with its own lock file format (`poetry.lock`). Manages virtual environments, build, and publish in one tool. Slower than `uv` but widely used.

---

**`venv`**: Python's built-in virtual environment module (`python3 -m venv .venv`). Creates an isolated Python environment with its own `site-packages`. Always use a venv; never install into the system Python.

---

**`pyproject.toml` build backends**: The tool that actually builds the distribution. `hatchling`, `flit-core`, `setuptools`, `maturin` (Rust extensions) are common choices.

---

**`ruff`**: An extremely fast (Rust-implemented) Python linter and formatter. Implements a superset of flake8, isort, pyupgrade, and black rules. Replaces `black` + `flake8` + `isort` with one tool.

---

**`mypy`**: The original Python static type checker. Slower than pyright but has a larger plugin ecosystem.

---

**`pytest`**: The de-facto Python testing framework. Discovers tests via naming conventions (`test_*.py`), uses plain `assert` with rich introspection, and has a powerful fixture system.

---

**`pre-commit`**: A framework for managing git pre-commit hooks. Runs ruff, pyright, pytest, or any script automatically before every `git commit`.

---

**src layout**: Placing package code in `src/mypackage/` rather than directly in the project root. Prevents the package from being importable without installation, making `pip install -e .` mandatory — which catches import path bugs early.

---

**`pip install -e .`**: Editable install: installs the package in "development mode" so that edits to the source are reflected immediately without reinstalling.

---

## Intuition

Python's packaging history is famously chaotic — `setup.py`, `requirements.txt`, `setup.cfg`, `MANIFEST.in`, `tox.ini` all coexisted for a decade. `pyproject.toml` (PEP 518, 517) is the unification point: one file to describe what your package is, what it depends on, and how to build it. The tool ecosystem is now converging on this file as the single source of truth.

`uv` changes the development-speed equation. Where `pip install numpy pandas scikit-learn` might take 30 seconds, `uv add numpy pandas scikit-learn` takes 2–3 seconds because `uv` downloads wheels in parallel, uses a global cache, and resolves dependencies in Rust.

## `pyproject.toml` Structure

A complete `pyproject.toml` for a production-grade library:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mypackage"
version = "0.1.0"
description = "A well-typed Python package"
readme = "README.md"
requires-python = ">=3.11"
license = { text = "MIT" }
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.7",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "ruff>=0.4",
    "pyright>=1.1",
    "pre-commit>=3.7",
]

[project.scripts]
mypackage-cli = "mypackage.cli:main"

[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.pyright]
pythonVersion = "3.11"
typeCheckingMode = "strict"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--tb=short -q"
```

> [!IMPORTANT]
> `requires-python = ">=3.11"` in `pyproject.toml` is enforced by pip and uv — users on older Python get a clear error rather than cryptic `ImportError`. Always set this to the minimum version you test against.

## Package Management with `uv`

### Daily Workflows

```bash
# Create a new project
uv init myproject
cd myproject

# Add a dependency
uv add httpx

# Add a dev dependency
uv add --dev pytest ruff pyright

# Sync environment to lock file (reproducible installs)
uv sync

# Run a command in the project's environment
uv run pytest

# Create and activate a venv manually
uv venv .venv
source .venv/bin/activate   # macOS/Linux
# .venv\Scripts\activate      # Windows

# Install an existing project (editable)
uv pip install -e ".[dev]"
```

`uv.lock` is `uv`'s lock file — commit it to version control for deterministic builds. It pins every transitive dependency to an exact version and hash.

> [!TIP]
> Use `uv run pytest` instead of activating the venv manually. `uv run` automatically selects the correct environment for the current project directory. This is safer than activating — you cannot accidentally run tests with the wrong Python version.

## src Layout

The `src` layout is the recommended project structure for libraries:

```
myproject/
├── pyproject.toml
├── README.md
├── uv.lock
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── core.py
│       └── cli.py
└── tests/
    ├── __init__.py
    ├── test_core.py
    └── conftest.py
```

Without the `src` layout, Python can import your package directly from the project root even when it is not installed, masking missing `__init__.py` files and incorrect package boundaries. With `src`, the package is only importable after `pip install -e .`.

## Linting and Formatting with `ruff`

`ruff` replaces `flake8`, `isort`, `pyupgrade`, and `black` in a single sub-millisecond tool.

```bash
# Check for lint errors
ruff check src/

# Fix auto-fixable issues in place
ruff check --fix src/

# Format (black-compatible)
ruff format src/

# Check and format in CI
ruff check src/ && ruff format --check src/
```

Configuring `ruff` in `pyproject.toml`:

```toml
[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes (unused imports, undefined names)
    "I",    # isort (import order)
    "UP",   # pyupgrade (modernise syntax)
    "B",    # flake8-bugbear (common bugs)
    "SIM",  # flake8-simplify
    "ANN",  # flake8-annotations (enforce type hints)
]
ignore = ["E501"]  # line length handled by formatter
```

## Type Checking with `pyright`

`pyright` in `strict` mode catches: missing annotations, incompatible types, unreachable code, uninitialized variables, narrowing failures.

```bash
# Install
uv add --dev pyright

# Check
pyright src/

# In VSCode: Pylance uses pyright under the hood — install the Pylance extension
```

CI configuration (`.github/workflows/ci.yml`):

```yaml
---
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --all-extras
      - run: uv run ruff check src/ tests/
      - run: uv run ruff format --check src/ tests/
      - run: uv run pyright src/

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --all-extras
      - run: uv run pytest --tb=short
```

## `pytest` and Test Structure

```python
# tests/conftest.py
from typing import Generator
import pytest


@pytest.fixture
def temp_dir(tmp_path: "pytest.TempPathFactory") -> Generator[str, None, None]:
    """Provide a temporary directory for tests."""
    yield str(tmp_path)


# tests/test_core.py
from mypackage.core import process


def test_process_returns_correct_result() -> None:
    assert process(10) == 20


def test_process_raises_on_negative() -> None:
    with pytest.raises(ValueError, match="negative"):
        process(-1)


@pytest.mark.parametrize("n,expected", [(0, 0), (1, 2), (5, 10)])
def test_process_parametrized(n: int, expected: int) -> None:
    assert process(n) == expected
```

## `pre-commit` Hooks

`pre-commit` runs checks automatically before every `git commit`, preventing bad code from reaching the repo.

```yaml
# .pre-commit-config.yaml
---
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.10
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: local
    hooks:
      - id: pyright
        name: pyright
        entry: uv run pyright
        language: system
        types: [python]
        pass_filenames: false
```

```bash
pre-commit install   # install hooks into .git/hooks/pre-commit
pre-commit run --all-files  # run on all files manually
```

## Real-world Example

A complete minimal project scaffold showing `pyproject.toml`, `src` layout, a typed module, and its test.

```python
# src/mypackage/core.py
from __future__ import annotations


class ProcessingError(Exception):
    """Raised when processing fails."""


def process(n: int) -> int:
    """
    Double a non-negative integer.

    Args:
        n: The integer to double. Must be >= 0.

    Returns:
        n * 2

    Raises:
        ValueError: If n is negative.
    """
    if n < 0:
        raise ValueError(f"Expected non-negative integer, got {n}")
    return n * 2
```

```python
# tests/test_core.py
import pytest
from mypackage.core import process, ProcessingError


def test_double_positive() -> None:
    assert process(5) == 10


def test_double_zero() -> None:
    assert process(0) == 0


def test_negative_raises() -> None:
    with pytest.raises(ValueError, match="non-negative"):
        process(-1)


@pytest.mark.parametrize("n", [1, 2, 100, 1_000_000])
def test_double_parametric(n: int) -> None:
    assert process(n) == n * 2
```

> [!NOTE]
> `pytest` discovers tests in any file matching `test_*.py` or `*_test.py`. Plain `assert` statements produce rich failure messages — pytest rewrites them at collection time to display the actual vs expected values. No `self.assertEqual`; no test class required (though test classes are supported).

## In Practice

**`uv` lockfile discipline.** Commit `uv.lock` to version control. This ensures every developer and CI run uses the exact same transitive dependency versions. Run `uv sync` after pulling to update your environment.

**Pinning versions in `pyproject.toml`.** Use compatible-release constraints: `httpx>=0.27,<1.0` for libraries (avoid breaking changes); `httpx==0.27.3` only in application `pyproject.toml` where exact reproducibility matters more than flexibility.

**Namespace packages.** Without `__init__.py`, a directory becomes a PEP 420 namespace package — importable but not a regular package. This is useful for splitting a large package across multiple source trees, but confusing otherwise. Always include `__init__.py` in regular packages.

> [!CAUTION]
> Never run `pip install` into the system Python on macOS — Homebrew and macOS system tools depend on the system Python's `site-packages`. A bad pip install can break system tools in ways that are difficult to diagnose. Always use a virtual environment (`uv venv`, `python3 -m venv`, or `conda`). On macOS, `pip3 install` now refuses to install into the system Python by default (PEP 668), but older versions or explicit `--break-system-packages` can still cause damage.

## Pitfalls

- **"I'll manage dependencies with `requirements.txt`."** — `requirements.txt` has no dependency resolution — you must compute the transitive closure manually. Use `pyproject.toml` + `uv lock` for proper dependency management. `requirements.txt` is useful only for pinning for deployment.
- **"Global pip install is fine for development."** — It pollutes the system Python and causes version conflicts between projects. Always use a virtual environment.
- **"black and flake8 are the standard."** — As of 2024, `ruff` does both jobs 10–100× faster with zero configuration. New projects should use `ruff`.
- **"I don't need type hints in tests."** — `pytest` fixtures, parametrize decorators, and assertions all benefit from type hints. pyright checks test files too. Annotate test helpers and fixtures.
- **"Poetry's lock file is equivalent to uv.lock."** — Similar concept, different format. `poetry.lock` and `uv.lock` are not interchangeable. If you migrate from poetry to uv, regenerate the lock file.

## Exercises

### Exercise 1 — `pyproject.toml` from scratch

Write a minimal `pyproject.toml` for a package named `datatools` that: requires Python 3.11+, depends on `polars>=0.20`, has dev dependencies `pytest>=8` and `ruff>=0.4`, and exposes a CLI entry point `datatools` pointing to `datatools.cli:main`.

#### Solution

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "datatools"
version = "0.1.0"
description = "Data manipulation utilities"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "polars>=0.20",
]

[project.optional-dependencies]
dev = [
    "pytest>=8",
    "ruff>=0.4",
]

[project.scripts]
datatools = "datatools.cli:main"

[tool.ruff]
line-length = 88
target-version = "py311"

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Key points: `[build-system]` selects the build backend; `[project]` holds PEP 621 metadata; `[project.scripts]` maps the CLI name to the callable; `[project.optional-dependencies]` groups dev deps separately so `uv sync` (without `--all-extras`) installs only runtime deps in production.

---

### Exercise 2 — pre-commit hook order

You have ruff, pyright, and pytest in pre-commit. What order should they run and why?

#### Solution

Order: `ruff check --fix` → `ruff format` → `pyright` → `pytest` (optional in pre-commit, better in CI).

Reasoning:
1. **ruff check** first: fix auto-fixable lint errors before the formatter runs (some fixes change code that the formatter then needs to handle).
2. **ruff format**: apply formatting after lint fixes so the committed code is consistently formatted.
3. **pyright**: type checking after formatting ensures the formatter didn't introduce syntax pyright couldn't parse (rare but possible in edge cases). More importantly, type checking on auto-fixed code catches issues the lint fixes may have introduced.
4. **pytest** in pre-commit is controversial: it runs the full test suite on every commit, which can be slow for large projects. Better to run `pytest` in CI. In pre-commit, restrict to a fast smoke test: `pytest tests/unit/ -x -q`.

---

### Exercise 3 — Editable installs and src layout

Why does `python3 -c "import mypackage"` fail before running `pip install -e .` in a `src`-layout project, and succeed in a flat-layout project?

#### Solution

In a flat layout (`mypackage/` at the project root), `python3` finds `mypackage/` in the current directory because `.` is in `sys.path` by default. The package is importable without installation.

In a `src` layout (`src/mypackage/`), the package is inside `src/`. Python does not add `src/` to `sys.path` automatically — only `.`. So `import mypackage` raises `ModuleNotFoundError`.

Running `pip install -e .` adds an entry to the environment's `site-packages` that points at `src/`, which Python does include in `sys.path`. Now `import mypackage` finds the package through the installed path.

The benefit of this friction: it forces you to test the package as it will actually be installed, not as it happens to be in the current directory. Missing `__init__.py`, incorrect package names, and missing data files all surface immediately rather than in production.

## Sources

- PEP 517 — Build System Interface — https://peps.python.org/pep-0517/
- PEP 518 — Specifying Minimum Build System Requirements — https://peps.python.org/pep-0518/
- PEP 621 — Storing Project Metadata in `pyproject.toml` — https://peps.python.org/pep-0621/
- uv documentation — https://docs.astral.sh/uv/
- ruff documentation — https://docs.astral.sh/ruff/
- pytest documentation — https://docs.pytest.org/
- pyright documentation — https://github.com/microsoft/pyright
- pre-commit documentation — https://pre-commit.com/
- Hatch build backend — https://hatch.pypa.io/

## Related

- [1 - What is Python](./1-what-is-python.md)
- [6 - Type Hints and Static Typing](./6-type-hints-and-static-typing.md)
- [12 - Building Production Services in Python](./12-building-production-services-in-python.md)
