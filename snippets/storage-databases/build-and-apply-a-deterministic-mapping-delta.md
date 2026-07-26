---
title: "Build and Apply a Deterministic Mapping Delta"
snippet_type: algorithm
use_cases:
  - data-transformation
  - persistence
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md
  - ../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Build and Apply a Deterministic Mapping Delta

## Idea and Problem

Represent the exact differences between two string-key mappings as sorted set and delete operations that can be applied to a shallow copy.

A deterministic delta is useful when complete snapshots are easy to compare
but wasteful to transmit or retain repeatedly. Membership tests distinguish an
absent key from a key whose stored value is `None`, while sorted keys make the
operation sequence stable across runs.

## When to Use

Use this form when mappings are modest enough for a full key comparison and
value equality represents a meaningful change. Both sides must share a schema
and string-key namespace. Choose a domain-specific change format when nested
values need recursive diffs, conflicts require resolution, or updates must be
transactional.

## Implementation

```python
from collections.abc import Iterable, Mapping
from dataclasses import dataclass
from typing import Generic, TypeVar


ValueT = TypeVar("ValueT")


@dataclass(frozen=True, slots=True)
class SetValue(Generic[ValueT]):
    key: str
    value: ValueT


@dataclass(frozen=True, slots=True)
class DeleteKey:
    key: str


def _require_string_keys(mapping: Mapping[str, object]) -> None:
    if any(not isinstance(key, str) for key in mapping):
        raise TypeError("mapping keys must be strings")


def build_mapping_delta(
    base: Mapping[str, ValueT],
    current: Mapping[str, ValueT],
) -> tuple[SetValue[ValueT] | DeleteKey, ...]:
    _require_string_keys(base)
    _require_string_keys(current)

    operations = []
    for key in sorted(set(base) | set(current)):
        if key not in current:
            operations.append(DeleteKey(key))
        elif key not in base or base[key] != current[key]:
            operations.append(SetValue(key, current[key]))
    return tuple(operations)


def apply_mapping_delta(
    base: Mapping[str, ValueT],
    operations: Iterable[SetValue[ValueT] | DeleteKey],
) -> dict[str, ValueT]:
    _require_string_keys(base)
    result = dict(base)
    for operation in operations:
        if isinstance(operation, DeleteKey):
            result.pop(operation.key, None)
        else:
            result[operation.key] = operation.value
    return result
```

## Example

```python
shared_value = {"labels": ["stable"]}
base = {
    "changed": {"revision": 1},
    "keep": shared_value,
    "obsolete": 3,
    "removed_none": None,
}
current = {
    "added": 0,
    "added_none": None,
    "changed": {"revision": 2},
    "keep": shared_value,
}

delta = build_mapping_delta(base, current)
restored = apply_mapping_delta(base, delta)
applied_twice = apply_mapping_delta(restored, delta)

assert (
    delta,
    restored == current,
    applied_twice == current,
    restored is not base,
    restored["keep"] is shared_value,
    base["obsolete"],
    "obsolete" not in current,
) == (
    (
        SetValue("added", 0),
        SetValue("added_none", None),
        SetValue("changed", {"revision": 2}),
        DeleteKey("obsolete"),
        DeleteKey("removed_none"),
    ),
    True,
    True,
    True,
    True,
    3,
    True,
)
```

## Trade-offs and Limitations

The algorithm inspects the union of all keys and sorts it, so it is not an
incremental change tracker. Values are compared with ordinary equality and
retained by reference; nested ownership is shallow. Reapplying the same delta
is idempotent for stable values, but there is no base version, conflict check,
schema migration, durability, or atomic write. Add those policies at the
storage boundary rather than hiding them in this representation.

## Related Snippets

<!-- catalog:related:start -->
- [Apply Partial Dataclass Updates with an Omitted-Value Sentinel](../python-language/apply-partial-dataclass-updates-with-an-omitted-value-sentinel.md)
- [Make a Defensive Copy at a Mutable Input Boundary](../python-language/make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
