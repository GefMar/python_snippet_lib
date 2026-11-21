---
title: "Overwrite a BytesIO Range Without Moving Its Cursor"
snippet_type: recipe
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - update-native-unsigned-integers-in-place-through-a-cast-memoryview.md
  - ../data-processing/read-csv-from-a-borrowed-binary-stream-without-closing-it.md
---

# Overwrite a BytesIO Range Without Moving Its Cursor

## Idea and Problem

Replace a fixed number of bytes inside a live BytesIO stream without reading, seeking, or copying the full stream.

A writable buffer view targets a slice by absolute offset. Validate the full replacement span first and release the view at scope exit, because an outstanding view prevents the owner from resizing or closing the stream.

## When to Use

Use this to fill a fixed-size slot or repair an in-memory record while preserving a caller-owned stream's read or write position. The stream must be open, and no other actor should mutate it during the operation.

## Implementation

```python
from io import BytesIO


def overwrite_range(stream: BytesIO, offset: int, replacement: bytes) -> None:
    if type(offset) is not int:
        raise TypeError("offset must be an integer")
    if offset < 0:
        raise ValueError("offset must not be negative")

    with stream.getbuffer() as view:
        end = offset + len(replacement)
        if end > len(view):
            raise ValueError("the replacement exceeds the stream size")
        view[offset:end] = replacement
```

## Example

```python
with BytesIO(b"0000 payload") as stream:
    stream.seek(8)
    overwrite_range(stream, 0, b"0007")
    original_position = stream.tell()

    try:
        overwrite_range(stream, 11, b"too long")
    except ValueError:
        rejected_overrun = True
    else:
        rejected_overrun = False

    patched = stream.getvalue()
    stream.seek(0, 2)
    stream.write(b"!")
    appended = stream.getvalue()

assert (original_position, rejected_overrun, patched, appended, stream.closed) == (
    8,
    True,
    b"0007 payload",
    b"0007 payload!",
    True,
)
```

## Trade-offs and Limitations

This changes bytes in place, not the stream length, and does not synchronize concurrent access. The helper does not close the stream or return a view. If another view already exists, it continues to constrain the stream after this helper returns. The stream implementation may copy backing storage internally; this code does not make a full explicit copy during the update.

## Related Snippets

<!-- catalog:related:start -->
- [Update Native Unsigned Integers In Place Through a Cast memoryview](update-native-unsigned-integers-in-place-through-a-cast-memoryview.md)
- [Read CSV from a Borrowed Binary Stream Without Closing It](../data-processing/read-csv-from-a-borrowed-binary-stream-without-closing-it.md)
<!-- catalog:related:end -->
