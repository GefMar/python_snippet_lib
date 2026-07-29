---
title: "Join Non-Overlapping Half-Open Validity Intervals by Exact Overlap"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
  - ../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md
  - ../algorithms-data-structures/subtract-one-canonical-half-open-integer-interval-set-from-another.md
---

# Join Non-Overlapping Half-Open Validity Intervals by Exact Overlap

## Idea and Problem

Join two ordered effective-dated snapshots by emitting every positive-length intersection together with the labels from both sides.

Each side is already canonical: intervals are non-empty, sorted, and do not
overlap within that side. Therefore, after comparing one left interval with one
right interval, whichever interval ends first can never match a later interval
on the other side. Advancing that pointer produces a linear join without
materializing a timestamp grid.

## When to Use

Use this for bounded in-memory validity windows whose labels must be combined
only while both windows are active. Half-open boundaries make adjacent windows
unambiguous: `[2, 5)` and `[5, 8)` touch but do not overlap.

Canonicalize or reject overlapping input before this boundary. Use a different
algorithm when multiple intervals on the same side may be active at once,
because then one point can participate in several matches and the simple
two-pointer invariant no longer holds. Parse dates and resolve time zones
before calling this integer-domain join.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_INTERVALS_PER_SIDE = 10_000
_MAX_LABEL_BYTES = 256
_MAX_TOTAL_LABEL_BYTES = 1_048_576


@dataclass(frozen=True, slots=True)
class ValidityInterval:
    start: int
    end: int
    label: str


@dataclass(frozen=True, slots=True)
class ValidityOverlap:
    start: int
    end: int
    left_label: str
    right_label: str


def _validated_intervals(
    value: object,
    *,
    field: str,
) -> tuple[ValidityInterval, ...]:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(value) > _MAX_INTERVALS_PER_SIDE:
        raise ValueError(f"{field} exceeds the interval-count limit")

    previous_end: int | None = None
    total_label_bytes = 0
    for interval in value:
        if type(interval) is not ValidityInterval:
            raise TypeError(
                f"each {field} item must be an exact ValidityInterval"
            )
        if type(interval.start) is not int or type(interval.end) is not int:
            raise TypeError(f"each {field} boundary must be an exact integer")
        if not (
            _MIN_INT64 <= interval.start <= _MAX_INT64
            and _MIN_INT64 <= interval.end <= _MAX_INT64
        ):
            raise ValueError(
                f"each {field} boundary must be in the signed 64-bit range"
            )
        if interval.start >= interval.end:
            raise ValueError(f"each {field} interval must be non-empty")
        if previous_end is not None and interval.start < previous_end:
            raise ValueError(
                f"{field} intervals must be sorted and non-overlapping"
            )

        if type(interval.label) is not str:
            raise TypeError(f"each {field} label must be an exact string")
        try:
            label_bytes = len(interval.label.encode("utf-8"))
        except UnicodeEncodeError:
            raise ValueError(
                f"each {field} label must be UTF-8 encodable"
            ) from None
        if not 1 <= label_bytes <= _MAX_LABEL_BYTES:
            raise ValueError(f"a {field} label is outside the byte limit")
        total_label_bytes += label_bytes
        if total_label_bytes > _MAX_TOTAL_LABEL_BYTES:
            raise ValueError(f"{field} labels exceed the aggregate byte limit")

        previous_end = interval.end

    return value


def join_validity_intervals(
    left: tuple[ValidityInterval, ...],
    right: tuple[ValidityInterval, ...],
) -> tuple[ValidityOverlap, ...]:
    """Return every positive-length overlap between two canonical sides."""
    checked_left = _validated_intervals(left, field="left")
    checked_right = _validated_intervals(right, field="right")

    overlaps: list[ValidityOverlap] = []
    left_index = 0
    right_index = 0

    while left_index < len(checked_left) and right_index < len(checked_right):
        left_interval = checked_left[left_index]
        right_interval = checked_right[right_index]
        overlap_start = max(left_interval.start, right_interval.start)
        overlap_end = min(left_interval.end, right_interval.end)

        if overlap_start < overlap_end:
            overlaps.append(
                ValidityOverlap(
                    start=overlap_start,
                    end=overlap_end,
                    left_label=left_interval.label,
                    right_label=right_interval.label,
                )
            )

        if left_interval.end < right_interval.end:
            left_index += 1
        elif right_interval.end < left_interval.end:
            right_index += 1
        else:
            left_index += 1
            right_index += 1

    return tuple(overlaps)
```

## Example

```python
left = (
    ValidityInterval(0, 4, "left-a"),
    ValidityInterval(4, 8, "left-b"),
    ValidityInterval(10, 12, "left-c"),
)
right = (
    ValidityInterval(2, 5, "right-x"),
    ValidityInterval(5, 10, "right-y"),
    ValidityInterval(10, 11, "right-z"),
)

observed = join_validity_intervals(left, right)
expected = (
    ValidityOverlap(2, 4, "left-a", "right-x"),
    ValidityOverlap(4, 5, "left-b", "right-x"),
    ValidityOverlap(5, 8, "left-b", "right-y"),
    ValidityOverlap(10, 11, "left-c", "right-z"),
)

touching_only = join_validity_intervals(
    (ValidityInterval(0, 2, "first"),),
    (ValidityInterval(2, 3, "second"),),
)

try:
    join_validity_intervals(
        (
            ValidityInterval(0, 4, "one"),
            ValidityInterval(3, 5, "overlap"),
        ),
        right,
    )
except ValueError:
    internal_overlap_rejected = True
else:
    internal_overlap_rejected = False

try:
    join_validity_intervals(
        (ValidityInterval(True, 2, "invalid"),),
        right,
    )
except TypeError:
    boolean_boundary_rejected = True
else:
    boolean_boundary_rejected = False

assert (
    observed == expected
    and touching_only == ()
    and internal_overlap_rejected
    and boolean_boundary_rejected
)
```

## Trade-offs and Limitations

Validation takes `O(L + R + B)` time for `B` inspected label bytes. The join
takes `O(L + R + K)` time and `O(K)` result memory for `K` overlaps. Because
each side is internally non-overlapping, `K` is also linear in the input sizes;
the algorithm never forms a Cartesian product.

The function requires canonical order instead of sorting or coalescing on the
caller's behalf. It preserves labels by value but has no parsing, timezone,
priority, tolerance, open-ended-interval, payload-merging, or streaming
semantics. A gap on either side produces no row, and an interval that only
touches another interval at one endpoint is deliberately absent from output.

## Related Snippets

<!-- catalog:related:start -->
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
- [Find a Point in Disjoint Half-Open Intervals](../algorithms-data-structures/find-a-point-in-disjoint-half-open-intervals.md)
- [Subtract One Canonical Half-Open Integer Interval Set from Another](../algorithms-data-structures/subtract-one-canonical-half-open-integer-interval-set-from-another.md)
<!-- catalog:related:end -->
