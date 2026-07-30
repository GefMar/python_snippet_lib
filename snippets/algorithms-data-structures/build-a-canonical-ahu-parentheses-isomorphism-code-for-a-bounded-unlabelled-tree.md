---
title: "Build a Canonical AHU Parentheses Isomorphism Code for a Bounded Unlabelled Tree"
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
  - find-a-canonical-diameter-path-in-a-bounded-undirected-tree.md
  - find-every-centroid-of-a-bounded-undirected-tree.md
---

# Build a Canonical AHU Parentheses Isomorphism Code for a Bounded Unlabelled Tree

## Idea and Problem

Build one opaque balanced-parentheses code that is equal exactly when two validated bounded trees are isomorphic after vertex labels are ignored.

The AHU rooted-tree encoding wraps the sorted encodings of a vertex's children
between `(` and `)`. Sorting removes child declaration order, while the
parentheses retain the complete branching structure. An unrooted tree has one
center or two adjacent centers. Encoding the whole tree from every center and
taking the lexicographically smaller full code removes the remaining root
choice.

The result depends only on the abstract unlabelled tree. Vertex indexes are
used solely to declare the input edge set; changing every index consistently,
reordering edges, or reversing endpoints cannot change the returned code.

## When to Use

Use this function when small or medium unweighted trees need an exact equality
key for isomorphism grouping, fixture comparison, or independent structural
checks. Comparing two returned strings decides whether the corresponding
validated trees have the same unlabelled shape.

Treat the code as an opaque equality value. Use a labelled-tree representation
when vertex identities matter, and use a dedicated isomorphism implementation
when a vertex-to-vertex mapping, canonical relabelling, automorphism count,
labels, weights, or general graphs are required.

## Implementation

```python
_MAX_AHU_VERTICES = 512
_UNSEEN_PARENT = -2
_NO_PARENT = -1


def _validated_ahu_tree(
    vertex_count: object,
    edges: object,
) -> list[list[int]]:
    if type(vertex_count) is not int:
        raise TypeError("vertex_count must be an exact non-boolean integer")
    if type(edges) is not tuple:
        raise TypeError("edges must be an exact tuple")
    if not 1 <= vertex_count <= _MAX_AHU_VERTICES:
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

    parents = [_UNSEEN_PARENT] * vertex_count
    parents[0] = _NO_PARENT
    order = [0]
    for vertex in order:
        for neighbor in adjacency[vertex]:
            if parents[neighbor] != _UNSEEN_PARENT:
                continue
            parents[neighbor] = vertex
            order.append(neighbor)
    if len(order) != vertex_count:
        raise ValueError("edges must form one connected tree")

    return adjacency


def _ahu_tree_centers(adjacency: list[list[int]]) -> tuple[int, ...]:
    if len(adjacency) == 1:
        return (0,)

    degrees = [len(neighbors) for neighbors in adjacency]
    leaves = [vertex for vertex, degree in enumerate(degrees) if degree == 1]
    remaining = len(adjacency)
    while remaining > 2:
        remaining -= len(leaves)
        next_leaves: list[int] = []
        for leaf in leaves:
            for neighbor in adjacency[leaf]:
                if degrees[neighbor] == 0:
                    continue
                degrees[neighbor] -= 1
                if degrees[neighbor] == 1:
                    next_leaves.append(neighbor)
            degrees[leaf] = 0
        leaves = next_leaves
    return tuple(leaves)


def _rooted_ahu_parentheses_code(
    adjacency: list[list[int]],
    root: int,
) -> str:
    parents = [_UNSEEN_PARENT] * len(adjacency)
    parents[root] = _NO_PARENT
    order = [root]
    for vertex in order:
        for neighbor in adjacency[vertex]:
            if neighbor == parents[vertex]:
                continue
            parents[neighbor] = vertex
            order.append(neighbor)

    codes = [""] * len(adjacency)
    for vertex in reversed(order):
        child_codes = [
            codes[neighbor] for neighbor in adjacency[vertex] if parents[neighbor] == vertex
        ]
        child_codes.sort()
        codes[vertex] = "(" + "".join(child_codes) + ")"
    return codes[root]


def build_canonical_ahu_tree_code(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> str:
    """Return an opaque unlabelled-tree isomorphism code."""
    adjacency = _validated_ahu_tree(vertex_count, edges)
    return min(
        _rooted_ahu_parentheses_code(adjacency, center) for center in _ahu_tree_centers(adjacency)
    )
```

## Example

