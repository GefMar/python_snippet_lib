---
title: "Co-Group Bounded Sorted Integer Keys with Full-Outer Coverage"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - merge-join-bounded-sorted-records-with-duplicate-key-cartesian-matches.md
  - audit-bounded-keyed-half-open-intervals-with-gap-and-overlap-evidence.md
  - ../algorithms-data-structures/merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
---

# Co-Group Bounded Sorted Integer Keys with Full-Outer Coverage

## Idea and Problem

Co-group two validated sorted integer-key collections into one compact record per key in their union, including keys present on only one side.

Each output stores optional half-open index spans into the original tuples.
Duplicate runs are represented once instead of being expanded into Cartesian
pairs. Two monotonic cursors produce groups in ascending key order after both
complete inputs have passed the same bounded signed-integer contract.

## When to Use

Use this algorithm when two finite inputs are already sorted, duplicate keys
must stay grouped, unmatched keys must remain visible, and callers can use
indexes to access the original records. It is a compact preparation step for
full-outer reconciliation, aggregate comparison, or deciding later how each
duplicate run should be paired.

Use a Cartesian merge join when every equal-key pair is itself an output. Use
a database or external merge when inputs do not fit in memory, and sort first
when rejecting out-of-order input is not the intended behavior.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_KEY_COUNT = 50_000

IndexSpan = tuple[int, int]


@dataclass(frozen=True, slots=True)
class KeyGroup:
    key: int
    left_span: IndexSpan | None
    right_span: IndexSpan | None


def _validate_sorted_integer_keys(name: str, keys: tuple[int, ...]) -> None:
    if type(keys) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    if len(keys) > _MAX_KEY_COUNT:
        raise ValueError(f"{name} contains more than 50000 keys")

    previous: int | None = None
    for index, key in enumerate(keys):
        if type(key) is not int:
            raise TypeError(f"{name}[{index}] must be an exact integer")
        if not _MIN_INT64 <= key <= _MAX_INT64:
            raise ValueError(f"{name}[{index}] is outside signed 64-bit range")
        if previous is not None and key < previous:
            raise ValueError(f"{name} must be nondecreasing")
        previous = key


def _integer_run_end(keys: tuple[int, ...], start: int) -> int:
    end = start + 1
    while end < len(keys) and keys[end] == keys[start]:
        end += 1
    return end


def co_group_sorted_integer_keys(
    left: tuple[int, ...],
    right: tuple[int, ...],
) -> tuple[KeyGroup, ...]:
    """Return ascending full-outer groups with half-open input spans."""
    _validate_sorted_integer_keys("left", left)
    _validate_sorted_integer_keys("right", right)

    groups: list[KeyGroup] = []
    left_index = 0
    right_index = 0
    while left_index < len(left) or right_index < len(right):
        if right_index == len(right) or (
            left_index < len(left) and left[left_index] < right[right_index]
        ):
            left_end = _integer_run_end(left, left_index)
            groups.append(KeyGroup(left[left_index], (left_index, left_end), None))
            left_index = left_end
        elif left_index == len(left) or right[right_index] < left[left_index]:
            right_end = _integer_run_end(right, right_index)
            groups.append(KeyGroup(right[right_index], None, (right_index, right_end)))
            right_index = right_end
        else:
            left_end = _integer_run_end(left, left_index)
            right_end = _integer_run_end(right, right_index)
            groups.append(
                KeyGroup(
                    left[left_index],
                    (left_index, left_end),
                    (right_index, right_end),
                )
            )
            left_index = left_end
            right_index = right_end
    return tuple(groups)


```

## Example

```python
left_keys = (1, 2, 2, 5)
right_keys = (2, 2, 3, 5, 5, 5)

groups = co_group_sorted_integer_keys(left_keys, right_keys)

assert groups == (
    KeyGroup(1, (0, 1), None),
    KeyGroup(2, (1, 3), (0, 2)),
    KeyGroup(3, None, (2, 3)),
    KeyGroup(5, (3, 4), (3, 6)),
)
assert co_group_sorted_integer_keys((), ()) == ()
assert co_group_sorted_integer_keys((), (4, 4)) == (KeyGroup(4, None, (0, 2)),)
```

## Trade-offs and Limitations

Complete validation and grouping take `O(n + m)` time. The immutable result
uses `O(g)` space for `g` distinct keys across both inputs. Because each input
contains at most 50,000 values, the union can contain at most 100,000 groups;
there is no separate or partial output budget.

The spans address the caller-owned input tuples and do not copy or combine
payloads. They remain meaningful only for those exact tuples. The function
does not sort, coerce booleans or other integer-like objects, repair invalid
order, stream results, compare attached records, or choose a policy for pairing
members of duplicate runs.

This is a full-outer co-group, not a join: it emits unmatched keys but never
expands equal-key runs into individual pairs. Both inputs are validated in full
before the first output object is built, so an error on the second input cannot
leave a caller-visible partial result.

## Related Snippets

<!-- catalog:related:start -->
- [Merge-Join Bounded Sorted Records with Duplicate-Key Cartesian Matches](merge-join-bounded-sorted-records-with-duplicate-key-cartesian-matches.md)
- [Audit Bounded Keyed Half-Open Intervals with Gap and Overlap Evidence](audit-bounded-keyed-half-open-intervals-with-gap-and-overlap-evidence.md)
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](../algorithms-data-structures/merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
<!-- catalog:related:end -->
