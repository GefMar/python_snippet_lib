---
title: "Derive Deterministic Huffman Code Lengths from Bounded Byte Frequencies"
snippet_type: algorithm
use_cases:
  - data-transformation
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md
  - ../algorithms-data-structures/estimate-stream-frequencies-with-a-count-min-sketch.md
---

# Derive Deterministic Huffman Code Lengths from Bounded Byte Frequencies

## Idea and Problem

Derive optimal binary prefix-code lengths from exact byte frequencies while making every equal-weight merge deterministic.

Huffman construction repeatedly joins the two lightest active subtrees. Each
heap entry also carries the smallest byte symbol in its subtree, so equal
weights are resolved independently of declaration order. Traversing the final
tree yields lengths; assigning actual canonical bit patterns remains a separate
step.

## When to Use

Use this algorithm when a small, complete byte-frequency table is already
available and a reproducible optimal set of prefix-code lengths is required.
The exact integer weights may be counts or any other positive relative costs;
multiplying every weight by the same positive factor does not change the
optimization problem.

Use a length-limited Huffman algorithm when a format imposes a smaller maximum
code length. Use a format-specific codec when headers, symbol ordering, bit
packing, end markers, or decoding behavior are part of the contract. Deriving
lengths alone does not create an interoperable encoded stream.

## Implementation

```python
from heapq import heapify, heappop, heappush

_MAX_INT64 = (1 << 63) - 1
_MAX_HUFFMAN_SYMBOLS = 64


def derive_deterministic_huffman_code_lengths(
    frequencies: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, int], ...]:
    """Return byte-symbol and Huffman-length pairs ordered by symbol."""
    if type(frequencies) is not tuple:
        raise TypeError("frequencies must be an exact tuple")
    if not 1 <= len(frequencies) <= _MAX_HUFFMAN_SYMBOLS:
        raise ValueError("frequency count is outside the supported range")

    validated: list[tuple[int, int]] = []
    seen_symbols: set[int] = set()
    for index, declaration in enumerate(frequencies):
        if type(declaration) is not tuple:
            raise TypeError(f"frequencies[{index}] must be an exact tuple")
        if len(declaration) != 2:
            raise ValueError(f"frequencies[{index}] must contain two items")

        symbol, frequency = declaration
        if type(symbol) is not int:
            raise TypeError(f"frequencies[{index}][0] must be an exact integer")
        if not 0 <= symbol <= 255:
            raise ValueError(f"frequencies[{index}][0] is outside the byte range")
        if symbol in seen_symbols:
            raise ValueError(f"frequencies[{index}][0] is duplicated")
        seen_symbols.add(symbol)

        if type(frequency) is not int:
            raise TypeError(f"frequencies[{index}][1] must be an exact integer")
        if not 1 <= frequency <= _MAX_INT64:
            raise ValueError(f"frequencies[{index}][1] is outside the supported range")
        validated.append((symbol, frequency))

    if sum(frequency for _, frequency in validated) > _MAX_INT64:
        raise ValueError("total frequency exceeds the supported range")
    if len(validated) == 1:
        return ((validated[0][0], 1),)

    nodes: list[int | tuple[int, int]] = []
    frontier: list[tuple[int, int, int]] = []
    for symbol, frequency in sorted(validated):
        node_index = len(nodes)
        nodes.append(symbol)
        frontier.append((frequency, symbol, node_index))
    heapify(frontier)

    while len(frontier) > 1:
        left_weight, left_minimum, left_index = heappop(frontier)
        right_weight, right_minimum, right_index = heappop(frontier)
        parent_index = len(nodes)
        nodes.append((left_index, right_index))
        heappush(
            frontier,
            (
                left_weight + right_weight,
                min(left_minimum, right_minimum),
                parent_index,
            ),
        )

    lengths: dict[int, int] = {}
    stack = [(frontier[0][2], 0)]
    while stack:
        node_index, depth = stack.pop()
        node = nodes[node_index]
        if type(node) is int:
            lengths[node] = depth
            continue
        left_index, right_index = node
        stack.append((right_index, depth + 1))
        stack.append((left_index, depth + 1))

    return tuple((symbol, lengths[symbol]) for symbol, _ in sorted(validated))
```

## Example

