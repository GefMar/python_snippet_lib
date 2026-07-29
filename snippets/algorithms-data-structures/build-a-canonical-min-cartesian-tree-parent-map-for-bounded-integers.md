---
title: "Build a Canonical Min-Cartesian-Tree Parent Map for Bounded Integers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-the-canonical-largest-rectangle-under-a-bounded-integer-histogram.md
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
---

# Build a Canonical Min-Cartesian-Tree Parent Map for Bounded Integers

## Idea and Problem

Represent one bounded integer sequence as the unique binary tree that preserves input order and gives every parent the smaller value-and-index key.

A min-Cartesian tree combines two invariants. Its inorder traversal visits the
original indexes from left to right, while every parent has a smaller key than
either child. Adding the index to the key makes repeated values deterministic:
the earlier index wins whenever equal values compete as the root of one span.

The result stores only a parent for each original index. The root has `None`;
for every other position, comparing its index with its parent reveals whether
it is in the parent's left or right subtree. A monotonic stack constructs this
map without recursively searching for the minimum of every span.

## When to Use

Use this representation for one immutable sequence when the Cartesian-tree
structure itself is useful, such as preparing a static range-minimum method,
reasoning about nearest lower boundaries, or validating an equivalent tree
built by another algorithm. Exact index-based ties make repeated values safe
for reproducible tests and serialized parent maps.

Use a direct recursive minimum search for tiny fixtures when simplicity matters
more than worst-case time. Use a segment tree when values change or arbitrary
range-minimum queries must be answered directly. A parent map alone is not an
LCA or RMQ query index; those capabilities require additional preprocessing.

## Implementation

```python
_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_CARTESIAN_VALUES = 100_000


def build_min_cartesian_tree_parent_map(
    values: tuple[int, ...],
) -> tuple[int | None, ...]:
    """Return parent indexes for the canonical min-Cartesian tree."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if len(values) > _MAX_CARTESIAN_VALUES:
        raise ValueError("value count exceeds the supported limit")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact non-boolean integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    parents: list[int | None] = [None] * len(values)
    stack: list[int] = []

    for index, value in enumerate(values):
        last_popped: int | None = None
        while stack and value < values[stack[-1]]:
            last_popped = stack.pop()

        if stack:
            parents[index] = stack[-1]
        if last_popped is not None:
            parents[last_popped] = index
        stack.append(index)

    return tuple(parents)
```

## Example

```python
def cartesian_parents_by_recursive_minimum(
    values: tuple[int, ...],
) -> tuple[int | None, ...]:
    parents: list[int | None] = [None] * len(values)

    def build(start: int, stop: int, parent: int | None) -> None:
        if start == stop:
            return
        root = min(range(start, stop), key=lambda index: (values[index], index))
        parents[root] = parent
        build(start, root, root)
        build(root + 1, stop, root)

    build(0, len(values), None)
    return tuple(parents)


def assert_cartesian_parent_invariants(
    values: tuple[int, ...],
    parents: tuple[int | None, ...],
) -> None:
    assert len(parents) == len(values)
    if not values:
        assert parents == ()
        return

    roots = [index for index, parent in enumerate(parents) if parent is None]
    assert len(roots) == 1
    left_children: list[int | None] = [None] * len(values)
    right_children: list[int | None] = [None] * len(values)

    for child, parent in enumerate(parents):
        if parent is None:
            continue
        assert type(parent) is int and 0 <= parent < len(values) and parent != child
        assert (values[parent], parent) < (values[child], child)
        children = left_children if child < parent else right_children
        assert children[parent] is None
        children[parent] = child

    inorder: list[int] = []
    pending: list[int] = []
    current: int | None = roots[0]
    while pending or current is not None:
        while current is not None:
            pending.append(current)
            current = left_children[current]
        current = pending.pop()
        inorder.append(current)
        current = right_children[current]
    assert tuple(inorder) == tuple(range(len(values)))


def exercise_short_cartesian_trees() -> int:
    from itertools import product

    checked = 0
    for length in range(8):
        for values in product((-1, 0, 2), repeat=length):
            actual = build_min_cartesian_tree_parent_map(values)
            assert actual == cartesian_parents_by_recursive_minimum(values)
            assert_cartesian_parent_invariants(values, actual)
            checked += 1
    return checked


increasing = tuple(range(_MAX_CARTESIAN_VALUES))
decreasing = tuple(range(_MAX_CARTESIAN_VALUES, 0, -1))
all_equal = (7,) * _MAX_CARTESIAN_VALUES
increasing_parents = build_min_cartesian_tree_parent_map(increasing)
decreasing_parents = build_min_cartesian_tree_parent_map(decreasing)
equal_parents = build_min_cartesian_tree_parent_map(all_equal)

for boundary_values, boundary_parents in (
    (increasing, increasing_parents),
    (decreasing, decreasing_parents),
    (all_equal, equal_parents),
):
    assert_cartesian_parent_invariants(boundary_values, boundary_parents)

extrema = (_MAX_INT64, _MIN_INT64, _MAX_INT64, _MIN_INT64)
extrema_parents = build_min_cartesian_tree_parent_map(extrema)

try:
    build_min_cartesian_tree_parent_map((0, True))
except TypeError:
    boolean_rejected = True
else:
    boolean_rejected = False

try:
    build_min_cartesian_tree_parent_map((_MAX_INT64 + 1,))
except ValueError:
    range_rejected = True
else:
    range_rejected = False

try:
    build_min_cartesian_tree_parent_map((0,) * (_MAX_CARTESIAN_VALUES + 1))
except ValueError:
    count_cap_enforced = True
else:
    count_cap_enforced = False

assert (
    exercise_short_cartesian_trees(),
    increasing_parents[:3],
    increasing_parents[-3:],
    decreasing_parents[:3],
    decreasing_parents[-3:],
    equal_parents[:3],
    extrema_parents,
    boolean_rejected,
    range_rejected,
    count_cap_enforced,
) == (
    3_280,
    (None, 0, 1),
    (99_996, 99_997, 99_998),
    (1, 2, 3),
    (99_998, 99_999, None),
    (None, 0, 1),
    (1, None, 3, 1),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Validation and construction take `O(N)` time. Each index is pushed once and
popped at most once, so the stack performs `O(N)` comparisons and occupies
`O(N)` memory. The parent list, stack, and immutable result use `O(N)` memory;
the list and returned tuple briefly coexist.

The strict pop condition is part of the canonical contract. A later equal
value does not pop an earlier index, matching the `(value, index)` heap key.
The parent map and fixed inorder indexes uniquely describe the binary tree,
but they do not provide constant-time children, ancestor, or range-minimum
queries without another representation or preprocessing step.

The function handles one immutable integer snapshot. It does not accept custom
keys or generic objects, build a max-Cartesian tree, support insertion or
replacement, serialize the tree, construct an LCA/RMQ index, or enumerate
alternative trees under another equality policy.

## Related Snippets

<!-- catalog:related:start -->
- [Find the Canonical Largest Rectangle Under a Bounded Integer Histogram](find-the-canonical-largest-rectangle-under-a-bounded-integer-histogram.md)
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
<!-- catalog:related:end -->
