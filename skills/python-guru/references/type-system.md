# Type System Reference

Modern Python type annotations as of 3.12/3.13. Use `from __future__ import annotations`
on 3.10 to 3.13 for forward references; not needed on 3.14+ (deferred by default).

---

## PEP 695: Inline Type Parameters (3.12+)

```python
# OLD — verbose, pollutes module namespace
from typing import TypeVar, Generic
T = TypeVar("T")
KT = TypeVar("KT")
VT = TypeVar("VT")

def first(items: list[T]) -> T: ...
class Stack(Generic[T]): ...

# NEW — inline, explicit, lazy-evaluated
def first[T](items: list[T]) -> T: ...

class Stack[T]:
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...

# Bounded type parameters
def largest[T: (int, float, str)](items: list[T]) -> T: ...

# With constraints
def compare[T: int | str](a: T, b: T) -> bool: ...

# Type aliases (replaces TypeAlias annotation)
type Vector = list[float]
type Matrix[T] = list[list[T]]
type IntOrStr = int | str
type Callback[**P, R] = Callable[P, R]

# ParamSpec still works with new syntax
def logged[**P, R](f: Callable[P, R]) -> Callable[P, R]: ...
```

**Anti-pattern:** Writing `T = TypeVar("T")` at module level for new code targeting 3.12+.

---

## Union Syntax

```python
# PREFERRED (3.10+)
def process(value: int | str | None) -> str | None: ...

# AVOID for new code
from typing import Optional, Union
def process(value: Optional[str]) -> Optional[str]: ...
def process(value: Union[int, str]) -> Union[str, None]: ...

# isinstance() with unions (3.10+)
if isinstance(x, int | str):
    ...
```

---

## Built-in Generic Types (3.9+)

```python
# Use built-in generics — never import from typing for these
def sort(items: list[int]) -> list[int]: ...
def lookup(mapping: dict[str, int], key: str) -> int: ...
def ordered(items: tuple[int, str, float]) -> None: ...
def unique(items: set[str]) -> frozenset[str]: ...

# collections.abc for abstract types (not typing)
from collections.abc import Sequence, Mapping, Callable, Iterator, Generator, Awaitable

def process(items: Sequence[int]) -> Iterator[str]: ...
def apply(fn: Callable[[int, str], bool]) -> None: ...
```

---

## `Self`: Fluent/Builder Returns (3.11+)

```python
from typing import Self

class QueryBuilder:
    def where(self, condition: str) -> Self:
        self._conditions.append(condition)
        return self

    def limit(self, n: int) -> Self:
        self._limit = n
        return self

# Anti-pattern: TypeVar just for self-return
T = TypeVar("T", bound="QueryBuilder")
def where(self: T, ...) -> T: ...   # verbose, use Self instead
```

---

## `@override`: Intentional Overrides (3.12+)

```python
from typing import override

class Base:
    def serialize(self) -> bytes: ...
    def validate(self) -> bool: ...

class Derived(Base):
    @override
    def serialize(self) -> bytes: ...   # type checker errors if Base has no .serialize()

    def seralize(self) -> bytes: ...    # typo! @override would catch this
```

**Rule:** Add `@override` to every method that overrides a parent. This is the primary defence
against refactoring regressions where parent method names change.

---

## `TypeIs` vs `TypeGuard` (3.13+ / 3.10+)

```python
from typing import TypeIs   # 3.13+

# TypeIs: narrows BOTH branches (True → narrowed, False → remainder)
def is_str_list(val: list[object]) -> TypeIs[list[str]]:
    return all(isinstance(x, str) for x in val)

if is_str_list(items):
    items  # list[str]
else:
    items  # list[object] (remainder — TypeIs narrows the else branch too)

# TypeGuard (3.10): only narrows the True branch, else stays original type
from typing import TypeGuard
def is_str(val: object) -> TypeGuard[str]:
    return isinstance(val, str)
```

**Rule:** Prefer `TypeIs` for new type guards (3.13+). Use `TypeGuard` only when the `False`
branch narrowing of `TypeIs` is incorrect for your use case.

---

## `Protocol`: Structural Subtyping

```python
from typing import Protocol, runtime_checkable

# Prefer Protocol over ABC when:
# - You want structural (duck) typing
# - You don't want to inherit from a base class
# - You're defining an interface for external types you don't control
@runtime_checkable   # enables isinstance() checks; adds overhead, use only when needed
class Serializable(Protocol):
    def to_bytes(self) -> bytes: ...
    def from_bytes(self, data: bytes) -> None: ...

def save(obj: Serializable) -> None:
    storage.write(obj.to_bytes())

# Any class with these methods satisfies the protocol — no inheritance needed
class MyModel:
    def to_bytes(self) -> bytes: return b"..."
    def from_bytes(self, data: bytes) -> None: ...

save(MyModel())  # type-checks correctly
```