```python
def tree_from_prufer(
    sequence: tuple[int, ...],
) -> tuple[tuple[int, int], ...]:
    vertex_count = len(sequence) + 2
    degrees = [1] * vertex_count
    for vertex in sequence:
        degrees[vertex] += 1

    edges: list[tuple[int, int]] = []
    for neighbor in sequence:
        leaf = next(vertex for vertex, degree in enumerate(degrees) if degree == 1)
        edges.append((leaf, neighbor))
        degrees[leaf] -= 1
        degrees[neighbor] -= 1

    remaining = [vertex for vertex, degree in enumerate(degrees) if degree == 1]
    edges.append((remaining[0], remaining[1]))
    return tuple(edges)


def canonical_adjacency_key_by_permutations(
    vertex_count: int,
    edges: tuple[tuple[int, int], ...],
) -> tuple[int, tuple[tuple[int, int], ...]]:
    from itertools import permutations

    canonical_edges = min(
        tuple(
            sorted(
                (
                    min(relabeling[first], relabeling[second]),
                    max(relabeling[first], relabeling[second]),
                )
                for first, second in edges
            )
        )
        for relabeling in permutations(range(vertex_count))
    )
    return vertex_count, canonical_edges


def exercise_small_unlabelled_trees() -> tuple[int, int]:
    from itertools import product

    key_to_code: dict[tuple[int, tuple[tuple[int, int], ...]], str] = {}
    code_to_key: dict[str, tuple[int, tuple[tuple[int, int], ...]]] = {}
    checked = 0

    cases = [(1, ())]
    for vertex_count in range(2, 7):
        cases.extend(
            (vertex_count, tree_from_prufer(sequence))
            for sequence in product(
                range(vertex_count),
                repeat=vertex_count - 2,
            )
        )

    for vertex_count, edges in cases:
        code = build_canonical_ahu_tree_code(vertex_count, edges)
        key = canonical_adjacency_key_by_permutations(vertex_count, edges)
        assert key_to_code.setdefault(key, code) == code
        assert code_to_key.setdefault(code, key) == key

        reversed_declarations = tuple((second, first) for first, second in reversed(edges))
        assert build_canonical_ahu_tree_code(vertex_count, reversed_declarations) == code
        checked += 1

    return checked, len(key_to_code)


def is_balanced_parentheses(code: str) -> bool:
    balance = 0
    for token in code:
        balance += 1 if token == "(" else -1
        if balance < 0:
            return False
    return balance == 0


maximum_path = tuple((vertex, vertex + 1) for vertex in range(_MAX_AHU_VERTICES - 1))
maximum_star = tuple((0, vertex) for vertex in range(1, _MAX_AHU_VERTICES))
path_code = build_canonical_ahu_tree_code(_MAX_AHU_VERTICES, maximum_path)
star_code = build_canonical_ahu_tree_code(_MAX_AHU_VERTICES, maximum_star)

value_errors = 0
for vertex_count, invalid_edges in (
    (0, ()),
    (_MAX_AHU_VERTICES + 1, ()),
    (3, ((0, 1),)),
    (3, ((0, 1), (1, 0))),
    (4, ((0, 1), (1, 2), (2, 0))),
    (2, ((0, 2),)),
    (2, ((0, 0),)),
    (2, ((0, 1, 2),)),
):
    try:
        build_canonical_ahu_tree_code(vertex_count, invalid_edges)
    except ValueError:
        value_errors += 1

type_errors = 0
for vertex_count, invalid_edges in (
    (True, ()),
    (1, []),
    (2, ([0, 1],)),
    (2, ((0, True),)),
    (2, ((0, 1.0),)),
):
    try:
        build_canonical_ahu_tree_code(vertex_count, invalid_edges)
    except TypeError:
        type_errors += 1

assert (
    exercise_small_unlabelled_trees(),
    build_canonical_ahu_tree_code(1, ()),
    build_canonical_ahu_tree_code(2, ((1, 0),)),
    build_canonical_ahu_tree_code(3, ((0, 1), (1, 2))),
    build_canonical_ahu_tree_code(4, ((0, 1), (0, 2), (0, 3))),
    len(path_code),
    is_balanced_parentheses(path_code),
    len(star_code),
    is_balanced_parentheses(star_code),
    path_code != star_code,
    value_errors,
    type_errors,
) == (
    (1_442, 14),
    "()",
    "(())",
    "(()())",
    "(()()())",
    1_024,
    True,
    1_024,
    True,
    True,
    8,
    5,
)
```

## Trade-offs and Limitations

Validation and center peeling take expected `O(V + E)` time and
`O(V + E)` memory; duplicate detection uses hashing. The returned code has
exactly `2 * V` characters. This direct implementation stores the complete
string for every rooted subtree. Its bounded simplicity trades away the
rank-compressed AHU variants: it can use `O(V**2)` intermediate characters,
while copied strings and comparison-based child sorting take up to
`O(V**2 * log(V))` character work. The 512-vertex limit makes those costs
explicit.

The rooted encoding is exactly `"(" + "".join(sorted(child_codes)) + ")"`.
For a two-center tree, the function compares the two complete rooted encodings,
not independently encoded center halves. Ordinary string ordering makes the
choice reproducible. The output is an opaque equality key rather than a
canonical vertex order or a format with a decoder.

Only simple, connected, unweighted, undirected trees are accepted. The
function does not return an isomorphism mapping, preserve vertex identities,
process forests or general graphs, handle vertex or edge labels, count
automorphisms, update a dynamic tree, or promise compatibility with another
AHU serialization convention.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode Bounded Labelled Trees with Prüfer Sequences](encode-and-decode-bounded-labelled-trees-with-prufer-sequences.md)
- [Find a Canonical Diameter Path in a Bounded Undirected Tree](find-a-canonical-diameter-path-in-a-bounded-undirected-tree.md)
- [Find Every Centroid of a Bounded Undirected Tree](find-every-centroid-of-a-bounded-undirected-tree.md)
<!-- catalog:related:end -->
