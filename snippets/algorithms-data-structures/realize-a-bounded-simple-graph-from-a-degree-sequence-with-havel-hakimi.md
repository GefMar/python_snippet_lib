---
title: "Realize a Bounded Simple Graph from a Degree Sequence with Havel-Hakimi"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-and-decode-bounded-labelled-trees-with-prufer-sequences.md
  - two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md
  - compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md
---

# Realize a Bounded Simple Graph from a Degree Sequence with Havel-Hakimi

## Idea and Problem

Decide whether a bounded sequence is the degree sequence of a labelled simple undirected graph and construct one deterministic realization when it is.

The Havel-Hakimi reduction repeatedly removes the vertex with the greatest
residual degree and joins it to that many other vertices with the greatest
residual degrees. A valid sequence is graphical exactly when these reductions
reach all zeros without demanding a missing or zero-degree neighbor. Original vertex
indices break every residual-degree tie, and normalized sorted edges make the
chosen realization reproducible.

## When to Use

Use this algorithm to build bounded graph fixtures, validate degree summaries,
or turn a known degree specification into one concrete adjacency structure.
Tuple position is the vertex label, so repeated degree values remain distinct.

Use a different generator when a graph must be connected, sampled uniformly,
optimized for another property, or allowed to contain loops, parallel edges,
directions, or weights. The deterministic tie rule chooses one realization;
it does not promise the globally lexicographically smallest edge set.

## Implementation

```python
_MAX_VERTICES = 128


def realize_simple_graph(
    degrees: tuple[int, ...],
) -> tuple[tuple[int, int], ...] | None:
    """Return one deterministic realization, or None for a nongraphical sequence."""
    if type(degrees) is not tuple:
        raise TypeError("degrees must be an exact tuple")
    if len(degrees) > _MAX_VERTICES:
        raise ValueError("degree sequence exceeds the supported vertex count")

    vertex_count = len(degrees)
    for index, degree in enumerate(degrees):
        if type(degree) is not int:
            raise TypeError(f"degrees[{index}] must be an exact integer")
        if not 0 <= degree < vertex_count:
            raise ValueError(f"degrees[{index}] is outside 0..n-1")

    if sum(degrees) % 2 != 0:
        return None

    residual = [(degree, vertex) for vertex, degree in enumerate(degrees)]
    edges: list[tuple[int, int]] = []

    while residual:
        residual.sort(key=lambda item: (-item[0], item[1]))
        degree, vertex = residual[0]
        if degree == 0:
            break

        remaining = residual[1:]
        if degree > len(remaining):
            return None
        if any(other_degree == 0 for other_degree, _ in remaining[:degree]):
            return None

        reduced: list[tuple[int, int]] = []
        for position, (other_degree, other_vertex) in enumerate(remaining):
            if position < degree:
                other_degree -= 1
                edge = (
                    (vertex, other_vertex)
                    if vertex < other_vertex
                    else (other_vertex, vertex)
                )
                edges.append(edge)
            reduced.append((other_degree, other_vertex))
        residual = reduced

    return tuple(sorted(edges))
```

## Example

```python
def feasible_degree_sequences(vertex_count: int) -> set[tuple[int, ...]]:
    from itertools import combinations

    possible_edges = tuple(combinations(range(vertex_count), 2))
    feasible: set[tuple[int, ...]] = set()
    for mask in range(1 << len(possible_edges)):
        observed = [0] * vertex_count
        for bit, (first, second) in enumerate(possible_edges):
            if mask & (1 << bit):
                observed[first] += 1
                observed[second] += 1
        feasible.add(tuple(observed))
    return feasible


def verify_all_sequences_through_six_vertices() -> int:
    from itertools import product

    checked = 0
    for vertex_count in range(7):
        feasible = feasible_degree_sequences(vertex_count)
        candidates = product(range(vertex_count), repeat=vertex_count)
        for candidate in candidates:
            realization = realize_simple_graph(candidate)
            assert (realization is not None) == (candidate in feasible)
            if realization is not None:
                observed = [0] * vertex_count
                assert realization == tuple(sorted(set(realization)))
                for first, second in realization:
                    assert 0 <= first < second < vertex_count
                    observed[first] += 1
                    observed[second] += 1
                assert tuple(observed) == candidate
            checked += 1
    return checked


complete_degrees = (127,) * 128
complete_edges = realize_simple_graph(complete_degrees)

assert (
    verify_all_sequences_through_six_vertices() == 50_070
    and realize_simple_graph(()) == ()
    and realize_simple_graph((3, 3, 1, 1)) is None
    and complete_edges is not None
    and len(complete_edges) == 8_128
)
```

## Trade-offs and Limitations

For `V` vertices, this direct bounded implementation takes
`O(V^2 log V + E log E)` time because it re-sorts residual vertices after each
reduction and sorts the final `E` edges. It uses `O(V + E)` auxiliary memory.
The small fixed vertex bound favors an auditable implementation over a more
complex bucketed priority structure.

Malformed inputs raise, while a well-formed but nongraphical sequence returns
`None`. The result is simple and degree-correct but has no connectedness,
randomness, planarity, coloring, or optimization guarantee.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode Bounded Labelled Trees with Prüfer Sequences](encode-and-decode-bounded-labelled-trees-with-prufer-sequences.md)
- [Two-Color a Bounded Undirected Graph or Return an Odd-Cycle Witness](two-color-a-bounded-undirected-graph-or-return-an-odd-cycle-witness.md)
- [Compute Core Numbers and a Deterministic Degeneracy Order for a Bounded Undirected Graph](compute-core-numbers-and-a-deterministic-degeneracy-order-for-a-bounded-undirected-graph.md)
<!-- catalog:related:end -->
