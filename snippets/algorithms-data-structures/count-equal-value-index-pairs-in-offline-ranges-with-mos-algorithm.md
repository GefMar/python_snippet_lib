---
title: "Count Equal-Value Index Pairs in Offline Ranges with Mo's Algorithm"
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
  - count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md
  - answer-static-half-open-range-minimum-queries-with-a-sparse-table.md
  - count-strict-inversions-in-a-bounded-integer-sequence.md
---

# Count Equal-Value Index Pairs in Offline Ranges with Mo's Algorithm

## Idea and Problem

Answer many static range queries by moving one shared half-open window and maintaining the number of equal-value index pairs inside it.

If a value currently occurs `c` times, adding another occurrence creates `c`
new unordered pairs. Removing one occurrence destroys `c - 1` pairs. Mo's
ordering groups queries by their left-endpoint block and alternates the
right-endpoint direction between blocks, reducing pointer movement while
preserving the original answer order.

## When to Use

Use this algorithm when one immutable sequence has a large batch of known
half-open range queries and each answer must count pairs of distinct positions
`i < j` whose values are equal. It is especially useful when a frequency-based
window aggregate is easy to update but has no simple invertible prefix form.

Use a direct `Counter` for a few short ranges. Prefer a specialized prefix or
Fenwick-tree method when the aggregate depends only on presence, and choose a
dynamic data structure when values change between queries. Mo's algorithm is
an offline technique: it materializes and reorders the complete query batch.

## Implementation

```python
from math import isqrt

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_MO_VALUES = 20_000
_MAX_MO_QUERIES = 20_000

type HalfOpenQuery = tuple[int, int]


def count_equal_value_pairs_offline(
    values: tuple[int, ...],
    queries: tuple[HalfOpenQuery, ...],
) -> tuple[int, ...]:
    """Return equal-value index-pair counts in declared query order."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_MO_VALUES:
        raise ValueError("value count exceeds the supported limit")
    for value_index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{value_index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{value_index}] is outside signed 64-bit range")

    if type(queries) is not tuple:
        raise TypeError("queries must be an exact tuple")
    if len(queries) > _MAX_MO_QUERIES:
        raise ValueError("query count exceeds the supported limit")
    for query_index, query in enumerate(queries):
        if type(query) is not tuple:
            raise TypeError(f"queries[{query_index}] must be an exact tuple")
        if len(query) != 2:
            raise ValueError(f"queries[{query_index}] must contain two bounds")
        start, stop = query
        if type(start) is not int or type(stop) is not int:
            raise TypeError(f"queries[{query_index}] bounds must be exact integers")
        if not 0 <= start <= stop <= len(values):
            raise ValueError(f"queries[{query_index}] is outside the value sequence")

    block_width = max(1, isqrt(max(1, len(values))))

    def query_key(query_index: int) -> tuple[int, int, int]:
        start, stop = queries[query_index]
        block = start // block_width
        ordered_stop = stop if block % 2 == 0 else -stop
        return block, ordered_stop, query_index

    order = sorted(range(len(queries)), key=query_key)
    answers = [0] * len(queries)
    frequencies: dict[int, int] = {}
    current_start = 0
    current_stop = 0
    pair_count = 0

    def add(value: int) -> None:
        nonlocal pair_count
        old_frequency = frequencies.get(value, 0)
        pair_count += old_frequency
        frequencies[value] = old_frequency + 1

    def remove(value: int) -> None:
        nonlocal pair_count
        old_frequency = frequencies[value]
        new_frequency = old_frequency - 1
        pair_count -= new_frequency
        if new_frequency:
            frequencies[value] = new_frequency
        else:
            del frequencies[value]

    for query_index in order:
        start, stop = queries[query_index]
        while current_start > start:
            current_start -= 1
            add(values[current_start])
        while current_stop < stop:
            add(values[current_stop])
            current_stop += 1
        while current_start < start:
            remove(values[current_start])
            current_start += 1
        while current_stop > stop:
            current_stop -= 1
            remove(values[current_stop])
        answers[query_index] = pair_count

    return tuple(answers)
```

## Example

```python
def pair_count_oracle(values: tuple[int, ...], start: int, stop: int) -> int:
    from collections import Counter

    frequencies = Counter(values[start:stop])
    return sum(count * (count - 1) // 2 for count in frequencies.values())


sample_values = (4, 1, 4, 2, 1, 4)
sample_queries = ((0, 6), (0, 3), (1, 5), (3, 3), (0, 6), (5, 6))
assert count_equal_value_pairs_offline(sample_values, sample_queries) == (
    4,
    1,
    1,
    0,
    4,
    0,
)
assert count_equal_value_pairs_offline((), ((0, 0), (0, 0))) == (0, 0)

# Exhaust every range of every short sequence over a small alphabet.
for length in range(6):
    ranges = tuple(
        (start, stop) for start in range(length + 1) for stop in range(start, length + 1)
    )
    for encoded in range(3**length):
        remaining = encoded
        candidate: list[int] = []
        for _ in range(length):
            candidate.append((-1, 0, 1)[remaining % 3])
            remaining //= 3
        values = tuple(candidate)
        expected = tuple(pair_count_oracle(values, *query) for query in ranges)
        assert count_equal_value_pairs_offline(values, ranges) == expected

maximum_values = (7,) * _MAX_MO_VALUES
maximum_queries = ((0, _MAX_MO_VALUES),) * _MAX_MO_QUERIES
expected_maximum = _MAX_MO_VALUES * (_MAX_MO_VALUES - 1) // 2
assert (
    count_equal_value_pairs_offline(maximum_values, maximum_queries)
    == (expected_maximum,) * _MAX_MO_QUERIES
)
```

## Trade-offs and Limitations

With block width near `sqrt(n)`, the conventional bound is
`O((n + q) * sqrt(n))` window updates after `O(q log q)` sorting, though the
actual movement depends on the query distribution. Frequencies, ordering and
answers require `O(n + q)` memory in the worst case. The alternating block
order is deterministic but is not guaranteed to minimize movement.

Counts describe unordered pairs of indexes, not distinct values or distinct
value pairs. A value appearing `c` times contributes `c * (c - 1) // 2`, so
the result can be quadratic in range length even though Python integers do not
overflow. Inputs are limited to exact signed 64-bit integers and validated
half-open ranges; arbitrary hashable values, online queries, point updates and
subarray materialization are outside this focused contract.

## Related Snippets

<!-- catalog:related:start -->
- [Count Distinct Values in Bounded Half-Open Ranges Offline with a Fenwick Tree](count-distinct-values-in-bounded-half-open-ranges-offline-with-a-fenwick-tree.md)
- [Answer Static Half-Open Range-Minimum Queries with a Sparse Table](answer-static-half-open-range-minimum-queries-with-a-sparse-table.md)
- [Count Strict Inversions in a Bounded Integer Sequence](count-strict-inversions-in-a-bounded-integer-sequence.md)
<!-- catalog:related:end -->
