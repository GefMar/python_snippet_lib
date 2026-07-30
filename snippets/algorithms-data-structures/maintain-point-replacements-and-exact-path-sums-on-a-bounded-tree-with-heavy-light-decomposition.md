---
title: "Maintain Point Replacements and Exact Path Sums on a Bounded Tree with Heavy-Light Decomposition"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - answer-bounded-lowest-common-ancestor-queries-with-binary-lifting.md
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
---

# Maintain Point Replacements and Exact Path Sums on a Bounded Tree with Heavy-Light Decomposition

## Idea and Problem

Replace individual vertex values and answer inclusive sums along arbitrary paths in one static tree.

Heavy-light decomposition selects at most one largest child for every vertex.
Following those heavy edges partitions the tree into linear chains. Any tree
path crosses only logarithmically many chains, so mapping the chains into one
array turns a path sum into a small collection of Fenwick range sums.

The topology and decomposition stay fixed while point values change. Each
replacement updates one Fenwick position; each query climbs chain heads until
both endpoints share a chain.

## When to Use

Use this structure when a validated tree is static, vertex values are replaced
between queries, and many exact inclusive path sums must be answered. It is a
good fit when recomputing a path with breadth-first search for every query is
too expensive but a fully dynamic tree would be unnecessary.

Use root-prefix sums for an immutable tree, or direct path reconstruction for
small inputs. A generic Fenwick or segment tree handles linear ranges but does
not map tree paths by itself. Choose a link-cut tree or another dynamic-tree
structure when edges can be inserted or removed.

## Implementation

