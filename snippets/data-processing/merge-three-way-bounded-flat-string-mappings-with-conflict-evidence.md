---
title: "Three-Way Merge Bounded Flat String Mappings with Conflict Evidence"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../storage-databases/build-and-apply-a-deterministic-mapping-delta.md
  - ../configuration-serialization/plan-a-bounded-json-reconciliation-under-explicit-path-policies.md
  - ../configuration-serialization/apply-a-bounded-rfc-7396-json-merge-patch-without-mutating-inputs.md
---

# Three-Way Merge Bounded Flat String Mappings with Conflict Evidence

## Idea and Problem

Merge two independently edited flat string mappings against a shared base, returning either one deterministic result or complete per-key conflict evidence.

A three-way merge can distinguish an edit from an unchanged value because both
sides are compared with the same base. Missing keys are states in the merge,
while `None` is used only in conflict evidence to display that state. The
fail-closed result never presents a partial mapping as if it were safe to use.

## When to Use

Use this algorithm for small, already parsed configuration fragments, labels,
or metadata whose keys and values are exact strings. It is appropriate when an
identical decision on both sides should merge automatically and every genuine
conflict must be resolved by a caller.

Choose a schema-aware reconciliation process when values are nested, paths have
different policies, text needs normalization, or conflicts should be resolved
automatically. Validate authorization and domain invariants after the merge and
before persisting the result.

## Implementation

```python
from dataclasses import dataclass


_MAX_FLAT_MAPPING_ENTRIES = 256
_MAX_FLAT_MAPPING_KEY_BYTES = 256
_MAX_FLAT_MAPPING_VALUE_BYTES = 4_096
_MAX_FLAT_MAPPING_TEXT_BYTES = 256 * 1_024
_MISSING = object()


@dataclass(frozen=True, slots=True)
class FlatMappingConflict:
    key: str
    base: str | None
    left: str | None
    right: str | None


@dataclass(frozen=True, slots=True)
class FlatMappingMerge:
    items: tuple[tuple[str, str], ...] | None
    conflicts: tuple[FlatMappingConflict, ...]


def _validated_mapping_size(mapping: object, *, name: str) -> int:
    if type(mapping) is not dict:
        raise TypeError(f"{name} must be an exact dict")
    if len(mapping) > _MAX_FLAT_MAPPING_ENTRIES:
        raise ValueError(f"{name} has too many entries")

    total_bytes = 0
    for key, value in mapping.items():
        if type(key) is not str or type(value) is not str:
            raise TypeError(f"{name} keys and values must be exact strings")
        # Every Unicode scalar needs at least one UTF-8 byte. These cheap
        # character-count guards bound the temporary encodings allocated by
        # the exact byte checks below.
        if len(key) > _MAX_FLAT_MAPPING_KEY_BYTES:
            raise ValueError(f"{name} contains an oversized key")
        if len(value) > _MAX_FLAT_MAPPING_VALUE_BYTES:
            raise ValueError(f"{name} contains an oversized value")
        try:
            key_size = len(key.encode("utf-8"))
            value_size = len(value.encode("utf-8"))
        except UnicodeEncodeError as error:
            raise ValueError(f"{name} contains text that is not valid UTF-8") from error
        if key_size > _MAX_FLAT_MAPPING_KEY_BYTES:
            raise ValueError(f"{name} contains an oversized key")
        if value_size > _MAX_FLAT_MAPPING_VALUE_BYTES:
            raise ValueError(f"{name} contains an oversized value")
        total_bytes += key_size + value_size
    return total_bytes


def merge_three_way_string_mappings(
    base: dict[str, str],
    left: dict[str, str],
    right: dict[str, str],
) -> FlatMappingMerge:
    total_bytes = sum(
        (
            _validated_mapping_size(base, name="base"),
            _validated_mapping_size(left, name="left"),
            _validated_mapping_size(right, name="right"),
        )
    )
    if total_bytes > _MAX_FLAT_MAPPING_TEXT_BYTES:
        raise ValueError("the aggregate mapping text is too large")

    keys = base.keys() | left.keys() | right.keys()
    if len(keys) > _MAX_FLAT_MAPPING_ENTRIES:
        raise ValueError("the mappings have too many distinct keys")

    items: list[tuple[str, str]] = []
    conflicts: list[FlatMappingConflict] = []
    for key in sorted(keys):
        base_value = base[key] if key in base else _MISSING
        left_value = left[key] if key in left else _MISSING
        right_value = right[key] if key in right else _MISSING

        if left_value == right_value:
            resolved = left_value
        elif left_value == base_value:
            resolved = right_value
        elif right_value == base_value:
            resolved = left_value
        else:
            conflicts.append(
                FlatMappingConflict(
                    key=key,
                    base=None if base_value is _MISSING else base_value,
                    left=None if left_value is _MISSING else left_value,
                    right=None if right_value is _MISSING else right_value,
                )
            )
            continue

        if resolved is not _MISSING:
            items.append((key, resolved))

    if conflicts:
        return FlatMappingMerge(items=None, conflicts=tuple(conflicts))
    return FlatMappingMerge(items=tuple(items), conflicts=())
```

