---
title: "Count Edge-Distinct Spanning Trees of a Bounded Undirected Multigraph with the Matrix-Tree Theorem"
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
  - compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
  - encode-and-decode-bounded-labelled-trees-with-prufer-sequences.md
---

# Count Edge-Distinct Spanning Trees of a Bounded Undirected Multigraph with the Matrix-Tree Theorem

## Idea and Problem

Count how many edge-distinct spanning trees a bounded undirected multigraph has without enumerating the trees.

The graph Laplacian records every non-loop edge occurrence: each endpoint
diagonal increases by one and the two corresponding off-diagonal entries
decrease by one. Parallel occurrences therefore contribute separately. The
Matrix-Tree Theorem says that deleting any one row and matching column leaves
a cofactor whose determinant is exactly the spanning-tree count.

Bareiss elimination evaluates that integer determinant without introducing
fractions at every step. Self-loops never join different vertices, so they are
validated but deliberately omitted from the Laplacian.

## When to Use

Use this function for a small, fully materialized undirected multigraph when
parallel edge occurrences represent distinct choices and only the exact count
of spanning trees is needed. It is useful for checking network redundancy,
validating graph generators, and replacing exponential tree enumeration with
dense exact algebra.

Use Kruskal's algorithm when edge weights and one minimum spanning forest are
the objective. Use direct subset enumeration when the individual trees are
needed, or a sparse or modular determinant implementation when a larger graph
makes dense Python big-integer elimination unsuitable.

## Implementation

```python
_MAX_SPANNING_VERTEX_COUNT = 32
_MAX_SPANNING_EDGE_COUNT = 4_096

_SpanningEdge = tuple[int, int]


def _bareiss_spanning_determinant(work: list[list[int]]) -> int:
    """Consume one integer matrix and return its exact determinant."""
    order = len(work)
    if order == 0:
        return 1

    sign = 1
    previous_pivot = 1
    for pivot_index in range(order - 1):
        if work[pivot_index][pivot_index] == 0:
            swap_index = next(
                (
                    row_index
                    for row_index in range(pivot_index + 1, order)
                    if work[row_index][pivot_index] != 0
                ),
                None,
            )
            if swap_index is None:
                return 0
            work[pivot_index], work[swap_index] = (
                work[swap_index],
                work[pivot_index],
            )
            sign = -sign

        pivot = work[pivot_index][pivot_index]
        for row_index in range(pivot_index + 1, order):
            eliminated = work[row_index][pivot_index]
            for column_index in range(pivot_index + 1, order):
                numerator = (
                    work[row_index][column_index] * pivot
                    - eliminated * work[pivot_index][column_index]
                )
                quotient, remainder = divmod(numerator, previous_pivot)
                if remainder:
                    raise AssertionError("Bareiss division must remain exact")
                work[row_index][column_index] = quotient
            work[row_index][pivot_index] = 0
        previous_pivot = pivot

    return sign * work[-1][-1]


def count_edge_distinct_spanning_trees(
    vertex_count: int,
    edges: tuple[_SpanningEdge, ...],
) -> int:
    """Return the exact number of spanning trees distinguished by edge index."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_SPANNING_VERTEX_COUNT:
        raise ValueError("vertex_count is outside 1..32")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_SPANNING_EDGE_COUNT:
        raise ValueError("edge count exceeds 4096")

    laplacian = [[0] * vertex_count for _ in range(vertex_count)]
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two endpoints")
        first, second = edge
        if type(first) is not int or type(second) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= first < vertex_count or not 0 <= second < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the graph")
        if first == second:
            continue

        laplacian[first][first] += 1
        laplacian[second][second] += 1
        laplacian[first][second] -= 1
        laplacian[second][first] -= 1

    cofactor_order = vertex_count - 1
    cofactor = [
        row[:cofactor_order] for row in laplacian[:cofactor_order]
    ]
    return _bareiss_spanning_determinant(cofactor)
```

## Example

