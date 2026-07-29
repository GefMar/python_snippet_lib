---
title: "Classify the Exact Intersection of Two Closed Integer Line Segments"
snippet_type: algorithm
use_cases:
  - data-transformation
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - compute-a-canonical-convex-hull-for-bounded-integer-points.md
  - compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md
  - map-points-between-rectangular-coordinate-spaces.md
---

# Classify the Exact Intersection of Two Closed Integer Line Segments

## Idea and Problem

Classify two closed integer-coordinate line segments as disjoint, meeting at one exact point, or sharing one positive-length collinear segment.

Integer cross products distinguish parallel, collinear, and crossing supporting
lines without a floating-point tolerance. For non-parallel lines, exact
`Fraction` parameters decide whether their unique crossing lies inside both
closed segments and preserve rational coordinates such as one third.

Collinear segments need a different result shape because their intersection
can have positive length. Ordering endpoints lexicographically gives one
canonical interval along any straight line, including vertical lines, so the
answer does not depend on segment order or endpoint direction.

## When to Use

Use this function when two fully materialized 2D segments have signed 64-bit
integer coordinates and a caller must distinguish no intersection, one exact
point, and a collinear overlap. It fits deterministic geometry checks, fixture
comparisons, validation, and preprocessing where boundary contact counts as an
intersection.

Both segments are closed and may degenerate to a single point. The immutable
result contains a `kind` tag and zero, one, or two rational points: `none` has
no points, `point` has one, and `overlap` has two distinct lexicographically
ordered endpoints. Use a geometry library when floating coordinates,
coordinate reference systems, tolerances, or larger geometric objects matter.

## Implementation

```python
from dataclasses import dataclass
from fractions import Fraction
from typing import Literal, TypeAlias

_MIN_INT64 = -(1 << 63)
_MAX_INT64 = (1 << 63) - 1

IntegerPoint: TypeAlias = tuple[int, int]
IntegerSegment: TypeAlias = tuple[IntegerPoint, IntegerPoint]
RationalPoint: TypeAlias = tuple[Fraction, Fraction]


@dataclass(frozen=True, slots=True)
class SegmentIntersection:
    kind: Literal["none", "point", "overlap"]
    points: tuple[RationalPoint, ...]


def _validated_integer_segment(value: object, *, field: str) -> IntegerSegment:
    if type(value) is not tuple:
        raise TypeError(f"{field} must be an exact tuple")
    if len(value) != 2:
        raise ValueError(f"{field} must contain exactly two endpoints")

    for point_index, point in enumerate(value):
        if type(point) is not tuple:
            raise TypeError(f"{field}[{point_index}] must be an exact tuple")
        if len(point) != 2:
            raise ValueError(f"{field}[{point_index}] must contain two coordinates")
        for coordinate_index, coordinate in enumerate(point):
            if type(coordinate) is not int:
                raise TypeError(
                    f"{field}[{point_index}][{coordinate_index}] must be an exact integer"
                )
            if not _MIN_INT64 <= coordinate <= _MAX_INT64:
                raise ValueError(
                    f"{field}[{point_index}][{coordinate_index}] is outside signed 64-bit range"
                )
    return value


def _difference(first: IntegerPoint, second: IntegerPoint) -> IntegerPoint:
    return second[0] - first[0], second[1] - first[1]


def _cross(first: IntegerPoint, second: IntegerPoint) -> int:
    return first[0] * second[1] - first[1] * second[0]


def _rational_point(point: IntegerPoint) -> RationalPoint:
    return Fraction(point[0]), Fraction(point[1])


def classify_segment_intersection(
    first: IntegerSegment,
    second: IntegerSegment,
) -> SegmentIntersection:
    """Return the exact closed-segment intersection under one tagged shape."""
    first_start, first_stop = _validated_integer_segment(first, field="first")
    second_start, second_stop = _validated_integer_segment(second, field="second")

    first_direction = _difference(first_start, first_stop)
    second_direction = _difference(second_start, second_stop)
    start_offset = _difference(first_start, second_start)
    denominator = _cross(first_direction, second_direction)

    if denominator != 0:
        first_parameter = Fraction(
            _cross(start_offset, second_direction),
            denominator,
        )
        second_parameter = Fraction(
            _cross(start_offset, first_direction),
            denominator,
        )
        if not (0 <= first_parameter <= 1 and 0 <= second_parameter <= 1):
            return SegmentIntersection("none", ())

        point = (
            Fraction(first_start[0]) + first_parameter * first_direction[0],
            Fraction(first_start[1]) + first_parameter * first_direction[1],
        )
        return SegmentIntersection("point", (point,))

    if (
        _cross(start_offset, first_direction) != 0
        or _cross(
            start_offset,
            second_direction,
        )
        != 0
    ):
        return SegmentIntersection("none", ())

    first_low, first_high = sorted((first_start, first_stop))
    second_low, second_high = sorted((second_start, second_stop))
    overlap_start = max(first_low, second_low)
    overlap_stop = min(first_high, second_high)
    if overlap_start > overlap_stop:
        return SegmentIntersection("none", ())

    rational_start = _rational_point(overlap_start)
    if overlap_start == overlap_stop:
        return SegmentIntersection("point", (rational_start,))
    return SegmentIntersection(
        "overlap",
        (rational_start, _rational_point(overlap_stop)),
    )
```

