---
title: "Build a Canonical Optimal Binary Search Tree from Bounded Successful and Gap Weights"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md
  - build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md
  - build-a-canonical-min-cartesian-tree-parent-map-for-bounded-integers.md
---

# Build a Canonical Optimal Binary Search Tree from Bounded Successful and Gap Weights

## Idea and Problem

Choose a binary search tree shape that minimizes exact weighted depths for both successful keys and unsuccessful gaps.

Keys already have sorted positions, so every subtree corresponds to one
contiguous key interval. Trying each key as that interval's root combines the
best left and right subtrees. Moving both subtrees one level deeper adds the
sum of every successful-key and unsuccessful-gap weight in the interval.

The empty interval has one external gap leaf whose base cost is its weight.
Consequently a full tree uses the classical external-path convention: the root
key has depth one and its adjacent gap leaves have depth two. Choosing the
smallest root index on an equal interval cost makes reconstruction stable.

## When to Use

Use this dynamic program for a small static ordered key set when successful and
unsuccessful lookup weights are known and an exact, reproducible tree shape is
more important than construction speed. Integer counts can be supplied
directly; they do not need to be normalized probabilities.

Use a balanced search tree for changing keys or adversarial access patterns.
Use a cache-aware layout optimizer when memory hierarchy, branch prediction,
updates, or measured latency matters more than weighted abstract depth.

## Implementation

```python
from dataclasses import dataclass
from functools import cache
from random import Random

_MAX_OPTIMAL_BST_KEYS = 128
_MAX_ACCESS_WEIGHT = 2**31 - 1
_MAX_TOTAL_ACCESS_WEIGHT = 2**63 - 1


@dataclass(frozen=True, slots=True)
class OptimalBinarySearchTree:
    weighted_path_cost: int
    root_index: int
    left_children: tuple[int | None, ...]
    right_children: tuple[int | None, ...]


def canonical_optimal_binary_search_tree(
    successful_weights: tuple[int, ...],
    gap_weights: tuple[int, ...],
) -> OptimalBinarySearchTree:
    """Return the smallest-root-tied optimal static BST shape."""
    if type(successful_weights) is not tuple:
        raise TypeError("successful_weights must be an exact tuple")
    key_count = len(successful_weights)
    if not 1 <= key_count <= _MAX_OPTIMAL_BST_KEYS:
        raise ValueError("key count is outside 1..128")
    if type(gap_weights) is not tuple:
        raise TypeError("gap_weights must be an exact tuple")
    if len(gap_weights) != key_count + 1:
        raise ValueError("gap_weights must contain exactly key_count + 1 values")

    for weight in (*successful_weights, *gap_weights):
        if type(weight) is not int:
            raise TypeError("access weights must be exact integers")
        if not 0 <= weight <= _MAX_ACCESS_WEIGHT:
            raise ValueError("access weight is outside the nonnegative signed 32-bit range")
    total_weight = sum(successful_weights) + sum(gap_weights)
    if not 1 <= total_weight <= _MAX_TOTAL_ACCESS_WEIGHT:
        raise ValueError("combined access weight is outside 1..2**63 - 1")

    successful_prefix = [0]
    for weight in successful_weights:
        successful_prefix.append(successful_prefix[-1] + weight)
    gap_prefix = [0]
    for weight in gap_weights:
        gap_prefix.append(gap_prefix[-1] + weight)

    costs = [[0] * (key_count + 1) for _ in range(key_count + 1)]
    roots = [[-1] * (key_count + 1) for _ in range(key_count + 1)]
    for boundary, gap_weight in enumerate(gap_weights):
        costs[boundary][boundary] = gap_weight

    for width in range(1, key_count + 1):
        for left in range(key_count - width + 1):
            right = left + width
            interval_weight = (
                successful_prefix[right]
                - successful_prefix[left]
                + gap_prefix[right + 1]
                - gap_prefix[left]
            )
            best_cost: int | None = None
            best_root = -1
            for root in range(left, right):
                candidate = costs[left][root] + costs[root + 1][right] + interval_weight
                if best_cost is None or candidate < best_cost:
                    best_cost = candidate
                    best_root = root
            if best_cost is None:
                raise AssertionError("every nonempty interval has a root")
            costs[left][right] = best_cost
            roots[left][right] = best_root

    root_index = roots[0][key_count]
    left_children: list[int | None] = [None] * key_count
    right_children: list[int | None] = [None] * key_count
    pending = [(0, key_count)]
    while pending:
        left, right = pending.pop()
        root = roots[left][right]
        if left < root:
            child = roots[left][root]
            left_children[root] = child
            pending.append((left, root))
        if root + 1 < right:
            child = roots[root + 1][right]
            right_children[root] = child
            pending.append((root + 1, right))

    return OptimalBinarySearchTree(
        weighted_path_cost=costs[0][key_count],
        root_index=root_index,
        left_children=tuple(left_children),
        right_children=tuple(right_children),
    )
```

