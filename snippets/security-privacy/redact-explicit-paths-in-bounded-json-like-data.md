---
title: "Redact Explicit Paths in Bounded JSON-Like Data"
snippet_type: recipe
use_cases:
  - data-transformation
  - observability
  - security
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md
  - ../configuration-serialization/get-nested-values-with-a-validated-dot-path.md
  - ../observability-operations/format-log-records-as-json-with-explicit-extra-fields.md
---

# Redact Explicit Paths in Bounded JSON-Like Data

## Idea and Problem

Copy a bounded JSON-like value and replace leaves at explicitly tokenized field paths without retaining any part of the original values.

A dedicated `EACH_ITEM` token traverses lists without overloading punctuation
inside field names. Each configured path reports its match count, while strict
copying rejects cycles, unsupported objects, non-string mapping keys, and
excessive structures before the redacted result is exposed.

## When to Use

Use this recipe at a narrow logging or diagnostic boundary when the allowed
JSON-like shape is known and a reviewed set of exact paths identifies values
that must be masked. Configure paths in code, treat missing paths as observable
schema drift, and decide whether a zero match should fail the surrounding
operation.

Prefer constructing telemetry from an allowlist of safe fields. Redaction is a
secondary control for a value that is already in memory; it does not discover
secrets, erase originals, or make arbitrary application objects safe to log.

## Implementation

```python
import math
from dataclasses import dataclass
from typing import TypeAlias


_MAX_PATHS = 64
_MAX_PATH_DEPTH = 16
_MAX_FIELD_LENGTH = 256
_MAX_CONTAINER_DEPTH = 64
_MAX_NODES = 100_000


class RedactionError(ValueError):
    pass


class RedactionLimitError(RedactionError):
    pass


class _EachItem:
    __slots__ = ()


EACH_ITEM = _EachItem()
PathToken: TypeAlias = str | _EachItem
RedactionPath: TypeAlias = tuple[PathToken, ...]


@dataclass(frozen=True, slots=True)
class PathOutcome:
    path: RedactionPath
    matches: int


@dataclass(frozen=True, slots=True)
class RedactionResult:
    value: object
    outcomes: tuple[PathOutcome, ...]

    @property
    def total_matches(self) -> int:
        return sum(outcome.matches for outcome in self.outcomes)

    @property
    def missing_path_indexes(self) -> tuple[int, ...]:
        return tuple(
            index
            for index, outcome in enumerate(self.outcomes)
            if outcome.matches == 0
        )


def _validate_redaction_limits(max_nodes: int, max_depth: int) -> None:
    for name, value, lower, upper in (
        ("max_nodes", max_nodes, 1, _MAX_NODES),
        ("max_depth", max_depth, 0, _MAX_CONTAINER_DEPTH),
    ):
        if type(value) is not int:
            raise TypeError(f"{name} must be an integer")
        if not lower <= value <= upper:
            raise ValueError(f"{name} is outside the supported range")


def _validate_paths(paths: tuple[RedactionPath, ...]) -> None:
    if type(paths) is not tuple:
        raise TypeError("paths must be a tuple")
    if not 1 <= len(paths) <= _MAX_PATHS:
        raise RedactionLimitError("path count is outside the supported range")

    for path in paths:
        if type(path) is not tuple:
            raise TypeError("each path must be a tuple")
        if not 1 <= len(path) <= _MAX_PATH_DEPTH:
            raise RedactionLimitError("path depth is outside the supported range")
        for token in path:
            if token is EACH_ITEM:
                continue
            if type(token) is not str:
                raise TypeError("path tokens must be field names or EACH_ITEM")
            if not 1 <= len(token) <= _MAX_FIELD_LENGTH:
                raise RedactionLimitError(
                    "field-name length is outside the supported range"
                )

    for index, path in enumerate(paths):
        for other in paths[index + 1 :]:
            shared = min(len(path), len(other))
            if path[:shared] == other[:shared]:
                raise RedactionError("paths must not duplicate or contain each other")


def redact_json_paths(
    value: object,
    paths: tuple[RedactionPath, ...],
    *,
    marker: str = "[REDACTED]",
    max_nodes: int = 10_000,
    max_depth: int = 32,
) -> RedactionResult:
    _validate_paths(paths)
    _validate_redaction_limits(max_nodes, max_depth)
    if type(marker) is not str:
        raise TypeError("marker must be text")
    if not 1 <= len(marker) <= 128:
        raise ValueError("marker length is outside the supported range")

    active_containers: set[int] = set()
    nodes_seen = 0

    def clone(current: object, depth: int) -> object:
        nonlocal nodes_seen
        nodes_seen += 1
        if nodes_seen > max_nodes:
            raise RedactionLimitError("max_nodes was exceeded")

        if current is None or type(current) in (bool, int, str):
            return current
        if type(current) is float:
            if not math.isfinite(current):
                raise RedactionError("non-finite numbers are not supported")
            return current
        if type(current) not in (list, dict):
            raise RedactionError("the value contains a non-JSON-like object")
        if depth > max_depth:
            raise RedactionLimitError("max_depth was exceeded")

        identity = id(current)
        if identity in active_containers:
            raise RedactionError("the value contains a container cycle")
        active_containers.add(identity)
        try:
            if type(current) is list:
                return [clone(item, depth + 1) for item in current]
            if any(type(key) is not str for key in current):
                raise RedactionError("mapping keys must be text")
            return {
                key: clone(item, depth + 1)
                for key, item in current.items()
            }
        finally:
            active_containers.remove(identity)

    copied = clone(value, 0)

    def apply_path(current: object, path: RedactionPath, index: int) -> int:
        token = path[index]
        final = index == len(path) - 1
        if token is EACH_ITEM:
            if type(current) is not list:
                raise RedactionError("a list wildcard reached a non-list value")
            if final:
                matches = len(current)
                current[:] = [marker] * matches
                return matches
            return sum(
                apply_path(item, path, index + 1)
                for item in current
            )

        if type(current) is not dict:
            raise RedactionError("a field token reached a non-object value")
        if token not in current:
            return 0
        if final:
            current[token] = marker
            return 1
        return apply_path(current[token], path, index + 1)

    outcomes = tuple(
        PathOutcome(path=path, matches=apply_path(copied, path, 0))
        for path in paths
    )
    return RedactionResult(value=copied, outcomes=outcomes)
```

