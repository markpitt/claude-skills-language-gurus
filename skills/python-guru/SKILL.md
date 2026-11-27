---
name: python-guru
description: Enforces Python 3.12+ best practices when writing, reviewing, or refactoring Python code. Use when the user asks for Python help, code review, idiomatic Python, Pythonic patterns, Python project setup, debugging, or performance advice. Covers modern tooling (uv, ruff, pyright), type system (PEP 695 generics, Protocol, TypeIs), async (TaskGroup, timeout), testing (pytest + Hypothesis), security, and anti-pattern elimination. Do NOT use for general software architecture unrelated to Python.
license: Apache-2.0
metadata:
  version: "1.1.0"
---

# Python Guru

Enforce modern, idiomatic Python 3.12+ across every task. Never suggest
deprecated patterns, legacy tooling, or outdated idioms. When in doubt,
check the reference files before answering.

Python 3.15 (9 October 2026) is the current release. The floor stays at 3.12
because that is where PEP 695 type parameters and the `type` statement arrive,
and they are used throughout. New projects should pin a current release
(`uv python pin 3.14` or `3.15`). Code for 3.14 and later needs no
`from __future__ import annotations`, because annotations are deferred by
default.

## When to Use

- Writing new Python code of any kind
- Reviewing or refactoring existing Python code
- Setting up a new Python project (structure, tooling, CI)
- Debugging Python errors or unexpected behaviour
- Advising on async, typing, testing, or performance
- User asks: "is this Pythonic?", "how do I do X in Python?", "review my code"

## When NOT to Use

- Pure architecture discussions with no Python specifics
- Non-Python languages (even in mixed-language repos)
- System administration tasks where Python is incidental

## Enforcement Checklist

Run through this for **every** block of Python code you write or review:

### Tooling
- [ ] `uv` used for package/project management; no `pip install` directives in docs
- [ ] `ruff` configured in `pyproject.toml` with `UP`, `B`, `SIM`, `TC`, `S` rules enabled
- [ ] `pyright` (not mypy) as primary type checker; `typeCheckingMode = "standard"` minimum. Astral's `ty` is faster but still in beta (0.0.x), so mention it as an option, not the default
- [ ] All config in `pyproject.toml`; no `setup.py`, `setup.cfg`, `.flake8`, `mypy.ini`
- [ ] `requires-python = ">=3.12"` set

### Types
- [ ] `X | None` instead of `Optional[X]`
- [ ] `list[int]` / `dict[str, int]` instead of `List[int]` / `Dict[str, int]`
- [ ] `[T]` inline syntax (PEP 695) instead of `T = TypeVar("T")` for 3.12+ targets
- [ ] `type Alias = ...` statement instead of `TypeAlias` annotation
- [ ] `@override` on every method that intentionally overrides a parent method
- [ ] `Self` for fluent/builder return types
- [ ] `Protocol` preferred over `ABC` for structural interfaces
- [ ] `@dataclass(slots=True, frozen=True)` for value objects
- [ ] `if TYPE_CHECKING:` block for import-time-only type references

### Async
- [ ] `asyncio.TaskGroup` instead of `asyncio.gather()` for structured concurrency
- [ ] `asyncio.timeout()` context manager instead of `asyncio.wait_for()`
- [ ] `asyncio.get_running_loop()` inside coroutines (never `get_event_loop()`)
- [ ] `CancelledError` always re-raised after cleanup
- [ ] Blocking calls wrapped in `asyncio.to_thread()`

### Testing
- [ ] `pytest` with `@pytest.mark.parametrize`; no `unittest.TestCase`
- [ ] Hypothesis `@given` for pure functions that accept unconstrained input
- [ ] Fixtures use `yield` with teardown; scoped appropriately
- [ ] Tests outside `src/`; `tmp_path` for filesystem; no real network in unit tests
- [ ] `branch = true` in coverage; `fail_under = 90`

### Security
- [ ] Parameterised queries. Never f-strings into SQL
- [ ] `secrets` module for tokens/passwords. Never `random`
- [ ] `subprocess.run(["cmd", arg], ...)`. Never `shell=True` with user input
- [ ] No `eval()` or `exec()` on external input
- [ ] `LiteralString` annotations on functions that accept SQL/HTML/shell fragments
- [ ] Secrets sourced from environment variables. Never hardcoded
- [ ] `secrets.compare_digest()` for token comparison

### Common Anti-Patterns (instant fail)
- [ ] No mutable default arguments (`def f(x=[])`)
- [ ] No bare `except:`. Always at least `except Exception:`
- [ ] No `type(x) is Foo`. Use `isinstance(x, Foo)`
- [ ] No `datetime.utcnow()`. Use `datetime.now(UTC)`
- [ ] No string concatenation in loops. Use `str.join()`
- [ ] No `os.path.join()`. Use `pathlib.Path`
- [ ] No silent exception swallowing: log and re-raise, or use `suppress()` explicitly
- [ ] No root logger (`logging.getLogger()`). Always `logging.getLogger(__name__)`
- [ ] No f-strings in log calls. Use `%`-style lazy formatting

## Quick-Reference: Old → New

| Old | Current |
|---|---|
| `pip` + `virtualenv` + `pyenv` | `uv` |
| `black` + `isort` + `flake8` | `ruff format` + `ruff check` |
| `TypeVar("T")` | `[T]` inline type parameter |
| `Optional[X]` | `X \| None` |
| `Union[A, B]` | `A \| B` |
| `Dict[str, int]`, `List[int]` | `dict[str, int]`, `list[int]` |
| `TypeAlias` annotation | `type Alias = ...` statement |
| `asyncio.gather()` | `asyncio.TaskGroup` |
| `asyncio.wait_for()` | `asyncio.timeout()` |
| `datetime.utcnow()` | `datetime.now(UTC)` |
| `setup.py` / `setup.cfg` | `pyproject.toml` only |
| `ABC` for interfaces | `Protocol` |
| `@dataclass` | `@dataclass(slots=True)` |

## Project Bootstrap

For a new Python project, create this structure immediately:

```text
my-project/
├── src/
│   └── my_package/
│       ├── __init__.py   # define __all__
│       └── ...
├── tests/
│   ├── conftest.py
│   └── test_*.py
├── pyproject.toml
├── uv.lock               # commit for apps; gitignore for libraries
└── mise.toml             # optional but recommended
```

Bootstrap commands:

```bash
uv init my-project --lib   # src layout, pyproject.toml
cd my-project
uv add --dev pytest ruff pyright hypothesis
```

Minimal `pyproject.toml`:

```toml
[project]
requires-python = ">=3.12"

[tool.ruff]
target-version = "py312"

[tool.ruff.lint]
select = ["E","W","F","I","B","C4","UP","SIM","TC","RUF","PT","S","ANN"]
ignore = ["ANN101","ANN102","ANN401"]

[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "standard"

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--strict-markers -q"

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
fail_under = 90
```

## References

Load these files when you need depth beyond this overview:

- `references/tooling.md`: uv, ruff, pyright, mise configuration + CI setup
- `references/type-system.md`: full type annotation guide, PEP 695, Protocol, TypedDict
- `references/async.md`: async patterns, TaskGroup, pitfalls, anyio
- `references/testing.md`: pytest fixtures, Hypothesis strategies, coverage
- `references/security.md`: injection prevention, secrets, subprocess safety, t-strings
- `references/anti-patterns.md`: full anti-pattern catalogue with fixes
