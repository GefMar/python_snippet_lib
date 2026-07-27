---
title: "Extract a Finite 2D Bounding Box from Bounded WKB"
snippet_type: integration
use_cases:
  - data-transformation
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies:
  - name: shapely
    version: "2.1.2"
related:
  - collect-expected-parse-failures-without-stopping-a-batch.md
  - ../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md
  - ../algorithms-data-structures/map-points-between-rectangular-coordinate-spaces.md
---

# Extract a Finite 2D Bounding Box from Bounded WKB

## Idea and Problem

Decode one size-capped WKB byte string and return an immutable finite two-dimensional bounding box, or no box for an empty geometry.

Shapely performs the binary geometry parsing, including byte-order handling.
The wrapper adds an exact input-type boundary, a byte cap, explicit malformed
input behavior, and a scan that rejects every non-finite X or Y coordinate
before constructing the result.

## When to Use

Use this integration after a transport or record parser has isolated exactly
one binary WKB value in memory. The payload must be an exact non-empty `bytes`
object no larger than the fixed cap. Both little-endian and big-endian WKB are
accepted when Shapely 2.1.2 supports the encoded geometry type.

The result describes raw X/Y coordinate extents only. It is suitable when the
caller already knows the coordinate reference system and needs a compact
spatial envelope. It does not decode hexadecimal text, merge row fields,
infer or transform a CRS, or retry with another geometry parser.

## Implementation

```python
import math
from dataclasses import dataclass

from shapely import from_wkb, get_coordinates
from shapely.errors import GEOSException


_MAX_WKB_BYTES = 1_048_576


@dataclass(frozen=True, slots=True)
class BoundingBox2D:
    min_x: float
    min_y: float
    max_x: float
    max_y: float


def _finite_xy(coordinate: object) -> tuple[float, float]:
    x = float(coordinate[0])
    y = float(coordinate[1])
    if not math.isfinite(x) or not math.isfinite(y):
        raise ValueError("WKB contains a non-finite X or Y coordinate")
    return x, y


def extract_finite_2d_bounding_box(
    payload: bytes,
) -> BoundingBox2D | None:
    if type(payload) is not bytes:
        raise TypeError("payload must be an exact bytes object")
    if not payload:
        raise ValueError("payload must not be empty")
    if len(payload) > _MAX_WKB_BYTES:
        raise ValueError("payload exceeds the supported WKB byte limit")

    try:
        geometry = from_wkb(payload, on_invalid="raise")
        if geometry.is_empty:
            return None
        coordinates = get_coordinates(
            geometry,
            include_z=False,
            include_m=False,
        )
    except GEOSException as error:
        raise ValueError("payload is not valid WKB") from error

    iterator = iter(coordinates)
    try:
        first = next(iterator)
    except StopIteration as error:
        raise ValueError("non-empty WKB must contain coordinates") from error

    min_x, min_y = _finite_xy(first)
    max_x, max_y = min_x, min_y
    for coordinate in iterator:
        x, y = _finite_xy(coordinate)
        min_x = min(min_x, x)
        min_y = min(min_y, y)
        max_x = max(max_x, x)
        max_y = max(max_y, y)
    return BoundingBox2D(min_x, min_y, max_x, max_y)
```

## Example

```python
from shapely import GeometryCollection, LineString, to_wkb


payload = to_wkb(
    LineString([(4.5, -2.0), (1.0, 3.0), (7.0, 0.5)]),
    byte_order=1,
)

assert extract_finite_2d_bounding_box(payload) == BoundingBox2D(
    min_x=1.0,
    min_y=-2.0,
    max_x=7.0,
    max_y=3.0,
)
assert extract_finite_2d_bounding_box(to_wkb(GeometryCollection())) is None

try:
    extract_finite_2d_bounding_box(b"not WKB")
except ValueError:
    malformed_rejected = True
else:
    malformed_rejected = False

assert malformed_rejected
```

## Trade-offs and Limitations

The scan is linear in the number of coordinates and Shapely materializes a
coordinate array, although the input byte cap bounds that work. Empty
geometries return `None`; malformed binary encodings and non-finite X/Y values
raise `ValueError`. Topological invalidity is a different concern and is not
checked. Z and M ordinates are intentionally omitted from both validation and
the returned 2D envelope.

WKB carries coordinates, not a universally actionable CRS contract. Shapely
may retain an EWKB SRID, but this function neither validates it nor transforms
coordinates; the numbers remain in the source coordinate units.

Shapely 2.1.2 accepts a valid WKB geometry followed by trailing bytes and uses
the valid prefix. This wrapper deliberately relies on the public parser rather
than implementing a second WKB grammar, so it inherits that behavior. The
caller must therefore establish message framing before calling the function;
the byte cap is not proof that the entire payload was consumed. Mutable byte
buffers, hexadecimal strings, and `bytes` subclasses are rejected.

## Related Snippets

<!-- catalog:related:start -->
- [Collect Expected Parse Failures Without Stopping a Batch](collect-expected-parse-failures-without-stopping-a-batch.md)
- [Read a Bounded Range from Non-Overlapping Byte Segments](../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md)
- [Map Points Between Rectangular Coordinate Spaces](../algorithms-data-structures/map-points-between-rectangular-coordinate-spaces.md)
<!-- catalog:related:end -->
