---
title: "Maintain Bounded Range Additions and Leftmost Minima with a Lazy Segment Tree"
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
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - answer-static-half-open-range-minimum-queries-with-a-sparse-table.md
---

# Maintain Bounded Range Additions and Leftmost Minima with a Lazy Segment Tree

## Idea and Problem

Apply exact integer additions to half-open index ranges and report the leftmost minimum in any requested non-empty range without visiting every affected item.

A lazy segment tree stores the minimum value and its leftmost absolute index for
each node. A fully covered update changes that minimum immediately and records a
pending addition instead of descending. The addition is pushed to the children
only when a later partial update or query needs their values. Comparing
`(value, index)` pairs makes equal-minimum results deterministic.

## When to Use

Use this for a bounded in-memory sequence with many interleaved range additions
and range-minimum queries. It is useful for offset adjustments, sweep-line
scores, scheduling slack, and other workloads where every item in a contiguous
span changes by the same amount and ties must resolve to the earliest index.

Use a plain list when the operation batch is small. A difference array is
simpler when all updates precede all reads. Prefer a Fenwick tree for additive
prefix or range sums, and use the point-replacement segment-tree variant when
updates replace individual values instead of shifting whole ranges.

## Implementation

```python
from dataclasses import dataclass

_MIN_RANGE_ADD_INT64 = -(1 << 63)
_MAX_RANGE_ADD_INT64 = (1 << 63) - 1
_MAX_RANGE_ADD_VALUES = 65_536
_MAX_RANGE_ADD_OPERATIONS = 65_536


@dataclass(frozen=True, slots=True)
class RangeAdd:
    start: int
    stop: int
    delta: int


@dataclass(frozen=True, slots=True)
class RangeMinimum:
    start: int
    stop: int


@dataclass(frozen=True, slots=True)
class IndexedMinimum:
    value: int
    index: int


def _leftmost_minimum(
    left: IndexedMinimum | None,
    right: IndexedMinimum | None,
) -> IndexedMinimum | None:
    if left is None:
        return right
    if right is None:
        return left
    return min(left, right, key=lambda item: (item.value, item.index))


def _range_add_bounds(start: object, stop: object, length: int, field: str) -> tuple[int, int]:
    if type(start) is not int or type(stop) is not int:
        raise TypeError(f"{field} bounds must be exact integers")
    if not 0 <= start < stop <= length:
        raise ValueError(f"{field} must satisfy 0 <= start < stop <= length")
    return start, stop


def resolve_range_add_minimum_operations(
    values: tuple[int, ...],
    operations: tuple[RangeAdd | RangeMinimum, ...],
) -> tuple[IndexedMinimum, ...]:
    """Apply range additions and answer leftmost range-minimum queries."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_RANGE_ADD_VALUES:
        raise ValueError("value count is outside 1..65,536")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_RANGE_ADD_INT64 <= value <= _MAX_RANGE_ADD_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    if type(operations) is not tuple:
        raise TypeError("operations must be an exact tuple")
    if len(operations) > _MAX_RANGE_ADD_OPERATIONS:
        raise ValueError("operations contain more than 65,536 items")

    leaf_count = 1 << (len(values) - 1).bit_length()
    minima: list[IndexedMinimum | None] = [None] * (2 * leaf_count)
    pending_additions = [0] * (2 * leaf_count)
    for index, value in enumerate(values):
        minima[leaf_count + index] = IndexedMinimum(value, index)
    for node in range(leaf_count - 1, 0, -1):
        minima[node] = _leftmost_minimum(minima[2 * node], minima[2 * node + 1])

    def apply(node: int, delta: int) -> None:
        current = minima[node]
        if current is None:
            raise RuntimeError("an update reached an unused tree node")
        minima[node] = IndexedMinimum(current.value + delta, current.index)
        pending_additions[node] += delta

    def push(node: int) -> None:
        delta = pending_additions[node]
        if delta:
            apply(2 * node, delta)
            apply(2 * node + 1, delta)
            pending_additions[node] = 0

    def add_range(
        node: int,
        node_start: int,
        node_stop: int,
        update_start: int,
        update_stop: int,
        delta: int,
    ) -> None:
        if update_start <= node_start and node_stop <= update_stop:
            apply(node, delta)
            return
        push(node)
        middle = (node_start + node_stop) // 2
        if update_start < middle:
            add_range(2 * node, node_start, middle, update_start, update_stop, delta)
        if middle < update_stop:
            add_range(2 * node + 1, middle, node_stop, update_start, update_stop, delta)
        minima[node] = _leftmost_minimum(minima[2 * node], minima[2 * node + 1])

    def minimum_in_range(
        node: int,
        node_start: int,
        node_stop: int,
        query_start: int,
        query_stop: int,
    ) -> IndexedMinimum | None:
        if query_start <= node_start and node_stop <= query_stop:
            return minima[node]
        push(node)
        middle = (node_start + node_stop) // 2
        best: IndexedMinimum | None = None
        if query_start < middle:
            best = minimum_in_range(2 * node, node_start, middle, query_start, query_stop)
        if middle < query_stop:
            best = _leftmost_minimum(
                best,
                minimum_in_range(
                    2 * node + 1,
                    middle,
                    node_stop,
                    query_start,
                    query_stop,
                ),
            )
        return best

    answers: list[IndexedMinimum] = []
    for operation_index, operation in enumerate(operations):
        field = f"operations[{operation_index}]"
        if type(operation) is RangeAdd:
            start, stop = _range_add_bounds(operation.start, operation.stop, len(values), field)
            if type(operation.delta) is not int:
                raise TypeError(f"{field}.delta must be an exact integer")
            if not _MIN_RANGE_ADD_INT64 <= operation.delta <= _MAX_RANGE_ADD_INT64:
                raise ValueError(f"{field}.delta is outside the signed 64-bit range")
            add_range(1, 0, leaf_count, start, stop, operation.delta)
        elif type(operation) is RangeMinimum:
            start, stop = _range_add_bounds(operation.start, operation.stop, len(values), field)
            answer = minimum_in_range(1, 0, leaf_count, start, stop)
            if answer is None:
                raise RuntimeError("a validated non-empty range produced no minimum")
            answers.append(answer)
        else:
            raise TypeError(f"{field} must be an exact RangeAdd or RangeMinimum")
    return tuple(answers)
```

