---
title: "Answer Static Range Product Queries Modulo an Integer with a Disjoint Sparse Table"
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
  - answer-static-half-open-range-minimum-queries-with-a-sparse-table.md
  - maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md
  - maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md
---

# Answer Static Range Product Queries Modulo an Integer with a Disjoint Sparse Table

## Idea and Problem

Precompute disjoint suffix and prefix products so every non-empty static range product modulo an integer needs at most one final multiplication.

At level `k`, each block has two halves of length `2**k`. The table stores a
product from every position in the left half through that half's end, and a
product from the right half's start through every position in that half.

For a wider query, the highest bit where `start` and `stop - 1` differ selects
one block whose halves separate the two endpoints. One stored suffix and one
stored prefix then cover every requested value exactly once. This disjoint
coverage matters because multiplying overlapping blocks would count their
shared factors twice.

## When to Use

Use this index when an immutable bounded integer sequence receives many
range-product queries under one fixed modulus. It is especially useful when
zeros or a composite modulus make division by a prefix product unavailable.

Use a direct `math.prod` scan for a few short ranges. Use a segment tree when
values can change, and use ordinary prefix products with modular inverses only
when every admitted factor is invertible and that stronger contract is useful.
The source tuple and the complete disjoint table must fit in memory.

## Implementation

```python
_MAX_STATIC_RANGE_PRODUCT_VALUES = 32_768
_MAX_STATIC_RANGE_PRODUCT_MODULUS = (1 << 31) - 1


class StaticRangeProductMod:
    """Index non-empty static half-open range products under one modulus."""

    __slots__ = ("_levels", "_modulus", "_values")

    def __init__(self, values: tuple[int, ...], modulus: int) -> None:
        if type(values) is not tuple:
            raise TypeError("values must be an exact tuple")
        if not 1 <= len(values) <= _MAX_STATIC_RANGE_PRODUCT_VALUES:
            raise ValueError("value count is outside 1..32,768")
        if type(modulus) is not int:
            raise TypeError("modulus must be an exact integer")
        if not 2 <= modulus <= _MAX_STATIC_RANGE_PRODUCT_MODULUS:
            raise ValueError("modulus is outside 2..2^31-1")

        for index, value in enumerate(values):
            if type(value) is not int:
                raise TypeError(f"values[{index}] must be an exact integer")
            if not 0 <= value < modulus:
                raise ValueError(f"values[{index}] is not a canonical residue")

        size = len(values)
        levels: list[tuple[int, ...]] = []
        for level in range((size - 1).bit_length()):
            half_length = 1 << level
            block_length = 2 * half_length
            products = [0] * size

            for block_start in range(0, size, block_length):
                middle = min(block_start + half_length, size)
                block_stop = min(block_start + block_length, size)

                products[middle - 1] = values[middle - 1]
                for position in range(middle - 2, block_start - 1, -1):
                    products[position] = (
                        values[position] * products[position + 1]
                    ) % modulus

                if middle < block_stop:
                    products[middle] = values[middle]
                    for position in range(middle + 1, block_stop):
                        products[position] = (
                            products[position - 1] * values[position]
                        ) % modulus

            levels.append(tuple(products))

        self._values = values
        self._modulus = modulus
        self._levels = tuple(levels)

    def product(self, start: int, stop: int) -> int:
        """Return the product over the non-empty half-open range [start, stop)."""
        if type(start) is not int:
            raise TypeError("start must be an exact integer")
        if type(stop) is not int:
            raise TypeError("stop must be an exact integer")
        if not 0 <= start < stop <= len(self._values):
            raise ValueError("range must satisfy 0 <= start < stop <= value count")

        if stop - start == 1:
            return self._values[start]

        level = (start ^ (stop - 1)).bit_length() - 1
        left_product = self._levels[level][start]
        right_product = self._levels[level][stop - 1]
        return left_product * right_product % self._modulus
```

## Example

