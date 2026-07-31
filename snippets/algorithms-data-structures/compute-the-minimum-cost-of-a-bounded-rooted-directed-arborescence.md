---
title: "Compute the Minimum Cost of a Bounded Rooted Directed Arborescence"
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
  - build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md
  - compute-the-exact-minimum-directed-cycle-mean-with-karps-algorithm.md
  - send-a-required-flow-at-minimum-cost-with-a-residual-certificate.md
---

# Compute the Minimum Cost of a Bounded Rooted Directed Arborescence

## Idea and Problem

Compute the exact minimum cost of a directed spanning tree whose unique directed path from one root reaches every vertex.

A cheapest incoming edge can be selected independently for every non-root
vertex. If those edges contain no directed cycle, they already form a minimum
rooted arborescence. A selected cycle cannot remain in a tree, so
Chu-Liu/Edmonds contracts the whole cycle into one supervertex and solves a
smaller graph.

The selected incoming costs are added before contraction. An edge entering a
new component is then reduced by the selected incoming cost of its original
target. That adjustment measures the extra cost of replacing the target's
selected edge when the contracted cycle is eventually broken.

## When to Use

Use this function for a fully materialized bounded directed graph when only the
minimum cost of reaching every vertex through one rooted branching is needed.
Negative costs and parallel directed edges are supported, and `None` cleanly
distinguishes a graph with no root-spanning arborescence from a valid negative,
zero, or positive result.

Use an undirected minimum-spanning-tree algorithm when edge direction does not
matter. Choose a fuller Chu-Liu/Edmonds implementation or a graph library when
the selected edge identities, canonical ties, large graphs, dynamic updates,
degree constraints, or enumeration of alternative optima are required.

## Implementation

```python
_MAX_ARBORESCENCE_VERTICES = 64
_MAX_ARBORESCENCE_EDGES = 4_096
_MIN_SIGNED_32 = -(1 << 31)
_MAX_SIGNED_32 = (1 << 31) - 1


def minimum_rooted_arborescence_cost(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
    *,
    root: int,
) -> int | None:
    """Return the exact minimum rooted arborescence cost, or None."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_ARBORESCENCE_VERTICES:
        raise ValueError("vertex_count is outside 1..64")
    if type(root) is not int:
        raise TypeError("root must be an exact integer")
    if not 0 <= root < vertex_count:
        raise ValueError("root is outside the graph")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_ARBORESCENCE_EDGES:
        raise ValueError("edge count exceeds 4,096")

    current_edges: list[tuple[int, int, int]] = []
    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 3:
            raise ValueError(f"edges[{edge_index}] must contain three integers")
        source, target, weight = edge
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"edges[{edge_index}] endpoints must be exact integers")
        if not 0 <= source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError(f"edges[{edge_index}] endpoint is outside the graph")
        if type(weight) is not int:
            raise TypeError(f"edges[{edge_index}].weight must be an exact integer")
        if not _MIN_SIGNED_32 <= weight <= _MAX_SIGNED_32:
            raise ValueError(f"edges[{edge_index}].weight is outside signed 32-bit")
        if source != target:
            current_edges.append((source, target, weight))

    current_vertex_count = vertex_count
    current_root = root
    total_cost = 0

    while True:
        incoming_cost: list[int | None] = [None] * current_vertex_count
        predecessor = [-1] * current_vertex_count
        for source, target, weight in current_edges:
            if target == current_root:
                continue
            known_cost = incoming_cost[target]
            known_source = predecessor[target]
            if known_cost is None or (weight, source) < (known_cost, known_source):
                incoming_cost[target] = weight
                predecessor[target] = source

        incoming_cost[current_root] = 0
        predecessor[current_root] = current_root
        if any(cost is None for cost in incoming_cost):
            return None
        total_cost += sum(cost for cost in incoming_cost if cost is not None)

        component = [-1] * current_vertex_count
        seen_from = [-1] * current_vertex_count
        component_count = 0
        for start in range(current_vertex_count):
            vertex = start
            while seen_from[vertex] != start and component[vertex] == -1 and vertex != current_root:
                seen_from[vertex] = start
                vertex = predecessor[vertex]

            if vertex == current_root or component[vertex] != -1:
                continue
            component[vertex] = component_count
            cursor = predecessor[vertex]
            while cursor != vertex:
                component[cursor] = component_count
                cursor = predecessor[cursor]
            component_count += 1

        if component_count == 0:
            return total_cost

        for vertex in range(current_vertex_count):
            if component[vertex] == -1:
                component[vertex] = component_count
                component_count += 1

        contracted_edges: list[tuple[int, int, int]] = []
        for source, target, weight in current_edges:
            contracted_source = component[source]
            contracted_target = component[target]
            if contracted_source == contracted_target:
                continue
            selected_cost = incoming_cost[target]
            if selected_cost is None:
                raise AssertionError("every contracted target must have an incoming cost")
            contracted_edges.append(
                (
                    contracted_source,
                    contracted_target,
                    weight - selected_cost,
                )
            )

        current_root = component[current_root]
        current_vertex_count = component_count
        current_edges = contracted_edges
```

