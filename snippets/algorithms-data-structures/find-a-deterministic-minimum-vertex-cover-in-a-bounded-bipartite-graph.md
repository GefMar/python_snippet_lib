---
title: "Find a Deterministic Minimum Vertex Cover in a Bounded Bipartite Graph"
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
  - count-perfect-matchings-in-a-bounded-balanced-bipartite-graph-with-subset-dp.md
  - compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md
---

# Find a Deterministic Minimum Vertex Cover in a Bounded Bipartite Graph

## Idea and Problem

Select the fewest indexed vertices from two bipartite sides so every declared edge touches at least one selected vertex.

Kőnig's theorem equates the minimum vertex-cover size in a bipartite graph
with its maximum-matching size. An ascending augmenting-path search first
builds one deterministic maximum matching. Alternating reachability then
starts at unmatched left vertices, follows unmatched edges to the right, and
follows matched edges back to the left.

The resulting cover contains every unreachable left vertex and every reachable
right vertex. Sorting the validated edges and traversing indexes in ascending
order makes this particular minimum cover independent of edge declaration
order, without claiming that it is globally lexicographically first.

## When to Use

Use this function for a static bounded bipartite graph when every edge must be
covered by choosing as few endpoint vertices as possible. Independently sized
sides and isolated vertices are supported, so it fits small assignment,
incidence, and graph-structure checks without requiring a balanced graph.

Use the matching snippet when the selected edges, rather than a covering
vertex set, are the required result. General or weighted vertex cover needs a
different algorithm, and a specialized graph library is a better fit for much
larger, dynamic, or repeatedly queried graphs.

## Implementation

```python
from collections import deque

_MAX_COVER_SIDE_SIZE = 256
_MAX_COVER_EDGE_COUNT = 16_384

_CoverEdge = tuple[int, int]
_BipartiteCover = tuple[tuple[int, ...], tuple[int, ...]]


def deterministic_minimum_bipartite_vertex_cover(
    left_size: int,
    right_size: int,
    edges: tuple[_CoverEdge, ...],
) -> _BipartiteCover:
    """Return one edge-order-invariant minimum cover as left and right indexes."""
    if type(left_size) is not int or type(right_size) is not int:
        raise TypeError("side sizes must be exact integers")
    if not 0 <= left_size <= _MAX_COVER_SIDE_SIZE:
        raise ValueError("left_size is outside 0..256")
    if not 0 <= right_size <= _MAX_COVER_SIDE_SIZE:
        raise ValueError("right_size is outside 0..256")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_COVER_EDGE_COUNT:
        raise ValueError("edge count exceeds 16384")

    seen_edges: set[_CoverEdge] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")
        left, right = edge
        if type(left) is not int or type(right) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= left < left_size:
            raise ValueError(f"edges[{edge_index}] left endpoint is outside its side")
        if not 0 <= right < right_size:
            raise ValueError(f"edges[{edge_index}] right endpoint is outside its side")
        if edge in seen_edges:
            raise ValueError(f"edges[{edge_index}] duplicates an earlier edge")
        seen_edges.add(edge)

    adjacency_lists: list[list[int]] = [[] for _ in range(left_size)]
    for left, right in sorted(seen_edges):
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

    reachable_left = [False] * left_size
    reachable_right = [False] * right_size
    pending: deque[int] = deque()
    for left, matched_right in enumerate(matched_right_by_left):
        if matched_right == -1:
            reachable_left[left] = True
            pending.append(left)

    while pending:
        left = pending.popleft()
        matched_right = matched_right_by_left[left]
        for right in adjacency[left]:
            if right == matched_right or reachable_right[right]:
                continue
            reachable_right[right] = True
            next_left = matched_left_by_right[right]
            if next_left != -1 and not reachable_left[next_left]:
                reachable_left[next_left] = True
                pending.append(next_left)

    return (
        tuple(left for left, reached in enumerate(reachable_left) if not reached),
        tuple(right for right, reached in enumerate(reachable_right) if reached),
    )
```

## Example

