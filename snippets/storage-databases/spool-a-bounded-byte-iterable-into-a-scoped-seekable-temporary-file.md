---
title: "Spool a Bounded Byte Iterable into a Scoped Seekable Temporary File"
snippet_type: recipe
use_cases:
  - resource-management
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - split-a-binary-stream-into-exclusively-created-numbered-parts.md
  - replace-a-file-atomically-with-a-sibling-temporary-file.md
  - ../algorithms-data-structures/sort-newline-terminated-binary-records-with-bounded-merge-passes.md
---

# Spool a Bounded Byte Iterable into a Scoped Seekable Temporary File

## Idea and Problem

Consume one finite iterable of byte chunks into a seekable temporary file while keeping small payloads in memory and moving larger payloads to temporary storage.

The context manager validates every chunk and the configured count, chunk,
total, and in-memory threshold limits. It writes a chunk only after that chunk
fits every applicable bound, consumes the producer completely, rewinds the
file, and only then exposes it. The file is closed whether production,
consumption, or the caller's work inside the context fails.

## When to Use

Use this recipe when a downstream library needs a blocking seekable binary
file, but the upstream boundary naturally produces a small bounded sequence of
byte chunks. It is useful for parsers, archive readers, or upload adapters that
may seek and reread input during one local operation.

Keep the producer finite and treat the yielded file as scoped to the `with`
block. Use direct streaming when the consumer does not need seeking, and use an
explicit durable path when another process or a later operation must reopen the
data.

## Implementation

```python
from collections.abc import Iterable, Iterator
from contextlib import contextmanager
from tempfile import SpooledTemporaryFile
from typing import BinaryIO

_HARD_MAX_CHUNKS = 4_096
_HARD_MAX_CHUNK_BYTES = 4 * 1024 * 1024
_HARD_MAX_TOTAL_BYTES = 64 * 1024 * 1024
_HARD_MAX_MEMORY_THRESHOLD_BYTES = 16 * 1024 * 1024


def _positive_bounded_limit(value: object, *, name: str, maximum: int) -> int:
    if type(value) is not int:
        raise TypeError(f"{name} must be an exact integer")
    if not 1 <= value <= maximum:
        raise ValueError(f"{name} is outside the supported range")
    return value


@contextmanager
def spool_bounded_bytes(
    chunks: Iterable[bytes],
    *,
    max_chunks: int,
    max_chunk_bytes: int,
    max_total_bytes: int,
    max_size: int,
) -> Iterator[BinaryIO]:
    chunk_count_limit = _positive_bounded_limit(
        max_chunks,
        name="max_chunks",
        maximum=_HARD_MAX_CHUNKS,
    )
    chunk_size_limit = _positive_bounded_limit(
        max_chunk_bytes,
        name="max_chunk_bytes",
        maximum=_HARD_MAX_CHUNK_BYTES,
    )
    total_size_limit = _positive_bounded_limit(
        max_total_bytes,
        name="max_total_bytes",
        maximum=_HARD_MAX_TOTAL_BYTES,
    )
    memory_threshold = _positive_bounded_limit(
        max_size,
        name="max_size",
        maximum=_HARD_MAX_MEMORY_THRESHOLD_BYTES,
    )
    if chunk_size_limit > total_size_limit:
        raise ValueError("max_chunk_bytes must not exceed max_total_bytes")

    try:
        iterator = iter(chunks)
    except TypeError:
        raise TypeError("chunks must be an iterable") from None

    with SpooledTemporaryFile(max_size=memory_threshold, mode="w+b") as stream:
        chunk_count = 0
        total_bytes = 0

        for chunk in iterator:
            if chunk_count == chunk_count_limit:
                raise ValueError("chunk count exceeds max_chunks")
            if type(chunk) is not bytes:
                raise TypeError(f"chunks[{chunk_count}] must be exact bytes")

            chunk_bytes = len(chunk)
            if not 1 <= chunk_bytes <= chunk_size_limit:
                raise ValueError(f"chunks[{chunk_count}] has an invalid byte size")
            if chunk_bytes > total_size_limit - total_bytes:
                raise ValueError("chunk bytes exceed max_total_bytes")

            written = stream.write(chunk)
            if written != chunk_bytes:
                raise OSError("temporary-file write was incomplete")
            chunk_count += 1
            total_bytes += chunk_bytes

        if chunk_count == 0:
            raise ValueError("chunks must produce at least one non-empty chunk")

        stream.seek(0)
        yield stream
```

## Example

```python
chunks = (b"header\n", b"body\n")
scoped = None

with spool_bounded_bytes(
    chunks,
    max_chunks=2,
    max_chunk_bytes=8,
    max_total_bytes=12,
    max_size=8,
) as stream:
    scoped = stream
    complete_payload = stream.read()
    stream.seek(len(chunks[0]))
    body = stream.read()
    seekable_inside = stream.seekable()

assert scoped is not None
assert (complete_payload, body, seekable_inside, scoped.closed) == (
    b"header\nbody\n",
    b"body\n",
    True,
    True,
)
```

## Trade-offs and Limitations

`max_size` is the `SpooledTemporaryFile` rollover threshold, not a hard memory
limit and not the accepted-byte limit. Data remains in memory while its file
size is at or below the threshold and normally rolls to temporary storage only
after exceeding it. Calling `fileno()` forces rollover, so portable code should
not call it merely to inspect where the data lives. The producer's current
chunk, temporary copies, allocator overhead, and any consumer writes are
outside the threshold; `max_total_bytes` governs only bytes admitted from the
producer before the initial rewind.

The default temporary directory and whether an on-disk file has a visible name
depend on the platform and process environment. Upstream iteration is eager
and may have irreversible effects before a later chunk fails validation; file
cleanup cannot roll those effects back. The yielded handle is writable and
valid only inside the context, and it is not a durable, encrypted, atomic, or
cross-process publication mechanism. A failure while closing temporary storage
is propagated rather than hidden.

## Related Snippets

<!-- catalog:related:start -->
- [Split a Binary Stream into Exclusively Created Numbered Parts](split-a-binary-stream-into-exclusively-created-numbered-parts.md)
- [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Sort Newline-Terminated Binary Records with Bounded Merge Passes](../algorithms-data-structures/sort-newline-terminated-binary-records-with-bounded-merge-passes.md)
<!-- catalog:related:end -->
