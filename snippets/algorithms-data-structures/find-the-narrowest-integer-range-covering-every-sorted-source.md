---
title: "Find the Narrowest Integer Range Covering Every Sorted Source"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
  - ../data-processing/join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md
---

# Find the Narrowest Integer Range Covering Every Sorted Source

## Idea and Problem

Find one closed integer range that contains at least one value from every sorted source while making equal-width choices deterministic.

Keep one current value from each source in a min-heap and track their maximum.
Those values define a covering range. Advancing the source that owns the
minimum is the only move that can increase the lower bound and possibly shrink
the range; once that source is exhausted, no later complete cover exists.

## When to Use

Use this algorithm when several bounded, strictly increasing integer snapshots
must be represented by one tight common range. Examples include locating a
small cross-source time window or comparing candidate positions from several
ordered indexes. The explicit width, lower-bound, then upper-bound policy keeps
snapshots and tests reproducible.

Use a streaming algorithm when every source cannot be validated in memory, or
a nearest-neighbor or assignment algorithm when the selected values themselves
must be returned. Unsorted sources should be normalized under a separate,
explicit duplicate policy before this function is called.

## Implementation

```python
from heapq import heappop, heappush

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_SOURCES = 64
_MAX_TOTAL_VALUES = 100_000


def find_narrowest_covering_range(
    sources: tuple[tuple[int, ...], ...],
) -> tuple[int, int]:
    """Return the canonical narrowest closed range covering every source."""
    if type(sources) is not tuple:
        raise TypeError("sources must be an exact tuple")
    if not 1 <= len(sources) <= _MAX_SOURCES:
        raise ValueError("source count is outside the supported range")

    heads: list[tuple[int, int, int]] = []
    total_values = 0
    current_upper = _MIN_INT64

    for source_index, source in enumerate(sources):
        if type(source) is not tuple:
            raise TypeError(f"sources[{source_index}] must be an exact tuple")
        if not source:
            raise ValueError(f"sources[{source_index}] must not be empty")
        if len(source) > _MAX_TOTAL_VALUES - total_values:
            raise ValueError("aggregate value count exceeds the supported limit")

        previous: int | None = None
        for value_index, value in enumerate(source):
            if type(value) is not int:
                raise TypeError(f"sources[{source_index}][{value_index}] must be an exact integer")
            if not _MIN_INT64 <= value <= _MAX_INT64:
                raise ValueError(
                    f"sources[{source_index}][{value_index}] is outside the signed 64-bit range"
                )
            if previous is not None and value <= previous:
                raise ValueError(f"sources[{source_index}] must be strictly increasing")
            previous = value

        total_values += len(source)
        first = source[0]
        heappush(heads, (first, source_index, 0))
        current_upper = max(current_upper, first)

    initial_lower = heads[0][0]
    best = (current_upper - initial_lower, initial_lower, current_upper)

    while True:
        lower, source_index, value_index = heappop(heads)
        candidate = (current_upper - lower, lower, current_upper)
        if candidate < best:
            best = candidate

        next_index = value_index + 1
        source = sources[source_index]
        if next_index == len(source):
            break

        next_value = source[next_index]
        current_upper = max(current_upper, next_value)
        heappush(heads, (next_value, source_index, next_index))

    return best[1], best[2]
```

## Example

```python
def brute_covering_range(
    sources: tuple[tuple[int, ...], ...],
) -> tuple[int, int]:
    from itertools import product

    width, lower, upper = min(
        (max(selected) - min(selected), min(selected), max(selected))
        for selected in product(*sources)
    )
    assert width == upper - lower
    return lower, upper


def exercise_tiny_sources() -> int:
    from itertools import combinations, product

    domain = (-2, -1, 0, 1)
    source_options = tuple(
        option for size in range(1, len(domain) + 1) for option in combinations(domain, size)
    )
    checked = 0
    for source_count in range(1, 4):
        for tiny_sources in product(source_options, repeat=source_count):
            assert find_narrowest_covering_range(tiny_sources) == brute_covering_range(tiny_sources)
            checked += 1
    return checked


checked_cases = exercise_tiny_sources()
duplicate_across_sources = find_narrowest_covering_range(((-5, 1, 8), (0, 1, 7), (1, 9)))
equal_width_tie = find_narrowest_covering_range(((0, 10), (4, 14)))
extreme_width = find_narrowest_covering_range(((_MIN_INT64,), (_MAX_INT64,)))

try:
    find_narrowest_covering_range(((1, 1), (1, 2)))
except ValueError:
    duplicate_within_source_rejected = True
else:
    duplicate_within_source_rejected = False

try:
    find_narrowest_covering_range(((1, 2), []))  # type: ignore[arg-type]
except TypeError:
    list_source_rejected = True
else:
    list_source_rejected = False

assert (
    checked_cases,
    duplicate_across_sources,
    equal_width_tie,
    extreme_width,
    duplicate_within_source_rejected,
    list_source_rejected,
) == (
    3_615,
    (1, 1),
    (0, 4),
    (_MIN_INT64, _MAX_INT64),
    True,
    True,
)
```

## Trade-offs and Limitations

Validation costs `O(N)` time. The heap sweep costs
`O(N log max(K, 2))` time for `N` total values from `K` sources and uses
`O(K)` working memory. The returned width is calculated exactly by Python and
may exceed the signed 64-bit input range.

The result contains only the closed bounds, not the particular source values
that established them. Equal-width candidates prefer the smaller lower bound
and then the smaller upper bound. Values may repeat across different sources,
allowing a zero-width answer, but every individual source must be non-empty and
strictly increasing.

The function does not sort inputs, retain selection provenance, accept floats,
produce approximate ranges, consume unbounded streams, or support mutable
updates.

## Related Snippets

<!-- catalog:related:start -->
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
- [Join Bounded Strictly Increasing Sequences by the Latest Prior Timestamp](../data-processing/join-bounded-strictly-increasing-sequences-by-the-latest-prior-timestamp.md)
<!-- catalog:related:end -->
