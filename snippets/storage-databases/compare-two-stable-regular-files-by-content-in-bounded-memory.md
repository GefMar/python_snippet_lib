---
title: "Compare Two Stable Regular Files by Content in Bounded Memory"
snippet_type: recipe
use_cases:
  - validation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
  - read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md
---

# Compare Two Stable Regular Files by Content in Bounded Memory

## Idea and Problem

Compare the full contents of two stable regular files without loading them into memory or relying on cached file metadata.

Read equal-sized blocks from both files in lockstep. Different bytes or an early end on one side proves inequality; only matching end-of-file proves equality.

## When to Use

Use this on trusted paths to ordinary, blocking regular files when you need direct byte equality and can keep both files stable for the duration of the read.

## Implementation

```python
from pathlib import Path


def same_file_bytes(left: Path, right: Path, block_size: int = 65536) -> bool:
    if type(block_size) is not int or block_size <= 0:
        raise ValueError("block size must be a positive integer")

    with left.open("rb") as a, right.open("rb") as b:
        while True:
            left_bytes = a.read(block_size)
            right_bytes = b.read(block_size)
            if left_bytes != right_bytes:
                return False
            if not left_bytes:
                return True
```

## Example

```python
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    root = Path(directory)
    first, second = root / "first", root / "second"
    first.write_bytes(b"abc123")
    second.write_bytes(b"abc123")
    equal = same_file_bytes(first, second, block_size=3)

    second.write_bytes(b"abc12!")
    different_suffix = same_file_bytes(first, second, block_size=3)

    second.write_bytes(b"abc")
    shorter = same_file_bytes(first, second, block_size=3)

    first.write_bytes(b"")
    second.write_bytes(b"")
    both_empty = same_file_bytes(first, second, block_size=3)

assert (equal, different_suffix, shorter, both_empty) == (True, False, False, True)
```

## Trade-offs and Limitations

The snapshot is not atomic if another process modifies either file. This follows ordinary filesystem path resolution and is not a safe opener for untrusted paths, special devices, or pipes. The read uses bounded memory but may process both entire files; filesystem errors propagate.

## Related Snippets

<!-- catalog:related:start -->
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
- [Read an Exact POSIX Byte Range with os.pread Without Moving the File Descriptor Offset](read-an-exact-posix-byte-range-with-os-pread-without-moving-the-file-descriptor-offset.md)
<!-- catalog:related:end -->
