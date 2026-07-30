---
title: "Compute a Bounded Finite Cosine Similarity with Scaled Norms"
snippet_type: algorithm
use_cases:
  - data-transformation
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - compute-exact-jaccard-similarity-for-bounded-unique-text-feature-sets.md
  - compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md
  - fit-pca-with-numpy-and-report-cumulative-explained-variance.md
---

# Compute a Bounded Finite Cosine Similarity with Scaled Norms

## Idea and Problem

Measure the directional similarity of two bounded finite-float vectors without squaring their original components or multiplying their original magnitudes.

Dividing each vector by its own largest absolute component keeps every scaled
component in `[-1, 1]`. `math.hypot` then computes the scaled norms, while
`math.sumprod` combines the scaled component products before the dot product is
divided by both norms.

## When to Use

Use this function when two already materialized vectors contain comparable
finite binary64 features, have at most 4,096 components, and direction matters
more than magnitude. It is suitable for one-off similarity calculations where
strict input validation and no third-party dependency are useful.

Use an array or machine-learning library for matrices, batches, sparse vectors,
weights, device execution, or many repeated comparisons. Normalize features or
choose a domain-specific metric first when raw feature scales are not directly
comparable.

## Implementation

```python
import math

_MAX_COMPONENTS = 4_096


def bounded_cosine_similarity(
    left: tuple[float, ...],
    right: tuple[float, ...],
) -> float:
    if type(left) is not tuple or type(right) is not tuple:
        raise TypeError("left and right must be exact tuples")
    if len(left) != len(right):
        raise ValueError("left and right must have equal lengths")
    if not 1 <= len(left) <= _MAX_COMPONENTS:
        raise ValueError("vector length is outside the supported range")

    for name, values in (("left", left), ("right", right)):
        for index, value in enumerate(values):
            if type(value) is not float:
                raise TypeError(f"{name}[{index}] must be an exact float")
            if not math.isfinite(value):
                raise ValueError(f"{name}[{index}] must be finite")

    left_scale = max(abs(value) for value in left)
    right_scale = max(abs(value) for value in right)
    if left_scale == 0.0 or right_scale == 0.0:
        raise ValueError("cosine similarity is undefined for an all-zero vector")

    scaled_left = tuple(value / left_scale for value in left)
    scaled_right = tuple(value / right_scale for value in right)
    left_norm = math.hypot(*scaled_left)
    right_norm = math.hypot(*scaled_right)
    similarity = math.sumprod(scaled_left, scaled_right) / (left_norm * right_norm)

    if similarity > 1.0:
        similarity = 1.0
    elif similarity < -1.0:
        similarity = -1.0
    return 0.0 if similarity == 0.0 else similarity


```

## Example

```python
same_direction = bounded_cosine_similarity((3.0, 4.0), (6.0, 8.0))
opposite_direction = bounded_cosine_similarity((3.0, 4.0), (-6.0, -8.0))
orthogonal = bounded_cosine_similarity((3.0, 4.0), (4.0, -3.0))
expected = 32.0 / math.sqrt(14.0 * 77.0)
oblique = bounded_cosine_similarity((1.0, 2.0, 3.0), (4.0, 5.0, 6.0))

try:
    bounded_cosine_similarity((0.0, -0.0), (1.0, 2.0))
except ValueError:
    zero_rejected = True
else:
    zero_rejected = False

assert math.isclose(same_direction, 1.0, rel_tol=0.0, abs_tol=1e-15)
assert math.isclose(opposite_direction, -1.0, rel_tol=0.0, abs_tol=1e-15)
assert orthogonal == 0.0
assert math.copysign(1.0, orthogonal) == 1.0
assert math.isclose(oblique, expected, rel_tol=0.0, abs_tol=1e-15)
assert zero_rejected
```

## Trade-offs and Limitations

Validation, scaling, norm calculation, and the dot product each take `O(n)`
time. The two scaled tuples use `O(n)` additional space. The 4,096-component
limit bounds every pass and allocation.

Scaling prevents overflow from directly squaring or multiplying admitted
original components, but it does not make binary64 arithmetic exact. A
component can underflow to zero when it is extremely small relative to the
largest component in the same vector, changing the represented direction.
`math.hypot`, `math.sumprod`, division, and library implementations can also
produce least-significant cross-platform differences.

Clamping only corrects a final floating-point result just outside the
mathematical interval `[-1, 1]`; it does not repair lost input information.
The function rejects either all-zero vector and provides no missing-value
policy, complex values, weights, feature scaling, sparse representation,
batching, exact arithmetic, or cross-platform bitwise-reproducibility promise.

## Related Snippets

<!-- catalog:related:start -->
- [Compute Exact Jaccard Similarity for Bounded Unique Text-Feature Sets](compute-exact-jaccard-similarity-for-bounded-unique-text-feature-sets.md)
- [Compute a Bounded Numerically Stable Log-Sum-Exp for Finite Floats](compute-a-bounded-numerically-stable-log-sum-exp-for-finite-floats.md)
- [Fit PCA with NumPy and Report Cumulative Explained Variance](fit-pca-with-numpy-and-report-cumulative-explained-variance.md)
<!-- catalog:related:end -->
