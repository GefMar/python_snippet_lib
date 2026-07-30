---
title: "Write Bounded Byte Segments Exactly with os.writev"
snippet_type: recipe
use_cases:
  - performance-optimization
  - resource-management
  - validation
tested_python:
  - "3.13"
  - "3.14"
dependencies: []
related:
  - fill-a-preallocated-bytearray-exactly-with-os-readinto.md
  - read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md
  - ../networking-protocols/read-and-write-size-capped-varint-frames.md
---

# Write Bounded Byte Segments Exactly with os.writev

## Idea and Problem

Write a small byte-vector completely while preserving partial-write progress across segment boundaries and never joining the payload.

One `os.writev()` call can consume fewer bytes than requested. Keeping an
explicit segment index and intra-segment offset lets each retry expose only the
unwritten memoryview suffix plus later segments. A typed failure reports the
prefix already accepted by the descriptor so callers do not mistake a partial
side effect for an all-or-nothing operation.

## When to Use

Use this recipe on POSIX with Python 3.13 or later for one caller-owned blocking
file descriptor and 1 through 16 immutable byte segments. Keep every segment
non-empty and the aggregate payload at or below 1,048,576 bytes. It is useful
when headers and bodies already exist separately and copying them into one
temporary buffer would add avoidable work.

Use an ordinary `write()` loop for one segment, a buffered file object when
vectored I/O is not material, and an event-loop or readiness API for
nonblocking descriptors. The caller remains responsible for opening,
synchronizing, seeking, and closing the descriptor.

## Implementation

```python
import errno
import os
from tempfile import TemporaryFile
from unittest.mock import patch

_MAX_SEGMENTS = 16
_MAX_TOTAL_BYTES = 1024 * 1024


class ExactVectorWriteError(OSError):
    def __init__(
        self,
        error_number: int,
        message: str,
        *,
        bytes_written: int,
    ) -> None:
        self.bytes_written = bytes_written
        super().__init__(error_number, message)


def write_bounded_segments_exactly(
    fd: int,
    segments: tuple[bytes, ...],
) -> int:
    if os.name != "posix" or not hasattr(os, "writev"):
        raise NotImplementedError("os.writev requires a supported POSIX platform")
    if type(fd) is not int:
        raise TypeError("fd must be an exact integer")
    if fd < 0:
        raise ValueError("fd must be nonnegative")
    if type(segments) is not tuple:
        raise TypeError("segments must be an exact tuple")
    if not 1 <= len(segments) <= _MAX_SEGMENTS:
        raise ValueError("segment count is outside the supported range")

    total_bytes = 0
    for index, segment in enumerate(segments):
        if type(segment) is not bytes:
            raise TypeError(f"segments[{index}] must be exact bytes")
        if not segment:
            raise ValueError(f"segments[{index}] must be non-empty")
        if len(segment) > _MAX_TOTAL_BYTES - total_bytes:
            raise ValueError("segment bytes exceed the aggregate limit")
        total_bytes += len(segment)

    views = tuple(memoryview(segment) for segment in segments)
    segment_index = 0
    segment_offset = 0
    total_written = 0
    try:
        while segment_index < len(views):
            head = views[segment_index][segment_offset:]
            buffers = (head, *views[segment_index + 1 :])
            try:
                try:
                    written = os.writev(fd, buffers)
                except OSError as error:
                    error_number = (
                        error.errno if type(error.errno) is int else errno.EIO
                    )
                    raise ExactVectorWriteError(
                        error_number,
                        "vectored write failed",
                        bytes_written=total_written,
                    ) from error
            finally:
                head.release()

            remaining_total = total_bytes - total_written
            if type(written) is not int or not 1 <= written <= remaining_total:
                raise ExactVectorWriteError(
                    errno.EIO,
                    "vectored write returned an invalid byte count",
                    bytes_written=total_written,
                )
            total_written += written

            unassigned = written
            while unassigned:
                available = len(views[segment_index]) - segment_offset
                if unassigned < available:
                    segment_offset += unassigned
                    unassigned = 0
                else:
                    unassigned -= available
                    segment_index += 1
                    segment_offset = 0
    finally:
        for view in views:
            view.release()

    return total_written
```

## Example

```python
real_writev = os.writev


def write_at_most_three_bytes(
    fd: int,
    buffers: tuple[memoryview, ...],
) -> int:
    limited: list[memoryview] = []
    remaining = 3
    try:
        for buffer in buffers:
            if remaining == 0:
                break
            piece = buffer[:remaining]
            limited.append(piece)
            remaining -= len(piece)
        return real_writev(fd, tuple(limited))
    finally:
        for piece in limited:
            piece.release()


segments = (b"head:", b"payload", b":tail")
with (
    TemporaryFile(mode="w+b", buffering=0) as stream,
    patch.object(os, "writev", side_effect=write_at_most_three_bytes) as mocked,
):
    written = write_bounded_segments_exactly(stream.fileno(), segments)
    stream.seek(0)
    observed = stream.read()

assert written == sum(map(len, segments))
assert observed == b"head:payload:tail"
assert mocked.call_count > 1
```

## Trade-offs and Limitations

Every positive short write advances the descriptor and the in-memory cursor.
If a later call fails, `ExactVectorWriteError.bytes_written` reports only the
prefix this helper observed; it cannot roll that prefix back. The shared file
offset also advances, so concurrent users of the same open file description
can invalidate assumptions about position.

POSIX treats each individual `writev()` transfer as one non-intermingled write,
but this helper may need several transfers. Other writers can therefore
interleave between its short-write retries. The per-call rule does not provide
full-request completion, rollback, crash atomicity, or durability. The helper
assumes a blocking descriptor and can wait indefinitely. It does not handle
readiness, deadlines, cancellation, retries after a surfaced `OSError`,
`fsync`, sparse layout, or descriptor lifecycle.

The payload remains in its original immutable segments. Each system call
allocates only a bounded tuple and memoryview suffixes, so retained auxiliary
space is `O(K)` rather than `O(B)`; the operating system may still copy data.
Runtime is `O(B + C*K)` for `B` bytes, at most 16 segments, and `C` write
calls. Platform limits or descriptor-specific constraints can be stricter and
surface as ordinary wrapped I/O errors.

## Related Snippets

<!-- catalog:related:start -->
- [Fill a Preallocated Bytearray Exactly with os.readinto](fill-a-preallocated-bytearray-exactly-with-os-readinto.md)
- [Read an Exact POSIX Byte Range with os.pread Without Moving the File Descriptor Offset](read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md)
- [Read and Write Size-Capped Varint Frames](../networking-protocols/read-and-write-size-capped-varint-frames.md)
<!-- catalog:related:end -->
