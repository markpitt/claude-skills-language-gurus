# Anti-Patterns Catalogue

Comprehensive list of Python anti-patterns, each with a fix. Most are caught
automatically by `ruff` — the relevant rule code is noted where applicable.

---

## Type Annotation Anti-Patterns

### `Optional[X]` — use `X | None` (ruff `UP007`)
```python
# BAD
from typing import Optional
def greet(name: Optional[str]) -> Optional[str]: ...

# GOOD
def greet(name: str | None) -> str | None: ...
```

### `Union[A, B]` — use `A | B` (ruff `UP007`)
```python
# BAD
from typing import Union
def process(x: Union[int, str, float]) -> Union[str, bytes]: ...

# GOOD
def process(x: int | str | float) -> str | bytes: ...
```

### `List`, `Dict`, `Tuple` from `typing` — use built-ins (ruff `UP006`)
```python
# BAD
from typing import List, Dict, Tuple, Set, FrozenSet
def sort(items: List[int]) -> List[int]: ...

# GOOD
def sort(items: list[int]) -> list[int]: ...
```

### `TypeVar` module-level variable — use inline params (3.12+)
```python
# BAD
from typing import TypeVar, Generic
T = TypeVar("T")
def first(items: list[T]) -> T: ...

# GOOD (3.12+)
def first[T](items: list[T]) -> T: ...
```

### Using `type(x) is` — use `isinstance` (ruff `E721`)
```python
# BAD — doesn't work with subclasses
if type(x) is list: ...

# GOOD
if isinstance(x, list): ...
if isinstance(x, int | str): ...   # union isinstance (3.10+)
```

---

## Mutable Default Arguments (ruff `B006`)

```python
# BAD — same list shared across all calls
def add_tag(name: str, tags: list[str] = []) -> list[str]:
    tags.append(name)
    return tags

# GOOD
def add_tag(name: str, tags: list[str] | None = None) -> list[str]:
    if tags is None:
        tags = []
    tags.append(name)
    return tags

# Or with dataclasses (never directly assign mutable defaults there either):
from dataclasses import dataclass, field

@dataclass
class Config:
    tags: list[str] = field(default_factory=list)
```

---

## Exception Handling Anti-Patterns

### Bare `except` — always be specific
```python
# BAD — catches SystemExit, KeyboardInterrupt, GeneratorExit
try:
    do_something()
except:
    pass

# ALSO BAD — catches everything silently
try:
    do_something()
except Exception:
    pass      # swallowed — caller has no idea it failed

# GOOD — be specific, log, re-raise where appropriate
try:
    data = json.loads(raw)
except json.JSONDecodeError as e:
    raise ValidationError("body", f"Invalid JSON") from e
```

### Silent swallowing — always log or raise
```python
# BAD
try:
    process(item)
except Exception:
    pass

# GOOD — use suppress() when suppression is intentional and named
from contextlib import suppress
with suppress(FileNotFoundError):
    os.unlink(temp_file)

# GOOD — log unexpected errors and re-raise
try:
    process(item)
except Exception:
    logger.exception("Unexpected error processing item %r", item)
    raise
```

### Losing exception context (ruff `B904`)
```python
# BAD — original traceback lost
try:
    parse(data)
except ValueError:
    raise RuntimeError("parse failed")               # no `from`

# GOOD — chain exceptions
try:
    parse(data)
except ValueError as e:
    raise RuntimeError("parse failed") from e        # preserves context

# Intentional suppression of chain — use `from None`
try:
    sensitive_operation(user_token)
except Exception as e:
    raise PublicError("operation failed") from None  # hides internal details
```

---

## String Anti-Patterns

### String concatenation in loops (ruff `PERF401`)
```python
# BAD — O(n²) due to string immutability
result = ""
for item in items:
    result += str(item)

# GOOD — O(n)
result = "".join(str(item) for item in items)

# For building structured strings
parts = [f"Item {i}: {name}" for i, name in enumerate(items)]
output = "\n".join(parts)
```

### f-strings in logging calls (ruff `G004`)
```python
# BAD — always formats the string, even if log level is disabled
logger.debug(f"Processing user {user_id} with config {config!r}")

# GOOD — lazy: only formats if DEBUG is enabled
logger.debug("Processing user %s with config %r", user_id, config)
```

### Old-style `%s` formatting or `.format()` in new code
```python
# ACCEPTABLE for logging (lazy)
logger.info("Hello %s", name)

# GOOD for everything else — f-strings
message = f"Hello {name}"
url = f"https://api.example.com/users/{user_id}"

# GOOD for complex templates — Template or format()
from string import Template
t = Template("Hello $name, you have $count messages")
msg = t.safe_substitute(name=name, count=count)
```

---

## Path Anti-Patterns (ruff `PTH`)

