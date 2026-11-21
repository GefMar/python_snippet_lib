---
title: "Compute Exact Signed Squared Cosine from Sparse Integer Vectors"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-bounded-finite-cosine-similarity-with-scaled-norms.md
  - compute-exact-squared-pearson-correlation-with-direction.md
---

# Compute Exact Signed Squared Cosine from Sparse Integer Vectors

## Idea and Problem

Report the sign and exact squared magnitude of cosine similarity without expanding sparse feature vectors.

Missing keys represent zero. Only shared keys contribute to the dot product,
while each squared norm uses all values in its own vector. Keep the sign of
the dot product and divide its square by the product of the squared norms.
This leaves the magnitude as an exact fraction and avoids square roots.

## When to Use

Use this for sparse integer features that share one set of coordinate names.
Pass stable mappings with string keys and signed integer values. Missing keys
and explicit zero entries have the same meaning. A zero vector makes cosine
undefined and returns `None`. Two nonzero vectors with disjoint keys have a
defined cosine of zero.

## Implementation

```python
from collections.abc import Mapping
from dataclasses import dataclass
from fractions import Fraction


@dataclass(frozen=True, slots=True)
class SparseCosine:
    sign: int
    squared_magnitude: Fraction


def _squared_norm(vector: Mapping[str, int]) -> int:
    total = 0
    for key, value in vector.items():
        if type(key) is not str or type(value) is not int:
            raise TypeError("vectors require string keys and integer values")
        total += value * value
    return total


def exact_sparse_cosine(left: Mapping[str, int], right: Mapping[str, int]) -> SparseCosine | None:
    norm_product = _squared_norm(left) * _squared_norm(right)
    if not norm_product:
        return None

    if len(left) > len(right):
        left, right = right, left
    dot = sum(value * right.get(key, 0) for key, value in left.items())
    sign = (dot > 0) - (dot < 0)
    return SparseCosine(sign, Fraction(dot * dot, norm_product))
```

## Example

```python
left = {"alpha": 2, "beta": -1}

assert (
    exact_sparse_cosine(left, {"alpha": 1, "gamma": 2}),
    exact_sparse_cosine(left, {"alpha": -1, "gamma": 2}),
    exact_sparse_cosine(left, {"gamma": 3}),
    exact_sparse_cosine(left, {"alpha": 0}),
) == (
    SparseCosine(1, Fraction(4, 25)),
    SparseCosine(-1, Fraction(4, 25)),
    SparseCosine(0, Fraction(0)),
    None,
)
```

## Trade-offs and Limitations

The calculation scans both inputs for their norms and looks up only the
smaller mapping's keys for the dot product. Large integer products and fraction
reduction still have a cost. The function keeps only scalar accumulators.

The result contains the sign and squared magnitude as separate fields. For
a displayed cosine, take the square root of the magnitude and apply the
sign. That square root usually requires approximation. Cosine compares
vector directions and uses the original feature values without mean centering.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Bounded Finite Cosine Similarity with Scaled Norms](compute-a-bounded-finite-cosine-similarity-with-scaled-norms.md)
- [Compute Exact Squared Pearson Correlation with Direction](compute-exact-squared-pearson-correlation-with-direction.md)
<!-- catalog:related:end -->
