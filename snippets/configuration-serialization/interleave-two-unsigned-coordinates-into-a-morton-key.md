---
title: "Interleave Two Unsigned Coordinates into a Morton Key"
snippet_type: algorithm
use_cases:
  - data-transformation
  - serialization
tested_python:
  - "3.14"
dependencies: []
related:
  - pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md
  - encode-a-sortable-eight-byte-key-for-each-finite-binary64-float.md
---

# Interleave Two Unsigned Coordinates into a Morton Key

## Idea and Problem

Combine two unsigned grid coordinates into one reversible integer key that sorts cells in Morton order.

Number bits from zero at the least significant end. Move bit `i` of `x`
to bit `2*i` of the key, and bit `i` of `y` to bit `2*i + 1`. Sorting by
that integer visits shared high-bit grid quadrants together before sorting
their subdivisions. Reading the even and odd positions recovers `x` and `y`.

## When to Use

Use this to sort tiles or bucket grid cells by nested quadrants. Both
coordinates must be integers in `[0, 2**width)` with `1 <= width <= 32`.
The key is an unsigned integer in `[0, 2**(2*width))`. Use the same width
and axis convention throughout a dataset; the integer does not store either.

## Implementation

```python
def _check_width(width: int) -> None:
    if type(width) is not int:
        raise TypeError("width must be an integer")
    if not 1 <= width <= 32:
        raise ValueError("width must be in 1..32")


def morton_key(x: int, y: int, *, width: int) -> int:
    _check_width(width)
    if type(x) is not int or type(y) is not int:
        raise TypeError("coordinates must be integers")
    limit = 1 << width
    if not (0 <= x < limit and 0 <= y < limit):
        raise ValueError("coordinates do not fit width")

    key = 0
    for i in range(width):
        key |= ((x >> i) & 1) << (2 * i)
        key |= ((y >> i) & 1) << (2 * i + 1)
    return key


def morton_coordinates(key: int, *, width: int) -> tuple[int, int]:
    _check_width(width)
    if type(key) is not int:
        raise TypeError("key must be an integer")
    if not 0 <= key < 1 << (2 * width):
        raise ValueError("key does not fit twice the width")

    x = y = 0
    for i in range(width):
        x |= ((key >> (2 * i)) & 1) << i
        y |= ((key >> (2 * i + 1)) & 1) << i
    return x, y
```

## Example

```python
tiles = [(0, 1), (1, 1), (1, 0), (0, 0)]
assert sorted(tiles, key=lambda tile: morton_key(*tile, width=1)) == [
    (0, 0),
    (1, 0),
    (0, 1),
    (1, 1),
]
assert morton_key(5, 3, width=3) == 0b011011
assert morton_coordinates(0b011011, width=3) == (5, 3)
```

## Trade-offs and Limitations

Encoding and decoding each take one loop over the configured width. Width
32 allows keys larger than the maximum signed 64-bit integer, so preserve
unsigned order if you store the keys outside Python.

Nearby cells can be far apart in key order across quadrant boundaries. A
general query rectangle is not one contiguous key interval; decompose it
into appropriate ranges or filter candidates using their coordinates.

## Related Snippets

<!-- catalog:related:start -->
- [Pack and Unpack a Bounded Boolean Tuple with Explicit Bit Length](pack-and-unpack-a-bounded-boolean-tuple-with-explicit-bit-length.md)
- [Encode a Sortable Eight-Byte Key for Each Finite Binary64 Float](encode-a-sortable-eight-byte-key-for-each-finite-binary64-float.md)
<!-- catalog:related:end -->
