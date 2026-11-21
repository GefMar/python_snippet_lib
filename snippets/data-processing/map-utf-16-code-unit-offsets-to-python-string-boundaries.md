---
title: "Map UTF-16 Code-Unit Offsets to Python String Boundaries"
snippet_type: recipe
use_cases:
  - data-transformation
  - interoperability
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../observability-operations/truncate-valid-utf-8-text-to-a-byte-budget-with-an-explicit-marker.md
  - ../configuration-serialization/decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md
---

# Map UTF-16 Code-Unit Offsets to Python String Boundaries

## Idea and Problem

Translate a UTF-16 offset to a Python slicing index while rejecting offsets that split a surrogate pair.

Each Unicode scalar value contributes one Python string position. Values
above `U+FFFF` occupy two UTF-16 code units. Store the cumulative unit count
at every Python boundary, including the start and end. A binary search finds
the matching Python index only when the requested offset is in the table.

## When to Use

Use this when an editor, annotation tool or text exchange API supplies UTF-16
code-unit offsets into an exact text revision. Build the table once for that
string, then reuse it to convert multiple endpoints. For line-relative
positions, build the table for the exact line and apply the protocol's newline
rules before mapping columns.

## Implementation

```python
from bisect import bisect_left


def utf16_boundary_offsets(text: str) -> tuple[int, ...]:
    """Return the UTF-16 unit offset at each Python string boundary."""
    if type(text) is not str:
        raise TypeError("text must be a string")

    offsets = [0]
    units = 0
    for char in text:
        code_point = ord(char)
        if 0xD800 <= code_point <= 0xDFFF:
            raise ValueError("text must contain only Unicode scalar values")
        units += 2 if code_point > 0xFFFF else 1
        offsets.append(units)
    return tuple(offsets)


def python_index_from_utf16(boundaries: tuple[int, ...], offset: int) -> int:
    """Look up an exact offset in a table from utf16_boundary_offsets()."""
    if type(offset) is not int:
        raise TypeError("offset must be an integer")

    index = bisect_left(boundaries, offset)
    if index == len(boundaries) or boundaries[index] != offset:
        raise ValueError("offset is outside the text or splits a surrogate pair")
    return index
```

## Example

```python
text = "A\U0001f680e\u0301"
boundaries = utf16_boundary_offsets(text)
start = python_index_from_utf16(boundaries, 1)
stop = python_index_from_utf16(boundaries, 3)

try:
    python_index_from_utf16(boundaries, 2)
except ValueError:
    split_rejected = True
else:
    split_rejected = False

assert (boundaries, text[start:stop], split_rejected) == (
    (0, 1, 3, 4, 5),
    "\U0001f680",
    True,
)
```

## Trade-offs and Limitations

Construction takes one pass and stores one offset per boundary. Each lookup
takes logarithmically many comparisons. Rebuild the table whenever the source
text changes. For a single query on a very long line, a linear scan can avoid
the table's storage cost.

Offsets count UTF-16 units in decoded text independently of byte order or
encoding markers. Grapheme clusters can span several Python indices, as the
base `e` and combining accent do in the example. Use a grapheme-aware text
library when cursor movement must follow user-perceived characters.

## Related Snippets

<!-- catalog:related:start -->
- [Truncate Valid UTF-8 Text to a Byte Budget with an Explicit Marker](../observability-operations/truncate-valid-utf-8-text-to-a-byte-budget-with-an-explicit-marker.md)
- [Decode One Bounded Strict UTF-8 Stream Across Arbitrary Byte Chunks](../configuration-serialization/decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md)
<!-- catalog:related:end -->
