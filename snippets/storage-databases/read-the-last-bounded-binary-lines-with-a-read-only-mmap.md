---
title: "Read the Last Bounded Binary Lines with a Read-Only mmap"
snippet_type: recipe
use_cases:
  - parsing
  - performance-optimization
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - split-a-binary-stream-into-exclusively-created-numbered-parts.md
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
  - ../data-processing/limit-text-lines-across-arbitrary-chunks.md
---

# Read the Last Bounded Binary Lines with a Read-Only mmap

## Idea and Problem

Read a bounded number of final binary lines newest first without loading or scanning an already size-capped regular file from the beginning.

A read-only `mmap` searches backward for newline bytes inside a bounded window.
The function removes only the synthetic empty record after one terminal
newline, preserves meaningful interior empty lines, and materializes each
selected result before closing the mapping and file.

## When to Use

Use this recipe for a stable local regular file whose complete size is below a
trusted cap, when callers need only a few final byte records and will decide
how to decode them separately. It fits bounded diagnostics and small log-tail
operations. Use a streaming reverse reader or an indexed format when mapping
the complete file is undesirable.

## Implementation

```python
import mmap
import os
import stat
from pathlib import Path


class SelectedLineTooLongError(ValueError):
    pass


def read_last_binary_lines(
    path: Path,
    *,
    max_lines: int,
    max_file_bytes: int,
    max_line_bytes: int,
) -> tuple[bytes, ...]:
    limits = (
        ("max_lines", max_lines, 10_000),
        ("max_file_bytes", max_file_bytes, 1_000_000_000),
        ("max_line_bytes", max_line_bytes, 16_000_000),
    )
    for name, value, hard_cap in limits:
        if isinstance(value, bool) or not isinstance(value, int):
            raise TypeError(f"{name} must be an integer")
        if not 1 <= value <= hard_cap:
            raise ValueError(f"{name} must be between 1 and {hard_cap}")

    with Path(path).open("rb") as source:
        metadata = os.fstat(source.fileno())
        if not stat.S_ISREG(metadata.st_mode):
            raise ValueError("path must identify a regular file")
        file_size = metadata.st_size
        if file_size > max_file_bytes:
            raise ValueError("file exceeds max_file_bytes")
        if file_size == 0:
            return ()

        lines: list[bytes] = []
        with mmap.mmap(source.fileno(), length=0, access=mmap.ACCESS_READ) as mapped:
            end = file_size
            if mapped[end - 1 : end] == b"\n":
                end -= 1

            while len(lines) < max_lines and end >= 0:
                window_start = max(0, end - max_line_bytes - 1)
                separator = mapped.rfind(b"\n", window_start, end)
                if separator < 0:
                    if window_start > 0:
                        raise SelectedLineTooLongError(
                            "a selected line exceeds max_line_bytes",
                        )
                    start = 0
                else:
                    start = separator + 1

                if end - start > max_line_bytes:
                    raise SelectedLineTooLongError(
                        "a selected line exceeds max_line_bytes",
                    )
                lines.append(bytes(mapped[start:end]))
                if separator < 0:
                    break
                end = separator

        return tuple(lines)
```

## Example

```python
from tempfile import TemporaryDirectory


def read(path: Path, *, max_lines: int = 10, max_line_bytes: int = 8):
    return read_last_binary_lines(
        path,
        max_lines=max_lines,
        max_file_bytes=100,
        max_line_bytes=max_line_bytes,
    )


with TemporaryDirectory() as temporary_directory:
    path = Path(temporary_directory) / "records.bin"
    cases: list[tuple[bytes, tuple[bytes, ...]]] = []
    for content in (b"", b"only", b"only\n", b"\n", b"a\n\nb\n"):
        path.write_bytes(content)
        cases.append((content, read(path)))

    path.write_bytes(b"a\n\nb\n")
    bounded = read(path, max_lines=2)
    path.write_bytes(b"1234")
    exact_cap = read(path, max_line_bytes=4)
    path.write_bytes(b"12345")
    try:
        read(path, max_line_bytes=4)
    except SelectedLineTooLongError:
        oversized_rejected = True
    else:
        oversized_rejected = False

assert (cases, bounded, exact_cap, oversized_rejected) == (
    [
        (b"", ()),
        (b"only", (b"only",)),
        (b"only\n", (b"only",)),
        (b"\n", (b"",)),
        (b"a\n\nb\n", (b"b", b"", b"a")),
    ],
    (b"b", b""),
    (b"1234",),
    True,
)
```

## Trade-offs and Limitations

The complete file is mapped into the process address space even though only
bounded suffix windows are searched. The returned lines consume bounded
materialized memory, and the file-size cap bounds the mapping. Newline bytes
are removed, but carriage returns and all other bytes remain; decoding and
newline normalization belong to the caller. Concurrent truncation or
replacement is unsupported and can invalidate a mapping. Filesystem and
mapping errors propagate, and overlong selected lines fail instead of being
silently truncated.

## Related Snippets

<!-- catalog:related:start -->
- [Split a Binary Stream into Exclusively Created Numbered Parts](split-a-binary-stream-into-exclusively-created-numbered-parts.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
- [Limit Text Lines Across Arbitrary Chunks](../data-processing/limit-text-lines-across-arbitrary-chunks.md)
<!-- catalog:related:end -->