```python
_MIN_TREE_VALUE = -(1 << 63)
_MAX_TREE_VALUE = (1 << 63) - 1
_MAX_HLD_VERTICES = 10_000


class TreePathSums:
    """Own signed vertex values over one static heavy-light decomposition."""

    __slots__ = (
        "_depths",
        "_heads",
        "_parents",
        "_positions",
        "_tree",
        "_values",
    )

    def __init__(
        self,
        values: tuple[int, ...],
        edges: tuple[tuple[int, int], ...],
    ) -> None:
        if type(values) is not tuple:
            raise TypeError("values must be an exact tuple")
        if not 1 <= len(values) <= _MAX_HLD_VERTICES:
            raise ValueError("value count is outside 1..10000")
        for value_index, value in enumerate(values):
            self._validated_value(value, f"values[{value_index}]")

        if type(edges) is not tuple:
            raise TypeError("edges must be an exact tuple")
        vertex_count = len(values)
        if len(edges) != vertex_count - 1:
            raise ValueError("a tree must contain exactly V - 1 edges")

        adjacency: list[list[int]] = [[] for _ in range(vertex_count)]
        seen_edges: set[tuple[int, int]] = set()
        for edge_index, edge in enumerate(edges):
            if type(edge) is not tuple:
                raise TypeError(f"edges[{edge_index}] must be an exact tuple")
            if len(edge) != 2:
                raise ValueError(f"edges[{edge_index}] must contain two vertices")
            first, second = edge
            self._validated_input_vertex(
                first,
                vertex_count,
                f"edges[{edge_index}].first",
            )
            self._validated_input_vertex(
                second,
                vertex_count,
                f"edges[{edge_index}].second",
            )
            if first == second:
                raise ValueError(f"edges[{edge_index}] must not be a self-edge")
            normalized = (first, second) if first < second else (second, first)
            if normalized in seen_edges:
                raise ValueError(f"edges[{edge_index}] duplicates an edge")
            seen_edges.add(normalized)
            adjacency[first].append(second)
            adjacency[second].append(first)

        parents = [-2] * vertex_count
        depths = [0] * vertex_count
        parents[0] = -1
        order: list[int] = []
        pending = [0]
        while pending:
            vertex = pending.pop()
            order.append(vertex)
            for neighbor in adjacency[vertex]:
                if parents[neighbor] != -2:
                    continue
                parents[neighbor] = vertex
                depths[neighbor] = depths[vertex] + 1
                pending.append(neighbor)
        if len(order) != vertex_count:
            raise ValueError("edges must form one connected tree")

        subtree_sizes = [1] * vertex_count
        heavy_children = [-1] * vertex_count
        for vertex in reversed(order):
            best_child = -1
            for neighbor in adjacency[vertex]:
                if parents[neighbor] != vertex:
                    continue
                subtree_sizes[vertex] += subtree_sizes[neighbor]
                if best_child == -1 or subtree_sizes[neighbor] > subtree_sizes[
                    best_child
                ] or (
                    subtree_sizes[neighbor] == subtree_sizes[best_child]
                    and neighbor < best_child
                ):
                    best_child = neighbor
            heavy_children[vertex] = best_child

        heads = [0] * vertex_count
        positions = [0] * vertex_count
        next_position = 0
        chain_starts = [(0, 0)]
        while chain_starts:
            vertex, head = chain_starts.pop()
            while vertex != -1:
                heads[vertex] = head
                positions[vertex] = next_position
                next_position += 1
                light_children = [
                    neighbor
                    for neighbor in adjacency[vertex]
                    if parents[neighbor] == vertex
                    and neighbor != heavy_children[vertex]
                ]
                for child in reversed(light_children):
                    chain_starts.append((child, child))
                vertex = heavy_children[vertex]

        linear_values = [0] * vertex_count
        for vertex, value in enumerate(values):
            linear_values[positions[vertex]] = value
        tree = [0, *linear_values]
        for tree_index in range(1, len(tree)):
            parent_index = tree_index + (tree_index & -tree_index)
            if parent_index < len(tree):
                tree[parent_index] += tree[tree_index]

        self._parents = parents
        self._depths = depths
        self._heads = heads
        self._positions = positions
        self._tree = tree
        self._values = list(values)

    @staticmethod
    def _validated_input_vertex(
        vertex: object,
        vertex_count: int,
        label: str,
    ) -> int:
        if type(vertex) is not int:
            raise TypeError(f"{label} must be an exact integer")
        if not 0 <= vertex < vertex_count:
            raise ValueError(f"{label} is outside the tree")
        return vertex

    @staticmethod
    def _validated_value(value: object, label: str) -> int:
        if type(value) is not int:
            raise TypeError(f"{label} must be an exact integer")
        if not _MIN_TREE_VALUE <= value <= _MAX_TREE_VALUE:
            raise ValueError(f"{label} is outside signed 64-bit range")
        return value

    def _validated_vertex(self, vertex: object, label: str) -> int:
        return self._validated_input_vertex(
            vertex,
            len(self._values),
            label,
        )

    def _prefix_sum(self, stop: int) -> int:
        total = 0
        tree_index = stop
        while tree_index:
            total += self._tree[tree_index]
            tree_index -= tree_index & -tree_index
        return total

    def _range_sum(self, start: int, stop: int) -> int:
        return self._prefix_sum(stop) - self._prefix_sum(start)

    def replace(self, vertex: int, value: int) -> None:
        """Replace one signed-64-bit vertex value."""
        vertex = self._validated_vertex(vertex, "vertex")
        value = self._validated_value(value, "value")
        delta = value - self._values[vertex]
        self._values[vertex] = value
        tree_index = self._positions[vertex] + 1
        while tree_index < len(self._tree):
            self._tree[tree_index] += delta
            tree_index += tree_index & -tree_index

    def path_sum(self, first: int, second: int) -> int:
        """Return the exact sum on the inclusive path between two vertices."""
        first = self._validated_vertex(first, "first")
        second = self._validated_vertex(second, "second")
        total = 0

        while self._heads[first] != self._heads[second]:
            first_head = self._heads[first]
            second_head = self._heads[second]
            if self._depths[first_head] < self._depths[second_head]:
                first, second = second, first
                first_head = self._heads[first]
            total += self._range_sum(
                self._positions[first_head],
                self._positions[first] + 1,
            )
            first = self._parents[first_head]

        first_position = self._positions[first]
        second_position = self._positions[second]
        if first_position > second_position:
            first_position, second_position = second_position, first_position
        return total + self._range_sum(first_position, second_position + 1)
```

## Example

