---
title: "Render a Unified Diff for Byte Lines with Unknown Encoding"
snippet_type: testing-technique
use_cases:
  - testing
  - data-transformation
tested_python:
  - "3.14"
dependencies: []
related:
  - compare-a-bounded-text-capture-against-a-golden-fixture.md
  - ../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md
---

# Render a Unified Diff for Byte Lines with Unknown Encoding

## Idea and Problem

Produce a line-based diff for raw byte records without guessing either input's text encoding.

Pass the original records to `difflib.diff_bytes` and select unified format. Keep the returned byte lines separate so an unterminated input record does not run into the next diff record.

## When to Use

Use this for diagnostics on small byte-oriented snapshots when record boundaries are already known. Pass each record once, retaining its original newline bytes if present. Compare original bytes rather than decoding with a lossy error handler.

## Implementation

```python
from collections.abc import Sequence
from difflib import diff_bytes, unified_diff


def unified_byte_diff(before: Sequence[bytes], after: Sequence[bytes]) -> tuple[bytes, ...]:
    return tuple(
        diff_bytes(
            unified_diff,
            before,
            after,
            fromfile=b"before",
            tofile=b"after",
            n=1,
        )
    )
```

## Example

```python
before = (b"header\n", b"\xffold\n", b"tail")
after = (b"header\n", b"\xferevised\n", b"tail")

assert (unified_byte_diff(before, after), unified_byte_diff(before, before)) == (
    (
        b"--- before\n",
        b"+++ after\n",
        b"@@ -1,3 +1,3 @@\n",
        b" header\n",
        b"-\xffold\n",
        b"+\xferevised\n",
        b" tail",
    ),
    (),
)
```

## Trade-offs and Limitations

The report is a sequence of raw records, not a decoded string or a binary patch. Present arbitrary bytes with an escaping or transcoding policy instead of writing them directly to a text terminal. This holds both inputs and the output in memory, is not a streaming binary comparison, and does not guarantee a minimal change script.

## Related Snippets

<!-- catalog:related:start -->
- [Compare a Bounded Text Capture Against a Golden Fixture](compare-a-bounded-text-capture-against-a-golden-fixture.md)
- [Render a Stable Unified Diff for Nested JSON Values](../configuration-serialization/render-a-stable-unified-diff-for-nested-json-values.md)
<!-- catalog:related:end -->
