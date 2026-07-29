---
title: "Compute Exact Dominator Sets in a Bounded Directed Flow Graph"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md
  - compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md
  - find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md
---

# Compute Exact Dominator Sets in a Bounded Directed Flow Graph

## Idea and Problem

Find every vertex that must occur on every directed path from one declared start to each reachable vertex.

After a reachability pass, represent each dominator set as a Python-integer bit
mask. The start begins with only itself, while every other reachable vertex
begins with all reachable vertices. A synchronous round replaces each
non-start set with the vertex's singleton plus the intersection of its
reachable predecessors' previous-round sets.

The descending fixed point is exact. Unreachable vertices remain explicitly
`None` because ordinary start-relative dominance is undefined for them.

## When to Use

Use this algorithm for small static directed flow graphs when complete
dominator sets, a transparent fixed point, and deterministic tuple output are
more valuable than asymptotic speed. It also makes a practical oracle for a
more advanced implementation.

Use a specialized dominator algorithm for large control-flow graphs, repeated
queries, immediate-dominator trees, dominance frontiers, or incremental
updates. Decide separately how exceptional entries or multiple entry points
should be modeled.

## Implementation

```python
_MAX_DOMINATOR_VERTICES = 500
_MAX_DOMINATOR_EDGES = 20_000


def exact_dominator_sets(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
    *,
    start: int,
) -> tuple[tuple[int, ...] | None, ...]:
    """Return sorted start-relative dominators, or None when unreachable."""
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact integer")
    if not 1 <= vertex_count <= _MAX_DOMINATOR_VERTICES:
        raise ValueError("vertex_count is outside the supported range")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if len(edges) > _MAX_DOMINATOR_EDGES:
        raise ValueError("edge count exceeds the supported range")
    if type(start) is not int:
        raise TypeError("start must be an exact integer")
    if not 0 <= start < vertex_count:
        raise ValueError("start is outside the graph")

    successors: list[list[int]] = [[] for _ in range(vertex_count)]
    predecessors: list[list[int]] = [[] for _ in range(vertex_count)]
    seen_edges: set[tuple[int, int]] = set()

    for edge_index, edge in enumerate(edges):
        if type(edge) is not tuple:
            raise TypeError(f"edges[{edge_index}] must be an exact tuple")
        if len(edge) != 2:
            raise ValueError(f"edges[{edge_index}] must contain two vertices")
        source, target = edge
        if type(source) is not int or type(target) is not int:
            raise TypeError(f"edges[{edge_index}] vertices must be exact integers")
        if not 0 <= source < vertex_count or not 0 <= target < vertex_count:
            raise ValueError(f"edges[{edge_index}] is outside the graph")
        if edge in seen_edges:
            raise ValueError(f"edges[{edge_index}] duplicates an earlier edge")
        seen_edges.add(edge)
        successors[source].append(target)
        predecessors[target].append(source)

    reachable = [False] * vertex_count
    stack = [start]
    while stack:
        vertex = stack.pop()
        if reachable[vertex]:
            continue
        reachable[vertex] = True
        stack.extend(successors[vertex])

    reachable_vertices = tuple(
        vertex for vertex, is_reachable in enumerate(reachable) if is_reachable
    )
    reachable_mask = sum(1 << vertex for vertex in reachable_vertices)
    dominator_masks = [0] * vertex_count
    for vertex in reachable_vertices:
        dominator_masks[vertex] = reachable_mask
    dominator_masks[start] = 1 << start

    for _ in range(len(reachable_vertices)):
        next_masks = dominator_masks.copy()

        for vertex in reachable_vertices:
            if vertex == start:
                continue
            intersection = reachable_mask
            has_reachable_predecessor = False
            for predecessor in predecessors[vertex]:
                if reachable[predecessor]:
                    intersection &= dominator_masks[predecessor]
                    has_reachable_predecessor = True
            if not has_reachable_predecessor:
                raise AssertionError(
                    "a reachable non-start vertex must have a reachable predecessor"
                )
            next_masks[vertex] = intersection | (1 << vertex)

        if next_masks == dominator_masks:
            break
        dominator_masks = next_masks
    else:
        raise AssertionError("dominator fixed point exceeded its round bound")

    result: list[tuple[int, ...] | None] = []
    for vertex in range(vertex_count):
        if not reachable[vertex]:
            result.append(None)
            continue
        mask = dominator_masks[vertex]
        result.append(
            tuple(candidate for candidate in range(vertex_count) if mask & (1 << candidate))
        )
    return tuple(result)
```

## Example

```python
graph = exact_dominator_sets(
    6,
    (
        (0, 1),
        (0, 2),
        (1, 3),
        (2, 3),
        (3, 4),
        (4, 3),
        (5, 5),
    ),
    start=0,
)
chain = exact_dominator_sets(
    4,
    ((0, 1), (1, 2), (2, 3)),
    start=0,
)
singleton = exact_dominator_sets(1, ((0, 0),), start=0)

assert graph == (
    (0,),
    (0, 1),
    (0, 2),
    (0, 3),
    (0, 3, 4),
    None,
)
assert chain == (
    (0,),
    (0, 1),
    (0, 1, 2),
    (0, 1, 2, 3),
)
assert singleton == ((0,),)
```

## Trade-offs and Limitations

Let `V` and `E` be the admitted graph sizes and `R` the reachable vertex
count. At most `R` full synchronous rounds are needed, including a final
unchanged round. Each round scans the reachable vertices and their reachable
predecessor edges, giving at most `O(V * (V + E))` Python-integer bitset
operations. Each bit operation can itself touch `O(V)` bits. Stored dominator
masks occupy `O(V^2)` bits, excluding graph lists and materialized tuple output.

Keeping updates synchronous is part of the stated round bound and makes the
iteration independent of edge order. Updating masks in place can converge to
the same fixed point but changes propagation timing and invalidates that
specific accounting. Self-edges do not change the recurrence; duplicate edges
are rejected rather than silently normalized.

The function returns full sets, not immediate dominators, a dominator tree, or
dominance frontiers. It has one explicit start, does not add a synthetic entry,
does not explain individual path witnesses, and does not support graph changes
after computation. The simple fixed point is intentionally not a replacement
for asymptotically faster algorithms on large graphs.

## Related Snippets

<!-- catalog:related:start -->
- [Partition a Bounded Directed Graph into Deterministic Strongly Connected Components](partition-a-bounded-directed-graph-into-deterministic-strongly-connected-components.md)
- [Compute the Transitive Reduction of a Bounded Directed Acyclic Graph](compute-the-transitive-reduction-of-a-bounded-directed-acyclic-graph.md)
- [Find Articulation Points and Bridges in a Bounded Undirected Graph](find-articulation-points-and-bridges-in-a-bounded-undirected-graph.md)
<!-- catalog:related:end -->