```python
from itertools import product
from math import prod
from random import Random


def direct_range_product(
    values: tuple[int, ...],
    modulus: int,
    start: int,
    stop: int,
) -> int:
    return prod(values[start:stop]) % modulus


def exercise_every_range(values: tuple[int, ...], modulus: int) -> int:
    table = StaticRangeProductMod(values, modulus)
    checked = 0
    for start in range(len(values)):
        for stop in range(start + 1, len(values) + 1):
            assert table.product(start, stop) == direct_range_product(
                values,
                modulus,
                start,
                stop,
            )
            checked += 1
    return checked


checked_ranges = 0
for small_modulus, largest_size in ((2, 7), (4, 4)):
    for size in range(1, largest_size + 1):
        for small_values in product(range(small_modulus), repeat=size):
            checked_ranges += exercise_every_range(small_values, small_modulus)

generator = Random(0xD15_1017)
for size in (3, 5, 6, 7, 9, 17, 31, 33, 65):
    for small_modulus in (6, 97):
        for _ in range(4):
            random_values = tuple(
                generator.randrange(small_modulus) for _ in range(size)
            )
            checked_ranges += exercise_every_range(random_values, small_modulus)

ordinary_values = (2, 3, 0, 5, 4, 2)
ordinary = StaticRangeProductMod(ordinary_values, 12)
ordinary_answers = tuple(
    ordinary.product(start, stop)
    for start, stop in ((0, 2), (0, 6), (3, 6), (3, 4), (4, 6))
)

maximum_values = (0,) * _MAX_STATIC_RANGE_PRODUCT_VALUES
maximum = StaticRangeProductMod(maximum_values, 12)
maximum_ranges = (
    (0, _MAX_STATIC_RANGE_PRODUCT_VALUES),
    (1, _MAX_STATIC_RANGE_PRODUCT_VALUES - 1),
    (_MAX_STATIC_RANGE_PRODUCT_VALUES - 1, _MAX_STATIC_RANGE_PRODUCT_VALUES),
)
maximum_answers = tuple(
    maximum.product(start, stop) for start, stop in maximum_ranges
)
maximum_oracle = tuple(
    direct_range_product(maximum_values, 12, start, stop)
    for start, stop in maximum_ranges
)


def raises(error_type: type[BaseException], operation) -> bool:
    try:
        operation()
    except error_type:
        return True
    return False


invalid_calls = (
    lambda: StaticRangeProductMod([1], 2),
    lambda: StaticRangeProductMod((), 2),
    lambda: StaticRangeProductMod((0,), True),
    lambda: StaticRangeProductMod((0,), 1),
    lambda: StaticRangeProductMod((0,), _MAX_STATIC_RANGE_PRODUCT_MODULUS + 1),
    lambda: StaticRangeProductMod((True,), 2),
    lambda: StaticRangeProductMod((-1,), 2),
    lambda: StaticRangeProductMod((2,), 2),
    lambda: StaticRangeProductMod(
        (0,) * (_MAX_STATIC_RANGE_PRODUCT_VALUES + 1),
        2,
    ),
    lambda: ordinary.product(False, 1),
    lambda: ordinary.product(0, True),
    lambda: ordinary.product(0, 0),
    lambda: ordinary.product(4, 3),
    lambda: ordinary.product(0, len(ordinary_values) + 1),
)

assert (
    checked_ranges,
    ordinary_answers,
    maximum_answers,
    maximum_answers == maximum_oracle,
    sum(raises((TypeError, ValueError), call) for call in invalid_calls),
) == (
    36_386,
    (6, 0, 4, 5, 8),
    (0, 0, 0),
    True,
    len(invalid_calls),
)
```

## Trade-offs and Limitations

For `N` values, construction performs `O(N log N)` modular operations and
retains `O(N log N)` table entries. A non-singleton query performs two indexed
reads and one modular multiplication, while a singleton returns its stored
residue directly, so each query uses `O(1)` arithmetic. Python multiplication
and remainder still have operand-dependent costs; the explicit modulus cap
bounds operands in this profile.

At the maximum size, 15 levels retain 491,520 product positions in addition
to the source tuple and container overhead. This is a larger memory commitment
than a segment tree, and rebuilding is required after any source change.

Only exact tuples of canonical residues and non-empty one-dimensional ranges
are accepted. The class does not normalize negative integers, supply an
empty-range identity, support updates, change moduli, compact values into
native arrays, or generalize the table to caller-provided associative
operations.

## Related Snippets

<!-- catalog:related:start -->
- [Answer Static Half-Open Range-Minimum Queries with a Sparse Table](answer-static-half-open-range-minimum-queries-with-a-sparse-table.md)
- [Maintain Bounded Point Adds and Half-Open Range Sums with a Fenwick Tree](maintain-bounded-point-adds-and-half-open-range-sums-with-a-fenwick-tree.md)
- [Maintain Point Replacements and Half-Open Range Minima with a Segment Tree](maintain-point-replacements-and-half-open-range-minima-with-a-segment-tree.md)
<!-- catalog:related:end -->
