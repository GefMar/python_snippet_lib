---
title: "Find a Lexicographically First Minimum-Cost Perfect Assignment by Bitmask DP"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md
  - compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md
  - build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
---

# Find a Lexicographically First Minimum-Cost Perfect Assignment by Bitmask DP

## Idea and Problem

Assign every row of a small square integer cost matrix to a different column while minimizing exact total cost and resolving ties reproducibly.

A bitmask records which columns have already been assigned. Because the next
row is the mask's population count, each mask needs only one suffix optimum.
After computing those costs from larger masks to smaller masks, reconstruction
scans available columns in ascending order and chooses the first one that
preserves the optimum.

That forward choice returns the lexicographically first `columns_by_row` among
all assignments with minimum total cost without storing a complete path at
every dynamic-programming state.

## When to Use

Use this algorithm for a complete square assignment problem of order at most
18 when exact integer costs, a transparent implementation, and a specified tie
rule matter more than polynomial scaling. Rows and columns must already have
stable zero-based meanings, and every row-column pairing must be available.

This fits small planning, comparison, and test-oracle problems. Use a Hungarian
algorithm, minimum-cost-flow implementation, or optimization library for
larger matrices, forbidden pairs, capacities, rectangular inputs, additional
constraints, dual certificates, or repeatedly changing costs.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_ASSIGNMENT_ORDER = 18


def minimum_cost_perfect_assignment(
    cost_matrix: tuple[tuple[int, ...], ...],
) -> tuple[int, tuple[int, ...]]:
    """Return minimum total cost and lexicographically first columns by row."""
    if type(cost_matrix) is not tuple:
        raise TypeError("cost_matrix must be an exact tuple")
    order = len(cost_matrix)
    if not 1 <= order <= _MAX_ASSIGNMENT_ORDER:
        raise ValueError("matrix order is outside the supported range")
    for row_index, row in enumerate(cost_matrix):
        if type(row) is not tuple:
            raise TypeError(f"cost_matrix[{row_index}] must be an exact tuple")
        if len(row) != order:
            raise ValueError("cost_matrix must be square")
        for column_index, cost in enumerate(row):
            if type(cost) is not int:
                raise TypeError(
                    f"cost_matrix[{row_index}][{column_index}] must be an exact integer"
                )
            if not _MIN_INT64 <= cost <= _MAX_INT64:
                raise ValueError(
                    f"cost_matrix[{row_index}][{column_index}] is outside the signed 64-bit range"
                )

    full_mask = (1 << order) - 1
    suffix_costs = [0] * (full_mask + 1)

    for mask in range(full_mask - 1, -1, -1):
        row_index = mask.bit_count()
        best_cost: int | None = None
        available = full_mask ^ mask
        while available:
            column_bit = available & -available
            column_index = column_bit.bit_length() - 1
            candidate = cost_matrix[row_index][column_index] + suffix_costs[mask | column_bit]
            if best_cost is None or candidate < best_cost:
                best_cost = candidate
            available ^= column_bit
        if best_cost is None:
            raise AssertionError("an incomplete assignment must have a free column")
        suffix_costs[mask] = best_cost

    columns_by_row: list[int] = []
    mask = 0
    while mask != full_mask:
        row_index = len(columns_by_row)
        for column_index in range(order):
            column_bit = 1 << column_index
            if mask & column_bit:
                continue
            candidate = cost_matrix[row_index][column_index] + suffix_costs[mask | column_bit]
            if candidate == suffix_costs[mask]:
                columns_by_row.append(column_index)
                mask |= column_bit
                break
        else:
            raise AssertionError("an optimal continuation must exist")

    return suffix_costs[0], tuple(columns_by_row)
```

## Example

```python
ordinary = ((4, 1, 3), (2, 0, 5), (3, 2, 2))
all_tied = ((0, 0, 0), (0, 0, 0), (0, 0, 0))
negative_tie = ((-1, -1), (-1, -1))
boundary = tuple(
    tuple(0 if row == column else 1 for column in range(_MAX_ASSIGNMENT_ORDER))
    for row in range(_MAX_ASSIGNMENT_ORDER)
)

assert (
    minimum_cost_perfect_assignment(ordinary),
    minimum_cost_perfect_assignment(all_tied),
    minimum_cost_perfect_assignment(negative_tie),
    minimum_cost_perfect_assignment(((_MAX_INT64,),)),
    minimum_cost_perfect_assignment(boundary),
) == (
    (5, (1, 0, 2)),
    (0, (0, 1, 2)),
    (-2, (0, 1)),
    (_MAX_INT64, (0,)),
    (0, tuple(range(_MAX_ASSIGNMENT_ORDER))),
)
```

## Trade-offs and Limitations

For matrix order `n`, the dynamic program examines `O(n * 2^n)` transitions
and stores `O(2^n)` exact costs plus an `O(n)` result. The admitted order of 18
still has 262,144 masks. Python integers preserve totals beyond signed 64-bit
range, but arithmetic cost grows with their bit length.

Forward reconstruction is essential to the tie rule: it chooses the smallest
available column compatible with the already proven suffix optimum. Changing
iteration order or reconstructing backward can return a different optimal
assignment.

The matrix is complete and square. There is no representation for unavailable
pairs, no impossible result, and no support for capacities or unmatched rows.
The exponential state space makes this implementation unsuitable for large
production assignment problems even when their cost matrix is dense.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
- [Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp](compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md)
- [Build and Evaluate a Bounded Binary Assignment Constraint System](build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
<!-- catalog:related:end -->