## Example

```python
def orientation(
    first: IntegerPoint,
    second: IntegerPoint,
    third: IntegerPoint,
) -> int:
    return (second[0] - first[0]) * (third[1] - first[1]) - (second[1] - first[1]) * (
        third[0] - first[0]
    )


def point_is_on_segment(
    point: IntegerPoint,
    segment: IntegerSegment,
) -> bool:
    start, stop = segment
    return (
        orientation(start, stop, point) == 0
        and min(start[0], stop[0]) <= point[0] <= max(start[0], stop[0])
        and min(start[1], stop[1]) <= point[1] <= max(start[1], stop[1])
    )


def orientation_reference(
    first: IntegerSegment,
    second: IntegerSegment,
) -> SegmentIntersection:
    first_start, first_stop = first
    second_start, second_stop = second
    orientations = (
        orientation(first_start, first_stop, second_start),
        orientation(first_start, first_stop, second_stop),
        orientation(second_start, second_stop, first_start),
        orientation(second_start, second_stop, first_stop),
    )

    if orientations == (0, 0, 0, 0):
        shared_endpoints = sorted(
            {
                point
                for point in (first_start, first_stop, second_start, second_stop)
                if point_is_on_segment(point, first) and point_is_on_segment(point, second)
            }
        )
        if not shared_endpoints:
            return SegmentIntersection("none", ())
        rational_start = _rational_point(shared_endpoints[0])
        if len(shared_endpoints) == 1:
            return SegmentIntersection("point", (rational_start,))
        return SegmentIntersection(
            "overlap",
            (rational_start, _rational_point(shared_endpoints[-1])),
        )

    first_straddles = (
        orientations[0] == 0
        or orientations[1] == 0
        or (orientations[0] < 0 < orientations[1] or orientations[1] < 0 < orientations[0])
    )
    second_straddles = (
        orientations[2] == 0
        or orientations[3] == 0
        or (orientations[2] < 0 < orientations[3] or orientations[3] < 0 < orientations[2])
    )
    if not first_straddles or not second_straddles:
        return SegmentIntersection("none", ())

    first_a = first_start[1] - first_stop[1]
    first_b = first_stop[0] - first_start[0]
    first_c = first_start[0] * first_stop[1] - first_stop[0] * first_start[1]
    second_a = second_start[1] - second_stop[1]
    second_b = second_stop[0] - second_start[0]
    second_c = second_start[0] * second_stop[1] - second_stop[0] * second_start[1]
    determinant = first_a * second_b - second_a * first_b
    point = (
        Fraction(first_b * second_c - second_b * first_c, determinant),
        Fraction(first_c * second_a - second_c * first_a, determinant),
    )
    return SegmentIntersection("point", (point,))


def exercise_small_segment_pairs() -> int:
    from itertools import product

    points = tuple(product((-1, 0, 1), repeat=2))
    segments = tuple(product(points, repeat=2))
    checked = 0
    for first in segments:
        reversed_first = first[::-1]
        for second in segments:
            expected = orientation_reference(first, second)
            reversed_second = second[::-1]
            assert classify_segment_intersection(first, second) == expected
            assert classify_segment_intersection(second, first) == expected
            assert classify_segment_intersection(reversed_first, second) == expected
            assert classify_segment_intersection(first, reversed_second) == expected
            checked += 1
    return checked


proper_crossing = classify_segment_intersection(
    ((0, 0), (3, 3)),
    ((0, 2), (3, 0)),
)
negative_slope_overlap = classify_segment_intersection(
    ((4, 0), (0, 4)),
    ((3, 1), (5, -1)),
)
point_on_segment = classify_segment_intersection(
    ((1, 1), (1, 1)),
    ((0, 0), (2, 2)),
)
extreme_crossing = classify_segment_intersection(
    ((_MIN_INT64, 0), (_MAX_INT64, 0)),
    ((0, _MIN_INT64), (0, _MAX_INT64)),
)

invalid_inputs: tuple[tuple[object, object, type[Exception]], ...] = (
    ([((0, 0), (1, 1))], ((0, 1), (1, 0)), TypeError),
    (((0, 0),), ((0, 1), (1, 0)), ValueError),
    (((0, 0), [1, 1]), ((0, 1), (1, 0)), TypeError),
    (((0, 0), (1,)), ((0, 1), (1, 0)), ValueError),
    (((0, 0), (1, True)), ((0, 1), (1, 0)), TypeError),
    (((0, 0), (_MAX_INT64 + 1, 1)), ((0, 1), (1, 0)), ValueError),
)
validation_rejections = []
for invalid_first, valid_second, expected_exception in invalid_inputs:
    try:
        classify_segment_intersection(invalid_first, valid_second)
    except expected_exception:
        validation_rejections.append(True)
    else:
        validation_rejections.append(False)

assert (
    exercise_small_segment_pairs(),
    proper_crossing,
    negative_slope_overlap,
    point_on_segment,
    classify_segment_intersection(((0, 0), (1, 0)), ((2, 0), (3, 0))),
    extreme_crossing,
    all(validation_rejections),
) == (
    6_561,
    SegmentIntersection("point", ((Fraction(6, 5), Fraction(6, 5)),)),
    SegmentIntersection(
        "overlap",
        (
            (Fraction(3), Fraction(1)),
            (Fraction(4), Fraction(0)),
        ),
    ),
    SegmentIntersection("point", ((Fraction(1), Fraction(1)),)),
    SegmentIntersection("none", ()),
    SegmentIntersection("point", ((Fraction(0), Fraction(0)),)),
    True,
)
```

