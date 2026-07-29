---
title: "Interleave Bounded Finite Tuple Sources Round-Robin Until Exhausted"
snippet_type: algorithm
use_cases:
  - data-transformation
  - performance-optimization
  - validation
tested_python:
  - "3.14"
dependencies: []
related:
  - yield-stream-items-with-bounded-neighbor-context.md
  - ../algorithms-data-structures/merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md
  - ../python-language/batch-any-iterable-lazily-with-itertools-batched.md
---

# Interleave Bounded Finite Tuple Sources Round-Robin Until Exhausted

## Idea and Problem

Interleave several finite tuple sources by taking one item from each active source per round until every source is exhausted.

A source that runs out leaves the rotation, while the remaining sources keep
their declaration order. Tracking only active `(source, index)` pairs avoids
rescanning empty sources in every later round and preserves both the order and
identity of every original item.

## When to Use

Use this algorithm when all inputs are already materialized, their combined
size is bounded, and the output must alternate sources predictably without
padding shorter inputs. Empty sources are allowed, including a collection in
which every source is empty.

Use a lazy iterator when the result should be consumed incrementally, and use
an explicit scheduler when sources have weights, priorities, backpressure or
asynchronous availability. This function describes deterministic tuple
transformation rather than execution fairness.

## Implementation

```python
from collections import deque
from typing import TypeVar

ItemT = TypeVar("ItemT")

_MAX_SOURCES = 256
_MAX_TOTAL_ITEMS = 65_536


def interleave_tuple_sources_round_robin(
    sources: tuple[tuple[ItemT, ...], ...],
) -> tuple[ItemT, ...]:
    """Return one declaration-ordered round-robin interleave."""
    if type(sources) is not tuple:
        raise TypeError("sources must be an exact tuple")
    if not 1 <= len(sources) <= _MAX_SOURCES:
        raise ValueError("source count is outside the supported range")

    total_items = 0
    for source_index, source in enumerate(sources):
        if type(source) is not tuple:
            raise TypeError(f"sources[{source_index}] must be an exact tuple")
        total_items += len(source)
        if total_items > _MAX_TOTAL_ITEMS:
            raise ValueError("total item count exceeds the supported limit")

    active: deque[tuple[tuple[ItemT, ...], int]] = deque(
        (source, 0) for source in sources if source
    )
    result: list[ItemT] = []
    while active:
        source, item_index = active.popleft()
        result.append(source[item_index])
        next_index = item_index + 1
        if next_index < len(source):
            active.append((source, next_index))

    return tuple(result)
```

## Example

```python
first = ["first"]
second = ["second"]
third = ["third"]

interleaved = interleave_tuple_sources_round_robin(
    ((first, third), (), (second,))
)
all_empty = interleave_tuple_sources_round_robin(((), ()))

assert interleaved == (first, second, third)
assert interleaved[0] is first and interleaved[1] is second
assert all_empty == ()
```

## Trade-offs and Limitations

For `S` sources and `N` total items, validation and interleaving take
`O(S + N)` time. The active deque uses `O(S)` auxiliary memory, and the
materialized result uses `O(N)` memory. Items are retained by reference rather
than copied, so their identities and any external mutability remain visible.

Unlike padding-based transposition, exhausted sources contribute no sentinel
values. The exact tuple boundary deliberately rejects lazy and infinite
inputs. The function does not perform I/O, await sources, apply weights or
priorities, manage backpressure, or make scheduler-fairness guarantees.

## Related Snippets

<!-- catalog:related:start -->
- [Yield Stream Items with Bounded Neighbor Context](yield-stream-items-with-bounded-neighbor-context.md)
- [Merge Bounded Sorted Integer Runs with Observable Source-Order Ties](../algorithms-data-structures/merge-bounded-sorted-integer-runs-with-observable-source-order-ties.md)
- [Batch Any Iterable Lazily with itertools.batched](../python-language/batch-any-iterable-lazily-with-itertools-batched.md)
<!-- catalog:related:end -->
