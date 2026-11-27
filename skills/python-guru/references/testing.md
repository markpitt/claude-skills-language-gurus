# Testing Reference

Modern Python testing with pytest, Hypothesis, and coverage. No `unittest.TestCase`
for new code. Tests are plain functions with fixtures.

---

## pytest Fundamentals

### Basic structure

```python
# tests/test_user.py
from my_package.user import User, create_user

def test_create_user_sets_name() -> None:
    user = create_user("Alice")
    assert user.name == "Alice"

def test_create_user_default_active() -> None:
    user = create_user("Bob")
    assert user.is_active is True
```

### Parametrize: don't write repeated tests

```python
import pytest

@pytest.mark.parametrize("raw,expected", [
    ("  hello  ", "hello"),
    ("\thello\n", "hello"),
    ("hello", "hello"),
    ("", ""),
])
def test_normalize(raw: str, expected: str) -> None:
    assert normalize(raw) == expected

# Parametrize with IDs for readable failure output
@pytest.mark.parametrize("value", [None, "", "  "], ids=["none", "empty", "whitespace"])
def test_is_blank(value: str | None) -> None:
    assert is_blank(value) is True
```

### Fixtures: dependency injection over setup/teardown

```python
import pytest
from pathlib import Path

# Basic fixture
@pytest.fixture
def sample_user() -> User:
    return User(name="Alice", email="alice@example.com")

# Fixture with teardown (always use yield, not return)
@pytest.fixture
def temp_db(tmp_path: Path):
    db = Database(tmp_path / "test.db")
    db.initialize()
    yield db
    db.close()    # teardown runs after test, even if test fails

# Scoped fixtures — expensive resources shared across tests
@pytest.fixture(scope="session")
def http_client():
    with httpx.Client(base_url="http://testserver") as client:
        yield client

# Fixture factory
@pytest.fixture
def make_user():
    def _make(name: str = "Alice", role: str = "viewer") -> User:
        return User(name=name, role=role)
    return _make

def test_admin_access(make_user) -> None:
    admin = make_user(role="admin")
    assert admin.can_delete() is True
```

### Exception testing

```python
def test_raises_on_invalid_email() -> None:
    with pytest.raises(ValidationError, match="invalid email"):
        create_user("Alice", email="not-an-email")

# Check exception attributes
def test_error_has_field() -> None:
    with pytest.raises(ValidationError) as exc_info:
        validate_age(-1)
    assert exc_info.value.field == "age"
```

### Markers

```python
# Custom markers (register in pyproject.toml to avoid warnings)
@pytest.mark.slow
def test_large_import() -> None: ...

@pytest.mark.integration
def test_real_database() -> None: ...

# Skip conditionally
@pytest.mark.skipif(sys.platform == "win32", reason="Unix-only")
def test_symlinks() -> None: ...

# Expected failures
@pytest.mark.xfail(reason="known regression, issue #123")
def test_edge_case() -> None: ...
```

Register markers:
```toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with -m 'not slow')",
    "integration: marks integration tests requiring external services",
]
```

---

## Hypothesis: Property-Based Testing

Use whenever a function accepts unconstrained input. Hypothesis finds edge cases
that `parametrize` never covers.

```python
from hypothesis import given, assume, settings, HealthCheck
from hypothesis import strategies as st

# Basic: test a property holds for all inputs
@given(st.lists(st.integers()))
def test_sort_is_idempotent(xs: list[int]) -> None:
    assert sorted(sorted(xs)) == sorted(xs)

@given(st.text())
def test_encode_decode_roundtrip(s: str) -> None:
    assert decode(encode(s)) == s

# Assume: filter invalid inputs within the strategy
@given(st.integers())
def test_inverse(n: int) -> None:
    assume(n != 0)    # skip if n == 0
    assert inverse(n) * n == pytest.approx(1.0)

# Composite strategies
@st.composite
def user_data(draw) -> dict:
    name = draw(st.text(min_size=1, max_size=100))
    age = draw(st.integers(min_value=0, max_value=150))
    return {"name": name, "age": age}

@given(user_data())
def test_create_user(data: dict) -> None:
    user = create_user(**data)
    assert user.name == data["name"]

# Settings for CI (fewer examples to keep tests fast)
@settings(max_examples=50, suppress_health_check=[HealthCheck.too_slow])
@given(st.binary())
def test_parse_bytes(data: bytes) -> None:
    result = safe_parse(data)
    assert isinstance(result, dict | None)
```

