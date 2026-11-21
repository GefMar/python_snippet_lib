---
title: "Read a Fixed-Width Byte Record with the Index Protocol"
snippet_type: recipe
use_cases:
  - validation
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-an-immutable-slice-aware-sequence.md
  - ../data-processing/decode-bounded-fixed-width-ascii-records-across-arbitrary-byte-chunks.md
---

# Read a Fixed-Width Byte Record with the Index Protocol

## Idea and Problem

Read one fixed-width record by an integer index without coercing a float to an integer.

Apply the same index protocol used by Python sequence operations, then enforce this record format's own boundary and type rules. A record-number object can expose an integer index without being an integer subclass.

## When to Use

Use this for an aligned byte block with a fixed record width and zero-based, nonnegative record numbers. Accept objects that deliberately provide `__index__` but exclude booleans to catch accidental flag values.

## Implementation

```python
from operator import index
from typing import SupportsIndex


def read_record(data: bytes, width: int, record: SupportsIndex) -> bytes:
    if type(width) is not int or width <= 0 or len(data) % width:
        raise ValueError("expected a positive width and an aligned byte block")
    if isinstance(record, bool):
        raise TypeError("a boolean is not a record number")
    number = index(record)
    if not 0 <= number < len(data) // width:
        raise IndexError("record number is outside the byte block")
    start = number * width
    return data[start : start + width]
```

## Example

```python
class RecordNumber:
    def __index__(self) -> int:
        return 2


records = b"a1b2c3"
failures = []
for value in (True, 1.0, -1, 3):
    try:
        read_record(records, 2, value)
    except (TypeError, IndexError) as error:
        failures.append(type(error))

assert (read_record(records, 2, RecordNumber()), read_record(records, 2, 0), failures) == (
    b"c3",
    b"a1",
    [TypeError, TypeError, IndexError, IndexError],
)
```

## Trade-offs and Limitations

The slice returns bytes, not a writable view. This policy does not support Python's negative index convention or incomplete records. The index protocol may execute code supplied by the object; use it with trusted objects and validate serialized inputs before constructing them.

## Related Snippets

<!-- catalog:related:start -->
- [Build an Immutable Slice-Aware Sequence](build-an-immutable-slice-aware-sequence.md)
- [Decode Bounded Fixed-Width ASCII Records Across Arbitrary Byte Chunks](../data-processing/decode-bounded-fixed-width-ascii-records-across-arbitrary-byte-chunks.md)
<!-- catalog:related:end -->