```python
def reference_huffman_lengths(
    frequencies: tuple[tuple[int, int], ...],
) -> tuple[tuple[int, int], ...]:
    if len(frequencies) == 1:
        return ((frequencies[0][0], 1),)

    depths = {symbol: 0 for symbol, _ in frequencies}
    active = [(frequency, symbol, (symbol,)) for symbol, frequency in frequencies]
    while len(active) > 1:
        active.sort(key=lambda item: (item[0], item[1]))
        left = active.pop(0)
        right = active.pop(0)
        members = left[2] + right[2]
        for symbol in members:
            depths[symbol] += 1
        active.append((left[0] + right[0], min(left[1], right[1]), members))
    return tuple(sorted(depths.items()))


def optimal_weighted_path_cost(weights: tuple[int, ...]) -> int:
    from functools import cache

    subset_weights = [0] * (1 << len(weights))
    for mask in range(1, len(subset_weights)):
        lowest_bit = mask & -mask
        index = lowest_bit.bit_length() - 1
        subset_weights[mask] = subset_weights[mask ^ lowest_bit] + weights[index]

    @cache
    def solve(mask: int) -> int:
        if mask & (mask - 1) == 0:
            return 0
        anchor = mask & -mask
        best: int | None = None
        subset = (mask - 1) & mask
        while subset:
            if subset != mask and subset & anchor:
                candidate = solve(subset) + solve(mask ^ subset) + subset_weights[mask]
                best = candidate if best is None else min(best, candidate)
            subset = (subset - 1) & mask
        if best is None:
            raise AssertionError("a non-singleton subset must have a partition")
        return best

    return solve((1 << len(weights)) - 1)


def exercise_huffman_examples() -> tuple[object, ...]:
    from itertools import permutations, product

    cases_checked = 0
    for symbol_count in range(2, 6):
        for weights in product(range(1, 4), repeat=symbol_count):
            declarations = tuple(enumerate(weights))
            derived = derive_deterministic_huffman_code_lengths(declarations)
            reference = reference_huffman_lengths(declarations)
            weighted_cost = sum(weights[symbol] * length for symbol, length in derived)
            maximum_length = max(length for _, length in derived)
            kraft_units = sum(1 << (maximum_length - length) for _, length in derived)
            assert derived == reference
            assert weighted_cost == optimal_weighted_path_cost(weights)
            assert kraft_units == 1 << maximum_length
            cases_checked += 1

    singleton = derive_deterministic_huffman_code_lengths(((17, 5),))
    singleton_kraft = (
        sum(1 << (1 - length) for _, length in singleton),
        1 << 1,
    )
    equal_weights = derive_deterministic_huffman_code_lengths(((3, 7), (2, 7), (1, 7), (0, 7)))
    leaf_subtree_tie = ((0, 1), (1, 1), (2, 2), (3, 2))
    tie_result = derive_deterministic_huffman_code_lengths(leaf_subtree_tie)
    permuted_tie_results = {
        derive_deterministic_huffman_code_lengths(order) for order in permutations(leaf_subtree_tie)
    }

    invalid_inputs = (
        (((0, 1), (0, 2)), ValueError),
        (((True, 1),), TypeError),
        (((0, _MAX_INT64), (1, 1)), ValueError),
        (((0, 1, 2),), ValueError),
    )
    rejected = 0
    for declarations, expected_error in invalid_inputs:
        try:
            derive_deterministic_huffman_code_lengths(declarations)
        except expected_error:
            rejected += 1

    return (
        cases_checked,
        singleton,
        singleton_kraft,
        equal_weights,
        tie_result,
        permuted_tie_results,
        rejected,
    )


assert exercise_huffman_examples() == (
    360,
    ((17, 1),),
    (1, 2),
    ((0, 2), (1, 2), (2, 2), (3, 2)),
    ((0, 3), (1, 3), (2, 2), (3, 1)),
    {((0, 3), (1, 3), (2, 2), (3, 1))},
    4,
)
```

## Trade-offs and Limitations

For `K` symbols, heap construction takes `O(K log K)` time. The node table,
heap, traversal stack, and output use `O(K)` memory. With at most 64 symbols, a
multi-symbol tree has depth at most 63; the explicit one-symbol convention uses
length one instead of an empty code word.

For two or more symbols the result is a complete binary prefix tree and its
code lengths satisfy Kraft equality. The one-symbol convention is deliberately
incomplete, with Kraft sum one half. Huffman lengths minimize weighted path
cost, but equal-cost trees are possible; the minimum-symbol tie rule chooses
one stable result and is not a universal file-format convention.

The function accepts only positive exact frequencies and does not impose a
smaller maximum length. It does not assign canonical bits, pack or decode a
payload, add an end marker, serialize a frequency table, or define framing.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Binary Prefix Codebook from Declared Byte Code Lengths](build-a-canonical-binary-prefix-codebook-from-declared-byte-code-lengths.md)
- [Estimate Stream Frequencies with a Count-Min Sketch](../algorithms-data-structures/estimate-stream-frequencies-with-a-count-min-sketch.md)
<!-- catalog:related:end -->
