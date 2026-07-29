---
title: "Build a Canonical Reduced XOR Basis for Bounded Unsigned Integers"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md
  - decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
---

# Build a Canonical Reduced XOR Basis for Bounded Unsigned Integers

## Idea and Problem

Reduce bounded unsigned integers to the unique binary row-reduced basis of exactly the same XOR span.

Treat each bit position as a column over the two-element field, where XOR is
vector addition. Descending elimination first keeps at most one vector for
each highest set bit. A second pass clears every retained pivot from all other
rows, producing reduced row-echelon form rather than an insertion-order basis.

The returned vectors are ordered by descending pivot bit. This makes the result
independent of input order, duplicate values, zero values, and the particular
dependent rows used to describe the same span.

## When to Use

Use a reduced XOR basis when a bounded collection of unsigned bit patterns
needs one canonical linear-span representation. It supports reproducible span
comparison and is a compact starting point for membership, rank, and attainable
XOR-value calculations.

Use a set when only exact input membership matters. Use ordinary integer,
modular, or rational linear algebra when addition is not XOR, and use packed
native arrays when substantially wider or larger binary matrices must be
processed with tight memory and throughput requirements.

## Implementation

```python
_MIN_XOR_BIT_WIDTH = 1
_MAX_XOR_BIT_WIDTH = 64
_MAX_XOR_VALUE_COUNT = 100_000


def canonical_reduced_xor_basis(
    values: tuple[int, ...],
    *,
    bit_width: int,
) -> tuple[int, ...]:
    """Return the unique descending-pivot reduced basis of the input XOR span."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if type(bit_width) is not int:
        raise TypeError("bit_width must be an exact non-boolean integer")
    if not _MIN_XOR_BIT_WIDTH <= bit_width <= _MAX_XOR_BIT_WIDTH:
        raise ValueError("bit_width is outside the supported range")
    if len(values) > _MAX_XOR_VALUE_COUNT:
        raise ValueError("value count exceeds the supported limit")

    upper_bound = 1 << bit_width
    for value_index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{value_index}] must be an exact integer")
        if not 0 <= value < upper_bound:
            raise ValueError(f"values[{value_index}] must be unsigned and fit within bit_width")

    basis = [0] * bit_width
    for declared_value in values:
        reduced = declared_value
        for pivot in range(bit_width - 1, -1, -1):
            pivot_mask = 1 << pivot
            if reduced & pivot_mask == 0:
                continue
            if basis[pivot] == 0:
                basis[pivot] = reduced
                break
            reduced ^= basis[pivot]

    for pivot in range(bit_width):
        pivot_vector = basis[pivot]
        if pivot_vector == 0:
            continue
        pivot_mask = 1 << pivot
        for higher_pivot in range(pivot + 1, bit_width):
            if basis[higher_pivot] & pivot_mask:
                basis[higher_pivot] ^= pivot_vector

    return tuple(basis[pivot] for pivot in range(bit_width - 1, -1, -1) if basis[pivot] != 0)
```

## Example