---

## `TypedDict`: Typed Dictionaries

```python
from typing import TypedDict, Required, NotRequired, ReadOnly  # ReadOnly: 3.13+

class Config(TypedDict):
    host: str
    port: int
    debug: NotRequired[bool]   # optional key
    secret: ReadOnly[str]      # type checkers enforce no mutation

# Total=False makes all keys optional
class Overrides(TypedDict, total=False):
    host: str
    port: int

# Inheritance for partial updates
class BaseConfig(TypedDict):
    host: Required[str]

# **kwargs with full typing (3.12+)
from typing import Unpack

def connect(**kwargs: Unpack[Config]) -> None: ...
```

---

## `@dataclass`: Preferred Patterns

```python
from dataclasses import dataclass, field
from typing import ClassVar

@dataclass(slots=True, frozen=True)   # slots: less memory, faster; frozen: immutable
class Point:
    x: float
    y: float

    # ClassVar NOT included in __init__
    DIMENSIONS: ClassVar[int] = 2

@dataclass(slots=True)
class Event:
    name: str
    tags: list[str] = field(default_factory=list)   # mutable defaults need field()
    _cache: dict = field(default_factory=dict, repr=False, compare=False)

# copy.replace() for "updating" frozen dataclasses (3.13+)
from copy import replace
new_point = replace(point, x=1.0)

# kw_only=True forces keyword-only init (prevents positional confusion)
@dataclass(kw_only=True)
class ServiceConfig:
    host: str
    port: int = 8080
    timeout: float = 30.0
```

**Rules:**
- Always `slots=True` unless you add attributes dynamically
- Always `frozen=True` for value objects (coordinates, IDs, config)
- Never use mutable default values: always `field(default_factory=...)`

---

## `@final` and `assert_never`

```python
from typing import final, Never, assert_never

@final
class SessionToken:
    """Cannot be subclassed — type checker + runtime enforcement."""
    ...

# Exhaustiveness checking — ensure all union cases are handled
type Shape = Circle | Rectangle | Triangle

def area(shape: Shape) -> float:
    match shape:
        case Circle(r=r):
            return 3.14 * r ** 2
        case Rectangle(w=w, h=h):
            return w * h
        case Triangle():
            return ...
        case _ as unreachable:
            assert_never(unreachable)   # type error if Shape gains a new variant
```

---

## `LiteralString`: Injection Prevention (3.11+)

```python
from typing import LiteralString

def run_query(query: LiteralString, params: tuple) -> list:
    """Only accepts compile-time string literals — not user-controlled strings."""
    cursor.execute(query, params)

# Type checker flags this — user_input is not LiteralString
run_query(f"SELECT * FROM {user_input}", ())  # ERROR
run_query("SELECT * FROM users WHERE id = ?", (user_id,))  # OK
```

---

## `warnings.deprecated` (3.13+)

```python
from warnings import deprecated

@deprecated("Use process_v2() instead. Will be removed in v3.0.")
def process(data: bytes) -> str:
    ...

class LegacyAPI:
    @deprecated("Migrate to NewAPI.fetch()")
    def get(self) -> dict: ...
```

---

## Import Organisation

```python
# Correct import structure for a module with type annotations
from __future__ import annotations   # on 3.10-3.13; omit on 3.14+

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    # These imports only happen at type-check time, not at runtime
    # Avoids circular imports and speeds up startup
    from collections.abc import Sequence
    from mypackage.heavy_module import HeavyClass

# Runtime imports go outside TYPE_CHECKING
from pathlib import Path
```

**Rule:** Use `TC` ruff rule to automatically flag and fix imports that should be in
`TYPE_CHECKING`. This is especially important for heavy imports used only in annotations.

---

## Common Type Annotation Gotchas

| Mistake | Correct |
|---|---|
| `list[int, str]` | `list[int \| str]` (generic takes one arg) |
| `tuple[int]` (any length) | `tuple[int, ...]` (homogeneous variable-length) |
| `Callable[int, str]` | `Callable[[int], str]` (args must be a list) |
| `dict[str]` | `dict[str, Any]` or be specific |
| `type(x) is list` | `isinstance(x, list)` |
| Missing `-> None` on procedures | Always annotate return type, even `None` |
| Annotating `self: Type` | Only `self` (no annotation needed; use `Self` for return) |
