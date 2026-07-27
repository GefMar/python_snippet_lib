---
title: "Plan a Bounded Notebook Storage Key with Collision Suggestions"
snippet_type: recipe
use_cases:
  - security
  - validation
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - validate-a-conservative-unicode-filename-component.md
  - ../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md
  - ../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md
---

# Plan a Bounded Notebook Storage Key with Collision Suggestions

## Idea and Problem

Validate one raw relative POSIX notebook path and plan a namespaced storage key against an immutable snapshot of occupied keys.

The planner accepts only an exact lowercase `.ipynb` suffix and rejects
absolute paths, traversal, dot components, repeated separators, backslashes,
non-normalized components, and values outside fixed depth and byte limits. If
the requested key is occupied, it searches a fixed `-N` window and returns up
to five available suggestions without touching a filesystem or object store.
For a canonical one-to-nine-digit suffix such as `report-7.ipynb`, the search
continues at `report-8.ipynb` instead of nesting another suffix. A name already
at the numeric ceiling has no generated successor.

## When to Use

Use this recipe after a caller has taken a bounded immutable snapshot of logical
`PurePosixPath` storage keys and needs a deterministic advisory name for a
bounded notebook upload. It is suitable for user-facing collision feedback
before a separate storage operation. A namespace is exactly one validated key
component, while the supplied path is relative to that namespace.

This is not an authorization check or a reservation. Validate the caller's
permission and notebook content elsewhere, and use a conditional object-store
write or an exclusive filesystem create when publishing the selected key.

## Implementation

