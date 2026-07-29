---
title: "Find Every Centroid of a Bounded Undirected Tree"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md
  - partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md
  - traverse-a-parent-graph-with-breadth-first-search.md
---

# Find Every Centroid of a Bounded Undirected Tree

## Idea and Problem

Find the one or two vertices that split a validated bounded tree into the smallest possible largest remaining component.

Removing a vertex separates its child subtrees and the part above it. After an
iterative traversal fixes an arbitrary root, reverse traversal order supplies
every child subtree size. For each vertex, the largest resulting component is
the maximum child-subtree size or the complement outside its own subtree.
Minimizing that value gives exactly the tree centroids.

Centroids depend only on undirected connectivity. Reordering declarations or
reversing any endpoint pair therefore cannot change the ascending result.
A valid tree has one centroid, or two adjacent centroids when the best balance
falls on both sides of one edge.

## When to Use

Use this algorithm for a complete static tree when a balanced vertex is needed
for divide-and-conquer, recursive partition planning, or an independently
checkable structural summary. Integer vertex indexes must already identify the
closed vertex set, and every undirected edge must be available at once.

A tree centroid minimizes the largest component after vertex removal. It is
not the tree center, which minimizes the farthest path distance, and it is not
a geometric centroid or coordinate average. Use a dynamic-tree structure when
edges change, or a graph-specific partitioning method when cycles, weights,
capacities, or geometric coordinates are part of the objective.

## Implementation

```python
_MAX_TREE_VERTICES = 100_000
_UNSEEN_PARENT = -2
_NO_PARENT = -1


def find_tree_centroids(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    """Return every centroid of one validated simple undirected tree."""
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
    order: list[int] = []
    stack = [0]
    while stack:
        vertex = stack.pop()
        order.append(vertex)
        for neighbor in adjacency[vertex]:
            if parents[neighbor] != _UNSEEN_PARENT:
                continue
            parents[neighbor] = vertex
            stack.append(neighbor)

    if len(order) != vertex_count:
        raise ValueError("edges must form one connected tree")

    subtree_sizes = [1] * vertex_count
    largest_components = [0] * vertex_count
    for vertex in reversed(order):
        largest_child_subtree = 0
        for neighbor in adjacency[vertex]:
            if parents[neighbor] != vertex:
                continue
            child_size = subtree_sizes[neighbor]
            subtree_sizes[vertex] += child_size
            largest_child_subtree = max(largest_child_subtree, child_size)

        outside_subtree = vertex_count - subtree_sizes[vertex]
        largest_components[vertex] = max(largest_child_subtree, outside_subtree)

    best_balance = min(largest_components)
    return tuple(
        vertex
        for vertex, largest_component in enumerate(largest_components)
        if largest_component == best_balance
    )
```

## Example

```python
def centroids_by_vertex_removal(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
    for first, second in edges:
        adjacency[first].append(second)
        adjacency[second].append(first)

    balances: list[int] = []
    for removed in range(vertex_count):
        seen = [False] * vertex_count
        seen[removed] = True
        largest_component = 0
        for start in range(vertex_count):
            if seen[start]:
                continue
            seen[start] = True
            component_size = 0
            pending = [start]
            while pending:
                vertex = pending.pop()
                component_size += 1
                for neighbor in adjacency[vertex]:
                    if not seen[neighbor]:
                        seen[neighbor] = True
                        pending.append(neighbor)
            largest_component = max(largest_component, component_size)
        balances.append(largest_component)

    best_balance = min(balances)
    return tuple(vertex for vertex, balance in enumerate(balances) if balance == best_balance)


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


def exercise_prufer_trees() -> int:
    from itertools import product

    assert find_tree_centroids(1, ()) == (0,)
    checked = 1
    for vertex_count in range(2, 8):
        for sequence in product(range(vertex_count), repeat=vertex_count - 2):
            edges = tree_from_prufer(sequence)
            expected = centroids_by_vertex_removal(vertex_count, edges)
            reversed_declarations = tuple((second, first) for first, second in reversed(edges))
            assert find_tree_centroids(vertex_count, edges) == expected
            assert find_tree_centroids(vertex_count, reversed_declarations) == expected
            checked += 1
    return checked


maximum_path = tuple((vertex, vertex + 1) for vertex in range(_MAX_TREE_VERTICES - 1))
maximum_star = tuple((0, vertex) for vertex in range(1, _MAX_TREE_VERTICES))
path_centroids = find_tree_centroids(_MAX_TREE_VERTICES, maximum_path)
star_centroids = find_tree_centroids(_MAX_TREE_VERTICES, maximum_star)

value_errors = 0
for vertex_count, invalid_edges in (
    (0, ()),
    (_MAX_TREE_VERTICES + 1, ()),
    (3, ((0, 1),)),
    (3, ((0, 1), (1, 0))),
    (4, ((0, 1), (1, 2), (2, 0))),
    (2, ((0, 2),)),
):
    try:
        find_tree_centroids(vertex_count, invalid_edges)
    except ValueError:
        value_errors += 1

try:
    find_tree_centroids(2, ((0, True),))
except TypeError:
    boolean_endpoint_rejected = True
else:
    boolean_endpoint_rejected = False

assert (
    exercise_prufer_trees(),
    path_centroids,
    star_centroids,
    value_errors,
    boolean_endpoint_rejected,
) == (
    18_249,
    (49_999, 50_000),
    (0,),
    6,
    True,
)
```

## Trade-offs and Limitations

Validation, adjacency construction, traversal, and postorder accumulation use
expected `O(V + E)` time and `O(V + E)` memory. Duplicate detection relies on
expected constant-time hashing of normalized endpoint pairs. Iterative stacks
avoid recursion depth limits on a 100,000-vertex path.

The complement size must be computed only after all child subtree sizes have
been accumulated. Rooting at vertex zero is an internal traversal choice and
does not affect the centroid definition, ascending result, edge-order
invariance, or endpoint-orientation invariance.

This function accepts only one complete simple unweighted tree. It does not
process a forest or arbitrary graph, use edge or vertex weights, return paths
or a diameter, find the minimum-eccentricity tree center, compute a geometric
centroid, maintain dynamic edges, or build a recursive centroid decomposition.

## Related Snippets

<!-- catalog:related:start -->
- [Find Articulation Points and Bridges in a Bounded Undirected Graph](find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md)
- [Partition a Bounded Undirected Graph into Deterministic Components with Union-Find](partition-a-bounded-undirected-graph-into-deterministic-components-with-union-find.md)
- [Traverse a Parent Graph with Breadth-First Search](traverse-a-parent-graph-with-breadth-first-search.md)
<!-- catalog:related:end -->
