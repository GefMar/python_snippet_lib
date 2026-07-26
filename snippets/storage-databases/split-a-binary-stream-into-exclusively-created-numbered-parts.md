---
title: "Split a Binary Stream into Exclusively Created Numbered Parts"
snippet_type: recipe
use_cases:
  - resource-management
  - persistence
  - automation
tested_python:
  - "3.14"
dependencies: []
related:
  - replace-a-file-atomically-with-a-sibling-temporary-file.md
  - store-bytes-by-their-content-digest.md
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
---

# Split a Binary Stream into Exclusively Created Numbered Parts

## Idea and Problem

Copy a finite blocking binary stream into bounded numbered files without overwriting any path that already exists.

One `read(size)` call is not guaranteed to fill the requested size, so each
part combines short reads until it reaches the limit or the stream reports EOF
with `b""`. Writing only after a part has been assembled ensures every
non-final file has exactly the requested size. Exclusive file creation makes a
concurrent or stale name collision visible instead of replacing data.

## When to Use

Use this recipe in a private staging directory when a finite export must be
split before a separate upload or archival step. The caller must own the
directory, bound the input independently, and keep the source open for the
duration of the call. Use a format-specific multipart client when consumers
need checksums, a manifest, retries, resume, or atomic publication of the
complete set.

## Implementation

```python
import stat
from pathlib import Path
from typing import BinaryIO


def _read_binary_part(source: BinaryIO, part_size: int) -> bytes:
    chunks = []
    remaining = part_size
    while remaining:
        chunk = source.read(remaining)
        if not isinstance(chunk, bytes):
            raise TypeError("source.read() must return bytes")
        if not chunk:
            break
        if len(chunk) > remaining:
            raise ValueError("source.read() returned more bytes than requested")
        chunks.append(chunk)
        remaining -= len(chunk)
    return b"".join(chunks)


def split_binary_stream(
    source: BinaryIO,
    directory: Path,
    *,
    part_size: int,
) -> tuple[Path, ...]:
    if not isinstance(directory, Path):
        raise TypeError("directory must be a Path")
    if isinstance(part_size, bool) or not isinstance(part_size, int):
        raise TypeError("part_size must be an integer")
    if part_size <= 0:
        raise ValueError("part_size must be positive")

    directory_mode = directory.stat(follow_symlinks=False).st_mode
    if not stat.S_ISDIR(directory_mode):
        raise ValueError("directory must be an existing non-link directory")
    if any(directory.iterdir()):
        raise FileExistsError("directory must be empty")

    paths = []
    index = 0
    while part := _read_binary_part(source, part_size):
        path = directory / f"part-{index:08d}.bin"
        with path.open("xb") as output:
            written = output.write(part)
            if written != len(part):
                raise OSError("part file write was incomplete")
        paths.append(path)
        index += 1
    return tuple(paths)
```

## Example

```python
from io import BytesIO
from tempfile import TemporaryDirectory


class ShortReader(BytesIO):
    def read(self, size: int | None = -1) -> bytes:
        requested = 2 if size is None or size < 0 else min(size, 2)
        return super().read(requested)


with TemporaryDirectory() as temporary_directory:
    part_directory = Path(temporary_directory) / "parts"
    part_directory.mkdir()

    source = ShortReader(b"abcdefghij")
    paths = split_binary_stream(source, part_directory, part_size=4)
    payloads = tuple(path.read_bytes() for path in paths)
    source_still_open = not source.closed

with TemporaryDirectory() as temporary_directory:
    empty_directory = Path(temporary_directory) / "parts"
    empty_directory.mkdir()
    empty_paths = split_binary_stream(BytesIO(b""), empty_directory, part_size=4)

assert (
    tuple(path.name for path in paths),
    payloads,
    b"".join(payloads),
    source_still_open,
    empty_paths,
) == (
    ("part-00000000.bin", "part-00000001.bin", "part-00000002.bin"),
    (b"abcd", b"efgh", b"ij"),
    b"abcdefghij",
    True,
    (),
)
```

## Trade-offs and Limitations

The function holds at most one part plus short-read fragments in memory and
performs a linear copy, but it does not cap the number of parts or total input
size. It accepts only blocking streams whose `read()` uses `b""` exclusively
for EOF. The directory check and each exclusive open prevent ordinary
overwrites, but the multi-file result is not atomic and a concurrent actor in
the staging directory creates a race. A read or write failure can leave all
completed files and possibly the current partial file; cleanup is deliberately
left to the directory owner. There is no manifest, checksum, durability sync,
compression, encryption, upload, resume, or reassembly protocol.

## Related Snippets

<!-- catalog:related:start -->
- [Replace a File Atomically with a Sibling Temporary File](replace-a-file-atomically-with-a-sibling-temporary-file.md)
- [Store Bytes by Their Content Digest](store-bytes-by-their-content-digest.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