```python
def tree_from_prufer(code: tuple[int, ...]) -> tuple[tuple[int, int], ...]:
    from heapq import heapify, heappop, heappush

    vertex_count = len(code) + 2
    degrees = [1] * vertex_count
    for vertex in code:
        degrees[vertex] += 1
    leaves = [vertex for vertex, degree in enumerate(degrees) if degree == 1]
    heapify(leaves)
    edges: list[tuple[int, int]] = []
    for neighbor in code:
        leaf = heappop(leaves)
        edges.append((leaf, neighbor))
        degrees[neighbor] -= 1
        if degrees[neighbor] == 1:
            heappush(leaves, neighbor)
    edges.append((heappop(leaves), heappop(leaves)))
    return tuple(edges)


def naive_path_sum(
    values: list[int],
    edges: tuple[tuple[int, int], ...],
    first: int,
    second: int,
) -> int:
    from collections import deque

    adjacency = [[] for _ in values]
    for left, right in edges:
        adjacency[left].append(right)
        adjacency[right].append(left)
    parents = [-1] * len(values)
    parents[first] = first
    pending: deque[int] = deque([first])
    while pending:
        vertex = pending.popleft()
        if vertex == second:
            break
        for neighbor in adjacency[vertex]:
            if parents[neighbor] == -1:
                parents[neighbor] = vertex
                pending.append(neighbor)
    total = 0
    vertex = second
    while vertex != first:
        total += values[vertex]
        vertex = parents[vertex]
    return total + values[first]


def exercise_small_trees() -> int:
    from itertools import product

    single = TreePathSums((7,), ())
    assert single.path_sum(0, 0) == 7
    checked = 1

    for vertex_count in range(2, 7):
        values = [vertex * 3 - vertex_count for vertex in range(vertex_count)]
        for code in product(range(vertex_count), repeat=vertex_count - 2):
            edges = tree_from_prufer(code)
            path_sums = TreePathSums(tuple(values), edges)
            for first in range(vertex_count):
                for second in range(vertex_count):
                    assert path_sums.path_sum(first, second) == naive_path_sum(
                        values,
                        edges,
                        first,
                        second,
                    )
                    checked += 1
    return checked


checked_paths = exercise_small_trees()

scenario_edges = ((0, 1), (1, 2), (1, 3), (3, 4), (3, 5), (5, 6))
scenario_values = [5, -2, 7, 4, 1, -3, 9]
scenario = TreePathSums(tuple(scenario_values), scenario_edges)
assert scenario.path_sum(2, 6) == naive_path_sum(
    scenario_values,
    scenario_edges,
    2,
    6,
)
scenario.replace(3, _MIN_TREE_VALUE)
scenario_values[3] = _MIN_TREE_VALUE
scenario.replace(6, _MAX_TREE_VALUE)
scenario_values[6] = _MAX_TREE_VALUE

boundary_values = (_MAX_TREE_VALUE,) * _MAX_HLD_VERTICES
boundary_edges = tuple(
    (vertex - 1, vertex) for vertex in range(1, _MAX_HLD_VERTICES)
)
boundary = TreePathSums(boundary_values, boundary_edges)

disconnected_rejected = False
try:
    TreePathSums((0, 0, 0, 0), ((0, 1), (1, 2), (2, 0)))
except ValueError:
    disconnected_rejected = True

assert (
    checked_paths == 50_069
    and scenario.path_sum(2, 6)
    == naive_path_sum(scenario_values, scenario_edges, 2, 6)
    and scenario.path_sum(6, 2) == scenario.path_sum(2, 6)
    and boundary.path_sum(0, _MAX_HLD_VERTICES - 1)
    == _MAX_HLD_VERTICES * _MAX_TREE_VALUE
    and disconnected_rejected
)
```

## Trade-offs and Limitations

Validation, rooting, subtree sizing, decomposition, and linear Fenwick
construction take `O(V + E)` time and memory. A replacement touches
`O(log V)` Fenwick cells. A path crosses `O(log V)` heavy chains and each
range sum costs `O(log V)`, giving `O(log**2 V)` time per query.

Stored values and replacements must be exact signed 64-bit integers, but sums
use unbounded Python integers and may exceed that range. Vertex zero is the
internal root. Equal-size heavy-child choices prefer the lower vertex index;
this only makes the private layout reproducible and does not alter results.

This class is mutable and not thread-safe. It supports vertex replacement and
sum only: no edge weights, increments, path updates, subtree API, min/max
aggregate, topology mutation, persistence, or concurrent mutation is provided.

## Related Snippets

<!-- catalog:related:start -->
- [Answer Bounded Lowest-Common-Ancestor Queries with Binary Lifting](answer-bounded-lowest-common-ancestor-queries-with-binary-lifting.md)
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
<!-- catalog:related:end -->
