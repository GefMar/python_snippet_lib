---
title: "Encode and Decode Bounded Labelled Trees with Prüfer Sequences"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-canonical-diameter-path-in-a-bounded-undirected-tree.md
  - find-every-centroid-of-a-bounded-undirected-tree.md
  - compute-exact-unweighted-distance-sums-for-every-vertex-of-a-bounded-tree.md
---

# Encode and Decode Bounded Labelled Trees with Prüfer Sequences

## Idea and Problem

Represent a labelled tree as a compact integer sequence and recover one canonical edge tuple from that sequence.

A tree over `n` labels has a Prüfer sequence of length `n - 2`. Encoding
repeatedly removes the smallest current leaf and records its only remaining
neighbor. Decoding reverses that rule: occurrence counts determine initial
degrees, and the smallest current leaf is joined to each recorded label.

The resulting correspondence is one-to-one for trees whose labels are the
closed range from zero through `n - 1`. Sorting normalized decoded edges makes
the returned representation independent of construction order and endpoint
orientation.

## When to Use

Use Prüfer sequences when bounded labelled trees need a deterministic compact
form, exhaustive enumeration, round-trip validation, or reproducible test-data
generation. The encoder accepts one complete simple connected tree with at
least two vertices. The decoder infers the vertex count from the code and
accepts an empty code as the two-vertex tree.

This representation is useful for labelled trees rather than tree-shaped
objects whose node identities are irrelevant. Use an ordinary edge list when
the sequence will not be enumerated, compared, or transmitted, and use a
domain-specific format when edges carry weights or other attributes.

## Implementation

```python
from collections import deque
from heapq import heapify, heappop, heappush

_MAX_PRUFER_VERTICES = 100_000
_MAX_PRUFER_CODE_LENGTH = _MAX_PRUFER_VERTICES - 2


def _validated_prufer_tree(
    vertex_count: object,
    edges: object,
) -> list[list[int]]:
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 2 <= vertex_count <= _MAX_PRUFER_VERTICES:
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

    visited = [False] * vertex_count
    visited[0] = True
    visited_count = 1
    pending: deque[int] = deque([0])
    while pending:
        vertex = pending.popleft()
        for neighbor in adjacency[vertex]:
            if visited[neighbor]:
                continue
            visited[neighbor] = True
            visited_count += 1
            pending.append(neighbor)
    if visited_count != vertex_count:
        raise ValueError("edges must form one connected tree")

    return adjacency


def encode_tree_as_prufer(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, ...]:
    """Return the Prüfer sequence of one validated labelled tree."""
    adjacency = _validated_prufer_tree(vertex_count, edges)
    degrees = [len(neighbors) for neighbors in adjacency]
    leaves = [vertex for vertex, degree in enumerate(degrees) if degree == 1]
    heapify(leaves)

    code: list[int] = []
    for _ in range(vertex_count - 2):
        leaf = heappop(leaves)
        neighbor = next(candidate for candidate in adjacency[leaf] if degrees[candidate] > 0)
        code.append(neighbor)
        degrees[leaf] = 0
        degrees[neighbor] -= 1
        if degrees[neighbor] == 1:
            heappush(leaves, neighbor)
    return tuple(code)


def decode_prufer_sequence(
    code: tuple[int, ...],
) -> tuple[tuple[int, int], ...]:
    """Return sorted normalized edges for one validated Prüfer sequence."""
    if type(code) is not tuple:
        raise TypeError("code must be an exact tuple")
    if len(code) > _MAX_PRUFER_CODE_LENGTH:
        raise ValueError("code exceeds the supported length")

    vertex_count = len(code) + 2
    degrees = [1] * vertex_count
    for code_index, vertex in enumerate(code):
        if type(vertex) is not int:
            raise TypeError(f"code[{code_index}] must be an exact integer")
        if not 0 <= vertex < vertex_count:
            raise ValueError(f"code[{code_index}] is outside the labelled tree")
        degrees[vertex] += 1

    leaves = [vertex for vertex, degree in enumerate(degrees) if degree == 1]
    heapify(leaves)
    edges: list[tuple[int, int]] = []

    for neighbor in code:
        leaf = heappop(leaves)
        edge = (leaf, neighbor) if leaf < neighbor else (neighbor, leaf)
        edges.append(edge)
        degrees[leaf] -= 1
        degrees[neighbor] -= 1
        if degrees[neighbor] == 1:
            heappush(leaves, neighbor)

    first_leaf = heappop(leaves)
    second_leaf = heappop(leaves)
    edges.append((first_leaf, second_leaf))
    return tuple(sorted(edges))
```