## Example

```python
from itertools import product
from random import Random


def brute_arborescence_cost(
    vertex_count: int,
    edges: tuple[tuple[int, int, int], ...],
    root: int,
) -> int | None:
    candidates: list[tuple[tuple[int, int], ...]] = []
    for target in range(vertex_count):
        if target == root:
            continue
        incoming = tuple(
            (source, weight)
            for source, edge_target, weight in edges
            if edge_target == target and source != target
        )
        if not incoming:
            return None
        candidates.append(incoming)

    best: int | None = None
    non_root_vertices = tuple(vertex for vertex in range(vertex_count) if vertex != root)
    for selected in product(*candidates):
        parent = {
            vertex: source for vertex, (source, _) in zip(non_root_vertices, selected, strict=True)
        }
        valid = True
        for start in non_root_vertices:
            visited: set[int] = set()
            vertex = start
            while vertex != root and vertex not in visited:
                visited.add(vertex)
                vertex = parent[vertex]
            if vertex != root:
                valid = False
                break
        if valid:
            cost = sum(weight for _, weight in selected)
            if best is None or cost < best:
                best = cost
    return best


# Exhaust every directed edge subset on three vertices under fixed signed costs.
complete_three = tuple(
    (source, target, ((source * 3 + target) % 5) - 2)
    for source in range(3)
    for target in range(3)
    if source != target
)
exhaustive_checks = 0
for mask in range(1 << len(complete_three)):
    subset = tuple(
        edge for edge_index, edge in enumerate(complete_three) if mask & (1 << edge_index)
    )
    for root in range(3):
        assert minimum_rooted_arborescence_cost(3, subset, root=root) == (
            brute_arborescence_cost(3, subset, root)
        )
        exhaustive_checks += 1

generator = Random(0xC1E_185)
random_checks = 400
for _ in range(random_checks):
    vertex_count = generator.randrange(1, 6)
    root = generator.randrange(vertex_count)
    generated: list[tuple[int, int, int]] = []
    for source in range(vertex_count):
        for target in range(vertex_count):
            if generator.randrange(4) == 0:
                generated.append((source, target, generator.randrange(-7, 8)))
                if source != target and generator.randrange(8) == 0:
                    generated.append((source, target, generator.randrange(-7, 8)))
    random_edges = tuple(generated)
    expected = brute_arborescence_cost(vertex_count, random_edges, root)
    actual = minimum_rooted_arborescence_cost(vertex_count, random_edges, root=root)
    assert actual == expected

    if actual is not None:
        offset = 11
        shifted = tuple(
            (source, target, weight + offset) for source, target, weight in random_edges
        )
        assert minimum_rooted_arborescence_cost(
            vertex_count,
            shifted,
            root=root,
        ) == actual + offset * (vertex_count - 1)

# Two selected cycles require contraction before the root can enter them.
nested_cycles = (
    (1, 2, -5),
    (2, 1, -4),
    (3, 4, -7),
    (4, 3, -6),
    (2, 3, 2),
    (4, 1, 3),
    (0, 1, 10),
    (0, 3, 9),
)
assert minimum_rooted_arborescence_cost(5, nested_cycles, root=0) == (
    brute_arborescence_cost(5, nested_cycles, 0)
)

maximum_edges = tuple(
    (
        source,
        target,
        _MIN_SIGNED_32 + target if source == 0 else _MAX_SIGNED_32,
    )
    for source in range(_MAX_ARBORESCENCE_VERTICES)
    for target in range(_MAX_ARBORESCENCE_VERTICES)
)
maximum_expected = (_MAX_ARBORESCENCE_VERTICES - 1) * _MIN_SIGNED_32 + sum(
    range(1, _MAX_ARBORESCENCE_VERTICES)
)
maximum_actual = minimum_rooted_arborescence_cost(
    _MAX_ARBORESCENCE_VERTICES,
    maximum_edges,
    root=0,
)
invalid_calls = (
    (lambda: minimum_rooted_arborescence_cost(True, (), root=0), TypeError),
    (lambda: minimum_rooted_arborescence_cost(0, (), root=0), ValueError),
    (lambda: minimum_rooted_arborescence_cost(65, (), root=0), ValueError),
    (lambda: minimum_rooted_arborescence_cost(1, (), root=True), TypeError),
    (lambda: minimum_rooted_arborescence_cost(1, (), root=1), ValueError),
    (lambda: minimum_rooted_arborescence_cost(1, [], root=0), TypeError),
    (
        lambda: minimum_rooted_arborescence_cost(
            1,
            ((0, 0, 0),) * (_MAX_ARBORESCENCE_EDGES + 1),
            root=0,
        ),
        ValueError,
    ),
    (lambda: minimum_rooted_arborescence_cost(2, ([0, 1, 2],), root=0), TypeError),
    (lambda: minimum_rooted_arborescence_cost(2, ((0, 1),), root=0), ValueError),
    (lambda: minimum_rooted_arborescence_cost(2, ((True, 1, 0),), root=0), TypeError),
    (lambda: minimum_rooted_arborescence_cost(2, ((0, 2, 0),), root=0), ValueError),
    (lambda: minimum_rooted_arborescence_cost(2, ((0, 1, True),), root=0), TypeError),
    (
        lambda: minimum_rooted_arborescence_cost(
            2,
            ((0, 1, _MAX_SIGNED_32 + 1),),
            root=0,
        ),
        ValueError,
    ),
)
rejected = 0
for invalid_call, expected_error in invalid_calls:
    try:
        invalid_call()
    except expected_error:
        rejected += 1
    else:
        raise AssertionError("invalid input must be rejected")

assert (
    exhaustive_checks == 192
    and random_checks == 400
    and maximum_actual == maximum_expected
    and minimum_rooted_arborescence_cost(1, ((0, 0, _MIN_SIGNED_32),), root=0) == 0
    and minimum_rooted_arborescence_cost(
        3,
        ((0, 1, 5), (0, 1, -2), (1, 2, 1), (0, 2, 10)),
        root=0,
    )
    == -1
    and minimum_rooted_arborescence_cost(2, ((0, 1, _MAX_SIGNED_32),), root=0) == _MAX_SIGNED_32
    and minimum_rooted_arborescence_cost(2, ((1, 0, -1),), root=0) is None
    and rejected == len(invalid_calls)
)
```