## Example

```python
from itertools import product


def assert_raises(error_type, operation):
    try:
        operation()
    except error_type as error:
        return error
    raise AssertionError(f"{error_type.__name__} was not raised")


# The independent oracle describes edits, rather than repeating the merge's
# branch order. All 27 base/left/right states for one key are exercised.
ABSENT = object()
states = (ABSENT, "a", "b")
conflict_count = 0
for base_state, left_state, right_state in product(states, repeat=3):
    mappings = tuple(
        {} if state is ABSENT else {"key": state} for state in (base_state, left_state, right_state)
    )
    result = merge_three_way_string_mappings(*mappings)
    left_changed = left_state != base_state
    right_changed = right_state != base_state

    if left_changed and right_changed and left_state != right_state:
        conflict_count += 1
        assert result == FlatMappingMerge(
            items=None,
            conflicts=(
                FlatMappingConflict(
                    "key",
                    None if base_state is ABSENT else base_state,
                    None if left_state is ABSENT else left_state,
                    None if right_state is ABSENT else right_state,
                ),
            ),
        )
    else:
        expected_state = left_state if left_changed else right_state
        expected_items = () if expected_state is ABSENT else (("key", expected_state),)
        assert result == FlatMappingMerge(expected_items, ())

    swapped = merge_three_way_string_mappings(mappings[0], mappings[2], mappings[1])
    if result.items is not None:
        assert swapped.items == result.items
    else:
        assert swapped.conflicts == tuple(
            FlatMappingConflict(item.key, item.base, item.right, item.left)
            for item in result.conflicts
        )

assert conflict_count == 6

base = {
    "": "old-empty-key",
    "both": "old",
    "delete": "gone",
    "left": "old",
    "right": "old",
    "same": "stable",
}
left = {
    "": "",
    "added": "new",
    "both": "updated",
    "left": "left-update",
    "right": "old",
    "same": "stable",
}
right = {
    "": "",
    "added": "new",
    "both": "updated",
    "left": "old",
    "right": "right-update",
    "same": "stable",
}
snapshots = (dict(base), dict(left), dict(right))
merged = merge_three_way_string_mappings(base, left, right)
assert merged == FlatMappingMerge(
    items=(
        ("", ""),
        ("added", "new"),
        ("both", "updated"),
        ("left", "left-update"),
        ("right", "right-update"),
        ("same", "stable"),
    ),
    conflicts=(),
)
assert (base, left, right) == snapshots

conflicted = merge_three_way_string_mappings(
    {"change": "base", "delete-change": "base"},
    {"add": "left", "change": "left"},
    {"add": "right", "change": "right", "delete-change": "right"},
)
assert conflicted == FlatMappingMerge(
    items=None,
    conflicts=(
        FlatMappingConflict("add", None, "left", "right"),
        FlatMappingConflict("change", "base", "left", "right"),
        FlatMappingConflict("delete-change", "base", None, "right"),
    ),
)

mixed = merge_three_way_string_mappings(
    {"clean": "old", "conflict": "old"},
    {"clean": "left", "conflict": "left"},
    {"clean": "old", "conflict": "right"},
)
assert mixed.items is None
assert mixed.conflicts == (FlatMappingConflict("conflict", "old", "left", "right"),)

maximum_union = {f"{index:03d}": "v" for index in range(256)}
assert len(merge_three_way_string_mappings(maximum_union, {}, {}).items) == 0
assert_raises(
    ValueError,
    lambda: merge_three_way_string_mappings(
        {f"{index:03d}": "v" for index in range(257)},
        {},
        {},
    ),
)
assert_raises(
    ValueError,
    lambda: merge_three_way_string_mappings(
        {f"b{index:03d}": "v" for index in range(129)},
        {f"l{index:03d}": "v" for index in range(128)},
        {},
    ),
)

assert merge_three_way_string_mappings({"é" * 128: ""}, {}, {}).items == ()
assert merge_three_way_string_mappings({"value": "é" * 2_048}, {}, {}).items == ()
assert_raises(
    ValueError,
    lambda: merge_three_way_string_mappings({"é" * 129: ""}, {}, {}),
)
assert_raises(
    ValueError,
    lambda: merge_three_way_string_mappings({"value": "é" * 2_049}, {}, {}),
)

exact_aggregate = {chr(33 + index): "x" * (4_096 if index < 63 else 4_032) for index in range(64)}
assert merge_three_way_string_mappings(exact_aggregate, {}, {}).items == ()
over_aggregate = dict(exact_aggregate)
over_aggregate[chr(96)] += "x"
assert_raises(
    ValueError,
    lambda: merge_three_way_string_mappings(over_aggregate, {}, {}),
)
surrogate_error = assert_raises(
    ValueError,
    lambda: merge_three_way_string_mappings({"\ud800": "value"}, {}, {}),
)
assert type(surrogate_error) is ValueError
assert isinstance(surrogate_error.__cause__, UnicodeEncodeError)


class Text(str):
    pass


class Mapping(dict):
    pass


assert_raises(
    TypeError,
    lambda: merge_three_way_string_mappings({Text("key"): "value"}, {}, {}),
)
assert_raises(
    TypeError,
    lambda: merge_three_way_string_mappings(Mapping(), {}, {}),
)
assert merge_three_way_string_mappings(
    {"key": "old"},
    {"key": "new"},
    {"key": "old"},
).items == (("key", "new"),)
```

## Trade-offs and Limitations

Validation is linear in the input text size; resolving `K` distinct keys costs
`O(K log K)` time for sorting and `O(K)` additional memory. Ordering follows
Python's exact Unicode code-point ordering without normalization or case
folding. Empty strings are valid data, but `None` in conflict evidence means a
key was absent and cannot represent an input value.

The fixed limits intentionally reject large mapping merges. The algorithm is
not recursive, does not assign precedence, and does not persist or lock data.
Any conflict makes `items` unavailable, so callers must resolve every reported
key and retry rather than accidentally applying an incomplete result.

## Related Snippets

<!-- catalog:related:start -->
- [Build and Apply a Deterministic Mapping Delta](../storage-databases/build-and-apply-a-deterministic-mapping-delta.md)
- [Plan a Bounded JSON Reconciliation Under Explicit Path Policies](../configuration-serialization/plan-a-bounded-json-reconciliation-under-explicit-path-policies.md)
- [Apply a Bounded RFC 7396 JSON Merge Patch without Mutating Inputs](../configuration-serialization/apply-a-bounded-rfc-7396-json-merge-patch-without-mutating-inputs.md)
<!-- catalog:related:end -->
