---
title: "Yield Stream Items with Bounded Neighbor Context"
snippet_type: recipe
use_cases:
  - data-transformation
  - performance-optimization
tested_python:
  - "3.14"
dependencies: []
related:
  - ../python-language/read-fixed-size-blocks-with-iter-sentinel.md
---

# Yield Stream Items with Bounded Neighbor Context

## Idea and Problem

Yield each stream item with bounded chronological history and one explicit lookahead without materializing the complete source.

Some transformations need a small amount of surrounding context for every
item. A bounded deque retains previous items, while the iteration loop reads
one following item before yielding the current one. Representing lookahead as
an empty or one-item tuple keeps a real `None` value distinct from end of
stream.

## When to Use

Use this recipe for a single ordered stream when each output needs up to a
fixed number of preceding items and at most one following item. The source must
already have the required grouping and ordering. Reset the iterator at group
boundaries rather than allowing context to cross between independent groups.

## Implementation

```python
from collections import deque
from collections.abc import Iterable, Iterator
from dataclasses import dataclass
from typing import Generic, TypeVar


ItemT = TypeVar("ItemT")


@dataclass(frozen=True, slots=True)
class NeighborContext(Generic[ItemT]):
    previous: tuple[ItemT, ...]
    current: ItemT
    following: tuple[ItemT, ...]


def iter_neighbor_contexts(
    items: Iterable[ItemT],
    *,
    history_size: int,
) -> Iterator[NeighborContext[ItemT]]:
    if history_size < 0:
        raise ValueError("history_size must not be negative")

    iterator = iter(items)
    try:
        current = next(iterator)
    except StopIteration:
        return

    previous: deque[ItemT] = deque(maxlen=history_size)
    for following in iterator:
        yield NeighborContext(tuple(previous), current, (following,))
        previous.append(current)
        current = following

    yield NeighborContext(tuple(previous), current, ())
```

## Example

```python
consumed = []


def source():
    for item in ("first", None, "third"):
        consumed.append(item)
        yield item


contexts = iter_neighbor_contexts(source(), history_size=1)
first_context = next(contexts)
consumed_after_first_output = tuple(consumed)
remaining_contexts = list(contexts)

zero_history = list(iter_neighbor_contexts((1, 2), history_size=0))
empty_result = list(iter_neighbor_contexts((), history_size=2))

try:
    list(iter_neighbor_contexts((1,), history_size=-1))
except ValueError:
    negative_history_rejected = True
else:
    negative_history_rejected = False

assert (
    first_context,
    consumed_after_first_output,
    remaining_contexts,
    zero_history,
    empty_result,
    negative_history_rejected,
) == (
    NeighborContext((), "first", (None,)),
    ("first", None),
    [
        NeighborContext(("first",), None, ("third",)),
        NeighborContext((None,), "third", ()),
    ],
    [NeighborContext((), 1, (2,)), NeighborContext((), 2, ())],
    [],
    True,
)
```

## Trade-offs and Limitations

The first output requires reading two source items, so errors or blocking in
the lookahead happen before that output is available. Each result copies up to
`history_size` references into a tuple, making total work proportional to the
requested context size. The function does not identify group boundaries,
rewind the source, or recover from upstream errors. It is lazy and bounded, but
it is not suitable when consumers need arbitrary future context.

## Related Snippets

<!-- catalog:related:start -->
- [Read Fixed-Size Blocks with iter() and a Sentinel](../python-language/read-fixed-size-blocks-with-iter-sentinel.md)
<!-- catalog:related:end -->
