---
title: "Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests"
snippet_type: testing-technique
use_cases:
  - testing
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - ../configuration-serialization/decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md
  - ../networking-protocols/encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md
---

# Enumerate Every Contiguous Chunking of Bounded Bytes for Stream Tests

## Idea and Problem

Enumerate every way to divide a short byte payload into non-empty contiguous chunks so stream behavior can be checked independently of fragmentation.

A non-empty payload of length `N` has `N - 1` gaps between adjacent bytes.
Choosing which gaps become cuts produces exactly `2**(N - 1)` chunkings.
Grouping cut sets first by their size and then by their positions gives a
stable order: the whole payload comes first, followed by two-chunk cases, and
the all-single-byte case comes last.

## When to Use

Use this generator for a small, already materialized fixture when a decoder,
parser, hash accumulator, or other stream consumer must behave identically for
every placement of chunk boundaries. It is especially useful for exposing
incorrect assumptions that a character, field, or protocol token always fits
inside one input chunk.

The thirteen-byte limit is intentionally small because the result count doubles
with every additional byte. Use selected boundary cases, seeded random
fragmentation, or property-based generation for longer payloads. If an API
allows empty chunks, test that separate behavior explicitly; this function
enumerates partitions of the payload itself and therefore emits only non-empty
chunks.

## Implementation

```python
from itertools import combinations, pairwise

_MAX_CHUNKING_BYTES = 13


def enumerate_contiguous_byte_chunkings(
    payload: bytes,
) -> tuple[tuple[bytes, ...], ...]:
    """Return every non-empty contiguous chunking in canonical cut order."""
    if type(payload) is not bytes:
        raise TypeError("payload must be exact bytes")
    if len(payload) > _MAX_CHUNKING_BYTES:
        raise ValueError("payload exceeds the supported byte limit")
    if not payload:
        return ((),)

    chunkings: list[tuple[bytes, ...]] = []
    cut_positions = range(1, len(payload))
    for cut_count in range(len(payload)):
        for cuts in combinations(cut_positions, cut_count):
            boundaries = (0, *cuts, len(payload))
            chunks = tuple(payload[start:stop] for start, stop in pairwise(boundaries))
            chunkings.append(chunks)

    return tuple(chunkings)
```

## Example

```python
def chunkings_from_cut_masks(payload: bytes) -> tuple[tuple[bytes, ...], ...]:
    """Independently enumerate and sort the bitmask representation of cuts."""
    if not payload:
        return ((),)

    def cut_positions(mask: int) -> tuple[int, ...]:
        return tuple(
            position for position in range(1, len(payload)) if mask & (1 << (position - 1))
        )

    masks = sorted(
        range(1 << (len(payload) - 1)),
        key=lambda mask: (mask.bit_count(), cut_positions(mask)),
    )
    result: list[tuple[bytes, ...]] = []
    for mask in masks:
        chunks: list[bytes] = []
        start = 0
        for position in range(1, len(payload)):
            if mask & (1 << (position - 1)):
                chunks.append(payload[start:position])
                start = position
        chunks.append(payload[start:])
        result.append(tuple(chunks))
    return tuple(result)


def exercise_short_binary_payloads() -> int:
    from itertools import product

    checked = 0
    for length in range(8):
        for values in product((0, 1), repeat=length):
            payload = bytes(values)
            chunkings = enumerate_contiguous_byte_chunkings(payload)
            expected_count = 1 if not payload else 1 << (len(payload) - 1)
            assert chunkings == chunkings_from_cut_masks(payload)
            assert len(chunkings) == expected_count
            assert len(chunkings) == len(set(chunkings))
            assert all(b"".join(chunks) == payload and all(chunks) for chunks in chunkings)
            checked += 1
    return checked


abcd_order = (
    (b"abcd",),
    (b"a", b"bcd"),
    (b"ab", b"cd"),
    (b"abc", b"d"),
    (b"a", b"b", b"cd"),
    (b"a", b"bc", b"d"),
    (b"ab", b"c", b"d"),
    (b"a", b"b", b"c", b"d"),
)
utf8_bytes = "€".encode()
utf8_chunkings = enumerate_contiguous_byte_chunkings(utf8_bytes)
maximal_payload = b"0123456789abc"
maximal_chunkings = enumerate_contiguous_byte_chunkings(maximal_payload)

try:
    enumerate_contiguous_byte_chunkings(bytearray(b"abc"))  # type: ignore[arg-type]
except TypeError:
    mutable_bytes_rejected = True
else:
    mutable_bytes_rejected = False

try:
    enumerate_contiguous_byte_chunkings(b"0123456789abcd")
except ValueError:
    oversized_payload_rejected = True
else:
    oversized_payload_rejected = False

assert (
    enumerate_contiguous_byte_chunkings(b""),
    enumerate_contiguous_byte_chunkings(b"abcd"),
    exercise_short_binary_payloads(),
    (b"\xe2", b"\x82", b"\xac") in utf8_chunkings,
    len(maximal_chunkings),
    maximal_chunkings[0] == (maximal_payload,),
    maximal_chunkings[-1] == tuple(bytes((value,)) for value in maximal_payload),
    mutable_bytes_rejected,
    oversized_payload_rejected,
) == (
    ((),),
    abcd_order,
    255,
    True,
    4_096,
    True,
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

For `N > 0`, the function returns `C = 2**(N - 1)` chunkings. Constructing
their tuples and byte slices takes `Theta(N * C)` conservative copy and output
work because each logical result contains all `N` payload bytes. The result
uses output-proportional memory, and the temporary list briefly coexists with
the returned outer tuple. Python may reuse some trivial `bytes` slice objects,
but callers should not depend on that implementation detail.

Ordering by chunk count keeps cases of similar fragmentation together; within
one count, lexicographic cut positions make the order independent of raw
bitmask numbering. The empty payload has one partition with zero chunks. For a
non-empty payload, every emitted chunk is non-empty, contiguous, and appears in
the same byte order as the input.

This function materializes every result. It does not generate empty chunks,
sample random boundaries, perform actual I/O, decode text, execute a parser,
define protocol framing, preserve mutable buffer identity, or scale beyond the
fixed combinatorial limit.

## Related Snippets

<!-- catalog:related:start -->
- [Decode One Bounded Strict UTF-8 Stream Across Arbitrary Byte Chunks](../configuration-serialization/decode-one-bounded-strict-utf-8-stream-across-arbitrary-byte-chunks.md)
- [Encode a Bounded HTTP/1.1 Chunked Body for Protocol Tests](../networking-protocols/encode-a-bounded-http-1-1-chunked-body-for-protocol-tests.md)
<!-- catalog:related:end -->