```python
def xor_span(values: tuple[int, ...]) -> frozenset[int]:
    reachable = {0}
    for value in values:
        snapshot = tuple(reachable)
        reachable.update(existing ^ value for existing in snapshot)
    return frozenset(reachable)


def is_fully_reduced(basis: tuple[int, ...]) -> bool:
    pivots = tuple(vector.bit_length() - 1 for vector in basis)
    if pivots != tuple(sorted(pivots, reverse=True)):
        return False
    if len(set(pivots)) != len(pivots):
        return False
    return all(
        row_index == pivot_index or vector & (1 << pivot) == 0
        for row_index, vector in enumerate(basis)
        for pivot_index, pivot in enumerate(pivots)
    )


def brute_canonical_basis(
    span: frozenset[int],
) -> tuple[int, ...]:
    from itertools import combinations

    rank = len(span).bit_length() - 1
    if rank == 0:
        return ()

    matches: list[tuple[int, ...]] = []
    for selected in combinations(sorted(span - {0}), rank):
        candidate = tuple(
            sorted(
                selected,
                key=lambda vector: vector.bit_length(),
                reverse=True,
            )
        )
        if is_fully_reduced(candidate) and xor_span(candidate) == span:
            matches.append(candidate)
    assert len(matches) == 1
    return matches[0]


def exercise_small_xor_spaces() -> tuple[int, int]:
    checked = 0
    distinct_subspaces = 0
    for bit_width in range(1, 5):
        universe = tuple(range(1 << bit_width))
        references: dict[frozenset[int], tuple[int, ...]] = {}
        for declaration_mask in range(1 << len(universe)):
            values = tuple(
                value
                for position, value in enumerate(universe)
                if declaration_mask & (1 << position)
            )
            span = xor_span(values)
            expected = references.get(span)
            if expected is None:
                expected = brute_canonical_basis(span)
                references[span] = expected

            actual = canonical_reduced_xor_basis(
                values,
                bit_width=bit_width,
            )
            reordered = tuple(reversed(values))
            reordered += (values[0],) if values else (0,)
            assert actual == expected
            assert is_fully_reduced(actual)
            assert xor_span(actual) == span
            assert (
                canonical_reduced_xor_basis(
                    reordered,
                    bit_width=bit_width,
                )
                == expected
            )
            checked += 1
        distinct_subspaces += len(references)
    return checked, distinct_subspaces


def raises(error_type: type[Exception], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


maximum_values = ((1 << 64) - 1, 1 << 63, 1, 0) * (_MAX_XOR_VALUE_COUNT // 4)
maximum_basis = canonical_reduced_xor_basis(
    maximum_values,
    bit_width=64,
)

validation_rejections = (
    raises(
        TypeError,
        lambda: canonical_reduced_xor_basis([1], bit_width=1),
    ),
    raises(
        TypeError,
        lambda: canonical_reduced_xor_basis((1,), bit_width=True),
    ),
    raises(
        ValueError,
        lambda: canonical_reduced_xor_basis((0,), bit_width=0),
    ),
    raises(
        ValueError,
        lambda: canonical_reduced_xor_basis((0,), bit_width=65),
    ),
    raises(
        TypeError,
        lambda: canonical_reduced_xor_basis((False,), bit_width=1),
    ),
    raises(
        ValueError,
        lambda: canonical_reduced_xor_basis((-1,), bit_width=1),
    ),
    raises(
        ValueError,
        lambda: canonical_reduced_xor_basis((2,), bit_width=1),
    ),
    raises(
        ValueError,
        lambda: canonical_reduced_xor_basis(
            (0,) * (_MAX_XOR_VALUE_COUNT + 1),
            bit_width=1,
        ),
    ),
)

assert (
    exercise_small_xor_spaces(),
    canonical_reduced_xor_basis(
        (0b1101, 0b1011, 0b0110),
        bit_width=4,
    ),
    canonical_reduced_xor_basis((), bit_width=1),
    len(maximum_values),
    maximum_basis,
    xor_span(maximum_basis) == xor_span(maximum_values),
    all(validation_rejections),
) == (
    (65_812, 90),
    (0b1011, 0b0110),
    (),
    100_000,
    (1 << 63, (1 << 63) - 2, 1),
    True,
    True,
)
```

## Trade-offs and Limitations

For m declared values and bit width w, validation and descending elimination
use O(mw) bounded-word operations. Full pivot clearing adds O(w²) operations.
The working basis and immutable result use O(w) memory; the supplied input
tuple is already materialized. With the declared maximum of 64 bits, every
intermediate remains a bounded non-negative Python integer.

Reduction intentionally discards declaration order, duplicates, zero rows, and
the coefficients needed to reconstruct a basis vector from particular input
positions. The canonical tuple describes a span, not a multiset or provenance
record. Building all attainable XOR values can still require exponential
2**rank space even though the basis itself has at most 64 vectors.

This function does not solve arithmetic over other finite fields, accept
negative integers, retain vectors wider than the declared bit width, provide a
cryptographic digest, count representations, or perform incremental deletion.
A mutable online basis can make insertion cheaper to expose, but it must be
fully reduced again before its representation is canonical.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Decide Bounded Non-Negative Subset-Sum Reachability with an Integer Bitset](decide-bounded-non-negative-subset-sum-reachability-with-an-integer-bitset.md)
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
<!-- catalog:related:end -->
