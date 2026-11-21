---
title: "Measure the Unsigned Angle Between Two Planar Vectors"
snippet_type: recipe
use_cases:
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../machine-learning-statistics/compute-a-bounded-finite-cosine-similarity-with-scaled-norms.md
  - compute-an-exact-area-centroid-for-a-simple-integer-polygon.md
---

# Measure the Unsigned Angle Between Two Planar Vectors

## Idea and Problem

Measure the smaller unsigned angle between two nonzero planar vectors without recovering it from a rounded cosine.

Scale each vector by its largest absolute component, then pass the magnitude of the determinant and the dot product to atan2. Both quantities scale by the same positive factor, so the angle is unchanged by the scaling.

## When to Use

Use this for finite pairs of binary floating-point coordinates, with radians in the closed interval from zero to pi. It is useful when a small angle should not collapse solely because its cosine rounded to one.

## Implementation

```python
from math import atan2, isfinite


def unsigned_angle_2d(a: tuple[float, float], b: tuple[float, float]) -> float:
    if not all(map(isfinite, (*a, *b))):
        raise ValueError("coordinates must be finite")
    scale_a = max(map(abs, a))
    scale_b = max(map(abs, b))
    if min(scale_a, scale_b) == 0:
        raise ValueError("a zero vector has no angle")

    ax, ay = (coordinate / scale_a for coordinate in a)
    bx, by = (coordinate / scale_b for coordinate in b)
    determinant = ax * by - ay * bx
    dot = ax * bx + ay * by
    return atan2(abs(determinant), dot)
```

## Example

```python
from math import isclose, pi

small = unsigned_angle_2d((1.0, 0.0), (1.0, 1e-20))
right_angle = unsigned_angle_2d((1e280, 0.0), (0.0, 1e-280))
opposite = unsigned_angle_2d((1.0, 1.0), (-1.0, -1.0))

assert (isclose(small, 1e-20, rel_tol=1e-15), right_angle, opposite) == (True, pi / 2, pi)
```

## Trade-offs and Limitations

This computes an unsigned angle and does not identify clockwise versus counterclockwise rotation. Scaling reduces overflow risk in the products, but cancellation in the determinant or unrepresentably small component ratios can still lose precision. For exact orientation predicates, use arithmetic appropriate to the coordinate domain.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Bounded Finite Cosine Similarity with Scaled Norms](../machine-learning-statistics/compute-a-bounded-finite-cosine-similarity-with-scaled-norms.md)
- [Compute an Exact Area Centroid for a Simple Integer Polygon](compute-an-exact-area-centroid-for-a-simple-integer-polygon.md)
<!-- catalog:related:end -->