### Key strategies

| Strategy | Use for |
|---|---|
| `st.integers()` | Any integer; use `min_value`/`max_value` |
| `st.text()` | Unicode strings; use `alphabet` to restrict |
| `st.floats(allow_nan=False)` | Floats (set `allow_nan` explicitly) |
| `st.lists(st.integers())` | Lists of a given type |
| `st.dictionaries(st.text(), st.integers())` | Dict with typed keys/values |
| `st.one_of(st.none(), st.integers())` | `int \| None` |
| `st.from_type(MyClass)` | Auto-generation from type annotations |
| `st.builds(MyClass, name=st.text())` | Build dataclasses/models with overrides |

---

## Coverage

```toml
[tool.coverage.run]
source = ["src"]
branch = true                # branch coverage, not just line coverage
omit = ["src/*/migrations/*"]

[tool.coverage.report]
fail_under = 90
show_missing = true
exclude_lines = [
    "if TYPE_CHECKING:",
    "@overload",
    "raise NotImplementedError",
    "\\.\\.\\.",               # abstract method bodies
    "if __name__ == .__main__.:",
]
```

```bash
uv run pytest --cov=src --cov-report=term-missing --cov-report=xml
```

---

## Test Organisation

```
tests/
├── conftest.py          # shared fixtures (auto-loaded by pytest)
├── unit/
│   ├── test_user.py
│   └── test_validation.py
├── integration/
│   └── test_database.py
└── e2e/
    └── test_api.py
```

### conftest.py: shared fixtures and configuration

```python
# tests/conftest.py
import pytest
from pathlib import Path
from my_package.db import Database

@pytest.fixture(scope="session")
def db_path(tmp_path_factory) -> Path:
    return tmp_path_factory.mktemp("db") / "test.db"

@pytest.fixture(scope="session")
def db(db_path: Path):
    database = Database(db_path)
    database.migrate()
    yield database
    database.close()
```

---

## Common Testing Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| `unittest.TestCase` | Plain functions + pytest fixtures |
| Mocking everything | Use real implementations + `tmp_path` / fake servers |
| `setUp` / `tearDown` | Fixtures with `yield` |
| Global state in tests | Scope fixtures properly; use `tmp_path` |
| Testing implementation details | Test behaviour, not internals |
| One assert per test (too granular) | Group related assertions; use fixtures for setup |
| Magic numbers in assertions | Named constants or parametrize IDs |
| `import unittest; unittest.main()` | `pytest` runs without a `main()` block |
| Hardcoded paths | `tmp_path` fixture, `importlib.resources` for fixtures data |

---

## Snapshot Testing

For HTML rendering, serialisation output, or complex data structures:

```python
# syrupy (recommended)
from syrupy.assertion import SnapshotAssertion

def test_render(snapshot: SnapshotAssertion) -> None:
    result = render_template("email.html", user=user)
    assert result == snapshot   # creates snapshot on first run; fails on drift

# Update snapshots
# uv run pytest --snapshot-update
```

---

## Async Testing

```python
# pytest-anyio (recommended — backend agnostic)
import anyio
import pytest

@pytest.mark.anyio
async def test_async_fetch() -> None:
    result = await fetch("https://example.com")
    assert result["status"] == "ok"

# Configure default backend in conftest.py:
# pytest_plugins = ('anyio',)
# anyio_backend = "asyncio"
```