## Example

```python

Shape = tuple[int, "Shape | None", "Shape | None"]


@cache
def all_shapes(left: int, right: int) -> tuple[Shape | None, ...]:
    if left == right:
        return (None,)
    return tuple(
        (root, left_shape, right_shape)
        for root in range(left, right)
        for left_shape in all_shapes(left, root)
        for right_shape in all_shapes(root + 1, right)
    )


def direct_weighted_cost(
    shape: Shape | None,
    successful: tuple[int, ...],
    gaps: tuple[int, ...],
    left: int,
    right: int,
    depth: int,
) -> int:
    if shape is None:
        assert left == right
        return gaps[left] * depth
    root, left_shape, right_shape = shape
    return (
        successful[root] * depth
        + direct_weighted_cost(left_shape, successful, gaps, left, root, depth + 1)
        + direct_weighted_cost(right_shape, successful, gaps, root + 1, right, depth + 1)
    )


def shape_key(shape: Shape | None) -> tuple[object, ...]:
    if shape is None:
        return ()
    root, left, right = shape
    return root, shape_key(left), shape_key(right)


def shape_result(
    shape: Shape,
    cost: int,
    key_count: int,
) -> OptimalBinarySearchTree:
    left_children: list[int | None] = [None] * key_count
    right_children: list[int | None] = [None] * key_count
    pending = [shape]
    while pending:
        root, left, right = pending.pop()
        if left is not None:
            left_children[root] = left[0]
            pending.append(left)
        if right is not None:
            right_children[root] = right[0]
            pending.append(right)
    return OptimalBinarySearchTree(cost, shape[0], tuple(left_children), tuple(right_children))


def exhaustive_tree_oracle(
    successful: tuple[int, ...],
    gaps: tuple[int, ...],
) -> OptimalBinarySearchTree:
    candidates = tuple(
        (
            direct_weighted_cost(shape, successful, gaps, 0, len(successful), 1),
            shape_key(shape),
            shape,
        )
        for shape in all_shapes(0, len(successful))
        if shape is not None
    )
    cost, _, best_shape = min(candidates, key=lambda candidate: (candidate[0], candidate[1]))
    assert best_shape is not None
    return shape_result(best_shape, cost, len(successful))


assert canonical_optimal_binary_search_tree((3, 3), (1, 1, 1)) == OptimalBinarySearchTree(
    weighted_path_cost=17,
    root_index=0,
    left_children=(None, None),
    right_children=(1, None),
)

rng = Random(0x0B57)
checked = 0
for _ in range(5_000):
    key_count = rng.randrange(1, 8)
    successful = tuple(rng.randrange(5) for _ in range(key_count))
    gaps = tuple(rng.randrange(5) for _ in range(key_count + 1))
    if sum(successful) + sum(gaps) == 0:
        gaps = (1, *gaps[1:])
    assert canonical_optimal_binary_search_tree(
        successful,
        gaps,
    ) == exhaustive_tree_oracle(successful, gaps)
    checked += 1

assert checked == 5_000
```

## Trade-offs and Limitations

For `N` keys, the deliberately transparent interval dynamic program takes
`O(N³)` time and retains `O(N²)` cost and root tables. Knuth optimization can
reduce the time for this recurrence, but it adds a second invariant and is not
needed under the 128-key teaching bound. Exact Python integer arithmetic keeps
comparisons stable even when weighted costs exceed a machine word.

The canonical rule is local and explicit: on an equal interval cost, the
smallest root index wins, after which both subintervals apply the same rule.
Gap leaves influence the weighted cost but are not returned as ordinary key
nodes. The objective is classical weighted external-path depth, not a claim
about measured comparison counts or cache behavior.

The result is a static shape over already sorted key positions. It does not
store keys or values, maintain balance after updates, infer future access
frequencies, or protect against adversarial changes in workload.

## Related Snippets

<!-- catalog:related:start -->
- [Plan a Minimum-Cost Parenthesization for a Bounded Matrix Chain](plan-a-minimum-cost-parenthesization-for-a-bounded-matrix-chain.md)
- [Build a Bounded Immutable Text Trie for Longest-Prefix Lookup](build-a-bounded-immutable-text-trie-for-longest-prefix-lookup.md)
- [Build a Canonical Min-Cartesian-Tree Parent Map for Bounded Integers](build-a-canonical-min-cartesian-tree-parent-map-for-bounded-integers.md)
<!-- catalog:related:end -->
