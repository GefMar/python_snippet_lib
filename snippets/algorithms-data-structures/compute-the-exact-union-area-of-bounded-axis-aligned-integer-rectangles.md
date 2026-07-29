---
title: "Compute the Exact Union Area of Bounded Axis-Aligned Integer Rectangles"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md
  - find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md
  - classify-a-bounded-integer-point-under-the-even-odd-polygonal-fill-rule.md
---

# Compute the Exact Union Area of Bounded Axis-Aligned Integer Rectangles

## Idea and Problem

Measure the exact area covered by any of several half-open axis-aligned integer rectangles without double-counting overlaps.

Each rectangle contributes an opening event at its left edge and a closing
event at its right edge. An x-coordinate sweep multiplies the width of every
slab by the currently covered y-length. A segment tree over compressed
y-endpoints maintains that length even when intervals overlap, nest, duplicate,
or merely touch.

All events at one x-coordinate are applied as a group after charging the
preceding slab. Half-open boundaries therefore require no special area
correction.

## When to Use

Use this algorithm for a sparse bounded collection of exact rectangular
regions when overlap must count once and materializing every covered unit cell
would be too expensive. Integer coordinates make the result exact even when
the final area is wider than a machine integer.

Use a geometry library for rotated shapes, floating coordinates, polygon
clipping, topology, or numerical robustness. Use a bitmap or summed-area table
when the complete coordinate grid is already small and dense.

## Implementation

```python
from dataclasses import dataclass

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1
_MAX_RECTANGLE_COUNT = 20_000


@dataclass(frozen=True, slots=True)
class HalfOpenRectangle:
    left: int
    bottom: int
    right: int
    top: int


def exact_rectangle_union_area(
    rectangles: tuple[HalfOpenRectangle, ...],
) -> int:
    """Return the exact area covered by at least one rectangle."""
    if type(rectangles) is not tuple:
        raise TypeError("rectangles must be an exact tuple")
    if len(rectangles) > _MAX_RECTANGLE_COUNT:
        raise ValueError("rectangle count exceeds the supported range")

    events: list[tuple[int, int, int, int]] = []
    y_coordinates: list[int] = []

    for rectangle_index, rectangle in enumerate(rectangles):
        if type(rectangle) is not HalfOpenRectangle:
            raise TypeError(f"rectangles[{rectangle_index}] must be a HalfOpenRectangle")
        for field_name, coordinate in (
            ("left", rectangle.left),
            ("bottom", rectangle.bottom),
            ("right", rectangle.right),
            ("top", rectangle.top),
        ):
            if type(coordinate) is not int:
                raise TypeError(
                    f"rectangles[{rectangle_index}].{field_name} must be an exact integer"
                )
            if not _MIN_INT64 <= coordinate <= _MAX_INT64:
                raise ValueError(
                    f"rectangles[{rectangle_index}].{field_name} is outside the signed 64-bit range"
                )
        if rectangle.left >= rectangle.right:
            raise ValueError("every rectangle must have positive width")
        if rectangle.bottom >= rectangle.top:
            raise ValueError("every rectangle must have positive height")

        events.append(
            (
                rectangle.left,
                1,
                rectangle.bottom,
                rectangle.top,
            )
        )
        events.append(
            (
                rectangle.right,
                -1,
                rectangle.bottom,
                rectangle.top,
            )
        )
        y_coordinates.extend((rectangle.bottom, rectangle.top))

    if not events:
        return 0

    ordered_y = sorted(set(y_coordinates))
    y_index = {coordinate: index for index, coordinate in enumerate(ordered_y)}
    span_count = len(ordered_y) - 1
    coverage_counts = [0] * (4 * span_count)
    covered_lengths = [0] * (4 * span_count)

    def update(
        node: int,
        span_left: int,
        span_right: int,
        query_left: int,
        query_right: int,
        delta: int,
    ) -> None:
        if query_left <= span_left and span_right <= query_right:
            coverage_counts[node] += delta
        else:
            midpoint = (span_left + span_right) // 2
            if query_left < midpoint:
                update(
                    node * 2,
                    span_left,
                    midpoint,
                    query_left,
                    query_right,
                    delta,
                )
            if midpoint < query_right:
                update(
                    node * 2 + 1,
                    midpoint,
                    span_right,
                    query_left,
                    query_right,
                    delta,
                )

        if coverage_counts[node] > 0:
            covered_lengths[node] = ordered_y[span_right] - ordered_y[span_left]
        elif span_right - span_left == 1:
            covered_lengths[node] = 0
        else:
            covered_lengths[node] = covered_lengths[node * 2] + covered_lengths[node * 2 + 1]

    events.sort(key=lambda event: event[0])
    area = 0
    previous_x = events[0][0]
    event_index = 0

    while event_index < len(events):
        current_x = events[event_index][0]
        area += (current_x - previous_x) * covered_lengths[1]

        while event_index < len(events) and events[event_index][0] == current_x:
            _, delta, bottom, top = events[event_index]
            update(
                1,
                0,
                span_count,
                y_index[bottom],
                y_index[top],
                delta,
            )
            event_index += 1
        previous_x = current_x

    return area
```

## Example

```python
overlapping = (
    HalfOpenRectangle(0, 0, 3, 2),
    HalfOpenRectangle(1, 1, 4, 3),
    HalfOpenRectangle(0, 0, 3, 2),
    HalfOpenRectangle(4, 0, 5, 1),
    HalfOpenRectangle(2, 1, 3, 2),
)
disjoint = (
    HalfOpenRectangle(-3, -2, -1, 1),
    HalfOpenRectangle(2, 3, 5, 7),
)
full_coordinate_span = (
    HalfOpenRectangle(
        _MIN_INT64,
        _MIN_INT64,
        _MAX_INT64,
        _MAX_INT64,
    ),
)

assert exact_rectangle_union_area(overlapping) == 11
assert exact_rectangle_union_area(disjoint) == 18
assert exact_rectangle_union_area(()) == 0
assert exact_rectangle_union_area(full_coordinate_span) == (_MAX_INT64 - _MIN_INT64) ** 2
```

## Trade-offs and Limitations

For `n` rectangles, sorting endpoints and events plus `2n` tree updates uses
`O(n log n)` exact integer operations and `O(n)` auxiliary objects. Coordinate
compression preserves the real distance between adjacent y-values; tree leaves
are spans, not assumed unit cells. The tree recursion depth is logarithmic in
at most 40,000 distinct endpoints.

The returned Python integer can exceed signed 64-bit range. Duplicates and
nested rectangles increase event work but not area. Touching half-open edges
have zero overlap, and the result is independent of input order.

Only union area is returned. The function does not compute perimeter, overlap
depth, connected components, contributing rectangles, or a geometric shape.
It accepts neither degenerate nor rotated rectangles and does not define a
tolerance for floating-point coordinates.

## Related Snippets

<!-- catalog:related:start -->
- [Coalesce Bounded Half-Open Integer Intervals Under an Explicit Adjacency Policy](coalesce-bounded-half-open-integer-intervals-under-an-explicit-adjacency-policy.md)
- [Find Peak Coverage Spans Across Bounded Half-Open Integer Intervals](find-peak-coverage-spans-across-bounded-half-open-integer-intervals.md)
- [Classify a Bounded Integer Point Under the Even-Odd Polygonal Fill Rule](classify-a-bounded-integer-point-under-the-even-odd-polygonal-fill-rule.md)
<!-- catalog:related:end -->