```python
import re
import unicodedata
from dataclasses import dataclass
from pathlib import PurePosixPath


_MAX_RELATIVE_DEPTH = 8
_MAX_COMPONENT_UTF8_BYTES = 128
_MAX_KEY_UTF8_BYTES = 512
_MAX_NOTEBOOK_BYTES = 100 * 1024 * 1024
_MAX_OCCUPIED_KEYS = 10_000
_COLLISION_SEARCH_WINDOW = 100
_MAX_SUGGESTIONS = 5
_MAX_COLLISION_NUMBER = 999_999_999
_NAMESPACE = re.compile(
    r"[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?",
    re.ASCII,
)


@dataclass(frozen=True, slots=True)
class NotebookStoragePlan:
    namespace: str
    relative_path: PurePosixPath
    requested_key: PurePosixPath
    size_bytes: int
    available: bool
    collision_suggestions: tuple[PurePosixPath, ...]


def _validated_component(value: str, *, name: str) -> str:
    if not isinstance(value, str):
        raise TypeError(f"{name} must be text")
    if not value or value in {".", ".."}:
        raise ValueError(f"{name} must be a non-dot component")
    if "/" in value or "\\" in value:
        raise ValueError(f"{name} must be one POSIX key component")
    if value != value.strip() or not value.isprintable():
        raise ValueError(f"{name} must be stripped printable text")
    if unicodedata.normalize("NFC", value) != value:
        raise ValueError(f"{name} must already use NFC normalization")
    if any(unicodedata.category(character).startswith("C") for character in value):
        raise ValueError(f"{name} contains a disallowed Unicode category")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError as error:
        raise ValueError(f"{name} must be valid UTF-8 text") from error
    if len(encoded) > _MAX_COMPONENT_UTF8_BYTES:
        raise ValueError(f"{name} exceeds the component byte limit")
    return value


def _validated_relative_notebook_path(raw_path: str) -> PurePosixPath:
    if not isinstance(raw_path, str):
        raise TypeError("raw_path must be text")
    if not raw_path or raw_path.startswith("/"):
        raise ValueError("raw_path must be a non-empty relative POSIX path")
    if "\\" in raw_path:
        raise ValueError("raw_path must not contain backslashes")
    if "//" in raw_path:
        raise ValueError("raw_path must not contain repeated separators")

    raw_components = raw_path.split("/")
    if not 1 <= len(raw_components) <= _MAX_RELATIVE_DEPTH:
        raise ValueError("raw_path depth is outside the supported range")
    components = tuple(
        _validated_component(component, name="raw_path component")
        for component in raw_components
    )
    filename = components[-1]
    if not filename.endswith(".ipynb") or filename == ".ipynb":
        raise ValueError("raw_path must end with the exact .ipynb suffix")
    return PurePosixPath(*components)


def _validated_namespace(value: str) -> str:
    if not isinstance(value, str):
        raise TypeError("namespace must be text")
    if _NAMESPACE.fullmatch(value) is None:
        raise ValueError("namespace must be a canonical lowercase ASCII identifier")
    return value


def _validate_key_size(key: PurePosixPath) -> None:
    if len(str(key).encode("utf-8")) > _MAX_KEY_UTF8_BYTES:
        raise ValueError("storage key exceeds the supported UTF-8 byte limit")


def _occupied_snapshot(
    occupied_keys: tuple[PurePosixPath, ...] | frozenset[PurePosixPath],
) -> frozenset[PurePosixPath]:
    if type(occupied_keys) not in (tuple, frozenset):
        raise TypeError("occupied_keys must be an immutable tuple or frozenset")
    if len(occupied_keys) > _MAX_OCCUPIED_KEYS:
        raise ValueError("occupied_keys exceeds the supported entry limit")

    snapshot = tuple(occupied_keys)
    if len(set(snapshot)) != len(snapshot):
        raise ValueError("occupied_keys must not contain duplicate keys")
    for key in snapshot:
        if type(key) is not PurePosixPath:
            raise TypeError("every occupied key must be a PurePosixPath")
        if key.is_absolute() or not key.parts:
            raise ValueError("occupied keys must be non-empty relative paths")
        if len(key.parts) > _MAX_RELATIVE_DEPTH + 1:
            raise ValueError("an occupied key exceeds the supported depth")
        for component in key.parts:
            _validated_component(component, name="occupied key component")
        _validate_key_size(key)
    return frozenset(snapshot)


def _available_suggestions(
    requested_key: PurePosixPath,
    occupied: frozenset[PurePosixPath],
) -> tuple[PurePosixPath, ...]:
    stem = requested_key.name.removesuffix(".ipynb")
    base, separator, numeric_suffix = stem.rpartition("-")
    if (
        separator
        and base
        and numeric_suffix.isascii()
        and numeric_suffix.isdecimal()
        and len(numeric_suffix) <= 9
        and (numeric_suffix == "0" or not numeric_suffix.startswith("0"))
    ):
        stem = base
        first_number = int(numeric_suffix) + 1
    else:
        first_number = 1
    suggestions = []
    for number in range(
        first_number,
        first_number + _COLLISION_SEARCH_WINDOW,
    ):
        if number > _MAX_COLLISION_NUMBER:
            break
        filename = f"{stem}-{number}.ipynb"
        if len(filename.encode("utf-8")) > _MAX_COMPONENT_UTF8_BYTES:
            continue
        suggestion = requested_key.parent / filename
        if len(str(suggestion).encode("utf-8")) > _MAX_KEY_UTF8_BYTES:
            continue
        if suggestion not in occupied:
            suggestions.append(suggestion)
            if len(suggestions) == _MAX_SUGGESTIONS:
                break
    return tuple(suggestions)


def plan_notebook_storage_key(
    namespace: str,
    raw_path: str,
    *,
    size_bytes: int,
    occupied_keys: tuple[PurePosixPath, ...] | frozenset[PurePosixPath],
) -> NotebookStoragePlan:
    namespace = _validated_namespace(namespace)
    relative_path = _validated_relative_notebook_path(raw_path)
    if isinstance(size_bytes, bool) or not isinstance(size_bytes, int):
        raise TypeError("size_bytes must be an integer")
    if not 0 <= size_bytes <= _MAX_NOTEBOOK_BYTES:
        raise ValueError("size_bytes is outside the supported range")

    occupied = _occupied_snapshot(occupied_keys)
    requested_key = PurePosixPath(namespace) / relative_path
    _validate_key_size(requested_key)
    available = requested_key not in occupied
    suggestions = () if available else _available_suggestions(requested_key, occupied)
    return NotebookStoragePlan(
        namespace=namespace,
        relative_path=relative_path,
        requested_key=requested_key,
        size_bytes=size_bytes,
        available=available,
        collision_suggestions=suggestions,
    )
```

