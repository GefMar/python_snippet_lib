---
title: "Find a Deterministic Minimum Vertex-Disjoint Path Cover of a Bounded DAG"
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
  - find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md
  - resolve-stable-ordering-constraints-with-topological-sort.md
  - compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md
---

# Find a Deterministic Minimum Vertex-Disjoint Path Cover of a Bounded DAG

## Idea and Problem

Cover every vertex of a bounded directed acyclic graph with the fewest vertex-disjoint directed paths, independently of edge input order.

Split each vertex into a left and right copy and find a maximum matching using
the graph's directed edges. Every match joins one vertex to its successor in a
path, so a matching of size `M` yields `V - M` paths. Sorted adjacency and
ascending augmenting searches fix one reproducible maximum. Starting from
vertices unmatched on the right and following matched successors then
reconstructs the complete cover, including singleton paths.

## When to Use

Use this algorithm for a small, fully materialized DAG with stable integer
vertex indexes when the goal is to minimize the number of disjoint execution,
dependency, or lineage chains. It also works as a compact oracle for testing a
larger graph implementation.

Confirm that every vertex must occur exactly once and that consecutive
vertices must follow declared edges. Use a different formulation when paths
may overlap, edges carry weights, the graph is cyclic, or the desired objective
is minimum total cost rather than minimum path count.

## Implementation

```python
_MAX_PATH_COVER_VERTICES = 256
_MAX_PATH_COVER_EDGES = 4_096


def minimum_vertex_disjoint_path_cover(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, ...], ...]:
    """Return one input-order-independent minimum path cover of a DAG."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 0 <= vertex_count <= _MAX_PATH_COVER_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_PATH_COVER_EDGES:
        raise ValueError("edge count exceeds the supported limit")

    unique_edges: set[tuple[int, int]] = set()
    for index, edge in enumerate(edges):
        if type(edge) is not tuple or len(edge) != 2:
            raise TypeError(f"edges[{index}] must be an exact two-item tuple")
        source, target = edge
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"edges[{index}] endpoints must be exact integers")
        if not 0 <= source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError(f"edges[{index}] has an out-of-range endpoint")
        if source == target:
            raise ValueError(f"edges[{index}] is a self-loop")
        if edge in unique_edges:
            raise ValueError(f"edges[{index}] duplicates an earlier edge")
        unique_edges.add(edge)

    adjacency_lists: list[list[int]] = [[] for _ in range(vertex_count)]
    indegrees = [0] * vertex_count
    for source, target in sorted(unique_edges):
        adjacency_lists[source].append(target)
        indegrees[target] += 1
    adjacency = tuple(tuple(neighbors) for neighbors in adjacency_lists)

    ready = [vertex for vertex, degree in enumerate(indegrees) if degree == 0]
    visited_count = 0
    while ready:
        source = ready.pop()
        visited_count += 1
        for target in adjacency[source]:
            indegrees[target] -= 1
            if indegrees[target] == 0:
                ready.append(target)
    if visited_count != vertex_count:
        raise ValueError("edges must describe an acyclic graph")

    matched_left_by_right = [-1] * vertex_count
    seen_at_search = [0] * vertex_count

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

    for left in range(vertex_count):
        augment(left, left + 1)

    predecessors = [-1] * vertex_count
    successors = [-1] * vertex_count
    for right, left in enumerate(matched_left_by_right):
        if left != -1:
            predecessors[right] = left
            successors[left] = right

    paths: list[tuple[int, ...]] = []
    visited = [False] * vertex_count
    for start in range(vertex_count):
        if predecessors[start] != -1:
            continue
        path: list[int] = []
        current = start
        while current != -1:
            if visited[current]:
                raise AssertionError("matching reconstruction revisited a vertex")
            visited[current] = True
            path.append(current)
            current = successors[current]
        paths.append(tuple(path))

    if not all(visited):
        raise AssertionError("matching reconstruction omitted a vertex")
    return tuple(paths)


```

## Example

```python
graph_edges = (
    (0, 2),
    (0, 3),
    (1, 3),
    (1, 4),
    (2, 5),
    (3, 5),
    (3, 6),
    (4, 6),
)
cover = minimum_vertex_disjoint_path_cover(7, graph_edges)
reordered = minimum_vertex_disjoint_path_cover(7, tuple(reversed(graph_edges)))

try:
    minimum_vertex_disjoint_path_cover(3, ((0, 1), (1, 2), (2, 0)))
except ValueError:
    cycle_rejected = True
else:
    cycle_rejected = False

assert (
    cover,
    reordered,
    minimum_vertex_disjoint_path_cover(0, ()),
    cycle_rejected,
) == (
    ((0, 2, 5), (1, 3, 6), (4,)),
    ((0, 2, 5), (1, 3, 6), (4,)),
    (),
    True,
)
```

## Trade-offs and Limitations

Validation and adjacency construction take `O(V + E log E)` time, including
canonical edge sorting. At most one depth-first augmenting search per left
vertex takes `O(V * E)` time, for `O(V + E log E + V * E)` total time.
Adjacency, degree, matching, and reconstruction state use `O(V + E)` memory.
The vertex bound limits augmenting-path recursion to at most 256 nested calls.

The returned cover has minimum path count and is invariant to edge tuple order.
It is only one deterministic optimum selected by the stated traversal: it is
not promised to be lexicographically smallest or canonical across different
matching algorithms. The function rejects duplicate edges, self-loops, and
cycles, including cycles in disconnected components. It does not return a
matching certificate, prove uniqueness, accept implicit vertices, attach edge
costs, or update a cover incrementally.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Deterministic Maximum-Cardinality Matching in a Bounded Bipartite Graph](find-a-deterministic-maximum-cardinality-matching-in-a-bounded-bipartite-graph.md)
- [Resolve Stable Ordering Constraints with Topological Sort](resolve-stable-ordering-constraints-with-topological-sort.md)
- [Compute the Transitive Reduction of a Bounded Directed Acyclic Graph](compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md)
<!-- catalog:related:end -->
