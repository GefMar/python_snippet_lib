---
title: "Fit an Exact Unweighted Isotonic Regression with Pool-Adjacent Violators"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md
  - compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md
  - ../algorithms-data-structures/rank-bounded-records-with-stable-ties-and-neighbor-windows.md
---

# Fit an Exact Unweighted Isotonic Regression with Pool-Adjacent Violators

## Idea and Problem

Fit a bounded integer sequence to the unique nondecreasing least-squares sequence while preserving every fitted level as an exact fraction.

The pool-adjacent-violators algorithm starts with one block per observation.
Whenever two neighboring block means are out of order, their observations are
pooled and the mean is recomputed. Pooling equal means as well gives one
canonical maximal block for every fitted level.

## When to Use

Use this algorithm when observations have a fixed meaningful order, every
observation has equal weight, and the fitted values must not decrease. Exact
`Fraction` levels are useful when deterministic comparisons matter and rounding
to binary floating point would obscure equal fitted blocks.

Use a statistical library when observations have weights, missing values, or
floating-point inputs, or when decreasing, multidimensional, online, or
uncertainty-aware regression is required. The complete input must be available
in memory, and the expanded fit is materialized, so this implementation is
intended for bounded data.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_OBSERVATIONS = 4_096


@dataclass(frozen=True, slots=True)
class IsotonicBlock:
    start: int
    stop: int
    count: int
    total: int
    mean: Fraction


@dataclass(frozen=True, slots=True)
class IsotonicFit:
    blocks: tuple[IsotonicBlock, ...]
    fitted: tuple[Fraction, ...]


def fit_unweighted_isotonic_regression(values: tuple[int, ...]) -> IsotonicFit:
    """Return the exact nondecreasing least-squares fit of integer values."""
    if type(values) is not tuple:
        raise TypeError("values must be an exact tuple")
    if not 1 <= len(values) <= _MAX_OBSERVATIONS:
        raise ValueError("observation count is outside the supported range")

    for index, value in enumerate(values):
        if type(value) is not int:
            raise TypeError(f"values[{index}] must be an exact integer")
        if not _MIN_INT64 <= value <= _MAX_INT64:
            raise ValueError(f"values[{index}] is outside the signed 64-bit range")

    working_blocks: list[tuple[int, int, int, int]] = []
    for index, value in enumerate(values):
        working_blocks.append((index, index + 1, 1, value))
        while len(working_blocks) >= 2:
            left = working_blocks[-2]
            right = working_blocks[-1]
            if left[3] * right[2] < right[3] * left[2]:
                break
            working_blocks.pop()
            working_blocks.pop()
            working_blocks.append(
                (
                    left[0],
                    right[1],
                    left[2] + right[2],
                    left[3] + right[3],
                )
            )

    blocks = tuple(
        IsotonicBlock(
            start=start,
            stop=stop,
            count=count,
            total=total,
            mean=Fraction(total, count),
        )
        for start, stop, count, total in working_blocks
    )
    fitted = tuple(
        block.mean
        for block in blocks
        for _ in range(block.count)
    )
    return IsotonicFit(blocks=blocks, fitted=fitted)
```

## Example

```python
fit = fit_unweighted_isotonic_regression((3, 1, 2, 2, 5, 4))
constant = fit_unweighted_isotonic_regression((7, 7, 7))

assert (fit.blocks, fit.fitted, constant.blocks) == (
    (
        IsotonicBlock(0, 4, 4, 8, Fraction(2)),
        IsotonicBlock(4, 6, 2, 9, Fraction(9, 2)),
    ),
    (Fraction(2),) * 4 + (Fraction(9, 2),) * 2,
    (IsotonicBlock(0, 3, 3, 21, Fraction(7)),),
)
```

## Trade-offs and Limitations

Each observation creates one stack block, and every merge removes two blocks
before adding their union, so the algorithm performs `O(n)` stack operations
and uses `O(n)` memory for the working stack, frozen blocks, and expanded fit.
The fitted vector uniquely minimizes the sum of squared deviations over all
nondecreasing real sequences; within each final block, its least-squares
constant is that block's arithmetic mean.

Totals, cross-products, and fitted fractions use exact Python integer
arithmetic even when derived values exceed the signed 64-bit input range. The
linear operation count treats arithmetic as unit cost: multiplication, gcd,
and `Fraction` construction costs still depend on operand bit length. Merging
on greater-than-or-equal means makes neighboring final levels strictly
increasing and therefore makes equal-level blocks maximal.

This fit is one-dimensional and unweighted. It does not accept floats or
missing values, compute a decreasing fit, impose multidimensional constraints,
support online updates, quantify statistical uncertainty, or choose a later
float-conversion or rounding policy.

## Related Snippets

<!-- catalog:related:start -->
- [Accumulate and Merge Finite Mean and Variance Statistics Under a Count Limit](accumulate-and-merge-finite-mean-and-variance-statistics-under-a-count-limit.md)
- [Compute an Exact Integer Median and Unscaled Median Absolute Deviation](compute-an-exact-integer-median-and-unscaled-median-absolute-deviation.md)
- [Rank Bounded Records with Stable Ties and Neighbor Windows](../algorithms-data-structures/rank-bounded-records-with-stable-ties-and-neighbor-windows.md)
<!-- catalog:related:end -->
