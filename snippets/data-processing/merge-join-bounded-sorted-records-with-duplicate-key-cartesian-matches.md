---
title: "Merge-Join Bounded Sorted Records with Duplicate-Key Cartesian Matches"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-two-strictly-increasing-streams-by-exact-timestamp.md
  - join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
  - ../algorithms-data-structures/merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
---

# Merge-Join Bounded Sorted Records with Duplicate-Key Cartesian Matches

## Idea and Problem

Join two validated sorted record collections without losing the many-to-many matches created by duplicate keys.

Two cursors skip unequal keys and stop together at equal-key groups. If one
side has `a` records and the other has `b`, that key contributes `a * b`
matches. A preliminary group scan counts the complete output before any match
objects are allocated.

The second scan emits each left index against every right index in its group,
which gives stable key, left-index, then right-index ordering.

## When to Use

Use this merge join when both finite inputs are already sorted by the same
exact integer key, duplicate keys are meaningful, and a complete in-memory
inner join fits a declared output budget. Returning indexes avoids copying
payload strings into every Cartesian match.

Use a database, external sort, or streaming join when inputs are unsorted,
larger than memory, or outputs must spill. Use a lookup mapping when one side
is small and preserving sorted merge behavior is unimportant.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RECORD_COUNT = 50_000
_MAX_VALUE_LENGTH = 256
_MAX_VALUE_BYTES = 1_024
_MAX_OUTPUT_PAIRS = 200_000


@dataclass(frozen=True, slots=True)
class KeyedTextRecord:
    key: int
    value: str


@dataclass(frozen=True, slots=True)
class JoinMatch:
    key: int
    left_index: int
    right_index: int


def _validate_sorted_records(
    name: str,
    records: tuple[KeyedTextRecord, ...],
) -> None:
    if type(records) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(records) > _MAX_RECORD_COUNT:
        raise ValueError(f"{name} contains too many records")

    previous_key: int | None = None
    for index, record in enumerate(records):
        if type(record) is not KeyedTextRecord:
            raise TypeError(f"{name}[{index}] must be a KeyedTextRecord")
        if type(record.key) is not int:
            raise TypeError(f"{name}[{index}].key must be an exact integer")
        if not _MIN_INT64 <= record.key <= _MAX_INT64:
            raise ValueError(f"{name}[{index}].key is outside signed 64-bit range")
        if type(record.value) is not str:
            raise TypeError(f"{name}[{index}].value must be an exact string")
        if len(record.value) > _MAX_VALUE_LENGTH:
            raise ValueError(f"{name}[{index}].value is too long")
        try:
            encoded_value = record.value.encode("utf-8")
        except UnicodeEncodeError as error:
            raise ValueError(f"{name}[{index}].value contains an invalid Unicode scalar") from error
        if len(encoded_value) > _MAX_VALUE_BYTES:
            raise ValueError(f"{name}[{index}].value exceeds the UTF-8 byte limit")
        if previous_key is not None and record.key < previous_key:
            raise ValueError(f"{name} keys must be nondecreasing")
        previous_key = record.key


def _group_end(
    records: tuple[KeyedTextRecord, ...],
    start: int,
) -> int:
    end = start + 1
    while end < len(records) and records[end].key == records[start].key:
        end += 1
    return end


def merge_join_duplicate_keys(
    left: tuple[KeyedTextRecord, ...],
    right: tuple[KeyedTextRecord, ...],
    *,
    max_output_pairs: int,
) -> tuple[JoinMatch, ...]:
    """Return every equal-key index pair in stable merge order."""
    if type(max_output_pairs) is not int:
        raise TypeError("max_output_pairs must be an exact integer")
    if not 0 <= max_output_pairs <= _MAX_OUTPUT_PAIRS:
        raise ValueError("max_output_pairs is outside the supported range")

    _validate_sorted_records("left", left)
    _validate_sorted_records("right", right)

    required_pairs = 0
    left_index = 0
    right_index = 0
    while left_index < len(left) and right_index < len(right):
        if left[left_index].key < right[right_index].key:
            left_index = _group_end(left, left_index)
        elif left[left_index].key > right[right_index].key:
            right_index = _group_end(right, right_index)
        else:
            left_end = _group_end(left, left_index)
            right_end = _group_end(right, right_index)
            required_pairs += (left_end - left_index) * (right_end - right_index)
            if required_pairs > max_output_pairs:
                raise ValueError("complete join exceeds max_output_pairs")
            left_index = left_end
            right_index = right_end

    matches: list[JoinMatch] = []
    left_index = 0
    right_index = 0
    while left_index < len(left) and right_index < len(right):
        left_key = left[left_index].key
        right_key = right[right_index].key
        if left_key < right_key:
            left_index = _group_end(left, left_index)
        elif left_key > right_key:
            right_index = _group_end(right, right_index)
        else:
            left_end = _group_end(left, left_index)
            right_end = _group_end(right, right_index)
            matches.extend(
                JoinMatch(left_key, current_left, current_right)
                for current_left in range(left_index, left_end)
                for current_right in range(right_index, right_end)
            )
            left_index = left_end
            right_index = right_end

    if len(matches) != required_pairs:
        raise AssertionError("counting and emission scans disagree")
    return tuple(matches)
```

## Example

```python
left_records = (
    KeyedTextRecord(1, "left one"),
    KeyedTextRecord(2, "left two-a"),
    KeyedTextRecord(2, "left two-b"),
    KeyedTextRecord(4, "left four"),
)
right_records = (
    KeyedTextRecord(2, "right two-a"),
    KeyedTextRecord(2, "right two-b"),
    KeyedTextRecord(3, "right three"),
)

matches = merge_join_duplicate_keys(
    left_records,
    right_records,
    max_output_pairs=4,
)

assert matches == (
    JoinMatch(2, 1, 0),
    JoinMatch(2, 1, 1),
    JoinMatch(2, 2, 0),
    JoinMatch(2, 2, 1),
)
assert [
    (left_records[match.left_index].value, right_records[match.right_index].value)
    for match in matches
] == [
    ("left two-a", "right two-a"),
    ("left two-a", "right two-b"),
    ("left two-b", "right two-a"),
    ("left two-b", "right two-b"),
]
assert merge_join_duplicate_keys((), right_records, max_output_pairs=0) == ()
```

## Trade-offs and Limitations

Complete validation and the counting scan are linear in both inputs. Emission
takes `O(z)` for `z` matches, giving `O(n + m + z)` total time and `O(z)` output
state. The preliminary scan prevents a large duplicate group from allocating a
partial result before the budget failure is known.

The output budget limits match records, not input bytes or the memory retained
by input payloads. A key with large duplicate groups can still reach the limit
quickly. Payloads remain in the caller-owned inputs and are addressed by stable
indexes.

This is an inner equality join only. It does not sort or repair inputs, coerce
keys, copy payloads, emit unmatched records, stop at a partial cap, stream,
spill, or choose among duplicates. UTF-8 validation provides a bounded text
profile, not normalization or collation.

## Related Snippets

<!-- catalog:related:start -->
- [Join Two Strictly Increasing Streams by Exact Timestamp](join-two-strictly-increasing-streams-by-exact-timestamp.md)
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](../algorithms-data-structures/merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
<!-- catalog:related:end -->
