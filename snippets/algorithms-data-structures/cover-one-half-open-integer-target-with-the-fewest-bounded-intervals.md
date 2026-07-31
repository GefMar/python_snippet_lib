---
title: "Cover One Half-Open Integer Target with the Fewest Bounded Intervals"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
  - choose-a-deterministic-minimum-point-set-stabbing-bounded-closed-integer-intervals.md
  - select-a-minimum-cost-set-cover-with-canonical-ties.md
---

# Cover One Half-Open Integer Target with the Fewest Bounded Intervals

## Idea and Problem

Select the fewest supplied half-open intervals whose union continuously covers one target interval.

Start at the target's left frontier. Among every candidate that contains that
frontier, choose the one reaching farthest to the right, advance to that reach,
and repeat. Replacing the first interval of any feasible cover with this
farthest-reaching choice cannot increase the number of intervals still needed,
which gives the greedy algorithm its minimum-cardinality guarantee.

The deterministic tie is local: equal clipped reaches choose the smaller
original candidate index. It does not promise the globally lexicographically
smallest tuple among every optimal cover.

## When to Use

Use this algorithm when one continuous range must be covered with the fewest
items from a bounded collection of intervals, every candidate has the same
unit selection cost, and an uncovered gap should be reported cleanly. Examples
include choosing the fewest time windows, numeric range fragments, or retained
segments needed to span one requested extent.

Use a weighted interval algorithm when candidates have different costs. Use a
general set-cover solver when the universe is an arbitrary finite collection
rather than every point of one ordered interval. Those problems have different
structure and do not inherit this farthest-reach proof.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_CANDIDATES = 10_000


@dataclass(frozen=True, slots=True)
class HalfOpenInterval:
    start: int
    stop: int


def _validate_interval(name: str, interval: object) -> HalfOpenInterval:
    if type(interval) is not HalfOpenInterval:
        raise TypeError(f"{name} must be an exact HalfOpenInterval")
    if type(interval.start) is not int or type(interval.stop) is not int:
        raise TypeError(f"{name} endpoints must be exact integers")
    if not (
        _MIN_INT64 <= interval.start <= _MAX_INT64 and _MIN_INT64 <= interval.stop <= _MAX_INT64
    ):
        raise ValueError(f"{name} endpoints are outside the signed 64-bit range")
    if interval.start >= interval.stop:
        raise ValueError(f"{name} must be non-empty")
    return interval


def minimum_half_open_interval_cover(
    target: HalfOpenInterval,
    candidates: tuple[HalfOpenInterval, ...],
) -> tuple[int, ...] | None:
    """Return original indexes in greedy order, or None for an uncovered gap."""
    target = _validate_interval("target", target)
    if type(candidates) is not tuple:
        raise TypeError("candidates must be an exact tuple")
    if len(candidates) > _MAX_CANDIDATES:
        raise ValueError("candidate count exceeds the supported limit")

    indexed: list[tuple[int, HalfOpenInterval]] = []
    for index, candidate in enumerate(candidates):
        indexed.append((index, _validate_interval(f"candidates[{index}]", candidate)))
    indexed.sort(key=lambda item: (item[1].start, item[0]))

    selected: list[int] = []
    cursor = target.start
    scan = 0

    while cursor < target.stop:
        best_index: int | None = None
        best_reach = cursor

        while scan < len(indexed) and indexed[scan][1].start <= cursor:
            original_index, candidate = indexed[scan]
            clipped_reach = min(candidate.stop, target.stop)
            if clipped_reach > best_reach or (
                clipped_reach == best_reach
                and clipped_reach > cursor
                and (best_index is None or original_index < best_index)
            ):
                best_index = original_index
                best_reach = clipped_reach
            scan += 1

        if best_index is None:
            return None
        selected.append(best_index)
        cursor = best_reach

    return tuple(selected)
```

## Example

```python
def covers_target(
    target: HalfOpenInterval,
    candidates: tuple[HalfOpenInterval, ...],
    indexes: tuple[int, ...],
) -> bool:
    frontier = target.start
    chosen = sorted(
        (candidates[index] for index in indexes),
        key=lambda interval: (interval.start, interval.stop),
    )
    for interval in chosen:
        if interval.stop <= frontier:
            continue
        if interval.start > frontier:
            return False
        frontier = interval.stop
        if frontier >= target.stop:
            return True
    return False


def oracle_minimum_size(
    target: HalfOpenInterval,
    candidates: tuple[HalfOpenInterval, ...],
) -> int | None:
    from itertools import combinations

    for size in range(len(candidates) + 1):
        for indexes in combinations(range(len(candidates)), size):
            if covers_target(target, candidates, indexes):
                return size
    return None


target = HalfOpenInterval(1, 10)
candidates = (
    HalfOpenInterval(0, 4),
    HalfOpenInterval(0, 5),
    HalfOpenInterval(4, 8),
    HalfOpenInterval(5, 9),
    HalfOpenInterval(8, 12),
    HalfOpenInterval(20, 30),
)
selected = minimum_half_open_interval_cover(target, candidates)

tie_candidates = (
    HalfOpenInterval(-1, 5),
    HalfOpenInterval(0, 5),
    HalfOpenInterval(5, 10),
)
tie_selected = minimum_half_open_interval_cover(
    HalfOpenInterval(0, 10),
    tie_candidates,
)

assert selected == (1, 3, 4)
assert selected is not None and len(selected) == oracle_minimum_size(
    target,
    candidates,
)
assert tie_selected == (0, 2)
assert (
    minimum_half_open_interval_cover(
        HalfOpenInterval(0, 10),
        (HalfOpenInterval(0, 4), HalfOpenInterval(5, 10)),
    )
    is None
)
assert minimum_half_open_interval_cover(
    HalfOpenInterval(0, 10),
    (HalfOpenInterval(-5, 20),),
) == (0,)
```

## Trade-offs and Limitations

For `n` candidates, sorting takes `O(n log n)` time and `O(n)` memory. The
frontier sweep is linear because each sorted candidate is examined once. The
10,000-candidate cap bounds both costs, while signed-64 endpoints bound the
accepted coordinate domain. Python's comparisons avoid arithmetic overflow.

Coverage is continuous over the real extent represented by `[start, stop)`;
it is not a check of only the integer coordinates inside that interval. Empty
targets and empty candidate intervals are rejected so that every successful
step must make strict progress. Duplicate, containing, disjoint, and entirely
outside candidates are allowed and retain their original indexes.

The result minimizes only candidate count. The farthest-reach and
original-index rule fixes each local choice, but a different optimal cover can
have a lexicographically smaller complete index tuple. The function does not
support costs, priorities, circular ranges, online insertions, all-optimum
enumeration, or approximate gap filling.

## Related Snippets

<!-- catalog:related:start -->
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Choose a Deterministic Minimum Point Set Stabbing Bounded Closed Integer Intervals](choose-a-deterministic-minimum-point-set-stabbing-bounded-closed-integer-intervals.md)
- [Select a Minimum-Cost Set Cover with Canonical Ties](select-a-minimum-cost-set-cover-with-canonical-ties.md)
<!-- catalog:related:end -->
