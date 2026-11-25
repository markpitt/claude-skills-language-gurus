# Async Reference

Modern asyncio patterns for Python 3.11+. Structured concurrency is the
default; unstructured (`gather`, `create_task` without tracking) is a code smell.

---

## Core Rules

1. **Always start with `asyncio.run()`** — never `loop.run_until_complete()`
2. **Prefer `TaskGroup` over `gather()`** for concurrent tasks
3. **Use `asyncio.timeout()` context manager** over `wait_for()`
4. **Always re-raise `CancelledError`** after cleanup
5. **Use `asyncio.to_thread()`** for blocking code in async context
6. **Use `asyncio.get_running_loop()`** inside coroutines, never `get_event_loop()`

---

## TaskGroup — Structured Concurrency (3.11+)

```python
import asyncio

# PREFERRED: TaskGroup — cancels all siblings on first failure
async def fetch_all(urls: list[str]) -> list[bytes]:
    results: list[bytes] = []
    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(fetch(url)) for url in urls]
    # All tasks complete here (or ExceptionGroup is raised)
    return [t.result() for t in tasks]

# AVOID: gather() — weaker guarantees
results = await asyncio.gather(fetch(url1), fetch(url2))
# Problems:
#   - On failure, other tasks keep running unless return_exceptions=False
#   - Silent failures with return_exceptions=True
#   - No structured lifetime
```

### TaskGroup + ExceptionGroup

```python
try:
    async with asyncio.TaskGroup() as tg:
        tg.create_task(coro_a())
        tg.create_task(coro_b())
except* ValueError as eg:
    for exc in eg.exceptions:
        logger.error("ValueError: %s", exc)
except* httpx.HTTPError as eg:
    logger.error("%d HTTP errors occurred", len(eg.exceptions))
```

**Rule:** All exceptions from `TaskGroup` arrive as `ExceptionGroup`. Use `except*` (3.11+)
to handle them — never `except Exception` which would catch the entire group at once.

---

## Timeouts (3.11+)

```python
import asyncio

# PREFERRED: asyncio.timeout() context manager
async def fetch_with_timeout(url: str) -> bytes:
    async with asyncio.timeout(10.0):
        return await fetch(url)

# With reschedule
async def fetch_with_retry(url: str, deadline: float) -> bytes:
    async with asyncio.timeout_at(deadline):
        return await fetch(url)

# Checking if timeout is close
async def adaptive_task() -> None:
    async with asyncio.timeout(30.0) as deadline:
        for chunk in data_chunks:
            if deadline.expired():
                break
            await process(chunk)

# Error handling — TimeoutError raised OUTSIDE the context manager
try:
    async with asyncio.timeout(5.0):
        result = await long_operation()
except TimeoutError:
    use_cached_result()

# AVOID: asyncio.wait_for() — verbose, less composable
try:
    result = await asyncio.wait_for(long_operation(), timeout=5.0)
except asyncio.TimeoutError:
    ...
```

---

## Cancellation — Never Swallow CancelledError

```python
# CORRECT: always re-raise after cleanup
async def worker() -> None:
    try:
        await long_running_task()
    except asyncio.CancelledError:
        await cleanup()   # do cleanup
        raise             # ALWAYS re-raise — never suppress

# WRONG: swallows cancellation, task appears done but wasn't
async def worker_bad() -> None:
    try:
        await long_running_task()
    except asyncio.CancelledError:
        pass              # BUG: cancellation silently swallowed

# uncancel() — for shielding specific sub-tasks (advanced)
async def shielded() -> None:
    task = asyncio.current_task()
    assert task is not None
    try:
        await asyncio.shield(critical_operation())
    except asyncio.CancelledError:
        task.uncancel()   # mark cancellation handled
        raise
```

---

## Fire-and-Forget Tasks (with proper tracking)

```python
_background_tasks: set[asyncio.Task] = set()

def schedule(coro: Coroutine) -> asyncio.Task:
    """Schedule a fire-and-forget task while keeping a strong reference."""
    task = asyncio.create_task(coro)
    _background_tasks.add(task)
    task.add_done_callback(_background_tasks.discard)
    return task

# Without this pattern, tasks can be garbage-collected mid-flight
# (Python holds only a weak reference to tasks)
```

---

## Blocking Code in Async Context

```python
import asyncio

# IO-bound blocking (network, filesystem, database)
result = await asyncio.to_thread(blocking_db_query, user_id)

# CPU-bound
loop = asyncio.get_running_loop()
with concurrent.futures.ProcessPoolExecutor() as pool:
    result = await loop.run_in_executor(pool, cpu_heavy_computation, data)

# File IO — use asyncio.to_thread() or aiofiles for true async
import aiofiles
async def read_file(path: str) -> str:
    async with aiofiles.open(path) as f:
        return await f.read()

# Semaphore for concurrency limiting
sem = asyncio.Semaphore(10)

async def limited_fetch(url: str) -> bytes:
    async with sem:
        return await fetch(url)
```

---

## Common Async Pitfalls

| Anti-pattern | Fix |
|---|---|
| `asyncio.get_event_loop()` in coroutine | `asyncio.get_running_loop()` |
| `time.sleep(n)` in async code | `await asyncio.sleep(n)` |
| `open()` for IO in async-heavy code | `asyncio.to_thread(open, ...)` or `aiofiles` |
| `asyncio.gather()` and ignoring exceptions | `TaskGroup` with `except*` |
| Missing `await` on coroutine call | Static analysis (`ruff B008`, pyright) catches this |
| Long CPU work in a coroutine | `loop.run_in_executor()` with `ProcessPoolExecutor` |
| Creating tasks without storing the reference | Keep in a set, use `add_done_callback` to discard |
| `asyncio.wait_for()` for timeouts | `async with asyncio.timeout():` |

---

## asyncio vs anyio

| Use case | Recommendation |
|---|---|
| Application code | `asyncio` directly — no extra dependency |
| Library code that must be backend-agnostic | `anyio` — works with asyncio and trio |
| Framework code (web servers, test runners) | Check if already uses anyio/trio |

```python
# anyio for libraries
import anyio

async def main() -> None:
    async with anyio.create_task_group() as tg:
        tg.start_soon(task_a)
        tg.start_soon(task_b)

# Run with asyncio backend
anyio.run(main)

# Run with trio backend
anyio.run(main, backend="trio")
```

---

## Testing Async Code

```python
import asyncio
import pytest

# pytest-anyio (preferred over pytest-asyncio for new projects)
import anyio
import pytest

@pytest.mark.anyio
async def test_fetch() -> None:
    result = await fetch("https://example.com")
    assert result.status_code == 200

# With pytest-asyncio
@pytest.mark.asyncio
async def test_task_group() -> None:
    async with asyncio.TaskGroup() as tg:
        t1 = tg.create_task(asyncio.sleep(0))
    assert t1.done()
```

---

## Async Context Managers and Iterators

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def db_transaction(conn):
    async with conn.transaction():
        try:
            yield conn
        except Exception:
            await conn.rollback()
            raise

# Async generators / iterators
async def paginate(url: str) -> AsyncIterator[list[dict]]:
    page = 1
    while True:
        data = await fetch_page(url, page)
        if not data:
            break
        yield data
        page += 1

async def consume() -> None:
    async for page in paginate("https://api.example.com/items"):
        process(page)
```
