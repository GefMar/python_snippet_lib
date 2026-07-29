---
title: "Answer Static Half-Open Range-Minimum Queries with a Sparse Table"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
  - build-a-canonical-min-cartesian-tree-parent-map-for-bounded-integers.md
  - compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md
---

# Answer Static Half-Open Range-Minimum Queries with a Sparse Table

## Idea and Problem

Precompute overlapping power-of-two blocks so every static range-minimum query returns its value and leftmost position in constant time.

For a non-empty half-open range, choose the greatest power of two that fits its
length. One block starts at the range's left boundary and another ends at its
right boundary. Those blocks may overlap, but taking a minimum is idempotent,
so their two stored answers still cover the complete range correctly.

Each table entry is a pair of value and original index. Normal tuple ordering
therefore chooses the smaller value first and the leftmost index when values
tie.

## When to Use

Use this function when a fully materialized signed-integer sequence will not
change and a bounded batch contains enough range-minimum queries to justify
preprocessing. It is useful for offline analysis, immutable snapshots, and
repeatable checks where equal minima need one stable position.

Use a direct scan for a few short queries. Use a segment tree when point values
can change, or another range structure when the operation is not idempotent.
The complete values and query batches must fit in memory before this function
starts building its sparse table.

## Implementation

```python
from typing import TypeAlias

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_STATIC_RMQ_VALUES = 100_000
_MAX_STATIC_RMQ_QUERIES = 100_000

RangeMinimum: TypeAlias = tuple[int, int]
HalfOpenRange: TypeAlias = tuple[int, int]


def answer_static_range_minimum_queries(
    values: tuple[int, ...],
    queries: tuple[HalfOpenRange, ...],
) -> tuple[RangeMinimum, ...]:
    """Return the value and leftmost index of every declared range minimum."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_STATIC_RMQ_VALUES:
        raise ValueError("value count is outside the supported range")

    for value_index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{value_index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{value_index}] is outside the signed 64-bit range")

    if type(queries) is not tuple:
        raise TypeError("queries must be an exact tuple")
    if len(queries) > _MAX_STATIC_RMQ_QUERIES:
        raise ValueError("query count exceeds the supported limit")

    checked_queries: list[HalfOpenRange] = []
    for query_index, query in enumerate(queries):
        if type(query) is not tuple:
            raise TypeError(f"queries[{query_index}] must be an exact tuple")
        if len(query) != 2:
            raise ValueError(f"queries[{query_index}] must contain two bounds")
        start, stop = query
        if type(start) is not int:
            raise TypeError(f"queries[{query_index}].start must be an exact integer")
        if type(stop) is not int:
            raise TypeError(f"queries[{query_index}].stop must be an exact integer")
        if not 0 <= start < stop <= len(values):
            raise ValueError(
                f"queries[{query_index}] must satisfy 0 <= start < stop <= value count"
            )
        checked_queries.append((start, stop))

    if not checked_queries:
        return ()

    levels: list[tuple[RangeMinimum, ...]] = [
        tuple((value, index) for index, value in enumerate(values))
    ]
    block_length = 2
    while block_length <= len(values):
        half_length = block_length // 2
        previous = levels[-1]
        levels.append(
            tuple(
                min(previous[start], previous[start + half_length])
                for start in range(len(values) - block_length + 1)
            )
        )
        block_length *= 2

    answers: list[RangeMinimum] = []
    for start, stop in checked_queries:
        query_length = stop - start
        level = query_length.bit_length() - 1
        selected_length = 1 << level
        answers.append(
            min(
                levels[level][start],
                levels[level][stop - selected_length],
            )
        )
    return tuple(answers)
```

## Example

