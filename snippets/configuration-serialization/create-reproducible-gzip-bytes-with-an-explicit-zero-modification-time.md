---
title: "Create Reproducible gzip Bytes with an Explicit Zero Modification Time"
snippet_type: recipe
use_cases:
  - resource-management
  - serialization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - build-a-deterministic-size-capped-ustar-archive-from-bytes.md
  - ../security-privacy/decompress-exactly-one-zlib-stream-under-input-and-output-limits.md
  - ../storage-databases/store-bytes-by-their-content-digest.md
---

# Create Reproducible gzip Bytes with an Explicit Zero Modification Time

## Idea and Problem

Remove wall-clock metadata from one bounded in-memory gzip result while keeping the compression settings explicit.

`gzip.compress()` normally writes a complete gzip member around a DEFLATE
payload. Passing `mtime=0` fixes the four-byte modification-time field, so the
same input and compression level produce the same bytes within one supported
Python and zlib runtime.

## When to Use

Use this recipe for small generated payloads when repeated work in one governed
runtime must not change solely because the current time changed. It fits cache
artifacts, reproducible local fixtures, and content-addressed outputs whose
producer version and compression level are part of the contract.

Use a versioned packaging process when compressed bytes must remain identical
across Python or zlib upgrades. Use a bounded decompression boundary for
untrusted gzip input; reproducible creation does not make later decompression
safe.

## Implementation

```python
import gzip

_MAX_GZIP_INPUT_BYTES = 4_194_304


def create_reproducible_gzip_bytes(
    data: bytes,
    *,
    compresslevel: int = 9,
) -> bytes:
    """Compress one bounded byte string with an explicit zero mtime."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    if len(data) > _MAX_GZIP_INPUT_BYTES:
        raise ValueError("data exceeds the supported input limit")
    if type(compresslevel) is not int:
        raise TypeError("compresslevel must be an exact integer")
    if not 0 <= compresslevel <= 9:
        raise ValueError("compresslevel must be between zero and nine")

    return gzip.compress(data, compresslevel=compresslevel, mtime=0)
```

## Example

```python
payload = b"bounded payload\n" * 8
first = create_reproducible_gzip_bytes(payload, compresslevel=6)
second = create_reproducible_gzip_bytes(payload, compresslevel=6)

assert (
    first == second,
    first[4:8],
    gzip.decompress(first),
    create_reproducible_gzip_bytes(b""),
) == (
    True,
    b"\x00\x00\x00\x00",
    payload,
    gzip.compress(b"", compresslevel=9, mtime=0),
)
```

## Trade-offs and Limitations

The input cap is checked before compression and includes empty input. The
function materializes the complete gzip result, which may be slightly larger
than incompressible source data, and the compressor uses additional bounded
runtime memory. Higher compression levels can trade more CPU for a smaller
result without guaranteeing that the result will shrink.

Zero modification time removes one source of variation; it is not a
cross-version canonical gzip format. Python, zlib, compression-level, or gzip
header behavior can change output bytes, so producer versions belong in any
long-lived reproducibility contract. The gzip CRC detects some accidental
corruption but is not authentication. This function provides no encryption,
signature, streaming, archive metadata, decompression limit, or protection
against malicious compressed input.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Deterministic Size-Capped USTAR Archive from Bytes](build-a-deterministic-size-capped-ustar-archive-from-bytes.md)
- [Decompress Exactly One Zlib Stream Under Input and Output Limits](../security-privacy/decompress-exactly-one-zlib-stream-under-input-and-output-limits.md)
- [Store Bytes by Their Content Digest](../storage-databases/store-bytes-by-their-content-digest.md)
<!-- catalog:related:end -->