```python
def minimum_cover_size_by_vertex_subsets(
    left_size: int,
    right_size: int,
    edges: tuple[_CoverEdge, ...],
) -> int:
    vertex_count = left_size + right_size
    for selected_count in range(vertex_count + 1):
        for selected_mask in range(1 << vertex_count):
            if selected_mask.bit_count() != selected_count:
                continue
            if all(
                selected_mask & (1 << left)
                or selected_mask & (1 << (left_size + right))
                for left, right in edges
            ):
                return selected_count
    raise AssertionError("selecting every vertex must cover every edge")


def maximum_matching_size_by_search(
    left_size: int,
    right_size: int,
    edges: tuple[_CoverEdge, ...],
) -> int:
    from functools import cache

    adjacency = tuple(
        tuple(right for right in range(right_size) if (left, right) in edges)
        for left in range(left_size)
    )

    @cache
    def search(left: int, used_rights: int) -> int:
        if left == left_size:
            return 0
        best = search(left + 1, used_rights)
        for right in adjacency[left]:
            right_bit = 1 << right
            if not used_rights & right_bit:
                best = max(best, 1 + search(left + 1, used_rights | right_bit))
        return best

    return search(0, 0)


def exercise_every_tiny_graph() -> int:
    from itertools import product

    checked = 0
    for left_size in range(4):
        for right_size in range(4):
            possible_edges = tuple(product(range(left_size), range(right_size)))
            for edge_flags in range(1 << len(possible_edges)):
                edges = tuple(
                    edge
                    for edge_index, edge in enumerate(possible_edges)
                    if edge_flags & (1 << edge_index)
                )
                cover = deterministic_minimum_bipartite_vertex_cover(
                    left_size,
                    right_size,
                    edges,
                )
                reversed_cover = deterministic_minimum_bipartite_vertex_cover(
                    left_size,
                    right_size,
                    tuple(reversed(edges)),
                )
                left_cover, right_cover = cover
                assert cover == reversed_cover
                assert all(
                    left in left_cover or right in right_cover
                    for left, right in edges
                )
                assert len(left_cover) + len(right_cover) == (
                    minimum_cover_size_by_vertex_subsets(
                        left_size,
                        right_size,
                        edges,
                    )
                )
                assert len(left_cover) + len(right_cover) == (
                    maximum_matching_size_by_search(
                        left_size,
                        right_size,
                        edges,
                    )
                )
                checked += 1
    return checked


maximum_edges = tuple(
    (left, right)
    for left in range(128)
    for right in range(128)
)
maximum_edge_cover = deterministic_minimum_bipartite_vertex_cover(
    128,
    128,
    maximum_edges,
)
side_boundary_edges = tuple(
    (index, index) for index in range(_MAX_COVER_SIDE_SIZE)
)
side_boundary_cover = deterministic_minimum_bipartite_vertex_cover(
    _MAX_COVER_SIDE_SIZE,
    _MAX_COVER_SIDE_SIZE,
    side_boundary_edges,
)

rejected = 0
for left_size, right_size, edges in (
    (True, 0, ()),
    (0, True, ()),
    (_MAX_COVER_SIDE_SIZE + 1, 0, ()),
    (1, 1, [(0, 0)]),
    (1, 1, ((0, 0), (0, 0))),
    (1, 1, ((1, 0),)),
    (1, 1, ((0, True),)),
    (1, 1, ((0, 0),) * (_MAX_COVER_EDGE_COUNT + 1)),
):
    try:
        deterministic_minimum_bipartite_vertex_cover(
            left_size,
            right_size,
            edges,
        )
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_every_tiny_graph(),
    deterministic_minimum_bipartite_vertex_cover(0, 0, ()),
    deterministic_minimum_bipartite_vertex_cover(3, 4, ()),
    maximum_edge_cover,
    side_boundary_cover,
    rejected,
) == (
    689,
    ((), ()),
    ((), ()),
    (tuple(range(128)), ()),
    (tuple(range(_MAX_COVER_SIDE_SIZE)), ()),
    8,
)
```

## Trade-offs and Limitations

Sorting `E` distinct edges takes `O(E * log(E))` time. The ascending
augmenting-path phase takes `O(L * E)` worst-case time, and alternating
reachability adds `O(L + R + E)`. Adjacency, matching, visitation, queue, and
validation state retain `O(L + R + E)` references. Recursion is bounded by
the smaller admitted side and reaches at most 256 nested augmenting calls.

The two returned tuples contain sorted zero-based indexes. Their combined size
equals the maximum-matching size, but another valid construction can return a
different minimum cover. This result is deterministic under edge permutation;
it is not promised to be the globally lexicographically first optimum.

The function accepts only a static simple bipartite graph. It does not return
the matching, enumerate all covers, attach vertex weights, solve general-graph
vertex cover, derive a maximum independent set, or update after edge changes.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
- [Count Perfect Matchings in a Bounded Balanced Bipartite Graph with Subset DP](count-perfect-matchings-in-a-bounded-balanced-bipartite-graph-with-subset-dp.md)
- [Compute a Bounded Integer Maximum Flow and Canonical Minimum Cut with Edmonds-Karp](compute-a-bounded-integer-maximum-flow-and-canonical-minimum-cut-with-edmonds-karp.md)
<!-- catalog:related:end -->