```python
def brute_range_minimum(
    values: tuple[int, ...],
    start: int,
    stop: int,
) -> RangeMinimum:
    return min((values[index], index) for index in range(start, stop))


def exercise_small_static_ranges() -> int:
    from itertools import product

    checked = 0
    for value_count in range(1, 7):
        queries = tuple(
            (start, stop)
            for start in range(value_count)
            for stop in range(start + 1, value_count + 1)
        )
        for values in product((-1, 0, 1), repeat=value_count):
            expected = tuple(brute_range_minimum(values, start, stop) for start, stop in queries)
            assert answer_static_range_minimum_queries(values, queries) == expected
            checked += len(queries)
    return checked


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


ordinary_values = (7, 3, 9, 3, 5, -2, -2)
ordinary_queries = ((0, 7), (1, 5), (2, 5), (5, 7), (4, 5))
ordinary_answers = answer_static_range_minimum_queries(
    ordinary_values,
    ordinary_queries,
)

maximum_values = (_MAX_INT64,) * (_MAX_STATIC_RMQ_VALUES // 2) + (_MIN_INT64,) * (
    _MAX_STATIC_RMQ_VALUES // 2
)
maximum_queries = ((0, _MAX_STATIC_RMQ_VALUES),) * _MAX_STATIC_RMQ_QUERIES
maximum_answers = answer_static_range_minimum_queries(
    maximum_values,
    maximum_queries,
)

validation_rejections = (
    raises(
        TypeError,
        lambda: answer_static_range_minimum_queries([1], ()),
    ),
    raises(
        ValueError,
        lambda: answer_static_range_minimum_queries((), ()),
    ),
    raises(
        TypeError,
        lambda: answer_static_range_minimum_queries((True,), ()),
    ),
    raises(
        ValueError,
        lambda: answer_static_range_minimum_queries((_MAX_INT64 + 1,), ()),
    ),
    raises(
        TypeError,
        lambda: answer_static_range_minimum_queries((1,), [(0, 1)]),
    ),
    raises(
        TypeError,
        lambda: answer_static_range_minimum_queries((1,), ([0, 1],)),
    ),
    raises(
        TypeError,
        lambda: answer_static_range_minimum_queries((1,), ((False, 1),)),
    ),
    raises(
        ValueError,
        lambda: answer_static_range_minimum_queries((1,), ((0, 0),)),
    ),
    raises(
        ValueError,
        lambda: answer_static_range_minimum_queries((1,), ((0, 2),)),
    ),
    raises(
        ValueError,
        lambda: answer_static_range_minimum_queries(
            (0,) * (_MAX_STATIC_RMQ_VALUES + 1),
            (),
        ),
    ),
    raises(
        ValueError,
        lambda: answer_static_range_minimum_queries(
            (0,),
            ((0, 1),) * (_MAX_STATIC_RMQ_QUERIES + 1),
        ),
    ),
)

assert (
    exercise_small_static_ranges(),
    ordinary_answers,
    answer_static_range_minimum_queries((5,), ((0, 1),)),
    answer_static_range_minimum_queries((5,), ()),
    len(maximum_answers),
    set(maximum_answers),
    all(validation_rejections),
) == (
    19_956,
    ((-2, 5), (3, 1), (3, 3), (-2, 5), (5, 4)),
    ((5, 0),),
    (),
    100_000,
    {(_MIN_INT64, 50_000)},
    True,
)
```

## Trade-offs and Limitations

For a non-empty query batch, preprocessing uses O(n log n) time and memory.
Each query then takes O(1) time, and materializing all answers uses O(q)
additional memory. An empty query batch returns after complete input validation
without building the table. Signed 64-bit bounds apply to stored values, while
the algorithm only compares them and cannot overflow through arithmetic.

The O(1) query cost is purchased with substantially more memory than a segment
tree and only applies while the source tuple remains unchanged. The function
rebuilds the table on every call; a reusable index object is a better interface
when several query batches share the same values.

Only non-empty half-open ranges are accepted. The implementation does not
support updates, empty-range identities, generic comparison keys, range sums,
two-dimensional queries, streaming values, compact native numeric storage, or
concurrent mutation.

## Related Snippets

<!-- catalog:related:start -->
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
- [Build a Canonical Min-Cartesian-Tree Parent Map for Bounded Integers](build-a-canonical-min-cartesian-tree-parent-map-for-bounded-integers.md)
- [Compute Full-Window Trailing Maxima with a Monotonic Index Deque](compute-full-window-trailing-maxima-with-a-monotonic-index-deque.md)
<!-- catalog:related:end -->
