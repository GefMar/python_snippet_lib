---
title: "Convolve Bounded Subset Functions Modulo an Integer"
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
  - apply-bounded-subset-zeta-and-mobius-transforms-modulo-an-integer.md
  - convolve-bounded-integer-vectors-under-xor-with-the-fast-walsh-hadamard-transform.md
  - count-perfect-matchings-in-a-bounded-balanced-bipartite-graph-with-subset-dp.md
---

# Convolve Bounded Subset Functions Modulo an Integer

## Idea and Problem

Convolve two dense subset functions without enumerating every submask of every output set.

For arrays indexed by `K`-bit masks, subset convolution is

`result[mask] = sum(left[submask] * right[mask ^ submask])`

over every `submask` contained in `mask`. Direct evaluation takes `3**K`
terms across the complete result. Fast subset convolution groups each input by
subset cardinality, applies a subset zeta transform to every rank, multiplies
the ranked transforms, and applies subset Möbius inversion. The answer for a
mask is the recovered component at that mask's population count.

All arithmetic is modulo an explicit integer. The transforms use addition,
subtraction, and multiplication but no division, so composite moduli are valid.

## When to Use

Use this operation when two values attached to disjoint parts of a subset must
be combined for every possible union. Examples include bounded subset dynamic
programs, exact-set decompositions, and joining two families of local choices.

Both functions must be complete dense tables with the same power-of-two
length, and bit `i` of an index must represent the same ground-set element in
both tables. Use direct submask enumeration for a few requested masks or very
small `K`; use a problem-specific sparse algorithm when most entries are zero.

## Implementation

```python
_MAX_SUBSET_CONVOLUTION_SIZE = 1 << 14
_MAX_SUBSET_CONVOLUTION_MODULUS = (1 << 31) - 1


def _validate_subset_function(
    name: str,
    values: tuple[int, ...],
    modulus: int,
) -> None:
    if type(values) is not tuple:
        raise TypeError(f"{name} must be an exact tuple")
    size = len(values)
    if not 1 <= size <= _MAX_SUBSET_CONVOLUTION_SIZE:
        raise ValueError(f"{name} must contain 1..16,384 residues")
    if size & (size - 1):
        raise ValueError(f"{name} length must be a power of two")
    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"{name}[{index}] must be an exact integer")
        if not 0 <= value < modulus:
            raise ValueError(f"{name}[{index}] is not a canonical residue")


def _subset_zeta_in_place(values: list[int], modulus: int) -> None:
    stride = 1
    while stride < len(values):
        block = 2 * stride
        for start in range(0, len(values), block):
            for offset in range(stride):
                lower = start + offset
                upper = lower + stride
                values[upper] = (values[upper] + values[lower]) % modulus
        stride = block


def _subset_mobius_in_place(values: list[int], modulus: int) -> None:
    stride = 1
    while stride < len(values):
        block = 2 * stride
        for start in range(0, len(values), block):
            for offset in range(stride):
                lower = start + offset
                upper = lower + stride
                values[upper] = (values[upper] - values[lower]) % modulus
        stride = block


def subset_convolve_mod(
    left: tuple[int, ...],
    right: tuple[int, ...],
    modulus: int,
) -> tuple[int, ...]:
    """Return the disjoint-union subset convolution modulo modulus."""
    if type(modulus) is not int:
        raise TypeError("modulus must be an exact integer")
    if not 2 <= modulus <= _MAX_SUBSET_CONVOLUTION_MODULUS:
        raise ValueError("modulus is outside 2..2^31-1")
    _validate_subset_function("left", left, modulus)
    _validate_subset_function("right", right, modulus)
    if len(left) != len(right):
        raise ValueError("left and right must have equal lengths")

    size = len(left)
    bit_count = size.bit_length() - 1
    rank_count = bit_count + 1
    left_ranked = [[0] * size for _ in range(rank_count)]
    right_ranked = [[0] * size for _ in range(rank_count)]
    masks_by_rank: list[list[int]] = [[] for _ in range(rank_count)]

    for mask, (left_value, right_value) in enumerate(
        zip(left, right, strict=True),
    ):
        rank = mask.bit_count()
        left_ranked[rank][mask] = left_value
        right_ranked[rank][mask] = right_value
        masks_by_rank[rank].append(mask)

    for ranked_function in (left_ranked, right_ranked):
        for rank_values in ranked_function:
            _subset_zeta_in_place(rank_values, modulus)

    result = [0] * size
    for output_rank in range(rank_count):
        transformed_product = [0] * size
        for left_rank in range(output_rank + 1):
            left_values = left_ranked[left_rank]
            right_values = right_ranked[output_rank - left_rank]
            for mask in range(size):
                transformed_product[mask] = (
                    transformed_product[mask] + left_values[mask] * right_values[mask]
                ) % modulus

        _subset_mobius_in_place(transformed_product, modulus)
        for mask in masks_by_rank[output_rank]:
            result[mask] = transformed_product[mask]

    return tuple(result)
```

