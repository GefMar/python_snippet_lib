---
title: "Batch Items by Estimated Byte Size"
snippet_type: algorithm
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - yield-stream-items-with-bounded-neighbor-context.md
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
---

# Batch Items by Estimated Byte Size

## Idea and Problem

Group a stream lazily into tuples whose positive estimated sizes never exceed a strict per-batch byte limit.

A count-based batch can vary widely in encoded size. Measuring each item once
before adding it provides a predictable estimate-based bound. Rejecting an item
larger than the limit preserves the guarantee instead of quietly yielding an
oversized single-item batch.

## When to Use

Use this algorithm before a bounded payload or buffering boundary when the
caller can cheaply estimate each item's encoded size. The estimator must
return a positive integer and include any per-item overhead required by the
target format. Use actual serialization or a protocol-aware builder when an
estimate cannot safely represent the real payload.

## Implementation

```python
from collections.abc import Callable, Iterable, Iterator
from typing import TypeVar


ItemT = TypeVar("ItemT")


class ItemExceedsBatchLimit(ValueError):
    pass


def batch_by_estimated_size(
    items: Iterable[ItemT],
    *,
    max_bytes: int,
    estimate_size: Callable[[ItemT], int],
) -> Iterator[tuple[ItemT, ...]]:
    if isinstance(max_bytes, bool) or not isinstance(max_bytes, int):
        raise TypeError("max_bytes must be an integer")
    if max_bytes <= 0:
        raise ValueError("max_bytes must be positive")

    batch = []
    batch_size = 0
    for item in items:
        item_size = estimate_size(item)
        if isinstance(item_size, bool) or not isinstance(item_size, int):
            raise TypeError("estimated item size must be an integer")
        if item_size <= 0:
            raise ValueError("estimated item size must be positive")
        if item_size > max_bytes:
            raise ItemExceedsBatchLimit(
                f"estimated item size {item_size} exceeds limit {max_bytes}"
            )

        if batch and batch_size + item_size > max_bytes:
            yield tuple(batch)
            batch = []
            batch_size = 0

        batch.append(item)
        batch_size += item_size

    if batch:
        yield tuple(batch)
```

## Example

```python
items = (b"aa", b"bbb", b"c", b"dddd")
measurements = []


def measured_length(item: bytes) -> int:
    measurements.append(item)
    return len(item)


batches = list(
    batch_by_estimated_size(
        items,
        max_bytes=5,
        estimate_size=measured_length,
    )
)

late_failure = batch_by_estimated_size(
    (b"aaa", b"bbb", b"oversized"),
    max_bytes=5,
    estimate_size=len,
)
first_completed_batch = next(late_failure)
try:
    next(late_failure)
except ItemExceedsBatchLimit:
    oversized_rejected = True
else:
    oversized_rejected = False

try:
    list(batch_by_estimated_size((b"",), max_bytes=5, estimate_size=len))
except ValueError:
    zero_size_rejected = True
else:
    zero_size_rejected = False

assert (
    batches,
    tuple(measurements),
    first_completed_batch,
    oversized_rejected,
    zero_size_rejected,
    list(batch_by_estimated_size((), max_bytes=5, estimate_size=len)),
) == (
    [(b"aa", b"bbb"), (b"c", b"dddd")],
    items,
    (b"aaa",),
    True,
    True,
    [],
)
```

## Trade-offs and Limitations

The bound is only as accurate as the estimator and excludes batch-envelope
overhead unless the caller accounts for it. Positive estimates prevent an
unbounded number of zero-size items in one batch, but they also reject
legitimate empty encodings. A late invalid or oversized item can raise after
earlier batches were consumed. The function has no item-count limit, retry,
serialization, or transport policy.

## Related Snippets

<!-- catalog:related:start -->
- [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
