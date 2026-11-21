---
title: "Decompress Gzip Bytes with an Explicit Output Limit"
snippet_type: recipe
use_cases:
  - parsing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - create-reproducible-gzip-bytes-with-an-explicit-zero-modification-time.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
---

# Decompress Gzip Bytes with an Explicit Output Limit

## Idea and Problem

Read gzip bytes without returning more decoded data than the caller's budget allows.

Request one byte more than the output limit. A full extra byte proves the
limit is exceeded. A shorter read reaches the end of the stream, so the
reader checks the trailer before the function returns the result.

## When to Use

Use this when the caller already has a small gzip payload in memory and
needs an all-or-nothing decoded value. Choose limits from the expected
compressed and decoded sizes. This recipe accepts the concatenated members
and zero padding understood by Python's gzip reader.

## Implementation

```python
import gzip
import sys
from io import BytesIO


def decompress_gzip(blob: bytes, *, input_limit: int, output_limit: int) -> bytes:
    if input_limit < 0 or not 0 <= output_limit < sys.maxsize:
        raise ValueError("invalid size limits")
    if len(blob) > input_limit:
        raise ValueError("compressed input exceeds its limit")
    if not blob.startswith(b"\x1f\x8b"):
        raise gzip.BadGzipFile("expected a gzip member")

    with BytesIO(blob) as compressed, gzip.GzipFile(fileobj=compressed, mode="rb") as stream:
        decoded = stream.read(output_limit + 1)

    if len(decoded) > output_limit:
        raise ValueError("decoded output exceeds its limit")
    return decoded
```

## Example

```python
blob = gzip.compress(b"alpha") + gzip.compress(b"beta")
decoded = decompress_gzip(blob, input_limit=128, output_limit=9)
empty = decompress_gzip(gzip.compress(b""), input_limit=128, output_limit=0)

try:
    decompress_gzip(blob, input_limit=128, output_limit=8)
except ValueError:
    oversized = True
else:
    oversized = False

try:
    decompress_gzip(blob[:-1], input_limit=128, output_limit=9)
except EOFError:
    truncated = True
else:
    truncated = False

assert (decoded, empty, oversized, truncated) == (b"alphabeta", b"", True, True)
```

## Trade-offs and Limitations

An oversized stream may be rejected before its trailer is checked. A
successful decode checks gzip integrity fields, which do not authenticate
the data. Invalid streams can raise errors from gzip or zlib.

The limits check supplied and returned bytes. Python's decoder can buffer
some additional data internally, and the caller has already allocated the
compressed input. For large files, stream through a size-limited sink instead
of accumulating the result.

## Related Snippets

<!-- catalog:related:start -->
- [Create Reproducible gzip Bytes with an Explicit Zero Modification Time](create-reproducible-gzip-bytes-with-an-explicit-zero-modification-time.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