## Example

```python
from itertools import product
from random import Random


def direct_subset_convolution(
    left: tuple[int, ...],
    right: tuple[int, ...],
    modulus: int,
) -> tuple[int, ...]:
    result: list[int] = []
    for mask in range(len(left)):
        total = 0
        submask = mask
        while True:
            total += left[submask] * right[mask ^ submask]
            if submask == 0:
                break
            submask = (submask - 1) & mask
        result.append(total % modulus)
    return tuple(result)


assert subset_convolve_mod((1, 2, 3, 4), (5, 6, 7, 8), 12) == (5, 4, 10, 0)

checked = 0
for small_modulus in (2, 4, 5):
    for size in (1, 2, 4):
        alphabet = range(small_modulus) if size < 4 else range(2)
        vectors = tuple(product(alphabet, repeat=size))
        for left_values in vectors:
            for right_values in vectors:
                expected = direct_subset_convolution(
                    left_values,
                    right_values,
                    small_modulus,
                )
                actual = subset_convolve_mod(
                    left_values,
                    right_values,
                    small_modulus,
                )
                assert actual == expected
                assert actual == subset_convolve_mod(
                    right_values,
                    left_values,
                    small_modulus,
                )
                checked += 1

rng = Random(0x5AB_5E7)
for size in (8, 16):
    for small_modulus in (6, 17):
        for _ in range(25):
            left_values = tuple(rng.randrange(small_modulus) for _ in range(size))
            right_values = tuple(rng.randrange(small_modulus) for _ in range(size))
            assert subset_convolve_mod(
                left_values,
                right_values,
                small_modulus,
            ) == direct_subset_convolution(
                left_values,
                right_values,
                small_modulus,
            )
            checked += 1

modulus = 15
boundary = tuple((index * 7) % modulus for index in range(16))
identity = (1,) + (0,) * 15
assert checked == 1_810
assert subset_convolve_mod(boundary, identity, modulus) == boundary

maximum_shape = (0,) * _MAX_SUBSET_CONVOLUTION_SIZE
assert subset_convolve_mod(maximum_shape, maximum_shape, 2) == maximum_shape
```

## Trade-offs and Limitations

For `N = 2**K` masks, the ranked transforms, rank products, and inversions take
`O(K**2 * N)` modular operations. Two `(K + 1) * N` ranked matrices, one
output-rank work row, and the result retain `O(K * N)` storage. Building and
inverting one output rank at a time avoids retaining a third ranked matrix.

The `N <= 2**14` cap bounds the dense pure-Python work, but it is not a latency
promise. Lists, references, and Python integer objects give the ranked matrices
a substantial constant factor; benchmark maximum-size inputs in the target
environment. Direct enumeration is `O(3**K)` for a complete output table and is
best kept as a small-input oracle.

Inputs and outputs are materialized exact tuples of canonical residues. The
function does not normalize signed values, infer a modulus, choose a bit-to-set
mapping, pad unequal tables, exploit sparsity, or compute XOR, AND, or OR
convolution.

## Related Snippets

<!-- catalog:related:start -->
- [Apply Bounded Subset Zeta and Möbius Transforms Modulo an Integer](apply-bounded-subset-zeta-and-mobius-transforms-modulo-an-integer.md)
- [Convolve Bounded Integer Vectors under XOR with the Fast Walsh-Hadamard Transform](convolve-bounded-integer-vectors-under-xor-with-the-fast-walsh-hadamard-transform.md)
- [Count Perfect Matchings in a Bounded Balanced Bipartite Graph with Subset DP](count-perfect-matchings-in-a-bounded-balanced-bipartite-graph-with-subset-dp.md)
<!-- catalog:related:end -->
