---
title: "Fill a Preallocated Bytearray Exactly with os.readinto"
snippet_type: recipe
use_cases:
  - performance-optimization
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md
  - split-a-binary-stream-into-exclusively-created-numbered-parts.md
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
---

# Fill a Preallocated Bytearray Exactly with os.readinto

## Idea and Problem

Fill one caller-owned bytearray in place from a blocking file descriptor, preserving both short-read progress and partial state when the transfer fails.

`os.readinto` writes directly into a writable buffer and may return fewer bytes
than requested. Repeatedly exposing only the unfilled memoryview suffix prevents
a later read from overwriting the prefix and avoids allocating and joining
temporary byte strings. A zero result before the bytearray is full is a distinct
early-end condition rather than successful completion.

## When to Use

Use this recipe on Python 3.14 or later when the caller already owns an open,
blocking file descriptor and an exact `bytearray` of the required final size.
The bytearray must contain between 1 and 1,048,576 bytes, and the caller must
accept visible prefix mutation and descriptor-offset advancement if the read
ends early or raises.

Use `os.pread` when the shared descriptor offset must not move. Use a file
object's `readinto` method when the caller owns a buffered file-object contract,
or collect immutable chunks when an all-or-nothing returned value matters more
than in-place storage.

## Implementation

```python
import os

_MAX_BUFFER_BYTES = 1_048_576
_MAX_FILE_DESCRIPTOR = 2_147_483_647


class UnexpectedEOF(EOFError):
    def __init__(self, *, filled: int, expected: int) -> None:
        self.filled = filled
        self.expected = expected
        super().__init__(f"unexpected EOF after {filled} of {expected} bytes")


def fill_bytearray_exactly(fd: int, buffer: bytearray) -> bytearray:
    if type(fd) is not int:
        raise TypeError("fd must be an exact integer")
    if not 0 <= fd <= _MAX_FILE_DESCRIPTOR:
        raise ValueError("fd is outside the supported range")
    if type(buffer) is not bytearray:
        raise TypeError("buffer must be an exact bytearray")
    if not 1 <= len(buffer) <= _MAX_BUFFER_BYTES:
        raise ValueError("buffer length is outside the supported range")

    expected = len(buffer)
    filled = 0
    view = memoryview(buffer)
    try:
        while filled < expected:
            suffix = view[filled:]
            try:
                count = os.readinto(fd, suffix)
            finally:
                suffix.release()
            if count == 0:
                raise UnexpectedEOF(filled=filled, expected=expected)
            filled += count
    finally:
        view.release()
    return buffer
```

## Example

```python
def exercise_fill() -> tuple[bool, bytes, int, bool, int, int, bytes, bool]:
    from tempfile import TemporaryFile
    from unittest.mock import patch

    real_readinto = os.readinto

    def read_at_most_two(fd: int, target: memoryview) -> int:
        short_target = target[:2]
        try:
            return real_readinto(fd, short_target)
        finally:
            short_target.release()

    with TemporaryFile(mode="w+b", buffering=0) as stream:
        stream.write(b"abcdefgh")
        stream.seek(1)
        buffer = bytearray(b"????")
        with patch.object(os, "readinto", side_effect=read_at_most_two):
            returned = fill_bytearray_exactly(stream.fileno(), buffer)
        success_offset = stream.tell()
        descriptor_survived_success = os.fstat(stream.fileno()).st_size == 8

    with TemporaryFile(mode="w+b", buffering=0) as stream:
        stream.write(b"xyz")
        stream.seek(0)
        incomplete = bytearray(b"!!!!!!")
        with patch.object(os, "readinto", side_effect=read_at_most_two):
            try:
                fill_bytearray_exactly(stream.fileno(), incomplete)
            except UnexpectedEOF as error:
                early_filled = error.filled
                early_expected = error.expected
            else:
                raise AssertionError("early EOF was accepted")
        early_offset = stream.tell()
        descriptor_survived_eof = os.fstat(stream.fileno()).st_size == 3

    return (
        returned is buffer,
        bytes(buffer),
        success_offset,
        descriptor_survived_success,
        early_filled,
        early_expected,
        bytes(incomplete),
        descriptor_survived_eof and early_offset == 3,
    )


assert exercise_fill() == (True, b"bcde", 5, True, 3, 6, b"xyz!!!", True)
```

## Trade-offs and Limitations

The function performs `O(B)` transfer work with `O(1)` auxiliary storage for a
bytearray of length `B`. Every positive read advances by at least one byte, so
success needs at most `B` system calls; an early zero result also keeps the
total at or below `B`. The caller retains the only payload allocation, and the
main and suffix memoryviews are released even when reading raises.

The helper never closes, seeks, duplicates, locks, or changes blocking mode on
the descriptor. `OSError` propagates unchanged. The already written prefix and
the descriptor's current offset remain advanced after either `OSError` or
`UnexpectedEOF`; there is no rollback. Concurrent descriptor use can interleave
offset changes and bytes, so the caller must provide exclusive cursor ownership.

This recipe excludes nonblocking descriptors, readiness waits, timeouts,
cancellation, retries, generic writable buffers, file objects, and seek-based
recovery. It also assumes the operating-system `readinto` contract: a positive
result never exceeds the writable suffix supplied by the function.

## Related Snippets

<!-- catalog:related:start -->
- [Read an Exact POSIX Byte Range with os.pread Without Moving the File Descriptor Offset](read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md)
- [Split a Binary Stream into Exclusively Created Numbered Parts](split-a-binary-stream-into-exclusively-created-numbered-parts.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
