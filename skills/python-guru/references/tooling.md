# Tooling Reference

Modern Python tooling stack as of 2026. Every component is a replacement
for one or more legacy tools.

## uv: Package and Project Manager

Replaces: `pip`, `pip-tools`, `virtualenv`, `venv`, `pyenv`, `poetry`, `pipx`, `pipenv`

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create a new project (recommended: lib layout with src/)
uv init my-project --lib
cd my-project

# Add runtime dependencies
uv add fastapi httpx

# Add dev-only dependencies
uv add --dev pytest ruff pyright hypothesis pytest-cov

# Install / sync from lockfile (CI)
uv sync --frozen   # strict: fail if lockfile is out of date
uv sync            # update lockfile if needed

# Run commands inside the managed venv (no `source .venv/bin/activate` needed)
uv run python script.py
uv run pytest
uv run ruff check .

# Manage Python versions
uv python install 3.13
uv python pin 3.13   # writes .python-version

# Ephemeral tool execution (like npx)
uvx ruff check .
uvx pyright .

# Single-file scripts with inline dependencies (PEP 723)
# At top of script.py:
# /// script
# requires-python = ">=3.12"
# dependencies = ["requests>=2.28", "rich"]
# ///
uv run script.py
```

### Lockfile policy
- **Applications** (deployed services, CLIs): commit `uv.lock`. Reproducible installs
- **Libraries** (published to PyPI): add `uv.lock` to `.gitignore`. Consumers pin their own

### CI pattern
```yaml
- uses: astral-sh/setup-uv@v5
- run: uv sync --frozen
- run: uv run pytest
```

---

## ruff: Linter and Formatter

Replaces: `black`, `isort`, `flake8`, `pylint`, `pyupgrade`, `autoflake`, `bandit` (partial)

### Full recommended config

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes (undefined names, unused imports)
    "I",    # isort (import ordering)
    "B",    # flake8-bugbear (common bugs and design issues)
    "C4",   # flake8-comprehensions (simplify comprehensions)
    "UP",   # pyupgrade (modernise syntax — Optional → X|None etc.)
    "SIM",  # flake8-simplify
    "TID",  # flake8-tidy-imports (ban relative imports, etc.)
    "TC",  # flake8-type-checking (move type-only imports to TYPE_CHECKING)
    "RUF",  # ruff-specific rules (misc quality)
    "PT",   # flake8-pytest-style
    "S",    # flake8-bandit (security)
    "ANN",  # annotations (enforces type hints)
    "PERF", # perflint (performance anti-patterns)
    "PIE",  # flake8-pie (misc improvements)
    "ERA",  # eradicate (commented-out code)
]
ignore = [
    "ANN101",  # missing type for `self`
    "ANN102",  # missing type for `cls`
    "ANN401",  # allow Any in some cases
    "S101",    # allow assert in tests
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101", "ANN"]  # no type annotations required in tests

[tool.ruff.lint.isort]
known-first-party = ["my_package"]   # replace with your package name
force-sort-within-sections = true

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
line-ending = "auto"
```

### Common usage
```bash
uv run ruff check .          # lint
uv run ruff check --fix .    # lint + auto-fix
uv run ruff format .         # format (replaces black)
uv run ruff format --check . # CI format check
```

### Key rule groups to know

| Group | What it catches |
|---|---|
| `UP` | `Optional[X]` → `X\|None`, `List[int]` → `list[int]`, old-style unions |
| `B` | Mutable defaults, `assert False`, bare `raise`, loop variable issues |
| `TC` | Imports only needed for type hints (should be in `TYPE_CHECKING`) |
| `S` | `subprocess` with shell, `eval`, hardcoded secrets, weak crypto |
| `PERF` | List comp passed to sum/any/all, `+=` string loop |
| `ERA` | Commented-out code left in codebase |

---

## pyright: Type Checker

Replaces: `mypy` (primary), faster and better support for new PEPs

### Configuration

In `pyproject.toml`:
```toml
[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "standard"   # or "strict" for new projects
venvPath = "."
venv = ".venv"
reportMissingImports = true
reportMissingTypeStubs = false  # avoids noise for stubs-less packages
```

Or `pyrightconfig.json` (for editor integration):
```json
{
  "pythonVersion": "3.12",
  "typeCheckingMode": "standard",
  "venvPath": ".",
  "venv": ".venv"
}
```

### Strictness levels
- `"off"`: no checks (useless)
- `"basic"`: catches obvious errors
- `"standard"`, **recommended default**, catches most real bugs
- `"strict"`: requires full annotations everywhere; good for new greenfield code

### mypy vs pyright

| | pyright | mypy |
|---|---|---|
| Speed | Very fast (incremental, background daemon) | Slow (daemon helps somewhat) |
| PEP support | Immediate (often day-1) | Lags by months |
| VS Code | Native via Pylance | Requires plugin |
| `--strict` mode | More complete | Some gaps |
| Recommendation | **Primary** | Secondary / legacy compat only |

---

## mise: Runtime and Task Runner

Replaces: `nvm`, `asdf`, `direnv` (partially), `make`, `invoke`, `tox`

```toml
# mise.toml
[tools]
python = "3.13"
uv = "latest"
ruff = "latest"
pyright = "latest"

[env]
PYTHONPATH = "src"           # expose src/ for imports during dev
PYTHONDONTWRITEBYTECODE = "1"
PYTHONUNBUFFERED = "1"

[tasks.test]
run = "uv run pytest tests/ -q"

[tasks.lint]
run = [
    "uv run ruff check .",
    "uv run ruff format --check .",
    "uv run pyright .",
]

[tasks."lint:fix"]
run = ["uv run ruff check --fix .", "uv run ruff format ."]

[tasks.check]
depends = ["lint", "test"]
description = "Full pre-push check"
```

---

## pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.0    # pin to a version
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: check-yaml
      - id: check-toml
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-merge-conflict
      - id: detect-private-key
```

Install: `uv run pre-commit install`

---

## CI: GitHub Actions full pipeline

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true
      - run: uv sync --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run pyright .
      - run: uv run pytest --cov=src --cov-report=xml -q
      - uses: codecov/codecov-action@v4
        with:
          files: coverage.xml
```