```python
# BAD — os.path API
import os
path = os.path.join(base_dir, "subdir", "file.txt")
if os.path.exists(path):
    with open(path) as f:
        content = f.read()
parent = os.path.dirname(os.path.abspath(__file__))

# GOOD — pathlib
from pathlib import Path
path = Path(base_dir) / "subdir" / "file.txt"
if path.exists():
    content = path.read_text()
parent = Path(__file__).parent.resolve()

# Glob patterns
py_files = list(Path("src").glob("**/*.py"))
config = next(Path(".").glob("*.toml"), None)
```

---

## Datetime Anti-Patterns (ruff `DTZ`)

```python
import datetime

# BAD — deprecated in 3.12, returns naive datetime (no timezone)
now = datetime.datetime.utcnow()

# BAD — naive (no timezone), ambiguous
now = datetime.datetime.now()

# GOOD — always timezone-aware
from datetime import datetime, UTC, timezone
now = datetime.now(UTC)
local_now = datetime.now(timezone(datetime.timedelta(hours=5, minutes=30)))

# Convert from timestamp
ts = datetime.fromtimestamp(unix_timestamp, tz=UTC)
```

---

## Async Anti-Patterns

```python
# BAD — blocking in async context
async def handler():
    time.sleep(1)          # blocks event loop, freezes all other coroutines
    data = open(path).read()   # blocking IO

# GOOD
async def handler():
    await asyncio.sleep(1)
    data = await asyncio.to_thread(Path(path).read_text)

# BAD — get_event_loop() outside coroutine (deprecated/broken on 3.12+)
loop = asyncio.get_event_loop()

# GOOD — inside coroutines
loop = asyncio.get_running_loop()
# Or just use asyncio.run() at the entry point
asyncio.run(main())
```

---

## Import Anti-Patterns

```python
# BAD — star imports pollute namespace, hide dependencies
from module import *

# BAD — circular imports from runtime imports
# (use TYPE_CHECKING block + annotations instead)

# GOOD — explicit imports with __all__ in the source module
from my_package.user import User, create_user

# GOOD — type-only imports in TYPE_CHECKING block (ruff TCH001/TCH002/TCH003)
from __future__ import annotations
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from my_package.heavy import HeavyClass
```

---

## Resource Management Anti-Patterns

```python
# BAD — resource not closed on exception
f = open(path)
data = f.read()
f.close()

# GOOD — context manager guarantees cleanup
with open(path) as f:
    data = f.read()

# BAD — multiple resources, partial failure
conn = db.connect()
cursor = conn.cursor()
# if cursor() raises, conn is never closed

# GOOD
from contextlib import ExitStack
with ExitStack() as stack:
    conn = stack.enter_context(db.connect())
    cursor = stack.enter_context(conn.cursor())
```

---

## Class Anti-Patterns

```python
# BAD — __init__ doing too much, hard to test
class UserService:
    def __init__(self):
        self.db = Database("prod.db")   # hardcoded dependency
        self.cache = Redis("localhost")

# GOOD — dependency injection
class UserService:
    def __init__(self, db: Database, cache: Cache) -> None:
        self.db = db
        self.cache = cache

# BAD — dataclass with mutable default
@dataclass
class Config:
    tags: list[str] = []   # all instances share same list!

# GOOD
@dataclass
class Config:
    tags: list[str] = field(default_factory=list)
```

---

## Logging Anti-Patterns

```python
# BAD — root logger (affects all libraries)
logging.basicConfig(level=logging.DEBUG)
logging.info("started")

# GOOD — module-level logger
import logging
logger = logging.getLogger(__name__)
logger.info("started")

# BAD — print() for diagnostics in production code
print(f"Processing {item_id}")

# GOOD
logger.debug("Processing item %s", item_id)

# BAD — swallow exception without logging
except Exception:
    pass

# GOOD — always log unexpected exceptions
except Exception:
    logger.exception("Unexpected failure")   # includes full traceback
    raise
```

---

## Performance Anti-Patterns (ruff `PERF`)

```python
# BAD — list comprehension passed to any/all (creates full list before checking)
if any([x > 0 for x in items]):   # ruff PERF101

# GOOD — generator (short-circuits)
if any(x > 0 for x in items):

# BAD — repeated attribute lookup in tight loop
for item in items:
    result_list.append(process(item))   # ruff PERF401: lookup per iteration

# GOOD — local reference
append = result_list.append
for item in items:
    append(process(item))

# BAD — list(dict.keys()) is redundant (dict is already iterable)
for key in list(d.keys()):

# GOOD
for key in d:

# BAD — __slots__ omitted from hot-path objects
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y        # uses __dict__, ~200 bytes overhead

# GOOD
@dataclass(slots=True)
class Point:
    x: float
    y: float
```