## Example

```python
def exercise_small_prufer_spaces() -> int:
    from itertools import product

    checked = 0
    for vertex_count in range(2, 8):
        for code in product(
            range(vertex_count),
            repeat=vertex_count - 2,
        ):
            edges = decode_prufer_sequence(code)
            reversed_declarations = tuple((second, first) for first, second in reversed(edges))
            assert encode_tree_as_prufer(vertex_count, edges) == code
            assert encode_tree_as_prufer(vertex_count, reversed_declarations) == code
            assert decode_prufer_sequence(encode_tree_as_prufer(vertex_count, edges)) == edges
            checked += 1
    return checked


maximum_path_edges = tuple((vertex, vertex + 1) for vertex in range(_MAX_PRUFER_VERTICES - 1))
maximum_star_edges = tuple((0, vertex) for vertex in range(1, _MAX_PRUFER_VERTICES))
path_code = encode_tree_as_prufer(
    _MAX_PRUFER_VERTICES,
    maximum_path_edges,
)
star_code = encode_tree_as_prufer(
    _MAX_PRUFER_VERTICES,
    maximum_star_edges,
)

value_errors = 0
for vertex_count, invalid_edges in (
    (1, ()),
    (3, ((0, 1),)),
    (3, ((0, 1), (1, 0))),
    (4, ((0, 1), (1, 2), (2, 0))),
    (2, ((0, 2),)),
    (2, ((0, 0),)),
    (2, ((0, 1, 2),)),
):
    try:
        encode_tree_as_prufer(vertex_count, invalid_edges)
    except ValueError:
        value_errors += 1

for invalid_code in (
    (3,),
    (0,) * (_MAX_PRUFER_CODE_LENGTH + 1),
):
    try:
        decode_prufer_sequence(invalid_code)
    except ValueError:
        value_errors += 1

type_errors = 0
for vertex_count, invalid_edges in (
    (True, ()),
    (2, [(0, 1)]),
    (2, ([0, 1],)),
    (2, ((0, True),)),
):
    try:
        encode_tree_as_prufer(vertex_count, invalid_edges)
    except TypeError:
        type_errors += 1

for invalid_code in (
    [0],
    (False,),
):
    try:
        decode_prufer_sequence(invalid_code)
    except TypeError:
        type_errors += 1

assert (
    exercise_small_prufer_spaces(),
    decode_prufer_sequence(()),
    len(path_code),
    path_code[:3],
    path_code[-3:],
    decode_prufer_sequence(path_code) == maximum_path_edges,
    len(star_code),
    star_code[:3],
    star_code[-3:],
    decode_prufer_sequence(star_code) == maximum_star_edges,
    value_errors,
    type_errors,
) == (
    18_248,
    ((0, 1),),
    99_998,
    (1, 2, 3),
    (99_996, 99_997, 99_998),
    True,
    99_998,
    (0, 0, 0),
    (0, 0, 0),
    True,
    9,
    6,
)
```

## Trade-offs and Limitations

Encoding validates the tree and scans its adjacency lists in `O(n)` total
work, while heap operations make both encoding and decoding `O(n log n)`.
The decoded-edge sort has the same upper bound. Adjacency lists, degrees, the
heap and the immutable result use `O(n)` memory.

The sequence is canonical only for the declared integer labels. Relabelling
the same abstract tree generally changes its code, and the representation does
not establish isomorphism between differently labelled trees. The
implementation deliberately rejects the one-vertex tree because the classical
`n - 2` sequence contract starts at two vertices.

Only simple, connected, undirected, unweighted static trees are represented.
The code does not preserve edge order, endpoint orientation, weights, metadata
or a preferred root. It does not encode forests, general graphs, dynamic
updates or unlabelled-tree equivalence.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Canonical Diameter Path in a Bounded Undirected Tree](find-a-canonical-diameter-path-in-a-bounded-undirected-tree.md)
- [Find Every Centroid of a Bounded Undirected Tree](find-every-centroid-of-a-bounded-undirected-tree.md)
- [Compute Exact Unweighted Distance Sums for Every Vertex of a Bounded Tree](compute-exact-unweighted-distance-sums-for-every-vertex-of-a-bounded-tree.md)
<!-- catalog:related:end -->
