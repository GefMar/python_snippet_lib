---
title: "Find a Canonical Diameter Path in a Bounded Undirected Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-every-centroid-of-a-bounded-undirected-tree.md
  - compute-exact-unweighted-distance-sums-for-every-vertex-of-a-bounded-tree.md
  - traverse-a-parent-graph-with-breadth-first-search.md
---

# Find a Canonical Diameter Path in a Bounded Undirected Tree

## Idea and Problem

Return one longest simple path of a validated bounded tree while resolving multiple diameter choices by the smallest ordered endpoint pair.

A traversal from any vertex reaches a diameter endpoint; a second traversal
from that endpoint reveals the diameter length and another endpoint. Distances
from those two endpoints also determine every vertex's eccentricity: in a
tree, it is the larger of the two distances. The vertices whose eccentricity
equals the diameter are exactly the possible diameter endpoints.

Choosing the smallest such vertex and then its smallest farthest vertex fixes
the canonical endpoint pair. The unique tree path between them fixes the full
result, independently of edge declaration order or endpoint orientation.

## When to Use

Use this function when a complete, static, unweighted tree needs one stable
longest path for structural analysis, worst-case hop planning, test fixtures,
or deterministic decomposition. Vertices must be the closed integer range from
zero through `vertex_count - 1`, and all undirected edges must be available at
once.

The diameter maximizes distance between two vertices. It is different from a
centroid, which balances component sizes after removal, and from a center,
which minimizes maximum distance. Use weighted-tree traversals when edges have
costs, and graph-diameter algorithms when cycles are allowed.

## Implementation

```python
from collections import deque

_MAX_DIAMETER_VERTICES = 100_000
_UNVISITED = -1


def _tree_distances(
    adjacency: list[list[int]],
    start: int,
) -> tuple[list[int], list[int]]:
    distances = [_UNVISITED] * len(adjacency)
    parents = [_UNVISITED] * len(adjacency)
    distances[start] = 0
    parents[start] = start
    pending: deque[int] = deque([start])

    while pending:
        vertex = pending.popleft()
        for neighbor in adjacency[vertex]:
            if distances[neighbor] != _UNVISITED:
                continue
            distances[neighbor] = distances[vertex] + 1
            parents[neighbor] = vertex
            pending.append(neighbor)
    return distances, parents


def find_canonical_tree_diameter(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    """Return the diameter path with the smallest ordered endpoint pair."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 1 <= vertex_count <= _MAX_DIAMETER_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if len(edges) != vertex_count - 1:
        raise ValueError("a tree must contain exactly vertex_count - 1 edges")

    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    seen_edges: set[tuple[int, int]] = set()
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")
        first, second = edge
        if type(first) is not int:
            raise TypeError(f"edges[{edge_index}].first must be an exact integer")
        if type(second) is not int:
            raise TypeError(f"edges[{edge_index}].second must be an exact integer")
        if not 0 <= first < vertex_count:
            raise ValueError(f"edges[{edge_index}].first is outside the tree")
        if not 0 <= second < vertex_count:
            raise ValueError(f"edges[{edge_index}].second is outside the tree")
        if first == second:
            raise ValueError(f"edges[{edge_index}] is a self-edge")

        normalized = (first, second) if first < second else (second, first)
        if normalized in seen_edges:
            raise ValueError(f"edges[{edge_index}] duplicates an undirected edge")
        seen_edges.add(normalized)
        adjacency[first].append(second)
        adjacency[second].append(first)

    distances_from_zero, _ = _tree_distances(adjacency, 0)
    if _UNVISITED in distances_from_zero:
        raise ValueError("edges must form one connected tree")

    first_seed_distance = max(distances_from_zero)
    first_seed = min(
        vertex
        for vertex, distance in enumerate(distances_from_zero)
        if distance == first_seed_distance
    )
    distances_from_first_seed, _ = _tree_distances(adjacency, first_seed)
    diameter_length = max(distances_from_first_seed)
    second_seed = min(
        vertex
        for vertex, distance in enumerate(distances_from_first_seed)
        if distance == diameter_length
    )
    distances_from_second_seed, _ = _tree_distances(adjacency, second_seed)

    first_endpoint = min(
        vertex
        for vertex in range(vertex_count)
        if max(
            distances_from_first_seed[vertex],
            distances_from_second_seed[vertex],
        )
        == diameter_length
    )
    distances_from_first, parents = _tree_distances(adjacency, first_endpoint)
    second_endpoint = min(
        vertex
        for vertex, distance in enumerate(distances_from_first)
        if distance == diameter_length
    )

    reversed_path = [second_endpoint]
    while reversed_path[-1] != first_endpoint:
        reversed_path.append(parents[reversed_path[-1]])
    return tuple(reversed(reversed_path))
```

## Example

