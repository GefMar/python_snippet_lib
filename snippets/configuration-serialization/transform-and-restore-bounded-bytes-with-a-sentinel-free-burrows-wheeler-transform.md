---
title: "Transform and Restore Bounded Bytes with a Sentinel-Free Burrows-Wheeler Transform"
snippet_type: algorithm
use_cases:
  - data-transformation
  - serialization
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - ../algorithms-data-structures/build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md
  - ../algorithms-data-structures/find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md
  - derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md
---

# Transform and Restore Bounded Bytes with a Sentinel-Free Burrows-Wheeler Transform

## Idea and Problem

Reorder a byte block so equal contexts tend to become adjacent while retaining one row index that makes the transformation reversible.

The Burrows-Wheeler transform sorts every cyclic rotation and records the byte
preceding each rotation. The row whose rotation starts at byte zero is the
primary row. No sentinel byte is needed, so every possible byte value remains
available as data.

Cyclic prefix doubling orders rotation indexes without materializing all
rotated byte strings. During inversion, occurrence ranks distinguish equal
bytes in the last column, and LF mapping walks backward through the original
block.

## When to Use

Use this bounded, in-memory transform when experimenting with reversible block
sorting, building compact reference fixtures, or verifying another BWT
implementation. Indexed tie-breaking makes forward output reproducible for
periodic inputs whose rotations can be identical.

BWT alone does not compress bytes. A practical codec still needs a specified
container, primary-row encoding, entropy stages, integrity protection, and
resource limits. Use an established compression format for interoperable or
untrusted production data.

## Implementation

```python
from dataclasses import dataclass
from itertools import product
from random import Random

_MAX_BWT_BYTES = 65_536


@dataclass(frozen=True, slots=True)
class BurrowsWheelerBlock:
    last_column: bytes
    primary_row: int


def sentinel_free_bwt(data: bytes) -> BurrowsWheelerBlock:
    """Return the indexed cyclic-rotation BWT of one bounded byte block."""
    if type(data) is not bytes:
        raise TypeError("data must be exact bytes")
    length = len(data)
    if length > _MAX_BWT_BYTES:
        raise ValueError("data exceeds 65536 bytes")
    if length == 0:
        return BurrowsWheelerBlock(b"", 0)

    order = list(range(length))
    ranks = list(data)
    prefix_length = 1
    while prefix_length < length:
        order.sort(
            key=lambda start: (
                ranks[start],
                ranks[(start + prefix_length) % length],
                start,
            )
        )
        next_ranks = [0] * length
        class_index = 0
        previous_key: tuple[int, int] | None = None
        for start in order:
            key = ranks[start], ranks[(start + prefix_length) % length]
            if previous_key is not None and key != previous_key:
                class_index += 1
            next_ranks[start] = class_index
            previous_key = key
        ranks = next_ranks
        if class_index + 1 == length:
            break
        prefix_length *= 2

    return BurrowsWheelerBlock(
        last_column=bytes(data[(start - 1) % length] for start in order),
        primary_row=order.index(0),
    )


def restore_sentinel_free_bwt(
    last_column: bytes,
    primary_row: int,
) -> bytes:
    """Invert one bounded last column with occurrence-ranked LF mapping."""
    if type(last_column) is not bytes:
        raise TypeError("last_column must be exact bytes")
    length = len(last_column)
    if length > _MAX_BWT_BYTES:
        raise ValueError("last_column exceeds 65536 bytes")
    if type(primary_row) is not int:
        raise TypeError("primary_row must be an exact integer")
    if length == 0:
        if primary_row != 0:
            raise ValueError("an empty last column requires primary_row zero")
        return b""
    if not 0 <= primary_row < length:
        raise ValueError("primary_row is outside the last column")

    totals = [0] * 256
    occurrences: list[int] = []
    for value in last_column:
        occurrences.append(totals[value])
        totals[value] += 1

    offsets = [0] * 256
    preceding = 0
    for value, count in enumerate(totals):
        offsets[value] = preceding
        preceding += count
    lf = tuple(
        offsets[value] + occurrence
        for value, occurrence in zip(last_column, occurrences, strict=True)
    )

    restored = bytearray(length)
    row = primary_row
    for output_index in range(length - 1, -1, -1):
        restored[output_index] = last_column[row]
        row = lf[row]
    return bytes(restored)
```

## Example

```python
def naive_indexed_bwt(data: bytes) -> BurrowsWheelerBlock:
    if not data:
        return BurrowsWheelerBlock(b"", 0)
    order = sorted(
        range(len(data)),
        key=lambda start: (data[start:] + data[:start], start),
    )
    return BurrowsWheelerBlock(
        last_column=bytes(data[(start - 1) % len(data)] for start in order),
        primary_row=order.index(0),
    )


banana = sentinel_free_bwt(b"banana")
assert banana == naive_indexed_bwt(b"banana")
assert restore_sentinel_free_bwt(banana.last_column, banana.primary_row) == b"banana"
assert sentinel_free_bwt(b"") == BurrowsWheelerBlock(b"", 0)

checked = 0
for length in range(8):
    for values in product(range(3), repeat=length):
        data = bytes(values)
        block = sentinel_free_bwt(data)
        assert block == naive_indexed_bwt(data)
        assert restore_sentinel_free_bwt(block.last_column, block.primary_row) == data
        checked += 1

rng = Random(0xB7_00)
samples = [
    bytes(range(256)),
    bytes(range(256)) * 4,
    b"ab" * 2_048,
    b"same" * 1_024,
]
samples.extend(rng.randbytes(rng.randrange(513)) for _ in range(1_000))
for data in samples:
    block = sentinel_free_bwt(data)
    assert restore_sentinel_free_bwt(block.last_column, block.primary_row) == data
    checked += 1

assert checked == 4_284
```

## Trade-offs and Limitations

Prefix doubling performs `O(log N)` Python sorts, each costing
`O(N log N)`, for `O(N log² N)` time and `O(N)` state. The inverse uses
`O(N + 256)` time and state. This is deliberately transparent reference code;
specialized suffix-array construction can be faster and more memory-efficient.

The pair `(last_column, primary_row)` is meaningful only under the documented
byte ordering and indexed-rotation convention. The inverse validates shape and
range, but it cannot authenticate a block or detect arbitrary corruption. No
sentinel, framing, entropy coding, streaming protocol, checksum, or external
file format is defined here.

## Related Snippets

<!-- catalog:related:start -->
- [Build a Canonical Suffix Array and Adjacent LCP Table for Bounded Unicode Text](../algorithms-data-structures/build-a-canonical-suffix-array-and-adjacent-lcp-table-for-bounded-unicode-text.md)
- [Find the Earliest Lexicographically Smallest Rotation of Bounded Unicode Text with Booth's Algorithm](../algorithms-data-structures/find-the-earliest-lexicographically-smallest-rotation-of-bounded-unicode-text-with-booths-algorithm.md)
- [Derive Deterministic Huffman Code Lengths from Bounded Byte Frequencies](derive-deterministic-huffman-code-lengths-from-bounded-byte-frequencies.md)
<!-- catalog:related:end -->
