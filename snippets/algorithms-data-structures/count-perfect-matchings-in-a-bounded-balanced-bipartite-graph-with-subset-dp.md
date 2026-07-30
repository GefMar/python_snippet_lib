---
title: "Count Perfect Matchings in a Bounded Balanced Bipartite Graph with Subset DP"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md
  - find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md
  - count-topological-orders-of-a-bounded-directed-graph-with-subset-dp.md
---

# Count Perfect Matchings in a Bounded Balanced Bipartite Graph with Subset DP

## Idea and Problem

Count every labeled perfect matching between two equally sized indexed vertex sets without materializing the matchings.

A bit mask records which right-side vertices have already been used. Its set
bit count identifies the next left-side vertex, so every partial assignment
with the same mask has the same legal continuations. Adding their counts into
successor masks counts each complete bijection exactly once.

## When to Use

Use this dynamic program for one balanced bipartite graph of at most 20
vertices per side when the exact number of perfect assignments matters. It is
useful for small combinatorial checks, schedule alternatives, and independent
test oracles where enumerating every matching would be too large.

Use an augmenting-path matcher when only existence or one maximum-cardinality
matching is needed. Use assignment or flow algorithms for costs, capacities,
rectangular sides, or larger graphs. The exponential subset table is not a
production-scale matching representation.

## Implementation

```python
_MAX_PERFECT_MATCHING_SIDE_SIZE = 20
_MAX_PERFECT_MATCHING_EDGE_COUNT = 400


def count_perfect_bipartite_matchings(
    side_size: int,
    edges: tuple[tuple[int, int], ...],
) -> int:
    """Return the exact number of labeled perfect matchings."""
    if type(side_size) is not int:
        raise TypeError("side_size must be an exact integer")
    if not 0 <= side_size <= _MAX_PERFECT_MATCHING_SIDE_SIZE:
        raise ValueError("side_size is outside 0..20")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_PERFECT_MATCHING_EDGE_COUNT:
        raise ValueError("edge count exceeds 400")

    right_masks = [0] * side_size
    seen_edges: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")

        left, right = edge
        if type(left) is not int or type(right) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= left < side_size or not 0 <= right < side_size:
            raise ValueError(f"edges[{edge_index}] endpoint is outside its side")
        if edge in seen_edges:
            raise ValueError(f"edges[{edge_index}] duplicates an earlier edge")
        seen_edges.add(edge)
        right_masks[left] |= 1 << right

    full_mask = (1 << side_size) - 1
    prefix_counts = [0] * (full_mask + 1)
    prefix_counts[0] = 1

    for used_rights, prefix_count in enumerate(prefix_counts):
        left = used_rights.bit_count()
        if left == side_size or not prefix_count:
            continue

        available = right_masks[left] & (full_mask ^ used_rights)
        while available:
            right_bit = available & -available
            prefix_counts[used_rights | right_bit] += prefix_count
            available ^= right_bit

    return prefix_counts[full_mask]
```

## Example

```python
def count_matchings_by_permutation(
    side_size: int,
    edges: tuple[tuple[int, int], ...],
) -> int:
    from itertools import permutations

    allowed = set(edges)
    return sum(
        all((left, right) in allowed for left, right in enumerate(assignment))
        for assignment in permutations(range(side_size))
    )


def exercise_every_small_graph() -> int:
    from itertools import product

    checked = 0
    for side_size in range(5):
        possible_edges = tuple(product(range(side_size), repeat=2))
        for edge_flags in range(1 << len(possible_edges)):
            edges = tuple(
                edge
                for edge_index, edge in enumerate(possible_edges)
                if edge_flags & (1 << edge_index)
            )
            assert count_perfect_bipartite_matchings(
                side_size,
                edges,
            ) == count_matchings_by_permutation(side_size, edges)
            checked += 1
    return checked


maximum_complete_edges = tuple(
    (left, right)
    for left in range(_MAX_PERFECT_MATCHING_SIDE_SIZE)
    for right in range(_MAX_PERFECT_MATCHING_SIDE_SIZE)
)
unique_edges = ((2, 2), (0, 0), (3, 3), (1, 1))

duplicate_rejected = False
try:
    count_perfect_bipartite_matchings(2, ((0, 1), (0, 1)))
except ValueError:
    duplicate_rejected = True


def factorial_by_product(value: int) -> int:
    result = 1
    for factor in range(2, value + 1):
        result *= factor
    return result

assert (
    exercise_every_small_graph(),
    count_perfect_bipartite_matchings(0, ()),
    count_perfect_bipartite_matchings(4, ()),
    count_perfect_bipartite_matchings(4, unique_edges),
    count_perfect_bipartite_matchings(4, tuple(reversed(unique_edges))),
    count_perfect_bipartite_matchings(
        _MAX_PERFECT_MATCHING_SIDE_SIZE,
        maximum_complete_edges,
    ),
    duplicate_rejected,
) == (
    66_067,
    1,
    0,
    1,
    1,
    factorial_by_product(_MAX_PERFECT_MATCHING_SIDE_SIZE),
    True,
)
```

## Trade-offs and Limitations

For side size `N` and `E` declared edges, validation takes expected `O(E)` set
work. The dynamic program performs `O(N * 2**N)` possible transitions and
retains `O(2**N + N + E)` Python integer or collection references. Exact
counts can approach `N!`, so their additions and retained values are not
constant-cost as their bit lengths grow.

Vertices are distinct labels `0` through `N - 1` on each side. Edge order has
no meaning, and duplicate declarations are rejected. The empty balanced graph
has one empty perfect matching; a non-empty graph without a perfect assignment
returns zero.

The function returns no witness and does not enumerate matchings. It does not
find a merely maximum-cardinality matching, minimize costs, accept rectangular
sides, represent parallel edges or capacities, sample matchings, or update the
graph incrementally.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
- [Find a Lexicographically First Minimum-Cost Perfect Assignment by Bitmask DP](find-a-lexicographically-first-minimum-cost-perfect-assignment-by-bitmask-dp.md)
- [Count Topological Orders of a Bounded Directed Graph with Subset DP](count-topological-orders-of-a-bounded-directed-graph-with-subset-dp.md)
<!-- catalog:related:end -->
