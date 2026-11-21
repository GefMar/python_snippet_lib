---
title: "Compute Each Leave-One-Out Product Under a Composite Modulus"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - answer-static-range-product-queries-modulo-an-integer-with-a-disjoint-sparse-table.md
  - invert-bounded-nonzero-prime-field-residues-with-one-modular-inversion.md
---

# Compute Each Leave-One-Out Product Under a Composite Modulus

## Idea and Problem

Compute the product of every factor except the one at each position, even when modular division is unavailable.

The result at position `i` is the product of the prefix before `i` and the
suffix after `i`. Accumulate the prefixes from left to right and multiply in
the suffixes from right to left. Reducing each multiplication modulo the
modulus preserves the required residue. Zeros and noninvertible factors work
under the same rule.

## When to Use

Use this when a finite product has one factor per record and every record
needs the result with its own factor removed. Supply integers and an integer
modulus of at least two. The modulus can be composite. Equal factors at
different positions are treated as separate occurrences.

## Implementation

```python
from collections.abc import Iterable


def leave_one_out_products(values: Iterable[int], modulus: int) -> tuple[int, ...]:
    if type(modulus) is not int:
        raise TypeError("modulus must be an integer")
    if modulus < 2:
        raise ValueError("modulus must be at least two")

    factors: list[int] = []
    for value in values:
        if type(value) is not int:
            raise TypeError("each factor must be an integer")
        factors.append(value % modulus)

    result = [1] * len(factors)
    prefix = 1
    for index, factor in enumerate(factors):
        result[index] = prefix
        prefix = prefix * factor % modulus

    suffix = 1
    for index in range(len(factors) - 1, -1, -1):
        result[index] = result[index] * suffix % modulus
        suffix = suffix * factors[index] % modulus
    return tuple(result)
```

## Example

```python
assert (
    leave_one_out_products(iter((2, 3, 4)), modulus=12),
    leave_one_out_products((2, 0, 3), modulus=10),
    leave_one_out_products((0, 0, 7), modulus=10),
    leave_one_out_products((-3, 1, 6), modulus=10),
    leave_one_out_products((4,), modulus=12),
    leave_one_out_products((), modulus=12),
) == ((0, 8, 6), (0, 6, 0), (0, 0, 0), (6, 2, 7), (1,), ())
```

## Trade-offs and Limitations

The algorithm performs a linear number of modular multiplications and stores
the reduced factors plus the output. A singleton returns the empty product
`1`, and an empty input returns an empty tuple. Input reduction and later
multiplications still depend on the bit lengths involved.

The reverse pass requires all factors to be available. If only one position
is excluded, use a single product over the remaining factors. For arbitrary
range queries or interleaved updates, use an index designed for those queries.

## Related Snippets

<!-- catalog:related:start -->
- [Answer Static Range Product Queries Modulo an Integer with a Disjoint Sparse Table](answer-static-range-product-queries-modulo-an-integer-with-a-disjoint-sparse-table.md)
- [Invert Bounded Nonzero Prime-Field Residues with One Modular Inversion](invert-bounded-nonzero-prime-field-residues-with-one-modular-inversion.md)
<!-- catalog:related:end -->
