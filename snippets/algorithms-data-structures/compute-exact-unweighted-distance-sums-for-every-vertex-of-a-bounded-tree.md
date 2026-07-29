---
title: "Compute Exact Unweighted Distance Sums for Every Vertex of a Bounded Tree"
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
  - traverse-a-parent-graph-with-breadth-first-search.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
---

# Compute Exact Unweighted Distance Sums for Every Vertex of a Bounded Tree

## Idea and Problem

Compute the sum of unweighted shortest-path distances from every vertex of a validated bounded tree without running a separate traversal from every source.

After an iterative traversal roots the tree at vertex zero, the root's answer
is the sum of all depths. A reverse pass accumulates every subtree size. Moving
the root across one parent-child edge makes each vertex in the child's subtree
one step closer and every other vertex one step farther, which gives
`answer[child] = answer[parent] + V - 2 * subtree_size[child]`.

The root is only an internal calculation choice. Every returned value depends
solely on undirected connectivity, so edge declaration order and endpoint
orientation cannot change the vertex-ordered result.

## When to Use

Use this algorithm for a complete static unweighted tree when every vertex
needs its total graph distance to the whole vertex set. The result can support
bounded facility-location comparisons, structural summaries, or validation
against another tree representation without materializing a pairwise distance
matrix.

Use repeated shortest-path searches for a general graph, and use a weighted
rerooting formula when edges carry lengths. Choose a dynamic-tree structure
when edges change between queries.

## Implementation

```python
_MAX_TREE_VERTICES = 100_000
_UNSEEN_PARENT = -2
_NO_PARENT = -1


def tree_distance_sums(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    """Return each vertex's sum of distances to all tree vertices."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 1 <= vertex_count <= _MAX_TREE_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if len(edges) != vertex_count - 1:
        raise ValueError("a tree must contain exactly vertex_count - 1 edges")

    normalized_edges: list[tuple[int, int]] = []
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
        normalized_edges.append(normalized)

    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in normalized_edges:
        adjacency[first].append(second)
        adjacency[second].append(first)

    parents = [_UNSEEN_PARENT] * vertex_count
    parents[0] = _NO_PARENT
    depths = [0] * vertex_count
    order: list[int] = []
    stack = [0]
    while stack:
        vertex = stack.pop()
        order.append(vertex)
        for neighbor in adjacency[vertex]:
            if parents[neighbor] != _UNSEEN_PARENT:
                continue
            parents[neighbor] = vertex
            depths[neighbor] = depths[vertex] + 1
            stack.append(neighbor)

    if len(order) != vertex_count:
        raise ValueError("edges must form one connected tree")

    subtree_sizes = [1] * vertex_count
    for vertex in reversed(order[1:]):
        subtree_sizes[parents[vertex]] += subtree_sizes[vertex]

    answers = [0] * vertex_count
    answers[0] = sum(depths)
    for vertex in order[1:]:
        parent = parents[vertex]
        answers[vertex] = answers[parent] + vertex_count - 2 * subtree_sizes[vertex]
    return tuple(answers)
```

## Example

