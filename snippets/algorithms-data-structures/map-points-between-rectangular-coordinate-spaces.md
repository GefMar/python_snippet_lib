---
title: "Map Points Between Rectangular Coordinate Spaces"
snippet_type: algorithm
use_cases:
  - data-transformation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related: []
---

# Map Points Between Rectangular Coordinate Spaces

## Idea and Problem

Apply an independent affine transformation on each axis to map a finite point from one rectangular coordinate space into another.

Normalizing a point within source bounds and expanding that position into
target bounds separates domain coordinates from output coordinates. Reversed
target bounds naturally invert an axis, while values outside the source bounds
are extrapolated by the same linear rule.

## When to Use

Use this mapping when both spaces are rectangular, axes are independent, and
linear scaling is the intended transform. It fits simple charts, image
coordinates, normalized measurements, and viewport conversion. Use a geometry
or graphics transform when rotation, skew, perspective, clipping, or preserved
aspect ratio is required.

## Implementation

```python
from dataclasses import dataclass
from math import isfinite


@dataclass(frozen=True, slots=True)
class Point2D:
    x: float
    y: float


@dataclass(frozen=True, slots=True)
class RectangularBounds:
    x_start: float
    x_stop: float
    y_start: float
    y_stop: float


def map_between_rectangles(
    point: Point2D,
    *,
    source: RectangularBounds,
    target: RectangularBounds,
) -> Point2D:
    coordinates = (
        point.x,
        point.y,
        source.x_start,
        source.x_stop,
        source.y_start,
        source.y_stop,
        target.x_start,
        target.x_stop,
        target.y_start,
        target.y_stop,
    )
    if not all(isfinite(value) for value in coordinates):
        raise ValueError("coordinates must be finite")

    source_x_span = source.x_stop - source.x_start
    source_y_span = source.y_stop - source.y_start
    target_x_span = target.x_stop - target.x_start
    target_y_span = target.y_stop - target.y_start
    spans = (source_x_span, source_y_span, target_x_span, target_y_span)
    if not all(isfinite(span) and span != 0 for span in spans):
        raise ValueError("coordinate spans must be finite and non-zero")

    x_position = (point.x - source.x_start) / source_x_span
    y_position = (point.y - source.y_start) / source_y_span
    mapped = Point2D(
        target.x_start + x_position * target_x_span,
        target.y_start + y_position * target_y_span,
    )
    if not isfinite(mapped.x) or not isfinite(mapped.y):
        raise ValueError("mapped coordinates must be finite")
    return mapped
```

## Example

```python
source = RectangularBounds(10, 20, -5, 5)
target = RectangularBounds(0, 100, 100, 0)

mapped = tuple(
    map_between_rectangles(point, source=source, target=target)
    for point in (Point2D(10, -5), Point2D(15, 0), Point2D(20, 5))
)
extrapolated = map_between_rectangles(
    Point2D(5, 10),
    source=source,
    target=target,
)

try:
    map_between_rectangles(
        Point2D(0, 0),
        source=RectangularBounds(1, 1, 0, 1),
        target=target,
    )
except ValueError:
    zero_span_rejected = True
else:
    zero_span_rejected = False

try:
    map_between_rectangles(
        Point2D(float("inf"), 0),
        source=source,
        target=target,
    )
except ValueError:
    non_finite_rejected = True
else:
    non_finite_rejected = False

try:
    map_between_rectangles(
        Point2D(0, 0),
        source=RectangularBounds(-1e308, 1e308, -1, 1),
        target=target,
    )
except ValueError:
    overflowing_span_rejected = True
else:
    overflowing_span_rejected = False

assert (
    mapped,
    extrapolated,
    zero_span_rejected,
    non_finite_rejected,
    overflowing_span_rejected,
) == (
    (Point2D(0.0, 100.0), Point2D(50.0, 50.0), Point2D(100.0, 0.0)),
    Point2D(-50.0, -50.0),
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Floating-point arithmetic can introduce small rounding differences, so callers
should use tolerance-based comparison when inputs are not exactly representable.
Even finite endpoints can produce a non-finite span or result at extreme
magnitudes; the implementation rejects those cases.
The function deliberately extrapolates instead of clipping and returns floats
instead of rounded pixels. It scales axes independently and can distort aspect
ratio. Degenerate bounds are rejected even when collapsing an axis might be a
meaningful domain-specific operation.

## Related Snippets

<!-- catalog:related:start -->
No related snippets.
<!-- catalog:related:end -->