## Trade-offs and Limitations

Each contraction round scans all current edges and vertices, and every round
removes at least one vertex. The bounded implementation therefore uses
`O(V * (V + E))` time and `O(V + E)` working memory. Whenever a spanning
arborescence exists, `E >= V - 1`, giving the usual `O(VE)` form. Exact Python
integers prevent overflow in accumulated and adjusted costs.

Vertices are integer IDs in `0..vertex_count-1`. Edge weights are signed-32
inputs, parallel directed edges remain distinct candidates, and self-loops are
validated but ignored. Incoming edges to the root cannot be selected. A
missing incoming candidate at any contraction level proves that no rooted
spanning arborescence exists.

Only the optimum cost is returned. Equal-cost edge choices are intentionally
not exposed, so this function does not reconstruct a tree, preserve original
edge identity through contraction, provide a tie policy, count optima, handle
multiple roots, or solve undirected, degree-constrained, or dynamic variants.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Deterministic Minimum Spanning Forest with Kruskal's Algorithm](build-a-deterministic-minimum-spanning-forest-with-kruskals-algorithm.md)
- [Compute the Exact Minimum Directed Cycle Mean with Karp's Algorithm](compute-the-exact-minimum-directed-cycle-mean-with-karps-algorithm.md)
- [Send a Required Flow at Minimum Cost with a Residual Certificate](send-a-required-flow-at-minimum-cost-with-a-residual-certificate.md)
<!-- catalog:related:end -->
