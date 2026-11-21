---
title: "Decode Fixed-Endian Signed 16-Bit Samples into a Native Array"
snippet_type: recipe
use_cases:
  - data-transformation
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/update-native-unsigned-integers-in-place-through-a-cast-memoryview.md
  - encode-and-decode-a-bounded-big-endian-complex-vector-with-struct.md
---

# Decode Fixed-Endian Signed 16-Bit Samples into a Native Array

## Idea and Problem

Convert an entire signed 16-bit sample block into a compact native array even if the wire byte order differs.

Import bytes into an array of signed shorts, then swap byte order in place only when it differs from the host. Check the native item size before treating the array as signed 16-bit storage.

## When to Use

Use this for an already framed sequence of signed 16-bit integers with known byte order. A native mutable array is useful if later operations consume numeric values through the buffer protocol rather than a tuple of Python integers.

## Implementation

```python
import sys
from array import array
from typing import Literal


def decode_samples(data: bytes, byte_order: Literal["big", "little"]) -> array:
    if byte_order not in {"big", "little"}:
        raise ValueError("byte order must be big or little")
    if len(data) % 2:
        raise ValueError("partial sample at end of block")

    samples = array("h")
    if samples.itemsize != 2:
        raise RuntimeError("this host does not have two-byte signed shorts")
    samples.frombytes(data)
    if byte_order != sys.byteorder:
        samples.byteswap()
    return samples
```

## Example

```python
big = bytes.fromhex("8000 fffe 0000 1234 7fff")
little = bytes.fromhex("0080 feff 0000 3412 ff7f")
expected = [-32768, -2, 0, 4660, 32767]

try:
    decode_samples(big[:-1], "big")
except ValueError:
    rejected_partial = True
else:
    rejected_partial = False

assert (
    list(decode_samples(big, "big")),
    list(decode_samples(little, "little")),
    list(decode_samples(b"", "big")),
    rejected_partial,
) == (expected, expected, [], True)
```

## Trade-offs and Limitations

The conversion copies the full block, and the returned array is mutable. Supply an input size bound at the framing boundary. Serializing the result with `tobytes()` uses native byte order, not the original order. For files containing mixed fields or if the native short size is unsupported, use a fixed binary layout with `struct`.

## Related Snippets

<!-- catalog:related:start -->
- [Update Native Unsigned Integers In Place Through a Cast memoryview](../python-language/update-native-unsigned-integers-in-place-through-a-cast-memoryview.md)
- [Encode and Decode a Bounded Big-Endian Complex Vector with struct](encode-and-decode-a-bounded-big-endian-complex-vector-with-struct.md)
<!-- catalog:related:end -->