## Example

```python
from itertools import product
from random import Random


def direct_range_add_oracle(
    values: tuple[int, ...],
    operations: tuple[RangeAdd | RangeMinimum, ...],
) -> tuple[IndexedMinimum, ...]:
    current = list(values)
    answers: list[IndexedMinimum] = []
    for operation in operations:
        if type(operation) is RangeAdd:
            for index in range(operation.start, operation.stop):
                current[index] += operation.delta
        else:
            value = min(current[operation.start : operation.stop])
            index = current.index(value, operation.start, operation.stop)
            answers.append(IndexedMinimum(value, index))
    return tuple(answers)


exhaustive_checked = 0
for size in range(1, 5):
    ranges = tuple((start, stop) for start in range(size) for stop in range(start + 1, size + 1))
    all_queries = tuple(RangeMinimum(start, stop) for start, stop in ranges)
    for small_values in product((-1, 0, 1), repeat=size):
        for update_start, update_stop in ranges:
            for delta in (-2, 0, 2):
                operations = (
                    *all_queries,
                    RangeAdd(update_start, update_stop, delta),
                    *all_queries,
                )
                assert resolve_range_add_minimum_operations(
                    small_values,
                    operations,
                ) == direct_range_add_oracle(small_values, operations)
                exhaustive_checked += 1

rng = Random(0)
random_checked = 0
for _ in range(300):
    size = rng.randrange(1, 65)
    values = tuple(rng.randrange(-1_000_000, 1_000_001) for _ in range(size))
    mixed_operations: list[RangeAdd | RangeMinimum] = []
    for _ in range(rng.randrange(1, 129)):
        start = rng.randrange(size)
        stop = rng.randrange(start + 1, size + 1)
        if rng.randrange(5) < 3:
            mixed_operations.append(RangeAdd(start, stop, rng.randrange(-(10**12), 10**12 + 1)))
        else:
            mixed_operations.append(RangeMinimum(start, stop))
    mixed_operations.append(RangeMinimum(0, size))
    operations = tuple(mixed_operations)
    assert resolve_range_add_minimum_operations(
        values,
        operations,
    ) == direct_range_add_oracle(values, operations)
    random_checked += 1

known_operations = (
    RangeMinimum(0, 4),
    RangeAdd(0, 2, -3),
    RangeMinimum(0, 4),
    RangeAdd(2, 4, -3),
    RangeMinimum(0, 4),
    RangeMinimum(2, 4),
    RangeAdd(1, 3, 10),
    RangeMinimum(0, 4),
    RangeMinimum(2, 3),
)
known_answers = resolve_range_add_minimum_operations((5, 1, 1, 4), known_operations)

overflow_answers = resolve_range_add_minimum_operations(
    (_MAX_RANGE_ADD_INT64, _MIN_RANGE_ADD_INT64),
    (
        RangeAdd(0, 2, _MAX_RANGE_ADD_INT64),
        RangeAdd(1, 2, _MIN_RANGE_ADD_INT64),
        RangeAdd(0, 1, _MAX_RANGE_ADD_INT64),
        RangeMinimum(0, 1),
        RangeMinimum(0, 2),
    ),
)

maximum_add = RangeAdd(0, _MAX_RANGE_ADD_VALUES, _MAX_RANGE_ADD_INT64)
maximum_operations = (maximum_add,) * (_MAX_RANGE_ADD_OPERATIONS - 1) + (
    RangeMinimum(0, _MAX_RANGE_ADD_VALUES),
)
maximum_answers = resolve_range_add_minimum_operations(
    (0,) * _MAX_RANGE_ADD_VALUES,
    maximum_operations,
)


def rejects(values: object, operations: object) -> bool:
    try:
        resolve_range_add_minimum_operations(values, operations)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


invalid_calls = (
    ((), ()),
    ([1], ()),
    ((True,), ()),
    ((_MIN_RANGE_ADD_INT64 - 1,), ()),
    ((_MAX_RANGE_ADD_INT64 + 1,), ()),
    ((0,) * (_MAX_RANGE_ADD_VALUES + 1), ()),
    ((1,), []),
    ((1,), (RangeMinimum(0, 1),) * (_MAX_RANGE_ADD_OPERATIONS + 1)),
    ((1,), ((0, 1),)),
    ((1,), (RangeMinimum(True, 1),)),
    ((1,), (RangeMinimum(0, 0),)),
    ((1,), (RangeMinimum(0, 2),)),
    ((1,), (RangeAdd(True, 1, 0),)),
    ((1,), (RangeAdd(0, 1, True),)),
    ((1,), (RangeAdd(0, 1, _MIN_RANGE_ADD_INT64 - 1),)),
    ((1,), (RangeAdd(0, 1, _MAX_RANGE_ADD_INT64 + 1),)),
    ((1,), (RangeAdd(-1, 1, 0),)),
)

assert (
    exhaustive_checked,
    random_checked,
    known_answers,
    overflow_answers,
    maximum_answers,
    sum(rejects(values, operations) for values, operations in invalid_calls),
) == (
    3_006,
    300,
    (
        IndexedMinimum(1, 1),
        IndexedMinimum(-2, 1),
        IndexedMinimum(-2, 1),
        IndexedMinimum(-2, 2),
        IndexedMinimum(1, 3),
        IndexedMinimum(8, 2),
    ),
    (
        IndexedMinimum(3 * _MAX_RANGE_ADD_INT64, 0),
        IndexedMinimum(_MIN_RANGE_ADD_INT64 - 1, 1),
    ),
    (IndexedMinimum((_MAX_RANGE_ADD_OPERATIONS - 1) * _MAX_RANGE_ADD_INT64, 0),),
    len(invalid_calls),
)
```

## Trade-offs and Limitations

Building the tree and validating inputs takes `O(N + Q)` time. Each addition
or minimum query visits `O(log N)` nodes, so the complete batch takes
`O(N + Q log N)` time. The power-of-two tree holds `O(N)` minima and pending
additions; returned query records require `O(Q)` output space. Recursive depth
is at most 16 under the declared value cap.

Initial values and declared deltas must be signed 64-bit integers, but repeated
updates accumulate as exact Python integers and may exceed that range. Ranges
are non-empty and half-open. The implementation always breaks equal-value ties
by absolute index, including after overlapping lazy updates.

This batch interface does not expose point replacement, range assignment,
range sums, insertion, deletion, persistence, snapshots, concurrent access, or
an identity value for empty ranges. It constructs fresh detached tree state for
every call.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Answer Static Half-Open Range-Minimum Queries with a Sparse Table](answer-static-half-open-range-minimum-queries-with-a-sparse-table.md)
<!-- catalog:related:end -->
