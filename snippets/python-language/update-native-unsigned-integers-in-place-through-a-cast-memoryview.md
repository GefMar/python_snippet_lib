---
title: "Update Native Unsigned Integers In Place Through a Cast memoryview"
snippet_type: recipe
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - ../storage-databases/fill-a-preallocated-bytearray-exactly-with-os-readinto.md
  - make-a-defensive-copy-at-a-mutable-input-boundary.md
---

# Update Native Unsigned Integers In Place Through a Cast memoryview

## Idea and Problem

Update selected native unsigned-integer lanes in one mutable byte buffer without copying the complete payload.

A writable `memoryview` can reinterpret the byte-oriented exporter with the
native `"I"` format. Validating every requested lane and staging its new value
before assignment prevents a bad index, duplicate index, underflow, or overflow
from leaving a partially updated buffer. The cast shares the original storage,
so successful assignments are immediately visible through the `bytearray`.

## When to Use

Use this recipe for one caller-owned `bytearray` that already contains
host-native unsigned integers and needs a bounded set of in-place adjustments.
It fits process-local buffers exchanged with native code that uses the same ABI,
or temporary numeric workspaces whose representation never crosses a machine
boundary.

Use `struct` with an explicit byte order for files, wire protocols, shared
formats, or data exchanged between unlike systems. Prefer an array or a numeric
library when typed values, vectorized arithmetic, or multidimensional operations
are the primary abstraction.

## Implementation

```python
from array import array
from struct import calcsize

_MAX_BUFFER_BYTES = 1024 * 1024
_MAX_LANES_PER_UPDATE = 4_096
_NATIVE_UINT_BYTES = calcsize("@I")
_NATIVE_UINT_MAX = (1 << (8 * _NATIVE_UINT_BYTES)) - 1


def update_native_unsigned_integers(
    buffer: bytearray,
    indices: tuple[int, ...],
    *,
    delta: int,
) -> bytearray:
    if type(buffer) is not bytearray:
        raise TypeError("buffer must be an exact bytearray")
    if not _NATIVE_UINT_BYTES <= len(buffer) <= _MAX_BUFFER_BYTES:
        raise ValueError("buffer size is outside the supported range")
    if len(buffer) % _NATIVE_UINT_BYTES:
        raise ValueError("buffer must contain complete native unsigned integers")
    if type(indices) is not tuple:
        raise TypeError("indices must be an exact tuple")
    if not 1 <= len(indices) <= _MAX_LANES_PER_UPDATE:
        raise ValueError("index count is outside the supported range")
    if type(delta) is not int:
        raise TypeError("delta must be an exact integer")
    if not -_NATIVE_UINT_MAX <= delta <= _NATIVE_UINT_MAX:
        raise ValueError("delta is outside the native unsigned-integer range")

    byte_view = memoryview(buffer)
    lanes = byte_view.cast("I")
    try:
        seen: set[int] = set()
        staged: list[tuple[int, int]] = []
        for position, index in enumerate(indices):
            if type(index) is not int:
                raise TypeError(f"indices[{position}] must be an exact integer")
            if not 0 <= index < len(lanes):
                raise IndexError(f"indices[{position}] is outside the buffer")
            if index in seen:
                raise ValueError("indices must be unique")
            seen.add(index)

            updated = lanes[index] + delta
            if not 0 <= updated <= _NATIVE_UINT_MAX:
                raise OverflowError("updated value does not fit native unsigned int")
            staged.append((index, updated))

        for index, updated in staged:
            lanes[index] = updated
    finally:
        lanes.release()
        byte_view.release()

    return buffer
```

## Example

```python
def native_uint_buffer(values: tuple[int, ...]) -> bytearray:
    source = array("I", values)
    return bytearray(source.tobytes())


def read_native_uints(buffer: bytearray) -> tuple[int, ...]:
    observed = array("I")
    observed.frombytes(buffer)
    return tuple(observed)


buffer = native_uint_buffer((1, 2, 3))
returned = update_native_unsigned_integers(buffer, (0, 2), delta=7)

maximum = (1 << (8 * _NATIVE_UINT_BYTES)) - 1
overflow_buffer = native_uint_buffer((maximum, 4))
before_overflow = bytes(overflow_buffer)
try:
    update_native_unsigned_integers(overflow_buffer, (1, 0), delta=1)
except OverflowError:
    overflow_rejected = True
else:
    overflow_rejected = False

assert (
    returned is buffer,
    read_native_uints(buffer),
    overflow_rejected,
    bytes(overflow_buffer) == before_overflow,
) == (True, (8, 2, 10), True, True)
```

## Trade-offs and Limitations

The `"I"` cast uses the interpreter's native byte order, C unsigned-int size,
and representation. It is deliberately not a promise of 32-bit lanes and is
unsuitable for persistent or interoperable bytes. The recipe makes no portable
alignment guarantee for unrelated foreign exporters; it accepts only an exact
`bytearray`, whose contiguous byte buffer is supported by this local contract.

The cast does not copy the payload, but validation allocates a set and a staged
list proportional to the at most 4,096 selected lanes. Staging gives
all-or-nothing behavior for validation failures in this single-threaded helper;
it does not provide transactionality for interpreter failure or concurrent
mutation. Other views observe successful writes immediately.

Active views prevent resizing the `bytearray`. Both views are released in a
`finally` block, including after validation or assignment errors. The caller
still owns synchronization, buffer lifetime, and the meaning of each lane.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](../configuration-serialization/encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Fill a Preallocated Bytearray Exactly with os.readinto](../storage-databases/fill-a-preallocated-bytearray-exactly-with-os-readinto.md)
- [Make a Defensive Copy at a Mutable Input Boundary](make-a-defensive-copy-at-a-mutable-input-boundary.md)
<!-- catalog:related:end -->