## Example

```python
payload = {
    "profile": {"name": "Ada", "token": "alpha-secret"},
    "items": [
        {"token": "first-secret", "count": 1},
        {"token": "second-secret", "count": 2},
    ],
}
result = redact_json_paths(
    payload,
    (
        ("profile", "token"),
        ("items", EACH_ITEM, "token"),
        ("optional",),
    ),
)

cycle: list[object] = []
cycle.append(cycle)
try:
    redact_json_paths(cycle, ((EACH_ITEM,),))
except RedactionError:
    cycle_rejected = True
else:
    cycle_rejected = False


class CustomDict(dict[str, object]):
    pass


try:
    redact_json_paths(CustomDict(token="value"), (("token",),))
except RedactionError:
    custom_container_rejected = True
else:
    custom_container_rejected = False

assert (
    result.value,
    tuple(outcome.matches for outcome in result.outcomes),
    result.total_matches,
    result.missing_path_indexes,
    payload["profile"]["token"],
    cycle_rejected,
    custom_container_rejected,
) == (
    {
        "profile": {"name": "Ada", "token": "[REDACTED]"},
        "items": [
            {"token": "[REDACTED]", "count": 1},
            {"token": "[REDACTED]", "count": 2},
        ],
    },
    (1, 2, 0),
    3,
    (2,),
    "alpha-secret",
    True,
    True,
)
```

## Trade-offs and Limitations

The full JSON-like structure is copied, so time and memory are `O(n)` before
path traversal. Applying up to 64 paths can revisit containers, and list
wildcards can multiply matches; the node cap bounds copying, not the number of
path visits. Missing fields are reported with a zero count, while reaching the
wrong container shape is an error. Prefix-overlapping paths are rejected so an
earlier replacement cannot change the meaning of a later path.

The fixed marker hides the value but still reveals the field, surrounding
shape, list length, and the fact that redaction occurred. Original values stay
in the input object and in process memory. Shared acyclic containers become
independent copies, tuples and custom mappings are rejected, and no serializer
is invoked on application objects. An explicit path list can become stale, so
prefer safe-field construction and monitor missing outcomes.

## Related Snippets

<!-- catalog:related:start -->
- [Redact a Printable ASCII Secret with a Bounded Visible Tail](redact-a-printable-ascii-secret-with-a-bounded-visible-tail.md)
- [Get Nested Values with a Validated Dot Path](../configuration-serialization/get-nested-values-with-a-validated-dot-path.md)
- [Format Log Records as JSON with Explicit Extra Fields](../observability-operations/format-log-records-as-json-with-explicit-extra-fields.md)
<!-- catalog:related:end -->