## Example

```python
occupied = frozenset(
    {
        PurePosixPath("workspace-a/reports/weekly.ipynb"),
        PurePosixPath("workspace-a/reports/weekly-1.ipynb"),
        PurePosixPath("workspace-a/reports/weekly-3.ipynb"),
    }
)
plan = plan_notebook_storage_key(
    "workspace-a",
    "reports/weekly.ipynb",
    size_bytes=4096,
    occupied_keys=occupied,
)
available_plan = plan_notebook_storage_key(
    "workspace-a",
    "reports/monthly.ipynb",
    size_bytes=0,
    occupied_keys=tuple(occupied),
)
ceiling_key = PurePosixPath(
    "workspace-a/reports/archive-999999999.ipynb"
)
ceiling_plan = plan_notebook_storage_key(
    "workspace-a",
    "reports/archive-999999999.ipynb",
    size_bytes=1,
    occupied_keys=(ceiling_key,),
)

invalid_inputs = (
    ("/weekly.ipynb", 1),
    ("reports//weekly.ipynb", 1),
    ("reports/./weekly.ipynb", 1),
    ("reports/../weekly.ipynb", 1),
    ("reports\\weekly.ipynb", 1),
    ("reports/weekly.IPYNB", 1),
    ("reports/weekly.ipynb", True),
)
rejected = []
for raw_path, size_bytes in invalid_inputs:
    try:
        plan_notebook_storage_key(
            "workspace-a",
            raw_path,
            size_bytes=size_bytes,
            occupied_keys=occupied,
        )
    except (TypeError, ValueError):
        rejected.append(True)

assert (
    plan,
    available_plan.available,
    available_plan.collision_suggestions,
    ceiling_plan.collision_suggestions,
    len(rejected),
    len(occupied),
) == (
    NotebookStoragePlan(
        namespace="workspace-a",
        relative_path=PurePosixPath("reports/weekly.ipynb"),
        requested_key=PurePosixPath("workspace-a/reports/weekly.ipynb"),
        size_bytes=4096,
        available=False,
        collision_suggestions=(
            PurePosixPath("workspace-a/reports/weekly-2.ipynb"),
            PurePosixPath("workspace-a/reports/weekly-4.ipynb"),
            PurePosixPath("workspace-a/reports/weekly-5.ipynb"),
            PurePosixPath("workspace-a/reports/weekly-6.ipynb"),
            PurePosixPath("workspace-a/reports/weekly-7.ipynb"),
        ),
    ),
    True,
    (),
    (),
    7,
    3,
)
```

## Trade-offs and Limitations

The occupied-key validation and snapshot copy use `O(n)` time and memory, and
the collision search performs at most one hundred set lookups. Suggestions can
be fewer than five when names in the fixed window are occupied, would exceed a
byte limit, or reach the nine-digit numeric ceiling. NFC is required rather
than applied silently, and the logical POSIX policy does not promise
compatibility with every filesystem or object store.

The snapshot can become stale immediately. This planner does not authorize a
namespace, parse notebook content, detect archive payloads, inspect symlinks,
reserve a key, or close a check/use race. Recheck policy at the storage boundary
and publish with conditional object creation or an exclusive create operation;
never treat a suggestion as proof that a later write is safe.

## Related Snippets

<!-- catalog:related:start -->
- [Validate a Conservative Unicode Filename Component](validate-a-conservative-unicode-filename-component.md)
- [Replace a File Atomically with a Sibling Temporary File](../storage-databases/replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Split a Binary Stream into Exclusively Created Numbered Parts](../storage-databases/split-a-binary-stream-into-exclusively-created-numbered-parts.md)
<!-- catalog:related:end -->
