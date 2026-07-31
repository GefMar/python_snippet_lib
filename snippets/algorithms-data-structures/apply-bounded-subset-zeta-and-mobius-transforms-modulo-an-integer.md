---
title: "Apply Bounded Subset Zeta and Möbius Transforms Modulo an Integer"
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
  - convolve-bounded-integer-vectors-under-xor-with-the-fast-walsh-hadamard-transform.md
  - count-perfect-matchings-in-a-bounded-balanced-bipartite-graph-with-subset-dp.md
  - count-topological-orders-of-a-bounded-directed-graph-with-subset-dp.md
---

# Apply Bounded Subset Zeta and Möbius Transforms Modulo an Integer

## Idea and Problem

Aggregate every bitmask over all of its submasks, or recover the original values, without enumerating each subset pair.

For an array indexed by `K`-bit masks, the subset zeta transform replaces
`value[mask]` with the sum of `value[submask]` for every submask of `mask`.
Processing one bit at a time shares partial sums between masks, reducing the
work from `3**K` direct contributions to `K * 2**K` additions. Reversing each
addition with subtraction gives the subset Möbius transform.

Both operations are performed modulo an explicit integer. No division is
needed, so the inverse remains valid for composite as well as prime moduli.
Bit `i` of an index consistently represents set element `i`.

## When to Use

Use these transforms when a value is attached to every subset of a small
ground set and a calculation needs all submask totals at once. Typical uses
include inclusion-exclusion preparation, subset statistics, and converting
between exact-set and cumulative-subset representations.

The complete table must fit in memory and its length must be a power of two.
Choose a modulus when results naturally live in modular arithmetic or exact
integer totals could grow too large. Use direct submask enumeration for only a
few queried masks, and use a problem-specific sparse method when most masks are
absent.

## Implementation

```python
_MAX_SUBSET_TRANSFORM_SIZE = 1 << 16
_MAX_SUBSET_MODULUS = (1 << 31) - 1


def _validated_subset_table(values: tuple[int, ...], modulus: int) -> list[int]:
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if type(modulus) is not int:
        raise TypeError("modulus must be an exact integer")
    if not 2 <= modulus <= _MAX_SUBSET_MODULUS:
        raise ValueError("modulus is outside 2..2^31-1")

    size = len(values)
    if not 1 <= size <= _MAX_SUBSET_TRANSFORM_SIZE:
        raise ValueError("values must contain 1..65,536 residues")
    if size & (size - 1):
        raise ValueError("values length must be a power of two")
    if any(type(value) is not int for value in values):
        raise TypeError("every value must be an exact integer")
    if any(not 0 <= value < modulus for value in values):
        raise ValueError("every value must be a canonical residue")
    return list(values)


def subset_zeta(values: tuple[int, ...], modulus: int) -> tuple[int, ...]:
    """Return sums over all submasks, reduced modulo modulus."""
    transformed = _validated_subset_table(values, modulus)
    size = len(transformed)

    stride = 1
    while stride < size:
        block = stride * 2
        for start in range(0, size, block):
            for offset in range(stride):
                lower = start + offset
                upper = lower + stride
                transformed[upper] = (transformed[upper] + transformed[lower]) % modulus
        stride = block
    return tuple(transformed)


def subset_mobius(values: tuple[int, ...], modulus: int) -> tuple[int, ...]:
    """Invert a subset zeta transform over integers modulo modulus."""
    transformed = _validated_subset_table(values, modulus)
    size = len(transformed)

    stride = 1
    while stride < size:
        block = stride * 2
        for start in range(0, size, block):
            for offset in range(stride):
                lower = start + offset
                upper = lower + stride
                transformed[upper] = (transformed[upper] - transformed[lower]) % modulus
        stride = block
    return tuple(transformed)
```

## Example

```python
from itertools import product


def direct_subset_sums(values: tuple[int, ...], modulus: int) -> tuple[int, ...]:
    totals: list[int] = []
    for mask in range(len(values)):
        total = 0
        submask = mask
        while True:
            total += values[submask]
            if submask == 0:
                break
            submask = (submask - 1) & mask
        totals.append(total % modulus)
    return tuple(totals)


def rejects(values: object, modulus: object) -> bool:
    try:
        subset_zeta(values, modulus)  # type: ignore[arg-type]
    except (TypeError, ValueError):
        return True
    return False


examined_tables = 0
for small_modulus in (2, 3, 4):
    for size in (1, 2, 4):
        for small_values in product(range(small_modulus), repeat=size):
            expected = direct_subset_sums(small_values, small_modulus)
            assert subset_zeta(small_values, small_modulus) == expected
            assert subset_mobius(expected, small_modulus) == small_values
            examined_tables += 1

basis = tuple(int(index == 3) for index in range(8))
basis_zeta = subset_zeta(basis, 12)
maximum_shape = (0,) * _MAX_SUBSET_TRANSFORM_SIZE

assert (
    examined_tables,
    basis_zeta,
    subset_mobius(basis_zeta, 12),
    subset_zeta(maximum_shape, 2)[-1],
    rejects((0, 1, 0), 5),
    rejects((0, 5), 5),
    rejects((0, True), 5),
    rejects((0,), 1),
    rejects([], 5),
    rejects((), 5),
    rejects((0,) * (_MAX_SUBSET_TRANSFORM_SIZE * 2), 5),
    rejects((0,), True),
) == (
    391,
    (0, 0, 0, 1, 0, 0, 0, 1),
    basis,
    0,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Each transform performs `O(N log N)` modular additions or subtractions and
retains `O(N)` residues for `N` masks. It always processes the full dense table,
even when only a few results are needed. Python's loop and arbitrary-precision
integer costs still apply, although the stored residues remain below the
declared modulus.

The direction is specifically from submasks to containing masks. A superset
transform needs the opposite update direction, while XOR, AND, and OR
convolutions have different algebras. The functions also do not perform subset
convolution, choose the ground-set ordering, or attach meaning to unused masks.

Canonical residues are required instead of being normalized silently. This
makes malformed inputs visible and ensures the one-element table is an exact
identity under both operations.

## Related Snippets

<!-- catalog:related:start -->
- [Convolve Bounded Integer Vectors under XOR with the Fast Walsh-Hadamard Transform](convolve-bounded-integer-vectors-under-xor-with-the-fast-walsh-hadamard-transform.md)
- [Count Perfect Matchings in a Bounded Balanced Bipartite Graph with Subset DP](count-perfect-matchings-in-a-bounded-balanced-bipartite-graph-with-subset-dp.md)
- [Count Topological Orders of a Bounded Directed Graph with Subset DP](count-topological-orders-of-a-bounded-directed-graph-with-subset-dp.md)
<!-- catalog:related:end -->
