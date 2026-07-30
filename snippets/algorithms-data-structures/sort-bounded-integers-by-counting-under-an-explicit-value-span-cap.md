---
title: "Sort Bounded Integers by Counting Under an Explicit Value-Span Cap"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - count-strict-inversions-in-a-bounded-integer-sequence.md
  - merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
  - sort-newline-terminated-binary-records-with-bounded-merge-passes.md
---

# Sort Bounded Integers by Counting Under an Explicit Value-Span Cap

## Idea and Problem

Sort a bounded tuple of integers by counting occurrences when the inclusive range from its minimum to maximum is explicitly small.

Each value maps to an offset from the minimum. A count array records how many
times every offset occurs, and scanning those counts in offset order emits one
canonical ascending tuple. The value-span cap is checked before allocating the
array, preventing a sparse input such as two distant integers from causing an
allocation proportional to their numeric distance.

## When to Use

Use counting sort when integer values occupy a known narrow range and an exact
ascending value tuple is the desired result. It is useful for bounded codes,
small signed deltas, and dense bucket identifiers where the span is at most
4,096 even if the number of values is much larger.

Use Python's stable `sorted()` for arbitrary or sparse ranges, custom keys, or
records whose relative order must be preserved. Use merge-based external
sorting when the values do not fit in memory.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_VALUE_COUNT = 262_144
_MAX_VALUE_SPAN = 4_096


def sort_bounded_integers_by_counting(
    values: tuple[int, ...],
) -> tuple[int, ...]:
    """Return values in ascending order when their inclusive span is bounded."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_VALUE_COUNT:
        raise ValueError("value count exceeds the supported limit")

    minimum: int | None = None
    maximum: int | None = None
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")
        minimum = value if minimum is None else min(minimum, value)
        maximum = value if maximum is None else max(maximum, value)

    if minimum is None or maximum is None:
        return ()

    value_span = maximum - minimum + 1
    if value_span > _MAX_VALUE_SPAN:
        raise ValueError("value span exceeds the supported limit")

    counts = [0] * value_span
    for value in values:
        counts[value - minimum] += 1

    return tuple(
        minimum + offset
        for offset, count in enumerate(counts)
        for _ in range(count)
    )
```

## Example

```python
def exercise_small_sequences() -> int:
    from itertools import product

    checked_cases = 0
    for length in range(8):
        for values in product((-2, 0, 3), repeat=length):
            assert sort_bounded_integers_by_counting(values) == tuple(sorted(values))
            checked_cases += 1
    return checked_cases


checked_cases = exercise_small_sequences()

duplicates = (3, -1, 3, 0, -1, 3)
maximum_count = (4,) * _MAX_VALUE_COUNT
lower_boundary = (_MIN_INT64 + _MAX_VALUE_SPAN - 1, _MIN_INT64)
upper_boundary = (_MAX_INT64, _MAX_INT64 - _MAX_VALUE_SPAN + 1)

rejected = 0
invalid_calls = (
    lambda: sort_bounded_integers_by_counting((0, True)),
    lambda: sort_bounded_integers_by_counting((_MIN_INT64 - 1,)),
    lambda: sort_bounded_integers_by_counting((_MAX_INT64 + 1,)),
    lambda: sort_bounded_integers_by_counting((0,) * (_MAX_VALUE_COUNT + 1)),
    lambda: sort_bounded_integers_by_counting((0, _MAX_VALUE_SPAN)),
)
for invalid_call in invalid_calls:
    try:
        invalid_call()
    except (TypeError, ValueError):
        rejected += 1

assert (
    checked_cases,
    sort_bounded_integers_by_counting(()),
    sort_bounded_integers_by_counting(duplicates),
    sort_bounded_integers_by_counting(maximum_count) == maximum_count,
    sort_bounded_integers_by_counting(lower_boundary),
    sort_bounded_integers_by_counting(upper_boundary),
    rejected,
) == (
    3_280,
    (),
    (-1, -1, 0, 3, 3, 3),
    True,
    (_MIN_INT64, _MIN_INT64 + _MAX_VALUE_SPAN - 1),
    (_MAX_INT64 - _MAX_VALUE_SPAN + 1, _MAX_INT64),
    5,
)
```

## Trade-offs and Limitations

Let `n` be the value count and `span = maximum - minimum + 1`. Validation,
counting, and emission take `O(n + span)` time. The count array uses `O(span)`
auxiliary memory, and the returned tuple uses `O(n)` memory. The span is
validated before the count array is allocated.

The function accepts at most 262,144 exact signed 64-bit non-Boolean integers
and rejects a non-empty span above 4,096. Python integer subtraction safely
computes the span even when the endpoints are far apart, so rejection does not
depend on fixed-width overflow behavior.

Counting equal scalar integers has no observable stability property: the
result contains values, not records retaining input identities. This function
does not accept key callbacks, sparse or unbounded ranges, external data,
in-place mutation, or a request for descending order.

## Related Snippets

<!-- catalog:related:start -->
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
- [Sort Newline-Terminated Binary Records with Bounded Merge Passes](sort-newline-terminated-binary-records-with-bounded-merge-passes.md)
<!-- catalog:related:end -->
