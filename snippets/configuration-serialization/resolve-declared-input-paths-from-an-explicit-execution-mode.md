---
title: "Resolve Declared Input Paths from an Explicit Execution Mode"
snippet_type: pattern
use_cases:
  - configuration
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - normalize-bounded-named-options-with-explicit-default-semantics.md
  - reject-unknown-options-with-conservative-typo-suggestions.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Resolve Declared Input Paths from an Explicit Execution Mode

## Idea and Problem

Select one complete immutable input snapshot from a closed execution mode, then return frozen path bindings in declared-name order.

The caller supplies exactly one managed or local snapshot. Selection depends
only on the explicit enum member: an empty snapshot is still supplied state,
and an unused empty snapshot is therefore not equivalent to absence. The
selected snapshot must have the exact expected key set before any binding is
returned.

## When to Use

Use this pattern at a small in-memory configuration boundary where an upstream
managed runner or local launcher has already collected all declared input
paths. Snapshot entries may arrive in any order, while the expected-name tuple
defines stable output order. Paths use a canonical lexical POSIX grammar and
may be absolute or relative without being resolved against a working directory.

The function accepts exact immutable records and tuples. It does not coerce raw
strings into modes, booleans into flags, path-like objects into strings, or
mutable mappings into snapshots.

## Implementation

```python
import re
from dataclasses import dataclass
from enum import StrEnum
from pathlib import PurePosixPath

_MAX_EXPECTED_INPUTS = 20
_MAX_SNAPSHOT_ENTRIES = 24
_MAX_PATH_DEPTH = 18
_MAX_COMPONENT_BYTES = 128
_MAX_PATH_BYTES = 768
_MAX_AGGREGATE_PATH_BYTES = 6_144
_INPUT_NAME = re.compile(r"[a-z][a-z0-9_]{0,39}", re.ASCII).fullmatch


class InputResolutionError(ValueError):
    pass


class ExecutionMode(StrEnum):
    MANAGED = "managed"
    LOCAL = "local"


@dataclass(frozen=True, slots=True)
class InputSnapshot:
    entries: tuple[tuple[str, str], ...]


@dataclass(frozen=True, slots=True)
class InputBinding:
    name: str
    path: PurePosixPath


class _AbsentSnapshot:
    __slots__ = ()


_ABSENT = _AbsentSnapshot()


def _checked_input_name(value: object) -> str:
    if type(value) is not str:
        raise TypeError("input names must be exact strings")
    if _INPUT_NAME(value) is None:
        raise InputResolutionError("an input name has invalid syntax")
    return value


def _checked_path(value: object) -> tuple[PurePosixPath, int]:
    if type(value) is not str:
        raise TypeError("input path values must be exact strings")
    try:
        encoded = value.encode("utf-8")
    except UnicodeEncodeError:
        raise InputResolutionError("an input path is not valid UTF-8") from None
    if not 1 <= len(encoded) <= _MAX_PATH_BYTES:
        raise InputResolutionError("an input path exceeds its byte bound")
    if value.startswith("//") or value.endswith("/"):
        raise InputResolutionError("an input path is not canonical")
    if "\x00" in value or "\\" in value:
        raise InputResolutionError("an input path contains a forbidden character")
    if any(ord(character) < 32 or ord(character) == 127 for character in value):
        raise InputResolutionError("an input path contains a control character")

    remainder = value[1:] if value.startswith("/") else value
    parts = tuple(remainder.split("/"))
    if len(parts) > _MAX_PATH_DEPTH:
        raise InputResolutionError("an input path exceeds its depth bound")
    if any(part in ("", ".", "..") for part in parts):
        raise InputResolutionError("an input path is empty, traversing, or not canonical")
    for part in parts:
        try:
            part_size = len(part.encode("utf-8"))
        except UnicodeEncodeError:
            raise InputResolutionError("an input path component is not valid UTF-8") from None
        if part_size > _MAX_COMPONENT_BYTES:
            raise InputResolutionError("an input path component exceeds its byte bound")

    path = PurePosixPath(value)
    if path.as_posix() != value:
        raise InputResolutionError("an input path is not canonical")
    return path, len(encoded)


def _select_snapshot(
    mode: ExecutionMode,
    managed_snapshot: InputSnapshot | _AbsentSnapshot,
    local_snapshot: InputSnapshot | _AbsentSnapshot,
) -> InputSnapshot:
    managed_supplied = managed_snapshot is not _ABSENT
    local_supplied = local_snapshot is not _ABSENT
    if managed_supplied == local_supplied:
        raise InputResolutionError("exactly one input snapshot must be supplied")

    if mode is ExecutionMode.MANAGED:
        selected = managed_snapshot
        unused = local_snapshot
    else:
        selected = local_snapshot
        unused = managed_snapshot
    if selected is _ABSENT or unused is not _ABSENT:
        raise InputResolutionError("the supplied snapshot does not match the execution mode")
    if type(selected) is not InputSnapshot:
        raise TypeError("the selected snapshot must be an exact InputSnapshot record")
    return selected


def resolve_declared_input_paths(
    mode: ExecutionMode,
    expected_names: tuple[str, ...],
    *,
    managed_snapshot: InputSnapshot | _AbsentSnapshot = _ABSENT,
    local_snapshot: InputSnapshot | _AbsentSnapshot = _ABSENT,
) -> tuple[InputBinding, ...]:
    if type(mode) is not ExecutionMode:
        raise TypeError("mode must be an exact ExecutionMode member")
    if type(expected_names) is not tuple:
        raise TypeError("expected names must be an exact tuple")
    if len(expected_names) > _MAX_EXPECTED_INPUTS:
        raise InputResolutionError("expected input count exceeds its bound")

    expected: list[str] = []
    expected_set: set[str] = set()
    for candidate in expected_names:
        name = _checked_input_name(candidate)
        if name in expected_set:
            raise InputResolutionError("expected input names must be unique")
        expected.append(name)
        expected_set.add(name)

    snapshot = _select_snapshot(mode, managed_snapshot, local_snapshot)
    if type(snapshot.entries) is not tuple:
        raise TypeError("snapshot entries must be an exact tuple")
    if len(snapshot.entries) > _MAX_SNAPSHOT_ENTRIES:
        raise InputResolutionError("snapshot entry count exceeds its bound")

    paths_by_name: dict[str, PurePosixPath] = {}
    aggregate_bytes = 0
    for entry in snapshot.entries:
        if type(entry) is not tuple or len(entry) != 2:
            raise TypeError("snapshot entries must be exact name-path pairs")
        name = _checked_input_name(entry[0])
        if name in paths_by_name:
            raise InputResolutionError("snapshot input names must be unique")
        path, path_bytes = _checked_path(entry[1])
        aggregate_bytes += path_bytes
        if aggregate_bytes > _MAX_AGGREGATE_PATH_BYTES:
            raise InputResolutionError("snapshot paths exceed their aggregate byte bound")
        paths_by_name[name] = path

    if set(paths_by_name) != expected_set:
        raise InputResolutionError("snapshot keys must match expected names exactly")
    return tuple(InputBinding(name, paths_by_name[name]) for name in expected)
```

