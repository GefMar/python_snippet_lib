---
title: "Write One Noninterleaved Record to a Shared POSIX Pipe"
snippet_type: recipe
use_cases:
  - concurrency-control
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - write-bounded-byte-segments-exactly-with-os-writev.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
---

# Write One Noninterleaved Record to a Shared POSIX Pipe

## Idea and Problem

Keep each small record contiguous when several producers write to one pipe.

Append the line feed before the write. POSIX guarantees that other writers
cannot interleave bytes within a pipe write of at most `PIPE_BUF` bytes.
Query that bound for the descriptor and count the delimiter against it.

## When to Use

Use this for cooperating producers sending short byte records to one reader
through a shared POSIX pipe or FIFO. The reader frames records on line feeds.
The caller owns the open, blocking write descriptor and must keep its
lifetime and blocking mode stable for the call.

## Implementation

```python
import os
import stat


def write_pipe_record(fd: int, payload: bytes) -> None:
    if os.name != "posix":
        raise NotImplementedError("a POSIX pipe or FIFO is required")
    if type(fd) is not int or type(payload) is not bytes:
        raise TypeError("expected an integer descriptor and bytes payload")
    if not stat.S_ISFIFO(os.fstat(fd).st_mode):
        raise ValueError("descriptor does not refer to a pipe or FIFO")
    if not os.get_blocking(fd):
        raise ValueError("descriptor must be blocking")

    limit = os.fpathconf(fd, "PC_PIPE_BUF")
    if limit <= 0:
        raise OSError("no positive atomic write bound is available")
    if len(payload) >= limit:
        raise ValueError("record including its delimiter exceeds PC_PIPE_BUF")
    if b"\n" in payload:
        raise ValueError("payload contains a line feed")

    record = payload + b"\n"
    written = os.write(fd, record)
    if written != len(record):
        raise OSError(f"short pipe write: {written} of {len(record)} bytes")
```

## Example

```python
read_fd, write_fd = os.pipe()
try:
    try:
        write_pipe_record(write_fd, b"job-a done")
        limit = os.fpathconf(write_fd, "PC_PIPE_BUF")
        try:
            write_pipe_record(write_fd, b"x" * limit)
        except ValueError:
            rejected = True
        else:
            rejected = False
        write_pipe_record(write_fd, b"job-b done")
    finally:
        os.close(write_fd)

    with os.fdopen(read_fd, "rb", closefd=False) as reader:
        prefix, rest = reader.read(4), reader.read()
finally:
    os.close(read_fd)

assert rejected and prefix == b"job-" and rest == b"a done\njob-b done\n"
```

## Trade-offs and Limitations

The bound is not the pipe capacity. A full pipe may block the writer, and a
closed reader can cause a broken-pipe error. An unexpected short write is a
failure with possible output; do not write the remainder separately or replay
the whole record blindly. Producers must follow the same framing contract.

Contiguous writes do not determine producer order or create read boundaries.
The reader still needs to handle split lines and several lines in one chunk.
Use a different transport or serialize writers when records cannot fit the
atomic write bound.

## Related Snippets

<!-- catalog:related:start -->
- [Write Bounded Byte Segments Exactly with os.writev](write-bounded-byte-segments-exactly-with-os-writev.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
<!-- catalog:related:end -->
