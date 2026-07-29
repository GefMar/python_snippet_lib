---
title: "Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md
  - build-and-evaluate-a-bounded-binary-assignment-constraint-system.md
  - match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md
---

# Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph

## Idea and Problem

Match as many left vertices as possible to distinct adjacent right vertices while making the chosen maximum independent of edge input order.

A depth-first augmenting-path search either reaches an unused right vertex or
reroutes an earlier match to make room. Sorting every adjacency list and
visiting left vertices in ascending order fixes the traversal decisions, while
the augmenting-path property guarantees maximum cardinality.

## When to Use

Use this algorithm for a small, fully materialized bipartite graph whose
vertices already have stable integer indexes. It fits deterministic assignment
checks where every edge has equal value and unmatched vertices are allowed.

Use maximum flow when capacities or a broader network model are already part
of the problem. Choose a matching library for large or dense graphs, weighted
or minimum-cost objectives, general graphs, dynamic updates, or a required
canonical optimum stronger than this traversal rule.

## Implementation

```python
from dataclasses import dataclass

_MAX_SIDE_SIZE = 256
_MAX_EDGES = 16_384


@dataclass(frozen=True, slots=True, order=True)
class BipartiteEdge:
    left: int
    right: int


def maximum_cardinality_bipartite_matching(
    left_size: int,
    right_size: int,
    edges: tuple[BipartiteEdge, ...],
) -> tuple[BipartiteEdge, ...]:
    """Return one deterministic maximum-cardinality matching."""
    if type(left_size) is not int or type(right_size) is not int:
        raise TypeError("side sizes must be exact integers")
    if not 1 <= left_size <= _MAX_SIDE_SIZE:
        raise ValueError("left_size is outside the supported range")
    if not 1 <= right_size <= _MAX_SIDE_SIZE:
        raise ValueError("right_size is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    validated_pairs: set[tuple[int, int]] = set()
    for index, edge in enumerate(edges):
        if type(edge) is not BipartiteEdge:
            raise TypeError(f"edges[{index}] must be an exact BipartiteEdge")
        if type(edge.left) is not int or type(edge.right) is not int:
            raise TypeError(f"edges[{index}] indexes must be exact integers")
        if not 0 <= edge.left < left_size:
            raise ValueError(f"edges[{index}].left is outside the left side")
        if not 0 <= edge.right < right_size:
            raise ValueError(f"edges[{index}].right is outside the right side")
        pair = (edge.left, edge.right)
        if pair in validated_pairs:
            raise ValueError(f"edges[{index}] duplicates an earlier edge")
        validated_pairs.add(pair)

    adjacency_lists: list[list[int]] = [[] for _ in range(left_size)]
    for left, right in sorted(validated_pairs):
        adjacency_lists[left].append(right)
    adjacency = tuple(tuple(neighbors) for neighbors in adjacency_lists)

    matched_left_by_right = [-1] * right_size
    seen_at_search = [0] * right_size

    def augment(left: int, search_number: int) -> bool:
        for right in adjacency[left]:
            if seen_at_search[right] == search_number:
                continue
            seen_at_search[right] = search_number
            previous_left = matched_left_by_right[right]
            if previous_left == -1 or augment(previous_left, search_number):
                matched_left_by_right[right] = left
                return True
        return False

    for left in range(left_size):
        augment(left, left + 1)

    matched_right_by_left = [-1] * left_size
    for right, left in enumerate(matched_left_by_right):
        if left != -1:
            matched_right_by_left[left] = right

    return tuple(
        BipartiteEdge(left, right)
        for left, right in enumerate(matched_right_by_left)
        if right != -1
    )
```

## Example

```python
graph_edges = (
    BipartiteEdge(0, 0),
    BipartiteEdge(0, 1),
    BipartiteEdge(1, 0),
    BipartiteEdge(1, 2),
    BipartiteEdge(2, 1),
)

matching = maximum_cardinality_bipartite_matching(3, 3, graph_edges)
reordered = maximum_cardinality_bipartite_matching(3, 3, tuple(reversed(graph_edges)))

assert (
    matching
    == reordered
    == (
        BipartiteEdge(0, 0),
        BipartiteEdge(1, 2),
        BipartiteEdge(2, 1),
    )
)
```

## Trade-offs and Limitations

Sorting `E` distinct edges and running one depth-first search per left vertex
takes `O(E log E + L + R + L * E)` time in the worst case, including bounded
array initialization and result construction. Epoch-marked visitation avoids
clearing an `R`-element array before every search. Adjacency, matching arrays,
visitation state, and recursion use `O(L + R + E)` memory. The fixed side limit
keeps augmenting-path recursion at no more than 256 nested calls.

The result has maximum cardinality and is invariant to edge tuple order, but
another deterministic traversal can select different pairs with the same
cardinality. This implementation does not compute a globally lexicographically
smallest maximum, attach weights or costs, solve stable or general-graph
matching, update a graph incrementally, or guarantee that a perfect matching
exists. Duplicate edges are rejected rather than silently coalesced.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp](compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md)
- [Build and Evaluate a Bounded Binary Assignment Constraint System](build-and-evaluate-a-bounded-binary-assignment-constraint-system.md)
- [Match Strict Mutual Nearest Neighbors with a Comparison Budget](match-strict-mutual-nearest-neighbors-with-a-comparison-budget.md)
<!-- catalog:related:end -->