## Example

```python
declared = ("rules_file", "event_stream", "lookup_table")
managed = InputSnapshot(
    (
        ("lookup_table", "/jobs/cedar/input/codes.csv"),
        ("rules_file", "/jobs/cedar/input/policy.toml"),
        ("event_stream", "/jobs/cedar/input/events.ndjson"),
    )
)

bindings = resolve_declared_input_paths(
    ExecutionMode.MANAGED,
    declared,
    managed_snapshot=managed,
)
assert tuple(binding.name for binding in bindings) == declared
assert tuple(binding.path.as_posix() for binding in bindings) == (
    "/jobs/cedar/input/policy.toml",
    "/jobs/cedar/input/events.ndjson",
    "/jobs/cedar/input/codes.csv",
)

try:
    resolve_declared_input_paths(
        ExecutionMode.MANAGED,
        declared,
        managed_snapshot=managed,
        local_snapshot=InputSnapshot(()),
    )
except InputResolutionError:
    supplied_unused_empty_rejected = True
else:
    supplied_unused_empty_rejected = False

assert supplied_unused_empty_rejected
assert bindings[0] == InputBinding(
    "rules_file",
    PurePosixPath("/jobs/cedar/input/policy.toml"),
)
```

## Trade-offs and Limitations

For `n` bounded declarations and total encoded path size `b`, validation and
allocation take `O(n + b)` expected time and `O(n + b)` memory. Exact tuple and
record checks prevent mutable or coercible inputs from crossing the boundary.
Counts, path depth, component bytes, individual UTF-8 path bytes, and aggregate
path bytes are all bounded.

This is a lexical selection boundary, not an input loader. It never tests file
existence, reads the filesystem or current working directory, resolves
symlinks, invokes callbacks, emits warnings, or falls back to a partial
snapshot. It neither opens nor closes resources; authorization, path
interpretation, loading, and lifecycle ownership remain with the caller.

## Related Snippets

<!-- catalog:related:start -->
- [Normalize Bounded Named Options with Explicit Default Semantics](normalize-bounded-named-options-with-explicit-default-semantics.md)
- [Reject Unknown Options with Conservative Typo Suggestions](reject-unknown-options-with-conservative-typo-suggestions.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
