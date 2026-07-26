---
title: "Read Fixed-Size Blocks with iter() and a Sentinel"
snippet_type: idiom
use_cases:
  - resource-management
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - walk-a-tree-recursively-with-yield-from.md
---

# Read Fixed-Size Blocks with iter() and a Sentinel

## Idea and Problem

Use the callable-and-sentinel form of iter to read a binary stream lazily until the end-of-file marker appears.

Many stream-processing tasks need the same loop: read a bounded block, stop at
end of file, and yield every non-empty result. Passing a zero-argument callable
and a sentinel to `iter` expresses that protocol without duplicating the read
and termination logic.

## When to Use

Use this form with a synchronous binary stream whose `read(size)` method returns
`b""` only when no more data is available. It is useful when the consumer can
process one block at a time and should not load the complete payload into
memory. The caller retains ownership of the stream and decides when to close
it.

## Implementation

```python
from collections.abc import Iterator
from functools import partial
from typing import BinaryIO


def iter_binary_blocks(stream: BinaryIO, block_size: int) -> Iterator[bytes]:
    if block_size <= 0:
        raise ValueError("block_size must be positive")

    return iter(partial(stream.read, block_size), b"")
```

## Example

```python
from io import BytesIO


stream = BytesIO(b"abcdefgh")
blocks = list(iter_binary_blocks(stream, 3))

try:
    iter_binary_blocks(stream, 0)
except ValueError as error:
    invalid_size_rejected = str(error) == "block_size must be positive"
else:
    invalid_size_rejected = False

assert (blocks, stream.closed, invalid_size_rejected) == (
    [b"abc", b"def", b"gh"],
    False,
    True,
)
```

## Trade-offs and Limitations

The sentinel is compared by equality, so it must be a value that cannot be a
valid non-terminal result. A stream that can return `b""` temporarily would
stop the iterator too early. The compact iterator also offers no built-in place
for retry, progress, cancellation, or per-read logging; use an explicit loop
when those policies matter. This helper is synchronous and deliberately does
not close the stream.

## Related Snippets

<!-- catalog:related:start -->
- [Walk a Tree Recursively with yield from](walk-a-tree-recursively-with-yield-from.md)
<!-- catalog:related:end -->