```python
def find_subset_representative(parents: list[int], vertex: int) -> int:
    while parents[vertex] != vertex:
        vertex = parents[vertex]
    return vertex


def count_spanning_trees_by_edge_subsets(
    vertex_count: int,
    edges: tuple[_SpanningEdge, ...],
) -> int:
    from itertools import combinations

    if vertex_count == 1:
        return 1

    tree_count = 0
    for selected_indexes in combinations(range(len(edges)), vertex_count - 1):
        parents = list(range(vertex_count))
        sizes = [1] * vertex_count

        valid = True
        for edge_index in selected_indexes:
            first, second = edges[edge_index]
            first_root = find_subset_representative(parents, first)
            second_root = find_subset_representative(parents, second)
            if first_root == second_root:
                valid = False
                break
            if sizes[first_root] < sizes[second_root]:
                first_root, second_root = second_root, first_root
            parents[second_root] = first_root
            sizes[first_root] += sizes[second_root]

        first_root = find_subset_representative(parents, 0)
        if valid and all(
            find_subset_representative(parents, vertex) == first_root
            for vertex in range(vertex_count)
        ):
            tree_count += 1
    return tree_count


def exercise_small_multigraphs() -> int:
    from itertools import product

    checked = 0
    for vertex_count in range(1, 4):
        edge_kinds = tuple(product(range(vertex_count), repeat=2))
        for edge_count in range(5):
            for edges in product(edge_kinds, repeat=edge_count):
                assert count_edge_distinct_spanning_trees(
                    vertex_count,
                    edges,
                ) == count_spanning_trees_by_edge_subsets(vertex_count, edges)
                checked += 1
    return checked


parallel_edges = tuple(
    (0, 1) if edge_index % 2 == 0 else (1, 0)
    for edge_index in range(23)
)
cycle = ((0, 1), (1, 2), (2, 3), (3, 0))
complete_four = tuple(
    (first, second)
    for first in range(4)
    for second in range(first + 1, 4)
)
boundary_path = tuple(
    (vertex, vertex + 1)
    for vertex in range(_MAX_SPANNING_VERTEX_COUNT - 1)
)
maximum_loops = ((0, 0),) * _MAX_SPANNING_EDGE_COUNT

rejected = 0
for vertex_count, edges in (
    (0, ()),
    (True, ()),
    (2, [(0, 1)]),
    (2, ((0, 2),)),
    (2, ((0, True),)),
    (2, ((0,),)),
    (1, ((0, 0),) * (_MAX_SPANNING_EDGE_COUNT + 1)),
):
    try:
        count_edge_distinct_spanning_trees(vertex_count, edges)
    except (TypeError, ValueError):
        rejected += 1

assert (
    exercise_small_multigraphs(),
    count_edge_distinct_spanning_trees(1, maximum_loops),
    count_edge_distinct_spanning_trees(2, parallel_edges),
    count_edge_distinct_spanning_trees(4, cycle),
    count_edge_distinct_spanning_trees(4, complete_four),
    count_edge_distinct_spanning_trees(3, ((0, 0), (0, 1), (1, 1))),
    count_edge_distinct_spanning_trees(3, ((0, 1),)),
    count_edge_distinct_spanning_trees(
        _MAX_SPANNING_VERTEX_COUNT,
        boundary_path,
    ),
    rejected,
) == (7_727, 1, 23, 4, 16, 0, 0, 1, 7)
```

## Trade-offs and Limitations

For `V` vertices and `E` declared edge occurrences, Laplacian construction
and Bareiss elimination perform `O(V**3 + E)` exact-integer operations. The
input, dense Laplacian, and cofactor occupy `O(V**2 + E)` references. These
bounds do not make arithmetic constant-cost: determinant entries and the exact
tree count can grow far beyond fixed-width integers.

Endpoints are zero-based vertex indexes. Edge orientation and declaration
order do not affect the answer, but every parallel occurrence remains a
separate selectable edge. Self-loops are accepted and ignored because they
cannot belong to a spanning tree. A single vertex has one empty spanning tree,
and a disconnected graph has determinant and count zero.

The function returns only a count. It does not enumerate trees, return a
connectivity certificate, attach weights, find a minimum spanning tree,
distinguish self-loop identities in the result, or use sparse or modular
linear algebra.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
- [Encode and Decode Bounded Labelled Trees with Prüfer Sequences](encode-and-decode-bounded-labelled-trees-with-prufer-sequences.md)
<!-- catalog:related:end -->
