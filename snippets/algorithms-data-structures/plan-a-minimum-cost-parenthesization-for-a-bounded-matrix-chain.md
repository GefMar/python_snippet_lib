---
title: "Plan a Minimum-Cost Parenthesization for a Bounded Matrix Chain"
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
  - solve-a-bounded-tridiagonal-integer-system-exactly-with-thomas-elimination.md
  - compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md
---

# Plan a Minimum-Cost Parenthesization for a Bounded Matrix Chain

## Idea and Problem

Choose a deterministic multiplication order for a compatible matrix chain that minimizes the exact number of scalar multiplications.

Matrix multiplication is associative, but intermediate shapes make different
parenthesizations cost different amounts. Interval dynamic programming records
the cheapest cost and split for every contiguous subchain, then reconstructs
one immutable binary plan.

When several plans have the same cost, choosing the smallest split index for
each optimal subproblem provides a complete reproducible tie rule without
rendering or comparing parenthesized strings.

## When to Use

Use this planner when every matrix dimension is known before execution and
scalar multiplication count is an appropriate cost model. A dimension tuple
`(d0, d1, ..., dm)` describes matrices `A0` through `A(m-1)`, where matrix
`Ai` has shape `di` by `d(i+1)`.

The nested result can drive a separate multiplication routine or explain an
optimizer's choice. Use measured library-specific planning instead when
memory traffic, sparsity, hardware kernels, parallelism, materialization, or
numerical behavior matters more than the textbook scalar-operation count.

## Implementation

```python
from typing import TypeAlias

_MAX_MATRIX_CHAIN_LENGTH = 200
_MAX_MATRIX_DIMENSION = 1_000_000

MatrixChainPlan: TypeAlias = int | tuple["MatrixChainPlan", "MatrixChainPlan"]


def minimum_cost_matrix_chain_plan(
    dimensions: tuple[int, ...],
) -> tuple[int, MatrixChainPlan]:
    """Return the minimum scalar cost and a smallest-split nested plan."""
    if type(dimensions) is not tuple:
        raise TypeError("dimensions must be an exact tuple")
    if not 2 <= len(dimensions) <= _MAX_MATRIX_CHAIN_LENGTH + 1:
        raise ValueError("matrix count is outside the supported range")
    for index, dimension in enumerate(dimensions):
        if type(dimension) is not int:
            raise TypeError(f"dimensions[{index}] must be an exact integer")
        if not 1 <= dimension <= _MAX_MATRIX_DIMENSION:
            raise ValueError(f"dimensions[{index}] is outside the supported range")

    matrix_count = len(dimensions) - 1
    costs = [[0] * matrix_count for _ in range(matrix_count)]
    splits = [[-1] * matrix_count for _ in range(matrix_count)]

    for width in range(2, matrix_count + 1):
        for left in range(matrix_count - width + 1):
            right = left + width - 1
            best_cost: int | None = None
            best_split = -1
            for split in range(left, right):
                candidate = (
                    costs[left][split]
                    + costs[split + 1][right]
                    + dimensions[left] * dimensions[split + 1] * dimensions[right + 1]
                )
                if best_cost is None or candidate < best_cost:
                    best_cost = candidate
                    best_split = split
            if best_cost is None:
                raise AssertionError("a non-singleton interval must have a split")
            costs[left][right] = best_cost
            splits[left][right] = best_split

    def build_plan(left: int, right: int) -> MatrixChainPlan:
        if left == right:
            return left
        split = splits[left][right]
        return (build_plan(left, split), build_plan(split + 1, right))

    return costs[0][matrix_count - 1], build_plan(0, matrix_count - 1)
```

## Example

```python
classic = minimum_cost_matrix_chain_plan((30, 35, 15, 5, 10, 20, 25))
equal_costs = minimum_cost_matrix_chain_plan((2, 2, 2, 2))
singleton = minimum_cost_matrix_chain_plan((4, 7))
large_exact = minimum_cost_matrix_chain_plan(
    (_MAX_MATRIX_DIMENSION, _MAX_MATRIX_DIMENSION, _MAX_MATRIX_DIMENSION)
)

assert (classic, equal_costs, singleton, large_exact) == (
    (15_125, ((0, (1, 2)), ((3, 4), 5))),
    (16, (0, (1, 2))),
    (0, 0),
    (_MAX_MATRIX_DIMENSION**3, (0, 1)),
)
```

## Trade-offs and Limitations

For `m` matrices, the interval search performs `O(m^3)` Python-integer
arithmetic operations and stores `O(m^2)` costs and split indexes. The nested
result uses `O(m)` objects. Costs are exact and can become much larger than a
signed machine integer, so their bit-operation cost grows with the dimensions.

The smallest-split rule is local to every optimal subproblem. It is a
deterministic structural convention, not a claim about lexicographic ordering
of a rendered expression. The recursive reconstruction depth is at most 200
under the admitted limit.

This function plans but does not multiply matrices, estimate peak intermediate
storage, inspect sparsity, choose kernels, reassociate floating-point results,
or compare measured execution time. The cubic planner is inappropriate for
very long chains or for optimizers with richer cost and resource models.

## Related Snippets

<!-- catalog:related:start -->
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Solve a Bounded Tridiagonal Integer System Exactly with Thomas Elimination](solve-a-bounded-tridiagonal-integer-system-exactly-with-thomas-elimination.md)
- [Compute a Distant Linear-Recurrence Term Modulo an Integer by Polynomial Reduction](compute-a-distant-linear-recurrence-term-modulo-an-integer-by-polynomial-reduction.md)
<!-- catalog:related:end -->
