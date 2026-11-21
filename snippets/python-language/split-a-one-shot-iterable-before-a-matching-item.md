---
title: "Split a One-Shot Iterable Before a Matching Item"
snippet_type: recipe
use_cases:
  - data-transformation
  - resource-management
tested_python:
  - "3.14"
dependencies: []
related:
  - ../data-processing/yield-only-complete-trailing-windows-from-a-one-shot-iterable.md
  - read-fixed-size-blocks-with-iter-sentinel.md
---

# Split a One-Shot Iterable Before a Matching Item

## Idea and Problem

Separate a bounded preamble from a one-shot input without dropping the item that begins the remainder.

Read until the predicate matches. Chain that one matched item in front of the same iterator so the caller receives it before any later data.

## When to Use

Use this to read a small prefix from a nonseekable iterator when the boundary item belongs to the next phase. The remainder is still lazy and should be consumed only once.

## Implementation

```python
from collections.abc import Callable, Iterable, Iterator
from itertools import chain


def split_before[T](
    items: Iterable[T], boundary: Callable[[T], bool], max_prefix: int
) -> tuple[tuple[T, ...], Iterator[T]]:
    if type(max_prefix) is not int or max_prefix < 0:
        raise ValueError("max_prefix must be a nonnegative integer")

    source = iter(items)
    prefix: list[T] = []
    for item in source:
        if boundary(item):
            return tuple(prefix), chain((item,), source)
        if len(prefix) == max_prefix:
            raise ValueError("prefix exceeds the item limit")
        prefix.append(item)
    return tuple(prefix), source
```

## Example

```python
source = iter(("meta", "version", "data", "first row"))
prefix, remaining = split_before(source, lambda item: item == "data", 2)
first = next(remaining)
assert (prefix, first, tuple(remaining), tuple(source)) == (
    ("meta", "version"),
    "data",
    ("first row",),
    (),
)
```

## Trade-offs and Limitations

The prefix is materialized and the remainder shares the original iterator. If the prefix overflows, the item that demonstrated the overflow has already been consumed. This helper neither rewinds nor closes caller-owned inputs.

## Related Snippets

<!-- catalog:related:start -->
- [Yield Only Complete Trailing Windows from a One-Shot Iterable](../data-processing/yield-only-complete-trailing-windows-from-a-one-shot-iterable.md)
- [Read Fixed-Size Blocks with iter() and a Sentinel](read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
