---
title: "Batch Any Iterable Lazily with itertools.batched"
snippet_type: idiom
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/batch-items-by-estimated-byte-size.md
  - read-fixed-size-blocks-with-iter-sentinel.md
---

# Batch Any Iterable Lazily with itertools.batched

## Idea and Problem

Use the standard library batching iterator to consume any iterable lazily as fixed-size tuples without reimplementing iterator slicing.

The final tuple is shorter by default when the input length is not divisible by
the requested size. Set `strict=True` when a partial final batch indicates
invalid input rather than a normal end of stream.

## When to Use

Use this idiom when an API or processing step accepts at most a fixed number of
items and the source may be a generator, file iterator, or another single-pass
stream. The underlying primitive was added in Python 3.12, but this wrapper
always passes its `strict` keyword and therefore requires Python 3.13 or later.
Use byte-size batching when item sizes vary enough that a fixed count does not
control memory or request size.

## Implementation

```python
from collections.abc import Iterable, Iterator
from itertools import batched
from typing import TypeVar


T = TypeVar("T")


def iter_fixed_batches(
    items: Iterable[T],
    size: int,
    *,
    require_complete: bool = False,
) -> Iterator[tuple[T, ...]]:
    return batched(items, size, strict=require_complete)
```

## Example

```python
consumed: list[int] = []


def source() -> Iterator[int]:
    for value in range(5):
        consumed.append(value)
        yield value


batches = iter_fixed_batches(source(), 2)
first = next(batches)
consumed_after_first = consumed.copy()
remaining = list(batches)

try:
    list(iter_fixed_batches(range(5), 2, require_complete=True))
except ValueError:
    incomplete_rejected = True
else:
    incomplete_rejected = False

assert (
    first,
    consumed_after_first,
    remaining,
    list(iter_fixed_batches([], 3)),
    incomplete_rejected,
) == (
    (0, 1),
    [0, 1],
    [(2, 3), (4,)],
    [],
    True,
)
```

## Trade-offs and Limitations

Each batch is a newly allocated tuple, so this bounds the item count rather
than the memory occupied by a batch. The source is consumed once and cannot be
rewound unless it provides that behavior itself. With `strict=True`, the error
is delayed until iteration reaches an incomplete final batch, whose items have
already been consumed. A batch size below one raises `ValueError`. Avoid
yielding shared sub-iterators as batches: consuming them out of order can make
their contents interfere.

## Related Snippets

<!-- catalog:related:start -->
- [Batch Items by Estimated Byte Size](../data-processing/batch-items-by-estimated-byte-size.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
