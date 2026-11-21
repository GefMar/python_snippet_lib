---
title: "Test Fixed-Width Header Changes in a Private File Mapping"
snippet_type: testing-technique
use_cases:
  - resource-management
  - testing
tested_python:
  - "3.14"
dependencies: []
related:
  - ../storage-databases/read-the-last-bounded-binary-lines-with-a-read-only-mmap.md
  - generate-a-bounded-one-edit-byte-mutation-corpus.md
---

# Test Fixed-Width Header Changes in a Private File Mapping

## Idea and Problem

Give a binary parser a modified fixed-width header while preserving the original bytes of a shared test fixture.

A copy-on-write `mmap.ACCESS_COPY` mapping starts with the file's bytes and
keeps assignments private to that mapping. A parser can read the changed
header using the same buffer interface it normally reads. A separate mapping
and the backing file retain the original header.

## When to Use

Use this when the parser accepts a buffer and the test changes only a field
within a known header prefix. Open a trusted, stable regular file on a system
that supports file-backed private mappings. Keep the parser invocation and
any buffer views within the mapping lifetime.

## Implementation

```python
import mmap
import os
import stat
from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path


@contextmanager
def private_header(path: Path, size: int) -> Iterator[mmap.mmap]:
    with path.open("rb") as source:
        info = os.fstat(source.fileno())
        if not stat.S_ISREG(info.st_mode) or not 0 < size <= info.st_size:
            raise ValueError("the header must be a nonempty prefix of a regular file")
        with mmap.mmap(source.fileno(), size, access=mmap.ACCESS_COPY) as header:
            yield header
```

## Example

```python
from struct import Struct
from tempfile import TemporaryDirectory

header_format = Struct(">4sH")
original = header_format.pack(b"GRID", 1) + b"sample rows"

with TemporaryDirectory() as directory:
    path = Path(directory) / "fixture.bin"
    path.write_bytes(original)

    with private_header(path, header_format.size) as trial:
        trial[4:6] = b"\x00\x02"
        observed = header_format.unpack_from(trial)
        with private_header(path, header_format.size) as reference:
            unchanged = header_format.unpack_from(reference)
        during = path.read_bytes()

    after = path.read_bytes()

assert (observed, unchanged, during, after, trial.closed, reference.closed) == (
    (b"GRID", 2),
    (b"GRID", 1),
    original,
    original,
    True,
    True,
)
```

## Trade-offs and Limitations

The mapping covers a fixed prefix. Assign an equal number of bytes when
changing a field and choose a different fixture strategy for tests that alter
the file length. Release any exported `memoryview` before closing the map.
Copy out a small result when it needs to outlive the scope.

The copy-on-write guarantee covers writes through this mapping. Other writers
can still modify or truncate the file, so keep it stable from open through
close. For a small header and a parser that works with `bytearray`, an
ordinary read into memory is often simpler.

## Related Snippets

<!-- catalog:related:start -->
- [Read the Last Bounded Binary Lines with a Read-Only mmap](../storage-databases/read-the-last-bounded-binary-lines-with-a-read-only-mmap.md)
- [Generate a Bounded One-Edit Byte Mutation Corpus](generate-a-bounded-one-edit-byte-mutation-corpus.md)
<!-- catalog:related:end -->
