---
title: "Read a Bounded Range from Non-Overlapping Byte Segments"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - find-a-point-in-disjoint-half-open-intervals.md
  - cover-a-half-open-integer-range-with-dyadic-intervals.md
  - ../storage-databases/read-the-last-bounded-binary-lines-with-a-read-only-mmap.md
---

# Read a Bounded Range from Non-Overlapping Byte Segments

## Idea and Problem

Build a bounded immutable index of disjoint byte segments and materialize an exact half-open address range, filling uncovered positions with one explicit byte value.

Construction sorts once and rejects duplicate or overlapping address ranges.
Adjacent segments are valid. A read of `count` bytes from `start` covers
`[start, start + count)` and always returns exactly `count` bytes after all
size and address checks succeed.

## When to Use

Use this algorithm for a finite sparse byte image that is already available as
trusted immutable `bytes` objects. It is suitable for deterministic range
assembly, protocol fixtures, and tests in which gaps have one known fill byte.
An empty segment collection is allowed; individual empty segments are rejected
because they cover no address and make duplicate-boundary rules ambiguous.

Use a sparse-file API, database, or streaming reader when the stored bytes or
requested output exceed the fixed in-memory limits. Define a separate conflict
policy when overlapping segments are meaningful.

## Implementation

```python
from bisect import bisect_right
from collections.abc import Iterable
from dataclasses import dataclass
from itertools import islice


_MAX_ADDRESS = (1 << 63) - 1
_MAX_SEGMENTS = 10_000
_MAX_SEGMENT_BYTES = 1_000_000
_MAX_STORED_BYTES = 8_000_000
_MAX_READ_BYTES = 1_000_000


@dataclass(frozen=True, slots=True)
class ByteSegment:
    address: int
    data: bytes

    def __post_init__(self) -> None:
        if type(self.address) is not int:
            raise TypeError("address must be an integer")
        if not 0 <= self.address <= _MAX_ADDRESS:
            raise ValueError("address is outside the supported range")
        if type(self.data) is not bytes:
            raise TypeError("data must be immutable bytes")
        if not 1 <= len(self.data) <= _MAX_SEGMENT_BYTES:
            raise ValueError("segment length is outside the supported range")
        if len(self.data) > _MAX_ADDRESS + 1 - self.address:
            raise ValueError("segment end exceeds the supported address space")

    @property
    def end(self) -> int:
        return self.address + len(self.data)


@dataclass(frozen=True, slots=True, init=False)
class ByteSegmentIndex:
    _segments: tuple[ByteSegment, ...]
    _starts: tuple[int, ...]

    def __init__(self, segments: Iterable[ByteSegment]) -> None:
        try:
            provided = tuple(islice(segments, _MAX_SEGMENTS + 1))
        except TypeError as error:
            raise TypeError("segments must be an iterable") from error
        if len(provided) > _MAX_SEGMENTS:
            raise ValueError("segment count exceeds the supported limit")
        if any(not isinstance(segment, ByteSegment) for segment in provided):
            raise TypeError("every item must be a ByteSegment")
        if sum(len(segment.data) for segment in provided) > _MAX_STORED_BYTES:
            raise ValueError("stored bytes exceed the supported limit")

        ordered = tuple(sorted(provided, key=lambda segment: segment.address))
        for left, right in zip(ordered, ordered[1:]):
            if right.address < left.end:
                raise ValueError("segments must not overlap or share an address")

        object.__setattr__(self, "_segments", ordered)
        object.__setattr__(
            self,
            "_starts",
            tuple(segment.address for segment in ordered),
        )

    def read(self, *, start: int, count: int, fill: int = 0) -> bytes:
        if type(start) is not int or type(count) is not int:
            raise TypeError("start and count must be integers")
        if not 0 <= start <= _MAX_ADDRESS:
            raise ValueError("start is outside the supported address space")
        if not 0 <= count <= _MAX_READ_BYTES:
            raise ValueError("count is outside the supported range")
        if count > _MAX_ADDRESS + 1 - start:
            raise ValueError("read end exceeds the supported address space")
        if type(fill) is not int:
            raise TypeError("fill must be an integer")
        if not 0 <= fill <= 255:
            raise ValueError("fill must be between 0 and 255")

        stop = start + count
        output = bytearray([fill]) * count
        position = bisect_right(self._starts, start) - 1
        if position < 0:
            position = 0
        elif self._segments[position].end <= start:
            position += 1

        for segment in self._segments[position:]:
            if segment.address >= stop:
                break
            copy_start = max(start, segment.address)
            copy_stop = min(stop, segment.end)
            source = memoryview(segment.data)
            output[copy_start - start : copy_stop - start] = source[
                copy_start - segment.address : copy_stop - segment.address
            ]
        return bytes(output)
```

## Example

```python
index = ByteSegmentIndex(
    (
        ByteSegment(8, b"XYZ"),
        ByteSegment(0, b"abc"),
        ByteSegment(3, b"de"),
    )
)

try:
    ByteSegmentIndex((ByteSegment(0, b"abc"), ByteSegment(2, b"overlap")))
except ValueError:
    overlap_rejected = True
else:
    overlap_rejected = False

try:
    index.read(start=_MAX_ADDRESS, count=2)
except ValueError:
    overflowing_read_rejected = True
else:
    overflowing_read_rejected = False

try:
    ByteSegment(4, b"")
except ValueError:
    empty_segment_rejected = True
else:
    empty_segment_rejected = False

assert (
    index.read(start=1, count=10, fill=ord(".")),
    index.read(start=2, count=4),
    ByteSegmentIndex(()).read(start=5, count=3, fill=255),
    overlap_rejected,
    overflowing_read_rejected,
    empty_segment_rejected,
) == (
    b"bcde...XYZ",
    b"cde\x00",
    b"\xff\xff\xff",
    True,
    True,
    True,
)
```

## Trade-offs and Limitations

Construction sorts in `O(n log n)` time and stores immutable tuples of segment
references and start addresses. A read finds its first candidate in
`O(log n)`, then scans intersecting segments and allocates an output whose size
was checked against `_MAX_READ_BYTES`. Converting the working byte array to
`bytes` briefly requires a second output-sized allocation.

The index treats addresses as unsigned 63-bit coordinates and does not merge
adjacent segments. It provides no file-format parsing, file access, lazy I/O,
checksums, authentication, signing, or provenance guarantees. Caller changes
cannot affect segment content because exact immutable `bytes` values are
required.

## Related Snippets

<!-- catalog:related:start -->
- [Find a Point in Disjoint Half-Open Intervals](find-a-point-in-disjoint-half-open-intervals.md)
- [Cover a Half-Open Integer Range with Dyadic Intervals](cover-a-half-open-integer-range-with-dyadic-intervals.md)
- [Read the Last Bounded Binary Lines with a Read-Only mmap](../storage-databases/read-the-last-bounded-binary-lines-with-a-read-only-mmap.md)
<!-- catalog:related:end -->