## Trade-offs and Limitations

The algorithm performs `O(1)` exact arithmetic operations and stores `O(1)`
points because it always handles exactly two segments. This operation count is
not a constant machine-time claim: integer products, `Fraction` reduction, and
greatest-common-divisor work depend on operand bit lengths. Signed 64-bit input
bounds keep that cost finite, while Python integers prevent intermediate
cross-product overflow.

The result is canonical for exact Cartesian coordinates. Reversing either
segment or swapping the two segments cannot change the tag or returned points.
Closed endpoints count as intersections, point segments are accepted, and a
positive-length collinear intersection is distinct from a single shared point.

This function does not apply floating-point tolerances, normalize coordinate
reference systems, intersect rays or infinite lines, classify more than two
segments, process curves or polygons, or accelerate a many-segment sweep. For
non-integer measurements or broader geometry operations, use a library whose
precision model and topology rules match the application.

## Related Snippets

<!-- catalog:related:start -->
- [Compute a Canonical Convex Hull for Bounded Integer Points](compute-a-canonical-convex-hull-for-bounded-integer-points.md)
- [Compute an Exact Integer-Matrix Determinant with Bareiss Elimination](compute-an-exact-integer-matrix-determinant-with-bareiss-elimination.md)
- [Map Points Between Rectangular Coordinate Spaces](map-points-between-rectangular-coordinate-spaces.md)
<!-- catalog:related:end -->
