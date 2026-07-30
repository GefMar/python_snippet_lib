---
title: "Enumerate Bounded Maximal Cliques Deterministically with Bron-Kerbosch"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md
  - find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md
  - two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md
---

# Enumerate Bounded Maximal Cliques Deterministically with Bron-Kerbosch

## Idea and Problem

Enumerate every inclusion-maximal clique in a small simple undirected graph with stable ordering.

A clique is a set of pairwise adjacent vertices. It is maximal when no other
vertex can be added while preserving that property; it need not have maximum
cardinality. The Bron-Kerbosch search maintains a growing clique, candidates
that may extend it, and excluded vertices that prevent duplicate output.

Choosing a pivot that has the most neighbors among the current candidates
reduces redundant branches. Vertex-number tie breaking and ascending branch
order make the traversal reproducible, while a final lexicographic sort makes
the public result independent of internal recursion order.

## When to Use

Use this algorithm when a graph has at most 21 vertices and every maximal
fully connected group is itself the required output. Examples include bounded
compatibility analysis, exhaustive fixture generation, and structural audits
where singleton cliques for isolated vertices are meaningful.

Use a graph library or a problem-specific optimizer for larger inputs. If only
one largest clique is needed, maximum-clique search is a different problem and
should not pay the output cost of enumerating every maximal clique.

## Implementation

```python
_MAX_VERTICES = 21
_MAX_MAXIMAL_CLIQUES = 2_187


def _validate_simple_undirected_graph(
    adjacency: tuple[frozenset[int], ...],
) -> tuple[frozenset[int], ...]:
    if type(adjacency) is not tuple:
        raise TypeError("adjacency must be an exact tuple")
    if len(adjacency) > _MAX_VERTICES:
        raise ValueError("the graph exceeds the supported vertex limit")

    vertex_count = len(adjacency)
    for vertex, neighbors in enumerate(adjacency):
        if type(neighbors) is not frozenset:
            raise TypeError("every adjacency row must be an exact frozenset")
        for neighbor in neighbors:
            if type(neighbor) is not int:
                raise TypeError("every neighbor must be an exact integer")
            if not 0 <= neighbor < vertex_count:
                raise ValueError("an adjacency row names an unknown vertex")
            if neighbor == vertex:
                raise ValueError("self-loops are not supported")

    for vertex, neighbors in enumerate(adjacency):
        for neighbor in neighbors:
            if vertex not in adjacency[neighbor]:
                raise ValueError("adjacency rows must be symmetric")
    return adjacency


def enumerate_maximal_cliques(
    adjacency: tuple[frozenset[int], ...],
) -> tuple[tuple[int, ...], ...]:
    graph = _validate_simple_undirected_graph(adjacency)
    if not graph:
        return ()

    found: list[tuple[int, ...]] = []

    def visit(
        chosen: set[int],
        candidates: set[int],
        excluded: set[int],
    ) -> None:
        if not candidates and not excluded:
            found.append(tuple(sorted(chosen)))
            if len(found) > _MAX_MAXIMAL_CLIQUES:
                raise RuntimeError("the proven output bound was exceeded")
            return

        pivot_pool = candidates | excluded
        pivot = min(
            pivot_pool,
            key=lambda vertex: (
                -len(candidates & graph[vertex]),
                vertex,
            ),
        )
        for vertex in sorted(candidates - graph[pivot]):
            visit(
                chosen | {vertex},
                candidates & graph[vertex],
                excluded & graph[vertex],
            )
            candidates.remove(vertex)
            excluded.add(vertex)

    visit(set(), set(range(len(graph))), set())
    return tuple(sorted(found))
```

## Example

```python
graph = (
    frozenset({1, 2}),
    frozenset({0, 2}),
    frozenset({0, 1, 3}),
    frozenset({2, 4}),
    frozenset({3}),
    frozenset(),
)

assert enumerate_maximal_cliques(graph) == (
    (0, 1, 2),
    (2, 3),
    (3, 4),
    (5,),
)
assert enumerate_maximal_cliques(()) == ()
assert enumerate_maximal_cliques(
    (frozenset(), frozenset(), frozenset())
) == ((0,), (1,), (2,))
```

## Trade-offs and Limitations

Validation takes `O(n + m)` membership work for `n` vertices and `m` directed
adjacency entries. Enumeration has exponential worst-case time and output. A
21-vertex graph can have exactly 2,187 maximal cliques, attained by a complete
seven-partite graph with three vertices in each part. The implementation keeps
that theorem-derived value as a defensive output invariant.

The recursive depth is at most 21, and each branch allocates bounded Python
sets. Final sorting provides a canonical tuple but adds work proportional to
the number and size of returned cliques. Integer vertex numbering is part of
the deterministic contract; the function does not preserve external labels.

This implementation accepts only a fully materialized simple undirected graph.
It does not repair asymmetric input, admit self-loops or parallel edges,
stream partial results, compute a maximum-weight clique, or make the search
practical for unbounded graphs.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Core Numbers and a Deterministic Degeneracy Order for a Bounded Undirected Graph](compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md)
- [Find Articulation Points and Bridges in a Bounded Undirected Graph](find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md)
- [Two-Color a Bounded Undirected Graph or Return an Odd-Cycle Witness](two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md)
<!-- catalog:related:end -->
