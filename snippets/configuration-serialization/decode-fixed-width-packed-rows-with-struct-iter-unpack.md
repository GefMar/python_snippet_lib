---
title: "Decode Fixed-Width Packed Rows with a struct Iterator"
snippet_type: recipe
use_cases:
  - parsing
  - interoperability
tested_python:
  - "3.14"
dependencies: []
related:
  - encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md
  - ../python-language/read-a-fixed-width-byte-record-with-the-index-protocol.md
---

# Decode Fixed-Width Packed Rows with a struct Iterator

## Idea and Problem

Iterate complete fixed-width binary records without slicing the byte buffer for each row.

Compile an explicit wire layout once, reject a partial trailing record, and check the record count before returning a lazy unpacking iterator.

## When to Use

Use this for a finite in-memory block with one repeated layout. This example reads an unsigned 16-bit channel and a signed 32-bit reading in big-endian order.

## Implementation

```python
from collections.abc import Iterator
from struct import Struct


ROW = Struct(">Hi")


def unpack_readings(payload: bytes, max_rows: int) -> Iterator[tuple[int, int]]:
    if max_rows < 0:
        raise ValueError("the record limit must not be negative")

    rows, extra = divmod(len(payload), ROW.size)
    if extra or rows > max_rows:
        raise ValueError("the block is misaligned or exceeds the record limit")

    return ROW.iter_unpack(payload)
```

## Example

```python
payload = bytes.fromhex("0002 ffffffff 0009 00000032")
actual = tuple(unpack_readings(payload, 2))
rejected = []
for data, limit in ((payload[:-1], 2), (payload, 1)):
    try:
        unpack_readings(data, limit)
    except ValueError as error:
        rejected.append(type(error))

assert (actual, tuple(unpack_readings(b"", 0)), rejected) == (
    ((2, -1), (9, 50)),
    (),
    [ValueError, ValueError],
)
```

## Trade-offs and Limitations

The iterator retains the input bytes until exhausted or discarded. The record limit does not limit bytes read earlier by another component. This handles one fixed shape only and does not validate domain-specific channel or reading ranges.

## Related Snippets

<!-- catalog:related:start -->
- [Encode and Decode One Exact-Size Big-Endian Binary Record with struct.Struct](encode-and-decode-one-exact-size-big-endian-binary-record-with-struct.md)
- [Read a Fixed-Width Byte Record with the Index Protocol](../python-language/read-a-fixed-width-byte-record-with-the-index-protocol.md)
<!-- catalog:related:end -->