```python
def tree_from_prufer(sequence: tuple[int, ...]) -> tuple[tuple[int, int], ...]:
    from heapq import heapify, heappop, heappush

    vertex_count = len(sequence) + 2
    degrees = [1] * vertex_count
    for vertex in sequence:
        degrees[vertex] += 1
    leaves = [vertex for vertex, degree in enumerate(degrees) if degree == 1]
    heapify(leaves)

    edges: list[tuple[int, int]] = []
    for vertex in sequence:
        leaf = heappop(leaves)
        edges.append((leaf, vertex))
        degrees[leaf] -= 1
        degrees[vertex] -= 1
        if degrees[vertex] == 1:
            heappush(leaves, vertex)
    edges.append((heappop(leaves), heappop(leaves)))
    return tuple(edges)


def canonical_diameter_by_all_pairs(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    if vertex_count == 1:
        return (0,)
    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in edges:
        adjacency[first].append(second)
        adjacency[second].append(first)

    best_pair = (0, 1)
    best_distance = -1
    best_parents: list[int] = []
    for start in range(vertex_count):
        distances, parents = _tree_distances(adjacency, start)
        for stop in range(start + 1, vertex_count):
            candidate = (start, stop)
            if distances[stop] > best_distance or (
                distances[stop] == best_distance and candidate < best_pair
            ):
                best_distance = distances[stop]
                best_pair = candidate
                best_parents = parents

    reversed_path = [best_pair[1]]
    while reversed_path[-1] != best_pair[0]:
        reversed_path.append(best_parents[reversed_path[-1]])
    return tuple(reversed(reversed_path))


def exercise_small_labelled_trees() -> int:
    from itertools import product

    assert find_canonical_tree_diameter(1, ()) == (0,)
    checked = 1
    for vertex_count in range(2, 8):
        for sequence in product(range(vertex_count), repeat=vertex_count - 2):
            edges = tree_from_prufer(sequence)
            expected = canonical_diameter_by_all_pairs(vertex_count, edges)
            reversed_declarations = tuple((second, first) for first, second in reversed(edges))
            assert find_canonical_tree_diameter(vertex_count, edges) == expected
            assert find_canonical_tree_diameter(vertex_count, reversed_declarations) == expected
            checked += 1
    return checked


maximum_path = tuple((vertex, vertex + 1) for vertex in range(_MAX_DIAMETER_VERTICES - 1))
maximum_star = tuple((0, vertex) for vertex in range(1, _MAX_DIAMETER_VERTICES))
path_diameter = find_canonical_tree_diameter(_MAX_DIAMETER_VERTICES, maximum_path)
star_diameter = find_canonical_tree_diameter(_MAX_DIAMETER_VERTICES, maximum_star)

value_errors = 0
for vertex_count, invalid_edges in (
    (0, ()),
    (3, ((0, 1),)),
    (3, ((0, 1), (1, 0))),
    (4, ((0, 1), (1, 2), (2, 0))),
    (2, ((0, 2),)),
    (2, ((0, 0),)),
):
    try:
        find_canonical_tree_diameter(vertex_count, invalid_edges)
    except ValueError:
        value_errors += 1

type_errors = 0
for vertex_count, invalid_edges in (
    (True, ()),
    (2, [(0, 1)]),
    (2, ((0, True),)),
):
    try:
        find_canonical_tree_diameter(vertex_count, invalid_edges)
    except TypeError:
        type_errors += 1

assert (
    exercise_small_labelled_trees(),
    len(path_diameter),
    path_diameter[:3],
    path_diameter[-3:],
    star_diameter,
    value_errors,
    type_errors,
) == (
    18_249,
    100_000,
    (0, 1, 2),
    (99_997, 99_998, 99_999),
    (1, 0, 2),
    6,
    3,
)
```

## Trade-offs and Limitations

Validation and four breadth-first traversals take `O(n)` time for a tree with
`n` vertices and `n - 1` edges. The adjacency lists, distance lists, parent
list and returned path use `O(n)` memory. Four traversals are a deliberate
constant-factor cost for choosing the globally smallest diameter endpoint pair
rather than merely returning the first diameter found by one two-sweep search.

The result is canonical only for the declared integer labels: relabeling the
same abstract tree can select a different endpoint pair. The path is unique
once its endpoints are chosen, but other diameter paths can have the same
length. The function materializes all edges and the complete returned path.

Only one simple, connected, undirected and unweighted static tree is accepted.
The function does not handle forests, parallel edges, self-edges, cycles,
weighted distances, negative costs, changing edges, recursion-oriented node
objects, graph diameter, tree centers, or centroid decomposition.

## Related Snippets

<!-- catalog:related:start -->
- [Find Every Centroid of a Bounded Undirected Tree](find-every-centroid-of-a-bounded-undirected-tree.md)
- [Compute Exact Unweighted Distance Sums for Every Vertex of a Bounded Tree](compute-exact-unweighted-distance-sums-for-every-vertex-of-a-bounded-tree.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