```python
def distance_sums_by_bfs(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    from collections import deque

    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in edges:
        adjacency[first].append(second)
        adjacency[second].append(first)

    answers: list[int] = []
    for source in range(vertex_count):
        distances = [-1] * vertex_count
        distances[source] = 0
        pending = deque([source])
        while pending:
            vertex = pending.popleft()
            for neighbor in adjacency[vertex]:
                if distances[neighbor] != -1:
                    continue
                distances[neighbor] = distances[vertex] + 1
                pending.append(neighbor)
        answers.append(sum(distances))
    return tuple(answers)


def tree_from_prufer(sequence: tuple[int, ...]) -> tuple[tuple[int, int], ...]:
    vertex_count = len(sequence) + 2
    degrees = [1] * vertex_count
    for vertex in sequence:
        degrees[vertex] += 1

    edges: list[tuple[int, int]] = []
    for vertex in sequence:
        leaf = next(index for index, degree in enumerate(degrees) if degree == 1)
        edges.append((leaf, vertex))
        degrees[leaf] -= 1
        degrees[vertex] -= 1

    remaining = [index for index, degree in enumerate(degrees) if degree == 1]
    edges.append((remaining[0], remaining[1]))
    return tuple(edges)


def exercise_labelled_trees() -> int:
    from itertools import product

    assert tree_distance_sums(1, ()) == (0,)
    checked = 1
    for vertex_count in range(2, 8):
        for sequence in product(range(vertex_count), repeat=vertex_count - 2):
            edges = tree_from_prufer(sequence)
            expected = distance_sums_by_bfs(vertex_count, edges)
            reoriented = tuple((second, first) for first, second in reversed(edges))
            assert tree_distance_sums(vertex_count, edges) == expected
            assert tree_distance_sums(vertex_count, reoriented) == expected
            checked += 1
    return checked


small_path = ((0, 1), (1, 2), (2, 3))
reversed_path = tuple((second, first) for first, second in reversed(small_path))

maximum_path = tuple((vertex, vertex + 1) for vertex in range(_MAX_TREE_VERTICES - 1))
maximum_path_sums = tree_distance_sums(_MAX_TREE_VERTICES, maximum_path)
path_boundary_matches = maximum_path_sums == tuple(
    vertex * (vertex + 1) // 2
    + (_MAX_TREE_VERTICES - 1 - vertex) * (_MAX_TREE_VERTICES - vertex) // 2
    for vertex in range(_MAX_TREE_VERTICES)
)

maximum_star = tuple((0, vertex) for vertex in range(1, _MAX_TREE_VERTICES))
maximum_star_sums = tree_distance_sums(_MAX_TREE_VERTICES, maximum_star)
star_boundary_matches = maximum_star_sums[0] == _MAX_TREE_VERTICES - 1 and all(
    distance_sum == 2 * _MAX_TREE_VERTICES - 3 for distance_sum in maximum_star_sums[1:]
)

value_errors = 0
for invalid_vertex_count, invalid_edges in (
    (0, ()),
    (_MAX_TREE_VERTICES + 1, ()),
    (3, ((0, 1),)),
    (3, ((0, 1), (1, 0))),
    (2, ((0, 0),)),
    (2, ((0, 2),)),
    (4, ((0, 1), (1, 2), (2, 0))),
):
    try:
        tree_distance_sums(invalid_vertex_count, invalid_edges)
    except ValueError:
        value_errors += 1

type_errors = 0
for invalid_vertex_count, invalid_edges in (
    (True, ()),
    (2, ([0, 1],)),
    (2, ((0, True),)),
):
    try:
        tree_distance_sums(invalid_vertex_count, invalid_edges)
    except TypeError:
        type_errors += 1

assert (
    exercise_labelled_trees(),
    tree_distance_sums(4, small_path),
    tree_distance_sums(4, reversed_path),
    path_boundary_matches,
    star_boundary_matches,
    value_errors,
    type_errors,
) == (
    18_249,
    (6, 4, 4, 6),
    (6, 4, 4, 6),
    True,
    True,
    7,
    3,
)
```

## Trade-offs and Limitations

Validation, adjacency construction, traversal, subtree accumulation, and
rerooting use expected `O(V + E)` time and `O(V + E)` memory. A valid tree
has `E = V - 1`, so both bounds are linear in `V`. Duplicate detection
relies on expected constant-time hashing of normalized endpoint pairs, and the
iterative traversal avoids recursion-depth failures on a 100,000-vertex path.

Python integers keep every sum exact. The largest possible answer is
`V * (V - 1) / 2`, attained at a path endpoint, but addition cost still grows
with integer width rather than remaining a unit-cost machine operation.

This function accepts one complete simple undirected unweighted tree. It does
not process a forest, directed or cyclic graph, use edge weights, return paths
or a pairwise matrix, maintain dynamic edges, find a diameter or
minimum-eccentricity center, or select a centroid. Minimizing total distance is
a different objective from minimizing the largest component after removal.

## Related Snippets

<!-- catalog:related:start -->
- [Find Every Centroid of a Bounded Undirected Tree](find-every-centroid-of-a-bounded-undirected-tree.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
<!-- catalog:related:end -->
