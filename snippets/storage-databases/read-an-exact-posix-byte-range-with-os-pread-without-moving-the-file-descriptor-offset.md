---
title: "Read an Exact POSIX Byte Range with os.pread Without Moving the File Descriptor Offset"
snippet_type: recipe
use_cases:
  - performance-optimization
  - resource-management
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - read-the-last-bounded-binary-lines-with-a-read-only-mmap.md
  - spool-a-bounded-byte-iterable-into-a-scoped-seekable-temporary-file.md
  - ../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md
---

# Read an Exact POSIX Byte Range with os.pread Without Moving the File Descriptor Offset

## Idea and Problem

Read one exact bounded byte range from a POSIX file descriptor without disturbing the descriptor's shared current offset.

`os.pread` supplies the read position explicitly, so it does not need a
seek-read-seek sequence that can race with other users of the same open file
description. A regular-file read may still return fewer bytes than requested,
so the function advances the explicit position until the range is complete or
end of file proves that it cannot be completed.

## When to Use

Use this recipe for a seekable POSIX descriptor when one component needs a
small random-access byte range but another component owns or observes the
descriptor's current offset. It is useful for bounded record headers, indexes,
and format probes that must fail rather than silently accept a truncated range.

Keep descriptor ownership, lifetime, locking, and file-stability policy outside
the function. Use ordinary sequential reads when there is only one cursor owner,
and use an application-level snapshot or lock when concurrent writers could
change the requested bytes during the loop.

## Implementation

```python
import os

_MAX_FILE_DESCRIPTOR = (1 << 31) - 1
_MAX_POSIX_OFFSET = (1 << 63) - 1
_MAX_EXACT_READ_BYTES = 1_048_576


def read_exact_at(
    file_descriptor: int,
    *,
    offset: int,
    byte_count: int,
) -> bytes:
    """Read one exact POSIX range without changing the descriptor offset."""
    if type(file_descriptor) is not int:
        raise TypeError("file_descriptor must be an exact non-boolean integer")
    if not 0 <= file_descriptor <= _MAX_FILE_DESCRIPTOR:
        raise ValueError("file_descriptor is outside the supported range")

    if type(offset) is not int:
        raise TypeError("offset must be an exact non-boolean integer")
    if not 0 <= offset <= _MAX_POSIX_OFFSET:
        raise ValueError("offset is outside the supported range")

    if type(byte_count) is not int:
        raise TypeError("byte_count must be an exact non-boolean integer")
    if not 0 <= byte_count <= _MAX_EXACT_READ_BYTES:
        raise ValueError("byte_count is outside the supported range")
    if offset + byte_count > _MAX_POSIX_OFFSET:
        raise ValueError("the half-open byte range exceeds the supported offset")
    if byte_count == 0:
        return b""

    chunks: list[bytes] = []
    position = offset
    remaining = byte_count
    while remaining:
        chunk = os.pread(file_descriptor, remaining, position)
        if not chunk:
            raise EOFError("the descriptor ended before the exact range was read")
        chunks.append(chunk)
        position += len(chunk)
        remaining -= len(chunk)

    return b"".join(chunks)
```

## Example

```python
def exercise_positioned_read() -> tuple[bytes, list[tuple[int, int]], bool, bool, bytes]:
    from tempfile import TemporaryFile
    from unittest.mock import patch

    with TemporaryFile(mode="w+b") as file:
        file.write(b"0123456789")
        file.flush()
        file.seek(7)
        original_position = file.tell()

        real_pread = os.pread
        observed_reads: list[tuple[int, int]] = []

        def short_pread(descriptor: int, count: int, position: int) -> bytes:
            short_count = min(count, 2)
            observed_reads.append((short_count, position))
            return real_pread(descriptor, short_count, position)

        with patch.object(os, "pread", side_effect=short_pread):
            selected = read_exact_at(file.fileno(), offset=2, byte_count=5)

        offset_was_unchanged = file.tell() == original_position
        try:
            read_exact_at(file.fileno(), offset=8, byte_count=3)
        except EOFError:
            early_eof_rejected = True
        else:
            early_eof_rejected = False

        with patch.object(os, "pread", side_effect=AssertionError("unexpected I/O")):
            empty = read_exact_at(file.fileno(), offset=10, byte_count=0)

    return selected, observed_reads, offset_was_unchanged, early_eof_rejected, empty


assert exercise_positioned_read() == (
    b"23456",
    [(2, 2), (2, 4), (1, 6)],
    True,
    True,
    b"",
)
```

## Trade-offs and Limitations

Validation, reading, and joining use `O(byte_count)` data work and memory. A
short read always makes progress, so at most `byte_count` successful system
calls are possible, but normal regular-file reads usually need far fewer. The
returned bytes, the chunk list, and the final join can temporarily retain more
than one copy of the bounded payload.

The function is POSIX-only and does not close, seek, lock, or otherwise own the
descriptor. A zero-length request deliberately performs no system call, so it
does not prove that the supplied descriptor is open. Non-seekable descriptors
and operating-system failures propagate as `OSError`; an early end of file is
reported separately as `EOFError`.

`pread` isolates the descriptor offset, not the file contents. Concurrent
writes or truncation can still produce an inconsistent multi-read view. This
recipe does not provide snapshot semantics, retries, mmap access, asynchronous
I/O, sparse-file interpretation, or a Windows fallback.

## Related Snippets

<!-- catalog:related:start -->
- [Read the Last Bounded Binary Lines with a Read-Only mmap](read-the-last-bounded-binary-lines-with-a-read-only-mmap.md)
- [Spool a Bounded Byte Iterable into a Scoped Seekable Temporary File](spool-a-bounded-byte-iterable-into-a-scoped-seekable-temporary-file.md)
- [Read a Bounded Range from Non-Overlapping Byte Segments](../algorithms-data-structures/read-a-bounded-range-from-non-overlapping-byte-segments.md)
<!-- catalog:related:end -->
